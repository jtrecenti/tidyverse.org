---
title: tibble 3.0.0
slug: tibble-3-0-0
description: >
    tibble 3.0.0 is on CRAN now! Tibbles are a modern reimagining of the data frame, keeping what time has shown to be effective, and throwing out what is not, with nicer default output too! This article describes the latest major release and provides an outlook on further developments
date: 2020-04-09
author: Kirill Müller
photo:
  url: https://unsplash.com/photos/dbOV1qSiL-c
  author: Vinicius Amano
categories:
- package
- programming
---





Version 3.0.0 of the tibble package is on CRAN now. Tibbles are a modern reimagining of the data frame, keeping what time has shown to be effective, and throwing out what is not, with nicer default output too! Grab the latest version with:

```r
install.packages("tibble")
```

Tibble now fully embraces vctrs, using it under the hood for its subsetting and subset assignment ("subassignment") operations.
Accessing and updating rows and columns is now based on a rock-solid framework and works consistently for all types of columns, including list, data frame, and 
matrix columns.
We believe that the changes will ultimately lead to better and safer code.

This major release required quite some preparation, including a [new vignette](https://tibble.tidyverse.org/articles/invariants.html) that describes the behavior of subsetting and subset assignment operations and the reasoning behind it.
For a complete overview please see the [release notes](https://tibble.tidyverse.org/news/index.html).

In a nutshell: if an object is a vector, it can be part of a tibble.
My new [Awesome vectors](https://github.com/krlmlr/awesome-vctrs#readme) list aims at giving an overview of available implementations of vector types in R.
If you're using a specialized class, or even implemented one, please file an issue in that repository or contribute an example.
For problems with tibble, use the [issue tracker](https://github.com/tidyverse/tibble/issues) to report bugs or suggest ideas, your contributions are always welcome.

The rest of the post is about the technical details of a tibble, and therefore mostly suited for interested R programmers:

- What can be part of a tibble?
- Size and length
- Sturdy recycling


## What can be part of a tibble?

Tibbles and data frames are collections of columns, where each column is a vector of the same size.
Neat.

What is a vector?
What is its size?

The new [vctrs package](https://vctrs.r-lib.org) is dedicated to answering these surprisingly tricky questions.
Because this blog post describes many functions of this package, we load it.


```r
library(vctrs)
```

The [`vec_is()`](https://vctrs.r-lib.org/reference/vec_assert.html) function decides if an object is a vector.
This is important, because some objects are inherently scalar and cannot be added as a column to a data frame.

Obviously, integers, characters, and other atomic objects (logical, numeric, complex, and raw) are vectors.
Environments, functions, and other "special" types of objects are clearly non-vectors.
Most objects that consist of an atomic type with a `"class"` attribute are also vectors: examples are `POSIXct` and [`hms::hms()`](https://hms.tidyverse.org/reference/hms.html).
Lists are harder because some lists are vectors and some are not.

The `vec_is()` function implements a heuristic that works automatically in most cases and adds a few special cases from base R.
By relying on `vec_is()`, the `tibble()` function and others can give an early error if used with an inherent scalar:


```r
library(tibble)
model <- lm(y ~ x, data.frame(x = 1:3, y = 2:4), model = FALSE)
tibble(model)
#> Error: All columns in a tibble must be vectors.
#> x Column `model` is a `lm` object.
time <- Sys.time()
tibble(time)
#> # A tibble: 1 x 1
#>   time               
#>   <dttm>             
#> 1 2020-04-09 20:38:28
```

The new [`tibble_row()`](https://tibble.tidyverse.org/reference/tibble.html) function reverses this: inherent scalars are wrapped in lists:


```r
tibble_row(model)
#> # A tibble: 1 x 1
#>   model 
#>   <list>
#> 1 <lm>
tibble_row(time)
#> # A tibble: 1 x 1
#>   time               
#>   <dttm>             
#> 1 2020-04-09 20:38:28
tibble_row(time = rep(time, 2))
#> Error: All vectors must be size one, use `list()` to wrap.
#> x Column `time` is of size 2.
```

If you have implemented a vector class, double-check that [`vec_is()`](https://vctrs.r-lib.org/reference/vec_assert.html) returns `TRUE` for your objects.
Please also add it to my [Awesome vectors](https://github.com/krlmlr/awesome-vctrs#readme) list, or file an issue.


## Size and length

Data frames and matrices are also recognized vectors, and can be part of a tibble:


```r
df <- data.frame(a = 1:3, b = 2:4)
m <- matrix(1:6, ncol = 3)

vec_is(df)
#> [1] TRUE
vec_is(m)
#> [1] TRUE
tibble(packed = df)
#> # A tibble: 3 x 1
#>   packed$a    $b
#>      <int> <int>
#> 1        1     2
#> 2        2     3
#> 3        3     4
tibble(m)
#> # A tibble: 2 x 1
#>   m[,1]  [,2]  [,3]
#>   <int> <int> <int>
#> 1     1     3     5
#> 2     2     4     6
```

The "elements" of a data frame or matrix are its rows.
All subsetting and subassignment operations now use [`vec_slice()`](https://vctrs.r-lib.org/reference/vec_slice.html) under the hood.
Contrary to `[`, slicing will work along the rows for matrices and data frames.

For these and a few types, length and size are different: the length refers to the size of the internal data format, whereas the size is the number of elements.
The [`vec_size()`](https://vctrs.r-lib.org/reference/vec_size.html) function, modeled after `NROW()`, returns the latter:


```r
vec_size(df)
#> [1] 3
length(df)
#> [1] 2

vec_size(m)
#> [1] 2
length(m)
#> [1] 6
```

For your own code, it is almost always safer to use `vec_size()` instead of `length()`.
Use `ncol()` to count the columns in a data frame.


## Sturdy recycling

We always recycled only vectors of size one in `tibble()` and `as_tibble()`.
This now also applies to subassignment.
We believe that most of the time this is an unintended error.
Please use an explicit `rep()` if you really need to create a column that consists of multiple repetitions of a vector.


```r
x <- tibble(a = 1:4)
x$a <- 1:2
#> Error: Assigned data `1:2` must be compatible with existing data.
#> x Existing data has 4 rows.
#> x Assigned data has 2 rows.
#> ℹ Only vectors of size 1 are recycled.
x$a <- rep(1:2, 2)
x
#> # A tibble: 4 x 1
#>       a
#>   <int>
#> 1     1
#> 2     2
#> 3     1
#> 4     2
```

Related errors may also appear when applying a pattern that works with regular data frames:


```r
x <- data.frame(a = 1, b = 2)
x[1, ] <- c(a = 3, b = 4)
x
#>   a b
#> 1 3 4

x <- tibble(a = 1, b = 2)
x[1, ] <- c(a = 3, b = 4)
#> Error: Assigned data `c(a = 3, b = 4)` must be compatible with row subscript `1`.
#> x 1 row must be assigned.
#> x Assigned data has 2 rows.
```

This is because all vectors on the right-hand side are treated as columnar data.
Convert to a list to treat the input as row data:


```r
x[1, ] <- list(a = 3, b = 4)
x
#> # A tibble: 1 x 2
#>       a     b
#>   <dbl> <dbl>
#> 1     3     4
```

The ambiguity between a row vector and a column vector also affects the `as_tibble()` function.
For this reason, it is now superseded for atomic and list inputs.
In new code, use the new [`as_tibble_row()` and `as_tibble_col()`](https://tibble.tidyverse.org/reference/as_tibble.html) functions to clarify intent.


```r
as_tibble_row(c(a = 3, b = 4))
#> # A tibble: 1 x 2
#>       a     b
#>   <dbl> <dbl>
#> 1     3     4
as_tibble_col(c(a = 3, b = 4))
#> # A tibble: 2 x 1
#>   value
#>   <dbl>
#> 1     3
#> 2     4
```


## Acknowledgments

Due to the nature of the changes, about 60 CRAN packages were failing with our release candidate.
Many thanks to the maintainers of downstream packages who were very helpful in making this upgrade a smooth experience.

Thanks to the following contributors who sent issues, pull requests, and comments since tibble 2.1.3:

[&#x0040;adamdsmith](https://github.com/adamdsmith), [&#x0040;alankjackson](https://github.com/alankjackson), [&#x0040;anabbott](https://github.com/anabbott), [&#x0040;batpigandme](https://github.com/batpigandme), [&#x0040;billdenney](https://github.com/billdenney), [&#x0040;borisleto](https://github.com/borisleto), [&#x0040;Breza](https://github.com/Breza), [&#x0040;Cervangirard](https://github.com/Cervangirard), [&#x0040;courtiol](https://github.com/courtiol), [&#x0040;dan-reznik](https://github.com/dan-reznik), [&#x0040;daviddalpiaz](https://github.com/daviddalpiaz), [&#x0040;DavisVaughan](https://github.com/DavisVaughan), [&#x0040;elinw](https://github.com/elinw), [&#x0040;EmilHvitfeldt](https://github.com/EmilHvitfeldt), [&#x0040;eran3006](https://github.com/eran3006), [&#x0040;frederikziebell](https://github.com/frederikziebell), [&#x0040;gavinsimpson](https://github.com/gavinsimpson), [&#x0040;gdequeiroz](https://github.com/gdequeiroz), [&#x0040;guiastrennec](https://github.com/guiastrennec), [&#x0040;hadley](https://github.com/hadley), [&#x0040;HashRocketSyntax](https://github.com/HashRocketSyntax), [&#x0040;hope-data-science](https://github.com/hope-data-science), [&#x0040;jennybc](https://github.com/jennybc), [&#x0040;jmgirard](https://github.com/jmgirard), [&#x0040;kevinwolz](https://github.com/kevinwolz), [&#x0040;kieranjmartin](https://github.com/kieranjmartin), [&#x0040;lionel-](https://github.com/lionel-), [&#x0040;LudvigOlsen](https://github.com/LudvigOlsen), [&#x0040;mabafaba](https://github.com/mabafaba), [&#x0040;matteodefelice](https://github.com/matteodefelice), [&#x0040;MatthieuStigler](https://github.com/MatthieuStigler), [&#x0040;md0u80c9](https://github.com/md0u80c9), [&#x0040;michaelquinn32](https://github.com/michaelquinn32), [&#x0040;mitchelloharawild](https://github.com/mitchelloharawild), [&#x0040;moodymudskipper](https://github.com/moodymudskipper), [&#x0040;msberends](https://github.com/msberends), [&#x0040;pavopax](https://github.com/pavopax), [&#x0040;rbjanis](https://github.com/rbjanis), [&#x0040;romainfrancois](https://github.com/romainfrancois), [&#x0040;rvg02010](https://github.com/rvg02010), [&#x0040;sfirke](https://github.com/sfirke), [&#x0040;Shians](https://github.com/Shians), [&#x0040;ShixiangWang](https://github.com/ShixiangWang), [&#x0040;stephensrmmartin](https://github.com/stephensrmmartin), [&#x0040;stufield](https://github.com/stufield), [&#x0040;Tazinho](https://github.com/Tazinho), [&#x0040;TimTeaFan](https://github.com/TimTeaFan), [&#x0040;tyluRp](https://github.com/tyluRp), [&#x0040;wgrundlingh](https://github.com/wgrundlingh), [&#x0040;xvrdm](https://github.com/xvrdm), [&#x0040;yannabraham](https://github.com/yannabraham), [&#x0040;ycroissant](https://github.com/ycroissant), [&#x0040;yogat3ch](https://github.com/yogat3ch), and [&#x0040;yutannihilation](https://github.com/yutannihilation).

Your contributions are very valuable and important to us!
