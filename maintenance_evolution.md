# Package evolution - changing stuff in your package {#evolution}

<div class="summaryblock">
<p>This chapter presents our guidance for changing stuff in your package: changing parameter names, changing function names, deprecating functions.</p>
<p><em>This chapter was initially contributed as a tech note on rOpenSci website by <a href="https://github.com/sckott">Scott Chamberlain</a>; you can read the original version <a href="https://ropensci.org/technotes/2017/01/05/package-evolution/">here</a>.</em></p>
</div>

## Philosophy of changes

Everyone's free to have their own opinion about how freely parameters/functions/etc. are changed in a library - rules about package changes are not enforced by CRAN or otherwise. Generally, as a library gets more mature, changes to user facing methods (i.e., exported functions in an R package) should become very rare. Libraries that are dependencies of many other libraries are likely to be more careful about changes, and should be. 

## Parameters: changing parameter names

Sometimes parameter names must be changed for clarity, or some other reason. 

A possible approach is to catch all parameters passed in to the function and check against some list of parameters, and stop or warn with a meaningful message.

```r
foo_bar <- function(x, y) {
    calls <- names(sapply(match.call(), deparse))[-1]
    if(any("x" %in% calls)) {
        stop("use 'y' instead of 'x'")
    }
    y^2
}

foo_bar(x = 5)
#> Error in foo_bar(x = 5) : use 'y' instead of 'x' 
```

Or instead of stopping with error, you could check for use of `x` parameter and set it to `y` internally. 

```r
foo_bar <- function(x, y) {
    calls <- names(sapply(match.call(), deparse))[-1]
    if(any("x" %in% calls)) {
        y <- x
    }
    y^2
}

foo_bar(x = 5)
#> 25
```

Be aware of the parameter `...`. If your function has `...`, and you have already removed a parameter (lets call it `z`), a user may have older code that uses `z`. When they pass in `z`, it's not a parameter in the function definition, and will likely be silently ignored -- not what you want. So, do make sure to always check for removed parameters moving forward since you can't force users to upgrade.

## Functions: changing function names

If you must change a function name, do it gradually, as with any other change in your package. 

Let's say you have a function `foo`.

```r
foo <- function(x) x + 1
```

However, you want to change the function name to `bar`. 

Instead of simply changing the function name and `foo` no longer existing straight away, in the first version of the package where `bar` appears, make an alias like:

```r
#' foo - add 1 to an input
#' @export
foo <- function(x) x + 1

#' @export
#' @rdname foo
bar <- foo
```

With the above solution, the user can use either `foo()` or `bar()` -- either will do the same thing, as they are the same function.

It's also useful to have a message but then you'll only want to throw that message when they use the old function name, e.g.,

```r
#' foo - add 1 to an input
#' @export
foo <- function(x) {
    if (as.character(match.call()[[1]]) == "foo") {
        warning("please use bar() instead of foo()", call. = FALSE)
    }
    x + 1
}

#' @export
#' @rdname foo
bar <- foo
```

After users have used the package version for a while (with both `foo` and `bar`), in the next version you can remove the old function name (`foo`), and only have `bar`.

```r
#' bar - add 1 to an input
#' @export
bar <- function(x) x + 1
```

## Functions: deprecate & defunct

To remove a function from a package (let's say your package name is `helloworld`), you can use the following protocol:

* Mark the function as deprecated in package version `x` (e.g., `v0.2.0`)

In the function itself, use `.Deprecated()` like:

```r
foo <- function() {
    .Deprecated(msg = "'foo' will be removed in the next version")
}
```

There's options in `.Deprecated` for specifying a new function name, as well as a new package name, which makes sense when moving functions into different packages.

The message that's given by `.Deprecated` is a warning, so can be suppressed by users with `suppressWarnings()` if desired.

Make a man page for deprecated functions like:

```r
#' Deprecated functions in helloworld
#' 
#' These functions still work but will be removed (defunct) in the next version.
#' 
#' \itemize{
#'  \item \code{\link{foo}}: This function is deprecated, and will
#'  be removed in the next version of this package.
#' }
#' 
#' @name helloworld-deprecated
NULL
```

This creates a man page that users can access like ``?`helloworld-deprecated` `` and they'll see in the documentation index. Add any functions to this page as needed, and take away as a function moves to defunct (see below).

* In the next version (`v0.3.0`) you can make the function defunct (that is, completely gone from the package, except for a man page with a note about it).

In the function itself, use `.Defunct()` like:

```r
foo <- function() {
    .Defunct(msg = "'foo' has been removed from this package")
}
```

Note that the message in `.Defunct` is an error so that the function stops whereas `.Deprecated` uses a warning that let the function proceed.

In addition, it's good to add `...` to all defunct functions so that if users pass in any parameters they'll get the same defunct message instead of a `unused argument` message, so like:

```r
foo <- function(...) {
    .Defunct(msg = "'foo' has been removed from this package")
}
```

Without `...` gives:

```r
foo(x = 5)
#> Error in foo(x = 5) : unused argument (x = 5)
```

And with `...` gives:

```r
foo(x = 5)
#> Error: 'foo' has been removed from this package
```

Make a man page for defunct functions like:

```r
#' Defunct functions in helloworld
#' 
#' These functions are gone, no longer available.
#' 
#' \itemize{
#'  \item \code{\link{foo}}: This function is defunct.
#' }
#' 
#' @name helloworld-defunct
NULL
```

This creates a man page that users can access like ``?`helloworld-defunct` `` and they'll see in the documentation index. Add any functions to this page as needed. You'll likely want to keep this man page indefinitely.
