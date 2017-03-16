<!-- 
.. title: DyND Callables: Speed and Flexibility
.. slug: dynd-callables-speed-and-flexibility
.. date: 2016-03-15 13:22:33 UTC-06:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

(This is a post I wrote for the [Continuum Developer Blog](https://www.continuum.io/blog/developer-blog). You can see the original [here](https://www.continuum.io/blog/developer-blog/dynd-callables-speed-and-flexibility))

## Introduction

We've been working hard to improve DyND in a wide variety of ways over the past few months.
While there is still a lot of churn in our codebase, now is a good time to show a few basic examples of the great functionality that's already there.
The library is available on GitHub at [https://github.com/libdynd/libdynd](https://github.com/libdynd/libdynd).

Today I want to focus on DyND's callable objects.
Much of the code in this post is still experimental and subject to change.
Keep that in mind when considering where to use it.

All the examples here will be in C++14 unless otherwise noted.
The build configuration should be set up to indicate that C++14, the DyND headers, and the DyND shared libraries should be used.
Output from a line that prints something will be shown directly in the source files as comments.
DyND also has a Python interface, so several examples in Python will also be included.

## Getting Started

DyND's callables are, at the most basic level, functions that operate on arrays.
At the very lowest level, a callable can access all of the data, type-defined metadata (e.g. stride information), and metadata for the arrays passed to it as arguments.
This makes it possible to use callables for functionality like multiple dispatch, views based on stride manipulation, reductions, and broadcasting.
The simplest case is using a callable to wrap a non-broadcasting non-dispatched function call so that it can be used on scalar arrays.

Here's an example of how to do that:

[//]: # ({% raw %})
```C++
#include <iostream>
// Main header for DyND arrays:
#include <dynd/array.hpp>
// Main callable object header:
#include <dynd/callable.hpp>

using namespace dynd;

// Write a function to turn into a DyND callable.
double f(double a, double b) {
  return a * (a - b);
}

// Make the callable.
nd::callable f_callable = nd::callable(f);

// Main function to show how this works:
int main() {
  // Initialize two arrays containing scalar values.
  nd::array a = 1.;
  nd::array b = 2.;
  
  // Print the dynamic type signature of the callable f_callable.
  std::cout << f_callable << std::endl;
  // <callable <(float64, float64) -> float64> at 000001879424CF60>
  
  // Call the callable and print its output.
  std::cout << f_callable(a, b) << std::endl;
  //array(-1,
  //      type="float64")
}

```
[//]: # ({% endraw %})

The constructor for `dynd::nd::callable` does most of the work here.
Using some interesting templating mechanisms internally, it is able to infer the argument types and return type for the function, select the corresponding DyND types, and form a DyND type that represents an analogous function call.
The result is a callable object that wraps a pointer to the function `f` and knows all of the type information about the pointer it is wrapping.
This callable can only be used with input arrays that have types that match the types for the original function's arguments.

The extra type information contained in this callable is `"(float64, float64) -> float64"`, as can be seen when the callable is printed.
The syntax here comes from the [datashape](http://datashape.readthedocs.org/en/latest/) data description system—the same type system used by [Blaze](http://blaze.readthedocs.org/en/latest/index.html), [Odo](http://odo.readthedocs.org/en/latest/), and several other libraries.

One key thing to notice here is that the callable created now does its type checking dynamically rather than at compile time.
DyND has its own system of types that is used to represent data and the functions that operate on it at runtime.
While this does have some runtime cost, dynamic type checking removes the requirement that a C++ compiler verify the types for every operation.
The dynamic nature of the DyND type system makes it possible to write code that operates in a generic way on both builtin and user-defined types in both static and dynamic languages.
I'll leave discussion of the finer details of the DyND type system for another day though.

## Broadcasting

DyND has other functions that make it possible to add additional semantics to a callable.
These are higher-order functions (functions that operate on other functions), and they are used on existing callables rather than function pointers.
The types for these functions are patterns that can be matched against a variety of different argument types.

Things like array broadcasting, reductions, and multiple dispatch are all currently available.
In the case of broadcasting and reductions, the new callable calls the wrapped function many times and handles the desired iteration structure over the arrays itself.
In the case of multiple dispatch, different implementations of a function can be called based on the types of the inputs.
DyND's multiple dispatch semantics are currently under revision, so I'll just show broadcasting and reductions here.

DyND provides broadcasting through the function `dynd::nd::functional::elwise`.
It follows broadcasting semantics similar to those followed by NumPy's generalized universal functions—though it is, in many ways, more general.
The following example shows how to use `elwise` to create a callable that follows broadcasting semantics:

[//]: # ({% raw %})
```C++
// Include <cmath> to get std::exp.
#include <cmath>
#include <iostream>

#include <dynd/array.hpp>
#include <dynd/callable.hpp>
// Header containing dynd::nd::functional::elwise.
#include <dynd/func/elwise.hpp>

using namespace dynd;

double myfunc_core(double a, double b) {
  return a * (a - b);
}

nd::callable myfunc = nd::functional::elwise(nd::callable(myfunc_core));

int main() {
  // Initialize some arrays to demonstrate broadcasting semantics.
  // Use brace initialization from C++11.
  nd::array a{{1., 2.}, {3., 4.}};
  nd::array b{5., 6.};
  // Create an additional array with a ragged second dimension as well.
  nd::array c{{9., 10.}, {11.}};

  // Print the dynamic type signature of the callable.
  std::cout << myfunc << std::endl;
  // <callable <(Dims... * float64, Dims... * float64) -> Dims... * float64>
  //  at 000001C223FC5BE0>

  // Call the callable and print its output.
  // Broadcast along the rows of a.
  std::cout << myfunc(a, b) << std::endl;
  // array([[-4, -8], [-6, -8]],
  //       type="2 * 2 * float64")

  // Broadcast the second row of c across the second row of a.
  std::cout << myfunc(a, c) << std::endl;
  // array([[ -8, -16], [-24, -28]],
  //       type="2 * 2 * float64")

}

```
[//]: # ({% endraw %})

A similar function can be constructed in Python using DyND's Python bindings and Python 3's function type annotations.
If Numba is installed, it is used to get JIT-compiled code that has performance relatively close to the speed of the code generated by the C++ compiler.

```Python
from dynd import nd, ndt

@nd.functional.elwise
def myfunc(a: ndt.float64, b: ndt.float64) -> ndt.float64:
    return a * (a - b)
```

## Reductions

Reductions are formed from functions that take two inputs and produce a single output.
Examples of reductions include taking the sum, max, min, and product of the items in an array.
Here we'll work with a reduction that takes the maximum of the absolute values of the items in an array.
In DyND this can be implemented by using `nd::functional::reduction` on a callable that takes two floating point inputs and returns the maximum of their absolute values.
Here's an example:

[//]: # ({% raw %})
```C++
// Include <algorithm> to get std::max.
#include <algorithm>
// Include <cmath> to get std::abs.
#include <cmath>
#include <iostream>

#include <dynd/array.hpp>
#include <dynd/callable.hpp>
// Header containing dynd::nd::functional::reduction.
#include <dynd/func/reduction.hpp>

using namespace dynd;

// Wrap the function as a callable.
// Then use dynd::nd::functional::reduction to make a reduction from it.
// This time just wrap a C++ lambda function rather than a pointer
// to a different function.
nd::callable inf_norm = nd::functional::reduction(nd::callable(
        [](double a, double b) { return std::max(std::abs(a), std::abs(b));}));

// Demonstrate the reduction working along both axes simultaneously.
int main() {
  nd::array a{{1., 2.}, {3., 4.}};
  
  // Take the maximum absolute value over the whole array.
  std::cout << inf_norm(a) << std::endl;
  // array(4,
  //       type="float64")
}

```
[//]: # ({% endraw %})

Again, in Python, it is relatively easy to create a similar callable.

```Python
from dynd import nd, ndt

@nd.functional.reduction
def inf_norm(a: ndt.float64, b: ndt.float64) -> ndt.float64:
    return max(abs(a), abs(b))
```

The type for the reduction callable `inf_norm` is a bit longer.
It is `(Dims... * float64, axes: ?Fixed * int32, identity: ?float64, keepdims: ?bool) -> Dims... * float64`.
This signature represents a callable that accepts a single input array and has several optional keyword arguments.
In Python, passing keyword arguments to callables works the same as it would for any other function.
Currently, in C++, initializer lists mapping strings to values are used since the names of the keyword arguments are not necessarily known at compile time.

## Exporting Callables to Python

The fact that DyND callables are C++ objects with a single C++ type makes it easy to wrap them for use in Python.
This is done using the wrappers for the callable class already built in to DyND's Python bindings.
Using DyND in Cython merits a discussion of its own, so I'll only include a minimal example here.

This Cython code in particular is still using experimental interfaces.
The import structure and function names here are very likely to change.

The first thing needed is a header that creates the desired callable.
Since this will only be included once in a single Cython based module, additional guards to make sure the header is only applied once are not needed.

```C++
// inf_norm_reduction.hpp
#include <algorithm>
#include <cmath>

#include <dynd/array.hpp>
#include <dynd/callable.hpp>
#include <dynd/func/reduction.hpp>

static dynd::nd::callable inf_norm =
    dynd::nd::functional::reduction(dynd::nd::callable(
        [](double a, double b) { return std::max(std::abs(a), std::abs(b));}));
```

The callable can now be exposed to Python through Cython.
Some work still needs to be done in DyND's Python bindings to simplify the system-specific configuration for linking extensions like this to the DyND libraries.
For simplicity, I'll just show the commented Cython distutils directives that can be used to build this file on 64 bit Windows with a libdynd built and installed from source in the default location.
Similar configurations can be put together for other systems.

```Cython
# py_inf_norm.pyx
# distutils: include_dirs = "c:/Program Files/libdynd/include"
# distutils: library_dirs = "c:/Program Files/libdynd/lib"
# distutils: libraries = ["libdynd", "libdyndt"]

from dynd import nd, ndt

from dynd.cpp.callable cimport callable as cpp_callable
from dynd.nd.callable cimport dynd_nd_callable_from_cpp

cdef extern from "inf_norm_reduction.hpp" nogil:
    # Have Cython call this "cpp_inf_norm", but use "inf_norm" in
    # the generated C++ source.
    cpp_callable inf_norm

py_inf_norm = dynd_nd_callable_from_cpp(inf_norm)
```

To build the extension I used the following setup file and ran it with the command `python setup.py build_ext --inplace`.

```Python
# setup.py
# This is a fairly standard setup script for a minimal Cython module.
from distutils.core import setup
from Cython.Build import cythonize
setup(ext_modules=cythonize("py_inf_norm.pyx", language='c++'))
```

## A Short Benchmark

`elwise` and `reduction` are not heavily optimized yet, but, using IPython's `timeit` magic, it's clear that DyND is already doing well:

```Python
In [1]: import numpy as np

In [2]: from py_inf_norm import py_inf_norm

In [3]: a = np.random.rand(10000, 10000)

In [4]: %timeit py_inf_norm(a)
1 loop, best of 3: 231 ms per loop

In [5]: %timeit np.linalg.norm(a.ravel(), ord=np.inf)
1 loop, best of 3: 393 ms per loop
```

These are just a few examples of the myriad of things you can do with DyND's callables.
For more information take a look at our [github repo](https://github.com/libdynd/libdynd) as well as [libdynd.org](libdynd.org).
We'll be adding a lot more functionality, documentation, and examples in the coming months.
