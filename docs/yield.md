# 实现 yield 

First, let's see what yield is by looking at some usage examples
首先，通过查看一些用法示例来了解yield

```
foo = with-yield(λ(yield){
  yield(1);
  yield(2);
  yield(3);
  "DONE";
});

println(foo());  # prints 1
println(foo());  # prints 2
println(foo());  # prints 3
println(foo());  # prints DONE
```

When called, yield stops execution of the function and returns a value. Pretty much like return so far. But when you call the function again, it resumes from where the last yield left off, as you can see in the above example. It prints 1, 2, 3, DONE.
调用时，yield停止执行该函数并返回一个值。到目前为止，很像返回。但是，当您再次调用该函数时，它将从上次中断的收益处恢复，如上例所示。它打印1，2，3，完成。 

A more real-world usage:
更实际的用法：

```
fib = with-yield(λ(yield){
  let loop (a = 1, b = 1) {
    yield(b);
    loop(b, a + b);
  };
});

## print the first 50 fibonacci numbers
let loop (i = 0) {
  if i < 50 {
    println(fib());
    loop(i + 1);
  };
};
```

The fib function contains an infinite loop. There is no termination condition. But yield interrupts that loop and returns the current number. Calling the function again resumes from where it left off, so to print the first 50 numbers then we just need to call the function 50 times.

fib函数包含一个无限循环。 没有终止条件。 但是yield会中断该循环并返回当前数字。 从上次中断的地方重新开始调用该函数，因此要打印前50个数字，我们只需要调用该函数50次即可。

This program will also run much faster than the duble-recursive version of fib, because it keeps track of the previous two values internally and just returns the next one.
这个程序的运行速度也比双递归版本的fib快得多，因为它在内部跟踪前两个值，只返回下一个值。

#### 一次失败的尝试
我不会把时间浪费在失败的尝试上，除非你能从中学到宝贵的教训，所以请容忍我。

要实现yield，我们似乎需要保存函数的初始延续。我们需要它，这样我们才能早点退出，就像我们在返回时所做的那样。然而，在实际从一个yield函数返回之前，我们还必须保存yield本身的延续，以便以后能够返回到函数内部的那个点。至少看起来是这样。我们可以尝试这样的实现:
```
with-yield = λ(func) {
  ## with-yield returns a function of no arguments
  λ() {
    CallCC(λ(kret){        # kret is the "return" continuation
      let (yield) {

        ## define yield
        yield = λ(value) { # yield takes a value to return
          CallCC(λ(kyld){  # kyld is yield's own continuation…
            func = kyld;   # …save it for later
            kret(value);   # and return.
          });
        };

        ## finally, call the current func, passing it the yield.
        func(yield);
      };
    });
  };
};
```

这是我执行yield的第一次尝试，一切看起来都很有意义。但如果我们尝试运行上面的第一个示例来定义with-yield，它将是一个无尽的循环。如果您愿意，请尝试下面的方法，但请准备刷新页面(我实际上在该页中修改了Execute，以使用setTimeout而不是异常来清除堆栈，因此它不会冻结您的浏览器)。如果单击Run，您将在输出中简短地注意到它输出了1、2、3，后面是无穷无尽的“DONE”流。

```
with-yield = λ(func) {
  λ() {
    CallCC(λ(kret){
      let (yield) {
        yield = λ(value) {
          CallCC(λ(kyld){
            func = kyld;
            kret(value);
          });
        };
        func(yield);
      };
    });
  };
};

foo = with-yield(λ(yield){
  yield(1);
  yield(2);
  yield(3);
  "DONE";
});

println(foo());
println(foo());
println(foo());
println(foo());

```
它循环的原因很微妙。 但是首先，如果您想看到更奇怪的东西，请编辑上面的代码，并将最后4行println更改为此：
```
println(foo());
foo();
```

输出将完全相同。 再次：如果运行此示例，则应刷新页面，否则页面将无休止地运行，从而耗尽电池，并最终耗尽计算机的所有RAM。

#### 问题1:yield 从不改变

需要注意的一个问题是，我们为第一个yield保存的continuation (kyld，如果遵循with-yield的实现，它将成为下一个要调用的func)嵌入了这个计算
```
yield(2);
  yield(3);
  "DONE";
```

但是，谁在这延续中屈服呢?它与第一个yield相同，后者实际上返回保存的第一个kret延续，即打印结果并从那里继续，从而产生一个循环。幸运的是，有一个简单的解决方法。很明显，我们应该只创建一次yield闭包，因为我们不能在第一次调用后在函数中更改它;为了返回到适当的退出点，我们维护了一个新的return变量
```
with-yield = λ(func) {
  let (return, yield) {
    yield = λ(value) {
      CallCC(λ(kyld){
        func = kyld;
        return(value);       # return is set below, on each call to the function
      });                    # it's always the next return point.
    };                       #
    λ(val) {                 #
      CallCC(λ(kret){        #
        return = kret;       # <- here
        func(val || yield);
      });
    };
  };
};

```

此外，认识到每次都向函数传递yield(只有第一次传递)没有意义，这个新版本允许在每次调用时传递另一个值(第一次调用除外)。这将是yield本身的返回值。

下面的代码仍然使用foo示例，但是它只使用了三次println(foo())
```
with-yield = λ(func) {
  let (return, yield) {
    yield = λ(value) {
      CallCC(λ(kyld){
        func = kyld;
        return(value);
      });
    };
    λ(val) {
      CallCC(λ(kret){
        return = kret;
        func(val || yield);
      });
    };
  };
};

foo = with-yield(λ(yield){
  yield(1);
  yield(2);
  yield(3);
  "DONE";
});

println(foo());
println(foo());
println(foo());
```

Result:
```
1
2
3
***Result: false
```

它似乎正常工作。 这是对第一个版本的明显改进，第一个版本循环了诸如print（foo（））之类的代码； foo（）。 但是，如果添加另一个println（foo（））会发生什么？ 循环！

#### 问题2：WTF？

这段时间循环的原因更为深刻。 这与我们延续的无限性质有关：它们对计算的整个未来进行编码，并且恰好包括从第一次调用返回到foo（）的过程。 当我们从那里回来时会发生什么？ -我们从头开始：

```
println(foo()); ## yield 1 <----------------- HERE -------------------+
println(foo()); ## yield 2                                            |
println(foo()); ## yield 3                                            |
println(foo()); ## we reached "DONE", so the first foo() RETURNS -->--+
```

Let's look at this line from with-yield:

```
func(val || yield);
        #...
```
当func通过调用yield退出时，它会调用continuation，所以执行不会到达#…线。但是当它完成时，为了到达函数的末尾(“DONE”行)，func将被有效地简化为一个函数，它将“DONE”返回到原始的延续，也就是调用第一个print。第二行上的foo()只是循环，但所有“DONE”行都由第一行打印。你可以运行下面的代码来验证这一点:

```
println(foo());
println("bar");
println(foo());
println(foo());
foo();
```

输出将是:1,bar, 2, 3, DONE, bar, DONE, bar， ....

因此，对于一个可能的修复，我们必须在func返回“natural”(简单地说，“完成”)时将其设置为其他值。我们将使它成为一个不做任何事情的函数，它只返回"no more continuations"。

```
val = func(val || yield);
func = λ() "NO MORE CONTINUATIONS";
kret(val);
```

现在不再循环，但是准备在运行以下代码时感到困惑：

```
with-yield = λ(func) {
  let (return, yield) {
    yield = λ(value) {
      CallCC(λ(kyld){
        func = kyld;
        return(value);
      });
    };
    λ(val) {
      CallCC(λ(kret){
        return = kret;
        val = func(val || yield);
        func = λ() "NO MORE CONTINUATIONS";
        kret(val);
      });
    };
  };
};

foo = with-yield(λ(yield){
  yield(1);
  yield(2);
  yield(3);
  "DONE";
});

println(foo());
println(foo());
println(foo());
println(foo());

```

我们本来希望1，2，3，DONE完成，但我们也三度获得“ NO MORE CONTINUATIONS”。 为了弄清楚会发生什么，我们可以交错打印一些图像，如下所示：
```
print("A. "); println(foo());
print("B. "); println(foo());
print("C. "); println(foo());
print("D. "); println(foo());

## and the output is:
A. 1
B. 2
C. 3
D. DONE
B. NO MORE CONTINUATIONS
C. NO MORE CONTINUATIONS
D. NO MORE CONTINUATIONS
***Result: false
```

这表明问题仍然存在：从函数的自然退出仍然可以回到第一个延续； 但是由于该函数现在是空操作，因此是“ B”。 行不会触发循环。

我们的实现仍然有用，只要该函数永不结束且仅通过yield退出即可。 这是斐波那契示例：
```
with-yield = λ(func) {
  let (return, yield) {
    yield = λ(value) {
      CallCC(λ(kyld){
        func = kyld;
        return(value);
      });
    };
    λ(val) {
      CallCC(λ(kret){
        return = kret;
        val = func(val || yield);
        func = λ() "NO MORE CONTINUATIONS";
        kret(val);
      });
    };
  };
};

fib = with-yield(λ(yield){
  let loop (a = 1, b = 1) {
    yield(b);
    loop(b, a + b);
  };
});

## print the first 50 fibonacci numbers
let loop (i = 0) {
  if i < 50 {
    println(fib());
    loop(i + 1);
  };
};
```

但是，如果我们想要一个可靠的实现（因此，当函数完成时不会返回到第一个调用站点），我们需要另一个概念，称为[“定界延续”](http://okmij.org/ftp/continuations/)。

##### 定界的延续：重置和移位 (Delimited continuations: reset and shift)

我们将通过另外两个运算符（重置和移位）来实现收益。 它们提供了“定界的延续”，就像普通的函数一样，它们是返回的延续。 重置会创建一个帧，而shift只会截取到该帧的延续，而不是像CallCC一样“截获程序的整个未来”。

重置和移位均采用单个函数参数。 在执行复位功能期间，调用shift可以使我们将值返回到调用复位的位置。

首先，让我们看看with-yield的样子：

```
with-yield = λ(func) {
  let (yield) {
    ## yield uses shift to return a value to the innermost
    ## reset call.  before doing that, it saves its own
    ## continuation in `func` — so calling func() again will
    ## resume from where it left off.
    yield = λ(val) {
      shift(λ(k){
        func = k;  # save the next continuation
        val;       # return the currently yielded value
      });
    };
    ## from with-yield we return a function with no arguments
    ## which uses reset to call the original function, passing
    ## to it the yield (initially) or any other value
    ## on subsequent calls
    λ(val) {
      reset( λ() func(val || yield) );
    };
  }
};

```

请注意，对函数的每次调用现在都嵌入在复位中。 这样可以确保我们不封装整个程序的延续，而只封装复位之前的那一部分。 根据定义，现在不可能循环。 当函数自然完成时，程序将继续执行而不是返回到第一次调用的位置。

您可以在下面运行我们麻烦的程序，然后您将看到它最终的运行情况符合预期。 它不会打印多于或少于它应有的内容，并且不会循环。
```
with-yield = λ(func) {
  let (yield) {
    yield = λ(val) {
      shift(λ(k){
        func = k;
        val;
      });
    };
    λ(val) {
      reset( λ() func(val || yield) );
    };
  }
};

foo = with-yield(λ(yield){
  yield(1);
  yield(2);
  yield(3);
  "DONE";
});

println(foo());  # prints 1
println(foo());  # prints 2
println(foo());  # prints 3
println(foo());  # prints DONE
```

Run Result:
```
1
2
3
DONE
***Result: false
```
#### 实现复位/移位 (Implementation of reset / shift)

尽管代码很短，但是这些运算符很难实现。 为头痛做准备。 我给你两个解决方案。 我不是该方法的作者，我可以告诉您，在单击该代码之前，我已经盯着很长时间了。 我在Oleg Kiselyov的这个Scheme程序中找到它。 我推荐这些文章以更多地了解这种令人振奋的概念。



As primitives

在本页中，它们被实现为原始函数。在原语中，当前的延续是显式的(我们不需要CallCC)，这使得理解幕后发生的事情更容易一些。下面是完整的实现(上)
```
var pstack = [];

function _goto(f) {
    f(function KGOTO(r){
        var h = pstack.pop();
        h(r);
    });
}

globalEnv.def("reset", function(KRESET, th){
    pstack.push(KRESET);
    _goto(th);
});

globalEnv.def("shift", function(KSHIFT, f){
    _goto(function(KGOTO){
        f(KGOTO, function SK(k1, v){
            pstack.push(k1);
            KSHIFT(v);
        });
    });
});
```
在我们使用了重置和移位的那一侧进行屈服可能会很有用

```
with-yield = λ(func) {
  let (yield) {
    yield = λ(val) {
      shift(λ(SK){
        func = SK;
        val;         ## return val
      });
    };
    λ(val) {
      reset( λ() func(val || yield) );
    };
  }
};
```

关键是重置和移位都使用_goto，这不是“正常”功能。 它没有自己的延续。 _goto接收一个正常的函数，并以继续符（KGOTO）调用它。 在执行f期间捕获的任何连续（甚至由CallCC捕获）都只能捕获到此KGOTO的“计算的未来”，因为上面没有任何内容。 因此，无论f是否正常退出，或通过调用延续，它最终都将运行KGOTO －从堆栈中获取下一个延续并将其应用于结果。

reset在_goto（th）之前将其自身的延续（KRESET）推入堆栈。 如果未执行此操作，则该函数退出时程序将停止，因为_goto之后未发生任何其他情况。 因此，当函数退出时，KGOTO将以任何方式恢复到该KRESET。

最后，shift以KGOTO连续性调用该函数，因此，如果正常退出，则KGOTO将从pstack的顶部继续，并将其传递给SK函数，该函数可以返回调用shift本身的位置（通过使用 班次本身的延续，KSHIFT）。 SK是一个定界的延续-它具有返回值的能力，就像一个函数一样-因此我们也需要将其k1延续推入堆栈。 为了说明这一点，我在此页中包含了shift2原语，该原语与上述移位类似，但没有以下行：pstack.push（k1）;。 试试这个例子：

```
println(reset(λ(){
    1 + shift( λ(k) k(k(2)) );
}));

println(reset(λ(){
    1 + shift2( λ(k) k(k(2)) );
}));
```

shift给我们一个延续（k），它是直到封闭复位为止的定界延续。 在这种情况下，继续操作是将移位结果加1：

当我们调用k时，我们将替换？。 上面有一个值。 我们称它两次k（k（2））。 我们提供2并继续执行程序，因此内部k（2）将返回3。因此，接下来的是k（3），它也提供3来代替问号，最后得到4。

shift2不正确：内部k（2）从不返回。

#### CallCC-based

如前所述，如果我们没有办法定义基本函数，而CallCC就是我们所拥有的，那么仍然有可能实现定界的延续。 下面的代码与原始版本几乎相同，但是由于我们没有JS数组，因此它使用列表来维护堆栈（就像我们看到的一样，尽管这里提供了列表，但是我们可以直接在λanguage中实现列表）。 作为原始元素）。 代码更长一些，因为我们需要那些CallCC来获取当前的延续（我们在原语中将其作为显式参数获得）

下面的代码中最棘手的部分是它如何实现goto，以及为什么必须这样做，但是我会很乐意弄清楚它。
```
pstack = NIL;

goto = false;

reset = λ(th) {
  CallCC(λ(k){
    pstack = cons(k, pstack);
    goto(th);
  });
};

shift = λ(f) {
  CallCC(λ(k){
    goto(λ(){
      f(λ(v){
        CallCC(λ(k1){
          pstack = cons(k1, pstack);
          k(v);
        });
      });
    });
  });
};

let (v = CallCC( λ(k){ goto = k; k(false) } )) {
  if v then let (r = v(), h = car(pstack)) {
    pstack = cdr(pstack);
    h(r);
  }
};
```