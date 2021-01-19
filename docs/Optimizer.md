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
```
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
```
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
```
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

```
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
```
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
```
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
```
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
```
(λ(){
  let (a = 2, b = 5) {
    print(a + b);
  };
})();
```
Transformation:
```
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
```
(function β_CC(β_K1) {
  var a, b;
  GUARD(arguments, β_CC);
  a = 2, b = 5, print(β_K1, a + b);
})(β_TOPLEVEL);
```
没有名称冲突，因此a和b变量不会重命名。 如您所见，to_cps通过将其编译为IIFE-s来增加了一个简单的let的开销，但是优化器能够丢弃该问题。

Transformation

```
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
```

```
(function β_CC(β_K1) {
  var a, β_K3, β_a$9;
  GUARD(arguments, β_CC);
  a = 2, β_K3 = function(β_R5) {
    print(β_K1, a);
  }, β_a$9 = 3, print(β_K3, β_a$9);
})(β_TOPLEVEL);

```