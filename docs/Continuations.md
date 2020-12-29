因此，我们的评估程序现在以“连续传递样式”实现，并且我们所有的函数（无论是在新的λanguage中定义还是在JavaScript中定义的原语）都将第一个参数作为结果调用的回调（尽管该参数是显式的）对于图元，但对于λ语言中定义的函数不可见。



该回调代表程序的整个未来。下一步工作的完整表示。我们将其称为“当前延续”，并且在代码中将名称k用作该变量。



一方面，如果我们从不调用延续，那么程序将停止。我们不能从λ语言中做到这一点，因为make_lambda创建的函数总是调用它们的延续。但是我们可以从原始中做到这一点。我在此页面中包括以下原始内容以说明这一点：

```js
globalEnv.def("halt", function(k){});
```

这是什么都不做的功能。它收到一个延续（k），但没有调用它。

```js
println("foo");
halt();
println("bar");

```

Run result:

```
foo
```

如果删除halt（），则输出为foo / bar / ***结果：false（因为最后一个println返回false）。但是使用halt（），输出仅为“ foo”。在这种情况下甚至没有结果，因为halt（）不会调用延续—因此，我们传递给要评估的回调（即显示*** Result行的回调）未到达。调用者函数从不会注意到程序已停止。如果您正在使用NodeJS，则可能已经用这把枪开了枪。



假设我们要编写一个“睡眠”功能，该功能可以延迟程序，但不会冻结浏览器（因此，不会产生脏循环）。我们可以简单地将其实现为原语：

```
globalEnv.def("sleep", function(k, milliseconds){
  setTimeout(function(){
    Execute(k, [ false ]); // continuations expect a value, pass false
  }, milliseconds);
});
```



一个不便之处是我们必须使用Execute，因为setTimeout将放弃当前的堆栈帧。直接调用k（false）不会包装在我们的循环中，并且在第一个Continuation异常上失败。这是我们的用法：（也请尝试单击“快速运行”几次，以查看调用是“并行”运行的）：

```
let loop (n = 0) {
  if n < 10 {
    println(n);
    sleep(250);
    loop(n + 1);
  }
};
println("And we're done");
```

Run result:

```
0
1
2
3
4
5
6
7
8
9
And we're done
***Result: false
```

那是用普通的JavaScript无法做到的，除了通过手动以连续传递样式重写代码（并且不能使用for循环）之外：

```
(function loop(n){
  if (n < 10) {
    console.log(n);
    setTimeout(function(){
      loop(n + 1);
    }, 250);
  } else {
    println("And we're done"); // rest of the program goes here.
  }
})(0);
```





想象一下围绕NodeJS文件系统API的一些原语(primitive)，即：

```
globalEnv.def("readFile", function(k, filename){
  fs.readFile(filename, function(err, data){
    // error handling is a bit more complex, ignoring for now
    Execute(k, [ data ]); // hope it's clear why we need the Execute
  });
});
globalEnv.def("writeFile", function(k, filename, data){
  fs.writeFile(filename, data, function(err){
    Execute(k, [ false ]);
  });
});
```

有了它们，我们可以在λanguage中做到这一点：
```
copyFile = λ(source, dest) {
  writeFile(dest, readFile(source));
};
copyFile("foo.txt", "bar.txt");
```
这是异步的。不再是回调地狱。

这是一个更加令人困惑的例子。 我写了以下原语(primitive)：

```
globalEnv.def("twice", function(k, a, b){
  k(a);
  k(b);
});
```

它接受两个参数a和b，并调用两次连续，每个参数一次。 让我们运行一个示例：
```
println(2 + twice(3, 4));
println("Done");

```
Run result:

```
5
Done
***Result: false
6
Done
***Result: false

```

如果您以前从未处理过continuations，那么输出就会很奇怪，我将让您自己去弄清楚。只是一个简短的提示:当你点击“运行”时，这个程序运行一次;但是它返回两次调用者


#### A generalization: CallCC

到目前为止，我们仅通过编写原始函数（在JS中）就设法忍住了火。 没有办法从λ语言截取当前的延续。 CallCC原语填补了这一空白。 该名称是Scheme的带当前持续呼叫的缩写（在Scheme中通常也被拼写为call / cc）。
```
globalEnv.def("CallCC", function(k, f){
    f(k, function CC(discarded, ret){
        k(ret);
    });
});
```
It “reifies” the current continuation into a function that can be called directly from the new λanguage. That function will ignore its own continuation (discarded) and will invoke instead the original continuation that CallCC had.
它将当前连续性“修改”为可以直接从新的λ语言调用的函数。 该函数将忽略其自身的连续性（丢弃），而是调用CallCC的原始连续性。

使用这个工具，我们可以实现(直接在λ语言，而不是作为基元!)以前无法想象的广泛的控制操作符，从异常到返回。让我们从后者开始。

##### 实现 Return
```
foo = λ(return){
  println("foo");
  return("DONE");
  println("bar");
};
CallCC(foo);
```

Run result：
```
foo
***Result: DONE
```

我们的foo函数接收一个有效地充当JS的return语句的参数（仅像普通函数一样被调用，因为那有点儿）。 它放弃了自己当前的延续（本来应该打印“ bar”），并以给定的返回值（“ DONE”）从函数中提前退出。 当然，只有将其嵌套在CallCC中，才能将延续作为返回值传递，这才起作用。 我们可以为此提供一个包装器：
```
with-return = λ(f) λ() CallCC(f);

foo = with-return(λ(return){
  println("foo");
  return("DONE");
  println("bar");
});

foo();

```
好处是，现在我们在调用foo时不需要使用CallCC。 当然，如果我们甚至不必将return命名为函数参数，或者首先使用with-return，那将是很好的，但是我们的λanguage不允许从其本身进行语法扩展，因此我们需要 至少修改解析器（Lisp为+1）。

#### Easy backtracking 简单回溯
假设我们要创建一个程序，查找所有成对的两个正整数，最大为100，然后乘以84。这不是一个难题，您可能会想像想象两个嵌套循环来解决它，但是在这里，我们将采用不同的方法 方法。 我们将实现两个功能，猜测和失败。 前者会选择一个数字，后者会告诉它“尝试另一个值”。 我们将以这种方式使用它们：
```
a = guess(1);  # returns some number >= 1
b = guess(a);  # returns some number >= a
if a * b == 84 {
  # we have a solution:
  print(a); print(" x "); println(b);
};
fail();  # go back to the last `guess` and try another value
```

```
fail = λ() false;
guess = λ(current) {
  CallCC(λ(k){
    let (prevFail = fail) {
      fail = λ(){
        current = current + 1;
        if current > 100 {
          fail = prevFail;
          fail();
        } else {
          k(current);
        };
      };
      k(current);
    };
  });
};

a = guess(1);
b = guess(a);
if a * b == 84 {
  print(a); print(" x "); println(b);
};
fail();
```

现在我们可以做进一步的优化，注意当a大于84的平方根或b大于84 / a时继续进行是没有意义的。 为此，我们可以猜测两个参数，即开始和结束，并且只能选择该范围内的数字。 但是我休息一下，让我们继续。

#### ES6 yield

EcmaScript 6将引入一个神奇的yield操作符，它的语义在当前JavaScript中无法实现，除非将代码由内而外转换为延续传递样式。在我们新的λ语言中，显式的延续给了我们实现yield的方法。

但获得收益率并不完全是在公园里散步。结果比我预期的要微妙得多，所以我把关于yield的讨论放到了单独的页面上。它是高级的，对于理解本材料的其余部分来说不是必需的，但是如果您想更深入地了解延续，我确实推荐它

#### Common Lisp的catch和throw

我们将实现两个构造，即catch和throw。 它们可以这样使用：
```
f1 = λ() {
  throw("foo", "EXIT");
  print("not reached");
};
println(catch("foo", λ() {
  f1();
  print("not reached");
}));  # prints EXIT
```
因此，要解释一下，catch（TAG，function）将安装一个捕获TAG“ exception”的钩子，并调用该函数。 throw（TAG，value）将跳回该TAG的最嵌套的catch，并使其返回value。 无论是捕获功能正常退出还是通过抛出退出，在捕获的“动态范围”完成后，都将卸载钩子。

我将在下面概述一个实现
```
## with no catch handlers, throw just prints an error.
## even better would be to provide an `error` primitive
## which throws a JavaScript exception. but I'll skip that.
throw = λ(){
  println("ERROR: No more catch handlers!");
  halt();
};

catch = λ(tag, func){
  CallCC(λ(k){
    let (rethrow = throw, ret) {
      ## install a new handler that catches the given tag.
      throw = λ(t, val) {
        throw = rethrow;  # either way, restore the saved handler
        if t == tag then k(val)
                    else throw(t, val);
      };
      ## then call our function and store the result
      ret = func();
      ## if our function returned normally (not via a throw)
      ## then we will get here.  restore the old handler.
      throw = rethrow; # XXX
      ## and return the result.
      ret;
    };
  });
};
```

您可以在下面测试
```
throw = λ(){
  println("ERROR: No more catch handlers!");
  halt();
};

catch = λ(tag, func){
  CallCC(λ(k){
    let (rethrow = throw, ret) {
      ## install a new handler that catches the given tag.
      throw = λ(t, val) {
        throw = rethrow;
        if t == tag then k(val)
                    else throw(t, val);
      };
      ## then call our function and store the result
      ret = func();
      ## if our function returned normally (not via a throw)
      ## then we will get here.  restore the old handler.
      throw = rethrow;  # XXX
      ## and return the result.
      ret;
    };
  });
};

f1 = λ() {
  throw("foo", "EXIT");
  print("not reached");
};
println(catch("foo", λ() {
  f1();
  print("not reached");
}));
```

#### 力量的阴暗面 The Dark Side of the Force

在上面对catch的描述中，我说过，在catch的“动态范围”完成后，将钩子卸载。 如果您看一下代码，情况似乎就是这样，但是CallCC的强大功能可以规避这一点。 从哲学上讲，这里有两个职位。 我全力支持“给人民的力量”-允许用户颠覆该语言可能不是一个坏主意。 但是在这种特殊情况下，我认为是这样，因为如果catch / throw违反了他们的诺言，那会给用户带来混乱而不是帮助。

犯规的方法很简单。 您将调用存储在catch外部的延续。 然后，先前的throw处理程序将不会恢复，因为catch甚至都不知道您已经退出了该块。 例如：
```
exit = false;  # create a global.
x = 0; # guard: don't loop endlessly when calling exit() below
CallCC( λ(k) exit = k );
## exit() will restart from here...
if x == 0 then catch("foo", λ(){
  println("in catch");
  x = 1; # guard up
  exit();
});
println("After catch");
throw("foo", "FOO");

```

上面的代码显示两次“After catch”，然后是“ERROR: No more catch handlers!”。正确的操作应该是只显示“After catch”一次，然后显示错误。但是通过退出函数并保存在catch的动态范围之外的continuation, catch定义中标记为“XXX”的行永远不会到达，因此旧的throw处理程序不会恢复，catch之外的throw调用将简单地跳转回去。

(为了正确地实现异常，我们需要带分隔符的延续。它们在yield的实现中进行了讨论。)

这是反对CallCC的许多有效参数之一(强调:反对CallCC提供的未定界延续)。尽管我很喜欢CallCC，但我认为最好将未分隔的延续保持在原始级别，并且没有CallCC将它们公开为一等公民。不过，我确实发现延续很吸引人，我认为对这个概念的良好理解对于能够实现在实践中真正有用的其他构造是至关重要的。