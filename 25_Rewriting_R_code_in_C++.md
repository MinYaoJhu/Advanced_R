---
title: "25_Rewriting_R_code_in_C++"
author: "Min-Yao"
date: "2023-10-06"
output: 
  html_document: 
    keep_md: yes
---

# 25 Rewriting R code in C++ {#rcpp}

## 25.1 Introduction

Sometimes R code just isn't fast enough. You've used profiling to figure out where your bottlenecks are, and you've done everything you can in R, but your code still isn't fast enough. In this chapter you'll learn how to improve performance by rewriting key functions in C++. This magic comes by way of the [Rcpp](http://www.rcpp.org/) package [@Rcpp] (with key contributions by Doug Bates, John Chambers, and JJ Allaire). 

Rcpp makes it very simple to connect C++ to R. While it is _possible_ to write C or Fortran code for use in R, it will be painful by comparison. Rcpp provides a clean, approachable API that lets you write high-performance code, insulated from R's complex C API. \index{Rcpp} \index{C++}

Typical bottlenecks that C++ can address include:

* Loops that can't be easily vectorised because subsequent iterations depend 
  on previous ones.

* Recursive functions, or problems which involve calling functions millions of 
  times. The overhead of calling a function in C++ is much lower than in R.

* Problems that require advanced data structures and algorithms that R doesn't 
  provide. Through the standard template library (STL), C++ has efficient 
  implementations of many important data structures, from ordered maps to 
  double-ended queues.

The aim of this chapter is to discuss only those aspects of C++ and Rcpp that are absolutely necessary to help you eliminate bottlenecks in your code. We won't spend much time on advanced features like object-oriented programming or templates because the focus is on writing small, self-contained functions, not big programs. A working knowledge of C++ is helpful, but not essential. Many good tutorials and references are freely available, including <http://www.learncpp.com/> and <https://en.cppreference.com/w/cpp>. For more advanced topics, the _Effective C++_ series by Scott Meyers is a popular choice. 

### Outline {-}

* Section \@ref(rcpp-intro) teaches you how to write C++ by 
  converting simple R functions to their C++ equivalents. You'll learn how 
  C++ differs from R, and what the key scalar, vector, and matrix classes
  are called.

* Section \@ref(sourceCpp) shows you how to use `sourceCpp()` to load
  a C++ file from disk in the same way you use `source()` to load a file of
  R code. 

* Section \@ref(rcpp-classes) discusses how to modify
  attributes from Rcpp, and mentions some of the other important classes.

* Section \@ref(rcpp-na) teaches you how to work with R's missing values 
  in C++.

* Section \@ref(stl) shows you how to use some of the most important data 
  structures and algorithms from the standard template library, or STL, 
  built-in to C++.

* Section \@ref(rcpp-case-studies) shows two real case studies where 
  Rcpp was used to get considerable performance improvements.

* Section \@ref(rcpp-package) teaches you how to add C++ code
  to a package.

* Section \@ref(rcpp-more) concludes the chapter with pointers to 
  more resources to help you learn Rcpp and C++.

### Prerequisites {-}

We'll use [Rcpp](http://www.rcpp.org/) to call C++ from R:


```r
library(Rcpp)
```

You'll also need a working C++ compiler. To get it:

* On Windows, install [Rtools](http://cran.r-project.org/bin/windows/Rtools/).
* On Mac, install Xcode from the app store.
* On Linux, `sudo apt-get install r-base-dev` or similar.

## 25.2 Getting started with C++ {#rcpp-intro}

`cppFunction()` allows you to write C++ functions in R: \indexc{cppFunction()}


```r
cppFunction('int add(int x, int y, int z) {
  int sum = x + y + z;
  return sum;
}')
# add works like a regular R function
add
```

```
## function (x, y, z) 
## .Call(<pointer: 0x00007ffe23a61690>, x, y, z)
```

```r
add(1, 2, 3)
```

```
## [1] 6
```

When you run this code, Rcpp will compile the C++ code and construct an R function that connects to the compiled C++ function. There's a lot going on underneath the hood but Rcpp takes care of all the details so you don't need to worry about them.

The following sections will teach you the basics by translating simple R functions to their C++ equivalents. We'll start simple with a function that has no inputs and a scalar output, and then make it progressively more complicated:

* Scalar input and scalar output
* Vector input and scalar output
* Vector input and vector output
* Matrix input and vector output

### No inputs, scalar output

Let's start with a very simple function. It has no arguments and always returns the integer 1:


```r
one <- function() 1L
```

The equivalent C++ function is:

```cpp
int one() {
  return 1;
}
```

We can compile and use this from R with `cppFunction()`


```r
cppFunction('int one() {
  return 1;
}')
```

This small function illustrates a number of important differences between R and C++:

* The syntax to create a function looks like the syntax to call a function; 
  you don't use assignment to create functions as you do in R.

* You must declare the type of output the function returns. This function 
  returns an `int` (a scalar integer). The classes for the most common types 
  of R vectors are: `NumericVector`, `IntegerVector`, `CharacterVector`, and 
  `LogicalVector`.

* Scalars and vectors are different. The scalar equivalents of numeric, 
  integer, character, and logical vectors are: `double`, `int`, `String`, and 
  `bool`.

* You must use an explicit `return` statement to return a value from a 
  function.

* Every statement is terminated by a `;`.

### Scalar input, scalar output

The next example function implements a scalar version of the `sign()` function which returns 1 if the input is positive, and -1 if it's negative:


```r
signR <- function(x) {
  if (x > 0) {
    1
  } else if (x == 0) {
    0
  } else {
    -1
  }
}

cppFunction('int signC(int x) {
  if (x > 0) {
    return 1;
  } else if (x == 0) {
    return 0;
  } else {
    return -1;
  }
}')
```

In the C++ version:

* We declare the type of each input in the same way we declare the type of the 
  output. While this makes the code a little more verbose, it also makes clear 
  the type of input the function needs.

* The `if` syntax is identical --- while there are some big differences between 
  R and C++, there are also lots of similarities! C++ also has a `while` 
  statement that works the same way as R's. As in R you can use `break` to 
  exit the loop, but to skip one iteration you need to use `continue` instead 
  of `next`.

### Vector input, scalar output

One big difference between R and C++ is that the cost of loops is much lower in C++. For example, we could implement the `sum` function in R using a loop. If you've been programming in R a while, you'll probably have a visceral reaction to this function!


```r
sumR <- function(x) {
  total <- 0
  for (i in seq_along(x)) {
    total <- total + x[i]
  }
  total
}
```

In C++, loops have very little overhead, so it's fine to use them. In Section \@ref(stl), you'll see alternatives to `for` loops that more clearly express your intent; they're not faster, but they can make your code easier to understand.


```r
cppFunction('double sumC(NumericVector x) {
  int n = x.size();
  double total = 0;
  for(int i = 0; i < n; ++i) {
    total += x[i];
  }
  return total;
}')
```

The C++ version is similar, but:

* To find the length of the vector, we use the `.size()` method, which returns 
  an integer. C++ methods are called with `.` (i.e., a full stop).
  
* The `for` statement has a different syntax: `for(init; check; increment)`. 
  This loop is initialised by creating a new variable called `i` with value 0.
  Before each iteration we check that `i < n`, and terminate the loop if it's 
  not. After each iteration, we increment the value of `i` by one, using the
  special prefix operator `++` which increases the value of `i` by 1.

* In C++, vector indices start at 0, which means that the last element is 
  at position `n - 1`. I'll say this again because it's so important: 
  __IN C++, VECTOR INDICES START AT 0__! This is a very common 
  source of bugs when converting R functions to C++.

* Use `=` for assignment, not `<-`.

* C++ provides operators that modify in-place: `total += x[i]` is equivalent to 
  `total = total + x[i]`. Similar in-place operators are `-=`, `*=`, and `/=`.

This is a good example of where C++ is much more efficient than R. As shown by the following microbenchmark, `sumC()` is competitive with the built-in (and highly optimised) `sum()`, while `sumR()` is several orders of magnitude slower.


```r
x <- runif(1e3)
bench::mark(
  sum(x),
  sumC(x),
  sumR(x)
)[1:6]
```

```
## # A tibble: 3 × 6
##   expression      min   median `itr/sec` mem_alloc `gc/sec`
##   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
## 1 sum(x)          1µs    1.1µs   866123.        0B        0
## 2 sumC(x)       2.2µs    2.6µs   243855.    2.49KB        0
## 3 sumR(x)      23.3µs   28.9µs    27796.   35.66KB        0
```

### Vector input, vector output

<!-- FIXME: come up with better example. Also fix in two other places it occurs -->

Next we'll create a function that computes the Euclidean distance between a value and a vector of values:


```r
pdistR <- function(x, ys) {
  sqrt((x - ys) ^ 2)
}
```

In R, it's not obvious that we want `x` to be a scalar from the function definition, and we'd need to make that clear in the documentation. That's not a problem in the C++ version because we have to be explicit about types:


```r
cppFunction('NumericVector pdistC(double x, NumericVector ys) {
  int n = ys.size();
  NumericVector out(n);

  for(int i = 0; i < n; ++i) {
    out[i] = sqrt(pow(ys[i] - x, 2.0));
  }
  return out;
}')
```

This function introduces only a few new concepts:

* We create a new numeric vector of length `n` with a constructor: 
 `NumericVector out(n)`. Another useful way of making a vector is to copy an 
 existing one: `NumericVector zs = clone(ys)`.

* C++ uses `pow()`, not `^`, for exponentiation.

Note that because the R version is fully vectorised, it's already going to be fast. 


```r
y <- runif(1e6)
bench::mark(
  pdistR(0.5, y),
  pdistC(0.5, y)
)[1:6]
```

```
## # A tibble: 2 × 6
##   expression          min   median `itr/sec` mem_alloc `gc/sec`
##   <bch:expr>     <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
## 1 pdistR(0.5, y)   5.68ms   6.29ms      146.    7.63MB     74.8
## 2 pdistC(0.5, y)   3.09ms   3.33ms      262.    7.63MB    129.
```

On my computer, it takes around 5 ms with a 1 million element `y` vector. The C++ function is about 2.5 times faster, ~2 ms, but assuming it took you 10 minutes to write the C++ function, you'd need to run it ~200,000 times to make rewriting worthwhile. The reason why the C++ function is faster is subtle, and relates to memory management. The R version needs to create an intermediate vector the same length as y (`x - ys`), and allocating memory is an expensive operation. The C++ function avoids this overhead because it uses an intermediate scalar.



### Using sourceCpp {#sourceCpp}

So far, we've used inline C++ with `cppFunction()`. This makes presentation simpler, but for real problems, it's usually easier to use stand-alone C++ files and then source them into R using `sourceCpp()`. This lets you take advantage of text editor support for C++ files (e.g., syntax highlighting) as well as making it easier to identify the line numbers in compilation errors. \indexc{sourceCpp()}

Your stand-alone C++ file should have extension `.cpp`, and needs to start with:

```cpp
#include <Rcpp.h>
using namespace Rcpp;
```

And for each function that you want available within R, you need to prefix it with:

```cpp
// [[Rcpp::export]]
```

:::sidebar
If you're familiar with roxygen2, you might wonder how this relates to `@export`. `Rcpp::export` controls whether a function is exported from C++ to R; `@export` controls whether a function is exported from a package and made available to the user.
:::

You can embed R code in special C++ comment blocks. This is really convenient if you want to run some test code:

```cpp
/*** R
# This is R code
*/
```

The R code is run with `source(echo = TRUE)` so you don't need to explicitly print output.

To compile the C++ code, use `sourceCpp("path/to/file.cpp")`. This will create the matching R functions and add them to your current session. Note that these functions can not be saved in a `.Rdata` file and reloaded in a later session; they must be recreated each time you restart R. 

For example, running `sourceCpp()` on the following file implements mean in C++ and then compares it to the built-in `mean()`:


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
double meanC(NumericVector x) {
  int n = x.size();
  double total = 0;

  for(int i = 0; i < n; ++i) {
    total += x[i];
  }
  return total / n;
}

/*** R
x <- runif(1e5)
bench::mark(
  mean(x),
  meanC(x)
)
*/
```

NB: If you run this code, you'll notice that `meanC()` is much faster than the built-in `mean()`. This is because it trades numerical accuracy for speed.

For the remainder of this chapter C++ code will be presented stand-alone rather than wrapped in a call to `cppFunction`. If you want to try compiling and/or modifying the examples you should paste them into a C++ source file that includes the elements described above. This is easy to do in RMarkdown: all you need to do is specify `engine = "Rcpp"`. 

### Exercises {#exercise-started}

1.  With the basics of C++ in hand, it's now a great time to practice by reading 
    and writing some simple C++ functions. For each of the following functions, 
    read the code and figure out what the corresponding base R function is. You
    might not understand every part of the code yet, but you should be able to 
    figure out the basics of what the function does.




```cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
    double f1(NumericVector x) {
      int n = x.size();
      double y = 0;
    
      for(int i = 0; i < n; ++i) {
        y += x[i] / n;
      }
      return y;
    }
    
    NumericVector f2(NumericVector x) {
      int n = x.size();
      NumericVector out(n);
    
      out[0] = x[0];
      for(int i = 1; i < n; ++i) {
        out[i] = out[i - 1] + x[i];
      }
      return out;
    }
    
    bool f3(LogicalVector x) {
      int n = x.size();
    
      for(int i = 0; i < n; ++i) {
        if (x[i]) return true;
      }
      return false;
    }
    
    int f4(Function pred, List x) {
      int n = x.size();
    
      for(int i = 0; i < n; ++i) {
        LogicalVector res = pred(x[i]);
        if (res[0]) return i + 1;
      }
      return 0;
    }
    
    NumericVector f5(NumericVector x, NumericVector y) {
      int n = std::max(x.size(), y.size());
      NumericVector x1 = rep_len(x, n);
      NumericVector y1 = rep_len(y, n);
    
      NumericVector out(n);
    
      for (int i = 0; i < n; ++i) {
        out[i] = std::min(x1[i], y1[i]);
      }
    
      return out;
    }
```
    
> f1: `mean()`

> f2: `cumsum()` : cumulative sums

Returns a vector whose elements are the cumulative sums

> f3: `any()`

any: Are Some Values True?
Given a set of logical vectors, is at least one of the values true?

> f4: `Position()`

position: Find or assign the implied position for graphing the levels of a factor. A new class "positioned", which inherits from "ordered" and "factor", is defined.
Description

The default values for plotting a factor x are the integers 1:length(levels(x)). These functions provide a way of specifying alternate plotting locations for the levels.


> f5: `pmin()`

pmin: Maxima and Minima for mcnodes

Description
Returns the parallel maxima and minima of the input values.


```r
x1 <- c(2, 8, 3, 4, 1, 5)               # First example vector
x2 <- c(0, 7, 5, 5, 6, 1)               # Second example vector
pmin(x1, x2)
```

```
## [1] 0 7 3 4 1 1
```


2.  To practice your function writing skills, convert the following functions 
    into C++. For now, assume the inputs have no missing values.
  
    1. `all()`.
    
    2. `cumprod()`, `cummin()`, `cummax()`.
    
    3. `diff()`. Start by assuming lag 1, and then generalise for lag `n`.
    
    4. `range()`.
    
    5. `var()`. Read about the approaches you can take on 
       [Wikipedia](http://en.wikipedia.org/wiki/Algorithms_for_calculating_variance).
       Whenever implementing a numerical algorithm, it's always good to check 
       what is already known about the problem.
       
Let's port these functions to C++.

1. `all()`

    
    ```cpp
    bool allC(LogicalVector x) {
      int n = x.size();
      
      for (int i = 0; i < n; ++i) {
        if (!x[i]) return false;
      }
      return true;
    }
    ```
    

```cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
bool allC(LogicalVector x) {
      int n = x.size();
      
      for (int i = 0; i < n; ++i) {
        if (!x[i]) return false;
      }
      return true;
    }

```




```r
a1 <- c(T,T,F,T,T,F)
a2 <- c(T,T,T,T,T,T)

all(a1)
```

```
## [1] FALSE
```

```r
allC(a1)
```

```
## [1] FALSE
```

```r
all(a2)
```

```
## [1] TRUE
```

```r
allC(a2)
```

```
## [1] TRUE
```

2. `cumprod()`, `cummin()`, `cummax()`.

    
    ```cpp
    NumericVector cumprodC(NumericVector x) {
      int n = x.size();
      NumericVector out(n);
      
      out[0] = x[0];
      for (int i = 1; i < n; ++i) {
        out[i]  = out[i - 1] * x[i];
      }
      return out;
    }
    
    NumericVector cumminC(NumericVector x) {
      int n = x.size();
      NumericVector out(n);
      
      out[0] = x[0];
      for (int i = 1; i < n; ++i) {
        out[i]  = std::min(out[i - 1], x[i]);
      }
      return out;
    }
    
    NumericVector cummaxC(NumericVector x) {
      int n = x.size();
      NumericVector out(n);
      
      out[0] = x[0];
      for (int i = 1; i < n; ++i) {
        out[i]  = std::max(out[i - 1], x[i]);
      }
      return out;
    }
    ```

3. `diff()` (Start by assuming lag 1, and then generalise for lag `n`.)

    
    ```cpp
    NumericVector diffC(NumericVector x) {
      int n = x.size();
      NumericVector out(n - 1);
      
      for (int i = 1; i < n; i++) {
        out[i - 1] = x[i] - x[i - 1];
      }
      return out ;
    }
    
    NumericVector difflagC(NumericVector x, int lag = 1) {
      int n = x.size();
    
      if (lag >= n) stop("`lag` must be less than `length(x)`.");
      
      NumericVector out(n - lag);
      
      for (int i = lag; i < n; i++) {
        out[i - lag] = x[i] - x[i - lag];
      }
      return out;
    }
    ```

4. `range()`

    
    ```cpp
    NumericVector rangeC(NumericVector x) {
      double omin = x[0], omax = x[0];
      int n = x.size();
    
      if (n == 0) stop("`length(x)` must be greater than 0.");
      
      for (int i = 1; i < n; i++) {
        omin = std::min(x[i], omin);
        omax = std::max(x[i], omax);
      }
      
      NumericVector out(2);
      out[0] = omin;
      out[1] = omax;
      return out;
    }
    ```

5. `var()`

    
    ```cpp
    double varC(NumericVector x) {
      int n = x.size();
      
      if (n < 2) {
        return NA_REAL;
      }
      
      double mx = 0;
      for (int i = 0; i < n; ++i) {
        mx += x[i] / n;
      }
      
      double out = 0;
      for (int i = 0; i < n; ++i) {
        out += pow(x[i] - mx, 2);
      }
      
      return out / (n - 1);
    }
    ```
