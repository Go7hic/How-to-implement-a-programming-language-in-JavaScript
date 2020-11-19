# How-to-implement-a-programming-language-in-JavaScript
【译】用 JavaScript 实现一门编程语言
> 原系列文章 http://lisperator.net/pltut/

在线阅读：https://go7hic.github.io/How-to-implement-a-programming-language-in-JavaScript/#/

## 目录
- Introduction
- λanguage description
- Writing a parser
  - Input stream
  - Token stream
  - The AST
  - The parser
  - Simple interpreter
  - Test what we have
  - Adding new constructs
  - How fast are we?
- CPS Evaluator
  - Guarding the stack
  - Continuations
  - Yield (advanced)
- Compiling to JS
  - JS code generator
  - CPS transformer
    - Samples
    - Improvements
  - Optimizer
- Wrapping up
- Real samples
  - Primitives
  - catDir
  - copyTree sequential
  - copyTree parallel
  - In fairness to Node
  - Error handling
