---
title: "Ch19_Quasiquotation"
author: "Min-Yao"
date: "2023-07-15"
output: 
  html_document: 
    keep_md: yes
---

# 19 Quasiquotation

## 19.1 Introduction

Now that you understand the tree structure of R code, it's time to return to one of the fundamental ideas that make `expr()` and `ast()` work: quotation. In tidy evaluation, all quoting functions are actually quasiquoting functions because they also support unquoting. Where quotation is the act of capturing an unevaluated expression, __unquotation__ is the ability to selectively evaluate parts of an otherwise quoted expression. Together, this is called quasiquotation. Quasiquotation makes it easy to create functions that combine code written by the function's author with code written by the function's user. This helps to solve a wide variety of challenging problems. 

Quasiquotation is one of the three pillars of tidy evaluation. You'll learn about the other two (quosures and the data mask) in Chapter \@ref(evaluation). When used alone, quasiquotation is most useful for programming, particularly for generating code. But when it's combined with the other techniques, tidy evaluation becomes a powerful tool for data analysis.

### Outline {-}

* Section \@ref(quasi-motivation) motivates the development of quasiquotation
  with a function, `cement()`, that works like `paste()` but automatically
  quotes its arguments so that you don't have to.
  
* Section \@ref(quoting) gives you the tools to quote expressions, whether
  they come from you or the user, or whether you use rlang or base R tools.
  
* Section \@ref(unquoting) introduces the biggest difference between rlang 
  quoting functions and base quoting function: unquoting with `!!` and `!!!`.

* Section \@ref(base-nonquote) discusses the three main non-quoting
  techniques that base R functions uses to disable quoting behaviour. 
  
* Section \@ref(tidy-dots) explores another place that you can use `!!!`,
  functions that take `...`. It also introduces the special `:=` operator,
  which allows you to dynamically change argument names.
  
* Section \@ref(expr-case-studies) shows a few practical uses of quoting to solve
  problems that naturally require some code generation.

* Section \@ref(history) finishes up with a little history of quasiquotation
  for those who are interested.

### Prerequisites {-}

Make sure you've read the metaprogramming overview in Chapter \@ref(meta-big-picture) to get a broad overview of the motivation and the basic vocabulary, and that you're familiar with the tree structure of expressions as described in Section \@ref(expression-details).

Code-wise, we'll mostly be using the tools from [rlang](https://rlang.r-lib.org), but at the end of the chapter you'll also see some powerful applications in conjunction with [purrr](https://purrr.tidyverse.org).


```r
library(rlang)
library(purrr)
```

### Related work {-}
\index{macros} 
\index{fexprs}
 
Quoting functions have deep connections to Lisp __macros__. But macros are usually run at compile-time, which doesn't exist in R, and they always input and output ASTs. See @lumley-2001 for one approach to implementing them in R. Quoting functions are more closely related to the more esoteric Lisp [__fexprs__](http://en.wikipedia.org/wiki/Fexpr), functions where all arguments are quoted by default. These terms are useful to know when looking for related work in other programming languages.

## 19.2 Motivation {#quasi-motivation}

We'll start with a concrete example that helps motivate the need for unquoting, and hence quasiquotation. Imagine you're creating a lot of strings by joining together words:


```r
paste("Good", "morning", "Hadley")
```

```
## [1] "Good morning Hadley"
```

```r
paste("Good", "afternoon", "Alice")
```

```
## [1] "Good afternoon Alice"
```

You are sick and tired of writing all those quotes, and instead you just want to use bare words. To that end, you've written the following function. (Don't worry about the implementation for now; you'll learn about the pieces later.)


```r
cement <- function(...) {
  args <- ensyms(...)
  paste(purrr::map(args, as_string), collapse = " ")
}

cement(Good, morning, Hadley)
```

```
## [1] "Good morning Hadley"
```

```r
cement(Good, afternoon, Alice)
```

```
## [1] "Good afternoon Alice"
```

Formally, this function quotes all of its inputs. You can think of it as automatically putting quotation marks around each argument. That's not precisely true as the intermediate objects it generates are expressions, not strings, but it's a useful approximation, and the root meaning of the term "quote".

This function is nice because we no longer need to type quotation marks. The problem comes when we want to use variables. It's easy to use variables with `paste()`: just don't surround them with quotation marks.


```r
name <- "Hadley"
time <- "morning"

paste("Good", time, name)
```

```
## [1] "Good morning Hadley"
```

Obviously this doesn't work with `cement()` because every input is automatically quoted:


```r
cement(Good, time, name)
```

```
## [1] "Good time name"
```

We need some way to explicitly _unquote_ the input to tell `cement()` to remove the automatic quote marks. Here we need `time` and `name` to be treated differently to `Good`. Quasiquotation gives us a standard tool to do so: `!!`, called "unquote", and pronounced bang-bang. `!!` tells a quoting function to drop the implicit quotes:


```r
cement(Good, !!time, !!name)
```

```
## [1] "Good morning Hadley"
```

It's useful to compare `cement()` and `paste()` directly. `paste()` evaluates its arguments, so we must quote where needed; `cement()` quotes its arguments, so we must unquote where needed.


```r
paste("Good", time, name)
cement(Good, !!time, !!name)
```

### 19.2.1 Vocabulary
\index{arguments!evaluated versus quoted}
\index{non-standard evaluation}

The distinction between quoted and evaluated arguments is important:

* An __evaluated__ argument obeys R's usual evaluation rules.

* A __quoted__ argument is captured by the function, and is processed in
  some custom way.

`paste()` evaluates all its arguments; `cement()` quotes all its arguments.

> If you're ever unsure about whether an argument is quoted or evaluated, try executing the code outside of the function. If it doesn't work or does something different, then that argument is quoted. For example, you can use this technique to determine that the first argument to `library()` is quoted:


```r
# works
library(MASS)

# fails
MASS
```

Talking about whether an argument is quoted or evaluated is a more precise way of stating whether or not a function uses non-standard evaluation (NSE). I will sometimes use "quoting function" as short-hand for a function that quotes one or more arguments, but generally, I'll talk about quoted arguments since that is the level at which the difference applies.

### 19.2.2 Exercises

1.  For each function in the following base R code, identify which arguments
    are quoted and which are evaluated.


```r
library(MASS)

mtcars2 <- subset(mtcars, cyl == 4)

with(mtcars2, sum(vs))
sum(mtcars2$am)

rm(mtcars2)
```



```r
library(MASS)
MASS
```

> MASS -> quoted


```r
mtcars2 <- subset(mtcars, cyl == 4)
mtcars2
mtcars
cyl
```

> mtcars -> evaluated

> cyl -> quoted


```r
with(mtcars2, sum(vs))
mtcars2
vs
```

> mtcars2 -> evaluated

> sum(vs) -> quoted


```r
sum(mtcars2$am)
mtcars2$am
am
```

> matcars$am -> evaluated

> am -> quoted by $()  

> We begin by adhering to the guidance provided in Advanced R, executing each argument independently of its corresponding function. The execution of `MASS`, `cyl`, `vs`, and `am` results in an "Object not found" error since they are not present within the global environment.

> This process serves to validate that the function arguments are indeed quoted. In contrast, for the remaining arguments, we can examine both the source code and the accompanying documentation to ascertain whether any quoting mechanisms are employed or if the arguments undergo evaluation.


```r
rm(mtcars2)
mtcars2

rm
```

> mtcars2 -> quoted

> Upon examination of the source code for rm(), it becomes apparent that the ... argument is captured as an unevaluated call, specifically a pairlist, utilizing the match.call() function. Subsequently, this call is transformed into a string format to facilitate subsequent evaluation.

2.  For each function in the following tidyverse code, identify which arguments
    are quoted and which are evaluated.


```r
library(dplyr)
library(ggplot2)

by_cyl <- mtcars %>%
  group_by(cyl) %>%
  summarise(mean = mean(mpg))

ggplot(by_cyl, aes(cyl, mean)) + geom_point()
```


```r
library(dplyr)
dplyr

library(ggplot2)
ggplot2

by_cyl <- mtcars %>%
  group_by(cyl) %>%
  summarise(mean = mean(mpg))

mtcars
cyl
mean = mean(mpg)
mpg
```

> dplyr   -> quoted

> ggplot2 -> quoted

> mtcars -> evaluated

> cyl -> quoted

> mean = mean(mpg) -> quoted

> mpg -> evaluated


```r
dplyr::summarise
```

```
## function (.data, ..., .by = NULL, .groups = NULL) 
## {
##     by <- enquo(.by)
##     if (!quo_is_null(by) && !is.null(.groups)) {
##         abort("Can't supply both `.by` and `.groups`.")
##     }
##     UseMethod("summarise")
## }
## <bytecode: 0x0000026657766898>
## <environment: namespace:dplyr>
```


```r
dplyr:::summarise.data.frame
```

```
## function (.data, ..., .by = NULL, .groups = NULL) 
## {
##     by <- compute_by({
##         {
##             .by
##         }
##     }, .data, by_arg = ".by", data_arg = ".data")
##     cols <- summarise_cols(.data, dplyr_quosures(...), by, "summarise")
##     out <- summarise_build(by, cols)
##     if (!cols$all_one) {
##         summarise_deprecate_variable_size()
##     }
##     if (!is_tibble(.data)) {
##         out <- as.data.frame(out)
##     }
##     if (identical(.groups, "rowwise")) {
##         out <- rowwise_df(out, character())
##     }
##     out
## }
## <bytecode: 0x0000026659143418>
## <environment: namespace:dplyr>
```

> To gain insight into the inner workings of `summarise()`, we delve into the source code. By tracing the S3-dispatch of `summarise()`, we discover that the ... argument is quoted.


```r
ggplot(by_cyl, aes(cyl, mean)) + geom_point()

by_cyl
aes(cyl, mean)
cyl
mean
```

> by_cyl -> evaluated

> aes(cyl, mean) -> evaluated

> cyl, mean -> quoted (via aes)



```r
ggplot2::aes
```

```
## function (x, y, ...) 
## {
##     xs <- arg_enquos("x")
##     ys <- arg_enquos("y")
##     dots <- enquos(...)
##     args <- c(xs, ys, dots)
##     args <- Filter(Negate(quo_is_missing), args)
##     local({
##         aes <- function(x, y, ...) NULL
##         inject(aes(!!!args))
##     })
##     aes <- new_aes(args, env = parent.frame())
##     rename_aes(aes)
## }
## <bytecode: 0x000002665886c0a8>
## <environment: namespace:ggplot2>
```



## 19.3 Quoting
\index{quoting}

The first part of quasiquotation is quotation: capturing an expression without evaluating it. We'll need a pair of functions because the expression can be supplied directly or indirectly, via lazily-evaluated function argument. I'll start with the rlang quoting functions, then circle back to those provided by base R.

### 19.3.1 Capturing expressions
\index{expressions!capturing}
\indexc{expr()}
\index{quoting!expr@\texttt{expr()}}

There are four important quoting functions. For interactive exploration, the most important is `expr()`, which captures its argument exactly as provided:


```r
expr(x + y)
```

```
## x + y
```

```r
expr(1 / 2 / 3)
```

```
## 1/2/3
```

(Remember that white space and comments are not part of the expression, so will not be captured by a quoting function.)

`expr()` is great for interactive exploration, because it captures what you, the developer, typed. It's not so useful inside a function:


```r
f1 <- function(x) expr(x)
f1(a + b + c)
```

```
## x
```

\indexc{enexpr()}
We need another function to solve this problem: `enexpr()`. This captures what the caller supplied to the function by looking at the internal promise object that powers lazy evaluation (Section \@ref(promises)).


```r
f2 <- function(x) enexpr(x)
f2(a + b + c)
```

```
## a + b + c
```

(It's called "en"-`expr()` by analogy to enrich. Enriching someone makes them richer; `enexpr()`ing a argument makes it an expression.)

To capture all arguments in `...`, use `enexprs()`.


```r
f <- function(...) enexprs(...)
f(x = 1, y = 10 * z)
```

```
## $x
## [1] 1
## 
## $y
## 10 * z
```

Finally, `exprs()` is useful interactively to make a list of expressions:


```r
exprs(x = x ^ 2, y = y ^ 3, z = z ^ 4)
# shorthand for
# list(x = expr(x ^ 2), y = expr(y ^ 3), z = expr(z ^ 4))
```

In short, use `enexpr()` and `enexprs()` to capture the expressions supplied as arguments _by the user_. Use `expr()` and `exprs()` to capture expressions that _you_ supply.

### 19.3.2 Capturing symbols
\index{symbols!capturing}
\indexc{ensym()}

Sometimes you only want to allow the user to specify a variable name, not an arbitrary expression. In this case, you can use `ensym()` or `ensyms()`. These are variants of `enexpr()` and `enexprs()` that check the captured expression is either symbol or a string (which is converted to a symbol[^string-symbol]). `ensym()` and `ensyms()` throw an error if given anything else.

[^string-symbol]: This is for compatibility with base R, which allows you to provide a string instead of a symbol in many places: `"x" <- 1`, `"foo"(x, y)`, `c("x" = 1)`.


```r
f <- function(...) ensyms(...)
f(x)
```

```
## [[1]]
## x
```

```r
f("x")
```

```
## [[1]]
## x
```


```r
f(1)
```


### 19.3.3 With base R
\index{expressions!capturing with base R}
\index{quoting!quote@\texttt{quote()}}

Each rlang function described above has an equivalent in base R. Their primary difference is that the base equivalents do not support unquoting (which we'll talk about very soon). This make them quoting functions, rather than quasiquoting functions.

The base equivalent of `expr()` is `quote()`:
  

```r
quote(x + y)
```

```
## x + y
```

The base function closest to `enexpr()` is `substitute()`:


```r
f3 <- function(x) substitute(x)
f3(x + y)
```

```
## x + y
```

\indexc{alist()}
The base equivalent to `exprs()` is `alist()`:
  

```r
alist(x = 1, y = x + 2)
```

```
## $x
## [1] 1
## 
## $y
## x + 2
```

The equivalent to `enexprs()` is an undocumented feature of `substitute()`[^peter-meilstrup]:


```r
f <- function(...) as.list(substitute(...()))
f(x = 1, y = 10 * z)
```

```
## $x
## [1] 1
## 
## $y
## 10 * z
```

[^peter-meilstrup]: Discovered by Peter Meilstrup and described in [R-devel on 2018-08-13](http://r.789695.n4.nabble.com/substitute-on-arguments-in-ellipsis-quot-dot-dot-dot-quot-td4751658.html).

There are two other important base quoting functions that we'll cover elsewhere:

* `bquote()` provides a limited form of quasiquotation, and is discussed in 
  Section \@ref(base-nonquote). 
  
* `~`, the formula, is a quoting function that also captures the environment. 
  It's the inspiration for quosures, the topic of the next chapter, and is 
  discussed in Section \@ref(quosure-impl).

### 19.3.4 Substitution
\indexc{substitute()}

You'll most often see `substitute()` used to capture unevaluated arguments. However, as well as quoting, `substitute()` also does substitution (as its name suggests!). If you give it an expression, rather than a symbol, it will substitute in the values of symbols defined in the current environment. 


```r
f4 <- function(x) substitute(x * 2)
f4(a + b + c)
```

```
## (a + b + c) * 2
```

I think this makes code hard to understand, because if it is taken out of context, you can't tell if the goal of `substitute(x + y)` is to replace `x`, `y`, or both. If you do want to use `substitute()` for substitution, I recommend that you use the second argument to make your goal clear:


```r
substitute(x * y * z, list(x = 10, y = quote(a + b)))
```

```
## 10 * (a + b) * z
```

### 19.3.5 Summary

When quoting (i.e. capturing code), there are two important distinctions: 

* Is it supplied by the developer of the code or the user of the code?
  In other words, is it fixed (supplied in the body of the function) or varying (supplied
  via an argument)?
  
* Do you want to capture a single expression or multiple expressions?

This leads to a 2 $\times$ 2 table of functions for rlang, Table \@ref(tab:quoting-rlang), and for base R, Table \@ref(tab:quoting-base).

|      | Developer | User        |
|------|-----------|-------------|
| One  | `expr()`  | `enexpr()`  |
| Many | `exprs()` | `enexprs()` |
Table: (\#tab:quoting-rlang) rlang quasiquoting functions

|      | Developer | User                         |
|------|-----------|------------------------------|
| One  | `quote()` | `substitute()`               |
| Many | `alist()` | `as.list(substitute(...()))` |
Table: (\#tab:quoting-base) base R quoting functions

### 19.3.6 Exercises

1.  How is `expr()` implemented? Look at its source code.


```r
expr
```

```
## function (expr) 
## {
##     enexpr(expr)
## }
## <bytecode: 0x0000026658a33a58>
## <environment: namespace:rlang>
```

> `expr()` serves as a straightforward wrapper, channeling its argument to `enexpr()`. 

2.  Compare and contrast the following two functions. Can you predict the
    output before running them?

    
    ```r
    f1 <- function(x, y) {
      exprs(x = x, y = y)
    }
    f2 <- function(x, y) {
      enexprs(x = x, y = y)
    }
    f1(a + b, c + d)
    f2(a + b, c + d)
    ```

> In both functions, multiple arguments can be captured, resulting in a named list of expressions. When using f1(), the arguments defined within the body of the function will be returned. This behavior occurs because exprs() captures the expressions specified by the developer during the definition of f1().


```r
f1(a + b, c + d)
```

> On the other hand, f2() will return the arguments provided to the function when it is called, as specified by the user.


```r
f2(a + b, c + d)
```


3.  What happens if you try to use `enexpr()` with an expression (i.e. 
    `enexpr(x + y)` ? What happens if `enexpr()` is passed a missing argument?



```r
enexpr(x + y)
```

> The first scenario results in an error being raise


```r
enexpr()
```

```r
is_missing(enexpr())
```

```
## [1] TRUE
```

> In the second case, `enexpr()` is called without any arguments, leading to a missing argument being returned. The function `is_missing()` confirms that `enexpr()` indeed returns a missing argument.

4.  How are `exprs(a)` and `exprs(a = )` different? Think about both the
    input and the output.


```r
exprs(a)
```

```
## [[1]]
## a
```

```r
str(exprs(a))
```

```
## List of 1
##  $ : symbol a
```

> In the code snippet exprs(a), the input a is considered as a symbol representing an unnamed argument. Consequently, the output will be an unnamed list, with the first element containing the symbol a.


```r
exprs(a = )
```

```
## $a
```

```r
str(exprs(a = ))
```

```
## List of 1
##  $ a: symbol
```

```r
test_exprs <- exprs(a = )
is_missing(test_exprs$a)
```

```
## [1] TRUE
```

> However, in the code snippet exprs(a = ), the first argument is named a, but no value is provided for it. As a result, the output will be a named list, where the first element is labeled as a, indicating the presence of a missing argument.

5.  What are other differences between `exprs()` and `alist()`? Read the 
    documentation for the named arguments of `exprs()` to find out.


```r
# Usage

# list(...)

# exprs(
#   ...,
#   .named = FALSE,
#   .ignore_empty = c("trailing", "none", "all"),
#   .unquote_names = TRUE
# )
```

> exprs() is the plural variant of expr(). It returns a list of expressions. It is like base::alist() but with injection support.

> The injection operators are extensions of R implemented by rlang to modify a piece of code before R processes it. There are two main families:

The dynamic dots operators, `!!!` and `"{"`.

The metaprogramming operators `!!`, `{{`, and `"{{"`. Splicing with `!!!` can also be done in metaprogramming context.

> exprs() offers several additional arguments: `.named` (default FALSE), `.ignore_empty` (options: "trailing", "none", "all"), and `.unquote_names` (default TRUE). 

> By using `.named`, all dots can be ensured to have names. If TRUE, unnamed inputs are automatically named with as_label(). This is equivalent to applying exprs_auto_name() on the result. If FALSE, unnamed elements are left as is and, if fully unnamed, the list is given minimal names (a vector of ""). If NULL, fully unnamed results are left with NULL names.

> The `.ignore_empty` parameter determines how empty arguments should be handled, allowing the options to handle only trailing dots ("trailing"), ignore none of the arguments ("none"), or ignore all of them ("all"). If "trailing", only the last argument is ignored if it is empty. Named arguments are not considered empty.

> Moreover, `.unquote_names` allows developers to treat `:=` as `=`, which can be beneficial for supporting unquoting (`!!`) on the left-hand side.


6.  The documentation for `substitute()` says:

    > Substitution takes place by examining each component of the parse tree 
    > as follows: 
    > 
    > * If it is not a bound symbol in `env`, it is unchanged. 
    > * If it is a promise object (i.e., a formal argument to a function) 
    >   the expression slot of the promise replaces the symbol. 
    > * If it is an ordinary variable, its value is substituted, unless 
    > `env` is .GlobalEnv in which case the symbol is left unchanged.
  
    Create examples that illustrate each of the above cases.

> Let's establish a fresh environment called `new_env`, which currently holds no objects. In this scenario, when we utilize `substitute()`, it will simply return its initial argument expr as is:


```r
new_env <- env()
substitute(x, new_env)
```

```
## x
```

> Now, if we construct a function that contains an argument directly returned after substitution, the function will simply yield the supplied expression:


```r
evaluate_expression <- function(x) substitute(x)

evaluate_expression(x + y * a / b)
```

```
## x + y * a/b
```

> However, when `substitute()` identifies (portions of) the expression within env, it will perform a literal substitution, except when env refers to .GlobalEnv.


```r
new_env$x <- 7
substitute(x, new_env)
```

```
## [1] 7
```

```r
x <- 7
substitute(x, .GlobalEnv)
```

```
## x
```

> In summary, the behavior of `substitute()` depends on whether the specified environment contains objects or is the `.GlobalEnv`.
