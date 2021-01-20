更新：在所有解释器，CPS评估程序和CPS编译器中都有一个错误：a（）&& b（）或a（）||之类的表达式 b（）总是评估双方。 当然，这是错误的，是否评估b（）取决于a（）的虚假值。 修复非常简单（将||和&&二进制操作转换为“ if”节点），但是我尚未更新这些页面。 真可惜

# Optimizer

在CPS中优化代码是许多学术研究的主题。 但是，即使不了解很多理论，我们仍然可以实现一些明显的改进。 对于fib函数，与以前的版本相比，加速约为3倍：
```
fib = λ(n) if n < 2 then n else fib(n - 1) + fib(n - 2);
time( λ() println(fib(27)) );

```
Result:

```
196418
Time: 468ms
***Result: false
```

与我们编写的其他代码一样，优化器以递归方式遍历AST，并根据一些规则将表达式尽可能地转换为更简单的表达式。 它一直有效直到找到数学家称之为“固定点”的：optimize（ast）== ast。 也就是说，无法再减少AST。 为此，它会在启动之前将change变量重置为零，并在每次AST减少时将其递增。 当变化保持为零时，我们就完成了。

### 例子

#### fib 函数

接下来，您可以看到“ fib”功能通过的转换顺序。 我正在展示JavaScript代码以发挥作用，但是请记住，优化器会接收AST并返回AST； make_js在结尾仅被调用一次。

优化器本身不是很优化-它可以单次执行更多操作，但是我选择一次进行单个修改，当change> 0时返回原始表达式不变（出于我稍后将详述的原因）。

- Initial code, as returned by to_cps.
```js
fib = function β_CC(β_K1, n) {
  GUARD(arguments, β_CC);
  (function β_CC(β_I2) {
    GUARD(arguments, β_CC);
    n < 2 ? β_I2(n) : fib(function β_CC(β_R4) {
      GUARD(arguments, β_CC);
      fib(function β_CC(β_R5) {
        GUARD(arguments, β_CC);
        β_I2(β_R4 + β_R5);
      }, n - 2);
    }, n - 1);
  })(function β_CC(β_R3) {
    GUARD(arguments, β_CC);
    β_K1(β_R3);
  });
};
```
- Unwraps one IIFE. β_I2 (which is the continuation of an "if" node) becomes a variable in the containing function.
```js
fib = function β_CC(β_K1, n) {
  var β_I2;
  GUARD(arguments, β_CC);
  β_I2 = function β_CC(β_R3) {
    GUARD(arguments, β_CC);
    β_K1(β_R3);
  }, n < 2 ? β_I2(n) : fib(function β_CC(β_R4) {
    GUARD(arguments, β_CC);
    fib(function β_CC(β_R5) {
      GUARD(arguments, β_CC);
      β_I2(β_R4 + β_R5);
    }, n - 2);
  }, n - 1);
};
```
- “Tail call optimization”. TCO is not really what happens here, but the net result is the same. It notices that β_I2 is effectively equivalent to β_K1 so it drops one needless closure.
```js
fib = function β_CC(β_K1, n) {
  var β_I2;
  GUARD(arguments, β_CC);
  β_I2 = β_K1, n < 2 ? β_I2(n) : fib(function β_CC(β_R4) {
    GUARD(arguments, β_CC);
    fib(function β_CC(β_R5) {
      GUARD(arguments, β_CC);
      β_I2(β_R4 + β_R5);
    }, n - 2);
  }, n - 1);
};
```
- Variable elimination. Since β_I2 is the same as β_K1, all occurrences of the former are replaced with the latter, and the assignment dropped.

```js
fib = function β_CC(β_K1, n) {
  var β_I2;
  GUARD(arguments, β_CC);
  β_K1, n < 2 ? β_K1(n) : fib(function β_CC(β_R4) {
    GUARD(arguments, β_CC);
    fib(function β_CC(β_R5) {
      GUARD(arguments, β_CC);
      β_K1(β_R4 + β_R5);
    }, n - 2);
  }, n - 1);
};
```

- Drops unneeded code. Since β_I2 is no longer used, the var statement can be dropped. Also, a pointless β_K1 is discarded.
```js
fib = function β_CC(β_K1, n) {
  GUARD(arguments, β_CC);
  n < 2 ? β_K1(n) : fib(function β_CC(β_R4) {
    GUARD(arguments, β_CC);
    fib(function β_CC(β_R5) {
      GUARD(arguments, β_CC);
      β_K1(β_R4 + β_R5);
    }, n - 2);
  }, n - 1);
};
```
- Discard some GUARD calls. When a function immediately calls another function, chances are that the stack will be reset by that next function, so we can afford to not guard, given that our stack nesting limit is pretty small (200 calls).
```js
fib = function β_CC(β_K1, n) {
  GUARD(arguments, β_CC);
  n < 2 ? β_K1(n) : fib(function(β_R4) {
    fib(function(β_R5) {
      β_K1(β_R4 + β_R5);
    }, n - 2);
  }, n - 1);
};
```

- No more changes. It reached the fixed point, so it stops there.
```js
fib = function β_CC(β_K1, n) {
  GUARD(arguments, β_CC);
  n < 2 ? β_K1(n) : fib(function(β_R4) {
    fib(function(β_R5) {
      β_K1(β_R4 + β_R5);
    }, n - 2);
  }, n - 1);
};
```

let nodes

I won't show all the intermediate steps on the next examples, but only the initial and final form.

Input code:
```js
(λ(){
  let (a = 2, b = 5) {
    print(a + b);
  };
})();
```
Transformation:
```js
(function β_CC(β_K1) {
  GUARD(arguments, β_CC);
  (function β_CC(β_K2, a) {
    GUARD(arguments, β_CC);
    (function β_CC(β_K3, b) {
      GUARD(arguments, β_CC);
      print(function β_CC(β_R4) {
        GUARD(arguments, β_CC);
        β_K3(β_R4);
      }, a + b);
    })(function β_CC(β_R5) {
      GUARD(arguments, β_CC);
      β_K2(β_R5);
    }, 5);
  })(function β_CC(β_R6) {
    GUARD(arguments, β_CC);
    β_K1(β_R6);
  }, 2);
})(function β_CC(β_R7) {
  GUARD(arguments, β_CC);
  β_TOPLEVEL(β_R7);
});
```

<pre>
(function β_CC(β_K1) {
  var a, b;
  GUARD(arguments, β_CC);
  a = 2, b = 5, print(β_K1, a + b);
})(β_TOPLEVEL);
</pre>


没有名称冲突，因此a和b变量不会重命名。 如您所见，to_cps通过将其编译为IIFE-s来增加了一个简单的let的开销，但是优化器能够丢弃该问题。

Transformation

<pre>
(function β_CC(β_K1) {
  GUARD(arguments, β_CC);
  (function β_CC(β_K2, a) {
    GUARD(arguments, β_CC);
    (function β_CC(β_K3, a) {
      GUARD(arguments, β_CC);
      print(function β_CC(β_R4) {
        GUARD(arguments, β_CC);
        β_K3(β_R4);
      }, a);
    })(function β_CC(β_R5) {
      GUARD(arguments, β_CC);
      print(function β_CC(β_R6) {
        GUARD(arguments, β_CC);
        β_K2(β_R6);
      }, a);
    }, 3);
  })(function β_CC(β_R7) {
    GUARD(arguments, β_CC);
    β_K1(β_R7);
  }, 2);
})(function β_CC(β_R8) {
  GUARD(arguments, β_CC);
  β_TOPLEVEL(β_R8);
});
</pre>

<pre>
(function β_CC(β_K1) {
  var a, β_K3, β_a$9;
  GUARD(arguments, β_CC);
  a = 2, β_K3 = function(β_R5) {
    print(β_K1, a);
  }, β_a$9 = 3, print(β_K3, β_a$9);
})(β_TOPLEVEL);
</pre>


#### 实现概述

如前所述，优化器将降低AST并简化节点，直到不再有减少的可能性为止。 与我们编写的其他AST步行者不同，这是“破坏性”的-它可能会修改输入的AST，而不是创建新的AST步行者。 它的主要优点是：

- 丢弃一些无用的闭包：λ（x）{y（x）}→y。
- 尽可能丢弃未使用的变量。
- 将IIFE-s解包到包含函数中。
- 计算常量表达式：a = 2 + 3 * 4→a = 14。
- 同样，如果“ if”的条件不变，则用适当的分支替换“ if”。
- 丢弃“ prog”节点中无副作用的表达式。
- 使用不加保护的“ lambda”节点提示：尽可能为true。


某些优化可能会“cascade”-例如，在用另一个变量替换变量之后，前者可能是丢弃的候选对象。 因此，在进行任何更改后重复此过程很重要。

#### make_scope
因为我们搞砸了环境（有时引入变量，有时将其删除），所以我们需要一种精确的方法来找出变量之间的联系。 例如，“对变量x的引用是什么？” 我编写了一个辅助函数make_scope来提供该功能。 它遍历AST，相应地创建Environment对象，并注释一些这样的节点：

- “ lambda”-由于它们创建了新的环境，因此每个“ lambda”节点都将获得一个指向其环境的env属性。 同样，“ lambda”节点将获得locs属性-此函数“本地”的变量名数组（将在最终代码中用var定义）。
- “ var”-变量节点将获得指向定义该变量的环境的env属性和指向定义对象的def属性。

定义对象将存储在定义它的环境中的每个变量中，为了方便起见，将存储在指向它的每个“ var”节点的def属性中。 它可能包含以下内容：
```
{
  refs: [/* array of "var" nodes pointing to this definition */],
  assigned: integer, // how many times is this var assigned to?
  cont: true or false, // is this a continuation argument?
}
```
此信息使优化器可以执行其操作。 例如，如果一个变量被分配的次数与引用次数相同，则意味着我们不会在任何地方使用它的值（因为每个赋值都将引用一个引用），因此可以将其删除。 或者，当我们需要用另一个Y替换变量X时，我们可以遍历X的引用以将每个引用节点中的名称更改为“ Y”。

我们添加到“ lambda”节点的locs属性将用作优化器放置其从IIFE-s解包的变量的位置。 make_scope将保留它（如果已经存在的话）（并会小心地在环境中定义那些变量），否则将其设置为空数组。 JS代码生成器（在js_lambda中）将包含这些名称的var语句（如果存在）（因此，在js_lambda中需要mod）。

我的第一步是在开始时调用make_scope，并让优化器为所做的每次更改维护所有env和def额外属性，其优点是我们可以一次减少多次。 但是我放弃了这么复杂的代码。 当前代码在每个优化器通过之前调用make_scope，并在进行任何精简后将其变为无操作（这是必要的，这样我们就不会使用过时的作用域信息）。

#### The optimize function

如前所述，它将在进入每个优化程序阶段之前调用make_scope，并重复进行直到没有更多更改为止。 最后，在返回优化表达式之前，它将顶级作用域保存在env属性中。 这是入口点：
```js
function optimize(exp) {
    do {
        var changes = 0;
        var defun = exp;
        make_scope(exp);
        exp = opt(exp);
    } while (changes);
    return exp;

    function opt(exp) {
        if (changes) return exp;
        switch (exp.type) {
          case "num"    :
          case "str"    :
          case "bool"   :
          case "var"    : return exp;
          case "binary" : return opt_binary (exp);
          case "assign" : return opt_assign (exp);
          case "if"     : return opt_if     (exp);
          case "prog"   : return opt_prog   (exp);
          case "call"   : return opt_call   (exp);
          case "lambda" : return opt_lambda (exp);
        }
        throw new Error("I don't know how to optimize " + JSON.stringify(exp));
    }

    ... // the opt_* functions here
}
```

我们不必关心“ let”节点，因为在CPS变压器之后，它们将被还原为IIFE-s。

“平凡”的节点（常量和变量）照原样返回。

每个opt_ *函数都必须在当前表达式的成员上递归调用opt。

opt_binary将优化二进制表达式：在左右节点上调用opt，然后检查它们是否为常数，如果是，则对表达式求值并替换为结果。

opt_assign也将优化左节点和右节点。 然后检查潜在的减少量。

opt_if进入cond，然后else。 当可以确定条件（是常量表达式）时，它会适当地返回then或else。

opt_prog将复制CPS转换器中已经完成的一些工作，例如丢弃没有副作用的表达式； 在此执行此操作很有用，因为优化会级联（因此，即使to_cps返回了严格的代码，经过一些优化后，我们最终还是可能在此处得到无副作用的表达式）。

opt_call优化函数调用。 如果我们调用的是匿名“ lambda”（IIFE），则将其定向到opt_iife（请参见代码），它将把该函数的参数和主体展开为当前程序。 这可能是我们做的最棘手（也是最有效）的优化。

opt_lambda将优化一个函数，该函数的唯一目的是使用相同的参数调用另一个函数。 如果不是这种情况，它将丢弃loc中未使用的局部变量，并使主体下降。

