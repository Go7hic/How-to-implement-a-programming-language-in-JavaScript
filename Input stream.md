这是三个部分中最小的一部分。我们将会创建一个“字符流对象”，它主要提供一些从字符串中正确读取字符的操作。一个字符流对象有4个方法：

- peek() — 返回下一个字符的值，但是不将它移出字符流
- next() — 返回下一个字符的值，但是将它移出字符流
- eof() — 当且仅当字符流中没有更多值时返回 true
- croak(msg) — 抛出异常 throw new Error(msg)

我之所以要把最后一个方法也包含在内，是因为字符流很容易追踪到当前读取的位置（行／列），而且这在错误信息中很重要。

依赖你自己的需求，你可以在这里随便添加自己想要的方法，但对我的教程来说，上面这些就足够了。

字符输入流处理的是字符序列，所以方法next() / peek()的返回值是字符（噢，因为javascript并没有字符类型，所以这里的字符指的是只包含一个字符的字符串）。

下面是字符流对象的完整代码，我称之为"InputStream"。它很简单，我相信你理解起来不会有啥问题：

```
function InputStream(input) {
	this.input = input || '';
	this.pos = 0;
	this.line = 1; 
	this.col = 0;
}
InputStream.prototype.peek = function() {
	return this.input.charAt(this.pos);
}
InputStream.prototype.next = function() {
	let ch = this.input.charAt(this.pos++);
	if(ch == '\n') {
		this.line ++;
		this.col = 0;
	}else {
		this.col ++;
	}
	return ch;
}
InputStream.prototype.eof = function() {
	return this.peek() == '';
}
InputStream.prototype.croak = function (msg) {
	throw new Error(msg + ' (' + this.line + ':' + this.col + ')');
}

module.exports = InputStream;
```

接下来，我们将要在上面的基础上写另外一个抽象的概念：词法分析器（lexer）
