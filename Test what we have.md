## What did we get

Our λanguage so far, although small, is (theoretically) capable to solve any problem that can be solved computationally. That's because some guys smarter than I'll ever be — Alonzo Church and Alan Turing — proved in a more civilized age that λ-calculus is equivalent to the Turing machine, and our λanguage models λ-Calculus.

What this really means is that even if our λanguage lacks certain features, we can implement them in terms of what we have, or if that proves too difficult, we can write an interpreter for another language in it.

### Loops

Loops are non-essential when we have recursion. I already shown a sample that loops via recursion. Let's try it. Note, you can edit the code in these samples, and run it with the dimmed "Run" button in the top-right (also: CTRL-ENTER to run, CTRL-L to clear output — these work while the cursor is in the code area on the left); use print / println for output. Feel free to play around, it's your browser who runs it.

```js
print_range = λ(a, b) if a <= b {
                        print(a);
                        if a + 1 <= b {
                          print(", ");
                          print_range(a + 1, b);
                        } else println("");
                      };
print_range(1, 10);

```
However, there is a problem if we increase the range to, say, 1000 instead of 10. I get “Maximum call stack size exceeded” after 600 (and something) is printed. That's because our evaluator is recursive and will exhaust the JavaScript stack.

This is one serious problem, but there are solutions. It might seem tempting to add some iteration keywords, like for or while, but let's not. Recursion is beautiful, we don't have to give it up. We'll see later how we can workaround this limitation.

### Data structures (or lack thereof)
Our λanguage appears to have three types: number, string, and boolean. It might seem impossible to create compound structures such as objects or lists. But in fact, we do have one more type: the function. It turns out that in λ-calculus we can use functions to construct any data structure, including objects with inheritance.

#### Here I'll show lists
Let's wish we had a function named cons which constructs a special object holding two values; let's call that object “cons cell” or “pair”. We'll name one of these values “car”, and the other “cdr”, because that's how they've been named in Lisp for decades. And given a cell object, we can use the functions car and cdr to retrieve the respective values from the pair. Therefore:

```
x = cons(10, 20);
print(car(x));    # prints 10
print(cdr(x));    # prints 20
```

Given these, it's easy to define a list:

  > A list is a cell object which has the first element in its car, and the rest of the elements in its cdr. But the cdr can store a single value! That value is a list. A list is a cell object which has the first element in its car, and the rest of the elements in its cdr. But the cdr can store a single value! That value is a list. […]

So it's a recursive data structure (defined in terms of itself). A single problem remains: when do we stop? Intuitively, we stop when the cdr is the empty list, but what is the empty list? For this we'll introduce a new special object called NIL. It can be treated as a cell (we can apply car and cdr on it, but the result is itself) — but it's not a real cell… let's just say it's a fake one. Then, here's how to construct the list 1, 2, 3, 4, 5:

```
x = cons(1, cons(2, cons(3, cons(4, cons(5, NIL)))));
print(car(x));                      # 1
print(car(cdr(x)));                 # 2  in Lisp this is abbrev. cadr
print(car(cdr(cdr(x))));            # 3                          caddr
print(car(cdr(cdr(cdr(x)))));       # 4                          cadddr
print(car(cdr(cdr(cdr(cdr(x))))));  # 5  but no abbreviation for this one.
```

It looks awful when the proper syntax is missing. But I just wanted to show that it's possible to build such a data structure in our seemingly limited λanguage. Here are the definitions:

```
cons = λ(a, b) λ(f) f(a, b);
car = λ(cell) cell(λ(a, b) a);
cdr = λ(cell) cell(λ(a, b) b);
NIL = λ(f) f(NIL, NIL);
```

When I first saw cons/car/cdr implemented this way I found intimidating the fact that they don't even need if (but that's not so surprising, since there is no if in the original λ-calculus). Of course, no language does it like this in practice because it's very inefficient, but that doesn't make it less beautiful. In English, the above code states:

- cons takes two values (a, b) and returns a function that closes over them. That function is the “cell object”. It takes a function argument (f) and calls it with both the values it stored.

- car takes a “cell object” (so, that function) and calls it with a function that receives two arguments and returns the first one.

- cdr is like car, but the function it sends to the cell returns the second argument.

- NIL mimics a cell, in that it's a function which takes one function argument (f) but always calls it with two NIL-s (so, both car(NIL) and cdr(NIL) will equal NIL).


```
cons = λ(a, b) λ(f) f(a, b);
car = λ(cell) cell(λ(a, b) a);
cdr = λ(cell) cell(λ(a, b) b);
NIL = λ(f) f(NIL, NIL);

x = cons(1, cons(2, cons(3, cons(4, cons(5, NIL)))));
println(car(x));                      # 1
println(car(cdr(x)));                 # 2
println(car(cdr(cdr(x))));            # 3
println(car(cdr(cdr(cdr(x)))));       # 4
println(car(cdr(cdr(cdr(cdr(x))))));  # 5

```
There are many interesting list algorithms that can be implemented recursively and it's very natural, since the structure itself is recursively defined. For example here's a function that applies a function to every element of a list:

```
foreach = λ(list, f)
            if list != NIL {
              f(car(list));
              foreach(cdr(list), f);
            };
foreach(x, println);

```
And here's another one that builds a list with a range of numbers:

```
range = λ(a, b)
          if a <= b then cons(a, range(a + 1, b))
                    else NIL;

# print the squares of 1..8
foreach(range(1, 8), λ(x) println(x * x));

```

The lists we have so far are immutable (you cannot alter the car nor the cdr once a cell is created). Most Lisps provide some way to alter a cons cell. In Scheme it's called set-car! / set-cdr!. In Common Lisp they're suggestively named rplaca / rplacd. We'll prefer the Scheme names this time:

```
cons = λ(x, y)
         λ(a, i, v)
           if a == "get"
              then if i == 0 then x else y
              else if i == 0 then x = v else y = v;

car = λ(cell) cell("get", 0);
cdr = λ(cell) cell("get", 1);
set-car! = λ(cell, val) cell("set", 0, val);
set-cdr! = λ(cell, val) cell("set", 1, val);

# NIL can be a real cons this time
NIL = cons(0, 0);
set-car!(NIL, NIL);
set-cdr!(NIL, NIL);

## test:
x = cons(1, 2);
println(car(x));
println(cdr(x));
set-car!(x, 10);
set-cdr!(x, 20);
println(car(x));
println(cdr(x));

```

This shows that it's possible to implement mutable compound data structures. I won't comment on how it works, it's pretty straightforward.

We could move forward and implement objects, but the inability to modify the syntax from within the λanguage will make it very awkward. The other option is to introduce new syntax in the tokenizer/parser, and maybe add new semantics in the evaluator. That's what all the mainstream languages do, and it's many times a necessary move in order to get acceptable performance. We'll explore adding some new syntax in the next section.
