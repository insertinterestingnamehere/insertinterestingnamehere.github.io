<!-- 
.. title: Structure Plans and More Examples
.. slug: structure-plans-and-more-examples
.. date: 2015-06-23 15:15:07 UTC-06:00
.. tags: 
.. category: DyND 
.. link: 
.. description: 
.. type: text
-->

Despite a few setbacks, these past two weeks have been ones of progress.
I was able to work with my mentors, Mark and Irwin, to come up with a plan for how to restructure the C++ namespaces in dynd-python.
We opted to separate the namespaces for the Python interoperability functions and the namespaces for the primary DyND C++ functionality.
This will make it so that Cython wrapper types for specific C++ types can have the same name without causing any sort of naming conflict.
Users that want to use both types within the same file can use the different namespaces to separate them.
Within Cython this will be even easier since types can be aliased on import the same way Python objects can.
I finished the name separation work shortly after my last blog post.

We also developed a plan with regards to how to structure the pxd files so that they can be easily used in third party extension modules.
I'm working to separate the existing Cython pxd files in the project so that there is one for each header.
That will make it so that the structure of the headers is clear and it will be easy to add new declarations.
It will also make it so that projects that do not wish to include swathes of project headers will be able to avoid that when they only want to include specific things.
On the other hand, for users that want to throw something together quickly, I'll also put together another set of pxd files that use Cython's relative cimports to mimic the namespace structure currently in the DyND library.
I am in the process of that restructuring, and hope to be able to get it done soon.

With regards to dynamically loading C++ classes via Cython, for now, we're going to continue including the C++ project headers and linking against the original DyND library.
I suspect that there is a way of dynamically loading C++ classes from other Python modules without having to link against the original module.
However, I haven't figured it out yet.
It appears to be a cool feature that could be added later, but brings relatively little added benefit other than a slightly simplified API.
Cython certainly does not support anything like that currently and getting it to do that would require significant modifications.

On the other hand, in Cython, it is relatively easy to export functions and variables in a way that does not require client extension modules to link against the original library.
For now I'm going to employ that approach so that, when all is said and done, as much of the existing library as possible can be loaded via Cython's cimport machinery.

I have spent some time putting together a basic example of working with DyND's C++ objects within Cython, but I have run across some unresolved issues with current project structure.
I expect that we'll be able to get those resolved and I'll be able to post a working example soon.

During the past two weeks I also had the opportunity to learn how DyND's arrfuncs work.
Arrfuncs operate on DyND arrays the same way that gufuncs operate on NumPy arrays.
I found that arrfuncs are surprisingly easy to both construct and use.
Here's a simple program that shows some arrfuncs in action:

[//]: # ({% raw %})
```C++
#include <iostream>
#include <dynd/array.hpp>
#include <dynd/func/arrfunc.hpp>
#include <dynd/func/apply.hpp>
#include <dynd/func/elwise.hpp>

using namespace dynd;

namespace {
// Functions to use to create arrfuncs.
double f(double x) { return x * x; }
double g(double x, double y) { return x * x + y * y; }
// A print function that mimics Python's,
// but only accepts one argument.
template <typename T> void print(T thing) { std::cout << thing << std::endl; }
}

int main() {
  // First make versions of f and g that operate on
  // DyND array types rather than native C++ types.
  nd::arrfunc square_0 = nd::functional::apply(&f);
  nd::arrfunc norm2_0 = nd::functional::apply(&g);
  // Now make versions of f and g that operate on DyND arrays
  // and follow NumPy-like broadcasting semantics.
  nd::arrfunc square = nd::functional::elwise(square_0);
  nd::arrfunc norm2 = nd::functional::elwise(norm2_0);
  // Try applying these functions to DyND scalars.
  nd::array a = 2.;
  nd::array b = 3.;
  print(square_0(a));
  print(norm2_0(a, b));
  print(square(a));
  print(norm2(a, b));
  // Now apply these functions to DyND arrays.
  a = {1., 2., 3., 2.};
  b = {3., 2., 1., 0.};
  print(square(a));
  print(norm2(a, b));
  // Run a two-dimensional example.
  a = {{1., 2.}, {2., 3.}, {3., 4.}};
  b = {{2., 1.}, {3., 3.}, {2., 1.}};
  print(square(a));
  print(norm2(a, b));
  // Broadcast the function with two arguments along the rows of a.
  b = {1., 2.};
  print(norm2(a, b));
  // Broadcast a scalar with an array.
  a = {{1., 2., 3.}, {2., 3., 4.}};
  b = 2.;
  print(norm2(a, b));
  print(norm2(b, a));
  // Broadcast a row and column together.
  a = {{3.}, {2.}, {1.}};
  b = {{2., 1.}};
  print(norm2(a, b));
}
```
[//]: # ({% endraw %})

I compiled this C++ file on Windows with a recent MinGW-w64 g++ using the command
```shell
g++ -O3 -I"c:/Program Files (x86)/libdynd/include" -std=c++11 ufunc.cpp -L"c:/Program Files (x86)/libdynd/lib/static" -ldynd -o ufunc.exe
```

Again, I was very impressed with how naturally I was able to create a C++ arrfunc that followed all the expected broadcasting rules.
In another few weeks I expect to be able to do this sort of thing just as naturally in Cython.
