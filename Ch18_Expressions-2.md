---
title: "Ch18_Expressions-2"
author: "Min-Yao"
date: "2023-07-12"
output: 
  html_document: 
    keep_md: yes
---

# 18 Expressions


```r
library(rlang)
library(lobstr)
```

## 18.4 Parsing and grammar {#grammar}
\index{grammar}

We've talked a lot about expressions and the AST, but not about how expressions are created from code that you type (like `"x + y"`). 

> The process by which a computer language takes a string and constructs an expression is called __parsing__, and is governed by a set of rules known as a __grammar__. In this section, we'll use `lobstr::ast()` to explore some of the details of R's grammar, and then show how you can transform back and forth between expressions and strings.

### 18.4.1 Operator precedence
\index{operator precedence}

Infix functions introduce two sources of ambiguity[^ambig]. The first source of ambiguity arises from infix functions: what does `1 + 2 * 3` yield? Do you get 9 (i.e. `(1 + 2) * 3`), or 7 (i.e. `1 + (2 * 3)`)? In other words, which of the two possible parse trees below does R use?

[^ambig]: This ambiguity does not exist in languages with only prefix or postfix calls. It's interesting to compare a simple arithmetic operation in Lisp (prefix) and Forth (postfix). In Lisp you'd write `(* (+ 1 2) 3))`; this avoids ambiguity by requiring parentheses everywhere. In Forth, you'd write `1 2 + 3 *`; this doesn't require any parentheses, but does require more thought when reading.

<img src="diagrams/expressions/ambig-order.png" width="933" />

Programming languages use conventions called __operator precedence__ to resolve this ambiguity. We can use `ast()` to see what R does:


```r
lobstr::ast(1 + 2 * 3)
```

```
## █─`+` 
## ├─1 
## └─█─`*` 
##   ├─2 
##   └─3
```

Predicting the precedence of arithmetic operations is usually easy because it's drilled into you in school and is consistent across the vast majority of programming languages. 

Predicting the precedence of other operators is harder. There's one particularly surprising case in R: `!` has a much lower precedence (i.e. it binds less tightly) than you might expect. This allows you to write useful operations like:


```r
lobstr::ast(!x %in% y)
```

```
## █─`!` 
## └─█─`%in%` 
##   ├─x 
##   └─y
```

R has over 30 infix operators divided into 18 precedence groups. While the details are described in `?Syntax`, very few people have memorised the complete ordering. If there's any confusion, use parentheses!


```r
lobstr::ast((1 + 2) * 3)
```

```
## █─`*` 
## ├─█─`(` 
## │ └─█─`+` 
## │   ├─1 
## │   └─2 
## └─3
```

Note the appearance of the parentheses in the AST as a call to the `(` function.

### 18.4.2 Associativity

The second source of ambiguity is introduced by repeated usage of the same infix function. For example, is `1 + 2 + 3` equivalent to `(1 + 2) + 3` or to `1 + (2 + 3)`?  This normally doesn't matter because `x + (y + z) == (x + y) + z`, i.e. addition is associative, but is needed because some S3 classes define `+` in a non-associative way. For example, ggplot2 overloads `+` to build up a complex plot from simple pieces; this is non-associative because earlier layers are drawn underneath later layers (i.e. `geom_point()` + `geom_smooth()` does not yield the same plot as `geom_smooth()` + `geom_point()`).

In R, most operators are __left-associative__, i.e. the operations on the left are evaluated first:


```r
lobstr::ast(1 + 2 + 3)
```

```
## █─`+` 
## ├─█─`+` 
## │ ├─1 
## │ └─2 
## └─3
```

There are two exceptions: exponentiation and assignment.


```r
lobstr::ast(2^2^3)
```

```
## █─`^` 
## ├─2 
## └─█─`^` 
##   ├─2 
##   └─3
```

```r
lobstr::ast(x <- y <- z)
```

```
## █─`<-` 
## ├─x 
## └─█─`<-` 
##   ├─y 
##   └─z
```

### 18.4.3 Parsing and deparsing {#parsing}
\index{parsing}
\indexc{parsing!parse\_expr@\texttt{parse\_expr()}}

Most of the time you type code into the console, and R takes care of turning the characters you've typed into an AST. But occasionally you have code stored in a string, and you want to parse it yourself. You can do so using `rlang::parse_expr()`:


```r
x1 <- "y <- x + 10"
x1
```

```
## [1] "y <- x + 10"
```

```r
is.call(x1)
```

```
## [1] FALSE
```

```r
x2 <- rlang::parse_expr(x1)
x2
```

```
## y <- x + 10
```

```r
is.call(x2)
```

```
## [1] TRUE
```

`parse_expr()` always returns a single expression. If you have multiple expression separated by `;` or `\n`, you'll need to use `rlang::parse_exprs()`. It returns a list of expressions:


```r
x3 <- "a <- 1; a + 1"
rlang::parse_exprs(x3)
```

```
## [[1]]
## a <- 1
## 
## [[2]]
## a + 1
```

If you find yourself working with strings containing code very frequently, you should reconsider your process. Read Chapter \@ref(quasiquotation) and consider whether you can generate expressions using quasiquotation more safely.

::: base
\indexc{parsing!parse@\texttt{parse()}}
The base equivalent to `parse_exprs()` is `parse()`. It is a little harder to use because it's specialised for parsing R code stored in files. You need to supply your string to the `text` argument and it returns an expression vector (Section \@ref(expression-vectors)). I recommend turning the output into a list:


```r
as.list(parse(text = x1))
```

```
## [[1]]
## y <- x + 10
```
:::

\index{deparsing}
\indexc{expr\_text()}

The inverse of parsing is __deparsing__: given an expression, you want the string that would generate it. This happens automatically when you print an expression, and you can get the string with `rlang::expr_text()`:


```r
z <- expr(y <- x + 10)
expr_text(z)
```

```
## [1] "y <- x + 10"
```

Parsing and deparsing are not perfectly symmetric because parsing generates an _abstract_ syntax tree. This means we lose backticks around ordinary names, comments, and whitespace:


```r
cat(expr_text(expr({
  # This is a comment
  x <-             `x` + 1
})))
```

```
## {
##     x <- x + 1
## }
```

::: base
\indexc{deparse()}
Be careful when using the base R equivalent, `deparse()`: it returns a character vector with one element for each line. Whenever you use it, remember that the length of the output might be greater than one, and plan accordingly.
:::

### 18.4.4 Exercises

1.  R uses parentheses in two slightly different ways as illustrated by
    these two calls:

    
    ```r
    f((1))
    `(`(1 + 1)
    ```

    Compare and contrast the two uses by referencing the AST.
    

```r
ast(f((1)))
```

```
## █─f 
## └─█─`(` 
##   └─1
```

> In the AST of the first example, the outer ( is not visible since it is part of the prefix function syntax for f(). However, the inner ( represents a function and is displayed as a symbol in the AST.


```r
ast(`(`(1 + 1))
```

```
## █─`(` 
## └─█─`+` 
##   ├─1 
##   └─1
```

> In the second example, we can observe that the outer ( is a function, while the inner ( is associated with its syntax.

1.  `=` can also be used in two ways. Construct a simple example that shows
    both uses.
    
> The symbol = serves a dual purpose in R, both for assignment and for naming arguments in function calls:


```r
a = c(b = 100)
a
b
```

However, when working with ast(), a direct attempt like the following results in an error:


```r
ast(a = c(b = 100))
```

> The error arises because b = prompts R to search for an argument named b. Since x is the only argument of ast(), an error is raised.

To overcome this issue, the simplest approach is to enclose the problematic line within braces:


```r
ast({a = c(b = 100)})
```

```
## █─`{` 
## └─█─`=` 
##   ├─a 
##   └─█─c 
##     └─b = 100
```

> When we disregard the braces and compare the trees, it becomes apparent that the first = symbol is utilized for assignment, while the second = is a component of the function call syntax.

1.  Does `-2^2` yield 4 or -4? Why?


```r
-2^2
```

```
## [1] -4
```

```r
ast(-2^2)
```

```
## █─`-` 
## └─█─`^` 
##   ├─2 
##   └─2
```
> The outcome obtained is -4 due to the higher precedence of the ^ operator over -. 

1.  What does `!1 + !1` return? Why?


```r
!1 + !1
```

```
## [1] FALSE
```

```r
ast(!1 + !1)
```

```
## █─`!` 
## └─█─`+` 
##   ├─1 
##   └─█─`!` 
##     └─1
```

> The evaluation process unfolds as follows: 

> First, !1 on the right-hand side is assessed. It yields FALSE since R coerces any non-zero numeric value to TRUE when a logical operator is applied. The negation of TRUE results in FALSE.

> Subsequently, 1 + FALSE is evaluated, giving the value 1 because FALSE is coerced to 0.

> Finally, !1 is evaluated, yielding FALSE.

> It is worth noting that if ! had a higher precedence, the intermediate result would be FALSE + FALSE, which would evaluate to 0.

1.  Why does `x1 <- x2 <- x3 <- 0` work? Describe the two reasons.


```r
x1 <- x2 <- x3 <- 0
x1
```

```
## [1] 0
```

```r
ast(x1 <- x2 <- x3 <- 0)
```

```
## █─`<-` 
## ├─x1 
## └─█─`<-` 
##   ├─x2 
##   └─█─`<-` 
##     ├─x3 
##     └─0
```

> There are two reasons for this behavior. First, the <- operator in R is right-associative, meaning that evaluation occurs from right to left.

> Secondly, the <- operator invisibly returns the value on the right-hand side.


```r
(x3 <- 0)
```

```
## [1] 0
```

> As a result, the assignment operation can be nested in this manner.

1.  Compare the ASTs of `x + y %+% z` and `x ^ y %+% z`. What have you learned
    about the precedence of custom infix functions?
    

```r
ast(x + y %+% z)
```

```
## █─`+` 
## ├─x 
## └─█─`%+%` 
##   ├─y 
##   └─z
```

> In this case, the expression y %+% z will be evaluated first, and the result will be added to x.



```r
ast(x ^ y %+% z)
```

```
## █─`%+%` 
## ├─█─`^` 
## │ ├─x 
## │ └─y 
## └─z
```

> Here, x ^ y will be calculated first, and the resulting value will serve as the first argument to %+%().

> From these examples, we can deduce that custom infix functions hold precedence between addition and exponentiation operations.

1.  What happens if you call `parse_expr()` with a string that generates
    multiple expressions? e.g. `parse_expr("x + 1; y + 1")`
    

```r
parse_expr("x + 1; y + 1")
```

```
## Error in `parse_expr()`:
## ! `x` must contain exactly 1 expression, not 2.
```

1.  What happens if you attempt to parse an invalid expression? e.g. `"a +"`
    or `"f())"`.
    

```r
parse_expr("a +")
```

```
## Error in parse(text = x, keep.source = FALSE): <text>:2:0: unexpected end of input
## 1: a +
##    ^
```

```r
parse_expr("f())")
```

```
## Error in parse(text = x, keep.source = FALSE): <text>:1:4: unexpected ')'
## 1: f())
##        ^
```

```r
parse(text = "a +")
```

```
## Error in parse(text = "a +"): <text>:2:0: unexpected end of input
## 1: a +
##    ^
```

```r
parse(text = "f())")
```

```
## Error in parse(text = "f())"): <text>:1:4: unexpected ')'
## 1: f())
##        ^
```

1.  `deparse()` produces vectors when the input is long. For example, the
    following call produces a vector of length two:

    
    ```r
    expr <- expr(g(a + b + c + d + e + f + g + h + i + j + k + l + 
      m + n + o + p + q + r + s + t + u + v + w + x + y + z))
    
    deparse(expr)
    ```

    What does `expr_text()` do instead?
    

```r
expr_text(expr)
```

```
## [1] "function (expr) \n{\n    enexpr(expr)\n}"
```

> The function expr_text() concatenates the outcomes obtained from deparse(expr) and employs a line break (\n) as the separator between them.

1.  `pairwise.t.test()` assumes that `deparse()` always returns a length one
    character vector. Can you construct an input that violates this expectation?
    What happens?



```r
# R version 4.3.0

d <- 1
pairwise.t.test(2, d + d + d + d + d + d + d + d + 
                  d + d + d + d + d + d + d + d + d)
```

```
## 
## 	Pairwise comparisons using t tests with pooled SD 
## 
## data:  2 and d + d + d + d + d + d + d + d + d + d + d + d + d + d + d + d + d 
## 
## <0 x 0 matrix>
## 
## P value adjustment method: holm
```

> To address potential unexpected output caused by exceeding the default width.cutoff value of 60 characters in deparse(), pairwise.t.test() in versions prior to R 4.0.0 utilized deparse(substitute(x)) in conjunction with paste(). This approach could result in the splitting of the expression into a character vector of length greater than 1.

> However, starting from R 4.0.0, pairwise.t.test() was updated to utilize the newly introduced deparse1() function, acting as a wrapper around deparse(). This change ensures that the result is a single string (character vector of length one), primarily used in name construction.

## 18.5 Walking AST with recursive functions {#ast-funs}
\index{recursion!over ASTs}
\index{ASTs!computing with}

To conclude the chapter I'm going to use everything you've learned about ASTs to solve more complicated problems. The inspiration comes from the base codetools package, which provides two interesting functions:

* `findGlobals()` locates all global variables used by a function. This
  can be useful if you want to check that your function doesn't inadvertently
  rely on variables defined in their parent environment.

* `checkUsage()` checks for a range of common problems including
  unused local variables, unused parameters, and the use of partial
  argument matching.

Getting all of the details of these functions correct is fiddly, so we won't fully develop the ideas. Instead we'll focus on the big underlying idea: recursion on the AST. Recursive functions are a natural fit to tree-like data structures because a recursive function is made up of two parts that correspond to the two parts of the tree:

* The __recursive case__ handles the nodes in the tree. Typically, you'll
  do something to each child of a node, usually calling the recursive function
  again, and then combine the results back together again. For expressions,
  you'll need to handle calls and pairlists (function arguments).

* The __base case__ handles the leaves of the tree. The base cases ensure
  that the function eventually terminates, by solving the simplest cases
  directly. For expressions, you need to handle symbols and constants in the
  base case.

To make this pattern easier to see, we'll need two helper functions. First we define `expr_type()` which will return "constant" for constant, "symbol" for symbols, "call", for calls, "pairlist" for pairlists, and the "type" of anything else:


```r
expr_type <- function(x) {
  if (rlang::is_syntactic_literal(x)) {
    "constant"
  } else if (is.symbol(x)) {
    "symbol"
  } else if (is.call(x)) {
    "call"
  } else if (is.pairlist(x)) {
    "pairlist"
  } else {
    typeof(x)
  }
}

expr_type(expr("a"))
```

```
## [1] "constant"
```

```r
expr_type(expr(x))
```

```
## [1] "symbol"
```

```r
expr_type(expr(f(1, 2)))
```

```
## [1] "call"
```

We'll couple this with a wrapper around the switch function:


```r
switch_expr <- function(x, ...) {
  switch(expr_type(x),
    ...,
    stop("Don't know how to handle type ", typeof(x), call. = FALSE)
  )
}
```

With these two functions in hand, we can write a basic template for any function that walks the AST using `switch()` (Section \@ref(switch)):


```r
recurse_call <- function(x) {
  switch_expr(x,
    # Base cases
    symbol = ,
    constant = ,

    # Recursive cases
    call = ,
    pairlist =
  )
}
```

Typically, solving the base case is easy, so we'll do that first, then check the results. The recursive cases are trickier, and will often require some functional programming.

### 18.5.1 Finding F and T

We'll start with a function that determines whether another function uses the logical abbreviations `T` and `F` because using them is often considered to be poor coding practice. Our goal is to return `TRUE` if the input contains a logical abbreviation, and `FALSE` otherwise. 

Let's first find the type of `T` versus `TRUE`:


```r
expr_type(expr(TRUE))
```

```
## [1] "constant"
```

```r
expr_type(expr(T))
```

```
## [1] "symbol"
```

`TRUE` is parsed as a logical vector of length one, while `T` is parsed as a name. This tells us how to write our base cases for the recursive function: a constant is never a logical abbreviation, and a symbol is an abbreviation if it's "F" or "T":


```r
logical_abbr_rec <- function(x) {
  switch_expr(x,
    constant = FALSE,
    symbol = as_string(x) %in% c("F", "T")
  )
}

logical_abbr_rec(expr(TRUE))
```

```
## [1] FALSE
```

```r
logical_abbr_rec(expr(T))
```

```
## [1] TRUE
```

I've written `logical_abbr_rec()` function assuming that the input will be an expression as this will make the recursive operation simpler. However, when writing a recursive function it's common to write a wrapper that provides defaults or makes the function a little easier to use. Here we'll typically make a wrapper that quotes its input (we'll learn more about that in the next chapter), so we don't need to use `expr()` every time.


```r
logical_abbr <- function(x) {
  logical_abbr_rec(enexpr(x))
}

logical_abbr(T)
```

```
## [1] TRUE
```

```r
logical_abbr(FALSE)
```

```
## [1] FALSE
```

Next we need to implement the recursive cases. Here we want to do the same thing for calls and for pairlists: recursively apply the function to each subcomponent, and return `TRUE` if any subcomponent contains a logical abbreviation. This is made easy by `purrr::some()`, which iterates over a list and returns `TRUE` if the predicate function is true for any element.


```r
logical_abbr_rec <- function(x) {
  switch_expr(x,
    # Base cases
    constant = FALSE,
    symbol = as_string(x) %in% c("F", "T"),

    # Recursive cases
    call = ,
    pairlist = purrr::some(x, logical_abbr_rec)
  )
}

logical_abbr(mean(x, na.rm = T))
```

```
## [1] TRUE
```

```r
logical_abbr(function(x, na.rm = T) FALSE)
```

```
## [1] TRUE
```

### 18.5.2 Finding all variables created by assignment

`logical_abbr()` is relatively simple: it only returns a single `TRUE` or `FALSE`. The next task, listing all variables created by assignment, is a little more complicated. We'll start simply, and then make the function progressively more rigorous. \indexc{find\_assign()}

We start by looking at the AST for assignment:


```r
ast(x <- 10)
```

```
## █─`<-` 
## ├─x 
## └─10
```

Assignment is a call object where the first element is the symbol `<-`, the second is the name of variable, and the third is the value to be assigned.

Next, we need to decide what data structure we're going to use for the results. Here I think it will be easiest if we return a character vector. If we return symbols, we'll need to use a `list()` and that makes things a little more complicated.

With that in hand we can start by implementing the base cases and providing a helpful wrapper around the recursive function. Here the base cases are straightforward because we know that neither a symbol nor a constant represents assignment.


```r
find_assign_rec <- function(x) {
  switch_expr(x,
    constant = ,
    symbol = character()
  )
}
find_assign <- function(x) find_assign_rec(enexpr(x))

find_assign("x")
```

```
## character(0)
```

```r
find_assign(x)
```

```
## character(0)
```

Next we implement the recursive cases. This is made easier by a function that should exist in purrr, but currently doesn't. `flat_map_chr()` expects `.f` to return a character vector of arbitrary length, and flattens all results into a single character vector.

<!-- GVW: by this point, will readers have seen the `.x` and `.f` conventions enough that they don't need explanation? -->


```r
flat_map_chr <- function(.x, .f, ...) {
  purrr::flatten_chr(purrr::map(.x, .f, ...))
}

flat_map_chr(letters[1:3], ~ rep(., sample(3, 1)))
```

```
## [1] "a" "a" "b" "c" "c"
```

The recursive case for pairlists is straightforward: we iterate over every element of the pairlist (i.e. each function argument) and combine the results. The case for calls is a little bit more complex: if this is a call to `<-` then we should return the second element of the call:


```r
find_assign_rec <- function(x) {
  switch_expr(x,
    # Base cases
    constant = ,
    symbol = character(),

    # Recursive cases
    pairlist = flat_map_chr(as.list(x), find_assign_rec),
    call = {
      if (is_call(x, "<-")) {
        as_string(x[[2]])
      } else {
        flat_map_chr(as.list(x), find_assign_rec)
      }
    }
  )
}

find_assign(a <- 1)
```

```
## [1] "a"
```

```r
find_assign({
  a <- 1
  {
    b <- 2
  }
})
```

```
## [1] "a" "b"
```

Now we need to make our function more robust by coming up with examples intended to break it. What happens when we assign to the same variable multiple times?


```r
find_assign({
  a <- 1
  a <- 2
})
```

```
## [1] "a" "a"
```

It's easiest to fix this at the level of the wrapper function:


```r
find_assign <- function(x) unique(find_assign_rec(enexpr(x)))

find_assign({
  a <- 1
  a <- 2
})
```

```
## [1] "a"
```

What happens if we have nested calls to `<-`? Currently we only return the first. That's because when `<-` occurs we immediately terminate recursion.


```r
find_assign({
  a <- b <- c <- 1
})
```

```
## [1] "a"
```

Instead we need to take a more rigorous approach. I think it's best to keep the recursive function focused on the tree structure, so I'm going to extract out `find_assign_call()` into a separate function.


```r
find_assign_call <- function(x) {
  if (is_call(x, "<-") && is_symbol(x[[2]])) {
    lhs <- as_string(x[[2]])
    children <- as.list(x)[-1]
  } else {
    lhs <- character()
    children <- as.list(x)
  }

  c(lhs, flat_map_chr(children, find_assign_rec))
}

find_assign_rec <- function(x) {
  switch_expr(x,
    # Base cases
    constant = ,
    symbol = character(),

    # Recursive cases
    pairlist = flat_map_chr(x, find_assign_rec),
    call = find_assign_call(x)
  )
}

find_assign(a <- b <- c <- 1)
```

```
## [1] "a" "b" "c"
```

```r
find_assign(system.time(x <- print(y <- 5)))
```

```
## [1] "x" "y"
```

The complete version of this function is quite complicated, it's important to remember we wrote it by working our way up by writing simple component parts.

### 18.5.3 Exercises

1.  `logical_abbr()` returns `TRUE` for `T(1, 2, 3)`. How could you modify
    `logical_abbr_rec()` so that it ignores function calls that use `T` or `F`?


```r
expr_type <- function(x) {
  if (rlang::is_syntactic_literal(x)) {
    "constant"
  } else if (is.symbol(x)) {
    "symbol"
  } else if (is.call(x)) {
    "call"
  } else if (is.pairlist(x)) {
    "pairlist"
  } else {
    typeof(x)
  }
}

switch_expr <- function(x, ...) {
  switch(expr_type(x),
         ...,
         stop("Don't know how to handle type ", 
              typeof(x), call. = FALSE))
}
```


```r
# original
logical_abbr_rec <- function(x) {
  switch_expr(x,
    # Base cases
    constant = FALSE,
    symbol = as_string(x) %in% c("F", "T"),

    # Recursive cases
    call = ,
    pairlist = purrr::some(x, logical_abbr_rec)
  )
}
```


```r
logical_abbr(T(1, 2, 3))
```

```
## [1] TRUE
```



```r
# updated
find_T_call <- function(x) {
  if (is_call(x, "T")) {
    x <- as.list(x)[-1]
    purrr::some(x, logical_abbr_rec)
  } else {
    purrr::some(x, logical_abbr_rec)
  }
}

logical_abbr_rec <- function(x) {
  switch_expr(
    x,
    # Base cases
    constant = FALSE,
    symbol = as_string(x) %in% c("F", "T"),
    
    # Recursive cases
    pairlist = purrr::some(x, logical_abbr_rec),
    call = find_T_call(x)
  )
}

logical_abbr <- function(x) {
  logical_abbr_rec(enexpr(x))
}
```


```r
logical_abbr(T(1, 2, 3))
```

```
## [1] FALSE
```

```r
logical_abbr(T(T, T(3, 4)))
```

```
## [1] TRUE
```

```r
logical_abbr(T(T))
```

```
## [1] TRUE
```

```r
logical_abbr(T())
```

```
## [1] FALSE
```

```r
logical_abbr()
```

```
## [1] FALSE
```

```r
logical_abbr(c(T, T, T))
```

```
## [1] TRUE
```


2.  `logical_abbr()` works with expressions. It currently fails when you give it
    a function. Why? How could you modify `logical_abbr()` to make it
    work? What components of a function will you need to recurse over?

    
    ```r
    logical_abbr(function(x = TRUE) {
      g(x + T)
    })
    ```

> The current implementation of the function fails when encountering a closure ("closure") because it is not handled in switch_expr() within logical_abbr_rec().


```r
f <- function(x = TRUE) {
  g(x + T)
}
```


```r
logical_abbr(!!f)
```

```
## Error: Don't know how to handle type closure
```

> If we want to make it work, we have to write a function to also iterate over the formals and the body of the input function.

3.  Modify `find_assign` to also detect assignment using replacement
    functions, i.e. `names(x) <- y`.

4.  Write a function that extracts all calls to a specified function.


## 18.6 Specialised data structures {#expression-special}

There are two data structures and one special symbol that we need to cover for the sake of completeness. They are not usually important in practice.

### 18.6.1 Pairlists
\index{pairlists}

Pairlists are a remnant of R's past and have been replaced by lists almost everywhere. The only place you are likely to see pairlists in R[^pairlists-c] is when working with calls to the `function` function, as the formal arguments to a function are stored in a pairlist:


```r
f <- expr(function(x, y = 10) x + y)

args <- f[[2]]
args
```

```
## $x
## 
## 
## $y
## [1] 10
```

```r
typeof(args)
```

```
## [1] "pairlist"
```

[^pairlists-c]: If you're working in C, you'll encounter pairlists more often. For example, call objects are also implemented using pairlists.

Fortunately, whenever you encounter a pairlist, you can treat it just like a regular list:


```r
pl <- pairlist(x = 1, y = 2)
length(pl)
```

```
## [1] 2
```

```r
pl$x
```

```
## [1] 1
```

Behind the scenes pairlists are implemented using a different data structure, a linked list instead of an array. That makes subsetting a pairlist much slower than subsetting a list, but this has little practical impact.

### 18.6.2 Missing arguments {#empty-symbol}
\index{symbols|empty}
\index{missing arguments}

The special symbol that needs a little extra discussion is the empty symbol, which is used to represent missing arguments (not missing values!). You only need to care about the missing symbol if you're programmatically creating functions with missing arguments; we'll come back to that in Section \@ref(unquote-missing).

You can make an empty symbol with  `missing_arg()` (or `expr()`):


```r
missing_arg()
```

```r
typeof(missing_arg())
```

```
## [1] "symbol"
```

An empty symbol doesn't print anything, so you can check if you have one with `rlang::is_missing()`:


```r
is_missing(missing_arg())
```

```
## [1] TRUE
```

You'll find them in the wild in function formals:


```r
f <- expr(function(x, y = 10) x + y)
args <- f[[2]]
is_missing(args[[1]])
```

```
## [1] TRUE
```

This is particularly important for `...` which is always associated with an empty symbol:


```r
f <- expr(function(...) list(...))
args <- f[[2]]
is_missing(args[[1]])
```

```
## [1] TRUE
```

The empty symbol has a peculiar property: if you bind it to a variable, then access that variable, you will get an error:


```r
m <- missing_arg()
m
```

```
## Error in eval(expr, envir, enclos): argument "m" is missing, with no default
```

But you won't if you store it inside another data structure!


```r
ms <- list(missing_arg(), missing_arg())
ms[[1]]
```
If you need to preserve the missingness of a variable, `rlang::maybe_missing()` is often helpful. It allows you to refer to a potentially missing variable without triggering the error. See the documentation for use cases and more details.

### 18.6.3 Expression vectors {#expression-vectors}
\index{expression vectors}
\index{expression vectors!expression@\texttt{expression()}}

Finally, we need to briefly discuss the expression vector. Expression vectors are only produced by two base functions: `expression()` and `parse()`:


```r
exp1 <- parse(text = c("
x <- 4
x
"))
exp2 <- expression(x <- 4, x)

typeof(exp1)
```

```
## [1] "expression"
```

```r
typeof(exp2)
```

```
## [1] "expression"
```

```r
exp1
```

```
## expression(x <- 4, x)
```

```r
exp2
```

```
## expression(x <- 4, x)
```

Like calls and pairlists, expression vectors behave like lists:


```r
length(exp1)
```

```
## [1] 2
```

```r
exp1[[1]]
```

```
## x <- 4
```

Conceptually, an expression vector is just a list of expressions. The only difference is that calling `eval()` on an expression evaluates each individual expression. I don't believe this advantage merits introducing a new data structure, so instead of expression vectors I just use lists of expressions.
