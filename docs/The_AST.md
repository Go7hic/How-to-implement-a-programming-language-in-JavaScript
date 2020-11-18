## Description of the AST

啥是AST呢？AST中文翻译为抽象语法树，也就是一个能准确代表我们程序语义的一种数据结构而已。我们的AST中每个节点都是一个纯js对象字面量，这个js对象有一个type属性，用来说明这个节点具体是什么类型的，然后还有其他一些属性，这些属性依据节点类型的不同而不同。下面给出我们创造的这门语言的所有AST节点的map表：

#### In short:
```js
num    { type: "num", value: NUMBER }
str    { type: "str", value: STRING }
bool   { type: "bool", value: true or false }
var    { type: "var", value: NAME }
lambda { type: "lambda", vars: [ NAME... ], body: AST }
call   { type: "call", func: AST, args: [ AST... ] }
if     { type: "if", cond: AST, then: AST, else: AST }
assign { type: "assign", operator: "=", left: AST, right: AST }
binary { type: "binary", operator: OPERATOR, left: AST, right: AST }
prog   { type: "prog", prog: [ AST... ] }
let    { type: "let", vars: [ VARS... ], body: AST }
```

#### Examples

![image-20191015234910780](/Users/go7hic/workspace/Github/How-to-implement-a-programming-language-in-JavaScript/image-20191015234910780.png)

![image-20191016002002431](/Users/go7hic/workspace/Github/How-to-implement-a-programming-language-in-JavaScript/image-20191016002002431.png)

![image-20191016003117285](/Users/go7hic/workspace/Github/How-to-implement-a-programming-language-in-JavaScript/image-20191016003117285.png)

![image-20191016103201171](/Users/go7hic/workspace/Github/How-to-implement-a-programming-language-in-JavaScript/image-20191016103201171.png)

![image-20191016103124496](/Users/go7hic/workspace/Github/How-to-implement-a-programming-language-in-JavaScript/image-20191016103124496.png)