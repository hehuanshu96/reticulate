R Interface to Python
================

J.J. Allaire — 2017-02-08

Overview
--------

The **reticulate** package provides an R interface to Python modules, classes, and functions. For example, this code imports the Python `os` module and calls some functions within it:

``` r
library(reticulate)
os <- import("os")
os$chdir("tests")
os$getcwd()
```

Functions and other data within Python modules and classes can be accessed via the `$` operator (analogous to the way you would interact with an R list, environment, or reference class).

When calling into Python, R data types are automatically converted to their equivalent Python types. When values are returned from Python to R they are converted back to R types. Types are converted as follows:

| R                      | Python            | Examples                                    |
|------------------------|-------------------|---------------------------------------------|
| Single-element vector  | Scalar            | `1`, `1L`, `TRUE`, `"foo"`                  |
| Multi-element vector   | List              | `c(1.0, 2.0, 3.0)`, `c(1L, 2L, 3L)`         |
| List of multiple types | Tuple             | `list(1L, TRUE, "foo")`                     |
| Named list             | Dict              | `list(a = 1L, b = 2.0)`, `dict(x = x_data)` |
| Matrix/Array           | NumPy ndarray     | `matrix(c(1,2,3,4), nrow = 2, ncol = 2)`    |
| Function               | Python function   | `function(x) x + 1`                         |
| NULL, TRUE, FALSE      | None, True, False | `NULL`, `TRUE`, `FALSE`                     |

If a Python object of a custom class is returned then an R reference to that object is returned. You can call methods and access properties of the object just as if it was an instance of an R reference class.

The **reticulate** package is compatible with all versions of Python &gt;= 2.7 and in addition requires NumPy &gt;= 1.11.

Installation
------------

You can install from GitHub as follows:

``` r
devtools::install_github("rstudio/reticulate")
```

Note that the package includes native C/C++ code so it's installation requires [R Tools](https://cran.r-project.org/bin/windows/Rtools/) on Windows and [Command Line Tools](http://osxdaily.com/2014/02/12/install-command-line-tools-mac-os-x/) on OS X. If the package installation fails because of inability to compile then install the appropriate tools for your platform based on the links above and try again.

### Locating Python

If the version of Python you want to use is located on the system `PATH` then it will be automatically discovered (via `Sys.which`) and used.

Alternatively, you can use one of the following functions to specify alternate versions of Python:

| Function        | Description                                           |
|-----------------|-------------------------------------------------------|
| use\_python     | Specify the path a specific Python binary.            |
| use\_virtualenv | Specify the directory containing a Python virtualenv. |
| use\_condaenv   | Specify the name of a Conda environment.              |

For example:

``` r
library(reticulate)
use_python("/usr/local/bin/python")
use_virtualenv("~/myenv")
use_condaenv("myenv")
```

Note that **reticulate** requires NumPy &gt;= 1.11 so versions of Python that don't satisfy this requirement will not be used.

Also note that the `use` functions are by default considered only hints as to where to find Python (i.e. they don't produce errors if the specified version doesn't exist). You can add the `required` parameter to ensure that the specified version of Python actually exists:

``` r
use_virtualenv("~/myenv", required = TRUE)
```

The order in which versions of Python will be discovered and used is as follows:

1.  If specified, at the locations referenced by calls to `use_python`, `use_virtualenv`, and `use_condaenv`.

2.  If specified, at the location referenced by the `RETICULATE_PYTHON` environment variable.

3.  At the location of the Python binary discovered on the system `PATH` (via the `Sys.which` function).

4.  At other customary locations for Python including `/usr/local/bin/python`, `/opt/local/bin/python`, etc.

The scanning for and binding to a version of Python typically occurs at the time of the first call to `import` within an R session. As a result, priority will be given to versions of Python that include the module specified within the call to `import` (i.e. versions that don't include it will be skipped).

You can use the `py_config` function to query for information about the specific version of Python in use as well as a list of other Python versions discovered on the system:

``` r
py_config()
```

Importing Modules
-----------------

The `import` function can be used to import any Python module. For example:

``` r
difflib <- import("difflib")
difflib$ndiff(foo, bar)

filecmp <- import("filecmp")
filecmp$cmp(dir1, dir2)
```

There are some special module names you should be aware of: `"__main__"` gives you access to the main module where code is executed by default; and `"__builtin__"` gives you access to various built in Python functions. For example:

``` r
main <- import("__main__")

py <- import("__builtin__")
py$print('foo')
```

The `"__main__"` module is generally useful if you have executed Python code from a file or string and want to get access to it's results (see the section below for more details).

Executing Code
--------------

You can execute Python code within the main module using the `py_run_file` and `py_run_string` functions. These functions both return a reference to the main Python module so you can access the results of their execution. For example:

``` r
py_run_file("script.py")

main <- py_run_string("x = 10")
main$x
```

Lists, Tuples, and Dictionaries
-------------------------------

The automatic conversion of R types to Python types works well in most cases, but occasionally you will need to be more explicit on the R side to provide Python the type it expects.

For example, if a Python API requires a list and you pass a single element R vector it will be converted to a Python scalar. To overcome this simply use the R `list` function explicitly:

``` r
foo$bar(indexes = list(42L))
```

Similarly, a Python API might require a `tuple` rather than a list. In that case you can use the `tuple` function:

``` r
tuple("a", "b", "c")
```

R named lists are converted to Python dictionaries however you can also explicitly create a Python dictionary using the `dict` function:

``` r
dict(foo = "bar", index = 42L)
```

This might be useful if you need to pass a dictionary that uses a more complex object (as opposed to a string) as it's key.

With Contexts
-------------

The R `with` generic function can be used to interact with Python context manager objects (in Python you use the `with` keyword to do the same). For example:

``` r
py <- import("__builtin__")
with(py$open("output.txt", "w") %as% file, {
  file$write("Hello, there!")
})
```

This example opens a file and ensures that it is automatically closed at the end of the with block. Note the use of the `%as%` operator to alias the object created by the context manager.

Iterators
---------

If a Python API returns an [iterator or generator](http://anandology.com/python-practice-book/iterators.html) you can interact with it using the `iterate` function. The `iterate` function can be used to apply an R function to each item yielded by the iterator:

``` r
iterate(iter, print)
```

If you don't pass a function to `iterate` the results will be collected into an R vector:

``` r
results <- iterate(iter)
```

Advanced Functions
------------------

There are several more advanced functions available that are useful principally when creating high level R interfaces for Python libraries:

<table style="width:100%;">
<colgroup>
<col width="20%" />
<col width="79%" />
</colgroup>
<thead>
<tr class="header">
<th>Function</th>
<th>Description</th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>py_config</td>
<td>Get information on the location and version of Python in use.</td>
</tr>
<tr class="even">
<td>py_available</td>
<td>Check whether a Python interface is available on this system.</td>
</tr>
<tr class="odd">
<td>py_has_attr</td>
<td>Check if an object has a specified attribute.</td>
</tr>
<tr class="even">
<td>py_get_attr</td>
<td>Get an attribute of a Python object.</td>
</tr>
<tr class="odd">
<td>py_call</td>
<td>Call a Python callable object with the specified arguments.</td>
</tr>
<tr class="even">
<td>py_capture_stdout</td>
<td>Capture all standard output for the specified expression and return it as an R character vector.</td>
</tr>
<tr class="odd">
<td>py_suppress_warnings</td>
<td>Execute the specified expression, suppressing the display Python warnings.</td>
</tr>
<tr class="even">
<td>py_str</td>
<td>Get the string representation of Python object.</td>
</tr>
<tr class="odd">
<td>py_xptr_str</td>
<td>Evaluate an expression that prints a string with a check for a null externalptr.</td>
</tr>
<tr class="even">
<td>py_is_null_xptr</td>
<td>Check whether a Python object is a null externalptr.</td>
</tr>
</tbody>
</table>
