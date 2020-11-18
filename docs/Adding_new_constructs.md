## Adding new constructs

我们的语言太少了。例如，没有办法引入新变量。正如我所提到的，我们需要使用IIFE，所以它看起来像这样（愚蠢的示例，无法想到更好的方法）：

```js
(λ(x, y){
  (λ(z){ ## it gets worse when one of the vars depends on another
    print(x + y + z);
  })(x + y);
})(2, 3);
```

我们想添加一个let关键字，使我们可以这样写：

```js

let (x = 2, y = 3, z = x + y) print(x + y + z);
```

对于每个定义，都可以使用先前定义的变量。可以这样转换：

```js
(λ(x){
  (λ(y){
    (λ(z){
      print(x + y + z);
    })(x + y);
  })(3);
})(2);
```

该转换可以由解析器直接处理，并且不需要对评估器进行任何更改。无需添加专用的“ let”节点，我们可以将整个对象转换为“ call”和“ lambda”节点。这意味着我们没有在语言中添加任何语义功能-它被称为“语法糖”，而将其转换为已知的AST节点的操作被称为“ desugaring”。

但是，由于无论如何都要修改解析器，因此我们添加一个专用的“ let”节点，因为可以更有效地对其进行评估（无需创建闭包并立即调用它们；我们只需要扩展范围）。

我们还将另外支持“ named let”语法，该语法已被Scheme普及。它使我们能够以更简单的方式定义循环表达式：

```js

print(let loop (n = 10)
        if n > 0 then n + loop(n - 1)
                 else 0);
```

这是一个递归循环，可计算总和10 + 9 + ... +0。就目前的语言而言，我们必须这样编写：

```
print((λ(loop){
         loop = λ(n) if n > 0 then n + loop(n - 1)
                              else 0;
         loop(10);
       })());
```

为了简化操作，我们将添加“命名函数”的概念。所以我们也可以这样写：

```
print((λ loop (n) if n > 0 then n + loop(n - 1)
                           else 0)
      (10));
```

回顾一下，我们需要做的修改是：

- 在lambda关键字之后支持一个可选名称。如果存在，则函数运行的环境必须定义该名称以指向函数本身。就像JavaScript中的命名函数表达式一样。
- 支持新的let关键字。紧随其后的是一个可选名称和一个带括号的变量定义列表（可能为空），形式为foo = EXPRESSION，以逗号分隔。let的主体是单个表达式（显然可以是{group}）。

#### Parser changes

首先，在tokenizer中进行了非常小的更改，我们需要将let添加到关键字列表中：

```
var keywords = " let if then else lambda λ true false ";
```

我们必须在解析器中修改parse_lambda来考虑可选函数名称：

```
function parse_lambda() {
    return {
        type: "lambda",
        name: input.peek().type == "var" ? input.next().value : null, // this line
        vars: delimited("(", ")", ",", parse_varname),
        body: parse_expression()
    };
}
```

现在让我们定义let解析器：

```
function parse_let() {
    skip_kw("let");
    if (input.peek().type == "var") {
        var name = input.next().value;
        var defs = delimited("(", ")", ",", parse_vardef);
        return {
            type: "call",
            func: {
                type: "lambda",
                name: name,
                vars: defs.map(function(def){ return def.name }),
                body: parse_expression(),
            },
            args: defs.map(function(def){ return def.def || FALSE })
        };
    }
    return {
        type: "let",
        vars: delimited("(", ")", ",", parse_vardef),
        body: parse_expression(),
    };
}
```

它处理两种情况。如果跟随let，我们有一个“ var”令牌，那么它就是一个命名的let。我们读取带分隔符的定义，因为它们位于parens中并以逗号分隔，并使用下面定义的parse_vardef帮助器。然后，我们返回一个“调用”节点，该节点调用一个命名函数表达式（因此整个结果节点是IIFE）。函数的参数名称是let中定义的变量，“调用”将注意以args发送值。当然，该函数的主体是通过parse_expression（）获取的。

如果它不是具名的let，那么我们将返回一个具有vars和body的“ let”节点。var包含{通过以下函数从let定义列表中解析的{name：VARIABLE，def：AST}：

```
function parse_vardef() {
    var name = parse_varname(), def;
    if (is_op("=")) {
        input.next();
        def = parse_expression();
    }
    return { name: name, def: def };
}
```

我们还需要考虑在调度程序中遇到let关键字（parse_atom）的可能性，因此我们添加以下行：

```
// can be before the one handling parse_if
if (is_kw("let")) return parse_let();
```

#### Evaluator changes



由于我们选择修改AST而不是将其删除到已知的AST节点中，因此我们必须更新评估程序以将更改考虑在内。

为了支持可选的函数名称，make_lambda函数变为：

```
function make_lambda(env, exp) {
    if (exp.name) {                    // these
        env = env.extend();            // lines
        env.def(exp.name, lambda);     // are
    }                                  // new
    function lambda() {
        var names = exp.vars;
        var scope = env.extend();
        for (var i = 0; i < names.length; ++i)
            scope.def(names[i], i < arguments.length ? arguments[i] : false);
        return evaluate(exp.body, scope);
    }
    return lambda;
}
```

如果存在函数名称，那么我们在创建闭包时会扩展范围，并定义名称以指向新创建的闭包。其余的保持不变。

最后，为了处理“让” AST，我们在评估器中添加以下情况：

```
case "let":
  exp.vars.forEach(function(v){
      var scope = env.extend();
      scope.def(v.name, v.def ? evaluate(v.def, env) : false);
      env = scope;
  });
  return evaluate(exp.body, env);
```

请注意，它扩展了每个变量的范围，并通过评估初始化代码（如果有）来定义变量。然后评估一下let主体。

#### Test

```js
println(let loop (n = 100)
          if n > 0 then n + loop(n - 1)
                   else 0);

let (x = 2, y = x + 1, z = x + y)
  println(x + y + z);

# errors out, the vars are bound to the let body
# print(x + y + z);

let (x = 10) {
  let (x = x * 2, y = x * x) {
    println(x);  ## 20
    println(y);  ## 400
  };
  println(x);  ## 10
};
```

打印结果

```
5050
10
20
400
10
***Result: false
```



#### 总结

进行这些更改并不难，因为我们自己编写了整个语言。但是，如果它是由其他人编写的，而我们没有资源，那该怎么办？此外，如果这是每个人都已安装的标准语言，并且期望程序符合某种（刚性）语法？该语言是JavaScript。假设您想向JavaScript添加新的语法构造—可以吗？即使您可以向维护JS的大公司提出建议，也没有机会，您的建议要花几年的时间才能被考虑，接受，标准化和广泛实施。在JavaScript中，我们是囚犯（除非我们在其之上实现自己的语言）。

我告诉过您我将要争论为什么Lisp是一种很棒的语言：在Lisp中，我们可以添加语法结构而无需更改解析器/评估器/编译器的实际实现，从而使我们的生活更加轻松，而无需将这些结构强加给Java的所有用户语言，当然也无需等待数年时间让某个委员会批准我们的想法，而其他人则可以实施它。Lisp是自由。如果Lisp没有命名函数或let构造，我们可以用比上面需要的更少的代码添加它们，它们看起来和感觉都像内置构造。