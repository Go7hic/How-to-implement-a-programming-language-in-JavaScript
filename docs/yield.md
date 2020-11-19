# Implementing yield

First, let's see what yield is by looking at some usage examples

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

A more real-world usage:

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

This program will also run much faster than the duble-recursive version of fib, because it keeps track of the previous two values internally and just returns the next one.
