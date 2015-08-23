<!-- 
.. title: GSOC Concluding Thoughts
.. slug: gsoc-concluding-thoughts
.. date: 2015-08-22 22:30:52 UTC-06:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

After many long weeks of hunting bugs and reorganizing code, my GSOC has come to an end.
My project turned out significantly different than I had expected, but much of my work still proved to be very valuable.
I originally envisioned being able to use many of DyND's features within Cython in a transparent way.
Much of the work that I did was in fixing bugs and laying the groundwork for making things like that possible.
There are still several bugs to fix and features to add in Cython before that's really possible, but I'm still pleased with the progress I made during my GSOC.
My bugfixes served to simplify some of the difficult problems that arise when wrapping and interfacing with nontrivial C++ code.
My restructuring of the DyND Python bindings, as expected, made it much easier to transition between the C++-level DyND constructs and the Python classes used in its Python bindings.

Here's a minimum working example that works with the latest versions of libdynd and dynd-python (as of this post).

```C++
// myfunc.cpp
#include "dynd/func/callable.hpp"
#include "dynd/func/apply.hpp"
#include "dynd/func/elwise.hpp"

// Write a function to turn into a DyND callable.
double f(double a, double b) {
  return a * (a - b);
}

// Make the callable object.
dynd::nd::callable f_broadcast = dynd::nd::functional::elwise(dynd::nd::functional::apply(&f));
```

```Cython
# custom_callable.pyx
# distutils: extra_compile_args = -std=c++11
# distutils: libraries = dynd

from dynd.cpp.func.callable cimport callable as cpp_callable
from dynd.nd.callable cimport callable
from dynd import nd

# Declare the C++ defined callable object.
cdef extern from "myfunc.cpp" nogil:
    cpp_callable f_broadcast

# Construct a Python object wrapping the DyND callable.
cdef callable py_f = callable()
py_f.v = f_broadcast

# A short demonstration.
def test():
    a = nd.array([1., 2., 3., 2.])
    b = nd.array([3., 2., 1., 0.])
    print py_f(a, b)
```

```Python
# setup.py
from distutils.core import setup
from Cython.Build import cythonize
setup(ext_modules=cythonize("*.pyx", language='c++'))
```

As you can see, it is now very easy for external modules to transition smoothly between using DyND in C++ and providing Python-level functionality.
There's still a lot to be done, but I find the fact that this is now such a simple process really remarkable.
This example uses the strengths of both Cython and Dynd to provide useful and significant functionality, and I'm thrilled it is finally working.
