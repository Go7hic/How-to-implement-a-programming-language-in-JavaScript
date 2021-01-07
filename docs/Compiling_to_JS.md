# Compilation to JavaScript

为了使我们的λ语言更有效，而不是评估（解释）AST，我们应该将其转换为目标语言（JavaScript）的程序。 然后，我们可以使用eval或新的Function运行它（或使用<script>标签加载它），我向您保证，速度的提高将是惊人的。 我将这项工作分为三个部分：

- JavaScript code generator
- Transformation to continuation-passing style
- Optimization

从直观上讲，我应该从“最后阶段”（代码生成）开始，这似乎很奇怪。 我这样做是因为它很有趣而且很容易，它将帮助我们实现和调试后面的阶段。 在后面的阶段之后，我们只需要对代码生成器进行少量修改。