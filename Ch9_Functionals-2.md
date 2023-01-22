---
title: "Ch9_Functionals-2"
author: "Min-Yao"
date: "2023-01-14"
output: 
  html_document: 
    keep_md: yes
---


```r
library(purrr)
```

## 9.5 Reduce family {#reduce}

After the map family, the next most important family of functions is the reduce family. This family is much smaller, with only two main variants, and is used less commonly, but it's a powerful idea, gives us the opportunity to discuss some useful algebra, and powers the map-reduce framework frequently used for processing very large datasets.

### 9.5.1 Basics
\indexc{reduce()} 
\index{fold|see {reduce}}

`reduce()` takes a vector of length _n_ and produces a vector of length 1 by calling a function with a pair of values at a time: `reduce(1:4, f)` is equivalent to `f(f(f(1, 2), 3), 4)`. 

<img src="diagrams/functionals/reduce.png" width="779" />

`reduce()` is a useful way to generalise a function that works with two inputs (a __binary__ function) to work with any number of inputs. Imagine you have a list of numeric vectors, and you want to find the values that occur in every element. First we generate some sample data:


```r
l <- map(1:4, ~ sample(1:10, 15, replace = T))
str(l)
```

```
## List of 4
##  $ : int [1:15] 3 1 1 5 10 1 6 8 1 1 ...
##  $ : int [1:15] 4 10 10 1 1 5 10 5 8 8 ...
##  $ : int [1:15] 8 3 1 5 7 6 9 6 2 6 ...
##  $ : int [1:15] 4 5 7 9 8 2 3 9 4 6 ...
```

To solve this challenge we need to use `intersect()` repeatedly:


```r
out <- l[[1]]
out <- intersect(out, l[[2]])
out <- intersect(out, l[[3]])
out <- intersect(out, l[[4]])
out
```

```
## [1] 3 1 5 8 2
```

`reduce()` automates this solution for us, so we can write:


```r
reduce(l, intersect)
```

```
## [1] 3 1 5 8 2
```

We could apply the same idea if we wanted to list all the elements that appear in at least one entry. All we have to do is switch from `intersect()` to `union()`:


```r
reduce(l, union)
```

```
##  [1]  3  1  5 10  6  8  2  4  9  7
```

Like the map family, you can also pass additional arguments. `intersect()` and `union()` don't take extra arguments so I can't demonstrate them here, but the principle is straightforward and I drew you a picture.

<img src="diagrams/functionals/reduce-arg.png" width="968" />

As usual, the essence of `reduce()` can be reduced to a simple wrapper around a for loop:


```r
simple_reduce <- function(x, f) {
  out <- x[[1]]
  for (i in seq(2, length(x))) {
    out <- f(out, x[[i]])
  }
  out
}
```

::: base 
The base equivalent is `Reduce()`. Note that the argument order is different: the function comes first, followed by the vector, and there is no way to supply additional arguments.
:::

### 9.5.2 Accumulate
\indexc{accumulate()}

The first `reduce()` variant, `accumulate()`, is useful for understanding how reduce works, because instead of returning just the final result, it returns all the intermediate results as well:


```r
accumulate(l, intersect)
```

```
## [[1]]
##  [1]  3  1  1  5 10  1  6  8  1  1 10  8  2  6  5
## 
## [[2]]
## [1]  3  1  5 10  8  2
## 
## [[3]]
## [1] 3 1 5 8 2
## 
## [[4]]
## [1] 3 1 5 8 2
```

Another useful way to understand reduce is to think about `sum()`: `sum(x)` is equivalent to `x[[1]] + x[[2]] + x[[3]] + ...`, i.e. ``reduce(x, `+`)``. Then ``accumulate(x, `+`)`` is the cumulative sum:


```r
x <- c(4, 3, 10)
reduce(x, `+`)
```

```
## [1] 17
```

```r
accumulate(x, `+`)
```

```
## [1]  4  7 17
```

### 9.5.3 Output types

In the above example using `+`, what should `reduce()` return when `x` is short, i.e. length 1 or 0? Without additional arguments, `reduce()` just returns the input when `x` is length 1:


```r
reduce(1, `+`)
```

```
## [1] 1
```

This means that `reduce()` has no way to check that the input is valid:


```r
reduce("a", `+`)
```

```
## [1] "a"
```

What if it's length 0? We get an error that suggests we need to use the `.init` argument:


```r
reduce(integer(), `+`)
```

```
## Error in `reduce()`:
## ! Must supply `.init` when `.x` is empty.
```

What should `.init` be here? To figure that out, we need to see what happens when `.init` is supplied:

<img src="diagrams/functionals/reduce-init.png" width="1015" />

So if we call ``reduce(1, `+`, init)`` the result will be `1 + init`. Now we know that the result should be just `1`, so that suggests that `.init` should be 0:


```r
reduce(integer(), `+`, .init = 0)
```

```
## [1] 0
```

This also ensures that `reduce()` checks that length 1 inputs are valid for the function that you're calling:


```r
reduce("a", `+`, .init = 0)
```

```
## Error in .x + .y: non-numeric argument to binary operator
```

If you want to get algebraic about it, 0 is called the __identity__ of the real numbers under the operation of addition: if you add a 0 to any number, you get the same number back. R applies the same principle to determine what a summary function with a zero length input should return:


```r
sum(integer())  # x + 0 = x
```

```
## [1] 0
```

```r
prod(integer()) # x * 1 = x
```

```
## [1] 1
```

```r
min(integer())  # min(x, Inf) = x
```

```
## [1] Inf
```

```r
max(integer())  # max(x, -Inf) = x
```

```
## [1] -Inf
```

```r
reduce(integer(), sum, .init = "x")
```

```
## [1] "x"
```

```r
reduce(integer(), prod, .init = "x")
```

```
## [1] "x"
```

```r
reduce(integer(), min, .init = "x")
```

```
## [1] "x"
```

```r
reduce(integer(), max, .init = "x")
```

```
## [1] "x"
```

If you're using `reduce()` in a function, you should always supply `.init`. Think carefully about what your function should return when you pass a vector of length 0 or 1, and make sure to test your implementation.

### 9.5.4 Multiple inputs
\indexc{reduce2()}

Very occasionally you need to pass two arguments to the function that you're reducing. For example, you might have a list of data frames that you want to join together, and the variables you use to join will vary from element to element. This is a very specialised scenario, so I don't want to spend much time on it, but I do want you to know that `reduce2()` exists.

The length of the second argument varies based on whether or not `.init` is supplied: if you have four elements of `x`, `f` will only be called three times. If you supply init, `f` will be called four times.

<img src="diagrams/functionals/reduce2.png" width="1299" />
<img src="diagrams/functionals/reduce2-init.png" width="1299" />

### 9.5.5 Map-reduce
\index{map-reduce}

You might have heard of map-reduce, the idea that powers technology like Hadoop. Now you can see how simple and powerful the underlying idea is: map-reduce is a map combined with a reduce. The difference for large data is that the data is spread over multiple computers. Each computer performs the map on the data that it has, then it sends the result to back to a coordinator which _reduces_ the individual results back to a single result.

As a simple example, imagine computing the mean of a very large vector, so large that it has to be split over multiple computers. You could ask each computer to calculate the sum and the length, and then return those to the coordinator which computes the overall mean by dividing the total sum by the total length.

## 9.6 Predicate functionals
\index{predicates} 
\index{functions!predicate|see {predicates}}

A __predicate__ is a function that returns a single `TRUE` or `FALSE`, like `is.character()`, `is.null()`, or `all()`, and we say a predicate __matches__ a vector if it returns `TRUE`. 

### 9.6.1 Basics

A __predicate functional__ applies a predicate to each element of a vector. purrr provides seven useful functions which come in three groups:

*   `some(.x, .p)` returns `TRUE` if _any_ element matches;  
    `every(.x, .p)` returns `TRUE` if _all_ elements match;  
    `none(.x, .p)` returns `TRUE` if _no_ element matches.
    
    These are similar to `any(map_lgl(.x, .p))`, `all(map_lgl(.x, .p))` and
    `all(map_lgl(.x, negate(.p)))` but they terminate early: `some()` returns
    `TRUE` when it sees the first `TRUE`, and `every()` and `none()` return
    `FALSE` when they see the first `FALSE` or `TRUE` respectively.

* `detect(.x, .p)` returns the _value_ of the first match;
  `detect_index(.x, .p)` returns the _location_ of the first match.

* `keep(.x, .p)` _keeps_ all matching elements;
  `discard(.x, .p)` _drops_ all matching elements.

The following example shows how you might use these functionals with a data frame:


```r
df <- data.frame(x = 1:3, y = c("a", "b", "c"))
detect(df, is.factor)
```

```
## NULL
```

```r
detect_index(df, is.factor)
```

```
## [1] 0
```

```r
str(keep(df, is.factor))
```

```
## 'data.frame':	3 obs. of  0 variables
```

```r
str(discard(df, is.factor))
```

```
## 'data.frame':	3 obs. of  2 variables:
##  $ x: int  1 2 3
##  $ y: chr  "a" "b" "c"
```

### 9.6.2 Map variants {#predicate-map}

`map()` and `modify()` come in variants that also take predicate functions, transforming only the elements of `.x` where `.p` is `TRUE`.


```r
df <- data.frame(
  num1 = c(0, 10, 20),
  num2 = c(5, 6, 7),
  chr1 = c("a", "b", "c"),
  stringsAsFactors = FALSE
)

str(map_if(df, is.numeric, mean))
```

```
## List of 3
##  $ num1: num 10
##  $ num2: num 6
##  $ chr1: chr [1:3] "a" "b" "c"
```

```r
str(modify_if(df, is.numeric, mean))
```

```
## 'data.frame':	3 obs. of  3 variables:
##  $ num1: num  10 10 10
##  $ num2: num  6 6 6
##  $ chr1: chr  "a" "b" "c"
```

```r
str(map(keep(df, is.numeric), mean))
```

```
## List of 2
##  $ num1: num 10
##  $ num2: num 6
```

### 9.6.3 Exercises

1.  Why isn't `is.na()` a predicate function? What base R function is closest
    to being a predicate version of `is.na()`?
    

```r
df <- data.frame(x = 1:3, y = c("a", "b", "c"))
str(df)
```

```
## 'data.frame':	3 obs. of  2 variables:
##  $ x: int  1 2 3
##  $ y: chr  "a" "b" "c"
```

```r
is.na(df)
```

```
##          x     y
## [1,] FALSE FALSE
## [2,] FALSE FALSE
## [3,] FALSE FALSE
```

```r
str(is.na(df))
```

```
##  logi [1:3, 1:2] FALSE FALSE FALSE FALSE FALSE FALSE
##  - attr(*, "dimnames")=List of 2
##   ..$ : NULL
##   ..$ : chr [1:2] "x" "y"
```

> `is.na()` is not a predicate function because it returns a logical vector.


```r
anyNA(df)
```

```
## [1] FALSE
```

> `anyNA()` is closest to being a predicate version of `is.na()` because it returns a single TRUE or FALSE.

2.  `simple_reduce()` has a problem when `x` is length 0 or length 1. Describe
    the source of the problem and how you might go about fixing it.
    
    
    ```r
    simple_reduce <- function(x, f) {
      out <- x[[1]]
      for (i in seq(2, length(x))) {
        out <- f(out, x[[i]])
      }
      out
    }
    ```


```r
simple_reduce(c(1:5), sum)
```

```
## [1] 15
```



```r
simple_reduce(1, sum)
```


```r
integer()
simple_reduce(integer(), sum)
```


```r
simple_reduce <- function(x, f, default = NULL) {
  if (length(x) == 0L & is.null(default) == TRUE)
    stop("x is length 0")
  
  if (length(x) == 0L & is.null(default) == FALSE)
    return(default)
  
  if (length(x) == 1L & is.null(default) == TRUE)
    return(x)
  
  if (length(x) == 1L & is.null(default) == FALSE)
    out <- f(default, x) 
  return(out)
  
  
  out <- x[[1]]
  for (i in seq(2, length(x))) {
    out <- f(out, x[[i]])
  }
  out
}
```


```r
simple_reduce(c(1:5), sum)
```

```
## [1] 3 1 5 8 2
```


```r
simple_reduce(c(1:5), sum, 1)
```

```
## [1] 3 1 5 8 2
```


```r
simple_reduce(1, sum)
```

```
## [1] 1
```



```r
simple_reduce(1, sum, 1)
```

```
## [1] 2
```


```r
simple_reduce(integer(), sum, 1)
```


```r
simple_reduce(integer(), sum)
```


3.  Implement the `span()` function from Haskell: given a list `x` and a 
    predicate function `f`, `span(x, f)` returns the location of the longest 
    sequential run of elements where the predicate is true. (Hint: you 
    might find `rle()` helpful.)

4.  Implement `arg_max()`. It should take a function and a vector of inputs, 
    and return the elements of the input where the function returns the highest 
    value. For example, `arg_max(-10:5, function(x) x ^ 2)` should return -10.
    `arg_max(-5:5, function(x) x ^ 2)` should return `c(-5, 5)`.
    Also implement the matching `arg_min()` function.

5.  The function below scales a vector so it falls in the range [0, 1]. How
    would you apply it to every column of a data frame? How would you apply it 
    to every numeric column in a data frame?

    
    ```r
    scale01 <- function(x) {
      rng <- range(x, na.rm = TRUE)
      (x - rng[1]) / (rng[2] - rng[1])
    }
    ```

## 9.7 Base functionals {#base-functionals}

To finish up the chapter, here I provide a survey of important base functionals that are not members of the map, reduce, or predicate families, and hence have no equivalent in purrr. This is not to say that they're not important, but they have more of a mathematical or statistical flavour, and they are generally less useful in data analysis.

### 9.7.1 Matrices and arrays
\indexc{apply()}

`map()` and friends are specialised to work with one-dimensional vectors. `base::apply()` is specialised to work with two-dimensional and higher vectors, i.e. matrices and arrays. You can think of `apply()` as an operation that summarises a matrix or array by collapsing each row or column to a single value. It has four arguments: 

* `X`, the matrix or array to summarise.

* `MARGIN`, an integer vector giving the dimensions to summarise over, 
  1 = rows, 2 = columns, etc. (The argument name comes from thinking about
  the margins of a joint distribution.)

* `FUN`, a summary function.

* `...` other arguments passed on to `FUN`.

A typical example of `apply()` looks like this


```r
a2d <- matrix(1:20, nrow = 5)
apply(a2d, 1, mean)
```

```
## [1]  8.5  9.5 10.5 11.5 12.5
```

```r
apply(a2d, 2, mean)
```

```
## [1]  3  8 13 18
```

<!-- HW: recreate diagrams from plyr paper -->

You can specify multiple dimensions to `MARGIN`, which is useful for high-dimensional arrays:


```r
a3d <- array(1:24, c(2, 3, 4))
apply(a3d, 1, mean)
```

```
## [1] 12 13
```

```r
apply(a3d, c(1, 2), mean)
```

```
##      [,1] [,2] [,3]
## [1,]   10   12   14
## [2,]   11   13   15
```

There are two caveats to using `apply()`: 

*    Like `base::sapply()`, you have no control over the output type; it 
     will automatically be simplified to a list, matrix, or vector. However, 
     you usually use `apply()` with numeric arrays and a numeric summary
     function so you are less likely to encounter a problem than with 
     `sapply()`.

*   `apply()` is also not idempotent in the sense that if the summary 
    function is the identity operator, the output is not always the same as 
    the input. 

    
    ```r
    a1 <- apply(a2d, 1, identity)
    identical(a2d, a1)
    ```
    
    ```
    ## [1] FALSE
    ```
    
    ```r
    a2 <- apply(a2d, 2, identity)
    identical(a2d, a2)
    ```
    
    ```
    ## [1] TRUE
    ```

*   Never use `apply()` with a data frame. It always coerces it to a matrix,
    which will lead to undesirable results if your data frame contains anything
    other than numbers.
    
    
    ```r
    df <- data.frame(x = 1:3, y = c("a", "b", "c"))
    apply(df, 2, mean)
    ```
    
    ```
    ## Warning in mean.default(newX[, i], ...): argument is not numeric or logical:
    ## returning NA
    
    ## Warning in mean.default(newX[, i], ...): argument is not numeric or logical:
    ## returning NA
    ```
    
    ```
    ##  x  y 
    ## NA NA
    ```

### 9.7.2 Mathematical concerns

Functionals are very common in mathematics. The limit, the maximum, the roots (the set of points where `f(x) = 0`), and the definite integral are all functionals: given a function, they return a single number (or vector of numbers). At first glance, these functions don't seem to fit in with the theme of eliminating loops, but if you dig deeper you'll find out that they are all implemented using an algorithm that involves iteration.

Base R provides a useful set:

* `integrate()` finds the area under the curve defined by `f()`
* `uniroot()` finds where `f()` hits zero
* `optimise()` finds the location of the lowest (or highest) value of `f()`

The following example shows how functionals might be used with a simple function, `sin()`:


```r
integrate(sin, 0, pi)
```

```
## 2 with absolute error < 2.2e-14
```

```r
str(uniroot(sin, pi * c(1 / 2, 3 / 2)))
```

```
## List of 5
##  $ root      : num 3.14
##  $ f.root    : num 1.22e-16
##  $ iter      : int 2
##  $ init.it   : int NA
##  $ estim.prec: num 6.1e-05
```

```r
str(optimise(sin, c(0, 2 * pi)))
```

```
## List of 2
##  $ minimum  : num 4.71
##  $ objective: num -1
```

```r
str(optimise(sin, c(0, pi), maximum = TRUE))
```

```
## List of 2
##  $ maximum  : num 1.57
##  $ objective: num 1
```

### 9.7.3 Exercises

1.  How does `apply()` arrange the output? Read the documentation and perform 
    some experiments.

2.  What do `eapply()` and `rapply()` do? Does purrr have equivalents?

3.  Challenge: read about the 
    [fixed point algorithm](https://mitpress.mit.edu/sites/default/files/sicp/full-text/book/book-Z-H-12.html#%25_idx_1096).
    Complete the exercises using R.
