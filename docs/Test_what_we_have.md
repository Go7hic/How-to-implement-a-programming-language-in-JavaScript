## What did we get

到目前为止，我们的λ语言虽然很小，但（理论上）能够解决任何可以通过计算解决的问题。这是因为Alonzo Church和Alan Turing是比我更聪明的人，在更文明的时代证明了λ微积分与Turing机器等效，而我们的λ语言模型λ微积分

这实际上意味着的是，即使我们的λ语言缺乏某些功能，我们也可以根据我们所拥有的语言来实现它们，或者如果事实证明太困难了，我们可以为其中的另一种语言编写解释器。

#### 循环

当我们有递归时，循环是不必要的。我已经展示了一个通过递归循环的示例。让我们尝试一下。请注意，您可以编辑这些示例中的代码，并使用右上角变暗的“运行”按钮运行它（也可以：CTRL-ENTER运行，CTRL-L清除输出-这些工作在光标位于左侧的代码区）；使用print / println进行输出。随时随地玩吧，是您的浏览器来运行它。

```js
print_range = λ(a, b) if a <= b {
                        print(a);
                        if a + 1 <= b {
                          print(", ");
                          print_range(a + 1, b);
                        } else println("");
                      };
print_range(1, 10);
```

打印结果：

```js
1, 2, 3, 4, 5, 6, 7, 8, 9, 10
***Result: false
```

但是，如果我们将范围增加到1000，而不是10，则会出现问题。在打印600（或其他内容）后，我得到“已超出最大调用堆栈大小”。这是因为我们的评估器是递归的，将耗尽JavaScript堆栈。



这是一个严重的问题，但是有解决方案。添加一些迭代关键字似乎很诱人，例如for或while，但不要这样。递归很漂亮，我们不必放弃。稍后我们将看到如何解决此限制.



#### 数据结构（或缺少数据结构）

我们的λ语言似乎具有三种类型：数字，字符串和布尔值。创建复合结构（例如对象或列表）似乎是不可能的。但实际上，我们还有另外一种类型：函数。事实证明，在λ微积分中，我们可以使用函数来构造任何数据结构，包括具有继承关系的对象



##### 在这里我将显示列表

希望我们有一个名为cons的函数，该函数构造一个包含两个值的特殊对象。我们将该对象称为“ cons cell”或“ pair”。我们将其中一个值命名为“ car”，将另一个值命名为“ cdr”，因为数十年来这就是在Lisp中对其进行命名的方式。给定一个单元格对象，我们可以使用函数car和cdr从该对中检索相应的值。因此：

```js
x = cons(10, 20);
print(car(x));    # prints 10
print(cdr(x));    # prints 20
```

有了这些，很容易定义一个列表：

> 列表是一个单元格对象，在其汽车中具有第一个元素，而在其cdr中具有其余元素。但是cdr可以存储单个值！该值是一个列表。列表是一个单元格对象，在其汽车中具有第一个元素，而在其cdr中具有其余元素。但是cdr可以存储单个值！该值是一个列表 […]

因此，这是一个递归数据结构（根据自身定义）。唯一的问题仍然存在：我们什么时候停止？直观地，当cdr为空列表时，我们停止，但是什么是空列表？为此，我们将介绍一个称为NIL的新特殊对象。它可以被视为一个单元格（我们可以在其上应用car和cdr，但结果本身就是它）—但这不是一个真正的单元格……让我们说这是一个假单元格。然后，下面是构造列表1、2、3、4、5的方法：

```js
x = cons(1, cons(2, cons(3, cons(4, cons(5, NIL)))));
print(car(x));                      # 1
print(car(cdr(x)));                 # 2  in Lisp this is abbrev. cadr
print(car(cdr(cdr(x))));            # 3                          caddr
print(car(cdr(cdr(cdr(x)))));       # 4                          cadddr
print(car(cdr(cdr(cdr(cdr(x))))));  # 5  but no abbreviation for this one.
```

当缺少正确的语法时，它看起来很糟糕。但是我只是想表明，有可能在我们看似有限的λ语言中建立这样的数据结构。以下是定义：

```js
cons = λ(a, b) λ(f) f(a, b);
car = λ(cell) cell(λ(a, b) a);
cdr = λ(cell) cell(λ(a, b) b);
NIL = λ(f) f(NIL, NIL);
```

当我第一次看到cons / car / cdr以这种方式实现时，我发现它们甚至不需要if就吓人了（但这并不令人惊讶，因为原始的λ微积分中没有if）。当然，没有一种语言在实践中会这样做，因为它的效率很低，但这并不会降低它的美观性。用英语来说，上面的代码指出：

- cons接受两个值(a, b)并返回一个关闭它们的函数。该功能是“单元对象”。它接受一个函数参数（f），并使用存储的两个值对其进行调用。

- car 接受一个“单元对象”（即该函数），并使用一个接收两个参数并返回第一个参数的函数对其进行调用。

- cdr就像car，但是它发送到单元格的函数返回第二个参数。

- NIL模仿一个单元格，因为它是一个函数，它接受一个函数参数（f），但始终使用两个NIL-s进行调用（因此car（NIL）和cdr（NIL）都等于NIL）。



  ```js
  cons = λ(a, b) λ(f) f(a, b);
  car = λ(cell) cell(λ(a, b) a);
  cdr = λ(cell) cell(λ(a, b) b);
  NIL = λ(f) f(NIL, NIL);
  
  x = cons(1, cons(2, cons(3, cons(4, cons(5, NIL)))));
  println(car(x));                      # 1
  println(car(cdr(x)));                 # 2
  println(car(cdr(cdr(x))));            # 3
  println(car(cdr(cdr(cdr(x)))));       # 4
  println(car(cdr(cdr(cdr(cdr(x))))));  # 5打印结果
  ```

  打印结果

  ```
  1
  2
  3
  4
  5
  ***Result: false
  ```

  有很多有趣的列表算法可以递归实现，这很自然，因为结构本身是递归定义的。例如，下面的函数将一个函数应用于列表的每个元素：

  ```js
  foreach = λ(list, f)
              if list != NIL {
                f(car(list));
                foreach(cdr(list), f);
              };
  foreach(x, println);
  打印结果
  ```

  打印结果

  ```js
  1
  2
  3
  4
  5
  ***Result: false
  ```


这是另一个构建带有一系列数字的列表的列表：

```js
range = λ(a, b)
          if a <= b then cons(a, range(a + 1, b))
                    else NIL;

# print the squares of 1..8
foreach(range(1, 8), λ(x) println(x * x));
```

打印结果：

```js
1
4
9
16
25
36
49
64
***Result: false
```

到目前为止，我们的列表是不可变的（一旦创建了单元，就无法更改汽车或cdr）。大多数Lisps提供了一些更改Cons单元的方法。在Scheme中，它称为set-car！/ set-cdr!。在Common Lisp中，它们被暗示地命名为rplaca / rplacd。这次我们更喜欢Scheme名称：

```js
cons = λ(x, y)
         λ(a, i, v)
           if a == "get"
              then if i == 0 then x else y
              else if i == 0 then x = v else y = v;

car = λ(cell) cell("get", 0);
cdr = λ(cell) cell("get", 1);
set-car! = λ(cell, val) cell("set", 0, val);
set-cdr! = λ(cell, val) cell("set", 1, val);

# NIL can be a real cons this time
NIL = cons(0, 0);
set-car!(NIL, NIL);
set-cdr!(NIL, NIL);

## test:
x = cons(1, 2);
println(car(x));
println(cdr(x));
set-car!(x, 10);
set-cdr!(x, 20);
println(car(x));
println(cdr(x));

```

打印结果：

```js
1
2
10
20
***Result: false
```

这表明可以实现可变的复合数据结构。我不会评论它是如何工作的，这非常简单。

我们可以前进并实现对象，但是无法从λ语言内部修改语法会使它很尴尬。另一种选择是在令牌生成器/解析器中引入新的语法，并可能在评估器中添加新的语义。这就是所有主流语言所要做的，并且它是获得可接受性能的许多必要步骤。我们将在下一节中探索添加一些新语法。



