---
title: "Ch4_Subsetting"
author: "Min-Yao"
date: "2022-09-17"
output: 
  html_document: 
    keep_md: yes
---

# 4 Subsetting {#subsetting}



## 4.1 Introduction
\index{subsetting}

R's subsetting operators are fast and powerful. Mastering them allows you to succinctly perform complex operations in a way that few other languages can match. Subsetting in R is easy to learn but hard to master because you need to internalise a number of interrelated concepts:

* There are six ways to subset atomic vectors.

* There are three subsetting operators, `[[`, `[`, and `$`.

* Subsetting operators interact differently with different vector 
  types (e.g., atomic vectors, lists, factors, matrices, and data frames).

* Subsetting can be combined with assignment.

Subsetting is a natural complement to `str()`. While `str()` shows you all the pieces of any object (its structure), subsetting allows you to pull out the pieces that you're interested in. For large, complex objects, I highly recommend using the interactive RStudio Viewer, which you can activate with `View(my_object)`.

### Quiz {-}

Take this short quiz to determine if you need to read this chapter. If the answers quickly come to mind, you can comfortably skip this chapter. Check your answers in Section \@ref(subsetting-answers).

1.  What is the result of subsetting a vector with positive integers, 
    negative integers, a logical vector, or a character vector?
    
> Before reading: Positive integers indicate selecting elements at a specific position in the vector. Negative integers indicate excluding elements at a specific position in the vector. A character vector indicates selecting elements that match the same name. I am not sure about a logical vector. I guess it means to use Ture or False to choose elements in the vector.

> After reading: Positive integers select elements at specific positions, negative integers drop elements; logical vectors keep elements at positions corresponding to `TRUE`; character vectors select elements with matching names.

2.  What's the difference between `[`, `[[`, and `$` when applied to a list?

> Before reading: `$` means to select a specific element in a list. `[` subset a list and return a new list. I am not sure about `[[`.  I guess it means similar to `$`.

> After reading: `[` selects sub-lists: it always returns a list. If you use it with a single positive integer, it returns a list of length one. `[[` selects an element within a list. `$` is a convenient shorthand: `x$y` is equivalent to `x[["y"]]`.

3.  When should you use `drop = FALSE`?

> Before reading: I don't know. :)

> After reading: Use `drop = FALSE` if you are subsetting a matrix, array, or data frame and you want to preserve the original dimensions. You should almost always use it when subsetting inside a function.

4.  If `x` is a matrix, what does `x[] <- 0` do? How is it different from
    `x <- 0`?
    
> Before reading: If `x` is a matrix, `x <- 0` assign number 0 to be `x`. On the other hand, `x[] <- 0` assign the number 0 to each individual element inside the matrix. Therefore, the output is a matrix with the same dimensions but all elements in the matrix are 0s.

> After reading: If `x` is a matrix, `x[] <- 0` will replace every element with 0, keeping the same number of rows and columns. In contrast, `x <- 0` completely replaces the matrix with the value 0.

5.  How can you use a named vector to relabel categorical variables?

> Before reading: I don't know. :)

> After reading: A named character vector can act as a simple lookup table: `c(x = 1, y = 2, z = 3)[c("y", "z", "x")]`

### Outline {-}

* Section \@ref(subset-multiple) starts by teaching you about `[`. 
  You'll learn the six ways to subset atomic vectors. You'll then 
  learn how those six ways act when used to subset lists, matrices, 
  and data frames.
  
* Section \@ref(subset-single) expands your knowledge of subsetting 
  operators to include `[[` and `$` and focuses on the important 
  principles of simplifying versus preserving.
  
* In Section \@ref(subassignment) you'll learn the art of 
  subassignment, which combines subsetting and assignment to modify 
  parts of an object.
  
* Section \@ref(applications) leads you through eight important, but
  not obvious, applications of subsetting to solve problems that you
  often encounter in data analysis.

## 4.2 Selecting multiple elements {#subset-multiple}
\indexc{[}

Use `[` to select any number of elements from a vector. To illustrate, I'll apply `[` to 1D atomic vectors, and then show how this generalises to more complex objects and more dimensions.

### 4.2.1 Atomic vectors
\index{subsetting!atomic vectors} 
\index{atomic vectors!subsetting} 

Let's explore the different types of subsetting with a simple vector, `x`.  


```r
x <- c(2.1, 4.2, 3.3, 5.4)
```

Note that the number after the decimal point represents the original position in the vector.

There are six things that you can use to subset a vector: 

*   __Positive integers__ return elements at the specified positions: 

    
    ```r
    x[c(3, 1)]
    ```
    
    ```
    ## [1] 3.3 2.1
    ```
    
    ```r
    x[order(x)]
    ```
    
    ```
    ## [1] 2.1 3.3 4.2 5.4
    ```
    
    ```r
    # Duplicate indices will duplicate values
    x[c(1, 1)]
    ```
    
    ```
    ## [1] 2.1 2.1
    ```
    
    ```r
    # Real numbers are silently truncated to integers
    x[c(2.1, 2.9)]
    ```
    
    ```
    ## [1] 4.2 4.2
    ```


```r
order(x)
```

```
## [1] 1 3 2 4
```

```r
x[order(x)]
```

```
## [1] 2.1 3.3 4.2 5.4
```


*   __Negative integers__ exclude elements at the specified positions:

    
    ```r
    x[-c(3, 1)]
    ```
    
    ```
    ## [1] 4.2 5.4
    ```

    Note that you can't mix positive and negative integers in a single subset:

    
    ```r
    #    x[c(-1, 2)]
    ```

*   __Logical vectors__ select elements where the corresponding logical 
    value is `TRUE`. This is probably the most useful type of subsetting
    because you can write an expression that uses a logical vector:
    
    
    ```r
    x[c(TRUE, TRUE, FALSE, FALSE)]
    ```
    
    ```
    ## [1] 2.1 4.2
    ```
    
    ```r
    x[x > 3]
    ```
    
    ```
    ## [1] 4.2 3.3 5.4
    ```

    \index{recycling}
    In `x[y]`, what happens if `x` and `y` are different lengths? The behaviour 
    is controlled by the __recycling rules__ where the shorter of the two is
    recycled to the length of the longer. This is convenient and easy to
    understand when one of `x` and `y` is length one, but I recommend avoiding
    recycling for other lengths because the rules are inconsistently applied
    throughout base R.
  
    
    ```r
    x[c(TRUE, FALSE)]
    ```
    
    ```
    ## [1] 2.1 3.3
    ```
    
    ```r
    # Equivalent to
    x[c(TRUE, FALSE, TRUE, FALSE)]
    ```
    
    ```
    ## [1] 2.1 3.3
    ```

    Note that a missing value in the index always yields a missing value in the output:

    
    ```r
    x[c(TRUE, TRUE, NA, FALSE)]
    ```
    
    ```
    ## [1] 2.1 4.2  NA
    ```

*   __Nothing__ returns the original vector. This is not useful for 1D vectors,
    but, as you'll see shortly, is very useful for matrices, data frames, and arrays. 
    It can also be useful in conjunction with assignment.

    
    ```r
    x[]
    ```
    
    ```
    ## [1] 2.1 4.2 3.3 5.4
    ```

*   __Zero__ returns a zero-length vector. This is not something you 
    usually do on purpose, but it can be helpful for generating test data.

    
    ```r
    x[0]
    ```
    
    ```
    ## numeric(0)
    ```

*   If the vector is named, you can also use __character vectors__ to return
    elements with matching names.

    
    ```r
    (y <- setNames(x, letters[1:4]))
    ```
    
    ```
    ##   a   b   c   d 
    ## 2.1 4.2 3.3 5.4
    ```
    
    ```r
    y[c("d", "c", "a")]
    ```
    
    ```
    ##   d   c   a 
    ## 5.4 3.3 2.1
    ```
    
    ```r
    # Like integer indices, you can repeat indices
    y[c("a", "a", "a")]
    ```
    
    ```
    ##   a   a   a 
    ## 2.1 2.1 2.1
    ```
    
    ```r
    # When subsetting with [, names are always matched exactly
    z <- c(abc = 1, def = 2)
    z[c("a", "d")]
    ```
    
    ```
    ## <NA> <NA> 
    ##   NA   NA
    ```

NB: Factors are not treated specially when subsetting. This means that subsetting will use the underlying integer vector, not the character levels. This is typically unexpected, so you should avoid subsetting with factors:


```r
y
```

```
##   a   b   c   d 
## 2.1 4.2 3.3 5.4
```

```r
y[factor("b")]
```

```
##   a 
## 2.1
```

```r
as.integer(factor("b"))
```

```
## [1] 1
```

```r
as.integer(factor("b","a"))
```

```
## [1] NA
```

### 4.2.2 Lists
\index{lists!subsetting} 
\index{subsetting!lists}

Subsetting a list works in the same way as subsetting an atomic vector. Using `[` always returns a list; `[[` and `$`, as described in Section \@ref(subset-single), let you pull out elements of a list.  

### 4.2.3 Matrices and arrays {#matrix-subsetting}
\index{subsetting!arrays} 
\index{arrays!subsetting}

You can subset higher-dimensional structures in three ways: 

* With multiple vectors.
* With a single vector.
* With a matrix.

The most common way of subsetting matrices (2D) and arrays (>2D) is a simple generalisation of 1D subsetting: supply a 1D index for each dimension, separated by a comma. Blank subsetting is now useful because it lets you keep all rows or all columns.


```r
a <- matrix(1:9, nrow = 3)
colnames(a) <- c("A", "B", "C")
a[1:2, ]
```

```
##      A B C
## [1,] 1 4 7
## [2,] 2 5 8
```

```r
a[c(TRUE, FALSE, TRUE), c("B", "A")]
```

```
##      B A
## [1,] 4 1
## [2,] 6 3
```

```r
a[0, -2]
```

```
##      A C
```

By default, `[` simplifies the results to the lowest possible dimensionality. For example, both of the following expressions return 1D vectors. You'll learn how to avoid "dropping" dimensions in Section \@ref(simplify-preserve):


```r
a[1, ]
```

```
## A B C 
## 1 4 7
```

```r
a[1, 1]
```

```
## A 
## 1
```

Because both matrices and arrays are just vectors with special attributes, you can subset them with a single vector, as if they were a 1D vector. Note that arrays in R are stored in column-major order:


```r
vals <- outer(1:5, 1:5, FUN = "paste", sep = ",")
vals
```

```
##      [,1]  [,2]  [,3]  [,4]  [,5] 
## [1,] "1,1" "1,2" "1,3" "1,4" "1,5"
## [2,] "2,1" "2,2" "2,3" "2,4" "2,5"
## [3,] "3,1" "3,2" "3,3" "3,4" "3,5"
## [4,] "4,1" "4,2" "4,3" "4,4" "4,5"
## [5,] "5,1" "5,2" "5,3" "5,4" "5,5"
```

```r
vals[c(4, 15)]
```

```
## [1] "4,1" "5,3"
```

You can also subset higher-dimensional data structures with an integer matrix (or, if named, a character matrix). Each row in the matrix specifies the location of one value, and each column corresponds to a dimension in the array. This means that you can use a 2 column matrix to subset a matrix, a 3 column matrix to subset a 3D array, and so on. The result is a vector of values:


```r
select <- matrix(ncol = 2, byrow = TRUE, c(
  1, 1,
  3, 1,
  2, 4
))
vals[select]
```

```
## [1] "1,1" "3,1" "2,4"
```

### 4.2.4 Data frames and tibbles {#df-subsetting}
\index{subsetting!data frames} 
\index{data frames!subsetting}

Data frames have the characteristics of both lists and matrices: 

* When subsetting with a single index, they behave like lists and index 
  the columns, so `df[1:2]` selects the first two columns.
  
* When subsetting with two indices, they behave like matrices, so
  `df[1:3, ]` selects the first three _rows_ (and all the columns)[^python-dims].

[^python-dims]: If you're coming from Python this is likely to be confusing, as you'd probably expect `df[1:3, 1:2]` to select three columns and two rows. Generally, R "thinks" about dimensions in terms of rows and columns while Python does so in terms of columns and rows.


```r
df <- data.frame(x = 1:3, y = 3:1, z = letters[1:3])

df[df$x == 2, ]
```

```
##   x y z
## 2 2 2 b
```

```r
df[c(1, 3), ]
```

```
##   x y z
## 1 1 3 a
## 3 3 1 c
```

```r
# There are two ways to select columns from a data frame
# Like a list
df[c("x", "z")]
```

```
##   x z
## 1 1 a
## 2 2 b
## 3 3 c
```

```r
# Like a matrix
df[, c("x", "z")]
```

```
##   x z
## 1 1 a
## 2 2 b
## 3 3 c
```

```r
# There's an important difference if you select a single 
# column: matrix subsetting simplifies by default, list 
# subsetting does not.
str(df["x"])
```

```
## 'data.frame':	3 obs. of  1 variable:
##  $ x: int  1 2 3
```

```r
str(df[, "x"])
```

```
##  int [1:3] 1 2 3
```

Subsetting a tibble with `[` always returns a tibble:


```r
df <- tibble::tibble(x = 1:3, y = 3:1, z = letters[1:3])

str(df["x"])
```

```
## tibble [3 × 1] (S3: tbl_df/tbl/data.frame)
##  $ x: int [1:3] 1 2 3
```

```r
str(df[, "x"])
```

```
## tibble [3 × 1] (S3: tbl_df/tbl/data.frame)
##  $ x: int [1:3] 1 2 3
```

### 4.2.5 Preserving dimensionality {#simplify-preserve}
\indexc{drop = FALSE} 
\index{subsetting!simplifying} 
\index{subsetting!preserving}

By default, subsetting a matrix or data frame with a single number, a single name, or a logical vector containing a single `TRUE`, will simplify the returned output, i.e. it will return an object with lower dimensionality. To preserve the original dimensionality, you must use `drop = FALSE`.

*   For matrices and arrays, any dimensions with length 1 will be dropped:
    
    
    ```r
    a <- matrix(1:4, nrow = 2)
    str(a[1, ])
    ```
    
    ```
    ##  int [1:2] 1 3
    ```
    
    ```r
    str(a[1, , drop = FALSE])
    ```
    
    ```
    ##  int [1, 1:2] 1 3
    ```

*   Data frames with a single column will return just the content of that column:

    
    ```r
    df <- data.frame(a = 1:2, b = 1:2)
    str(df[, "a"])
    ```
    
    ```
    ##  int [1:2] 1 2
    ```
    
    ```r
    str(df[, "a", drop = FALSE])
    ```
    
    ```
    ## 'data.frame':	2 obs. of  1 variable:
    ##  $ a: int  1 2
    ```

The default `drop = TRUE` behaviour is a common source of bugs in functions: you check your code with a data frame or matrix with multiple columns, and it works. Six months later, you (or someone else) uses it with a single column data frame and it fails with a mystifying error. When writing functions, get in the habit of always using `drop = FALSE` when subsetting a 2D object. For this reason, tibbles default to `drop = FALSE`, and `[` always returns another tibble.

Factor subsetting also has a `drop` argument, but its meaning is rather different. It controls whether or not levels (rather than dimensions) are preserved, and it defaults to `FALSE`. If you find you're using `drop = TRUE` a lot it's often a sign that you should be using a character vector instead of a factor.


```r
z <- factor(c("a", "b"))
z[1]
```

```
## [1] a
## Levels: a b
```

```r
z[1, drop = TRUE]
```

```
## [1] a
## Levels: a
```

### Exercises

1.  Fix each of the following common data frame subsetting errors:


```r
str(mtcars)
```

```
## 'data.frame':	32 obs. of  11 variables:
##  $ mpg : num  21 21 22.8 21.4 18.7 18.1 14.3 24.4 22.8 19.2 ...
##  $ cyl : num  6 6 4 6 8 6 8 4 4 6 ...
##  $ disp: num  160 160 108 258 360 ...
##  $ hp  : num  110 110 93 110 175 105 245 62 95 123 ...
##  $ drat: num  3.9 3.9 3.85 3.08 3.15 2.76 3.21 3.69 3.92 3.92 ...
##  $ wt  : num  2.62 2.88 2.32 3.21 3.44 ...
##  $ qsec: num  16.5 17 18.6 19.4 17 ...
##  $ vs  : num  0 0 1 1 0 1 0 1 1 1 ...
##  $ am  : num  1 1 1 0 0 0 0 0 0 0 ...
##  $ gear: num  4 4 4 3 3 3 3 4 4 4 ...
##  $ carb: num  4 4 1 1 2 1 4 2 2 4 ...
```

    
    ```r
    #mtcars[mtcars$cyl = 4, ]
    mtcars[mtcars$cyl == 4, ]
    #mtcars[-1:4, ]
    mtcars[-c(1:4), ]
    #mtcars[mtcars$cyl <= 5]
    mtcars[mtcars$cyl <= 5, ]
    mtcars[mtcars$cyl == 4 || 6, ]
    #mtcars[mtcars$cyl == (4 : 6), ]
    mtcars[mtcars$cyl == 4 | mtcars$cyl == 6, ]
    mtcars[mtcars$cyl %in% c(4, 6), ]
    ```


2.  Why does the following code yield five missing values? (Hint: why is 
    it different from `x[NA_real_]`?)
    
    
    ```r
    x <- 1:5
    x[NA]
    ```
    
    ```
    ## [1] NA NA NA NA NA
    ```
    
> NA has logical type and logical vectors are recycled to the same length as the vector being subset, i.e. x[NA] is recycled to x[NA, NA, NA, NA, NA].

> Note that a missing value in the index always yields a missing value in the output
    
3.  What does `upper.tri()` return? How does subsetting a matrix with it 
    work? Do we need any additional subsetting rules to describe its behaviour?
    
> Lower and Upper Triangular Part of a Matrix: Returns a matrix of logicals the same size of a given matrix with entries TRUE in the lower or upper triangle.
    

```r
(m2 <- matrix(1:20, 4, 5))
```

```
##      [,1] [,2] [,3] [,4] [,5]
## [1,]    1    5    9   13   17
## [2,]    2    6   10   14   18
## [3,]    3    7   11   15   19
## [4,]    4    8   12   16   20
```

```r
lower.tri(m2)
```

```
##       [,1]  [,2]  [,3]  [,4]  [,5]
## [1,] FALSE FALSE FALSE FALSE FALSE
## [2,]  TRUE FALSE FALSE FALSE FALSE
## [3,]  TRUE  TRUE FALSE FALSE FALSE
## [4,]  TRUE  TRUE  TRUE FALSE FALSE
```

```r
m2[lower.tri(m2)] <- NA
m2
```

```
##      [,1] [,2] [,3] [,4] [,5]
## [1,]    1    5    9   13   17
## [2,]   NA    6   10   14   18
## [3,]   NA   NA   11   15   19
## [4,]   NA   NA   NA   16   20
```

> upper.tri(x) returns a logical matrix, which contains TRUE values above the diagonal and FALSE values everywhere else. In upper.tri() the positions for TRUE and FALSE values are determined by comparing x’s row and column indices via .row(dim(x)) < .col(dim(x)).

    
    ```r
    x <- outer(1:5, 1:5, FUN = "*")
    x
    upper.tri(x)
    ```
    

```r
    x[upper.tri(x)]
```

```
## integer(0)
```
    
> When subsetting with logical matrices, all elements that correspond to TRUE will be selected. Matrices extend vectors with a dimension attribute, so the vector forms of subsetting can be used (including logical subsetting). We should take care, that the dimensions of the subsetting matrix match the object of interest — otherwise unintended selections due to vector recycling may occur. Please also note, that this form of subsetting returns a vector instead of a matrix, as the subsetting alters the dimensions of the object.

4.  Why does `mtcars[1:20]` return an error? How does it differ from the 
    similar `mtcars[1:20, ]`?


```r
#mtcars[1:20]
#Error in `[.data.frame`(mtcars, 1:20) : undefined columns selected

mtcars[1:20, ]
```

```
##                      mpg cyl  disp  hp drat    wt  qsec vs am gear carb
## Mazda RX4           21.0   6 160.0 110 3.90 2.620 16.46  0  1    4    4
## Mazda RX4 Wag       21.0   6 160.0 110 3.90 2.875 17.02  0  1    4    4
## Datsun 710          22.8   4 108.0  93 3.85 2.320 18.61  1  1    4    1
## Hornet 4 Drive      21.4   6 258.0 110 3.08 3.215 19.44  1  0    3    1
## Hornet Sportabout   18.7   8 360.0 175 3.15 3.440 17.02  0  0    3    2
## Valiant             18.1   6 225.0 105 2.76 3.460 20.22  1  0    3    1
## Duster 360          14.3   8 360.0 245 3.21 3.570 15.84  0  0    3    4
## Merc 240D           24.4   4 146.7  62 3.69 3.190 20.00  1  0    4    2
## Merc 230            22.8   4 140.8  95 3.92 3.150 22.90  1  0    4    2
## Merc 280            19.2   6 167.6 123 3.92 3.440 18.30  1  0    4    4
## Merc 280C           17.8   6 167.6 123 3.92 3.440 18.90  1  0    4    4
## Merc 450SE          16.4   8 275.8 180 3.07 4.070 17.40  0  0    3    3
## Merc 450SL          17.3   8 275.8 180 3.07 3.730 17.60  0  0    3    3
## Merc 450SLC         15.2   8 275.8 180 3.07 3.780 18.00  0  0    3    3
## Cadillac Fleetwood  10.4   8 472.0 205 2.93 5.250 17.98  0  0    3    4
## Lincoln Continental 10.4   8 460.0 215 3.00 5.424 17.82  0  0    3    4
## Chrysler Imperial   14.7   8 440.0 230 3.23 5.345 17.42  0  0    3    4
## Fiat 128            32.4   4  78.7  66 4.08 2.200 19.47  1  1    4    1
## Honda Civic         30.4   4  75.7  52 4.93 1.615 18.52  1  1    4    2
## Toyota Corolla      33.9   4  71.1  65 4.22 1.835 19.90  1  1    4    1
```

> When subsetting a data frame with a single vector, it behaves the same way as subsetting a list of columns. So, mtcars[1:20] would return a data frame containing the first 20 columns of the dataset. However, as mtcars has only 11 columns, the index will be out of bounds and an error is thrown. mtcars[1:20, ] is subsetted with two vectors, so 2d subsetting kicks in, and the first index refers to rows.

5.  Implement your own function that extracts the diagonal entries from a
    matrix (it should behave like `diag(x)` where `x` is a matrix).

> Matrix Diagonals: Extract or replace the diagonal of a matrix, or construct a diagonal matrix.


```r
diag
```

```
## function (x = 1, nrow, ncol, names = TRUE) 
## {
##     if (is.matrix(x)) {
##         if (nargs() > 1L && (nargs() > 2L || any(names(match.call()) %in% 
##             c("nrow", "ncol")))) 
##             stop("'nrow' or 'ncol' cannot be specified when 'x' is a matrix")
##         if ((m <- min(dim(x))) == 0L) 
##             return(vector(typeof(x), 0L))
##         y <- x[1 + 0L:(m - 1L) * (dim(x)[1L] + 1)]
##         if (names) {
##             nms <- dimnames(x)
##             if (is.list(nms) && !any(vapply(nms, is.null, NA)) && 
##                 identical((nm <- nms[[1L]][seq_len(m)]), nms[[2L]][seq_len(m)])) 
##                 names(y) <- nm
##         }
##         return(y)
##     }
##     if (is.array(x) && length(dim(x)) != 1L) 
##         stop("'x' is an array, but not one-dimensional.")
##     if (missing(x)) 
##         n <- nrow
##     else if (length(x) == 1L && nargs() == 1L) {
##         n <- as.integer(x)
##         x <- 1
##     }
##     else n <- length(x)
##     if (!missing(nrow)) 
##         n <- nrow
##     if (missing(ncol)) 
##         ncol <- n
##     .Internal(diag(x, n, ncol))
## }
## <bytecode: 0x000001fb71906908>
## <environment: namespace:base>
```


```r
M <- diag(3)
M
```

```
##      [,1] [,2] [,3]
## [1,]    1    0    0
## [2,]    0    1    0
## [3,]    0    0    1
```

```r
diag(M)
```

```
## [1] 1 1 1
```

> The elements in the diagonal of a matrix have the same row- and column indices. This characteristic can be used to create a suitable numeric matrix used for subsetting.


```r
m <- cbind(1, 1:7) # the '1' (= shorter vector) is recycled
m
```

```
##      [,1] [,2]
## [1,]    1    1
## [2,]    1    2
## [3,]    1    3
## [4,]    1    4
## [5,]    1    5
## [6,]    1    6
## [7,]    1    7
```

```r
m <- cbind(m, 8:14)[, c(1, 3, 2)] # insert a column
m
```

```
##      [,1] [,2] [,3]
## [1,]    1    8    1
## [2,]    1    9    2
## [3,]    1   10    3
## [4,]    1   11    4
## [5,]    1   12    5
## [6,]    1   13    6
## [7,]    1   14    7
```

> cbind: Combine R Objects by Rows or Columns: Take a sequence of vector, matrix or data-frame arguments and combine by columns or rows, respectively. These are generic functions with methods for other R classes.


```r
new_diag <- function(x) {
  n <- min(nrow(x), ncol(x))
  idx <- cbind(seq_len(n), seq_len(n))

  x[idx]
}

# Let's check if it works
(x <- matrix(1:25, 5, 5))
```

```
##      [,1] [,2] [,3] [,4] [,5]
## [1,]    1    6   11   16   21
## [2,]    2    7   12   17   22
## [3,]    3    8   13   18   23
## [4,]    4    9   14   19   24
## [5,]    5   10   15   20   25
```

```r
diag(x)
```

```
## [1]  1  7 13 19 25
```

```r
new_diag(x)
```

```
## [1]  1  7 13 19 25
```


6.  What does `df[is.na(df)] <- 0` do? How does it work?


```r
df
```

```
##   a b
## 1 1 1
## 2 2 2
```

```r
str(df)
```

```
## 'data.frame':	2 obs. of  2 variables:
##  $ a: int  1 2
##  $ b: int  1 2
```

```r
df[is.na(df)] <- 0
df
```

```
##   a b
## 1 1 1
## 2 2 2
```

```r
str(df)
```

```
## 'data.frame':	2 obs. of  2 variables:
##  $ a: int  1 2
##  $ b: int  1 2
```

> This expression replaces the NAs in df with 0. Here is.na(df) returns a logical matrix that encodes the position of the missing values in df. Subsetting and assignment are then combined to replace only the missing values.

## 4.3 Selecting a single element {#subset-single}
\index{subsetting!lists} 
\index{lists!subsetting}

There are two other subsetting operators: `[[` and `$`. `[[` is used for extracting single items, while `x$y` is a useful shorthand for `x[["y"]]`.

### 4.3.1 `[[`
\indexc{[[} 

`[[` is most important when working with lists because subsetting a list with `[` always returns a smaller list. To help make this easier to understand we can use a metaphor:

> If list `x` is a train carrying objects, then `x[[5]]` is
> the object in car 5; `x[4:6]` is a train of cars 4-6.
>
> --- \@RLangTip, <https://twitter.com/RLangTip/status/268375867468681216>

Let's use this metaphor to make a simple list:


```r
x <- list(1:3, "a", 4:6)
```
<img src="diagrams/subsetting/train.png" width="1110" />

When extracting a single element, you have two options: you can create a smaller train, i.e., fewer carriages, or you can extract the contents of a particular carriage. This is the difference between `[` and `[[`:

<img src="diagrams/subsetting/train-single.png" width="1110" />

When extracting multiple (or even zero!) elements, you have to make a smaller train:

<img src="diagrams/subsetting/train-multiple.png" width="1110" />

Because `[[` can return only a single item, you must use it with either a single positive integer or a single string. If you use a vector with `[[`, it will subset recursively, i.e. `x[[c(1, 2)]]` is equivalent to `x[[1]][[2]]`. This is a quirky feature that few know about, so I recommend avoiding it in favour of `purrr::pluck()`, which you'll learn about in Section \@ref(subsetting-oob).

While you must use `[[` when working with lists, I'd also recommend using it with atomic vectors whenever you want to extract a single value. For example, instead of writing:


```r
for (i in 2:length(x)) {
  out[i] <- fun(x[i], out[i - 1])
}
```

It's better to write: 


```r
for (i in 2:length(x)) {
  out[[i]] <- fun(x[[i]], out[[i - 1]])
}
```

Doing so reinforces the expectation that you are getting and setting individual values.

### 4.3.2 `$`
\indexc{\$}

`$` is a shorthand operator: `x$y` is roughly equivalent to `x[["y"]]`.  It's often used to access variables in a data frame, as in `mtcars$cyl` or `diamonds$carat`. One common mistake with `$` is to use it when you have the name of a column stored in a variable:




```r
var <- "cyl"
# Doesn't work - mtcars$var translated to mtcars[["var"]]
mtcars$var
```

```
## NULL
```

```r
# Instead use [[
mtcars[[var]]
```

```
##  [1] 6 6 4 6 8 6 8 4 4 6 6 8 8 8 8 8 8 4 4 4 4 8 8 8 8 4 4 4 8 6 8 4
```

The one important difference between `$` and `[[` is that `$` does (left-to-right) partial matching:


```r
x <- list(abc = 1)
x$a
```

```
## [1] 1
```

```r
x[["a"]]
```

```
## NULL
```

\index{options!warnPartialMatchDollar@\texttt{warnPartialMatchDollar}}
To help avoid this behaviour I highly recommend setting the global option `warnPartialMatchDollar` to `TRUE`:


```r
options(warnPartialMatchDollar = TRUE)
x$a
```

```
## Warning in x$a: partial match of 'a' to 'abc'
```

```
## [1] 1
```

(For data frames, you can also avoid this problem by using tibbles, which never do partial matching.)

### 4.3.3 Missing and out-of-bounds indices {#subsetting-oob}
\index{subsetting!with NA \& NULL} 
\index{subsetting!out of bounds}
\indexc{pluck()}
\indexc{chuck()}

It's useful to understand what happens with `[[` when you use an "invalid" index. The following table summarises what happens when you subset a logical vector, list, and `NULL` with a zero-length object (like `NULL` or `logical()`), out-of-bounds values (OOB), or a missing value (e.g. `NA_integer_`) with `[[`. Each cell shows the result of subsetting the data structure named in the row by the type of index described in the column. I've only shown the results for logical vectors, but other atomic vectors behave similarly, returning elements of the same type (NB: int = integer; chr = character).

| `row[[col]]` | Zero-length | OOB (int)  | OOB (chr) | Missing  |
|--------------|-------------|------------|-----------|----------|
| Atomic       | Error       | Error      | Error     | Error    |
| List         | Error       | Error      | `NULL`    | `NULL`   |
| `NULL`       | `NULL`      | `NULL`     | `NULL`    | `NULL`   |



If the vector being indexed is named, then the names of OOB, missing, or `NULL` components will be `<NA>`.

The inconsistencies in the table above led to the development of `purrr::pluck()` and `purrr::chuck()`. When the element is missing, `pluck()` always returns `NULL` (or the value of the `.default` argument) and `chuck()` always throws an error. The behaviour of `pluck()` makes it well suited for indexing into deeply nested data structures where the component you want may not exist (as is common when working with JSON data from web APIs). `pluck()` also allows you to mix integer and character indices, and provides an alternative default value if an item does not exist:


```r
x <- list(
  a = list(1, 2, 3),
  b = list(3, 4, 5)
)

purrr::pluck(x, "a", 1)
```

```
## [1] 1
```

```r
purrr::pluck(x, "c", 1)
```

```
## NULL
```

```r
purrr::pluck(x, "c", 1, .default = NA)
```

```
## [1] NA
```

### 4.3.4 `@` and `slot()`

There are two additional subsetting operators, which are needed for S4 objects: `@` (equivalent to `$`), and `slot()` (equivalent to `[[`). `@` is more restrictive than `$` in that it will return an error if the slot does not exist. These are described in more detail in Chapter \@ref(s4).

### 4.3.5 Exercises

1.  Brainstorm as many ways as possible to extract the third value from the
    `cyl` variable in the `mtcars` dataset.
    

```r
str(mtcars)
```

```
## 'data.frame':	32 obs. of  11 variables:
##  $ mpg : num  21 21 22.8 21.4 18.7 18.1 14.3 24.4 22.8 19.2 ...
##  $ cyl : num  6 6 4 6 8 6 8 4 4 6 ...
##  $ disp: num  160 160 108 258 360 ...
##  $ hp  : num  110 110 93 110 175 105 245 62 95 123 ...
##  $ drat: num  3.9 3.9 3.85 3.08 3.15 2.76 3.21 3.69 3.92 3.92 ...
##  $ wt  : num  2.62 2.88 2.32 3.21 3.44 ...
##  $ qsec: num  16.5 17 18.6 19.4 17 ...
##  $ vs  : num  0 0 1 1 0 1 0 1 1 1 ...
##  $ am  : num  1 1 1 0 0 0 0 0 0 0 ...
##  $ gear: num  4 4 4 3 3 3 3 4 4 4 ...
##  $ carb: num  4 4 1 1 2 1 4 2 2 4 ...
```

```r
#1
mtcars$cyl[[3]]
```

```
## [1] 4
```

```r
#2
mtcars[ , "cyl"][[3]]
```

```
## [1] 4
```

```r
#3
mtcars[["cyl"]][[3]]
```

```
## [1] 4
```

```r
#4
mtcars[3, ]$cyl
```

```
## [1] 4
```

```r
#5
mtcars[3, "cyl"]
```

```
## [1] 4
```

```r
#6
mtcars[3, ][ , "cyl"]
```

```
## [1] 4
```

```r
#7
mtcars[3, ][["cyl"]]
```

```
## [1] 4
```

```r
#8
mtcars[3, 2]
```

```
## [1] 4
```


2.  Given a linear model, e.g., `mod <- lm(mpg ~ wt, data = mtcars)`, extract
    the residual degrees of freedom. Then extract the R squared from the model
    summary (`summary(mod)`)


```r
mod <- lm(mpg ~ wt, data = mtcars)
summary(mod)
```

```
## 
## Call:
## lm(formula = mpg ~ wt, data = mtcars)
## 
## Residuals:
##     Min      1Q  Median      3Q     Max 
## -4.5432 -2.3647 -0.1252  1.4096  6.8727 
## 
## Coefficients:
##             Estimate Std. Error t value Pr(>|t|)    
## (Intercept)  37.2851     1.8776  19.858  < 2e-16 ***
## wt           -5.3445     0.5591  -9.559 1.29e-10 ***
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## Residual standard error: 3.046 on 30 degrees of freedom
## Multiple R-squared:  0.7528,	Adjusted R-squared:  0.7446 
## F-statistic: 91.38 on 1 and 30 DF,  p-value: 1.294e-10
```

```r
str(mod)
```

```
## List of 12
##  $ coefficients : Named num [1:2] 37.29 -5.34
##   ..- attr(*, "names")= chr [1:2] "(Intercept)" "wt"
##  $ residuals    : Named num [1:32] -2.28 -0.92 -2.09 1.3 -0.2 ...
##   ..- attr(*, "names")= chr [1:32] "Mazda RX4" "Mazda RX4 Wag" "Datsun 710" "Hornet 4 Drive" ...
##  $ effects      : Named num [1:32] -113.65 -29.116 -1.661 1.631 0.111 ...
##   ..- attr(*, "names")= chr [1:32] "(Intercept)" "wt" "" "" ...
##  $ rank         : int 2
##  $ fitted.values: Named num [1:32] 23.3 21.9 24.9 20.1 18.9 ...
##   ..- attr(*, "names")= chr [1:32] "Mazda RX4" "Mazda RX4 Wag" "Datsun 710" "Hornet 4 Drive" ...
##  $ assign       : int [1:2] 0 1
##  $ qr           :List of 5
##   ..$ qr   : num [1:32, 1:2] -5.657 0.177 0.177 0.177 0.177 ...
##   .. ..- attr(*, "dimnames")=List of 2
##   .. .. ..$ : chr [1:32] "Mazda RX4" "Mazda RX4 Wag" "Datsun 710" "Hornet 4 Drive" ...
##   .. .. ..$ : chr [1:2] "(Intercept)" "wt"
##   .. ..- attr(*, "assign")= int [1:2] 0 1
##   ..$ qraux: num [1:2] 1.18 1.05
##   ..$ pivot: int [1:2] 1 2
##   ..$ tol  : num 1e-07
##   ..$ rank : int 2
##   ..- attr(*, "class")= chr "qr"
##  $ df.residual  : int 30
##  $ xlevels      : Named list()
##  $ call         : language lm(formula = mpg ~ wt, data = mtcars)
##  $ terms        :Classes 'terms', 'formula'  language mpg ~ wt
##   .. ..- attr(*, "variables")= language list(mpg, wt)
##   .. ..- attr(*, "factors")= int [1:2, 1] 0 1
##   .. .. ..- attr(*, "dimnames")=List of 2
##   .. .. .. ..$ : chr [1:2] "mpg" "wt"
##   .. .. .. ..$ : chr "wt"
##   .. ..- attr(*, "term.labels")= chr "wt"
##   .. ..- attr(*, "order")= int 1
##   .. ..- attr(*, "intercept")= int 1
##   .. ..- attr(*, "response")= int 1
##   .. ..- attr(*, ".Environment")=<environment: R_GlobalEnv> 
##   .. ..- attr(*, "predvars")= language list(mpg, wt)
##   .. ..- attr(*, "dataClasses")= Named chr [1:2] "numeric" "numeric"
##   .. .. ..- attr(*, "names")= chr [1:2] "mpg" "wt"
##  $ model        :'data.frame':	32 obs. of  2 variables:
##   ..$ mpg: num [1:32] 21 21 22.8 21.4 18.7 18.1 14.3 24.4 22.8 19.2 ...
##   ..$ wt : num [1:32] 2.62 2.88 2.32 3.21 3.44 ...
##   ..- attr(*, "terms")=Classes 'terms', 'formula'  language mpg ~ wt
##   .. .. ..- attr(*, "variables")= language list(mpg, wt)
##   .. .. ..- attr(*, "factors")= int [1:2, 1] 0 1
##   .. .. .. ..- attr(*, "dimnames")=List of 2
##   .. .. .. .. ..$ : chr [1:2] "mpg" "wt"
##   .. .. .. .. ..$ : chr "wt"
##   .. .. ..- attr(*, "term.labels")= chr "wt"
##   .. .. ..- attr(*, "order")= int 1
##   .. .. ..- attr(*, "intercept")= int 1
##   .. .. ..- attr(*, "response")= int 1
##   .. .. ..- attr(*, ".Environment")=<environment: R_GlobalEnv> 
##   .. .. ..- attr(*, "predvars")= language list(mpg, wt)
##   .. .. ..- attr(*, "dataClasses")= Named chr [1:2] "numeric" "numeric"
##   .. .. .. ..- attr(*, "names")= chr [1:2] "mpg" "wt"
##  - attr(*, "class")= chr "lm"
```


```r
#1
mod$df.residual
```

```
## [1] 30
```

```r
#2
mod[["df.residual"]]
```

```
## [1] 30
```

```r
#3
str(summary(mod))
```

```
## List of 11
##  $ call         : language lm(formula = mpg ~ wt, data = mtcars)
##  $ terms        :Classes 'terms', 'formula'  language mpg ~ wt
##   .. ..- attr(*, "variables")= language list(mpg, wt)
##   .. ..- attr(*, "factors")= int [1:2, 1] 0 1
##   .. .. ..- attr(*, "dimnames")=List of 2
##   .. .. .. ..$ : chr [1:2] "mpg" "wt"
##   .. .. .. ..$ : chr "wt"
##   .. ..- attr(*, "term.labels")= chr "wt"
##   .. ..- attr(*, "order")= int 1
##   .. ..- attr(*, "intercept")= int 1
##   .. ..- attr(*, "response")= int 1
##   .. ..- attr(*, ".Environment")=<environment: R_GlobalEnv> 
##   .. ..- attr(*, "predvars")= language list(mpg, wt)
##   .. ..- attr(*, "dataClasses")= Named chr [1:2] "numeric" "numeric"
##   .. .. ..- attr(*, "names")= chr [1:2] "mpg" "wt"
##  $ residuals    : Named num [1:32] -2.28 -0.92 -2.09 1.3 -0.2 ...
##   ..- attr(*, "names")= chr [1:32] "Mazda RX4" "Mazda RX4 Wag" "Datsun 710" "Hornet 4 Drive" ...
##  $ coefficients : num [1:2, 1:4] 37.285 -5.344 1.878 0.559 19.858 ...
##   ..- attr(*, "dimnames")=List of 2
##   .. ..$ : chr [1:2] "(Intercept)" "wt"
##   .. ..$ : chr [1:4] "Estimate" "Std. Error" "t value" "Pr(>|t|)"
##  $ aliased      : Named logi [1:2] FALSE FALSE
##   ..- attr(*, "names")= chr [1:2] "(Intercept)" "wt"
##  $ sigma        : num 3.05
##  $ df           : int [1:3] 2 30 2
##  $ r.squared    : num 0.753
##  $ adj.r.squared: num 0.745
##  $ fstatistic   : Named num [1:3] 91.4 1 30
##   ..- attr(*, "names")= chr [1:3] "value" "numdf" "dendf"
##  $ cov.unscaled : num [1:2, 1:2] 0.38 -0.1084 -0.1084 0.0337
##   ..- attr(*, "dimnames")=List of 2
##   .. ..$ : chr [1:2] "(Intercept)" "wt"
##   .. ..$ : chr [1:2] "(Intercept)" "wt"
##  - attr(*, "class")= chr "summary.lm"
```

```r
summary(mod)$df
```

```
## [1]  2 30  2
```




## 4.4 Subsetting and assignment {#subassignment}
\index{subsetting!subassignment} 
\index{assignment!subassignment}
\index{lists!removing an element}

All subsetting operators can be combined with assignment to modify selected values of an input vector: this is called subassignment. The basic form is `x[i] <- value`:


```r
x <- 1:5
x[c(1, 2)] <- c(101, 102)
x
```

```
## [1] 101 102   3   4   5
```

I recommend that you should make sure that `length(value)` is the same as `length(x[i])`, and that `i` is unique. This is because, while R will recycle if needed, those rules are complex (particularly if `i` contains missing or duplicated values) and may cause problems.

With lists, you can use `x[[i]] <- NULL` to remove a component. To add a literal `NULL`, use `x[i] <- list(NULL)`: 


```r
x <- list(a = 1, b = 2)
x[["b"]] <- NULL
str(x)
```

```
## List of 1
##  $ a: num 1
```

```r
y <- list(a = 1, b = 2)
y["b"] <- list(NULL)
str(y)
```

```
## List of 2
##  $ a: num 1
##  $ b: NULL
```

Subsetting with nothing can be useful with assignment because it preserves the structure of the original object. Compare the following two expressions. In the first, `mtcars` remains a data frame because you are only changing the contents of `mtcars`, not `mtcars` itself. In the second, `mtcars` becomes a list because you are changing the object it is bound to.


```r
mtcars[] <- lapply(mtcars, as.integer)
is.data.frame(mtcars)
```

```
## [1] TRUE
```

```r
mtcars <- lapply(mtcars, as.integer)
is.data.frame(mtcars)
```

```
## [1] FALSE
```



## 4.5 Applications {#applications}

The principles described above have a wide variety of useful applications. Some of the most important are described below. While many of the basic principles of subsetting have already been incorporated into functions like `subset()`, `merge()`, and `dplyr::arrange()`, a deeper understanding of how those principles have been implemented will be valuable when you run into situations where the functions you need don't exist.

### 4.5.1 Lookup tables (character subsetting) {#lookup-tables}
\index{lookup tables}

Character matching is a powerful way to create lookup tables. Say you want to convert abbreviations: 


```r
x <- c("m", "f", "u", "f", "f", "m", "m")
lookup <- c(m = "Male", f = "Female", u = NA)
lookup[x]
```

```
##        m        f        u        f        f        m        m 
##   "Male" "Female"       NA "Female" "Female"   "Male"   "Male"
```

Note that if you don't want names in the result, use `unname()` to remove them.


```r
unname(lookup[x])
```

```
## [1] "Male"   "Female" NA       "Female" "Female" "Male"   "Male"
```

### 4.5.2 Matching and merging by hand (integer subsetting) {#matching-merging}
\index{matching and merging}
\indexc{match()}

You can also have more complicated lookup tables with multiple columns of information. For example, suppose we have a vector of integer grades, and a table that describes their properties:


```r
grades <- c(1, 2, 2, 3, 1)

info <- data.frame(
  grade = 3:1,
  desc = c("Excellent", "Good", "Poor"),
  fail = c(F, F, T)
)
```

Then, let's say we want to duplicate the `info` table so that we have a row for each value in `grades`. An elegant way to do this is by combining `match()` and integer subsetting (`match(needles, haystack)` returns the position where each `needle` is found in the `haystack`).


```r
id <- match(grades, info$grade)
id
```

```
## [1] 3 2 2 1 3
```

```r
info[id, ]
```

```
##     grade      desc  fail
## 3       1      Poor  TRUE
## 2       2      Good FALSE
## 2.1     2      Good FALSE
## 1       3 Excellent FALSE
## 3.1     1      Poor  TRUE
```

If you're matching on multiple columns, you'll need to first collapse them into a single column (with e.g. `interaction()`). Typically, however, you're better off switching to a function designed specifically for joining multiple tables like `merge()`, or `dplyr::left_join()`.

### 4.5.3 Random samples and bootstraps (integer subsetting)
\index{sampling} 
\index{bootstrapping}

You can use integer indices to randomly sample or bootstrap a vector or data frame. Just use `sample(n)` to generate a random permutation of `1:n`, and then use the results to subset the values: 


```r
df <- data.frame(x = c(1, 2, 3, 1, 2), y = 5:1, z = letters[1:5])

# Randomly reorder
df[sample(nrow(df)), ]
```

```
##   x y z
## 2 2 4 b
## 5 2 1 e
## 4 1 2 d
## 1 1 5 a
## 3 3 3 c
```

```r
# Select 3 random rows
df[sample(nrow(df), 3), ]
```

```
##   x y z
## 4 1 2 d
## 5 2 1 e
## 3 3 3 c
```

```r
# Select 6 bootstrap replicates
df[sample(nrow(df), 6, replace = TRUE), ]
```

```
##     x y z
## 2   2 4 b
## 4   1 2 d
## 4.1 1 2 d
## 4.2 1 2 d
## 2.1 2 4 b
## 3   3 3 c
```

The arguments of `sample()` control the number of samples to extract, and also whether sampling is done with or without replacement.

### 4.5.4 Ordering (integer subsetting)
\indexc{order()} 
\index{sorting}
 
`order()` takes a vector as its input and returns an integer vector describing how to order the subsetted vector[^pull-indices]:

[^pull-indices]: These are "pull" indices, i.e., `order(x)[i]` is an index of where each `x[i]` is located. It is not an index of where `x[i]` should be sent.


```r
x <- c("b", "c", "a")
order(x)
```

```
## [1] 3 1 2
```

```r
x[order(x)]
```

```
## [1] "a" "b" "c"
```

To break ties, you can supply additional variables to `order()`. You can also change the order from ascending to descending by using `decreasing = TRUE`. By default, any missing values will be put at the end of the vector; however, you can remove them with `na.last = NA` or put them at the front with `na.last = FALSE`.

For two or more dimensions, `order()` and integer subsetting makes it easy to order either the rows or columns of an object:


```r
# Randomly reorder df
df2 <- df[sample(nrow(df)), 3:1]
df2
```

```
##   z y x
## 2 b 4 2
## 4 d 2 1
## 5 e 1 2
## 1 a 5 1
## 3 c 3 3
```

```r
df2[order(df2$x), ]
```

```
##   z y x
## 4 d 2 1
## 1 a 5 1
## 2 b 4 2
## 5 e 1 2
## 3 c 3 3
```

```r
df2[, order(names(df2))]
```

```
##   x y z
## 2 2 4 b
## 4 1 2 d
## 5 2 1 e
## 1 1 5 a
## 3 3 3 c
```

You can sort vectors directly with `sort()`, or similarly `dplyr::arrange()`, to sort a data frame.

### 4.5.5 Expanding aggregated counts (integer subsetting)

Sometimes you get a data frame where identical rows have been collapsed into one and a count column has been added. `rep()` and integer subsetting make it easy to uncollapse, because we can take advantage of `rep()`s vectorisation: `rep(x, y)` repeats `x[i]` `y[i]` times.


```r
df <- data.frame(x = c(2, 4, 1), y = c(9, 11, 6), n = c(3, 5, 1))
rep(1:nrow(df), df$n)
```

```
## [1] 1 1 1 2 2 2 2 2 3
```

```r
df[rep(1:nrow(df), df$n), ]
```

```
##     x  y n
## 1   2  9 3
## 1.1 2  9 3
## 1.2 2  9 3
## 2   4 11 5
## 2.1 4 11 5
## 2.2 4 11 5
## 2.3 4 11 5
## 2.4 4 11 5
## 3   1  6 1
```


### 4.5.6 Removing columns from data frames (character \mbox{subsetting})

There are two ways to remove columns from a data frame. You can set individual columns to `NULL`: 


```r
df <- data.frame(x = 1:3, y = 3:1, z = letters[1:3])
df$z <- NULL
```

Or you can subset to return only the columns you want:


```r
df <- data.frame(x = 1:3, y = 3:1, z = letters[1:3])
df[c("x", "y")]
```

```
##   x y
## 1 1 3
## 2 2 2
## 3 3 1
```

If you only know the columns you don't want, use set operations to work out which columns to keep:


```r
df[setdiff(names(df), "z")]
```

```
##   x y
## 1 1 3
## 2 2 2
## 3 3 1
```

### 4.5.7 Selecting rows based on a condition (logical subsetting)
\index{subsetting!with logical vectors}
\indexc{subset()}
 
Because logical subsetting allows you to easily combine conditions from multiple columns, it's probably the most commonly used technique for extracting rows out of a data frame.  


```r
mtcars[mtcars$gear == 5, ]
```

```
##                 mpg cyl  disp  hp drat    wt qsec vs am gear carb
## Porsche 914-2  26.0   4 120.3  91 4.43 2.140 16.7  0  1    5    2
## Lotus Europa   30.4   4  95.1 113 3.77 1.513 16.9  1  1    5    2
## Ford Pantera L 15.8   8 351.0 264 4.22 3.170 14.5  0  1    5    4
## Ferrari Dino   19.7   6 145.0 175 3.62 2.770 15.5  0  1    5    6
## Maserati Bora  15.0   8 301.0 335 3.54 3.570 14.6  0  1    5    8
```

```r
mtcars[mtcars$gear == 5 & mtcars$cyl == 4, ]
```

```
##                mpg cyl  disp  hp drat    wt qsec vs am gear carb
## Porsche 914-2 26.0   4 120.3  91 4.43 2.140 16.7  0  1    5    2
## Lotus Europa  30.4   4  95.1 113 3.77 1.513 16.9  1  1    5    2
```

Remember to use the vector boolean operators `&` and `|`, not the short-circuiting scalar operators `&&` and `||`, which are more useful inside if statements. And don't forget [De Morgan's laws][demorgans], which can be useful to simplify negations:

* `!(X & Y)` is the same as `!X | !Y`
* `!(X | Y)` is the same as `!X & !Y`

For example, `!(X & !(Y | Z))` simplifies to `!X | !!(Y|Z)`, and then to `!X | Y | Z`.

### 4.5.8 Boolean algebra versus sets (logical and integer \mbox{subsetting})
\index{Boolean algebra} 
\index{set algebra}
\indexc{which()}

It's useful to be aware of the natural equivalence between set operations (integer subsetting) and Boolean algebra (logical subsetting). Using set operations is more effective when: 

* You want to find the first (or last) `TRUE`.

* You have very few `TRUE`s and very many `FALSE`s; a set representation 
  may be faster and require less storage.

`which()` allows you to convert a Boolean representation to an integer representation. There's no reverse operation in base R but we can easily create one: 


```r
x <- sample(10) < 4
which(x)
```

```
## [1] 4 5 9
```

```r
unwhich <- function(x, n) {
  out <- rep_len(FALSE, n)
  out[x] <- TRUE
  out
}
unwhich(which(x), 10)
```

```
##  [1] FALSE FALSE FALSE  TRUE  TRUE FALSE FALSE FALSE  TRUE FALSE
```

Let's create two logical vectors and their integer equivalents, and then explore the relationship between Boolean and set operations.


```r
(x1 <- 1:10 %% 2 == 0)
```

```
##  [1] FALSE  TRUE FALSE  TRUE FALSE  TRUE FALSE  TRUE FALSE  TRUE
```

```r
(x2 <- which(x1))
```

```
## [1]  2  4  6  8 10
```

```r
(y1 <- 1:10 %% 5 == 0)
```

```
##  [1] FALSE FALSE FALSE FALSE  TRUE FALSE FALSE FALSE FALSE  TRUE
```

```r
(y2 <- which(y1))
```

```
## [1]  5 10
```

```r
# X & Y <-> intersect(x, y)
x1 & y1
```

```
##  [1] FALSE FALSE FALSE FALSE FALSE FALSE FALSE FALSE FALSE  TRUE
```

```r
intersect(x2, y2)
```

```
## [1] 10
```

```r
# X | Y <-> union(x, y)
x1 | y1
```

```
##  [1] FALSE  TRUE FALSE  TRUE  TRUE  TRUE FALSE  TRUE FALSE  TRUE
```

```r
union(x2, y2)
```

```
## [1]  2  4  6  8 10  5
```

```r
# X & !Y <-> setdiff(x, y)
x1 & !y1
```

```
##  [1] FALSE  TRUE FALSE  TRUE FALSE  TRUE FALSE  TRUE FALSE FALSE
```

```r
setdiff(x2, y2)
```

```
## [1] 2 4 6 8
```

```r
# xor(X, Y) <-> setdiff(union(x, y), intersect(x, y))
xor(x1, y1)
```

```
##  [1] FALSE  TRUE FALSE  TRUE  TRUE  TRUE FALSE  TRUE FALSE FALSE
```

```r
setdiff(union(x2, y2), intersect(x2, y2))
```

```
## [1] 2 4 6 8 5
```

When first learning subsetting, a common mistake is to use `x[which(y)]` instead of `x[y]`. Here the `which()` achieves nothing: it switches from logical to integer subsetting but the result is exactly the same. In more general cases, there are two important differences. 

* When the logical vector contains `NA`, logical subsetting replaces these 
  values with `NA` while `which()` simply drops these values. It's not uncommon 
  to use `which()` for this side-effect, but I don't recommend it: nothing 
  about the name "which" implies the removal of missing values.

* `x[-which(y)]` is __not__ equivalent to `x[!y]`: if `y` is all FALSE, 
  `which(y)` will be `integer(0)` and `-integer(0)` is still `integer(0)`, so
  you'll get no values, instead of all values. 
  
In general, avoid switching from logical to integer subsetting unless you want, for example, the first or last `TRUE` value.

### 4.5.9 Exercises

1.  How would you randomly permute the columns of a data frame? (This is an
    important technique in random forests.) Can you simultaneously permute 
    the rows and columns in one step?


```r
# randomly permute the columns of a data frame
mtcars
```

```
##                      mpg cyl  disp  hp drat    wt  qsec vs am gear carb
## Mazda RX4           21.0   6 160.0 110 3.90 2.620 16.46  0  1    4    4
## Mazda RX4 Wag       21.0   6 160.0 110 3.90 2.875 17.02  0  1    4    4
## Datsun 710          22.8   4 108.0  93 3.85 2.320 18.61  1  1    4    1
## Hornet 4 Drive      21.4   6 258.0 110 3.08 3.215 19.44  1  0    3    1
## Hornet Sportabout   18.7   8 360.0 175 3.15 3.440 17.02  0  0    3    2
## Valiant             18.1   6 225.0 105 2.76 3.460 20.22  1  0    3    1
## Duster 360          14.3   8 360.0 245 3.21 3.570 15.84  0  0    3    4
## Merc 240D           24.4   4 146.7  62 3.69 3.190 20.00  1  0    4    2
## Merc 230            22.8   4 140.8  95 3.92 3.150 22.90  1  0    4    2
## Merc 280            19.2   6 167.6 123 3.92 3.440 18.30  1  0    4    4
## Merc 280C           17.8   6 167.6 123 3.92 3.440 18.90  1  0    4    4
## Merc 450SE          16.4   8 275.8 180 3.07 4.070 17.40  0  0    3    3
## Merc 450SL          17.3   8 275.8 180 3.07 3.730 17.60  0  0    3    3
## Merc 450SLC         15.2   8 275.8 180 3.07 3.780 18.00  0  0    3    3
## Cadillac Fleetwood  10.4   8 472.0 205 2.93 5.250 17.98  0  0    3    4
## Lincoln Continental 10.4   8 460.0 215 3.00 5.424 17.82  0  0    3    4
## Chrysler Imperial   14.7   8 440.0 230 3.23 5.345 17.42  0  0    3    4
## Fiat 128            32.4   4  78.7  66 4.08 2.200 19.47  1  1    4    1
## Honda Civic         30.4   4  75.7  52 4.93 1.615 18.52  1  1    4    2
## Toyota Corolla      33.9   4  71.1  65 4.22 1.835 19.90  1  1    4    1
## Toyota Corona       21.5   4 120.1  97 3.70 2.465 20.01  1  0    3    1
## Dodge Challenger    15.5   8 318.0 150 2.76 3.520 16.87  0  0    3    2
## AMC Javelin         15.2   8 304.0 150 3.15 3.435 17.30  0  0    3    2
## Camaro Z28          13.3   8 350.0 245 3.73 3.840 15.41  0  0    3    4
## Pontiac Firebird    19.2   8 400.0 175 3.08 3.845 17.05  0  0    3    2
## Fiat X1-9           27.3   4  79.0  66 4.08 1.935 18.90  1  1    4    1
## Porsche 914-2       26.0   4 120.3  91 4.43 2.140 16.70  0  1    5    2
## Lotus Europa        30.4   4  95.1 113 3.77 1.513 16.90  1  1    5    2
## Ford Pantera L      15.8   8 351.0 264 4.22 3.170 14.50  0  1    5    4
## Ferrari Dino        19.7   6 145.0 175 3.62 2.770 15.50  0  1    5    6
## Maserati Bora       15.0   8 301.0 335 3.54 3.570 14.60  0  1    5    8
## Volvo 142E          21.4   4 121.0 109 4.11 2.780 18.60  1  1    4    2
```

```r
mtcars[sample(ncol(mtcars))]
```

```
##                      qsec  mpg am drat vs gear  hp carb cyl    wt  disp
## Mazda RX4           16.46 21.0  1 3.90  0    4 110    4   6 2.620 160.0
## Mazda RX4 Wag       17.02 21.0  1 3.90  0    4 110    4   6 2.875 160.0
## Datsun 710          18.61 22.8  1 3.85  1    4  93    1   4 2.320 108.0
## Hornet 4 Drive      19.44 21.4  0 3.08  1    3 110    1   6 3.215 258.0
## Hornet Sportabout   17.02 18.7  0 3.15  0    3 175    2   8 3.440 360.0
## Valiant             20.22 18.1  0 2.76  1    3 105    1   6 3.460 225.0
## Duster 360          15.84 14.3  0 3.21  0    3 245    4   8 3.570 360.0
## Merc 240D           20.00 24.4  0 3.69  1    4  62    2   4 3.190 146.7
## Merc 230            22.90 22.8  0 3.92  1    4  95    2   4 3.150 140.8
## Merc 280            18.30 19.2  0 3.92  1    4 123    4   6 3.440 167.6
## Merc 280C           18.90 17.8  0 3.92  1    4 123    4   6 3.440 167.6
## Merc 450SE          17.40 16.4  0 3.07  0    3 180    3   8 4.070 275.8
## Merc 450SL          17.60 17.3  0 3.07  0    3 180    3   8 3.730 275.8
## Merc 450SLC         18.00 15.2  0 3.07  0    3 180    3   8 3.780 275.8
## Cadillac Fleetwood  17.98 10.4  0 2.93  0    3 205    4   8 5.250 472.0
## Lincoln Continental 17.82 10.4  0 3.00  0    3 215    4   8 5.424 460.0
## Chrysler Imperial   17.42 14.7  0 3.23  0    3 230    4   8 5.345 440.0
## Fiat 128            19.47 32.4  1 4.08  1    4  66    1   4 2.200  78.7
## Honda Civic         18.52 30.4  1 4.93  1    4  52    2   4 1.615  75.7
## Toyota Corolla      19.90 33.9  1 4.22  1    4  65    1   4 1.835  71.1
## Toyota Corona       20.01 21.5  0 3.70  1    3  97    1   4 2.465 120.1
## Dodge Challenger    16.87 15.5  0 2.76  0    3 150    2   8 3.520 318.0
## AMC Javelin         17.30 15.2  0 3.15  0    3 150    2   8 3.435 304.0
## Camaro Z28          15.41 13.3  0 3.73  0    3 245    4   8 3.840 350.0
## Pontiac Firebird    17.05 19.2  0 3.08  0    3 175    2   8 3.845 400.0
## Fiat X1-9           18.90 27.3  1 4.08  1    4  66    1   4 1.935  79.0
## Porsche 914-2       16.70 26.0  1 4.43  0    5  91    2   4 2.140 120.3
## Lotus Europa        16.90 30.4  1 3.77  1    5 113    2   4 1.513  95.1
## Ford Pantera L      14.50 15.8  1 4.22  0    5 264    4   8 3.170 351.0
## Ferrari Dino        15.50 19.7  1 3.62  0    5 175    6   6 2.770 145.0
## Maserati Bora       14.60 15.0  1 3.54  0    5 335    8   8 3.570 301.0
## Volvo 142E          18.60 21.4  1 4.11  1    4 109    2   4 2.780 121.0
```


```r
# simultaneously permute the rows and columns in one step
mtcars[sample(nrow(mtcars)), sample(ncol(mtcars))]
```

```
##                     drat  hp cyl carb gear  qsec  mpg  disp    wt vs am
## Mazda RX4           3.90 110   6    4    4 16.46 21.0 160.0 2.620  0  1
## Ford Pantera L      4.22 264   8    4    5 14.50 15.8 351.0 3.170  0  1
## Duster 360          3.21 245   8    4    3 15.84 14.3 360.0 3.570  0  0
## Merc 280            3.92 123   6    4    4 18.30 19.2 167.6 3.440  1  0
## Lincoln Continental 3.00 215   8    4    3 17.82 10.4 460.0 5.424  0  0
## Toyota Corona       3.70  97   4    1    3 20.01 21.5 120.1 2.465  1  0
## Chrysler Imperial   3.23 230   8    4    3 17.42 14.7 440.0 5.345  0  0
## Camaro Z28          3.73 245   8    4    3 15.41 13.3 350.0 3.840  0  0
## Toyota Corolla      4.22  65   4    1    4 19.90 33.9  71.1 1.835  1  1
## Datsun 710          3.85  93   4    1    4 18.61 22.8 108.0 2.320  1  1
## Merc 240D           3.69  62   4    2    4 20.00 24.4 146.7 3.190  1  0
## Ferrari Dino        3.62 175   6    6    5 15.50 19.7 145.0 2.770  0  1
## Fiat X1-9           4.08  66   4    1    4 18.90 27.3  79.0 1.935  1  1
## Merc 450SLC         3.07 180   8    3    3 18.00 15.2 275.8 3.780  0  0
## Valiant             2.76 105   6    1    3 20.22 18.1 225.0 3.460  1  0
## Merc 230            3.92  95   4    2    4 22.90 22.8 140.8 3.150  1  0
## AMC Javelin         3.15 150   8    2    3 17.30 15.2 304.0 3.435  0  0
## Mazda RX4 Wag       3.90 110   6    4    4 17.02 21.0 160.0 2.875  0  1
## Merc 450SE          3.07 180   8    3    3 17.40 16.4 275.8 4.070  0  0
## Merc 280C           3.92 123   6    4    4 18.90 17.8 167.6 3.440  1  0
## Hornet 4 Drive      3.08 110   6    1    3 19.44 21.4 258.0 3.215  1  0
## Fiat 128            4.08  66   4    1    4 19.47 32.4  78.7 2.200  1  1
## Maserati Bora       3.54 335   8    8    5 14.60 15.0 301.0 3.570  0  1
## Porsche 914-2       4.43  91   4    2    5 16.70 26.0 120.3 2.140  0  1
## Cadillac Fleetwood  2.93 205   8    4    3 17.98 10.4 472.0 5.250  0  0
## Hornet Sportabout   3.15 175   8    2    3 17.02 18.7 360.0 3.440  0  0
## Lotus Europa        3.77 113   4    2    5 16.90 30.4  95.1 1.513  1  1
## Volvo 142E          4.11 109   4    2    4 18.60 21.4 121.0 2.780  1  1
## Pontiac Firebird    3.08 175   8    2    3 17.05 19.2 400.0 3.845  0  0
## Merc 450SL          3.07 180   8    3    3 17.60 17.3 275.8 3.730  0  0
## Honda Civic         4.93  52   4    2    4 18.52 30.4  75.7 1.615  1  1
## Dodge Challenger    2.76 150   8    2    3 16.87 15.5 318.0 3.520  0  0
```


2.  How would you select a random sample of `m` rows from a data frame? 
    What if the sample had to be contiguous (i.e., with an initial row, a 
    final row, and every row in between)?
    

```r
# select a random sample of m rows from a data frame
m <- 5
mtcars[sample(nrow(mtcars), m), ]
```

```
##                     mpg cyl  disp  hp drat    wt  qsec vs am gear carb
## Cadillac Fleetwood 10.4   8 472.0 205 2.93 5.250 17.98  0  0    3    4
## Hornet 4 Drive     21.4   6 258.0 110 3.08 3.215 19.44  1  0    3    1
## Pontiac Firebird   19.2   8 400.0 175 3.08 3.845 17.05  0  0    3    2
## Porsche 914-2      26.0   4 120.3  91 4.43 2.140 16.70  0  1    5    2
## Volvo 142E         21.4   4 121.0 109 4.11 2.780 18.60  1  1    4    2
```


```r
start <- sample(nrow(mtcars) - m + 1, 1)
end <- start + m - 1
mtcars[start:end, , drop = FALSE]
```

```
##                    mpg cyl disp  hp drat    wt  qsec vs am gear carb
## Mazda RX4         21.0   6  160 110 3.90 2.620 16.46  0  1    4    4
## Mazda RX4 Wag     21.0   6  160 110 3.90 2.875 17.02  0  1    4    4
## Datsun 710        22.8   4  108  93 3.85 2.320 18.61  1  1    4    1
## Hornet 4 Drive    21.4   6  258 110 3.08 3.215 19.44  1  0    3    1
## Hornet Sportabout 18.7   8  360 175 3.15 3.440 17.02  0  0    3    2
```

    
3.  How could you put the columns in a data frame in alphabetical order?


```r
mtcars
```

```
##                      mpg cyl  disp  hp drat    wt  qsec vs am gear carb
## Mazda RX4           21.0   6 160.0 110 3.90 2.620 16.46  0  1    4    4
## Mazda RX4 Wag       21.0   6 160.0 110 3.90 2.875 17.02  0  1    4    4
## Datsun 710          22.8   4 108.0  93 3.85 2.320 18.61  1  1    4    1
## Hornet 4 Drive      21.4   6 258.0 110 3.08 3.215 19.44  1  0    3    1
## Hornet Sportabout   18.7   8 360.0 175 3.15 3.440 17.02  0  0    3    2
## Valiant             18.1   6 225.0 105 2.76 3.460 20.22  1  0    3    1
## Duster 360          14.3   8 360.0 245 3.21 3.570 15.84  0  0    3    4
## Merc 240D           24.4   4 146.7  62 3.69 3.190 20.00  1  0    4    2
## Merc 230            22.8   4 140.8  95 3.92 3.150 22.90  1  0    4    2
## Merc 280            19.2   6 167.6 123 3.92 3.440 18.30  1  0    4    4
## Merc 280C           17.8   6 167.6 123 3.92 3.440 18.90  1  0    4    4
## Merc 450SE          16.4   8 275.8 180 3.07 4.070 17.40  0  0    3    3
## Merc 450SL          17.3   8 275.8 180 3.07 3.730 17.60  0  0    3    3
## Merc 450SLC         15.2   8 275.8 180 3.07 3.780 18.00  0  0    3    3
## Cadillac Fleetwood  10.4   8 472.0 205 2.93 5.250 17.98  0  0    3    4
## Lincoln Continental 10.4   8 460.0 215 3.00 5.424 17.82  0  0    3    4
## Chrysler Imperial   14.7   8 440.0 230 3.23 5.345 17.42  0  0    3    4
## Fiat 128            32.4   4  78.7  66 4.08 2.200 19.47  1  1    4    1
## Honda Civic         30.4   4  75.7  52 4.93 1.615 18.52  1  1    4    2
## Toyota Corolla      33.9   4  71.1  65 4.22 1.835 19.90  1  1    4    1
## Toyota Corona       21.5   4 120.1  97 3.70 2.465 20.01  1  0    3    1
## Dodge Challenger    15.5   8 318.0 150 2.76 3.520 16.87  0  0    3    2
## AMC Javelin         15.2   8 304.0 150 3.15 3.435 17.30  0  0    3    2
## Camaro Z28          13.3   8 350.0 245 3.73 3.840 15.41  0  0    3    4
## Pontiac Firebird    19.2   8 400.0 175 3.08 3.845 17.05  0  0    3    2
## Fiat X1-9           27.3   4  79.0  66 4.08 1.935 18.90  1  1    4    1
## Porsche 914-2       26.0   4 120.3  91 4.43 2.140 16.70  0  1    5    2
## Lotus Europa        30.4   4  95.1 113 3.77 1.513 16.90  1  1    5    2
## Ford Pantera L      15.8   8 351.0 264 4.22 3.170 14.50  0  1    5    4
## Ferrari Dino        19.7   6 145.0 175 3.62 2.770 15.50  0  1    5    6
## Maserati Bora       15.0   8 301.0 335 3.54 3.570 14.60  0  1    5    8
## Volvo 142E          21.4   4 121.0 109 4.11 2.780 18.60  1  1    4    2
```

```r
mtcars[order(names(mtcars))]
```

```
##                     am carb cyl  disp drat gear  hp  mpg  qsec vs    wt
## Mazda RX4            1    4   6 160.0 3.90    4 110 21.0 16.46  0 2.620
## Mazda RX4 Wag        1    4   6 160.0 3.90    4 110 21.0 17.02  0 2.875
## Datsun 710           1    1   4 108.0 3.85    4  93 22.8 18.61  1 2.320
## Hornet 4 Drive       0    1   6 258.0 3.08    3 110 21.4 19.44  1 3.215
## Hornet Sportabout    0    2   8 360.0 3.15    3 175 18.7 17.02  0 3.440
## Valiant              0    1   6 225.0 2.76    3 105 18.1 20.22  1 3.460
## Duster 360           0    4   8 360.0 3.21    3 245 14.3 15.84  0 3.570
## Merc 240D            0    2   4 146.7 3.69    4  62 24.4 20.00  1 3.190
## Merc 230             0    2   4 140.8 3.92    4  95 22.8 22.90  1 3.150
## Merc 280             0    4   6 167.6 3.92    4 123 19.2 18.30  1 3.440
## Merc 280C            0    4   6 167.6 3.92    4 123 17.8 18.90  1 3.440
## Merc 450SE           0    3   8 275.8 3.07    3 180 16.4 17.40  0 4.070
## Merc 450SL           0    3   8 275.8 3.07    3 180 17.3 17.60  0 3.730
## Merc 450SLC          0    3   8 275.8 3.07    3 180 15.2 18.00  0 3.780
## Cadillac Fleetwood   0    4   8 472.0 2.93    3 205 10.4 17.98  0 5.250
## Lincoln Continental  0    4   8 460.0 3.00    3 215 10.4 17.82  0 5.424
## Chrysler Imperial    0    4   8 440.0 3.23    3 230 14.7 17.42  0 5.345
## Fiat 128             1    1   4  78.7 4.08    4  66 32.4 19.47  1 2.200
## Honda Civic          1    2   4  75.7 4.93    4  52 30.4 18.52  1 1.615
## Toyota Corolla       1    1   4  71.1 4.22    4  65 33.9 19.90  1 1.835
## Toyota Corona        0    1   4 120.1 3.70    3  97 21.5 20.01  1 2.465
## Dodge Challenger     0    2   8 318.0 2.76    3 150 15.5 16.87  0 3.520
## AMC Javelin          0    2   8 304.0 3.15    3 150 15.2 17.30  0 3.435
## Camaro Z28           0    4   8 350.0 3.73    3 245 13.3 15.41  0 3.840
## Pontiac Firebird     0    2   8 400.0 3.08    3 175 19.2 17.05  0 3.845
## Fiat X1-9            1    1   4  79.0 4.08    4  66 27.3 18.90  1 1.935
## Porsche 914-2        1    2   4 120.3 4.43    5  91 26.0 16.70  0 2.140
## Lotus Europa         1    2   4  95.1 3.77    5 113 30.4 16.90  1 1.513
## Ford Pantera L       1    4   8 351.0 4.22    5 264 15.8 14.50  0 3.170
## Ferrari Dino         1    6   6 145.0 3.62    5 175 19.7 15.50  0 2.770
## Maserati Bora        1    8   8 301.0 3.54    5 335 15.0 14.60  0 3.570
## Volvo 142E           1    2   4 121.0 4.11    4 109 21.4 18.60  1 2.780
```

```r
mtcars[sort(names(mtcars))]
```

```
##                     am carb cyl  disp drat gear  hp  mpg  qsec vs    wt
## Mazda RX4            1    4   6 160.0 3.90    4 110 21.0 16.46  0 2.620
## Mazda RX4 Wag        1    4   6 160.0 3.90    4 110 21.0 17.02  0 2.875
## Datsun 710           1    1   4 108.0 3.85    4  93 22.8 18.61  1 2.320
## Hornet 4 Drive       0    1   6 258.0 3.08    3 110 21.4 19.44  1 3.215
## Hornet Sportabout    0    2   8 360.0 3.15    3 175 18.7 17.02  0 3.440
## Valiant              0    1   6 225.0 2.76    3 105 18.1 20.22  1 3.460
## Duster 360           0    4   8 360.0 3.21    3 245 14.3 15.84  0 3.570
## Merc 240D            0    2   4 146.7 3.69    4  62 24.4 20.00  1 3.190
## Merc 230             0    2   4 140.8 3.92    4  95 22.8 22.90  1 3.150
## Merc 280             0    4   6 167.6 3.92    4 123 19.2 18.30  1 3.440
## Merc 280C            0    4   6 167.6 3.92    4 123 17.8 18.90  1 3.440
## Merc 450SE           0    3   8 275.8 3.07    3 180 16.4 17.40  0 4.070
## Merc 450SL           0    3   8 275.8 3.07    3 180 17.3 17.60  0 3.730
## Merc 450SLC          0    3   8 275.8 3.07    3 180 15.2 18.00  0 3.780
## Cadillac Fleetwood   0    4   8 472.0 2.93    3 205 10.4 17.98  0 5.250
## Lincoln Continental  0    4   8 460.0 3.00    3 215 10.4 17.82  0 5.424
## Chrysler Imperial    0    4   8 440.0 3.23    3 230 14.7 17.42  0 5.345
## Fiat 128             1    1   4  78.7 4.08    4  66 32.4 19.47  1 2.200
## Honda Civic          1    2   4  75.7 4.93    4  52 30.4 18.52  1 1.615
## Toyota Corolla       1    1   4  71.1 4.22    4  65 33.9 19.90  1 1.835
## Toyota Corona        0    1   4 120.1 3.70    3  97 21.5 20.01  1 2.465
## Dodge Challenger     0    2   8 318.0 2.76    3 150 15.5 16.87  0 3.520
## AMC Javelin          0    2   8 304.0 3.15    3 150 15.2 17.30  0 3.435
## Camaro Z28           0    4   8 350.0 3.73    3 245 13.3 15.41  0 3.840
## Pontiac Firebird     0    2   8 400.0 3.08    3 175 19.2 17.05  0 3.845
## Fiat X1-9            1    1   4  79.0 4.08    4  66 27.3 18.90  1 1.935
## Porsche 914-2        1    2   4 120.3 4.43    5  91 26.0 16.70  0 2.140
## Lotus Europa         1    2   4  95.1 3.77    5 113 30.4 16.90  1 1.513
## Ford Pantera L       1    4   8 351.0 4.22    5 264 15.8 14.50  0 3.170
## Ferrari Dino         1    6   6 145.0 3.62    5 175 19.7 15.50  0 2.770
## Maserati Bora        1    8   8 301.0 3.54    5 335 15.0 14.60  0 3.570
## Volvo 142E           1    2   4 121.0 4.11    4 109 21.4 18.60  1 2.780
```


```r
library(tibble)
mtcars_2 <- tibble::rownames_to_column(mtcars, "rownames")
mtcars_2
```

```
##               rownames  mpg cyl  disp  hp drat    wt  qsec vs am gear carb
## 1            Mazda RX4 21.0   6 160.0 110 3.90 2.620 16.46  0  1    4    4
## 2        Mazda RX4 Wag 21.0   6 160.0 110 3.90 2.875 17.02  0  1    4    4
## 3           Datsun 710 22.8   4 108.0  93 3.85 2.320 18.61  1  1    4    1
## 4       Hornet 4 Drive 21.4   6 258.0 110 3.08 3.215 19.44  1  0    3    1
## 5    Hornet Sportabout 18.7   8 360.0 175 3.15 3.440 17.02  0  0    3    2
## 6              Valiant 18.1   6 225.0 105 2.76 3.460 20.22  1  0    3    1
## 7           Duster 360 14.3   8 360.0 245 3.21 3.570 15.84  0  0    3    4
## 8            Merc 240D 24.4   4 146.7  62 3.69 3.190 20.00  1  0    4    2
## 9             Merc 230 22.8   4 140.8  95 3.92 3.150 22.90  1  0    4    2
## 10            Merc 280 19.2   6 167.6 123 3.92 3.440 18.30  1  0    4    4
## 11           Merc 280C 17.8   6 167.6 123 3.92 3.440 18.90  1  0    4    4
## 12          Merc 450SE 16.4   8 275.8 180 3.07 4.070 17.40  0  0    3    3
## 13          Merc 450SL 17.3   8 275.8 180 3.07 3.730 17.60  0  0    3    3
## 14         Merc 450SLC 15.2   8 275.8 180 3.07 3.780 18.00  0  0    3    3
## 15  Cadillac Fleetwood 10.4   8 472.0 205 2.93 5.250 17.98  0  0    3    4
## 16 Lincoln Continental 10.4   8 460.0 215 3.00 5.424 17.82  0  0    3    4
## 17   Chrysler Imperial 14.7   8 440.0 230 3.23 5.345 17.42  0  0    3    4
## 18            Fiat 128 32.4   4  78.7  66 4.08 2.200 19.47  1  1    4    1
## 19         Honda Civic 30.4   4  75.7  52 4.93 1.615 18.52  1  1    4    2
## 20      Toyota Corolla 33.9   4  71.1  65 4.22 1.835 19.90  1  1    4    1
## 21       Toyota Corona 21.5   4 120.1  97 3.70 2.465 20.01  1  0    3    1
## 22    Dodge Challenger 15.5   8 318.0 150 2.76 3.520 16.87  0  0    3    2
## 23         AMC Javelin 15.2   8 304.0 150 3.15 3.435 17.30  0  0    3    2
## 24          Camaro Z28 13.3   8 350.0 245 3.73 3.840 15.41  0  0    3    4
## 25    Pontiac Firebird 19.2   8 400.0 175 3.08 3.845 17.05  0  0    3    2
## 26           Fiat X1-9 27.3   4  79.0  66 4.08 1.935 18.90  1  1    4    1
## 27       Porsche 914-2 26.0   4 120.3  91 4.43 2.140 16.70  0  1    5    2
## 28        Lotus Europa 30.4   4  95.1 113 3.77 1.513 16.90  1  1    5    2
## 29      Ford Pantera L 15.8   8 351.0 264 4.22 3.170 14.50  0  1    5    4
## 30        Ferrari Dino 19.7   6 145.0 175 3.62 2.770 15.50  0  1    5    6
## 31       Maserati Bora 15.0   8 301.0 335 3.54 3.570 14.60  0  1    5    8
## 32          Volvo 142E 21.4   4 121.0 109 4.11 2.780 18.60  1  1    4    2
```

```r
library(dplyr)
```

```
## 
## Attaching package: 'dplyr'
```

```
## The following objects are masked from 'package:stats':
## 
##     filter, lag
```

```
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

```r
dplyr::arrange(mtcars_2,rownames)
```

```
##               rownames  mpg cyl  disp  hp drat    wt  qsec vs am gear carb
## 1          AMC Javelin 15.2   8 304.0 150 3.15 3.435 17.30  0  0    3    2
## 2   Cadillac Fleetwood 10.4   8 472.0 205 2.93 5.250 17.98  0  0    3    4
## 3           Camaro Z28 13.3   8 350.0 245 3.73 3.840 15.41  0  0    3    4
## 4    Chrysler Imperial 14.7   8 440.0 230 3.23 5.345 17.42  0  0    3    4
## 5           Datsun 710 22.8   4 108.0  93 3.85 2.320 18.61  1  1    4    1
## 6     Dodge Challenger 15.5   8 318.0 150 2.76 3.520 16.87  0  0    3    2
## 7           Duster 360 14.3   8 360.0 245 3.21 3.570 15.84  0  0    3    4
## 8         Ferrari Dino 19.7   6 145.0 175 3.62 2.770 15.50  0  1    5    6
## 9             Fiat 128 32.4   4  78.7  66 4.08 2.200 19.47  1  1    4    1
## 10           Fiat X1-9 27.3   4  79.0  66 4.08 1.935 18.90  1  1    4    1
## 11      Ford Pantera L 15.8   8 351.0 264 4.22 3.170 14.50  0  1    5    4
## 12         Honda Civic 30.4   4  75.7  52 4.93 1.615 18.52  1  1    4    2
## 13      Hornet 4 Drive 21.4   6 258.0 110 3.08 3.215 19.44  1  0    3    1
## 14   Hornet Sportabout 18.7   8 360.0 175 3.15 3.440 17.02  0  0    3    2
## 15 Lincoln Continental 10.4   8 460.0 215 3.00 5.424 17.82  0  0    3    4
## 16        Lotus Europa 30.4   4  95.1 113 3.77 1.513 16.90  1  1    5    2
## 17       Maserati Bora 15.0   8 301.0 335 3.54 3.570 14.60  0  1    5    8
## 18           Mazda RX4 21.0   6 160.0 110 3.90 2.620 16.46  0  1    4    4
## 19       Mazda RX4 Wag 21.0   6 160.0 110 3.90 2.875 17.02  0  1    4    4
## 20            Merc 230 22.8   4 140.8  95 3.92 3.150 22.90  1  0    4    2
## 21           Merc 240D 24.4   4 146.7  62 3.69 3.190 20.00  1  0    4    2
## 22            Merc 280 19.2   6 167.6 123 3.92 3.440 18.30  1  0    4    4
## 23           Merc 280C 17.8   6 167.6 123 3.92 3.440 18.90  1  0    4    4
## 24          Merc 450SE 16.4   8 275.8 180 3.07 4.070 17.40  0  0    3    3
## 25          Merc 450SL 17.3   8 275.8 180 3.07 3.730 17.60  0  0    3    3
## 26         Merc 450SLC 15.2   8 275.8 180 3.07 3.780 18.00  0  0    3    3
## 27    Pontiac Firebird 19.2   8 400.0 175 3.08 3.845 17.05  0  0    3    2
## 28       Porsche 914-2 26.0   4 120.3  91 4.43 2.140 16.70  0  1    5    2
## 29      Toyota Corolla 33.9   4  71.1  65 4.22 1.835 19.90  1  1    4    1
## 30       Toyota Corona 21.5   4 120.1  97 3.70 2.465 20.01  1  0    3    1
## 31             Valiant 18.1   6 225.0 105 2.76 3.460 20.22  1  0    3    1
## 32          Volvo 142E 21.4   4 121.0 109 4.11 2.780 18.60  1  1    4    2
```


[demorgans]: http://en.wikipedia.org/wiki/De_Morgan's_laws
