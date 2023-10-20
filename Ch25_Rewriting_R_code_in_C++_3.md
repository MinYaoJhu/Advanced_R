---
title: "Ch25_Rewriting_R_code_in_C++_3"
author: "Min-Yao"
date: "2023-10-20"
output: 
  html_document: 
    keep_md: yes
---


```r
library(Rcpp)
```

## 25.5 Standard Template Library {#stl}

The real strength of C++ is revealed when you need to implement more complex algorithms. The standard template library (STL) provides a set of extremely useful data structures and algorithms. This section will explain some of the most important algorithms and data structures and point you in the right direction to learn more. I can't teach you everything you need to know about the STL, but hopefully the examples will show you the power of the STL, and persuade you that it's useful to learn more. \index{standard template library}

If you need an algorithm or data structure that isn't implemented in STL, a good place to look is [boost](http://www.boost.org/doc/). Installing boost on your computer is beyond the scope of this chapter, but once you have it installed, you can use boost data structures and algorithms by including the appropriate header file with (e.g.) `#include <boost/array.hpp>`.

### 25.5.1 Using iterators

Iterators are used extensively in the STL: many functions either accept or return iterators. They are the next step up from basic loops, abstracting away the details of the underlying data structure. Iterators have three main operators: \index{iterators}

1. Advance with `++`.
1. Get the value they refer to, or __dereference__, with `*`. 
1. Compare with `==`. 

For example we could re-write our sum function using iterators:


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
double sum3(NumericVector x) {
  double total = 0;
  
  NumericVector::iterator it;
  for(it = x.begin(); it != x.end(); ++it) {
    total += *it;
  }
  return total;
}
```

The main changes are in the for loop:

* We start at `x.begin()` and loop until we get to `x.end()`. A small 
  optimization is to store the value of the end iterator so we don't need to 
  look it up each time. This only saves about 2 ns per iteration, so it's only 
  important when the calculations in the loop are very simple.

* Instead of indexing into x, we use the dereference operator to get its 
  current value: `*it`.

* Notice the type of the iterator: `NumericVector::iterator`. Each vector 
  type has its own iterator type: `LogicalVector::iterator`, 
  `CharacterVector::iterator`, etc.

This code can be simplified still further through the use of a C++11 feature: range-based for loops. C++11 is widely available, and can easily be activated for use with Rcpp by adding `[[Rcpp::plugins(cpp11)]]`.


```cpp
// [[Rcpp::plugins(cpp11)]]
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
double sum4(NumericVector xs) {
  double total = 0;
  
  for(const auto &x : xs) {
    total += x;
  }
  return total;
}
```

Iterators also allow us to use the C++ equivalents of the apply family of functions. For example, we could again rewrite `sum()` to use the `accumulate()` function, which takes a starting and an ending iterator, and adds up all the values in the vector. The third argument to `accumulate` gives the initial value: it's particularly important because this also determines the data type that `accumulate` uses (so we use `0.0` and not `0` so that `accumulate` uses a `double`, not an `int`.). To use `accumulate()` we need to include the `<numeric>` header.


```cpp
#include <numeric>
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
double sum5(NumericVector x) {
  return std::accumulate(x.begin(), x.end(), 0.0);
}
```

### 25.5.2 Algorithms

The `<algorithm>` header provides a large number of algorithms that work with iterators. A good reference is available at <https://en.cppreference.com/w/cpp/algorithm>. For example, we could write a basic Rcpp version of `findInterval()` that takes two arguments a vector of values and a vector of breaks, and locates the bin that each x falls into. This shows off a few more advanced iterator features. Read the code below and see if you can figure out how it works. \indexc{findInterval()}


```cpp
#include <algorithm>
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
IntegerVector findInterval2(NumericVector x, NumericVector breaks) {
  IntegerVector out(x.size());

  NumericVector::iterator it, pos;
  IntegerVector::iterator out_it;

  for(it = x.begin(), out_it = out.begin(); it != x.end(); 
      ++it, ++out_it) {
    pos = std::upper_bound(breaks.begin(), breaks.end(), *it);
    *out_it = std::distance(breaks.begin(), pos);
  }

  return out;
}
```

The key points are:

* We step through two iterators (input and output) simultaneously.

* We can assign into an dereferenced iterator (`out_it`) to change the values 
  in `out`.

* `upper_bound()` returns an iterator. If we wanted the value of the 
  `upper_bound()` we could dereference it; to figure out its location, we 
  use the `distance()` function.

* Small note: if we want this function to be as fast as `findInterval()` in R 
  (which uses handwritten C code), we need to compute the calls to `.begin()` 
  and `.end()` once and save the results.  This is easy, but it distracts from 
  this example so it has been omitted.  Making this change yields a function
  that's slightly faster than R's `findInterval()` function, but is about 1/10 
  of the code.

It's generally better to use algorithms from the STL than hand rolled loops. In _Effective STL_, Scott Meyers gives three reasons: efficiency, correctness, and maintainability. Algorithms from the STL are written by C++ experts to be extremely efficient, and they have been around for a long time so they are well tested. Using standard algorithms also makes the intent of your code more clear, helping to make it more readable and more maintainable. 

### 25.5.3 Data structures {#data-structures-rcpp}

The STL provides a large set of data structures: `array`, `bitset`, `list`, `forward_list`, `map`, `multimap`, `multiset`, `priority_queue`, `queue`, `deque`, `set`, `stack`, `unordered_map`, `unordered_set`, `unordered_multimap`, `unordered_multiset`, and `vector`.  The most important of these data structures are the `vector`, the `unordered_set`, and the `unordered_map`.  We'll focus on these three in this section, but using the others is similar: they just have different performance trade-offs. For example, the `deque` (pronounced "deck") has a very similar interface to vectors but a different underlying implementation that has different performance trade-offs. You may want to try it for your problem. A good reference for STL data structures is <https://en.cppreference.com/w/cpp/container> --- I recommend you keep it open while working with the STL.

Rcpp knows how to convert from many STL data structures to their R equivalents, so you can return them from your functions without explicitly converting to R data structures.

### 25.5.4 Vectors {#vectors-stl}

An STL vector is very similar to an R vector, except that it grows efficiently. This makes vectors appropriate to use when you don't know in advance how big the output will be.  Vectors are templated, which means that you need to specify the type of object the vector will contain when you create it: `vector<int>`, `vector<bool>`, `vector<double>`, `vector<String>`.  You can access individual elements of a vector using the standard `[]` notation, and you can add a new element to the end of the vector using `.push_back()`.  If you have some idea in advance how big the vector will be, you can use `.reserve()` to allocate sufficient storage. \index{vectors!in C++}

The following code implements run length encoding (`rle()`). It produces two vectors of output: a vector of values, and a vector `lengths` giving how many times each element is repeated. It works by looping through the input vector `x` comparing each value to the previous: if it's the same, then it increments the last value in `lengths`; if it's different, it adds the value to the end of `values`, and sets the corresponding length to 1.


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
List rleC(NumericVector x) {
  std::vector<int> lengths;
  std::vector<double> values;

  // Initialise first value
  int i = 0;
  double prev = x[0];
  values.push_back(prev);
  lengths.push_back(1);

  NumericVector::iterator it;
  for(it = x.begin() + 1; it != x.end(); ++it) {
    if (prev == *it) {
      lengths[i]++;
    } else {
      values.push_back(*it);
      lengths.push_back(1);

      i++;
      prev = *it;
    }
  }

  return List::create(
    _["lengths"] = lengths, 
    _["values"] = values
  );
}
```

(An alternative implementation would be to replace `i` with the iterator `lengths.rbegin()` which always points to the last element of the vector. You might want to try implementing that.)

Other methods of a vector are described at <https://en.cppreference.com/w/cpp/container/vector>.

### 25.5.5 Sets

Sets maintain a unique set of values, and can efficiently tell if you've seen a value before. They are useful for problems that involve duplicates or unique values (like `unique`, `duplicated`, or `in`). C++ provides both ordered (`std::set`) and unordered sets (`std::unordered_set`), depending on whether or not order matters for you. Unordered sets tend to be much faster (because they use a hash table internally rather than a tree), so even if you need an ordered set, you should consider using an unordered set and then sorting the output. Like vectors, sets are templated, so you need to request the appropriate type of set for your purpose: `unordered_set<int>`, `unordered_set<bool>`, etc. More details are available at <https://en.cppreference.com/w/cpp/container/set> and <https://en.cppreference.com/w/cpp/container/unordered_set>. \index{sets}

The following function uses an unordered set to implement an equivalent to `duplicated()` for integer vectors. Note the use of `seen.insert(x[i]).second`. `insert()` returns a pair, the `.first` value is an iterator that points to element and the `.second` value is a Boolean that's true if the value was a new addition to the set.


```cpp
// [[Rcpp::plugins(cpp11)]]
#include <Rcpp.h>
#include <unordered_set>
using namespace Rcpp;

// [[Rcpp::export]]
LogicalVector duplicatedC(IntegerVector x) {
  std::unordered_set<int> seen;
  int n = x.size();
  LogicalVector out(n);

  for (int i = 0; i < n; ++i) {
    out[i] = !seen.insert(x[i]).second;
  }

  return out;
}
```

### 25.5.6 Map
\index{hashmaps}

A map is similar to a set, but instead of storing presence or absence, it can store additional data. It's useful for functions like `table()` or `match()` that need to look up a value. As with sets, there are ordered (`std::map`) and unordered (`std::unordered_map`) versions. Since maps have a value and a key, you need to specify both types when initialising a map: `map<double, int>`, `unordered_map<int, double>`, and so on. The following example shows how you could use a `map` to implement `table()` for numeric vectors:


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
std::map<double, int> tableC(NumericVector x) {
  std::map<double, int> counts;

  int n = x.size();
  for (int i = 0; i < n; i++) {
    counts[x[i]]++;
  }

  return counts;
}
```

### 25.5.7 Exercises

To practice using the STL algorithms and data structures, implement the following using R functions in C++, using the hints provided:

1. `median.default()` using `partial_sort`.


```cpp
#include <algorithm>  // Include the C++ standard algorithms library
#include <Rcpp.h>     // Include Rcpp, a package for interfacing R and C++
using namespace Rcpp;  // Import the Rcpp namespace for convenience

// Declare a function named 'medianC' with the attribute [[Rcpp::export]]
// This allows the function to be called from R.
// The function takes a NumericVector 'x' as its argument.
// NumericVector is a vector type used in Rcpp for numerical data.
// The function returns a double, which is the median of the input vector.

// [[Rcpp::export]]
double medianC(NumericVector x) {
  int n = x.size();  // Get the size (number of elements) of the input vector 'x'

  if (n % 2 == 0) {
    // If the vector has an even number of elements:
    // Sort the first half of the vector in ascending order.
    std::partial_sort(x.begin(), x.begin() + n / 2 + 1, x.end());
    
    // Return the median, which is the average of the middle two elements.
    return (x[n / 2 - 1] + x[n / 2]) / 2;
  } else {
    // If the vector has an odd number of elements:
    // Sort the first (n + 1) / 2 elements in ascending order.
    std::partial_sort(x.begin(), x.begin() + (n + 1) / 2, x.end());
    
    // Return the middle element as the median.
    return x[(n + 1) / 2 - 1];
  }
}

```


```r
# Define new test vectors
test_vector1 <- c(2, 4, 6, 8, 10)     # Even-sized vector
test_vector2 <- c(3, 1, 4, 1, 5, 9)  # Odd-sized vector

# Test the medianC function with the new vectors
medianC_result1 <- medianC(test_vector1)
medianC_result2 <- medianC(test_vector2)

# Print the results
cat("Using medianC function:\n")
```

```
## Using medianC function:
```

```r
cat("Median of test_vector1:", medianC_result1, "\n")
```

```
## Median of test_vector1: 6
```

```r
cat("Median of test_vector2:", medianC_result2, "\n")
```

```
## Median of test_vector2: 3.5
```

2. `%in%` using `unordered_set` and the `find()` or `count()` methods.


```cpp
#include <Rcpp.h>     // Include Rcpp, a package for interfacing R and C++
#include <unordered_set> // Include the C++ standard unordered_set container
using namespace Rcpp;  // Import the Rcpp namespace for convenience

// Declare a function named 'inC' with the attribute [[Rcpp::export]]
// This allows the function to be called from R.
// The function takes two CharacterVector arguments, 'x' and 'table'.
// CharacterVector is a data type used in Rcpp for character vectors.
// The function returns a LogicalVector, indicating element presence in 'table'.
// [[Rcpp::export]]
LogicalVector inC(CharacterVector x, CharacterVector table) {
  std::unordered_set<String> seen;
  seen.insert(table.begin(), table.end());  // Create an unordered_set from 'table'
  
  int n = x.size();  // Get the size (number of elements) of the 'x' vector
  LogicalVector out(n);  // Create a LogicalVector 'out' to store the results

  // Loop through the elements in 'x' and check for membership in 'table'
  for (int i = 0; i < n; ++i) {
    out[i] = seen.find(x[i]) != seen.end(); // Set 'out[i]' to true if 'x[i]' is found in 'table'
  }

  return out;  // Return the LogicalVector containing the membership results
}

```


```r
# Define vectors for testing
x <- c("apple", "banana", "cherry")
query_vector <- c("banana", "date")

# Test the inC function with different vectors
result1 <- query_vector %in% x
result2 <- inC(query_vector, x)

# Print the results
cat("Using %in% operator in R:\n")
```

```
## Using %in% operator in R:
```

```r
cat("Membership check:", result1, "\n\n")
```

```
## Membership check: TRUE FALSE
```

```r
cat("Using inC function (C++):\n")
```

```
## Using inC function (C++):
```

```r
cat("Membership check:", result2, "\n")
```

```
## Membership check: TRUE FALSE
```


3. `unique()` using an `unordered_set` (challenge: do it in one line!).


```cpp
#include <Rcpp.h>
#include <unordered_set>
using namespace Rcpp;


// Declare a function named 'uniqueCC' with the attribute [[Rcpp::export]]
// This function is a one-liner that directly returns an unordered_set.
// [[Rcpp::export]]
std::unordered_set<double> uniqueCC(NumericVector x) {
  // Create and return an unordered_set directly from the input vector 'x'.
  return std::unordered_set<double>(x.begin(), x.end());
}

```


```r
# Define the test vector
x <- c(5, 5, 3, 1, 3, 8, 2, 8)

# Find unique elements using R's built-in 'unique' function
result1 <- unique(x)

# Find unique elements using the 'uniqueC' function (C++)
result2 <- uniqueCC(x)

# Print the results
cat("Using R's built-in 'unique' function:\n")
```

```
## Using R's built-in 'unique' function:
```

```r
cat("Unique elements:", result1, "\n\n")
```

```
## Unique elements: 5 3 1 8 2
```

```r
cat("Using 'uniqueC' function (C++):\n")
```

```
## Using 'uniqueC' function (C++):
```

```r
cat("Unique elements:", result2, "\n")
```

```
## Unique elements: 8 1 3 2 5
```


4. `min()` using `std::min()`, or `max()` using `std::max()`.


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
double minC(NumericVector x) {
  // Calculate the size (number of elements) of the input vector 'x'
  int n = x.size();
  
  // Initialize 'out' to the first element of the vector 'x'
  double out = x[0];
  
  // Iterate through the elements of the vector to find the minimum value
  for (int i = 0; i < n; i++) {
    // Compare the current value of 'out' (the minimum value found so far)
    // with the current element 'x[i]' using the std::min function.
    // Update 'out' to the smaller of the two values.
    out = std::min(out, x[i]);
  }
  
  // Return the minimum value found in the vector 'x'
  return out;
}

```


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
double maxC(NumericVector x) {
  int n = x.size();
  double out = x[0];
  
  for (int i = 0; i < n; i++) {
    out = std::max(out, x[i]);
  }
  
  return out;
}

```


```r
# Define a test vector
x <- c(5, 12, 7, 1, 9, 15, 2, 8)

# Find the minimum and maximum values using R's built-in functions
min_result_r <- min(x)
max_result_r <- max(x)

# Find the minimum and maximum values using the 'minC' and 'maxC' functions (C++)
min_result_c <- minC(x)
max_result_c <- maxC(x)

# Print and compare the results
cat("Using R's built-in 'min' function:\n")
```

```
## Using R's built-in 'min' function:
```

```r
cat("Minimum value:", min_result_r, "\n")
```

```
## Minimum value: 1
```

```r
cat("Maximum value:", max_result_r, "\n\n")
```

```
## Maximum value: 15
```

```r
cat("Using 'minC' and 'maxC' functions (C++):\n")
```

```
## Using 'minC' and 'maxC' functions (C++):
```

```r
cat("Minimum value:", min_result_c, "\n")
```

```
## Minimum value: 1
```

```r
cat("Maximum value:", max_result_c, "\n")
```

```
## Maximum value: 15
```

5. `which.min()` using `min_element`, or `which.max()` using `max_element`.


```cpp
#include <Rcpp.h>
#include <algorithm>
#include <iterator>
using namespace Rcpp;

// [[Rcpp::export]]
double which_minC(NumericVector x) {
  // Use the std::min_element function to find the iterator pointing to the minimum element in the range defined by x.begin() and x.end().
  // This iterator represents the minimum element in the NumericVector.
  NumericVector::iterator min_element_iterator = std::min_element(x.begin(), x.end());

  // Calculate the index of the minimum element by calculating the distance between the beginning of the vector (x.begin())
  // and the iterator pointing to the minimum element.
  // The result is stored in the 'out' variable.
  int out = std::distance(x.begin(), min_element_iterator);

  // Return the 1-based index of the minimum element.
  // In R, indexing is 1-based, so we add 1 to the result to make it consistent with R's indexing.
  return out + 1;
}

```


```cpp
#include <Rcpp.h>
#include <algorithm>
#include <iterator>
using namespace Rcpp;

// [[Rcpp::export]]
double which_maxC(NumericVector x) {
  // Use the std::max_element function to find the iterator pointing to the maximum element in the range defined by x.begin() and x.end().
  // This iterator represents the maximum element in the NumericVector.
  NumericVector::iterator max_element_iterator = std::max_element(x.begin(), x.end());

  // Calculate the index of the maximum element by calculating the distance between the beginning of the vector (x.begin())
  // and the iterator pointing to the maximum element.
  // The result is stored in the 'out' variable.
  int out = std::distance(x.begin(), max_element_iterator);

  // Return the 1-based index of the maximum element.
  // In R, indexing is 1-based, so we add 1 to the result to make it consistent with R's indexing.
  return out + 1;
}

```


```r
# Define a test vector
x <- c(5, 12, 7, 1, 9, 15, 2, 8)

# Find the index of the minimum and maximum values using R's built-in functions
which_min_result_r <- which.min(x)
which_max_result_r <- which.max(x)

# Find the index of the minimum and maximum values using the 'which_minC' and 'which_maxC' functions (C++)
which_min_result_c <- which_minC(x)
which_max_result_c <- which_maxC(x)

# Print and compare the results
cat("Using R's built-in 'which.min' and 'which.max' functions:\n")
```

```
## Using R's built-in 'which.min' and 'which.max' functions:
```

```r
cat("Index of minimum value:", which_min_result_r, "\n")
```

```
## Index of minimum value: 4
```

```r
cat("Index of maximum value:", which_max_result_r, "\n\n")
```

```
## Index of maximum value: 6
```

```r
cat("Using 'which_minC' and 'which_maxC' functions (C++):\n")
```

```
## Using 'which_minC' and 'which_maxC' functions (C++):
```

```r
cat("Index of minimum value:", which_min_result_c, "\n")
```

```
## Index of minimum value: 4
```

```r
cat("Index of maximum value:", which_max_result_c, "\n")
```

```
## Index of maximum value: 6
```


6. `setdiff()`, `union()`, and `intersect()` for integers using sorted ranges 
   and `set_union`, `set_intersection` and `set_difference`.
   

```cpp
#include <Rcpp.h>
#include <unordered_set>
#include <algorithm>
using namespace Rcpp;

// Enable C++11 features
// [[Rcpp::plugins(cpp11)]]

// Function to find the union of two IntegerVectors without duplicates
// [[Rcpp::export]]
IntegerVector unionC(IntegerVector x, IntegerVector y) {
  int nx = x.size();
  int ny = y.size();
  
  // Create a temporary IntegerVector with enough space to hold all elements from both input vectors
  IntegerVector tmp(nx + ny);
  
  // Sort both input vectors to prepare for set operations
  std::sort(x.begin(), x.end()); // Sorting for uniqueness
  std::sort(y.begin(), y.end());
  
  // Use std::set_union to find the union of the two sorted vectors and store it in 'tmp'
  IntegerVector::iterator out_end = std::set_union(
    x.begin(), x.end(), y.begin(), y.end(), tmp.begin()
  );
  
  int prev_value = 0;
  IntegerVector out;
  // Iterate through 'tmp' and remove duplicate elements
  for (IntegerVector::iterator it = tmp.begin();
       it != out_end; ++it) {
    if ((it != tmp.begin()) && (prev_value == *it)) continue;
    
    out.push_back(*it);
    
    prev_value = *it;
  }
  
  // Return the result, which is the union of 'x' and 'y' without duplicates
  return out;
}

// Function to find the intersection of two IntegerVectors without duplicates
// [[Rcpp::export]]
IntegerVector intersectC(IntegerVector x, IntegerVector y) {
  int nx = x.size();
  int ny = y.size();
  
  // Create a temporary IntegerVector with enough space to hold the intersection
  IntegerVector tmp(std::min(nx, ny));
  
  // Sort both input vectors to prepare for set operations
  std::sort(x.begin(), x.end());
  std::sort(y.begin(), y.end());
  
  // Use std::set_intersection to find the intersection of the two sorted vectors and store it in 'tmp'
  IntegerVector::iterator out_end = std::set_intersection(
    x.begin(), x.end(), y.begin(), y.end(), tmp.begin()
  );
  
  int prev_value = 0;  
  IntegerVector out;
  // Iterate through 'tmp' and remove duplicate elements
  for (IntegerVector::iterator it = tmp.begin();
       it != out_end; ++it) {
    if ((it != tmp.begin()) && (prev_value == *it)) continue;
    
    out.push_back(*it);
    
    prev_value = *it;
  }
  
  // Return the result, which is the intersection of 'x' and 'y' without duplicates
  return out;
}

// Function to find the set difference of two IntegerVectors
// [[Rcpp::export]]
IntegerVector setdiffC(IntegerVector x, IntegerVector y) {
  int nx = x.size();
  int ny = y.size();
  
  // Create a temporary IntegerVector to store the result
  IntegerVector tmp(nx);
  
  // Sort 'x' to prepare for set difference
  std::sort(x.begin(), x.end());
  
  int prev_value = 0;
  IntegerVector x_dedup;
  // Iterate through 'x' and remove duplicate elements
  for (IntegerVector::iterator it = x.begin();
       it != x.end(); ++it) {
    if ((it != x.begin()) && (prev_value == *it)) continue;
    
    x_dedup.push_back(*it);
    
    prev_value = *it;
  }
  
  // Sort 'y' to prepare for set difference
  std::sort(y.begin(), y.end());
  
  // Use std::set_difference to find the set difference of 'x_dedup' and 'y' and store it in 'tmp'
  IntegerVector::iterator out_end = std::set_difference(
    x_dedup.begin(), x_dedup.end(), y.begin(), y.end(), tmp.begin()
  );
  
  IntegerVector out;
  // Iterate through 'tmp' and store the result
  for (IntegerVector::iterator it = tmp.begin();
       it != out_end; ++it) {
    out.push_back(*it);
  }
  
  // Return the result, which is the set difference of 'x' and 'y'
  return out;
}

```


```r
# Define new test vectors
x <- c(1, 2, 3, 4, 5, 6, 7)
y <- c(4, 5, 6, 7, 8, 9, 10)

# Find the union, intersection, and set difference using the C++ functions
union_result_c <- unionC(x, y)
intersect_result_c <- intersectC(x, y)
setdiff_result_c <- setdiffC(x, y)

# Print the results
cat("Union (C++):\n")
```

```
## Union (C++):
```

```r
cat(union_result_c, "\n")
```

```
## 1 2 3 4 5 6 7 8 9 10
```

```r
cat("Intersection (C++):\n")
```

```
## Intersection (C++):
```

```r
cat(intersect_result_c, "\n")
```

```
## 4 5 6 7
```

```r
cat("Set Difference (C++):\n")
```

```
## Set Difference (C++):
```

```r
cat(setdiff_result_c, "\n")
```

```
## 1 2 3
```


## Case studies {#rcpp-case-studies}

The following case studies illustrate some real life uses of C++ to replace slow R code.

### Gibbs sampler

<!-- FIXME: needs more context? -->

The following case study updates an example [blogged about](http://dirk.eddelbuettel.com/blog/2011/07/14/) by Dirk Eddelbuettel, illustrating the conversion of a Gibbs sampler in R to C++. The R and C++ code shown below is very similar (it only took a few minutes to convert the R version to the C++ version), but runs about 20 times faster on my computer. Dirk's blog post also shows another way to make it even faster: using the faster random number generator functions in GSL (easily accessible from R through the RcppGSL package) can make it another two to three times faster. \index{Gibbs sampler}

The R code is as follows:


```r
gibbs_r <- function(N, thin) {
  mat <- matrix(nrow = N, ncol = 2)
  x <- y <- 0

  for (i in 1:N) {
    for (j in 1:thin) {
      x <- rgamma(1, 3, y * y + 4)
      y <- rnorm(1, 1 / (x + 1), 1 / sqrt(2 * (x + 1)))
    }
    mat[i, ] <- c(x, y)
  }
  mat
}
```

This is straightforward to convert to C++.  We:

* Add type declarations to all variables.

* Use `(` instead of `[` to index into the matrix.

* Subscript the results of `rgamma` and `rnorm` to convert from a vector 
  into a scalar.


```cpp
#include <Rcpp.h>
using namespace Rcpp;

// [[Rcpp::export]]
NumericMatrix gibbs_cpp(int N, int thin) {
  NumericMatrix mat(N, 2);
  double x = 0, y = 0;

  for(int i = 0; i < N; i++) {
    for(int j = 0; j < thin; j++) {
      x = rgamma(1, 3, 1 / (y * y + 4))[0];
      y = rnorm(1, 1 / (x + 1), 1 / sqrt(2 * (x + 1)))[0];
    }
    mat(i, 0) = x;
    mat(i, 1) = y;
  }

  return(mat);
}
```

Benchmarking the two implementations yields:


```r
bench::mark(
  gibbs_r(100, 10),
  gibbs_cpp(100, 10),
  check = FALSE
)
```

```
## # A tibble: 2 × 6
##   expression              min   median `itr/sec` mem_alloc `gc/sec`
##   <bch:expr>         <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
## 1 gibbs_r(100, 10)     7.66ms   14.8ms      71.5    4.98MB     7.15
## 2 gibbs_cpp(100, 10)  414.9µs    896µs    1210.      4.1KB     8.36
```

### R vectorisation versus C++ vectorisation

<!-- FIXME: needs more context? -->

This example is adapted from ["Rcpp is smoking fast for agent-based models in data frames"](https://gweissman.github.io/post/rcpp-is-smoking-fast-for-agent-based-models-in-data-frames/). The challenge is to predict a model response from three inputs. The basic R version of the predictor looks like:


```r
vacc1a <- function(age, female, ily) {
  p <- 0.25 + 0.3 * 1 / (1 - exp(0.04 * age)) + 0.1 * ily
  p <- p * if (female) 1.25 else 0.75
  p <- max(0, p)
  p <- min(1, p)
  p
}
```

We want to be able to apply this function to many inputs, so we might write a vector-input version using a for loop.


```r
vacc1 <- function(age, female, ily) {
  n <- length(age)
  out <- numeric(n)
  for (i in seq_len(n)) {
    out[i] <- vacc1a(age[i], female[i], ily[i])
  }
  out
}
```

If you're familiar with R, you'll have a gut feeling that this will be slow, and indeed it is. There are two ways we could attack this problem. If you have a good R vocabulary, you might immediately see how to vectorise the function (using `ifelse()`, `pmin()`, and `pmax()`). Alternatively, we could rewrite `vacc1a()` and `vacc1()` in C++, using our knowledge that loops and function calls have much lower overhead in C++.

Either approach is fairly straightforward. In R:


```r
vacc2 <- function(age, female, ily) {
  p <- 0.25 + 0.3 * 1 / (1 - exp(0.04 * age)) + 0.1 * ily
  p <- p * ifelse(female, 1.25, 0.75)
  p <- pmax(0, p)
  p <- pmin(1, p)
  p
}
```

(If you've worked R a lot you might recognise some potential bottlenecks in this code: `ifelse`, `pmin`, and `pmax` are known to be slow, and could be replaced with `p * 0.75 + p * 0.5 * female`, `p[p < 0] <- 0`, `p[p > 1] <- 1`.  You might want to try timing those variations.)

Or in C++:


```cpp
#include <Rcpp.h>
using namespace Rcpp;

double vacc3a(double age, bool female, bool ily){
  double p = 0.25 + 0.3 * 1 / (1 - exp(0.04 * age)) + 0.1 * ily;
  p = p * (female ? 1.25 : 0.75);
  p = std::max(p, 0.0);
  p = std::min(p, 1.0);
  return p;
}

// [[Rcpp::export]]
NumericVector vacc3(NumericVector age, LogicalVector female, 
                    LogicalVector ily) {
  int n = age.size();
  NumericVector out(n);

  for(int i = 0; i < n; ++i) {
    out[i] = vacc3a(age[i], female[i], ily[i]);
  }

  return out;
}
```

We next generate some sample data, and check that all three versions return the same values:


```r
n <- 1000
age <- rnorm(n, mean = 50, sd = 10)
female <- sample(c(T, F), n, rep = TRUE)
ily <- sample(c(T, F), n, prob = c(0.8, 0.2), rep = TRUE)

stopifnot(
  all.equal(vacc1(age, female, ily), vacc2(age, female, ily)),
  all.equal(vacc1(age, female, ily), vacc3(age, female, ily))
)
```

The original blog post forgot to do this, and introduced a bug in the C++ version: it used `0.004` instead of `0.04`.  Finally, we can benchmark our three approaches:


```r
bench::mark(
  vacc1 = vacc1(age, female, ily),
  vacc2 = vacc2(age, female, ily),
  vacc3 = vacc3(age, female, ily)
)
```

```
## # A tibble: 3 × 6
##   expression      min   median `itr/sec` mem_alloc `gc/sec`
##   <bch:expr> <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
## 1 vacc1        1.72ms   1.82ms      492.    7.86KB    30.0 
## 2 vacc2        74.2µs   89.9µs     9361.  148.85KB    13.2 
## 3 vacc3        45.1µs   46.8µs    19182.   14.48KB     4.17
```

Not surprisingly, our original approach with loops is very slow.  Vectorising in R gives a huge speedup, and we can eke out even more performance (about ten times) with the C++ loop. I was a little surprised that the C++ was so much faster, but it is because the R version has to create 11 vectors to store intermediate results, where the C++ code only needs to create 1.

## Using Rcpp in a package {#rcpp-package}

The same C++ code that is used with `sourceCpp()` can also be bundled into a package. There are several benefits of moving code from a stand-alone C++ source file to a package: \index{Rcpp!in a package}

1. Your code can be made available to users without C++ development tools.

1. Multiple source files and their dependencies are handled automatically by 
   the R package build system.

1. Packages provide additional infrastructure for testing, documentation, and 
   consistency.

To add `Rcpp` to an existing package, you put your C++ files in the `src/` directory and create or modify the following configuration files:

*   In `DESCRIPTION` add

    ```
    LinkingTo: Rcpp
    Imports: Rcpp
    ```

*   Make sure your `NAMESPACE` includes:

    ```
    useDynLib(mypackage)
    importFrom(Rcpp, sourceCpp)
    ```

    We need to import something (anything) from Rcpp so that internal Rcpp code 
    is properly loaded. This is a bug in R and hopefully will be fixed in the 
    future.
    
The easiest way to set this up automatically is to call `usethis::use_rcpp()`.

Before building the package, you'll need to run `Rcpp::compileAttributes()`. This function scans the C++ files for `Rcpp::export` attributes and generates the code required to make the functions available in R. Re-run `compileAttributes()` whenever functions are added, removed, or have their signatures changed. This is done automatically by the devtools package and by Rstudio.

For more details see the Rcpp package vignette, `vignette("Rcpp-package")`.

## Learning more {#rcpp-more}

This chapter has only touched on a small part of Rcpp, giving you the basic tools to rewrite poorly performing R code in C++. As noted, Rcpp has many other capabilities that make it easy to interface R to existing C++ code, including:

* Additional features of attributes including specifying default arguments,
  linking in external C++ dependencies, and exporting C++ interfaces from 
  packages. These features and more are covered in the Rcpp attributes vignette,
  `vignette("Rcpp-attributes")`.

* Automatically creating wrappers between C++ data structures and R data 
  structures, including mapping C++ classes to reference classes. A good 
  introduction to this topic is the Rcpp modules vignette, 
  `vignette("Rcpp-modules")`.

* The Rcpp quick reference guide, `vignette("Rcpp-quickref")`, contains a useful 
  summary of Rcpp classes and common programming idioms.

I strongly recommend keeping an eye on the [Rcpp homepage](http://www.rcpp.org) and signing up for the [Rcpp mailing list](http://lists.r-forge.r-project.org/cgi-bin/mailman/listinfo/rcpp-devel).

Other resources I've found helpful in learning C++ are:

* _Effective C++_ [@effective-cpp] and _Effective STL_ [@effective-stl].

* [_C++ Annotations_](http://www.icce.rug.nl/documents/cplusplus/cplusplus.html), 
  aimed at knowledgeable users of C (or any other language using a C-like 
  grammar, like Perl or Java) who would like to know more about, or make the 
  transition to, C++.

* [_Algorithm Libraries_](http://www.cs.helsinki.fi/u/tpkarkka/alglib/k06/), 
  which provides a more technical, but still concise, description of 
  important STL concepts. (Follow the links under notes.)

Writing performance code may also require you to rethink your basic approach: a solid understanding of basic data structures and algorithms is very helpful here. That's beyond the scope of this book, but I'd suggest the _Algorithm Design Manual_ [@alg-design-man], MIT's [_Introduction to Algorithms_](http://ocw.mit.edu/courses/electrical-engineering-and-computer-science/6-046j-introduction-to-algorithms-sma-5503-fall-2005/), _Algorithms_ by Robert Sedgewick and Kevin Wayne which has a free [online textbook](http://algs4.cs.princeton.edu/home/) and a matching [Coursera course](https://www.coursera.org/learn/algorithms-part1).

## Acknowledgments

I'd like to thank the Rcpp-mailing list for many helpful conversations, particularly Romain Francois and Dirk Eddelbuettel who have not only provided detailed answers to many of my questions, but have been incredibly responsive at improving Rcpp. This chapter would not have been possible without JJ Allaire; he encouraged me to learn C++ and then answered many of my dumb questions along the way.
