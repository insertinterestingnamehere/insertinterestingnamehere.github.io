<!-- 
.. title: Fixing Cython Bugs
.. slug: fixing-cython-bugs
.. date: 2015-07-13 13:26:32 UTC-06:00
.. tags: 
.. category: DyND
.. link: 
.. description: 
.. type: text
-->

Last week I went to the SciPy conference.
It was great.
I got to meet many of the major contributors to open source, including one of my mentors, Mark Wiebe.
I also got to give a [talk](https://www.youtube.com/watch?v=R4yB-8tB0J0) about some of my previous work on SciPy and see [my wife's talk](https://www.youtube.com/watch?v=Le1GdRFt3E8) about our new applied math curriculum here at BYU.
There were a lot of great presenters at the conference this year and it was good to see how many people are interested in and working on the software in the scientific Python stack.
Above all, there were free t-shirts. Lots of free t-shirts.

Other than that, the past few weeks I've spent a lot of time fixing and following up on a series of Cython bugs.
A lot of my time has been spent reading large chunks of the Cython codebase to properly understand what is going on.
My understanding of what is going on is still very incomplete, but I've managed to get enough of a hold on things to be able to start fixing bugs.
As I currently understand it, the Cython parser goes through a file and constructs a series of nodes representing the abstract syntax tree.
Once the abstract syntax tree is constructed, there is a type resolution pass on all the existing nodes where some of the more generic nodes can specialize themselves as instances of more specific nodes.
That helps localize parts of the code that deal with entirely different cases for the different types.
This type resolution phase is also where all the static types are checked for correctness.
Once types have all been checked and the proper nodes for code generation are in place, another pass is made to generate the correct code for each operation.
Compiler directives are taken into account and different preambles are included conditionally as well.
Most of the code for performing the type analysis pass and the code generation pass on the abstract syntax tree is in the different methods of the classes defined in ``Cython/Compiler/ExprNodes.py`` and ``Cython/Compiler/Nodes.py``.
That's a very rough overview of what is going on, but it has been enough for me to fix a few things.

The biggest bug I've been working on has been the fact that, until now, exception declarations for operators overloaded at the C++ level have not been respected in the generated C++ code.
This means that, when an operation like ``a + b`` ought to throw a C++ error, catch the C++ error, then translate it into a Python error, the only thing that really happens is the whole program crashing due to an uncaught C++ error.
Fixing this has been a little tricky since temporary variables must be allocated and used to store the result of each operation, and many of the overloadable operations are handled separately throughout the Cython code.
In spite of the fact that this is a large and delicate bug to fix, I have managed to get everything working for arithmetic and unary operators.
Indexing and function calls still need more work/testing.

I've also been discussing the best way to do multidimensional indexing with the Cython developers.
In C++, multidimensional indexing of C++ objects can be written easily as "a(1, 2, 3)".
That is all well and good, except that such an expression cannot be used properly as a left-value in assignment in Cython even though that is perfectly valid in C++.
The primary reasoning as far as I can tell is that allowing that would be decidedly un-pythonic.
My personal preference is very strongly toward allowing the C++ syntax when working with C++ objects, but we're still trying to settle on a reasonable solution.

I've also resurrected an existing patch to allow using an overloaded call operator for stack-allocated objects and put in some initial code toward allowing users to declare and use an overloaded ``operator=`` for a C++ class.
There will be more patches coming soon on that.

Once I've fixed a few more of these Cython bugs, I expect to be able to continue my work in refactoring DyND's Python bindings to make the transition between Python, Cython/DyND, and C++/DyND much smoother.
