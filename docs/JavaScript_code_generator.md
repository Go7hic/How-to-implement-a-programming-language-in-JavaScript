# JavaScript code generator

从AST生成JS代码非常简单。 按照我们[第一个evaluator](http://lisperator.net/pltut/eval1/)的形状，make_js是一个接收AST的递归函数。 这次环境不是那么有用，因为我们没有执行AST。 相反，我们将其转换为等效的JavaScript代码，并以字符串形式返回。

入口点如下所示：
```
function make_js(exp) {
    return js(exp);

    function js(exp) {
        switch (exp.type) {
          case "num"    :
          case "str"    :
          case "bool"   : return js_atom   (exp);
          case "var"    : return js_var    (exp);
          case "binary" : return js_binary (exp);
          case "assign" : return js_assign (exp);
          case "let"    : return js_let    (exp);
          case "lambda" : return js_lambda (exp);
          case "if"     : return js_if     (exp);
          case "prog"   : return js_prog   (exp);
          case "call"   : return js_call   (exp);
          default:
            throw new Error("Dunno how to make_js for " + JSON.stringify(exp));
        }
    }

    // NOTE, all the functions below will be embedded here.
}
```

js函数根据当前节点的类型进行分派，并仅调用其他生成器之一。

“ atomic”值转换为适当的JS常量，而JSON.stringify使我们无需做任何工作：

```
function js_atom(exp) {
    return JSON.stringify(exp.value); // cheating ;-)
}
```

现在，我们将按原样输出变量名，这并不完全正确：在λ语言中，我们允许变量名包含减号，例如，在JS中无效。 为了便于以后进行修复，所有名称都通过make_var函数进行处理，因此我们以后可以在单个位置进行更改：
```
function make_var(name) {
    return name;
}
function js_var(exp) {
    return make_var(exp.value);
}
```

对于二元运算符，我们只需编译左右运算符，然后将它们与运算符结合在一起。 请注意，我们完全将所有内容括起来，而不必担心美化输出-为此，我们可以使用UglifyJS。
```
function js_binary(exp) {
    return "(" + js(exp.left) + exp.operator + js(exp.right) + ")";
}
```

与evaluator相比，细心的读者会注意到这里的一些错误：在评估器中，我们确保数值运算符可以接收适当类型的操作数，并且不除以零； 同样，&&和||的语义 （关于被认为是虚假的东西）应该与普通JS有所不同。 让我们稍后再担心。
```
// assign nodes are compiled the same as binary
function js_assign(exp) {
    return js_binary(exp);
}
```

 lambda”节点也非常简单。 只需输出一个JavaScript函数表达式。 为了避免任何可能的错误，我们将括号括起来，输出function关键字，后跟名称（如果存在），然后输出参数列表。 主体是单个表达式，我们返回其结果。
```
 function js_lambda(exp) {
    var code = "(function ";
    if (exp.name)
        code += make_var(exp.name);
    code += "(" + exp.vars.map(make_var).join(", ") + ") {";
    code += "return " + js(exp.body) + " })";
    return code;
}
```

“ let”节点由IIFE-s处理。 这太过分了，我们可以做得更好，但是我们现在不会打扰。 使用IIFE处理节点非常简单：如果vars为空，则只需编译主体。 否则，为第一个变量/定义创建IIFE节点（这是一个“调用”节点）（如果缺少定义，则传递FALSE节点）-最后，编译IIFE节点。 这就是所谓的“减糖”。
```
function js_let(exp) {
    if (exp.vars.length == 0)
        return js(exp.body);
    var iife = {
        type: "call",
        func: {
            type: "lambda",
            vars: [ exp.vars[0].name ],
            body: {
                type: "let",
                vars: exp.vars.slice(1),
                body: exp.body
            }
        },
        args: [ exp.vars[0].def || FALSE ]
    };
    return "(" + js(iife) + ")";
}
```

要编译“ if”节点，我们使用JavaScript的三元运算符。 我们坚持以下事实：该语言中唯一的虚假值是false（因此，如果条件的结果不是false，则仅执行“ then”节点）。 如果缺少“ else”节点，则只需编译FALSE。

不像求值器只探测“then”或“else”，这里我们必须递归到两个分支来生成代码，因为此时我们无法知道条件的值。
```
function js_if(exp) {
    return "("
        +      js(exp.cond) + " !== false"
        +      " ? " + js(exp.then)
        +      " : " + js(exp.else || FALSE)
        +  ")";
}
```

最后，“编”和“调用”-非常琐碎的节点。 请注意，对于“程序”，我们使用JavaScript的排序运算符（逗号），以便获得单个表达式。 幸运的是，逗号具有相似的语义（它返回最后一个表达式的值），使其非常适合我们的需求。

```
function js_prog(exp) {
    return "(" + exp.prog.map(js).join(", ") + ")";
}
function js_call(exp) {
    return js(exp.func) + "(" + exp.args.map(js).join(", ") + ")";
}
```

#### Primitives and usage 原语和用法

这次生成的程序实际上使用普通的JavaScript变量，因此基元将仅仅是全局JS变量或函数。 这是一个基本用法示例：
```
// the "print" primitive
window.print = function(txt) {
  console.log(txt);
};

// some test code here
var code = "sum = lambda(x, y) x + y; print(sum(2, 3));";

// get the AST
var ast = parse(TokenStream(InputStream(code)));

// get JS code
var code = make_js(ast);

// additionally, if you want to see the beautified JS code using UglifyJS
// (or possibly other tools such as acorn/esprima + escodegen):
console.log( UglifyJS.parse(code).print_to_string({ beautify: true }) );

// execute it
eval(code); // prints 5
```

您实际上可以在下面看到它的运行情况。

#### Test
还记得我们的评估器的[第一个版本](http://lisperator.net/pltut/eval1/speed)（到目前为止，我们最快的）是如何在计算fib（27）上胜过第二个的吗？ （这比手写JS慢300倍）。 这是编译为JS之后的代码：
```
fib = λ(n) if n < 2 then n else fib(n - 1) + fib(n - 2);
time( λ() println(fib(27)) );

```

Run result:
```
196418
Time: 5ms
***Result: false
```

它和手写JS代码一样快。 当您在浏览器的console.log中单击“运行”时，可以看到生成的代码。

可能会很想在这里停止-它“与JS一样快”。 但是它也“和JS一样好”，这还不够好。 没有延续，递归再次受到限制。 我们在这里所做的充其量可以称为“翻译器”。

编译器将源语言转换为目标语言。 例如，将C ++转换为汇编程序。 或JavaScript语言。 编译器的真正工作不是生成代码，而是支持使源语言比目标语言更强大的功能。