<!-- 
.. title: First Impressions And Overcoming Compilation Errors
.. slug: first-impressions-and-overcoming-compilation-errors
.. date: 2015-05-21 22:23:11 UTC-06:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

Most of the past month was spent in a mad dash to finish my master's thesis.
Who knew that writing 100 pages of original research in math could take so long?
Either way, I'm finished and quite pleased with the result.
My thesis is written and I have been able to, mostly, move on to other endeavors, like starting the work on my GSOC project, the Cython API for DyND (see the [brief description](https://www.google-melange.com/gsoc/project/details/google/gsoc2015/iandh/5707702298738688) and [full proposal](https://github.com/numfocus/gsoc/blob/master/2015/proposals/ian-henriksen.md)).

I've been able to start familiarizing myself with the package a little more.
My background is primarily in using NumPy and Cython, but it has been really nice to be able to use a NumPy-like data structure in C++ without having to use the Python C API to do everything.
Stay tuned for a few nice snippets of code over the next few days.

Overall, I've been impressed by the amount of functionality that is already present.
A few things like allowing the use of an ellipsis in indexing or providing routines for basic linear algebra operations have yet to be implemented, but, for the most part, the library is easy to use and provides a great deal of useful functionality at both the C++ and Python levels.

Before beginning detailed work on my project, I was first faced with the task of getting everything working on my main development machine.
For a variety of reasons, it uses windows and needs to stay that way.
Undaunted, I ventured forth, and attempted to build libdynd and its corresponding Python bindings from source with my compiler of choice, MinGW-w64.

MinGW-w64 is a relatively new version of MinGW that targets 64 bit windows.
It is not as commonly used as MSVC, or gcc, so it is normal to have to fix a few compile errors when compiling large amounts of C/C++ code.
Python's support for MinGW is shaky as well, but usually things come together nicely with a little effort.

After several long and ugly battles with a variety of compiler errors, I was able to get it all working, though there are still some test failures that still need to be revisited.
The DyND source proved to be fairly easy to work with.
The most difficult errors I came up against came from conflicts between the environment provided by the compiler and some of the type definitions in the Python headers.
I'll document some of the useful information I picked up here.

For those that aren't familiar with it, the C preprocessor is an essential part of writing platform independent code.
Among other things, it lets you write specialized sections of code for pieces that are specific to a given operating system, compiler, or architecture.
For example, to write a section of code that applies only to Windows, you would write something like

```C++
// If the environment variable indicating that you're compiling on Windows is defined
#ifdef _WIN32
  // Platform-specific code goes here.
#endif
```
Similarly, if you want to differentiate between compilers,
```C++
#ifdef _MSC_VER
  // Code for MSVC compiler here.
#elseif __GNUCC__
  // Code for gcc/g++ here.
  // Compilers that implement a GCC-like environment
  // (e.g. ICC on Linux, or Clang on any platform)
  // will also use this case.
#else
  // Code for all other compilers.
#endif
```
Most of the work I did to get everything running involved modifying blocks that look like this to get the right combination.
Frequently, code is defined to do one thing on windows (with the assumption that only MSVC will be used) and another on Linux, even though, for MinGW, many of the Linux configurations are needed for the compiler.
In the past, I've found myself continually searching for what the right ways to detect a given compiler, operating system, or architecture, but it turns out there is a fairly complete [list](http://sourceforge.net/p/predef/wiki/Home/) where the variables indicating most common compilers and operating systems are shown.

The build system CMake also provides ways of detecting things like that.
A few useful examples:

```cmake
if(WIN32)
    # You're compiling for Windows
endif()
if("${CMAKE_SIZEOF_VOID_P}" EQUAL "8")
    # You're compiling for a 64-bit machine.
endif()
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    # You're compiling for gcc.
endif()
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
    # You're compiling for MSVC.
endif()
```

Now, on to some of the tricker bugs.
Hopefully including them here can help others resolve similar errors later on.

It is a less commonly known fact that the order of compiler flags matters if you are linking against different libraries.
(An aside, for those that are new to the command line flags for gcc, there is a nice introduction [here](http://www.rapidtables.com/code/linux/gcc.htm))
As it turns out, the order of the flags for gcc also matters in some less than intuitive cases.
After compiling libdynd, in order to compile a basic working example I had to do `g++ -std=c++11 -fPIC declare_array.cpp -ldynd` rather than `g++ -std=c++11 -ldynd -fPIC declare_array.cpp`.
Running the latter would give a series of linker errors like
```
undefined reference to 'dynd::detail::memory_block_free(dynd::memory_block_data*)'
```
`clang++` accepted both orderings of the flags without complaint.

I also ran into [this bug in Python](http://bugs.python.org/issue11566) where there is an unfortunate name collision between a preprocessor macro defined in Python and a function in the c++11 standard library.
The name collision results in the unusual error
```
error: '::hypot' has not been declared
```
coming from the c++11 standard library headers.
It appears that a [generally known solution](http://stackoverflow.com/a/12124708/1935144) is to include the standard library math header before `Python.h`.
This wasn't a particularly favorable resolution to the issue since the Python headers are supposed to come first, so I opted to define the symbol `_hypot` back again to be `hypot` via a command line argument, i.e. adding the flag `-D_hypot=hypot`.
It worked like a charm.

The last batch of errors came from my forgetting to define the macro `MS_WIN64` on the command line when working with the Python headers.
That macro is used internally to indicate that a Python extension module is being compiled for a 64 bit Python installation on Windows.
In this case, forgetting to include that flag resulted in the error
```
undefined reference to '__imp_Py_InitModule4'
```
at link time.

In trying to fix the various errors from not defining `MS_WIN64`, I also came across errors where `np_intp` (a typedef of `Py_intptr_t`) was not the correct size, resulting in the error
```
error: cannot convert 'long long int*' to 'npy_intp* {aka int*}' in argument passing
```
The fundamental problem is that the logic in the Python headers fails to detect the proper size of a pointer without some sort of extra compile switch.
Passing the command-line argument `-DMS_WIN64` to the compiler when compiling Python extensions was sufficient to address the issue.

With most of the compilation-related issues behind me, I am now continuing my work in putting together several simple examples of how DyND can be used to perform NumPy-like operations at the C++ level.
I'll be posting a few of them soon.

### Update (6/8/2015)
In fixing a few more compilation errors, I found another [useful table](http://nadeausoftware.com/articles/2012/01/c_c_tip_how_use_compiler_predefined_macros_detect_operating_system) of C preprocessor macros that can be used to identify a given operating system.
Among other things, it mentions that any BSD operating system can be detected using something like
```C++
#ifdef __unix__
#include <sys/param.h>
#ifdef BSD
// BSD stuff
#endif
#endif
```
The [table in the GCC manual](https://gcc.gnu.org/onlinedocs/cpp/Standard-Predefined-Macros.html#Standard-Predefined-Macros) also lists which preprocessor macros are required for a compiler implementing the relevant language standards.
