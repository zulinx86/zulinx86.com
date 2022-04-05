---
layout: post
title: Cython
---

# What is Cython?
- Cython is pronounced /ˈsaɪθɒn/.
- It makes writing C extensions for Python as easy as Python itself.
- It's designed to get C-like performance with code that is written mostly in Python with optional additional C-inspired syntax.
- Annotated Python-like code is compiled to C/C++, then automatically wrapped in interface code, producing extension modules that can be loaded by regular Python code with significantly less computational overhead at runtime.


# How to Compile
- Required Files
	- Cython code (`*.pyx`)
	- Setup script (`setup.py`)
		- Generates the extension module from Cython codes.
	- Main Python code (`*.py`)
		- Imports the generated extension module.
- Compile Process
	- `.pyx` -(Cython compiler)-> `.c` -(C compiler)-> `.so`


# How to Set Up
```sh
$ conda install cython
```

# Examples
## Hello World (To understand compile process)
### hello.pyx
```py
def say_hello():
	print("Hello World!")
```

### setup.py
```py
from setuptools import setup
from Cython.Build import cythonize

setup(name = "Hello World", ext_modoles = cythonize("*.pyx"))
```

### main.py
```py
import hello
hello.say_hello()
```

### Build and Execute
```sh
$ ls
hello.pyx       setup.py        main.py

$ python3 setup.py build_ext --inplace

$ tree
.
├── build
│   ├── lib.macosx-11-x86_64-3.9
│   │   └── hello.cpython-39-darwin.so
│   └── temp.macosx-11-x86_64-3.9
│       └── hello.o
├── hello.c
├── hello.cpython-39-darwin.so
├── hello.pyx
├── setup.py
└── main.py

$ python3 main.py
Hello World!
```


## Fibonacci Number
### fib_cython.pyx
```pyx
def fib_cython1(n):
	a, b = 0, 1
	for i in range(n):
		a, b = a + b, a
	return a

def fib_cython2(int n):
	a, b = 0, 1
	for i in range(n):
		a, b = a + b, b
	return a

def fib_cython3(int n):
	cdef int a, b, i

	a, b = 0, 1
	for i in range(n):
		a, b = a + b, b
	return a
```

### setup.py
```py
from setuptools import setup
from Cython.Build import cythonize

setup(name = "Fibonacci Number", ext_modules = cythonize("*.pyx"))
```

### main.py
```py
import time
from fib_cython import *

def fib_python(n):
	a, b = 0, 1
	for i in range(n):
		a, b = a + b, a
	return a

start_time = time.process_time()
fib_python(1000000)
end_time = time.process_time()
print("fib_python : {}".format(end_time - start_time))

start_time = time.process_time()
fib_cython1(1000000)
end_time = time.process_time()
print("fib_cython1: {}".format(end_time - start_time))

start_time = time.process_time()
fib_cython2(1000000)
end_time = time.process_time()
print("fib_cython2: {}".format(end_time - start_time))

start_time = time.process_time()
fib_cython3(1000000)
end_time = time.process_time()
print("fib_cython3: {}".format(end_time - start_time))
```

### Build and Execute
```
$ python3 setup.py build_ext --inplace
$ python3 main.py
fib_python : 10.717588
fib_cython1: 10.815109
fib_cython2: 0.02184700000000106
fib_cython3: 1.0000000010279564e-06
```
