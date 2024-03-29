---
title: "Ch5_Control_flow"
author: "Min-Yao"
date: "2022-10-02"
output: 
  html_document: 
    keep_md: yes
---

# 5 Control flow

## 5.1 Introduction

There are two primary tools of control flow: choices and loops. Choices, like `if` statements and `switch()` calls, allow you to run different code depending on the input. Loops, like `for` and `while`, allow you to repeatedly run code, typically with changing options. I'd expect that you're already familiar with the basics of these functions so I'll briefly cover some technical details and then introduce some useful, but lesser known, features.

The condition system (messages, warnings, and errors), which you'll learn about in Chapter \@ref(conditions), also provides non-local control flow. 

### Quiz {-}

Want to skip this chapter? Go for it, if you can answer the questions below. Find the answers at the end of the chapter in Section \@ref(control-flow-answers).

*   What is the difference between `if` and `ifelse()`?

> Before reading: if Statement: use it to execute a block of code, if a specified condition is true. ifelse() Function: use it when to check the condition for every element of a vector

> After reading: if works with scalars (if only works with a single TRUE or FALSE); ifelse() works with vectors (ifelse(): a vectorised function with test, yes, and no vectors (that will be recycled to the same length)).

*   In the following code, what will the value of `y` be if `x` is `TRUE`?
    What if `x` is `FALSE`? What if `x` is `NA`?
  
    
    ```r
    y <- if (x) 3
    ```

> Before reading: If `x` is `TRUE`, y = 3. If `x` is `FALSE`, y = NULL. If `x` is `NA`, y = NULL.

> After reading: When x is TRUE, y will be 3; when FALSE, y will be NULL; when NA the if statement will throw an error.


```r
x <- TRUE
y <- if (x) 3
y
```

```
## [1] 3
```


```r
x <- FALSE
y <- if (x) 3
y
```

```
## NULL
```


```r
x <- NA
y <- if (x) 3
y
```

*   What does `switch("x", x = , y = 2, z = 3)` return?


```r
switch("x", x = , y = 2, z = 3)
```

```
## [1] 2
```


> Before reading: NULL.

> After reading: This switch() statement makes use of fall-through so it will return 2. See details in Section 5.2.3.

### Outline {-}

* Section \@ref(choices) dives into the details of `if`, then discusses
  the close relatives `ifelse()` and `switch()`.
  
* Section \@ref(loops) starts off by reminding you of the basic structure
  of the for loop in R, discusses some common pitfalls, and then talks
  about the related `while` and `repeat` statements.

## 5.2 Choices
\indexc{if}

The basic form of an if statement in R is as follows:


```r
# if (condition) true_action
# if (condition) true_action else false_action
```

If `condition` is `TRUE`, `true_action` is evaluated; if `condition` is `FALSE`, the optional `false_action` is evaluated. 

Typically the actions are compound statements contained within `{`:


```r
grade <- function(x) {
  if (x > 90) {
    "A"
  } else if (x > 80) {
    "B"
  } else if (x > 50) {
    "C"
  } else {
    "F"
  }
}
```

`if` returns a value so that you can assign the results:


```r
x1 <- if (TRUE) 1 else 2
x2 <- if (FALSE) 1 else 2

c(x1, x2)
```

```
## [1] 1 2
```

(I recommend assigning the results of an `if` statement only when the entire expression fits on one line; otherwise it tends to be hard to read.)

When you use the single argument form without an else statement, `if` invisibly (Section \@ref(invisible)) returns `NULL` if the condition is `FALSE`. Since functions like `c()` and `paste()` drop `NULL` inputs, this allows for a compact expression of certain idioms:


```r
greet <- function(name, birthday = FALSE) {
  paste0(
    "Hi ", name,
    if (birthday) " and HAPPY BIRTHDAY"
  )
}
greet("Maria", FALSE)
```

```
## [1] "Hi Maria"
```

```r
greet("Jaime", TRUE)
```

```
## [1] "Hi Jaime and HAPPY BIRTHDAY"
```

### 5.2.1 Invalid inputs

The `condition` should evaluate to a single `TRUE` or `FALSE`. Most other inputs will generate an error:


```r
# if ("x") 1 # Error in if ("x") 1 : argument is not interpretable as logical
# if (logical()) 1 # Error in if (logical()) 1 : argument is of length zero
# if (NA) 1 # Error in if (NA) 1 : missing value where TRUE/FALSE needed
```

The exception is a logical vector of length greater than 1, which generates a warning:




```r
# if (c(TRUE, FALSE)) 1
```
> Error in if (c(TRUE, FALSE)) 1 : the condition has length > 1


In R 3.5.0 and greater, thanks to [Henrik Bengtsson](https://github.com/HenrikBengtsson/Wishlist-for-R/issues/38), you can turn this into an error by setting an environment variable:


```r
Sys.setenv("_R_CHECK_LENGTH_1_CONDITION_" = "true")
# if (c(TRUE, FALSE)) 1 # Error in if (c(TRUE, FALSE)) 1 : the condition has length > 1
```

I think this is good practice as it reveals a clear mistake that you might otherwise miss if it were only shown as a warning.

### 5.2.2 Vectorised if
\indexc{ifelse()}

Given that `if` only works with a single `TRUE` or `FALSE`, you might wonder what to do if you have a vector of logical values. Handling vectors of values is the job of `ifelse()`: a vectorised function with `test`, `yes`, and `no` vectors (that will be recycled to the same length):


```r
x <- 1:10
ifelse(x %% 5 == 0, "XXX", as.character(x))
```

```
##  [1] "1"   "2"   "3"   "4"   "XXX" "6"   "7"   "8"   "9"   "XXX"
```

```r
ifelse(x %% 2 == 0, "even", "odd")
```

```
##  [1] "odd"  "even" "odd"  "even" "odd"  "even" "odd"  "even" "odd"  "even"
```

Note that missing values will be propagated into the output.

I recommend using `ifelse()` only when the `yes` and `no` vectors are the same type as it is otherwise hard to predict the output type. See <https://vctrs.r-lib.org/articles/stability.html#ifelse> for additional discussion.

Another vectorised equivalent is the more general `dplyr::case_when()`. It uses a special syntax to allow any number of condition-vector pairs:


```r
dplyr::case_when(
  x %% 35 == 0 ~ "fizz buzz",
  x %% 5 == 0 ~ "fizz",
  x %% 7 == 0 ~ "buzz",
  is.na(x) ~ "???",
  TRUE ~ as.character(x)
)
```

```
##  [1] "1"    "2"    "3"    "4"    "fizz" "6"    "buzz" "8"    "9"    "fizz"
```

### 5.2.3 `switch()` statement {#switch}
\indexc{switch()}

Closely related to `if` is the `switch()`-statement. It's a compact, special purpose equivalent that lets you replace code like:


```r
x_option <- function(x) {
  if (x == "a") {
    "option 1"
  } else if (x == "b") {
    "option 2" 
  } else if (x == "c") {
    "option 3"
  } else {
    stop("Invalid `x` value")
  }
}
```

with the more succinct:


```r
x_option <- function(x) {
  switch(x,
    a = "option 1",
    b = "option 2",
    c = "option 3",
    stop("Invalid `x` value")
  )
}
```

The last component of a `switch()` should always throw an error, otherwise unmatched inputs will invisibly return `NULL`:


```r
(switch("c", a = 1, b = 2))
```

```
## NULL
```

If multiple inputs have the same output, you can leave the right hand side of `=` empty and the input will "fall through" to the next value. This mimics the behaviour of C's `switch` statement:


```r
legs <- function(x) {
  switch(x,
    cow = ,
    horse = ,
    dog = 4,
    human = ,
    chicken = 2,
    plant = 0,
    stop("Unknown input")
  )
}
legs("cow")
```

```
## [1] 4
```

```r
legs("dog")
```

```
## [1] 4
```

It is also possible to use `switch()` with a numeric `x`, but is harder to read, and has undesirable failure modes if `x` is a not a whole number. I recommend using `switch()` only with character inputs.

### 5.2.4 Exercises

1.  What type of vector does each of the following calls to `ifelse()`
    return?

    
    ```r
    ifelse(TRUE, 1, "no")
    ifelse(FALSE, 1, "no")
    ifelse(NA, 1, "no")
    ```

> The arguments of ifelse() are named test, yes and no. ifelse() returns the entry for yes when test is TRUE, the entry for no when test is FALSE, and NA when test is NA. Therefore, the expressions above return vectors of type double (1), character ("no") and logical (NA).

    Read the documentation and write down the rules in your own words.
    

```r
# ?ifelse()

utils::str(ifelse(TRUE, 1, "no"))
```

```
##  num 1
```

```r
utils::str(ifelse(FALSE, 1, "no"))
```

```
##  chr "no"
```

```r
utils::str(ifelse(NA, 1, "no"))
```

```
##  logi NA
```

> `ifelse(TRUE, 1, "no")` number, double vector

> `ifelse(FALSE, 1, "no")` character vector

> `ifelse(NA, 1, "no")` logical vector

2.  Why does the following code work?

    
    ```r
    x <- 1:10
    length(x)
    ```
    
    ```
    ## [1] 10
    ```
    
    ```r
    if (length(x)) "not empty" else "empty"
    ```
    
    ```
    ## [1] "not empty"
    ```
    
    ```r
    x <- numeric()
    length(x)
    ```
    
    ```
    ## [1] 0
    ```
    
    ```r
    if (length(x)) "not empty" else "empty"
    ```
    
    ```
    ## [1] "empty"
    ```

> 0 is treated as FALSE and all other numbers are treated as TRUE.

## 5.3 Loops
\index{loops}
\index{loops!for@\texttt{for}}
\indexc{for}

`for` loops are used to iterate over items in a vector. They have the following basic form:


```r
# for (item in vector) perform_action
```

For each item in `vector`, `perform_action` is called once; updating the value of `item` each time.


```r
for (i in 1:3) {
  print(i)
}
```

```
## [1] 1
## [1] 2
## [1] 3
```

(When iterating over a vector of indices, it's conventional to use very short variable names like `i`, `j`, or `k`.)

N.B.: `for` assigns the `item` to the current environment, overwriting any existing variable with the same name:


```r
i <- 100
for (i in 1:3) {}
i
```

```
## [1] 3
```

\indexc{next}
\indexc{break}
There are two ways to terminate a `for` loop early:

* `next` exits the current iteration.
* `break` exits the entire `for` loop.


```r
for (i in 1:10) {
  if (i < 3) 
    next

  print(i)
  
  if (i >= 5)
    break
}
```

```
## [1] 3
## [1] 4
## [1] 5
```

### 5.3.1 Common pitfalls
\index{loops!common pitfalls}

There are three common pitfalls to watch out for when using `for`. First, if you're generating data, make sure to preallocate the output container. Otherwise the loop will be very slow; see Sections \@ref(memory-profiling) and \@ref(avoid-copies) for more details. The `vector()` function is helpful here.


```r
means <- c(1, 50, 20)
out <- vector("list", length(means))
for (i in 1:length(means)) {
  out[[i]] <- rnorm(10, means[[i]])
}
```


```r
out
```

```
## [[1]]
##  [1]  2.273095269  1.114740208  1.097374308 -0.132010140  1.658537898
##  [6]  1.675372166  1.968531513  2.798526992  0.002700037  0.434845761
## 
## [[2]]
##  [1] 47.71532 50.24373 49.37676 49.94933 49.53847 47.00357 50.49425 48.83755
##  [9] 48.89843 49.14021
## 
## [[3]]
##  [1] 19.70867 20.85021 19.71076 19.82436 20.27944 19.51235 20.80037 20.86945
##  [9] 19.86296 19.91965
```

```r
str(out)
```

```
## List of 3
##  $ : num [1:10] 2.273 1.115 1.097 -0.132 1.659 ...
##  $ : num [1:10] 47.7 50.2 49.4 49.9 49.5 ...
##  $ : num [1:10] 19.7 20.9 19.7 19.8 20.3 ...
```


Next, beware of iterating over `1:length(x)`, which will fail in unhelpful ways if `x` has length 0:


```r
means <- c()
out <- vector("list", length(means))
for (i in 1:length(means)) {
 out[[i]] <- rnorm(10, means[[i]])
}
```

```
## Error in rnorm(10, means[[i]]): invalid arguments
```

> Error in rnorm(10, means[[i]]) : invalid arguments

This occurs because `:` works with both increasing and decreasing sequences:


```r
1:length(means)
```

```
## [1] 1 0
```

Use `seq_along(x)` instead. It always returns a value the same length as `x`:


```r
out <- vector("list", length(means))
for (i in seq_along(means)) {
  out[[i]] <- rnorm(10, means[[i]])
}
```


```r
seq_along(means)
```

```
## integer(0)
```


Finally, you might encounter problems when iterating over S3 vectors, as loops typically strip the attributes:


```r
xs <- as.Date(c("2020-01-01", "2010-01-01"))
for (x in xs) {
  print(x)
}
```

```
## [1] 18262
## [1] 14610
```

Work around this by calling `[[` yourself:


```r
for (i in seq_along(xs)) {
  print(xs[[i]])
}
```

```
## [1] "2020-01-01"
## [1] "2010-01-01"
```

### 5.3.2 Related tools {#for-family}
\indexc{while}
\indexc{repeat}

`for` loops are useful if you know in advance the set of values that you want to iterate over. If you don't know, there are two related tools with more flexible specifications:

* `while(condition) action`: performs `action` while `condition` is `TRUE`.

* `repeat(action)`: repeats `action` forever (i.e. until it encounters `break`).

R does not have an equivalent to the `do {action} while (condition)` syntax found in other languages.

You can rewrite any `for` loop to use `while` instead, and you can rewrite any `while` loop to use `repeat`, but the converses are not true. That means `while` is more flexible than `for`, and `repeat` is more flexible than `while`. It's good practice, however, to use the least-flexible solution to a problem, so you should use `for` wherever possible.

Generally speaking you shouldn't need to use `for` loops for data analysis tasks, as `map()` and `apply()` already provide less flexible solutions to most problems. You'll learn more in Chapter \@ref(functionals).

### 5.3.3 Exercises

1.  Why does this code succeed without errors or warnings? 
    
    
    ```r
    x <- numeric()
    out <- vector("list", length(x))
    out
    str(out)
    length(x)
    
    for (i in 1:length(x)) {
      out[i] <- x[i] ^ 2
    }
    out
    ```




> As x has length 0, 1:length(x) counts down from 1 to 0. The first iteration x[1] will generate an NA. The second iteration x[0] will return numeric(0), which will assign a 0-length vector to a 0-length subset. It works but doesn’t change the object.

> During the first iteration x[1] will generate an NA (out-of-bounds indexing for atomics). The resulting NA (from squaring) will be assigned to the empty length-1 list out[1] (out-of-bounds indexing for lists).

> The next iteration, x[0] will return numeric(0) (zero indexing for atomics). Again, squaring doesn’t change the value and numeric(0) is assigned to out[0] (zero indexing for lists). Assigning a 0-length vector to a 0-length subset works but doesn’t change the object.

2.  When the following code is evaluated, what can you say about the 
    vector being iterated?

    
    ```r
    xs <- c(1, 2, 3)
    for (x in xs) {
      xs <- c(xs, x * 2)
    }
    xs
    ```
    
    ```
    ## [1] 1 2 3 2 4 6
    ```

> the inputs are evaluated just once in the beginning of the loop. Otherwise, we would run into an infinite loop.

3.  What does the following code tell you about when the index is updated?

    
    ```r
    for (i in 1:3) {
      i <- i * 2
      print(i) 
    }
    ```
    
    ```
    ## [1] 2
    ## [1] 4
    ## [1] 6
    ```

> the index is updated in the beginning of each iteration. Therefore, reassigning the index symbol during one iteration doesn’t affect the following iterations. (Again, we would otherwise run into an infinite loop.)

## Quiz answers {#control-flow-answers}

* `if` works with scalars; `ifelse()` works with vectors.

* When `x` is `TRUE`, `y` will be `3`; when `FALSE`, `y` will be `NULL`;
  when `NA` the if statement will throw an error.

* This `switch()` statement makes use of fall-through so it will return 2.
  See details in Section \@ref(switch).

