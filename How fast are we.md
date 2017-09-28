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

