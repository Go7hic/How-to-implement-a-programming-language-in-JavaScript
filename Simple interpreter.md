## 一个简单的解释器

> 更新：在所有解释器，CPS评估程序和CPS编译器中都有一个错误：a（）&& b（）或a（）||之类的表达式b（）总是评估双方。当然，这是错误的，是否评估b（）取决于a（）的虚假值。修复非常简单（将||和&&二进制操作转换为“ if”节点），但是我尚未更新这些页面。真可惜
>
>

回顾一下：到目前为止，我们编写了3个函数：InputStream，TokenStream和 parse。要从一段代码中获取AST，我们可以执行以下操作：

```js
var ast = parse(TokenStream(InputStream(code)));
```

编写解释器比解析器容易。我们只需要遍历AST，以其正常顺序执行表达式即可。



#### 环境

正确执行的关键是正确维护环境-一个具有变量绑定的结构。它将作为参数传递给我们的评估函数。每次我们输入“ lambda”节点时，都必须使用新变量（函数的参数）扩展环境，并使用在运行时传递的值对其进行初始化。如果自变量在外部作用域中遮盖了变量（在这里我将交替使用“作用域”和“环境”一词），则在离开函数时，必须小心恢复先前的值。

实现此目的的最简单方法是使用JavaScript的原型继承。当我们输入一个函数时，我们将创建一个新环境，将其原型设置为外部（父）环境，并在新环境中评估函数主体。这样，当我们退出时，我们无需执行任何操作-外部环境将已经包含任何阴影绑定。

这是环境对象的定义：

```js
function Environment(parent) {
    this.vars = Object.create(parent ? parent.vars : null);
    this.parent = parent;
}
Environment.prototype = {
    extend: function() {
        return new Environment(this);
    },
    lookup: function(name) {
        var scope = this;
        while (scope) {
            if (Object.prototype.hasOwnProperty.call(scope.vars, name))
                return scope;
            scope = scope.parent;
        }
    },
    get: function(name) {
        if (name in this.vars)
            return this.vars[name];
        throw new Error("Undefined variable " + name);
    },
    set: function(name, value) {
        var scope = this.lookup(name);
        // let's not allow defining globals from a nested environment
        if (!scope && this.parent)
            throw new Error("Undefined variable " + name);
        return (scope || this).vars[name] = value;
    },
    def: function(name, value) {
        return this.vars[name] = value;
    }
};
```

环境对象有一个父对象，它指向父范围。对于全局范围，父级将为null。并且它具有vars属性，其中包含变量绑定。对于顶级（全局）作用域，它初始化为Object.create（null）；对于子作用域，它初始化为Object.create（parent.vars），以便通过原型继承“查看”当前绑定。

有以下几种方法：

- extend（）—创建一个子范围。
- lookup（name）—查找定义具有给定名称的变量的范围。
- get（name）—获取变量的当前值。如果未定义变量，则会引发错误。
- set（name，value）—设置变量的值。这需要查找定义变量的实际范围。如果未找到并且我们不在全局范围内，则会引发错误。
- def（name，value）-这会在当前范围内创建（或阴影或覆盖）变量

#### The `evaluate` function

现在我们有了环境，我们可以跳到主要问题了。该函数将是一个很大的switch语句，按节点类型调度，包含用于评估每种节点的逻辑。我将对每种情况发表评论：

```js
function evaluate(exp, env) {
    switch (exp.type) {
```

对于常量节点，我们只返回它们的值：

```js
case "num":
      case "str":
      case "bool":
        return exp.value;
```

从环境中获取变量。请记住，“ var”标记在value属性中包含名称：

```js
      case "var":
        return env.get(exp.value);
```

对于分配，我们需要检查左侧是否为“ var”令牌（如果不是，则抛出错误；我们暂时不支持分配给其他任何对象）。然后，我们使用env.set来设置值。请注意，需要先通过递归调用评估来计算该值。

```js
case "assign":
        if (exp.left.type != "var")
            throw new Error("Cannot assign to " + JSON.stringify(exp.left));
        return env.set(exp.left.value, evaluate(exp.right, env));
```

“二进制”节点需要将一个运算符应用于两个操作数。稍后我们将编写apply_op函数，这很简单。同样，我们需要递归调用评估器以计算左右操作数：

```js
case "binary":
        return apply_op(exp.operator,
                        evaluate(exp.left, env),
                        evaluate(exp.right, env));
```

实际上，“ lambda”节点将导致JavaScript关闭，因此可以像普通函数一样从JavaScript对其进行调用。我编写了一个make_lambda函数，稍后将对其进行定义：

```js
case "lambda":
        return make_lambda(env, exp);
```

评估“ if”节点很简单：首先评估条件。如果它不是false，则评估“ then”分支并返回其值。否则，评估“ else”分支（如果存在），或者返回false。

```js
case "if":
        var cond = evaluate(exp.cond, env);
        if (cond !== false) return evaluate(exp.then, env);
        return exp.else ? evaluate(exp.else, env) : false;
```

“编”是一系列表达。我们只是按顺序评估它们，然后返回最后一个的值。对于空序列，返回值初始化为false。

```js
case "prog":
        var val = false;
        exp.prog.forEach(function(exp){ val = evaluate(exp, env) });
        return val;
```

对于“调用”节点，我们需要调用一个函数。首先我们评估func，它应该返回一个普通的JS函数，然后我们评估args并应用该函数。

```js
case "call":
        var func = evaluate(exp.func, env);
        return func.apply(null, exp.args.map(function(arg){
            return evaluate(arg, env);
        }));
```

我们永远都不会到达这里，但以防万一我们在解析器中添加新的节点类型而忘记更新评估器，让我们抛出一个明显的错误。

```js
default:
        throw new Error("I don't know how to evaluate " + exp.type);
    }
}
```

这是评估程序的核心，您可以看到它非常简单。我们仍然需要编写另外两个函数，让我们从apply_op开始，因为它是最简单的一个：

```js
function apply_op(op, a, b) {
    function num(x) {
        if (typeof x != "number")
            throw new Error("Expected number but got " + x);
        return x;
    }
    function div(x) {
        if (num(x) == 0)
            throw new Error("Divide by zero");
        return x;
    }
    switch (op) {
      case "+"  : return num(a) + num(b);
      case "-"  : return num(a) - num(b);
      case "*"  : return num(a) * num(b);
      case "/"  : return num(a) / div(b);
      case "%"  : return num(a) % div(b);
      case "&&" : return a !== false && b;
      case "||" : return a !== false ? a : b;
      case "<"  : return num(a) < num(b);
      case ">"  : return num(a) > num(b);
      case "<=" : return num(a) <= num(b);
      case ">=" : return num(a) >= num(b);
      case "==" : return a === b;
      case "!=" : return a !== b;
    }
    throw new Error("Can't apply operator " + op);
}
```

它接收运算符和参数。只是一个无聊的开关来应用它。与JavaScript不同，JavaScript会将任何运算符应用于任何参数，然后继续进行运算是否有意义，我们要求数字运算符的操作数为数字，并且使用小辅助函数num和div的divizor不为零。对于字符串，我们将定义其他内容

#### `make_lambda`

make_lambda有点奥妙：

```js
function make_lambda(env, exp) {
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

如您所见，它返回一个普通的JavaScript函数，该函数包围环境和要评估的表达式。重要的是要了解，创建此闭包时不会发生任何事情，但是，当调用此闭包时，它将使用新的参数/值绑定扩展创建时保存的环境（如果传递的值少于函数的参数列表，缺少的将获得false值）。然后，它仅在新范围内评估主体。

#### Primitive functions

您可以观察到我们的语言不提供任何与外界互动的方式。在某些代码示例中，我使用了一些print和println函数，但未在任何地方定义它们。必须将它们定义为原始函数（也就是说，我们将用JavaScript编写它们并将其插入全局环境中）。

现在将其放在一起，这是一个测试程序：

```js
// some test code here
var code = "sum = lambda(x, y) x + y; print(sum(2, 3));";

// remember, parse takes a TokenStream which takes an InputStream
var ast = parse(TokenStream(InputStream(code)));

// create the global environment
var globalEnv = new Environment();

// define the "print" primitive function
globalEnv.def("print", function(txt){
  console.log(txt);
});

// run the evaluator
evaluate(ast, globalEnv); // will print 5
```

#### Code

```js
function InputStream(input) {
    var pos = 0, line = 1, col = 0;
    return {
        next  : next,
        peek  : peek,
        eof   : eof,
        croak : croak,
    };
    function next() {
        var ch = input.charAt(pos++);
        if (ch == "\n") line++, col = 0; else col++;
        return ch;
    }
    function peek() {
        return input.charAt(pos);
    }
    function eof() {
        return peek() == "";
    }
    function croak(msg) {
        throw new Error(msg + " (" + line + ":" + col + ")");
    }
}

function TokenStream(input) {
    var current = null;
    var keywords = " if then else lambda 位 true false ";
    return {
        next  : next,
        peek  : peek,
        eof   : eof,
        croak : input.croak
    };
    function is_keyword(x) {
        return keywords.indexOf(" " + x + " ") >= 0;
    }
    function is_digit(ch) {
        return /[0-9]/i.test(ch);
    }
    function is_id_start(ch) {
        return /[a-z位_]/i.test(ch);
    }
    function is_id(ch) {
        return is_id_start(ch) || "?!-<>=0123456789".indexOf(ch) >= 0;
    }
    function is_op_char(ch) {
        return "+-*/%=&|<>!".indexOf(ch) >= 0;
    }
    function is_punc(ch) {
        return ",;(){}[]".indexOf(ch) >= 0;
    }
    function is_whitespace(ch) {
        return " \t\n".indexOf(ch) >= 0;
    }
    function read_while(predicate) {
        var str = "";
        while (!input.eof() && predicate(input.peek()))
            str += input.next();
        return str;
    }
    function read_number() {
        var has_dot = false;
        var number = read_while(function(ch){
            if (ch == ".") {
                if (has_dot) return false;
                has_dot = true;
                return true;
            }
            return is_digit(ch);
        });
        return { type: "num", value: parseFloat(number) };
    }
    function read_ident() {
        var id = read_while(is_id);
        return {
            type  : is_keyword(id) ? "kw" : "var",
            value : id
        };
    }
    function read_escaped(end) {
        var escaped = false, str = "";
        input.next();
        while (!input.eof()) {
            var ch = input.next();
            if (escaped) {
                str += ch;
                escaped = false;
            } else if (ch == "\\") {
                escaped = true;
            } else if (ch == end) {
                break;
            } else {
                str += ch;
            }
        }
        return str;
    }
    function read_string() {
        return { type: "str", value: read_escaped('"') };
    }
    function skip_comment() {
        read_while(function(ch){ return ch != "\n" });
        input.next();
    }
    function read_next() {
        read_while(is_whitespace);
        if (input.eof()) return null;
        var ch = input.peek();
        if (ch == "#") {
            skip_comment();
            return read_next();
        }
        if (ch == '"') return read_string();
        if (is_digit(ch)) return read_number();
        if (is_id_start(ch)) return read_ident();
        if (is_punc(ch)) return {
            type  : "punc",
            value : input.next()
        };
        if (is_op_char(ch)) return {
            type  : "op",
            value : read_while(is_op_char)
        };
        input.croak("Can't handle character: " + ch);
    }
    function peek() {
        return current || (current = read_next());
    }
    function next() {
        var tok = current;
        current = null;
        return tok || read_next();
    }
    function eof() {
        return peek() == null;
    }
}

function parse(input) {
    var PRECEDENCE = {
        "=": 1,
        "||": 2,
        "&&": 3,
        "<": 7, ">": 7, "<=": 7, ">=": 7, "==": 7, "!=": 7,
        "+": 10, "-": 10,
        "*": 20, "/": 20, "%": 20,
    };
    var FALSE = { type: "bool", value: false };
    return parse_toplevel();
    function is_punc(ch) {
        var tok = input.peek();
        return tok && tok.type == "punc" && (!ch || tok.value == ch) && tok;
    }
    function is_kw(kw) {
        var tok = input.peek();
        return tok && tok.type == "kw" && (!kw || tok.value == kw) && tok;
    }
    function is_op(op) {
        var tok = input.peek();
        return tok && tok.type == "op" && (!op || tok.value == op) && tok;
    }
    function skip_punc(ch) {
        if (is_punc(ch)) input.next();
        else input.croak("Expecting punctuation: \"" + ch + "\"");
    }
    function skip_kw(kw) {
        if (is_kw(kw)) input.next();
        else input.croak("Expecting keyword: \"" + kw + "\"");
    }
    function skip_op(op) {
        if (is_op(op)) input.next();
        else input.croak("Expecting operator: \"" + op + "\"");
    }
    function unexpected() {
        input.croak("Unexpected token: " + JSON.stringify(input.peek()));
    }
    function maybe_binary(left, my_prec) {
        var tok = is_op();
        if (tok) {
            var his_prec = PRECEDENCE[tok.value];
            if (his_prec > my_prec) {
                input.next();
                return maybe_binary({
                    type     : tok.value == "=" ? "assign" : "binary",
                    operator : tok.value,
                    left     : left,
                    right    : maybe_binary(parse_atom(), his_prec)
                }, my_prec);
            }
        }
        return left;
    }
    function delimited(start, stop, separator, parser) {
        var a = [], first = true;
        skip_punc(start);
        while (!input.eof()) {
            if (is_punc(stop)) break;
            if (first) first = false; else skip_punc(separator);
            if (is_punc(stop)) break;
            a.push(parser());
        }
        skip_punc(stop);
        return a;
    }
    function parse_call(func) {
        return {
            type: "call",
            func: func,
            args: delimited("(", ")", ",", parse_expression),
        };
    }
    function parse_varname() {
        var name = input.next();
        if (name.type != "var") input.croak("Expecting variable name");
        return name.value;
    }
    function parse_if() {
        skip_kw("if");
        var cond = parse_expression();
        if (!is_punc("{")) skip_kw("then");
        var then = parse_expression();
        var ret = {
            type: "if",
            cond: cond,
            then: then,
        };
        if (is_kw("else")) {
            input.next();
            ret.else = parse_expression();
        }
        return ret;
    }
    function parse_lambda() {
        return {
            type: "lambda",
            vars: delimited("(", ")", ",", parse_varname),
            body: parse_expression()
        };
    }
    function parse_bool() {
        return {
            type  : "bool",
            value : input.next().value == "true"
        };
    }
    function maybe_call(expr) {
        expr = expr();
        return is_punc("(") ? parse_call(expr) : expr;
    }
    function parse_atom() {
        return maybe_call(function(){
            if (is_punc("(")) {
                input.next();
                var exp = parse_expression();
                skip_punc(")");
                return exp;
            }
            if (is_punc("{")) return parse_prog();
            if (is_kw("if")) return parse_if();
            if (is_kw("true") || is_kw("false")) return parse_bool();
            if (is_kw("lambda") || is_kw("位")) {
                input.next();
                return parse_lambda();
            }
            var tok = input.next();
            if (tok.type == "var" || tok.type == "num" || tok.type == "str")
                return tok;
            unexpected();
        });
    }
    function parse_toplevel() {
        var prog = [];
        while (!input.eof()) {
            prog.push(parse_expression());
            if (!input.eof()) skip_punc(";");
        }
        return { type: "prog", prog: prog };
    }
    function parse_prog() {
        var prog = delimited("{", "}", ";", parse_expression);
        if (prog.length == 0) return FALSE;
        if (prog.length == 1) return prog[0];
        return { type: "prog", prog: prog };
    }
    function parse_expression() {
        return maybe_call(function(){
            return maybe_binary(parse_atom(), 0);
        });
    }
}

function Environment(parent) {
    this.vars = Object.create(parent ? parent.vars : null);
    this.parent = parent;
}
Environment.prototype = {
    extend: function() {
        return new Environment(this);
    },
    lookup: function(name) {
        var scope = this;
        while (scope) {
            if (Object.prototype.hasOwnProperty.call(scope.vars, name))
                return scope;
            scope = scope.parent;
        }
    },
    get: function(name) {
        if (name in this.vars)
            return this.vars[name];
        throw new Error("Undefined variable " + name);
    },
    set: function(name, value) {
        var scope = this.lookup(name);
        if (!scope && this.parent)
            throw new Error("Undefined variable " + name);
        return (scope || this).vars[name] = value;
    },
    def: function(name, value) {
        return this.vars[name] = value;
    }
};

function evaluate(exp, env) {
    switch (exp.type) {
      case "num":
      case "str":
      case "bool":
        return exp.value;

      case "var":
        return env.get(exp.value);

      case "assign":
        if (exp.left.type != "var")
            throw new Error("Cannot assign to " + JSON.stringify(exp.left));
        return env.set(exp.left.value, evaluate(exp.right, env));

      case "binary":
        return apply_op(exp.operator,
                        evaluate(exp.left, env),
                        evaluate(exp.right, env));

      case "lambda":
        return make_lambda(env, exp);

      case "if":
        var cond = evaluate(exp.cond, env);
        if (cond !== false) return evaluate(exp.then, env);
        return exp.else ? evaluate(exp.else, env) : false;

      case "prog":
        var val = false;
        exp.prog.forEach(function(exp){ val = evaluate(exp, env) });
        return val;

      case "call":
        var func = evaluate(exp.func, env);
        return func.apply(null, exp.args.map(function(arg){
            return evaluate(arg, env);
        }));

      default:
        throw new Error("I don't know how to evaluate " + exp.type);
    }
}

function apply_op(op, a, b) {
    function num(x) {
        if (typeof x != "number")
            throw new Error("Expected number but got " + x);
        return x;
    }
    function div(x) {
        if (num(x) == 0)
            throw new Error("Divide by zero");
        return x;
    }
    switch (op) {
      case "+": return num(a) + num(b);
      case "-": return num(a) - num(b);
      case "*": return num(a) * num(b);
      case "/": return num(a) / div(b);
      case "%": return num(a) % div(b);
      case "&&": return a !== false && b;
      case "||": return a !== false ? a : b;
      case "<": return num(a) < num(b);
      case ">": return num(a) > num(b);
      case "<=": return num(a) <= num(b);
      case ">=": return num(a) >= num(b);
      case "==": return a === b;
      case "!=": return a !== b;
    }
    throw new Error("Can't apply operator " + op);
}

function make_lambda(env, exp) {
    function lambda() {
        var names = exp.vars;
        var scope = env.extend();
        for (var i = 0; i < names.length; ++i)
            scope.def(names[i], i < arguments.length ? arguments[i] : false);
        return evaluate(exp.body, scope);
    }
    return lambda;
}

/* -----[ entry point for NodeJS ]----- */

var globalEnv = new Environment();

globalEnv.def("time", function(func){
    try {
        console.time("time");
        return func();
    } finally {
        console.timeEnd("time");
    }
});

if (typeof process != "undefined") (function(){
    var util = require("util");
    globalEnv.def("println", function(val){
        util.puts(val);
    });
    globalEnv.def("print", function(val){
        util.print(val);
    });
    var code = "";
    process.stdin.setEncoding("utf8");
    process.stdin.on("readable", function(){
        var chunk = process.stdin.read();
        if (chunk) code += chunk;
    });
    process.stdin.on("end", function(){
        var ast = parse(TokenStream(InputStream(code)));
        evaluate(ast, globalEnv);
    });
})();
```

在下一节中，我们将介绍λ语言。

