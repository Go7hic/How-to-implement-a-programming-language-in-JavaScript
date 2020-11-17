## How fast are we?

The short answer is: dog slow!

But let's test. We'll use the double-recursive function for Fibonacci numbers, because it doesn't require much stack space yet the execution time grows exponentially as n grows. We'll compare the execution time of this function in plain JavaScript and in our λanguage.

I have included in this page the following primitives:
```
globalEnv.def("fibJS", function fibJS(n){
  if (n < 2) return n;
  return fibJS(n - 1) + fibJS(n - 2);
});

globalEnv.def("time", function(fn){
  var t1 = Date.now();
  var ret = fn();
  var t2 = Date.now();
  println("Time: " + (t2 - t1) + "ms");
  return ret;
});
```

time receives a function and prints out how much time was spent executing it. Let's run the following program:

```
fib = λ(n) if n < 2 then n else fib(n - 1) + fib(n - 2);


print("fib(10): ");
time( λ() println(fib(10)) );
print("fibJS(10): ");
time( λ() println(fibJS(10)) );
println("---");

print("fib(20): ");
time( λ() println(fib(20)) );
print("fibJS(20): ");
time( λ() println(fibJS(20)) );

println("---");

print("fib(27): ");
time( λ() println(fib(27)) );
print("fibJS(27): ");
time( λ() println(fibJS(27)) );

```
On my machine, using Google Chrome, for the last n (27) our λanguage works over one second, compared to 4 milliseconds (plain JavaScript). That is, of course, completely unacceptable.

We can and will compile λanguage to JavaScript. To make up for the limited recursion, we could add loop keywords (for / while); complex data structures like objects or arrays and even exceptions — all these be handled by the same features of the underlying language, JavaScript. But what would be the point? JS has its defects, right, but just slapping another syntax on top of it would not bring us any glory.

So I thought I'd give some more food for thought in this howto and we'll explore an evaluator in continuation-passing style, we'll play a bit with continuations and then a compiler to almost efficient JavaScript but which brings a language with some important semantic power that JavaScript doesn't have.

We're trying to build something better than our host language. Speed is important, but not the only goal

简短的答案是：贼慢！

但是，让我们测试一下。我们将对Fibonacci数使用双递归函数，因为它不需要太多的堆栈空间，但是执行时间随着n的增长而呈指数增长。我们将在普通JavaScript和λanguage中比较此函数的执行时间。

我在此页面中包含以下原始元素：

```js
globalEnv.def("fibJS", function fibJS(n){
  if (n < 2) return n;
  return fibJS(n - 1) + fibJS(n - 2);
});

globalEnv.def("time", function(fn){
  var t1 = Date.now();
  var ret = fn();
  var t2 = Date.now();
  println("Time: " + (t2 - t1) + "ms");
  return ret;
});
```

time接收一个函数并打印出执行该函数所花费的时间。让我们运行以下程序

```js
fib = λ(n) if n < 2 then n else fib(n - 1) + fib(n - 2);

print("fib(10): ");
time( λ() println(fib(10)) );
print("fibJS(10): ");
time( λ() println(fibJS(10)) );

println("---");

print("fib(20): ");
time( λ() println(fib(20)) );
print("fibJS(20): ");
time( λ() println(fibJS(20)) );

println("---");

print("fib(27): ");
time( λ() println(fib(27)) );
print("fibJS(27): ");
time( λ() println(fibJS(27)) );
```



```js
fib(10): 55
Time: 1ms
fibJS(10): 55
Time: 1ms
---
fib(20): 6765
Time: 34ms
fibJS(20): 6765
Time: 2ms
---
fib(27): 196418
Time: 387ms
fibJS(27): 196418
Time: 3ms
***Result: false
```

在我的机器上，使用Google Chrome浏览器，在最后n（27）内，λ语言的工作时间为一秒，而普通毫秒为4毫秒。那当然是完全不能接受的。



我们可以并将λ语言编译为JavaScript。为了弥补有限的递归，我们可以添加循环关键字（for / while）；复杂的数据结构，例如对象或数组，甚至是异常-所有这些都由基础语言JavaScript的相同功能处理。但是有什么意义呢？JS有其缺陷，是的，但是在它上面加上其他语法不会给我们带来任何荣耀。

因此，我想我将在本操作方法中提供更多思考的方法，我们将以延续传递样式探索一个评估器，我们将对延续进行一些尝试，然后再编译几乎有效的JavaScript，但它将语言带入JavaScript没有的一些重要的语义功能。

我们正在尝试构建比宿主语言更好的语言。速度很重要，但不是唯一的目标。