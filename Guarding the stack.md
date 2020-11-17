## 保护堆栈

在CPS评估程序中，堆栈被更快地耗尽，因为评估程序一直不断调用其他函数，并且永不返回。不要被return语句欺骗-它们是必需的，但是在非常深的递归情况下，它们永远不会到达，因为相反，我们会得到堆栈溢出错误。

让我们想象一下堆栈如何寻找一个非常简单的程序。我将显示伪代码，但我不包含env，因为这对于指出这一点并不重要

```js
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

只有在最后一次连续运行（K0）之后，一长串无意义的返回-s后退堆栈。如果即使对于一个琐碎的程序我们也嵌套了很多东西，那么很容易想象fib（13）是如何耗尽堆栈空间的。



#### 堆栈守卫

我们编写新评估程序的方式，栈简直就是垃圾。在每个步骤之后需要进行的所有计算都包含在回调中，该回调作为参数传递。因此，这就引出了一个问题：如果JavaScript提供某种方式来重置堆栈该怎么办？然后，我们可以不时地这样做，就像某种垃圾回收一样，深度递归也将起作用。

假设我们有一个可以做到这一点的GUARD函数。它接收两个值：要调用的函数和要传递给它的参数数组。它检查堆栈是否嵌套过多，如果嵌套过多，它将重置堆栈并在此后调用该函数。否则它什么都不做。

使用此函数，我们将按如下方式重写评估程序。我不会在每种情况下发表评论，因为它与以前的代码相同。唯一的补充是在执行其他任何操作之前，请确保对每个功能进行保护。

```js
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

对于匿名函数，我必须声明一个名称才能引用它们。我使用CC（代表“当前延续”）。一个替代方法是arguments.callee，但不要使用不推荐使用的API。



同样，make_lambda中有类似的单行更改：

```js
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

GUARD的实现非常简单。如何从深层嵌套呼叫突然退出？-使用例外。因此，我们将为堆栈嵌套限制维护一个全局变量，当它太深时，抛出该异常。我们抛出一个Continuation对象，该对象保存了计算的未来—一个要调用的函数及其参数：

```js
var STACKLEN;
function GUARD(f, args) {
    if (--STACKLEN < 0) throw new Continuation(f, args);
}
function Continuation(f, args) {
    this.f = f;
    this.args = args;
}
```

最后，我们需要设置一个循环来捕获Continuation对象。为了使整个技巧生效，我们将必须通过该循环运行评估程序。

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

Execute需要一个函数来运行，并传递参数。它会循环执行此操作，但要注意返回值—如果函数运行时没有破坏堆栈，我们将在此处停止。每次恢复循环时，都会初始化STACKLEN。我发现200物有所值。捕获到Continuation后，请恢复新函数和参数并继续循环。此时堆栈会被异常清除，因此我们可以再次嵌套.

要现在使用Execute运行评估程序，我们需要执行以下操作：

```
Execute(evaluate, [ ast, globalEnv, function(result){
    console.log("*** Result:", result);
}]);
```

#### 测试

我们的fib函数不会再失败了：

```js
fib = λ(n) if n < 2 then n else fib(n - 1) + fib(n - 2);
time( λ() println(fib(20)) );

```

Run result:

```
6765
Time: 439ms
***Result: false

```



不幸的是，如果您尝试使用fib（27），执行时间将比评估程序的第一个（非CPS）版本慢大约4倍。但是至少我们有无限的递归，例如：

```js
sum = λ(n, ret)
        if n == 0 then ret
                  else sum(n - 1, ret + n);

# compute 1 + 2 + ... + 50000
time( λ() println(sum(50000, 0)) );
```

Run result:

```
1250025000
Time: 715ms
***Result: false
```

我们的语言确实比JavaScript慢得多！试想一下，每个变量查找都必须通过我们的Environment对象。试图优化解释器是没有意义的，我们不会走得太远。提高速度的解决方案是将λanguage编译为本地JS，这就是我们要做的。但是首先让我们看看使用CPS评估程序会产生一些有趣的结果。