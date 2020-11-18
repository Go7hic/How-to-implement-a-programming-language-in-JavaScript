我们的词法分析器（lexer）在字符输入流的基础之上运行，并且通过相同的接口返回一个流对象，但是peek() / next()方法的返回值不再是字符，而是token。每个token是一个对象，它有两个属性：type和value。下面是我们的语言支持的一些token的例子：

```
{ type: "punc", value: "(" }           // 标点符号: 括号，逗号，分号等
{ type: "num", value: 5 }              // 数字
{ type: "str", value: "Hello World!" } // 字符串
{ type: "kw", value: "lambda" }        // 关键字
{ type: "var", value: "a" }            // 标识符
{ type: "op", value: "!=" }            // 操作符
```

空格和注释将会被跳过，不会返回token。

为了能写好词法分析器，我们需要对λanguage的语法有更仔细的认识。有个办法是，根据当前字符（由流对象的peek()方法返回）来决定下一个要读取的token:

首先，如果是空格，则直接跳过
如果input.eof()为true，则返回 null（注意，input是InputStream创建的字符输入流对象）
如果是井号 (#)，则跳过注释（这一行结束之后再重试）
如果是引号，则读取字符串
如果是数字，则继续读取一个数字
如果是个字母，则读取一个标识符或关键字token
如果是个标点符号， 则返回标点符号token
如果是个操作符，则返回操作符token
如果以上都不是，则通过input.croak()抛出一个异常
read_next方法实现了上面的功能（它可是我们词法分析器的主力噢～）：

```
TokenStream.prototype.read_next = function() {
    this.read_while(this.is_whitespace);
    if (this.input.eof()) return null;
    let ch = this.input.peek();
    if (ch == '#') {
        this.skip_comment();
        return this.read_next();
    }
    if (ch == '"') return this.read_string();
    if (this.is_digit(ch)) return this.read_number();
    if (this.is_id_start(ch)) return this.read_ident();
    if (this.is_punc(ch)) return {
        type  : 'punc',
        value : this.input.next()
    };
    if (this.is_op_char(ch)) return {
       type  : 'op',
        value : this.read_while(this.is_op_char)
    };
    this.input.croak("这个字符本宝宝处理不了: " + ch);
}
```

这是个“调度器”方法，next方法会调用它以获取下一个token。注意它使用了许多专门针对特定类型token的实用程序，比如read_string, read_number等。即使我们从不在别处调用它们，也没有任何必要使调度程序与这些功能的代码复杂化。

另一个要注意的事情是，我们不会在一个步骤中消耗所有的输入流。每当分析器调用下一个token时, 我们都会读取一个token。在分析错误的情况下, 我们甚至不到达流的末尾。

只要字符被允许作为标识符 (is_id) 的一部分，read_ident方法就会读取它们。标识符必须以字母或λ或 _ 开头，并且可以进一步包含此类字符或数字，或下列内容之一：?-<>=。因此，foo- bar 不会被读取为三个token，而是作为一个单独的标识符 ('var' token)。之所以有这条规则，是因为我希望能够定义名为'is-pair?'的函数或字符串 > = (对不起，这是我的 Lisper)（译者注：你开心就好～）。

逼逼了这么久，我觉得是时候上代码了。下面就是我们语言的词法分析器的完整代码，代码完了以后是一些小注释。

```
function TokenStream(input) {
	this.input = input || '';
	this.current = null;
}
TokenStream.prototype = {
	constructor: TokenStream,
	keywords: ' if then else lambda λ true false ',
	is_keyword: function(x) {
		return this.keywords.indexOf(' ' + x + ' ') > -1;
	},
	is_digit: function(ch) {
		return /[0-9]/i.test(ch);
	},
	is_id_start: function(ch) {
		return /[a-zλ_]/i.test(ch);
	},
	is_id: function(ch) {
		return this.is_id_start(ch) || '?!-<>=0123456789'.indexOf(ch) > -1;
	},
	is_op_char: function(ch) {
		return '+-*/%=&|<>!'.indexOf(ch) > -1;
	},
	is_punc: function(ch) {
		return ',;(){}[]'.indexOf(ch) > -1;
	},
	is_whitespace: function(ch) {
		return ' \t\n'.indexOf(ch) > -1;
	},
	read_while: function(predicate) {
		let str = '';
		while(!this.input.eof() && predicate(this.input.peek())){
			str += this.input.next();
		}
		return str;
	},
	read_number: function() {
        let has_dot = false;
        let number = this.read_while(ch => {
            if (ch == ".") {
                if (has_dot) return false;
                has_dot = true;
                return true;
            }
            return this.is_digit(ch);
        });
        return { type: "num", value: parseFloat(number) };
    },
    read_ident: function() {
        let id = this.read_while(is_id);
        return {
            type  : this.is_keyword(id) ? "kw" : "var",
            value : id
        };
    },
    read_escaped: function(end) {
        let escaped = false, str = '';
        this.input.next();
        while (!this.input.eof()) {
            var ch = this.input.next();
            if (escaped) {
                str += ch;
                escaped = false;
            } else if (ch == '\\') {
                escaped = true;
            } else if (ch == end) {
                break;
            } else {
                str += ch;
            }
        }
        return str;
    },
    read_string: function() {
        return { type: "str", value: this.read_escaped('"') };
    },
    skip_comment: function() {
        this.read_while(ch => ch != "\n");
        this.input.next();
    },
    read_next: function() {
        this.read_while(this.is_whitespace);
        if (this.input.eof()) return null;
        let ch = this.input.peek();
        if (ch == '#') {
            this.skip_comment();
            return this.read_next();
        }
        if (ch == '"') return this.read_string();
        if (this.is_digit(ch)) return this.read_number();
        if (this.is_id_start(ch)) return this.read_ident();
        if (this.is_punc(ch)) return {
            type  : 'punc',
            value : this.input.next()
        };
        if (this.is_op_char(ch)) return {
            type  : 'op',
            value : this.read_while(this.is_op_char)
        };
        this.input.croak("这个字符本宝宝处理不了: " + ch);
    },
    peek: function() {
        return this.current || (this.current = this.read_next());
    },
    next: function() {
        let tok = this.current;
        this.current = null;
        return tok || this.read_next();
    },
    eof: function() {
        return this.peek() == null;
    }
}
module.exports = TokenStream;
```

next方法并不总是会调用read_next方法，因为它可能已经被偷看过 (在这种情况下, read_next方法已经在流调用了)。因此我们需要一个current变量来追踪当前token。
我们只支持用通常记号的十进制数字 (没有1E5 这种东西，没有十六进制，没有八进制) 。但是如果我们需要更多，只需要修改read_number方法就好了，并且很容易实现。
与 javascript 不同，唯一不能在字符串中出现的未加引号的字符是引号字符本身和反斜杠。你需要对他们进行反斜杠转译。另外，字符串可以包含硬换行符、制表符和诸如此类的内容。我们不解析通常意义上的转义（像 \ n, \ t 等）。同上，但是这次改变会相当繁琐（在read_string方法中改）。

现在我们有足够强大的工具来方便地编写 解析器（parser），但我建议您首先看下 AST的描述。
