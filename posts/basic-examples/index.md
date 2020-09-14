<!-- 
.. title: Basic Examples
.. slug: basic-examples
.. date: 2015-06-08 10:26:00 UTC-06:00
.. tags: 
.. category: DyND 
.. link: 
.. description: 
.. type: text
-->

The past two weeks have been an adventure.
I've spent a great deal of time working to find a reasonable way to refactor the DyND Python bindings in a way that works well for exporting all the existing Python/C++ compatibility functions at both the Cython and C++ level.
After some discussion with my mentors, Mark and Irwin, I was able to get to what seems to be a workable model.
I'm in the process of doing all the needed refactoring to put those changes into place.
I'll post more details once I've finished the main structural changes to the Python bindings.

For now, I'd like to show a few basic examples of how to use DyND.
The examples here showcase some of the similarities with NumPy as well as the simplicity of the notation.
I haven't done any sort of performance comparison.
The implementations I've put together here using DyND work using array slicing and array arithmetic, so the cost of dispatching for types is still present in each array arithmetic operation.
More efficient implementations could be created by coding a version of an algorithm that operates directly on the buffers of the arrays given.
More efficient and general implementations could be constructed using DyND's arrfuncs, which are similar in purpose to NumPy's gufuncs, but I'm still figuring out how they work, so I'll have to include examples of how to do that later on.

For my examples here, I'll focus on three different algorithms for polynomial evaluation.
For simplicity, I'll assume the coefficients are passed as an array and the parameter value at which to evaluate the polynomial(s) is a single double precision value.
Each algorithm will be presented in Python (using NumPy), in C++ (using DyND), and in Fortran.
Each file will either compile to an executable (C++ and Fortran) or be run-able as a script (Python).
The NumPy and DyND implementations will be roughly equivalent and will both operate on arbitrary dimensional arrays, operating on the first axis of the array and broadcasting along all trailing axes.
The NumPy and DyND versions will also work with relatively generic types (either floating point or complex).
The Fortran versions will apply only to 1-dimensional arrays and will only operate on arrays of double precision real coefficients.

Regarding the notation, I found it very easy to use DyND's `irange` object with its overloaded comparison operators very easy to use.
It is used to specify ranges of indices over which to slice an array.
Since multiple arguments are not allowed for the indexing operator in C++, the call operator is used for array slicing.

For compiling these examples on Windows, I used gfortran and a recent development version (3.7.0 dev) of clang compiled with mingw-w64.
The exact compiler calls replacing "(algorithm)" with the name of each particular algorithm were
```shell
clang++ -fPIC -O3 -I"c:/Program Files (x86)/libdynd/include" -std=c++11 (algorithm).cpp -L"c:/Program Files (x86)/libdynd/lib/static" -ldynd -o (algorithm).exe
gfortran -fPIC -O3 (algorithm).f90 -o (algorithm)_f.exe
```
The corresponding compiler calls on BSD, Linux, or other Unix-based operating systems are similar with the `-I` and `-L` directories being modified (or omitted) according to the location of the DyND headers and library.
The same set of flags should work with recent versions of clang++ and g++.

## Horner's Algorithm
Horner's algorithm is the standard algorithm for evaluating a power basis polynomial at a parameter value, given the coefficients for the polynomial.
Here the coefficients are assumed to be given in order from highest to lowest degree.

```Python
# Python
from __future__ import print_function
import numpy as np

def horner(a, t):
    v = a[0]
    for i in xrange(1, a.shape[0]):
        v = t * v + a[i]
    return v

if __name__ == '__main__':
    a = np.array([1, -2, 3, 0], 'd')
    print(horner(a, .3333))
```

```C++
// C++
#include <iostream>
#include <dynd/array.hpp>

using namespace dynd;

nd::array horner(nd::array a, double t){
    nd::array v = a(0);
    size_t e = a.get_dim_size();
    for(size_t i=1; i < e; i++){
        v = t * v + a(i);}
    return v;}

int main(){
    nd::array a = {1., -2., 3., 0.};
    std::cout << horner(a, .3333) << std::endl;}
```

```FORTRAN
! Fortran
module hornermodule
implicit none
contains
double precision function horner(a, t)
    double precision, intent(in) :: a(:)
    double precision, intent(in) :: t
    double precision v
    integer i, n
    n = size(a)
    v = a(1)
    do i = 2, n
        v = t * v + a(i)
    end do
    horner = v
end function horner
end module hornermodule

program main
    use hornermodule
    double precision a(4)
    a = (/1, -2, 3, 0/)
    print *, horner(a, .3333D0)
end program main
```

## Decasteljau's Algorithm

Decasteljau's Algorithm is used to evaluate polynomials in Bernstein form.
It is also often used to evaluate B&eacute;zier curves.

```Python
# Python
from __future__ import print_function
import numpy as np

def decasteljau(a, t):
    for i in xrange(a.shape[0]-1):
        a = (1-t) * a[:-1] + t * a[1:]
    return a

if __name__ == '__main__':
    a = np.array([1, 2, 2, -1], 'd')
    print(decasteljau(a, .25))
```

```C++
// C++
#include <iostream>
#include <dynd/array.hpp>

using namespace dynd;

nd::array decasteljau(nd::array a, double t){
    size_t e = a.get_dim_size();
    for(size_t i=0; i < e-1; i++){
        a = (1.-t) * a(irange()<(e-i-1)) + t * a(0<irange());}
    return a;}

int main(){
    nd::array a = {1., 2., 2., -1.};
    std::cout << decasteljau(a, .25) << std::endl;}
```

In the Fortran version of this algorithm, dynamic allocation within the function is needed.

```FORTRAN
! Fortran
module decasteljaumodule
implicit none
contains
double precision function decasteljau(a, t)
    double precision, intent(in) :: a(:)
    double precision, intent(in) :: t
    double precision, allocatable :: b(:)
    integer i, n
    n = size(a)
    allocate(b(n-1))
    b = (1-t) * a(:n-1) + t * a(2:)
    do i = 2, n-1
        b(:n-i) = (1-t) * b(:n-i) + t * b(2:n-i+1)
    end do
    decasteljau = b(1)
end function decasteljau
end module decasteljaumodule

program main
    use decasteljaumodule
    double precision a(4)
    a = (/1, 2, 2, -1/)
    print *, decasteljau(a, .25D0)
end program main

```

## Clenshaw's Algorithm (Chebyshev Polynomials)
Clenshaw's algorithm provides a simple recurrence for evaluating Chebyshev polynomials.
There is a similar algorithm for evaluating Legendre polynomials, but we will not demonstrate it here.
The code here is a simplified adaptation of the code 

```Python
# Python
from __future__ import print_function
import numpy as np

def clenshaw(a, t):
    c0, c1 = a[-2], a[-1]
    for i in xrange(3, a.shape[0]+1):
        c0, c1 = a[-i] - c1, c0 + c1 * (2 * t)
    return c0 + c1 * t

if __name__ == '__main__':
    a = np.array([1, 2, -1, -4], 'd')
    print(clenshaw(a, -.5))
```

```C++
// C++
#include <iostream>
#include <dynd/array.hpp>

using namespace dynd;

nd::array clenshaw(nd::array a, double t){
    size_t e = a.get_dim_size();
    nd::array c0 = a(e-2), c1 = a(e-1), temp;
    for(size_t i=3; i<e+1; i++){
        temp = c0;
        c0 = a(e-i) - c1;
        c1 = temp + c1 * (2 * t);}
    return c0 + c1 * t;}

int main(){
    nd::array a = {1., 2., -1., -4.};
    std::cout << clenshaw(a, -.5) << std::endl;}
```

```FORTRAN
! Fortran
module clenshawmodule
implicit none
contains
double precision function clenshaw(a, t)
    double precision, intent(in) :: a(:)
    double precision, intent(in) :: t
    double precision c0, c1, temp
    integer i, n
    n = size(a)
    c0 = a(n-1)
    c1 = a(n)
    do i = 2, n-1
        temp = c0
        c0 = a(n-i) - c1
        c1 = temp + c1 * (2 * t)
    end do
    clenshaw = c0 + c1 * t
end function clenshaw
end module clenshawmodule

program main
    use clenshawmodule
    double precision a(4)
    a = (/1, 2, -1, -4/)
    print *, clenshaw(a, -.5D0)
end program main
```
