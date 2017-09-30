## Guarding the stack

In the CPS evaluator the stack is exhausted much quicker because the evaluator ever keeps calling other functions, and never returns. Do not be tricked by the return statements—they are necessary, but in the case of a very deep recursion they are never reached, because we get a stack overflow error instead.

Let's imagine how the stack looks for a very simple program. I'll show pseudo-code and I'm not including the env as it's not important for making this point

```
print(1 + 2 * 3);

## stack:
evaluate( print(1 + 2 * 3), K0 )
  evaluate( print, K1 )
    K1(print)  # it's a var, we fetch it from the environment
      evaluate( 1 + 2 * 3, K2 )
        evaluate( 2 * 3, K3 )
          evaluate( 2, K4 )
            K4(2)  # 2 is constant so call the continuation with it
              evaluate( 3, K5 )  # same for 3, but we're still nesting
                K5(3)
                  K3(6)  # the result of 2*3
                    evaluate( 1, K6 )  # again, constant
                      K6(1)
                        K2(7)  # the result of 1+2*3
                          print(K0, 7)  # finally get to call print
                            K0(false)  # original continuation. print returns false.
```

Only after the last continuation runs (K0) will a long sequence of pointless return-s rewind the stack. If we nest so much even for a trivial program, it's easy to imagine how fib(13) runs out of stack space.

### A stack guard
The way we wrote the new evaluator, the stack is simply garbage. All the computation that needs to happen after each step is enclosed in the callback, which is passed as an argument. So this begs the question: what if JavaScript would provide some way to reset the stack? Then we could do it every now and then, like some kind of garbage collection, and deep recursion will work.

Let's assume we have a GUARD function which can do that. It receives two values: a function to call and an array of arguments to pass to it. It checks if the stack is too nested, and if so, it'll reset the stack and call that function thereafter. Otherwise it does nothing.

Using this function we'll rewrite our evaluator as follows. I will not comment on each case because it's the same code as before; the only addition is GUARD-ing every single function, before doing anything else.

```
function evaluate(exp, env, callback) {
    GUARD(evaluate, arguments);
    switch (exp.type) {
      case "num":
      case "str":
      case "bool":
        callback(exp.value);
        return;

      case "var":
        callback(env.get(exp.value));
        return;

      case "assign":
        if (exp.left.type != "var")
            throw new Error("Cannot assign to " + JSON.stringify(exp.left));
        evaluate(exp.right, env, function CC(right){
            GUARD(CC, arguments);
            callback(env.set(exp.left.value, right));
        });
        return;

      case "binary":
        evaluate(exp.left, env, function CC(left){
            GUARD(CC, arguments);
            evaluate(exp.right, env, function CC(right){
                GUARD(CC, arguments);
                callback(apply_op(exp.operator, left, right));
            });
        });
        return;

      case "let":
        (function loop(env, i){
            GUARD(loop, arguments);
            if (i < exp.vars.length) {
                var v = exp.vars[i];
                if (v.def) evaluate(v.def, env, function CC(value){
                    GUARD(CC, arguments);
                    var scope = env.extend();
                    scope.def(v.name, value);
                    loop(scope, i + 1);
                }); else {
                    var scope = env.extend();
                    scope.def(v.name, false);
                    loop(scope, i + 1);
                }
            } else {
                evaluate(exp.body, env, callback);
            }
        })(env, 0);
        return;

      case "lambda":
        callback(make_lambda(env, exp));
        return;

      case "if":
        evaluate(exp.cond, env, function CC(cond){
            GUARD(CC, arguments);
            if (cond !== false) evaluate(exp.then, env, callback);
            else if (exp.else) evaluate(exp.else, env, callback);
            else callback(false);
        });
        return;

      case "prog":
        (function loop(last, i){
            GUARD(loop, arguments);
            if (i < exp.prog.length) evaluate(exp.prog[i], env, function CC(val){
                GUARD(CC, arguments);
                loop(val, i + 1);
            }); else {
                callback(last);
            }
        })(false, 0);
        return;

      case "call":
        evaluate(exp.func, env, function CC(func){
            GUARD(CC, arguments);
            (function loop(args, i){
                GUARD(loop, arguments);
                if (i < exp.args.length) evaluate(exp.args[i], env, function CC(arg){
                    GUARD(CC, arguments);
                    args[i + 1] = arg;
                    loop(args, i + 1);
                }); else {
                    func.apply(null, args);
                }
            })([ callback ], 0);
        });
        return;

      default:
        throw new Error("I don't know how to evaluate " + exp.type);
    }
}
```
For anonymous functions I had to declare a name in order to refer to them. I used CC (stands for “current continuation”). An alternative would be arguments.callee but let's not use deprecated API.

Also, a similar one-line change in make_lambda:

```
function make_lambda(env, exp) {
    if (exp.name) {
        env = env.extend();
        env.def(exp.name, lambda);
    }
    function lambda(callback) {
        GUARD(lambda, arguments);  // <-- this
        var names = exp.vars;
        var scope = env.extend();
        for (var i = 0; i < names.length; ++i)
            scope.def(names[i], i + 1 < arguments.length ? arguments[i + 1] : false);
        evaluate(exp.body, scope, callback);
    }
    return lambda;
}

```
The implementation of GUARD is really simple. How do you exit abruptly from e deeply nested call? — using exceptions. So we'll maintain a global variable for the stack nesting limit and when it's too deep, throw. We throw a Continuation object which holds the future of the computation — a function to call and its arguments:

```
var STACKLEN;
function GUARD(f, args) {
    if (--STACKLEN < 0) throw new Continuation(f, args);
}
function Continuation(f, args) {
    this.f = f;
    this.args = args;
}
```
Finally, we need to setup a loop that will catch Continuation objects. We'll have to run our evaluator through that loop in order for the whole trick to work.

```
function Execute(f, args) {
    while (true) try {
        STACKLEN = 200;
        return f.apply(null, args);
    } catch(ex) {
        if (ex instanceof Continuation)
            f = ex.f, args = ex.args;
        else throw ex;
    }
}
```

Execute takes a function to run and arguments to pass it. It does that in a loop, but note the return—in the event the function runs without blowing the stack, we stop there. STACKLEN is initialized every time we resume the loop. I found 200 to be a good value. When a Continuation is caught, reinstate the new function and arguments and continue the loop. The stack is cleared at this point by the exception, so we can nest again.

To run our evaluator now with Execute, we do something like this:

```
Execute(evaluate, [ ast, globalEnv, function(result){
    console.log("*** Result:", result);
}]);
```

### Test

Our fib function won't fail anymore:

```
fib = λ(n) if n < 2 then n else fib(n - 1) + fib(n - 2);
time( λ() println(fib(20)) );
```
Unfortunately, if you try fib(27) the execution time will be about about 4 times slower than with the first (non-CPS) version of the evaluator. But at least we have unlimited recursion, for example:

```
sum = λ(n, ret)
        if n == 0 then ret
                  else sum(n - 1, ret + n);

# compute 1 + 2 + ... + 50000
time( λ() println(sum(50000, 0)) );

```
Our language is much slower than JavaScript indeed! Just imagine that every variable lookup has to go through our Environment object. Trying to optimize the interpreter is pointless—we won't get far. The solution for better speed is to compile λanguage to native JS, and that's what we'll do. But first let's see some interesting consequences of having a CPS evaluator.
