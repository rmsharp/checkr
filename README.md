## Checkr <a href="https://travis-ci.org/peterhurford/checkr"><img src="https://img.shields.io/travis/peterhurford/checkr.svg"></a> <a href="https://codecov.io/github/peterhurford/checkr"><img src="https://img.shields.io/codecov/c/github/peterhurford/checkr.svg"></a> <a href="https://github.com/peterhurford/checkr/tags"><img src="https://img.shields.io/github/tag/peterhurford/checkr.svg"></a>

R is a dynamically typed language. This is pretty great for writing code quickly, but bad for architecturing large systems.

While [some have tried admirably](https://github.com/zatonovo/lambda.r), it seems like a bad idea to enforce a lot of static checks on R functions.  However, we can still be explicit about our preconditions and postconditions when writing R functions, to adopt a solid functional style.

Things still die on runtime instead of compile-time, which is sad, but functions are more explicit about what they do and less tests need to be written -- better than the status quo!

Another problem is writing tests.  Thanks to Hadley's [testthat package](https://github.com/hadley/testthat), writing tests for R code [is pretty easy](http://r-pkgs.had.co.nz/).  But writing tests take a long time and it's easy to forget to write certain tests.  And [while coverage tools in R exist](https://github.com/jimhester/covr), 100% test coverage is still insufficient for verifying that your code works.

Quickcheck, inspired by [the Haskell namesake](https://github.com/nick8325/quickcheck) (and in true Haskell style you can [see the corresponding academic paper](http://www.eecs.northwestern.edu/~robby/courses/395-495-2009-fall/quick.pdf)), aims to automatically verify your code through running hundreds of tests that you don't have to write yourself. Checkr implements a version of this in R so that you can generate tests automatically without having to write them yourself.

**Checkr** provides helpers to easily validate and test R functions.


## Validations

```R
library(checkr)

#' Add two numbers.
#'
#' @param x numeric. The number to add.
#' @param y numeric. The number to add.
add <- ensure(
  pre = list(x %is% numeric, y %is% numeric),
  post = result %is% numeric,  # `result` matches whatever the function returns.
  function(x, y) x + y)
add(1, 2)
# 3
add("a", 2)
# Error on x %is% numeric
add("a", "b")
# Error on x %is% numeric, y %is% numeric.
```

```R
#' Generate a random string.
#'
#' @param length numeric. The length of the random string to generate.
#' @param alphabet character. A list of characters to draw from to create the string.
random_string <- ensure(
  pre = list(length %is% numeric, length(length) == 1, length > 0,
    length < 1e+7, length %% 1 == 0,
    alphabet %is% list || alphabet %is% vector,
    alphabet %contains_only% simple_string,
    all(sapply(alphabet, nchar) == 1)),
  post = list(result %is% simple_string, nchar(result) == length),
  function(length, alphabet) {
    paste0(sample(alphabet, length, replace = TRUE), collapse = "")
  })
```


## Using Quickcheck

#### The Random String

Imagine you want to generate a random string of a given length from a given possible alphabet of characters.  Your R function might look like this:

```R
random_string <- function(length, alphabet) {
  paste0(sample(alphabet, 10), collapse = "")
}
```

This is a pretty simple function, but it's possible to make an error even on something this simple -- as you can see, we accidentally hardcoded the length as 10 instead of using the built-in `length` parameter (this isn't contrived -- this is a typo [I have made in real life](https://github.com/peterhurford/checkr/commit/585af6de4ee25622dfaa665e83106a2398cc946c)).

We may write some tests using testthat:

```R
test_that("it generates a random string from the given alphabet", {
  random_string <- random_string(10, letters)
  all(strsplit(random_string, "")[[1]] %in% letters)
})
test_that("it generates a random string of the given length", {
  random_string <- random_string(10, letters)
  expect_equal(nchar(random_string), 10)
})
```

But because we were lazy when writing the tests and had 10 in our mind, all the tests pass and we don't catch our error.

Additionally, we don't look for other errors, such as:

(a) Does it work when the alphabet is only a length 1 list?

(b) Does it work when the alphabet is a string?

(c) Does it work when length is a negative number?

(d) Does it work when length is a list?

For example, if we had written a thorough test for (a), we would have noticed that we're using `sample` with `replace = FALSE`, which means that if the `length` is larger than `length(alphabet)`, the function will crash.  We should use `replace = TRUE` instead!

...So we could add all these tests ourselves and be really thorough, or we could use quickcheck and just automatically test some simple properties:

```R
quickcheck(ensure(
  pre = list(length %is% numeric, length(length) == 1, length > 0, length < 1e+7,
    alphabet %is% list || alphabet %is% vector,
      alphabet %contains_only% simple_string),
  post = list(nchar(result) == length, length(result) == 1,
    is.character(result), all(strsplit(result, "")[[1]] %in% alphabet)),
  random_string))
```
```
Error: Quickcheck for random_string failed on item #1: length = 53L, alphabet = list("shtafWoWGRWmCSIRquDNxqskiKGyVdHFApld")
```

We use `ensure` to specify preconditions for what random items we should test our function with and to specify postconditions that must hold true for every run that satisfies the preconditions.  (I recommend making every-day use of validations even if not doing `quickcheck`, because it creates more clear functions that are more explicit about what they require and are less likely to crash in confusing ways.  ...They're also way easier to quickcheck.)

This quickcheck will automatically generate possible arguments that match the preconditions and then do some verifications, such as (a) verifying that the number of characters of the resulting string is the same as the `length` that you passed into the function, (b) that the resulting string is not a length > 1 vector, (c) that the resulting string is all characters, and (d) that all the characters in the string are within the given `alphabet`.

This easily accomplishes in two lines what normally takes five well thought-out and detailed tests.

(Why `length < 1e+7`?... Another thing I learned only by quickchecking -- you can break `sample` with sufficiently large lengths.)

#### Reversing and Property-based Testing

Let's say that we want to be confident that the `rev` function in R's base works as intended to reverse a list.  We could create a few test cases ourselves, or we could use quickcheck to specify **properties** that should hold about `rev` are actually true over our randomly generated examples:

First, we know that reversing a length-1 list should be itself.

```R
quickcheck(ensure(pre = list(length(x) == 1, x %is% vector || x %is% list),
  post = identical(result, x), function(x) rev(x)))
```

And when we run the Quickcheck, we get:

```
Quickcheck for function(x) rev(x) passed on 132 random examples!
```

...Here, 576 random possible test objects were created and these objects were filtered down to the 132 ones that met the specified preconditions (input must be a length 1 vector or list). All of these were then sent to the `rev` function and the result was then checked against the postcondition that `identical(result, x)` to make sure the result is identical to the original `x`.

And we can also test that the reverse of a reverse of a list is that same list:

```R
quickcheck(ensure(pre = list(x %is% vector || x %is% list),
  post = identical(result, x), function(x) rev(rev(x))))
```
```
Quickcheck for function(x) rev(rev(x)) passed on 708 random examples!
```


## Why not use Quickcheck by Revolution Analytics?

In June 2015 (8 months before me), Revolution Analytics released [their own version of Quickcheck for R](https://github.com/RevolutionAnalytics/quickcheck) which works [to also automatically verify properties of R functions](https://github.com/RevolutionAnalytics/quickcheck/blob/master/docs/tutorial.md).

However, this version of Quickcheck has a few important improvements:

(1) The tight integration with validations lets you more clearly specify the preconditions and postconditions.

(2) You can be a lot more specific about the preconditions you can specify on the random objects. Revolution Analytics' objects are always one class and all objects of that class, whereas with this package you can mix and match classes and specify other things (e.g., all >0).

(3) The random object generator is smarter (a.k.a. biased), making sure to explicitly test important things you might forget (e.g., a vector of all negative numbers) and that might not come up in a truly random generator (like Revolution Analytics').

(4) This version naturally integrates with Hadley's popular testthat package.


## Installation

This package is not yet available from CRAN.  Instead, it can be installed using [devtools](http://www.github.com/hadley/devtools):

```R
if (!require("devtools")) { install.packages("devtools") }
devtools::install_github("peterhurford/checkr")
```


## Credits

Inspired by [Cobra](http://cobra-language.com/).

Similar to the [ensurer](https://github.com/smbache/ensurer) package (and I think these two packages would work well together), but I didn't remember that package existed until now.

Also similar in syntax to [RDL](https://github.com/plum-umd/rdl) in Ruby, which I did not know about until months after I made this package.
