因此，我们的评估程序现在以“连续传递样式”实现，并且我们所有的函数（无论是在新的λanguage中定义还是在JavaScript中定义的原语）都将第一个参数作为结果调用的回调（尽管该参数是显式的）对于图元，但对于λ语言中定义的函数不可见。



该回调代表程序的整个未来。下一步工作的完整表示。我们将其称为“当前延续”，并且在代码中将名称k用作该变量。



一方面，如果我们从不调用延续，那么程序将停止。我们不能从λ语言中做到这一点，因为make_lambda创建的函数总是调用它们的延续。但是我们可以从原始中做到这一点。我在此页面中包括以下原始内容以说明这一点：

```js
globalEnv.def("halt", function(k){});
```

这是什么都不做的功能。它收到一个延续（k），但没有调用它。

```js
println("foo");
halt();
println("bar");

```

Run result:

```
foo
```

如果删除halt（），则输出为foo / bar / ***结果：false（因为最后一个println返回false）。但是使用halt（），输出仅为“ foo”。在这种情况下甚至没有结果，因为halt（）不会调用延续—因此，我们传递给要评估的回调（即显示*** Result行的回调）未到达。调用者函数从不会注意到程序已停止。如果您正在使用NodeJS，则可能已经用这把枪开了枪。



假设我们要编写一个“睡眠”功能，该功能可以延迟程序，但不会冻结浏览器（因此，不会产生脏循环）。我们可以简单地将其实现为原语：

```
globalEnv.def("sleep", function(k, milliseconds){
  setTimeout(function(){
    Execute(k, [ false ]); // continuations expect a value, pass false
  }, milliseconds);
});
```



一个不便之处是我们必须使用Execute，因为setTimeout将放弃当前的堆栈帧。直接调用k（false）不会包装在我们的循环中，并且在第一个Continuation异常上失败。这是我们的用法：（也请尝试单击“快速运行”几次，以查看调用是“并行”运行的）：

```
let loop (n = 0) {
  if n < 10 {
    println(n);
    sleep(250);
    loop(n + 1);
  }
};
println("And we're done");
```

Run result:

```
0
1
2
3
4
5
6
7
8
9
And we're done
***Result: false
```

那是用普通的JavaScript无法做到的，除了通过手动以连续传递样式重写代码（并且不能使用for循环）之外：

```
(function loop(n){
  if (n < 10) {
    console.log(n);
    setTimeout(function(){
      loop(n + 1);
    }, 250);
  } else {
    println("And we're done"); // rest of the program goes here.
  }
})(0);
```





想象一下围绕NodeJS文件系统API的一些原语，即：

```
globalEnv.def("readFile", function(k, filename){
  fs.readFile(filename, function(err, data){
    // error handling is a bit more complex, ignoring for now
    Execute(k, [ data ]); // hope it's clear why we need the Execute
  });
});
globalEnv.def("writeFile", function(k, filename, data){
  fs.writeFile(filename, data, function(err){
    Execute(k, [ false ]);
  });
});
```

有了它们，我们可以在λanguage中做到这一点：