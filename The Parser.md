## The parser

解析器会创建 The AST 中描述的AST节点。



由于我们在 token 生成器中所做的工作，解析器对 token 流进行操作，而不是处理单个字符。它仍然定义了许多辅助函数来降低复杂性。我将在这里讨论构成`parser` 的主要函数。让我们从最顶层的lambda解析器开始：

```js
function parse_lambda() {
    return {
        type: "lambda",
        vars: delimited("(", ")", ",", parse_varname),
        body: parse_expression()
    };
}
```

当已经看到 lambda 关键字并从输入中“吃掉”该关键字时，这个函数将被调用，因此它所关心的只是解析参数名称；这些参数在括号中并以逗号分隔。与其将代码放在`parse_lambda`中，我更喜欢编写一个带分隔符的函数来接受这些参数：起始token，结束token，分隔符以及一个解析这些起始/结束 token 之间必须存在的内容的函数。在这种情况下，它是`parse_varname`，如果遇到任何看起来不像变量的东西，它会抛出错误。该函数的主体是一个表达式，因此我们可以通过parse_expression获得它。

delimited 比较底层：

```js
function delimited(start, stop, separator, parser) {
    var a = [], first = true;
    skip_punc(start);
    while (!input.eof()) {
        if (is_punc(stop)) break;
        if (first) first = false; else skip_punc(separator);
        if (is_punc(stop)) break; // the last separator can be missing
        a.push(parser());
    }
    skip_punc(stop);
    return a;
}
```

如你所见，它使用了很多工具函数：`is_punc`  和  `skip_punc`。如果当前token是给定的标点符号（不“吃掉”），则 is_punc 将返回true，而skip_punc将确保当前token是那个标点（否则将引发错误），并将其从输入中丢弃。

解析整个程序的函数可能是最简单的：

```js
function parse_toplevel() {
    var prog = [];
    while (!input.eof()) {
        prog.push(parse_expression());
        if (!input.eof()) skip_punc(";");
    }
    return { type: "prog", prog: prog };
}
```

由于没有语句，因此只需调用`parse_expression（）`并读取表达式，直到到达输入的末尾。使用`skip_punc（“;”）`，我们需要在这些表达式之间使用分号。



另一个简单的例子：parse_if():

```js
function parse_if() {
    skip_kw("if");
    var cond = parse_expression();
    if (!is_punc("{")) skip_kw("then");
    var then = parse_expression();
    var ret = { type: "if", cond: cond, then: then };
    if (is_kw("else")) {
        input.next();
        ret.else = parse_expression();
    }
    return ret;
}
```

它用 skip_kw 跳过 if 关键字（如果当前 token 不是给定的关键字，则会抛出错误），并使用`parse_expression（）`读取条件。接下来，如果结果分支不是以`{`开头，那么我们需要关键字然后出现（我觉得没有它的语法太稀缺了）。这分支只是表达式，因此我们再次对它们使用`parse_expression（）`。else分支是可选的，因此我们需要在解析关键字之前检查该关键字是否存在。



拥有许多小型实用程序有助于使代码保持简单。我们几乎像使用专门用于解析的高级语言一样编写解析器。所有这些函数都是“相互递归”的，例如：有一个`parse_atom（）`函数是主调度程序—基于当前token，它调用其他函数。其中之一是`parse_if（）`（在当前标记为 if 时调用），然后依次调用`parse_expression（）`。但是`parse_expression（）`调用`parse_atom（）`。之所以没有无限循环，是因为在每个步骤中，一个功能或另一个功能都将推进至少一个token。

这种解析器称为“递归下降解析器”，它可能是手动编写的最简单的解析器。

#### 更底层的: parse_atom() and parse_expression()

`parse atom（）`完成主调度程序的工作，具体取决于当前令牌：



```js
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
        if (is_kw("lambda") || is_kw("λ")) {
            input.next();
            return parse_lambda();
        }
        var tok = input.next();
        if (tok.type == "var" || tok.type == "num" || tok.type == "str")
            return tok;
        unexpected();
    });
}
```

如果它看到一个开放的括号，则它必须是一个带括号的表达式-因此，跳过括号，调用`parse_expression（）`并期望一个封闭的括号。如果看到某个关键字，它将调用适当的解析器函数。如果看到常量或标识符，则按原样返回。如果没有任何效果，`unexpected（）`将引发错误。

当期望一个原子表达式并看到 `{` 时，它将调用 `parse_prog `来解析一个表达式序列。定义如下。此时它将进行一些小的优化-如果`prog`为空，则仅返回FALSE。如果它具有单个表达式，则返回它而不是`“ prog”`节点。否则，它将返回一个包含表达式的`“ prog”`节点。

```js
var FALSE = { type: "bool", value: false };

function parse_prog() {
    var prog = delimited("{", "}", ";", parse_expression);
    if (prog.length == 0) return FALSE;
    if (prog.length == 1) return prog[0];
    return { type: "prog", prog: prog };
}
```

这是`parse_expression（）`函数。与`parse_atom（）`相反，此函数将使用`maybe_binary（）`尽可能向右扩展一个表达式，下面将对此进行说明。

```js
function parse_expression() {
    return maybe_call(function(){
        return maybe_binary(parse_atom(), 0);
    });
}
```

#### The`maybe_*` functions

这些函数检查表达式之后的内容，以决定是将该表达式包装在另一个节点中，还是直接将其返回。

might_call（）非常简单。它接收到一个预期解析当前表达式的函数。如果在该表达式之后看到一个（标点符号，则它必须是一个“调用”节点，这就是parse_call（）的作用（包括在下面）。再次注意delimited（）如何方便地读取参数列表。

```js
function maybe_call(expr) {
    expr = expr();
    return is_punc("(") ? parse_call(expr) : expr;
}

function parse_call(func) {
    return {
        type: "call",
        func: func,
        args: delimited("(", ")", ",", parse_expression)
    };
}
```

##### 运算符优先级

may_binary（left，my_prec）用于组成1 + 2 * 3之类的二进制表达式。正确解析它们的技巧是正确定义运算符优先级，因此我们从此开始：

```js
var PRECEDENCE = {
    "=": 1,
    "||": 2,
    "&&": 3,
    "<": 7, ">": 7, "<=": 7, ">=": 7, "==": 7, "!=": 7,
    "+": 10, "-": 10,
    "*": 20, "/": 20, "%": 20,
};
```

这表示`*`比`+`优先级更高，因此像`1 + 2 * 3`这样的表达式必须读为（1 +（2 * 3））而不是（（1 + 2）* 3），这通常是从左到右的顺序运行解析器。

诀窍是读取原子表达式（仅1）并将其与当前优先级（my_prec）一起传递给maybe_binary（）（左侧参数）。mayry_binary会看下面的内容。如果没有看到运算符，或者它的优先级较小，则按原样返回left。

如果它是一个运算符，其优先级高于我们的运算符，则它会在一个新的“二进制”节点中向左换行，而在右侧，它会以新的优先级（*）重复该技巧：

```js
function maybe_binary(left, my_prec) {
    var tok = is_op();
    if (tok) {
        var his_prec = PRECEDENCE[tok.value];
        if (his_prec > my_prec) {
            input.next();
            var right = maybe_binary(parse_atom(), his_prec) // (*);
            var binary = {
                type     : tok.value == "=" ? "assign" : "binary",
                operator : tok.value,
                left     : left,
                right    : right
            };
            return maybe_binary(binary, my_prec);
        }
    }
    return left;
}
```

请注意，在返回二进制表达式之前，我们还必须在旧优先级（my_prec）上调用maybe_binary，以便在运算符具有更高优先级的情况下将表达式包装在另一个表达式中。如果这一切令人困惑，请一次又一次阅读代码（也许尝试在某些输入表达式上明智地执行它），直到获得它为止.

最后，由于my_prec最初为零，因此任何运算符都将触发“二进制”节点的构建（或当运算符为=时将触发“分配”）。

解析器中还有更多功能，因此我在下面包括了整个解析功能。

```js
var FALSE = { type: "bool", value: false };
function parse(input) {
    var PRECEDENCE = {
        "=": 1,
        "||": 2,
        "&&": 3,
        "<": 7, ">": 7, "<=": 7, ">=": 7, "==": 7, "!=": 7,
        "+": 10, "-": 10,
        "*": 20, "/": 20, "%": 20,
    };
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
            if (is_kw("lambda") || is_kw("λ")) {
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
```