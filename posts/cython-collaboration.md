<!-- 
.. title: Cython Collaboration
.. slug: cython-collaboration
.. date: 2015-08-05 15:55:07 UTC-06:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

I've spent the majority of these last few weeks working on adding new features to Cython.
I just recently finished my work there, though the pull request involving exception handling for overloaded operators is still waiting to be merged.
Overloading the assignment operator should now work in all cases for C++ classes.
Once the exception handling pull request is merged, translating C++ exceptions into Python exceptions should also work perfectly well.
It has been a relief to finally get these bugs taken care of.
There were a variety of features I wanted to add to Cython, but I've run out of time to work on them now, so they will have to wait till later.
The Cython developers were very helpful in getting these bugs fixed.
They've helped find and take care of a few corner cases that I missed in the pull requests that have been merged thus far.
I wasn't able to finish all the features I wanted, but I was able to make a lot of good progress and start building a consensus on what some features should look like further down the road.
I got a lot of good things done, but I'm ready to move back to working on DyND.

One of the major features I added was allowing C++ classes to use an overloaded `operator=` within Cython.
It is all working now and is on-schedule to be released as a part of 0.23.
Handling cascaded assignment proved to be particularly tricky, since Cython mimics the Python semantics in the case of `a = b = c` rather than following the C++ behavior for C++ objects.
With the help of the Cython developers, I was able to get it all working fine.

Making Cython respect the exception handling declarations for overloaded C++ operators used within Cython proved to be a lengthy bit of work.
In order to make it work properly, the type analysis code that determines when an exception handler should be used had to be connected to the C++ generation code that, if an exception handler is used, wraps the given arithmetic expression in a try-catch statement at the C++ level.
This was challenging, not so much because it was hard to see what needed to be changed in the output, but because the needed changes consisted of identifying and making many small changes in many places.
I'm pleased with the result now.
Respecting different exception declarations for different operators requires that Cython generate a separate try-catch block for each operator used.
Forming a try-except statement for each call to an overloaded operator also requires that the result of each subexpression be stored in a temporary variable.
In theory, good removal of temporaries and good branch prediction should make it so that all of these extra temporary variables and try-catch statements have no additional effect on the time needed to evaluate a given expression, but that ideal may be difficult for C++ compilers to reach.
Either way, that was the only way to properly conform to the existing API for overloaded operators that is already documented and used extensively in Cython.
At some future point it would be nice to add a way to let users consolidate different try-except blocks, but that is well outside the scope of my current project.

Thanks to the exception handling logic that I've added, things like the following raise an appropriate Python exception now rather than crash the Python interpreter:

```Cython
cdef extern from "myheader.hpp" nogil:
    cppclass thing:
        thing()
        thing(int)
        operator+(thing &other) except +

def myfunc():
    # Initialize some "thing"s to some values
    cdef thing a = ...
    cdef thing b = ...
    cdef thing c
    # Now do an operation that throws a C++ exception.
    c = a + b
```

One of the key features I was hoping to work through with the Cython developers was finding an appropriate API for assignment to a multidimensional index.
In C++ this is easily implemented as an assignment to an lvalue reference resulting from an overloaded or variadic function call.
The Cython developers were opposed to allowing assignment to function calls, since an interface of that sort is fairly un-pythonic.
That's a sensible opinion, but it leaves the problem of assigning to a multidimensional index unsolved.
The only workable proposal that came from the discussion was one that would allow inline definitions of methods like `__getitem__` and `__setitem__` in the Cython extern declaration of the C++ class.
Though that would provide a Python-like interface for users, I didn't opt to implement it that way for a variety of reasons.
First, when working with a C++ library with Python bindings written in Cython, it forces users to keep track of three interfaces (the C++ interface, the Cython interface built on top of the C++ interface, and the Python interface) instead of two (the C++ and Python interfaces).
Second, it makes library writers write larger amounts of oddly placed boilerplate code that is more difficult to understand and maintain.
Third, it causes possible naming conflicts when working with C++ classes that overload both `operator[]` and `operator()` (this is primarily a theoretical concern, but it is still troubling).
In order to allow C++-like control over the number of indices used (without using `std::tuple` and requiring C++11), these special functions would have to support C++ style function signature overloading as well, which requires that they have entirely different signatures than their corresponding Python variants.
The maintainability of such a feature is also a concern, since it would require a wide variety of changes to the Cython code base, all of which are unlikely to be used frequently in many other projects.
In spite of all these concerns, this is still a workable proposal, but I have avoided implementing it thus far in the hope that something better would surface.
Given how much time I have left to work on this GSOC project, this feature may have to be left for later.

Another important feature I spent some time trying to add was support for non-type template parameters.
I posted two different possible API's: one which specifies only the number of parameters and allows the C++ compiler to verify that all the parameters are the correct type, and another that requires that the types of the parameters be specified at the Cython level so that they can all be checked when Cython generates the C++ source file.
After waiting several days and receiving no response, I started work on the former, primarily because it seemed like the simplest option available.
Adding the additional type declarations to the template parameters would have required finding a way to apply the existing operator overloading logic to template parameters, and I was uncertain of whether or not that would be doable while still mirroring the C++ template semantics.
After I had spent a pair of days implementing most of the more lenient syntax, one of the Cython developers chimed in to support the stricter type checking.
The argument in favor of stricter type checking makes sense since it lets users avoid having to debug the generated C and C++ files, but it would have been nice to hear a second opinion before spending a few days implementing the other syntax.
This is another feature that, given how much time I have left, I'll probably have to leave for later.

While discussing non-type template parameters, we also discussed adding support for variadic templates.
The consensus on the syntax was to allow variadic template arguments to be used by simply adding an ellipsis where the C++ expression `typenames ...` would go.
I also proposed allowing something like `types ...`, and that syntax may be adopted in the end anyway.
I didn't get the time to add that feature, but at least the design ideas are there for someone to use later on.
Currently, due to some incomplete type checking, variadic templated functions are already usable in Cython, but this is really a bug and not a feature.
For example:

```C++
// variadic_print.hpp
#pragma once
#include <iostream>

template<typename T>
void cpp_print(T value){
  std::cout << value << std::endl;
}

template<typename T, typename... Args>
void cpp_print(T value, Args... args){
  std::cout << value << ", ";
  cpp_print(args...);
}
```

```Cython
# cpp_print.pyx
# The extra compile arguments here assume gcc or clang is used.
# distutils: extra_compile_args = ['-std=c++11']

from libcpp.string cimport string

cdef extern from "variadic_print.hpp" nogil:
    void cpp_print(...)

def test():
    cdef:
        int thing1 = 1
        double thing2 = 2
        long long thing3 = 3
        float thing4 = 4
        string thing5 = "5"
    cpp_print(thing1, thing2, thing3, thing4, thing5)
```

This definitely isn't a feature though.
It's just a cleverly exploited bug that hasn't been fixed yet and is certainly not usable in anything more than entertaining examples.

It would also be nice to see things like overloading in-place operations and overloading `operator||` and `operator&&` in Cython at some point, but I haven't done that yet.

All things considered, even though I wasn't able to accomplish all I had hoped in Cython, I was still able to do a lot of useful things.
The existing design philosophy in Cython didn't prove to be completely amenable to exposing a Cython-level API for a relatively advanced C++ library.
The existing features are designed primarily to allow users to write Python-like code that manipulates C and C++ objects.
The features that are focused toward interfacing with external C++ libraries are effective for constructing and exposing Python wrappers for C++ objects, but can often prove to be very unwieldy for exposing Cython-level APIs for C++ libraries.
It's certainly still possible, but it definitely stretches the limits of the tools at hand.

Now, moving forward after my work on Cython, I've shifted toward contributing to libdynd and dynd-python again.
I've worked with my mentors to isolate a mysterious segfault that we were seeing when using clang to build libdynd and dynd-python.
I've also worked on several improvements to the build system.
Everything should be building with MinGW again soon, but, for now, my linux environment is good enough to work on some new features too.
I've also spent some time working on some minimal examples of using the Cython declarations in dynd-python outside the installed Python package.

To help with testing libdynd and dynd-python, I've also overhauled the Travis-CI build to make use of the new container-based infrastructure.
Thus far the builds are running noticeably faster.
These changes should be a great help to all of us in the future as we continue to develop both packages.

Over these last few weeks I expect to spend most of my time making it as easy as possible to use as much of libdynd as possible within Cython.
The infrastructure is already in place for most of the core pieces that need to be taken care of, so I'm feeling optimistic.
