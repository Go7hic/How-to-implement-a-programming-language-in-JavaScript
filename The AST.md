## Description of the AST

啥是AST呢？AST中文翻译为抽象语法树，也就是一个能准确代表我们程序语义的一种数据结构而已。我们的AST中每个节点都是一个纯js对象字面量，这个js对象有一个type属性，用来说明这个节点具体是什么类型的，然后还有其他一些属性，这些属性依据节点类型的不同而不同。下面给出我们创造的这门语言的所有AST节点的map表：

#### In short:
```js
num { type: "num", value: NUMBER }
str { type: "str", value: STRING }
bool { type: "bool", value: true or false }
var { type: "var", value: NAME }
lambda { type: "lambda", vars: [ NAME... ], body: AST }
call { type: "call", func: AST, args: [ AST... ] }
if { type: "if", cond: AST, then: AST, else: AST }
assign { type: "assign", operator: "=", left: AST, right: AST }
binary { type: "binary", operator: OPERATOR, left: AST, right: AST }
prog { type: "prog", prog: [ AST... ] }
let { type: "let", vars: [ VARS... ], body: AST }
```

#### Examples

```
 Numbers ("num")

123.5 →	{ type: "num", value: 123.5 }

 Strings ("str")

"Hello World!" →	{ type: "str", value: "Hello World!" }

 Booleans ("bool")

true     { type: "bool", value: true }
false →	 { type: "bool", value: false }

 Identifiers ("var")

foo →	{ type: "var", value: "foo" }

 Functions ("lambda")

                       	{
                              type: "lambda",
lambda (x) 10   # or
λ (x) 10             →        vars: [ "x" ],
                              body: { type: "num", value: 10 }
                          }
                          
                          
 
 
Function calls ("call")


              {
                "type": "call",
                "func": { "type": "var", "value": "foo" },
                "args": [
foo(a, 1)  →	    { "type": "var", "value": "a" },
                  { "type": "num", "value": 1 }
                ]
              }




Conditionals ("if")


                                  {
                                    "type": "if",
                                    "cond": { "type": "var", "value": "foo" },
 if foo then bar else baz  →	       "then": { "type": "var", "value": "bar" },
                                    "else": { "type": "var", "value": "baz" }
                                  }



Assignment ("assign")


                                  {
                                    "type": "assign",
                                    "operator": "=",
a = 10 →	                           "left": { "type": "var", "value": "a" },
                                    "right": { "type": "num", "value": 10 }
                                  }
                                  
                                  
Binary expressions ("binary")

	
                                    {
                                      "type": "binary",
                                      "operator": "+",
                                      "left": { "type": "var", "value": "x" },
                                      "right": {
x + y * z  →                            "type": "binary",
                                        "operator": "*",
                                        "left": { "type": "var", "value": "y" },
                                        "right": { "type": "var", "value": "z" }
                                      }
                                    }   
    
    
    
    
 Sequences ("prog")

	
                                                          {
                                                            "type": "prog",
                                                            "prog": [
                                                              {
                                                                "type": "assign",
                                                                "operator": "=",
                                                                "left": { "type": "var", "value": "a" },
                                                                "right": { "type": "num", "value": 5 }
                                                              },
                                                              {
                                                                "type": "assign",
                                                                "operator": "=",
                                                                "left": { "type": "var", "value": "b" },
                                                                "right": {
{
  a = 5;
  b = a * 2;                  → 
  a + b;
}                                                                  "type": "binary",
                                                                  "operator": "*",
                                                                  "left": { "type": "var", "value": "a" },
                                                                  "right": { "type": "num", "value": 2 }
                                                                }
                                                              },
                                                              {
                                                                "type": "binary",
                                                                "operator": "+",
                                                                "left": { "type": "var", "value": "a" },
                                                                "right": { "type": "var", "value": "b" }
                                                              }
                                                            ]
                                                          }





Block scoped variables ("let")



                                              {
                                                "type": "let",
                                                "vars": [
                                                  {
                                                    "name": "a",
                                                    "def": { "type": "num", "value": 10 }
                                                  },
                                                  {
                                                    "name": "b",
  
let (a = 10, b = a * 10) {
  a + b;                             →	
}                                                  "def": {
                                                      "type": "binary",
                                                      "operator": "*",
                                                      "left": { "type": "var", "value": "a" },
                                                      "right": { "type": "num", "value": 10 }
                                                    }
                                                  }
                                                ],
                                                "body": {
                                                  "type": "binary",
                                                  "operator": "+",
                                                  "left": { "type": "var", "value": "a" },
                                                  "right": { "type": "var", "value": "b" }
                                                }
                                              }

```
