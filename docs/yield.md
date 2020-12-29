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
