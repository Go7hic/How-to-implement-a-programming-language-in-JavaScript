## Adding new constructs

Our language is a little too scarce. For example there is no way to introduce new variables. As I mentioned, we need to use an IIFE, so it'll look like this (silly example, can't think of anything better):

```
(λ(x, y){
  (λ(z){ ## it gets worse when one of the vars depends on another
    print(x + y + z);
  })(x + y);
})(2, 3);
```


We'd like to add a let keyword that will allow us to write it like this:

```
let (x = 2, y = 3, z = x + y) print(x + y + z);
```

For each of the definitions, variables that were previously defined become available. That let could be translated like this:

```
(λ(x){
  (λ(y){
    (λ(z){
      print(x + y + z);
    })(x + y);
  })(3);
})(2);
```

This transformation can be handled directly by the parser and it would require no changes in the evaluator. Instead of adding a dedicated "let" node, we can transform the whole thing into "call" and "lambda" nodes. What this means is that we're not adding any semantic power to the language — it's called “syntactic sugar”, and the operation of transforming it into AST nodes that are already known is called “desugaring”.

However, since we are going to modify the parser anyway, let's add a dedicated "let" node because it can be evaluated more efficiently (there is no need to create closures and call them immediately; we only need to extend the scope).

We will additionally support a “named let” syntax, which was popularized by Scheme. It allows us to define looping expressions in an easier way:

```
print(let loop (n = 10)
        if n > 0 then n + loop(n - 1)
                 else 0);
```

That's a recursive loop that computes the sum 10 + 9 + ... + 0. In the language we have so far we would have to write it like this:

```
print((λ(loop){
         loop = λ(n) if n > 0 then n + loop(n - 1)
                              else 0;
         loop(10);
       })());
```


To make it easier, we'll add the concept of “named functions”. So we'll also be able to write it like this:

```
print((λ loop (n) if n > 0 then n + loop(n - 1)
                           else 0)
      (10));
      
```

So to recap, the modifications we need to make are:

 - Support an optional name after the lambda keyword. If it's present then the environment in which the function runs must define that name to point to the function itself. That's exactly like named function expressions in JavaScript.

 - Support a new let keyword. Following it there's an optional name and a parenthesized list (possibly empty) of variable definitions in the form foo = EXPRESSION, separated by commas. The body of a let is a single expression (which can be a {group}, obviously).
 
 #### Parser changes
 
 First, a very small change in the tokenizer, we need to add let to the list of keywords:
 
 ```
 var keywords = " let if then else lambda λ true false ";
 ```
 
 We have to modify parse_lambda in the parser to account for the optional function name:
 
 ```
 function parse_lambda() {
    return {
        type: "lambda",
        name: input.peek().type == "var" ? input.next().value : null, // this line
        vars: delimited("(", ")", ",", parse_varname),
        body: parse_expression()
    };
}
```

Let's now define the let parser:

```
function parse_let() {
    skip_kw("let");
    if (input.peek().type == "var") {
        var name = input.next().value;
        var defs = delimited("(", ")", ",", parse_vardef);
        return {
            type: "call",
            func: {
                type: "lambda",
                name: name,
                vars: defs.map(function(def){ return def.name }),
                body: parse_expression(),
            },
            args: defs.map(function(def){ return def.def || FALSE })
        };
    }
    return {
        type: "let",
        vars: delimited("(", ")", ",", parse_vardef),
        body: parse_expression(),
    };
}
```

It handles both cases. If following the let we have a "var" token, then it's a named let. We read the definitions with delimited, as they're in parens and separated by commas, and use a parse_vardef helper which is defined below. Then we return a "call" node which calls a named function expression (so the whole result node is an IIFE). The function's argument names are the variables defined in let, and the "call" will take care to send the values in args. And the body of the function is, of course, fetched with parse_expression().

If it's not a named let then we return a "let" node, having vars and body. The vars contain { name: VARIABLE, def: AST }, as parsed from the let definition list by the following function:

```
function parse_vardef() {
    var name = parse_varname(), def;
    if (is_op("=")) {
        input.next();
        def = parse_expression();
    }
    return { name: name, def: def };
}
```


We also need to account for the possibility of encountering a let keyword in the dispatcher (parse_atom), so we're adding this line:

```
// can be before the one handling parse_if
if (is_kw("let")) return parse_let();
```

### Evaluator changes
 Since we opted to modify the AST instead of desugaring into known AST nodes, we have to update the evaluator to take the changes into account.

In order to support the optional function name, the make_lambda function becomes:

```
function make_lambda(env, exp) {
    if (exp.name) {                    // these
        env = env.extend();            // lines
        env.def(exp.name, lambda);     // are
    }                                  // new
    function lambda() {
        var names = exp.vars;
        var scope = env.extend();
        for (var i = 0; i < names.length; ++i)
            scope.def(names[i], i < arguments.length ? arguments[i] : false);
        return evaluate(exp.body, scope);
    }
    return lambda;
}
```

If the function name is present, then we extend the scope right when the closure is created and define the name to point to the newly created closure. The rest remains unchanged.

Finally, to handle the "let" AST we add the following case in the evaluator:

```
case "let":
  exp.vars.forEach(function(v){
      var scope = env.extend();
      scope.def(v.name, v.def ? evaluate(v.def, env) : false);
      env = scope;
  });
  return evaluate(exp.body, env);
```

Note it extends the scope for each variable, defining the variable by evaluating the initialization code, if any. Then just evaluate the let body.

```
println(let loop (n = 100)
          if n > 0 then n + loop(n - 1)
                   else 0);

let (x = 2, y = x + 1, z = x + y)
  println(x + y + z);

# errors out, the vars are bound to the let body
# print(x + y + z);

let (x = 10) {
  let (x = x * 2, y = x * x) {
    println(x);  ## 20
    println(y);  ## 400
  };
  println(x);  ## 10
};

```


#### Conclusion


It wasn't too hard to make these changes, since we wrote the whole language ourselves. But what if it were written by someone else and we didn't have the sources; moreover, if it was a standard language that everybody had installed, and it would expect programs to conform to a certain (rigid) syntax? That language is JavaScript. Imagine you wanted to add a new syntactic construct to JavaScript — can you do it? Not a chance, even if you can propose it to the big guys who maintain JS, it would take years for your proposal to be considered, accepted, standardized and widely implemented. In JavaScript, we're prisoners (unless we implement our own language on top of it).

I told you I was going to argue why Lisp is a great language: in Lisp we can add syntactic constructs without changing the actual implementation of the parser/evaluator/compiler, thus making our lifes easier without imposing those constructs to all the users of the language and certainly without waiting for years for some committee to approve our ideas and others to implement it. Lisp is freedom. If Lisp didn't have named functions or a let construct, we could add them in less code than was needed above, and they would look and feel like built-in constructs.
