---
title: "Ch25_Rewriting_R_code_in_C++_2"
author: "Min-Yao"
date: "2023-10-13"
output: 
  html_document: 
    keep_md: yes
---


```r
library(Rcpp)
```

## 25.3 Other classes {#rcpp-classes}

You've already seen the basic vector classes (`IntegerVector`, `NumericVector`, `LogicalVector`, `CharacterVector`) and their scalar (`int`, `double`, `bool`, `String`) equivalents. Rcpp also provides wrappers for all other base data types. The most important are for lists and data frames, functions, and attributes, as described below. Rcpp also provides classes for more types like `Environment`, `DottedPair`, `Language`, `Symbol`, etc, but these are beyond the scope of this chapter.

### 25.3.1 Lists and data frames

Rcpp also provides `List` and `DataFrame` classes, but they are more useful for output than input. This is because lists and data frames can contain arbitrary classes but C++ needs to know their classes in advance. If the list has known structure (e.g., it's an S3 object), you can extract the components and manually convert them to their C++ equivalents with `as()`. For example, the object created by `lm()`, the function that fits a linear model, is a list whose components are always of the same type. The following code illustrates how you might extract the mean percentage error (`mpe()`) of a linear model. This isn't a good example of when to use C++, because it's so easily implemented in R, but it shows how to work with an important S3 class. Note the use of `.inherits()` and the `stop()` to check that the object really is a linear model. \index{lists!in C++} \index{data frames!in C++}

<!-- FIXME: needs better motivation -->


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
double mpe(List mod) {
  if (!mod.inherits("lm")) stop("Input must be a linear model");

  NumericVector resid = as<NumericVector>(mod["residuals"]);
  NumericVector fitted = as<NumericVector>(mod["fitted.values"]);

  int n = resid.size();
  double err = 0;
  for(int i = 0; i < n; ++i) {
    err += resid[i] / (fitted[i] + resid[i]);
  }
  return err / n;
}
```


```r
mod <- lm(mpg ~ wt, data = mtcars)
mpe(mod)
```

```
## [1] -0.01541615
```

### 25.3.2 Functions {#functions-rcpp}
\index{functions!in C++}

You can put R functions in an object of type `Function`. This makes calling an R function from C++ straightforward. The only challenge is that we don't know what type of output the function will return, so we use the catchall type `RObject`. 


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
RObject callWithOne(Function f) {
  return f(1);
}
```


```r
callWithOne(function(x) x + 1)
```

```
## [1] 2
```

```r
callWithOne(paste)
```

```
## [1] "1"
```

Calling R functions with positional arguments is obvious:

```cpp
f("y", 1);
```

But you need a special syntax for named arguments:

```cpp
f(_["x"] = "y", _["value"] = 1);
```

### 25.3.3 Attributes
\index{attributes!in C++} 

All R objects have attributes, which can be queried and modified with `.attr()`. Rcpp also provides `.names()` as an alias for the name attribute. The following code snippet illustrates these methods. Note the use of `::create()`, a _class_ method. This allows you to create an R vector from C++ scalar values: 


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
NumericVector attribs() {
  NumericVector out = NumericVector::create(1, 2, 3);

  out.names() = CharacterVector::create("a", "b", "c");
  out.attr("my-attr") = "my-value";
  out.attr("class") = "my-class";

  return out;
}
```

For S4 objects, `.slot()` plays a similar role to `.attr()`.

## 25.4 Missing values {#rcpp-na}
\indexc{NA}

If you're working with missing values, you need to know two things:

* How R's missing values behave in C++'s scalars (e.g., `double`).
* How to get and set missing values in vectors (e.g., `NumericVector`).

### 25.4.1 Scalars

The following code explores what happens when you take one of R's missing values, coerce it into a scalar, and then coerce back to an R vector. Note that this kind of experimentation is a useful way to figure out what any operation does.


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
List scalar_missings() {
  int int_s = NA_INTEGER;
  String chr_s = NA_STRING;
  bool lgl_s = NA_LOGICAL;
  double num_s = NA_REAL;

  return List::create(int_s, chr_s, lgl_s, num_s);
}
```


```r
str(scalar_missings())
```

```
## List of 4
##  $ : int NA
##  $ : chr NA
##  $ : logi TRUE
##  $ : num NA
```

With the exception of `bool`, things look pretty good here: all of the missing values have been preserved. However, as we'll see in the following sections, things are not quite as straightforward as they seem.

#### 25.4.1.1 Integers

With integers, missing values are stored as the smallest integer. If you don't do anything to them, they'll be preserved. But, since C++ doesn't know that the smallest integer has this special behaviour, if you do anything to it you're likely to get an incorrect value: for example, `evalCpp('NA_INTEGER + 1')` gives -2147483647. 

So if you want to work with missing values in integers, either use a length 1 `IntegerVector` or be very careful with your code.

#### 25.4.1.2 Doubles

With doubles, you may be able to get away with ignoring missing values and working with NaNs (not a number). This is because R's NA is a special type of IEEE 754 floating point number NaN. So any logical expression that involves a NaN (or in C++, NAN) always evaluates as FALSE:




```r
evalCpp("NAN == 1")
```

```
## [1] FALSE
```

```r
evalCpp("NAN < 1")
```

```
## [1] FALSE
```

```r
evalCpp("NAN > 1")
```

```
## [1] FALSE
```

```r
evalCpp("NAN == NAN")
```

```
## [1] FALSE
```

(Here I'm using `evalCpp()` which allows you to see the result of running a single C++ expression, making it excellent for this sort of interactive experimentation.)

But be careful when combining them with Boolean values:


```r
evalCpp("NAN && TRUE")
```

```
## [1] TRUE
```

```r
evalCpp("NAN || FALSE")
```

```
## [1] TRUE
```

However, in numeric contexts NaNs will propagate NAs:


```r
evalCpp("NAN + 1")
```

```
## [1] NaN
```

```r
evalCpp("NAN - 1")
```

```
## [1] NaN
```

```r
evalCpp("NAN / 1")
```

```
## [1] NaN
```

```r
evalCpp("NAN * 1")
```

```
## [1] NaN
```

### 25.4.2 Strings

`String` is a scalar string class introduced by Rcpp, so it knows how to deal with missing values.

### 25.4.3 Boolean

While C++'s `bool` has two possible values (`true` or `false`), a logical vector in R has three (`TRUE`, `FALSE`, and `NA`). If you coerce a length 1 logical vector, make sure it doesn't contain any missing values; otherwise they will be converted to TRUE. An easy fix is to use `int` instead, as this can represent `TRUE`, `FALSE`, and `NA`.

### 25.4.4 Vectors {#vectors-rcpp}

With vectors, you need to use a missing value specific to the type of vector, `NA_REAL`, `NA_INTEGER`, `NA_LOGICAL`, `NA_STRING`:


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
List missing_sampler() {
  return List::create(
    NumericVector::create(NA_REAL),
    IntegerVector::create(NA_INTEGER),
    LogicalVector::create(NA_LOGICAL),
    CharacterVector::create(NA_STRING)
  );
}
```


```r
str(missing_sampler())
```

```
## List of 4
##  $ : num NA
##  $ : int NA
##  $ : logi NA
##  $ : chr NA
```

### 25.4.5 Exercises

1. Rewrite any of the functions from the first exercise of 
   Section \@ref(exercise-started) to deal with missing 
   values. If `na.rm` is true, ignore the missing values. If `na.rm` is false, 
   return a missing value if the input contains any missing values. Some 
   good functions to practice with are `min()`, `max()`, `range()`, `mean()`, 
   and `var()`.
   
> min
   

```cpp
#include <Rcpp.h>
using namespace Rcpp;

// Define a function minC that calculates the minimum value in a NumericVector
// while considering missing values based on the 'na_rm' argument.
// If 'na_rm' is TRUE, it treats NAs as missing values and returns Inf if all values are NA.
// If 'na_rm' is FALSE, it returns NA if there is any NA in the input vector.

// [[Rcpp::export]]
NumericVector minC(NumericVector x, bool na_rm = false) {
  int n = x.size();
  NumericVector out = NumericVector::create(R_PosInf);
  
  if (na_rm) {
    // If 'na_rm' is TRUE, ignore NAs and find the minimum value.
    for (int i = 0; i < n; ++i) {
      if (x[i] == NA_REAL) {
        continue; // Ignore NA values
      }
      if (x[i] < out[0]) {
        out[0] = x[i]; // Update minimum if a smaller value is found
      }
    }
  } else {
    // If 'na_rm' is FALSE, return NA if there is any NA in the input.
    for (int i = 0; i < n; ++i) {
      if (NumericVector::is_na(x[i])) {
        out[0] = NA_REAL; // Set the result to NA and exit if any NA is encountered
        return out;
      }
      if (x[i] < out[0]) {
        out[0] = x[i]; // Update minimum if a smaller value is found
      }
    }
  }
  
  return out; // Return the minimum value, which can be either a number or NA
}

```


```r
result1 <- minC(c(3, 1, 2, 4, NA))
result2 <- minC(c(3, 1, 2, 4, NA), na_rm = TRUE)
result3 <- minC(c(NA, NA, NA), na_rm = TRUE)

print(result1)
```

```
## [1] NA
```

```r
print(result2)
```

```
## [1] 1
```

```r
print(result3)
```

```
## [1] Inf
```

> max


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// Define a function maxC that calculates the maximum value in a NumericVector
// while considering missing values based on the 'na_rm' argument.
// If 'na_rm' is TRUE, it treats NAs as missing values and returns -Inf if all values are NA.
// If 'na_rm' is FALSE, it returns NA if there is any NA in the input vector.

// [[Rcpp::export]]
NumericVector maxC(NumericVector x, bool na_rm = false) {
  int n = x.size();
  NumericVector out = NumericVector::create(-R_PosInf);

  if (na_rm) {
    // If 'na_rm' is TRUE, ignore NAs and find the maximum value.
    for (int i = 0; i < n; ++i) {
      if (x[i] == NA_REAL) {
        continue; // Ignore NA values
      }
      if (x[i] > out[0]) {
        out[0] = x[i]; // Update maximum if a larger value is found
      }
    }
  } else {
    // If 'na_rm' is FALSE, return NA if there is any NA in the input.
    for (int i = 0; i < n; ++i) {
      if (NumericVector::is_na(x[i])) {
        out[0] = NA_REAL; // Set the result to NA and exit if any NA is encountered
        return out;
      }
      if (x[i] > out[0]) {
        out[0] = x[i]; // Update maximum if a larger value is found
      }
    }
  }

  return out; // Return the maximum value, which can be either a number or NA
}

```


```r
result1 <- maxC(c(3, 1, 2, 4, NA))
result2 <- maxC(c(3, 1, 2, 4, NA), na_rm = TRUE)
result3 <- maxC(c(NA, NA, NA), na_rm = TRUE)

print(result1)
```

```
## [1] NA
```

```r
print(result2)
```

```
## [1] 4
```

```r
print(result3)
```

```
## [1] -Inf
```

> range 


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// Define a function rangeC that calculates the range of a NumericVector
// while considering missing values based on the 'na_rm' argument.
// If 'na_rm' is TRUE, it treats NAs as missing values and returns a range
// excluding missing values.
// If 'na_rm' is FALSE, it returns a range including NAs if they are present.

// [[Rcpp::export]]
NumericVector rangeC(NumericVector x, bool na_rm = false) {
  int n = x.size();
  double min_val = R_PosInf;
  double max_val = -R_PosInf;

  for (int i = 0; i < n; ++i) {
    if (NumericVector::is_na(x[i])) {
      if (!na_rm) {
        min_val = NA_REAL;
        max_val = NA_REAL;
        break;
      }
    } else {
      if (x[i] < min_val) min_val = x[i];
      if (x[i] > max_val) max_val = x[i];
    }
  }

  return NumericVector::create(min_val, max_val);
}

```


```r
result1 <- rangeC(c(3, 1, 2, 4, NA))
result2 <- rangeC(c(3, 1, 2, 4, NA), na_rm = TRUE)
result3 <- rangeC(c(NA, NA, NA), na_rm = TRUE)

print(result1)
```

```
## [1] NA NA
```

```r
print(result2)
```

```
## [1] 1 4
```

```r
print(result3)
```

```
## [1]  Inf -Inf
```

> mean


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// Define a function meanC that calculates the mean of a NumericVector
// while considering missing values based on the 'na_rm' argument.
// If 'na_rm' is TRUE, it treats NAs as missing values and calculates the mean
// excluding missing values.
// If 'na_rm' is FALSE, it returns NA if there is any NA in the input vector.

// [[Rcpp::export]]
double meanC(NumericVector x, bool na_rm = false) {
  // Get the number of elements in the input NumericVector 'x'
  int n = x.size();
  // Initialize a variable to accumulate the sum of non-missing values
  double sum = 0.0;
  // Initialize a variable to count the number of non-missing values
  int count = 0;

  // Iterate over the elements of 'x'
  for (int i = 0; i < n; ++i) {
    // Check if the current element of 'x' is an NA value
    if (NumericVector::is_na(x[i])) {
      // If 'na_rm' is FALSE and an NA value is encountered, return NA_REAL
      if (!na_rm) {
        return NA_REAL; // Mean cannot be computed when NAs are not removed
      }
    } else {
      // If the current element is not an NA, add it to the sum
      sum += x[i];
      // Increment the count of non-missing values
      count++;
    }
  }

  // If there were no non-missing values in the input, return NA
  if (count == 0) {
    return NA_REAL; // Mean cannot be computed when there are no non-missing values
  }

  // Calculate and return the mean by dividing the sum by the count
  return sum / count;
}

```


```r
result1 <- meanC(c(3, 1, 2, 4, NA))
result2 <- meanC(c(3, 1, 2, 4, NA), na_rm = TRUE)
result3 <- meanC(c(NA, NA, NA), na_rm = TRUE)

print(result1)
```

```
## [1] NA
```

```r
print(result2)
```

```
## [1] 2.5
```

```r
print(result3)
```

```
## [1] NA
```

> var


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// Define a function varC that calculates the variance of a NumericVector
// while considering missing values based on the 'na_rm' argument.
// If 'na_rm' is TRUE, it treats NAs as missing values and calculates the variance
// excluding missing values.
// If 'na_rm' is FALSE, it returns NA if there is any NA in the input vector.

// [[Rcpp::export]]
double varC(NumericVector x, bool na_rm = false) {
  // Get the number of elements in the input NumericVector 'x'
  int n = x.size();
  // Initialize a variable to store the mean of non-missing values
  double mean = 0.0;
  // Initialize a variable to accumulate the sum of squared differences from the mean
  double sum_squared_diff = 0.0;
  // Initialize a variable to count the number of non-missing values
  int count = 0;

  // Iterate over the elements of 'x'
  for (int i = 0; i < n; ++i) {
    // Check if the current element of 'x' is an NA value
    if (NumericVector::is_na(x[i])) {
      // If 'na_rm' is FALSE and an NA value is encountered, return NA_REAL
      if (!na_rm) {
        return NA_REAL; // Variance cannot be computed when NAs are not removed
      }
    } else {
      // If the current element is not an NA, add it to the mean
      mean += x[i];
      // Increment the count of non-missing values
      count++;
    }
  }

  // If there were no non-missing values in the input, return NA
  if (count == 0) {
    return NA_REAL; // Variance cannot be computed when there are no non-missing values
  }

  // Calculate the mean of non-missing values
  mean /= count;

  // Iterate over the elements of 'x' again to compute the sum of squared differences
  for (int i = 0; i < n; ++i) {
    if (!NumericVector::is_na(x[i])) {
      double diff = x[i] - mean;
      sum_squared_diff += diff * diff;
    }
  }

  // Calculate and return the variance, dividing the sum of squared differences by (count - 1)
  return sum_squared_diff / (count - 1);
}

```


```r
result1 <- varC(c(3, 1, 2, 4, NA))
result2 <- varC(c(3, 1, 2, 4, NA), na_rm = TRUE)
result3 <- varC(c(NA, NA, NA), na_rm = TRUE)

print(result1)
```

```
## [1] NA
```

```r
print(result2)
```

```
## [1] 1.666667
```

```r
print(result3)
```

```
## [1] NA
```


> any


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// Define a function anyC that checks if there is any 'true' value in a LogicalVector x,
// while considering missing values based on the 'na_rm' argument.
// If 'na_rm' is FALSE, it treats NAs as true and returns NA if there is any NA.
// If 'na_rm' is TRUE, it treats NAs as missing values and returns false if all values are missing.

// [[Rcpp::export]]
LogicalVector anyC(LogicalVector x, bool na_rm = false) {
  int n = x.size();
  LogicalVector out = LogicalVector::create(false);

  if (na_rm == false) {
    // If 'na_rm' is FALSE, return NA if any NA is encountered, otherwise check for 'true' values.
    for (int i = 0; i < n; ++i) {
      if (LogicalVector::is_na(x[i])) {
        out[0] = NA_LOGICAL; // Set the result to NA and exit if any NA is encountered
        return out;
      } else if (x[i]) {
        out[0] = true; // If 'true' is found, set the result to true
      }
    }
  }

  if (na_rm) {
    // If 'na_rm' is TRUE, return false if all values are missing, otherwise check for 'true' values.
    for (int i = 0; i < n; ++i) {
      if (LogicalVector::is_na(x[i])) {
        continue; // Ignore NA values
      } else if (x[i]) {
        out[0] = true; // If 'true' is found, set the result to true
        return out;
      }
    }
  }
  
  return out; // Return the result, which can be true, false, or NA
}

```


```r
result1 <- anyC(c(TRUE, FALSE, NA, TRUE))
result2 <- anyC(c(TRUE, FALSE, NA, TRUE), na_rm = TRUE)

print(result1)
```

```
## [1] NA
```

```r
print(result2)
```

```
## [1] TRUE
```

2. Rewrite `cumsum()` and `diff()` so they can handle missing values. Note that 
   these functions have slightly more complicated behaviour.
   
> cumsum
   

```cpp
#include <Rcpp.h>
using namespace Rcpp;

// Define a function cumsumC that calculates the cumulative sum of a NumericVector x,
// while considering missing values based on the 'na_rm' argument.
// If 'na_rm' is FALSE, all values following the first NA are set to NA.
// If 'na_rm' is TRUE, NAs are treated like zeros.

// [[Rcpp::export]]
NumericVector cumsumC(NumericVector x, bool na_rm = false) {
  int n = x.size();
  NumericVector out(n); // Initialize the result vector
  LogicalVector is_missing = is_na(x); // Create a logical vector to identify missing values

  if (!na_rm) {
    // If 'na_rm' is FALSE, set values following the first NA to NA, or treat NAs as zeros.
    for (int i = 0; i < n; ++i) {
      if (is_missing[i] && i > 0) {
        out[i] = NA_REAL; // Set to NA if missing and not the first element
      } else {
        out[i] = (is_missing[i] ? 0 : x[i]) + (i > 0 ? out[i - 1] : 0); // Calculate cumulative sum
      }
    }
  }

  if (na_rm) {
    // If 'na_rm' is TRUE, treat NAs as zeros and calculate the cumulative sum accordingly.
    for (int i = 0; i < n; ++i) {
      out[i] = (is_missing[i] ? 0 : x[i]) + (i > 0 ? out[i - 1] : 0); // Calculate cumulative sum
    }
  }
  
  return out; // Return the cumulative sum vector
}

```


```r
result1 <- cumsumC(c(1, 2, NA, 4, NA, 3))
result2 <- cumsumC(c(1, 2, NA, 4, NA, 3), na_rm = TRUE)

print(result1)
```

```
## [1]  1  3 NA NA NA NA
```

```r
print(result2)
```

```
## [1]  1  3  3  7  7 10
```

> diff


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// Define a function diffC that calculates the differences between elements of a NumericVector x.
// The 'lag' argument specifies the lag between elements to compute differences.
// The 'na_rm' argument controls how missing values (NAs) are handled.

// [[Rcpp::export]]
NumericVector diffC(NumericVector x, int lag = 1, bool na_rm = false) {
  int n = x.size();
  
  if (lag >= n) stop("`lag` must be less than `length(x)`.");
  
  NumericVector out(n - lag);
  
  for (int i = lag; i < n; i++) {
    if (NumericVector::is_na(x[i]) || NumericVector::is_na(x[i - lag])) {
      if (!na_rm) {
        // If 'na_rm' is FALSE, return an NA vector of the same length as 'x' - 'lag'.
        return NumericVector(n - lag, NA_REAL);
      }
      out[i - lag] = NA_REAL;
      continue;
    }
    // Calculate the difference between elements at 'i' and 'i - lag'.
    out[i - lag] = x[i] - x[i - lag];
  }
  
  return out;
}

```


```r
result1 <- diffC(c(1, NA, 3, 5, NA, 10, 12), lag = 2)
result2 <- diffC(c(1, NA, 3, 5, NA, 10, 12), lag = 2, na_rm = TRUE)

print(result1)
```

```
## [1] NA NA NA NA NA
```

```r
print(result2)
```

```
## [1]  2 NA NA  5 NA
```

