根据语言来编写解析器是一个中等复杂的任务。
实质上，它必须将一段代码（我们通过查看字符检查）转换为“抽象语法树”（AST）。
AST是程序的结构化内存表示，它的“抽象”在于它不在乎什么字符是源代码制成的，但它忠实地表示了它的语义。
我写了一个单独的页面来描述我们的AST[describe our AST](http://lisperator.net/pltut/parser/the-ast)。

举个例子，下面的代码文本：
```
sum = lambda(a, b) {
  a + b;
};
print(sum(1, 2));
```

我们的解析器会以JavaScript对象的格式生成如下AST：
```
{
  type: "prog",
  prog: [
    // 第1行到第3行:
    {
      type: "assign",
      operator: "=",
      left: { type: "var", value: "sum" },
      right: {
        type: "lambda",
        vars: [ "a", "b" ],
        body: {
          // body本来应该是一个 "prog"，但是由于它仅仅包含一条表达式， 
          // 我们的解析器将其简化为表达式本身
          type: "binary",
          operator: "+",
          left: { type: "var", value: "a" },
          right: { type: "var", value: "b" }
        }
      }
    },
    // 第4行:
    {
      type: "call",
      func: { type: "var", value: "print" },
      args: [{
        type: "call",
        func: { type: "var", value: "sum" },
        args: [ { type: "num", value: 1 },
                { type: "num", value: 2 } ]
      }]
    }
  ]
}
```

编写解析器的主要困难在于很难正确地组织代码。 解析器应该在比从字符串中读取字符更高的层次上运行。 关于如何保持复杂性可管理的几点建议：

 - 多写几个函数并尽量保持它们小而美，每个函数只做并做好一件事；
 - 不要尝试使用正则表达式进行解析。它们没啥卵用。正则表达式可能对词法分析器（lexer）有点帮助，但我建议将它们限制在处理非常简单的事情上；
 - 不要尝试去猜测。当你不确定如何解析的时候，抛出一个异常并且保证错误信息包含错误发生的位置（异常所在的行／列）
 
为了简单起见，我把这部分的代码拆分成了3个主要部分来讲解（这三个部分又分别被拆分成了很多的小函数）：

 - 字符输入流
 - 词法分析器（lexer）
 - 解析器（parser）
