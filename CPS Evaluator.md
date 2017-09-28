> Update: There is a bug which slipped in all of interpreter, CPS evaluator and CPS compiler: expressions like a() && b() or a() || b() always evaluate both sides. This is wrong, of course, whether or not b() is evaluated depends on the falsy value of a(). The fix is pretty simple (convert || and && binary operations to "if" nodes) but I didn't update these pages yet. Shame on me

## CPS Evaluator

Our language has two problems: (1) recursion is limited by the JS stack, so there is no effective way to do iteration; and (2) it's slow. We'll solve the first problem here, unfortunately at the cost of making it even slower. We're going to rewrite our evaluator in “continuation-passing style” (CPS).

#### What is CPS

You do it in NodeJS all the time. For example:

```
fs.readFile("file.txt", "utf8", function CC(error, data){
  // this callback is the "continuation"
  // instead of returning a value by using `return`,
  // readFile will invoke our callback with the data.
});
```

At each step of the way, there's some callback you need to invoke in order to continue. Continuation-passing style makes control flow “explicit” — you don't return, you don't throw, you don't break or continue. No arbitrary jumps. You can't even use for or while loops with asynchronous functions. If that's the case, why would we even have all those keywords in the language?

Manually writing programs in CPS is unintuitive and error-prone, but we'll have to rewrite our evaluator this way.

### The evaluate function

The CPS evaluate will receive three arguments: the expression (AST), the environment, and a callback to invoke with the result. Here's the code and I'll comment on each case, as before:

```
function evaluate(exp, env, callback) {
    switch (exp.type) {
```
For constants, we just need to return their value. But remember, there's no return—instead we're invoking the callback with the value.

```
case "num":
      case "str":
      case "bool":
        callback(exp.value);
        return;
```

"var" is also simple: fetch the variable from the environment, pass it to the callback.

```
case "var":
        callback(env.get(exp.value));
        return;
```

For "assign" nodes we need to evaluate the "right" expression first. For this we're calling evaluate, passing a callback that will get the result (as right). And then we just invoke our original callback with the result of setting the variable (which will be the value, in fact).

```
      case "assign":
        if (exp.left.type != "var")
            throw new Error("Cannot assign to " + JSON.stringify(exp.left));
        evaluate(exp.right, env, function(right){
            callback(env.set(exp.left.value, right));
        });
        return;
```

Similarly, for "binary" nodes we need to evaluate the "left" node, then the "right" node, and then invoke the callback with the result of applying the operator. Same as before, we call evaluate recursively and pass a callback that carries on the next steps.

```
  case "binary":
        evaluate(exp.left, env, function(left){
            evaluate(exp.right, env, function(right){
                callback(apply_op(exp.operator, left, right));
            });
        });
        return;
```

"let" looks a little more complicated, but it's very simple. We have a number of variable definitions. Their "def" (initial value) can be missing, in which case we make them false by default; but when the value is present, we need to call evaluate recursively in order to compute it.

If you worked in NodeJS you might have solved similar problems many times already. Because of the callback, we can't use a straight for, so we need to compute those expressions one by one (imagine the evaluate function might not return immediately, but asynchronously). The loop function below (immediately invoked) receives an environment and the index of the current definition to compute.

   - If that index is equal to vars.length that means we finished and the environment has all the defs, hence we evaluate(exp.body, env, callback). Note that this time we're not invoking the callback ourselves, but just pass it to evaluate as the next thing to do after running exp.body.

   - If the index is smaller then evaluate the current definition and pass a callback that will loop(scope, i + 1), after extending the environment with the definition that we just computed.
   
   ```
    case "let":
        (function loop(env, i){
            if (i < exp.vars.length) {
                var v = exp.vars[i];
                if (v.def) evaluate(v.def, env, function(value){
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
     ```
     Again, we'll handle the "lambda" in a separate function that I'll describe below.
     
     ```
     case "lambda":
        callback(make_lambda(env, exp));
        return;
    ```
    For executing an "if", we evaluate the condition. If it's not false then evaluate the "then" branch, otherwise evaluate the "else" branch if it's present, otherwise pass false to the callback. Again note that for "then"/"else" we don't have to run the callback ourselves, but just pass it to evaluate as the “next thing to do” after computing those expressions.
    
    ```
          case "if":
        evaluate(exp.cond, env, function(cond){
            if (cond !== false) evaluate(exp.then, env, callback);
            else if (exp.else) evaluate(exp.else, env, callback);
            else callback(false);
        });
        return;
    ```
    A "prog" node is handled somewhat similar to "let", but it's simpler because it doesn't need to extend scope and define variables. Any case, the same general pattern: we have a loop function which handles the expression number i. When i is equal to prog.length then we're done, so just return the value that the last expression evaluated to (and by “return” I mean, of course, invoke the callback with it). Note we're keeping track of the last value by passing it as argument to loop (initially false, in case the prog body is empty).
    
    ```
    case "prog":
        (function loop(last, i){
            if (i < exp.prog.length) evaluate(exp.prog[i], env, function(val){
                loop(val, i + 1);
            }); else {
                callback(last);
            }
        })(false, 0);
        return;
    ```
  
For a "call" node we need to evaluate "func" and then evaluate the arguments, in order. Again, a loop function handles them similarly as we needed to do in "let" and "prog", only this time it builds an array with the results. The first value in that array must be the callback, because closures returned by make_lambda will also be in continuation-passing style, thus instead of using return they will invoke the callback with the result.

```
 case "call":
        evaluate(exp.func, env, function(func){
            (function loop(args, i){
                if (i < exp.args.length) evaluate(exp.args[i], env, function(arg){
                    args[i + 1] = arg;
                    loop(args, i + 1);
                }); else {
                    func.apply(null, args);
                }
            })([ callback ], 0);
        });
        return;
 ```
 
Same epilogue as before. If we don't know what to do, throw an error.

```
      default:
        throw new Error("I don't know how to evaluate " + exp.type);
    }
}
```


You can notice that each case above ends with a return. There's no return value though; the result is always passed to the callback. If anything, we would wish that execution never reaches back to the case; but it's how JS works. We need those return statements in order to not fall down to the next case.

#### The new make_lambda

In this version of our evaluator, all functions will receive as first argument the “continuation” — a callback to invoke with the result. Following it are the other arguments as passed at run-time, but this one is always inserted by the evaluator. Here's the code for the new make_lambda:

```
function make_lambda(env, exp) {
    if (exp.name) {
        env = env.extend();
        env.def(exp.name, lambda);
    }
    function lambda(callback) {
        var names = exp.vars;
        var scope = env.extend();
        for (var i = 0; i < names.length; ++i)
            scope.def(names[i],
                      i + 1 < arguments.length
                        ? arguments[i + 1]
                        : false);
        evaluate(exp.body, scope, callback);
    }
    return lambda;
}
```
It's only slightly modified. It extends the scope with the new variable bindings (for the arguments). It must account for that “callback” argument as being the first one (hence, i + 1 when wondering in the arguments). And finally it uses evaluate to run the function body in the new scope, as before, but passing the callback to the evaluator instead of returning the result.

Incidentally, notice that this is the only place where I used an iterative loop (for). That's because the arguments are already computed when the "call" node was evaluated—we need not call the evaluator for them and pass a callback to get the value of each argument.

### Primitives

For the CPS evaluator, primitive functions will receive a callback as first argument. We must keep this in mind while defining primitives. Here's some simple test code for the new version:

```
var code = "sum = lambda(x, y) x + y; print(sum(2, 3));";
var ast = parse(TokenStream(InputStream(code)));
var globalEnv = new Environment();

// define the "print" primitive function
globalEnv.def("print", function(callback, txt){
  console.log(txt);
  callback(false); // call the continuation with some return value
                   // if we don't call it, the program would stop
                   // abruptly after a print!
});

// run the evaluator
evaluate(ast, globalEnv, function(result){
  // the result of the entire program is now in "result"
});
```

### Small test
Let's play with fib again:

```
fib = λ(n) if n < 2 then n else fib(n - 1) + fib(n - 2);
time( λ() println(fib(10)) );

```
ut if we increase that to fib(27), we get a “stack overflow” error. Actually, the stack grows much faster now, so it turns out that this evaluator can only compute up to fib(12) (at least in my browser.. yours might vary). That's disappointing, but in the next page I'll show you how we can workaround it.
