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

我们的语言有两个问题：（1）递归受JS堆栈的限制，因此没有有效的方法来进行迭代；（2）它很慢。不幸的是，我们将在这里解决第一个问题，但要以使其变慢为代价。我们将以“连续传递风格”（CPS）重写评估程序。



#### 什么是CPS

您始终可以在NodeJS中进行操作。例如：

```
fs.readFile("file.txt", "utf8", function CC(error, data){
  // this callback is the "continuation"
  // instead of returning a value by using `return`,
  // readFile will invoke our callback with the data.
});
```

在此过程的每个步骤中，都需要调用一些回调才能继续。连续传递样式使控制流“显式”-您不返回，不抛出，不中断或继续。没有任意跳跃。您甚至不能使用带有异步功能的for或while循环。如果是这样，我们为什么还要在语言中使用所有这些关键字？



用CPS手动编写程序是不直观且容易出错的，但是我们必须以这种方式重写评估程序。

#### The `evaluate` function

CPS评估将接收三个参数：表达式（AST），环境和用于调用结果的回调。这是代码，我将像以前一样对每种情况进行评论：

```
function evaluate(exp, env, callback) {
    switch (exp.type) {
```

对于常量，我们只需要返回它们的值即可。但是请记住，没有返回值，而是使用值调用回调。

```
case "num":
      case "str":
      case "bool":
        callback(exp.value);
        return;
```

“ var”也很简单：从环境中获取变量，并将其传递给回调。

```
 case "var":
        callback(env.get(exp.value));
        return;
```

对于“分配”节点，我们需要首先评估“正确”表达式。为此，我们调用评估，传递一个将获取结果的回调（正确）。然后，我们只需使用设置变量的结果（实际上就是值）来调用原始回调。

```
 case "assign":
        if (exp.left.type != "var")
            throw new Error("Cannot assign to " + JSON.stringify(exp.left));
        evaluate(exp.right, env, function(right){
            callback(env.set(exp.left.value, right));
        });
        return;
```

同样，对于“二进制”节点，我们需要先评估“左”节点，然后评估“右”节点，然后调用回调，并应用运算符。与之前一样，我们递归调用评估，并传递一个进行下一步的回调。

```
 case "binary":
        evaluate(exp.left, env, function(left){
            evaluate(exp.right, env, function(right){
                callback(apply_op(exp.operator, left, right));
            });
        });
        return;
```

“ let”看起来稍微复杂一些，但是非常简单。我们有许多变量定义。它们的“ def”（初始值）可能会丢失，在这种情况下，我们默认将它们设置为false。但是当存在该值时，我们需要递归调用评估以便计算它。



如果您在NodeJS中工作，您可能已经解决了很多类似的问题。由于存在回调，因此不能使用直截了当，因此我们需要一个一个地计算这些表达式（假设评估函数可能不会立即返回，而是异步返回）。下面的循环函数（立即调用）接收环境和当前定义的索引以进行计算。



- 如果该索引等于vars.length，则意味着我们完成并且环境具有所有def，因此我们进行评估（exp.body，env，callback）。请注意，这一次我们不是自己调用回调，而是在运行exp.body之后将其传递给评估作为下一步操作。

- 如果索引较小，则在使用我们刚刚计算的定义扩展环境之后，评估当前定义并传递一个循环（范围，i + 1）的回调。

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

同样，我们将在下面将要描述的单独函数中处理“ lambda”。

```
 case "lambda":
        callback(make_lambda(env, exp));
        return;
```

为了执行“ if”，我们评估条件。如果不是false，则评估“ then”分支，否则评估“ else”分支（如果存在），否则将false传递给回调。再次注意，对于“ then” /“ else”，我们不必自己运行回调，而只需在计算这些表达式后将其传递为“下一步要做的事”即可。

```
case "if":
        evaluate(exp.cond, env, function(cond){
            if (cond !== false) evaluate(exp.then, env, callback);
            else if (exp.else) evaluate(exp.else, env, callback);
            else callback(false);
        });
        return;
```

“ prog”节点的处理与“ let”有点类似，但是它更简单，因为它不需要扩展范围和定义变量。在任何情况下，都是相同的一般模式：我们有一个循环函数来处理表达式编号i。当i等于prog.length时，我们就完成了，所以只需返回最后一个表达式求值的值即可（当然，通过“返回”，我的意思是用它调用回调）。请注意，我们通过将最后一个值作为参数传递给loop来跟踪最后一个值（如果prog主体为空，则最初为false）。

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

对于“呼叫”节点，我们需要先评估“ func”，然后再评估参数。同样，循环函数处理它们的方式与我们在“ let”和“ prog”中所需的类似，只是这一次它使用结果构建一个数组。该数组中的第一个值必须是回调，因为make_lambda返回的闭包也将采用连续传递样式，因此，它们将使用结果调用回调，而不是使用return。

```js
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

与以前相同的结尾。如果我们不知道该怎么办，则抛出错误。

```js
default:
        throw new Error("I don't know how to evaluate " + exp.type);
    }
}
```

您会注意到上面的每种情况都以返回结尾。但是没有返回值。结果总是传递给回调。如果有的话，我们希望执行永远不会回到案例中来。但这就是JS的工作方式。我们需要那些return语句，以免发生下一种情况。



##### 新的make_lambda

在此版本的评估程序中，所有函数均将第一个参数“ continuation”（连续性）作为接收参数，并使用结果进行回调。在运行时传递的其他参数是紧随其后的，但此参数始终由评估者插入。这是新的make_lambda的代码：

```js
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

它只是稍作修改。它使用新的变量绑定（用于参数）扩展了范围。它必须将“回调”参数视为第一个参数（因此，在参数中进行查询时，i + 1）。最后，它像以前一样使用评估在新作用域中运行函数主体，但是将回调传递给评估器而不是返回结果。

顺便说一句，请注意，这是我使用迭代循环（for）的唯一地方。这是因为在评估“调用”节点时已经计算了参数-我们无需为其调用评估器，而传递回调以获取每个参数的值。



#### Primitives

对于CPS评估程序，原始函数将接收回调作为第一个参数。在定义基元时，我们必须牢记这一点。这是新版本的一些简单测试代码:

```js
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

#### Small test

让我们再次玩fib：

```
fib = λ(n) if n < 2 then n else fib(n - 1) + fib(n - 2);
time( λ() println(fib(10)) );

```

Run result:

```
55
Time: 5ms
***Result: false
```

但是，如果将其增加到fib（27），则会出现“堆栈溢出”错误。实际上，现在堆栈的增长速度要快得多，因此事实证明，该评估器最多只能计算fib（12）（至少在我的浏览器中。您的浏览器可能有所不同）。令人失望，但是在下一页中，我将向您展示我们如何解决该问题。