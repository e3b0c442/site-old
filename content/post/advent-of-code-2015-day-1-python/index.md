---
title: "Advent of Code 2015 Day 1: Python"
date: 2022-04-03T02:40:20.929Z
draft: false
featured: false
tags:
  - advent-of-code-2015
  - advent-of-code
  - python
image:
  filename: featured
  focal_point: Smart
  preview_only: false
---
_The code for these articles can be found at https://github.com/e3b0c442/aoc._

Now that we've built our baseline for day 1 in C, let's look at Python.

Python is by nearly all measures the most universally popular programming language today. It has a simplicity and flexibility that is unmatched, but the lack of strong typing and interpreted nature can make it a poor fit for complex codebases. One area to take particular care is that unlike most other modern languages, whitespace has meaning in Python, so make sure your indents are at the same levels if you're following along at home.

Python has a much more comprehensive standard library, so we can keep our boilerplate pretty small. In order to improve the quality of our code, we'll be using type hints wherever possible.

_Note: we will be using Python 3.10 for this series of articles. In particular, the structural pattern matching feature added in 3.10 will be used._

### Boilerplate

Like with C, we'll have an entrypoint for each day, and an entrypoint to run all puzzles. Unlike C, these will not be in the same file. Let's look at the all-puzzles entrypoint, `aoc2015.py`:

```python {linenos=table}
#!/usr/bin/env python3

import sys
import time
from importlib import import_module
from lib import format_duration

if len(sys.argv) < 2:
    raise RuntimeError("Must provide path to input files")

start = time.time()
for i in range(1):
    day = import_module(f"day{i+1}")
    day_start = time.time()
    day.day(f"{sys.argv[1]}/{i+1}.txt")
    print(f"\tCompleted in {format_duration(time.time()-day_start)}")
print(f"All puzzles completed in {format_duration(time.time() - start)}")
```

Breaking things down:

```python {linenos=table}
#!/usr/bin/env python3
```

We start each file with a hashbang. When executed directly from the shell, this tells the shell where the interpreter is.

```python {linenos=table,linenostart=3}
import sys
import time
from importlib import import_module
from typing import List
from lib import format_duration
```

After the hashbang, we have our imports. These don't technically need to be at the top of the file, and there may actually be instances where you need to restrict an import to a certain scope; that said, unless scope restriction is required, placing imports at the top of the file is best practice.

```python {linenos=table,linenostart=9}
if len(sys.argv) < 2:
    raise RuntimeError("Must provide path to input files")
```

Now, like with C, we check to make sure there is an argument for the folder centaining the input files.

_"But wait!"_ you might say, _"where's the main function?"_. Unlike C and many other compiled languages, Python does not have an entrypoint function; rather, the file itself is the entrypoint. There _is_ a way to simulate a `main` function which we'll look at shortly.

```python {linenos=table,linenostart=11}
start = time.time()
```

We take a time reading so that we can measure execution.

```python {linenos=table,linenostart=12}
for i in range(1):
    day = import_module(f"day{i+1}")
    day_start = time.time()
    day.day(f"{sys.argv[1]}/{i+1}.txt")
    print(f"\tCompleted in {format_duration(time.time()-day_start)}")
``` 

Here is our loop over the days, similar to the C loop, but also fundamentally different.

Remember back when building our boilerplate with C, we discussed that C does not support _dynamic dispatch_; meaning, we cannot call a C function by a string name. We had to build an array of pointers to the functions themselves.

On the other hand, Python _does_ support dynamic dispatch, and we'll take advantage of this to simplify our code.

Let's break things down even further:

```python {linenos=table,linenostart=12}
for i in range(1):
```

In Python, `for` loops over an iterable object. In this case, the `range` function returns an iterator from `0` to `n-1` where `n` is the provided argument. `range` has optional arguments for controlling the start and the increment, but those are not necessary in our case.

```python{linenos=table,linenostart=13}
    day = import_module(f"day{i+1}")
```

Here, we dynamically import the module for the given day by string name. In Python, each file is a module, so we are building the filename (minus the extension) for the file. Note that we use `i+1` because our `range` is zero-indexed, while our days are 1-indexed.

```python{linenos=table,linenostart=14}
    day_start = time.time()
```

Here we take another time measurement for the start of that day's function.

```python{linenos=table,linenostart=15}
    day.day(f"{sys.argv[1]}/{i+1}.txt")
```
Here we call the `day` function of the day's module, building the input filename. Again, remember that we need to adjust the indexing.

```python{linenos=table,linenostart=16}
    print(f"\tCompleted in {format_duration(time.time()-day_start)}")
print(f"All puzzles completed in {format_duration(time.time() - start)}")
```
Finally, we print the execution times for each day (the indented line) and the puzzle set as a whole. Remember, whitespace has meaning in Python. The indented line is within the `for` loop; the outdented line is not.

You'll notice a function `format_duration` imported from `lib`. Like with C, this is a function we've had to build ourselves, so we've added a `lib` module with that code:

```python{linenos=table}
def format_duration(seconds: float) -> str:
    labels = ["s", "ms", "µs", "ns"]
    i: int = 0
    for i in range(4):
        if seconds > 1e-3:
            break
        seconds *= 1000
    return f"{seconds:.3f}{labels[i]}"
```

The logic of this function is identical to the C function. We have an array (called a _list_ in Python) with labels, loop over a limited range while adjusting the input and return a formatted string when the adjusted input has a sane value, with the correct label. 

This is (almost) all the boilerplate we need to complete day 1 in Python, so lets look at our solution.

### The day

```python{linenos=table}
#!/usr/bin/env python3

import sys
import time
from lib import format_duration


def day(input_file: str) -> None:
    with open(input_file) as f:
        input = f.read()

    print("Day 1: Not Quite Lisp")
    start = time.time()
    print(f"\tPart 1: {_part1(input)} ({format_duration(time.time() - start)})")
    start = time.time()
    print(f"\tPart 2: {_part2(input)} ({format_duration(time.time() - start)})")


def _part1(input: str) -> int:
    floor: int = 0

    for c in input:
        match c:
            case "(":
                floor += 1
            case ")":
                floor -= 1
            case _:
                raise RuntimeError("invalid input")

    return floor


def _part2(input: str) -> int:
    floor: int = 0

    for (i, c) in enumerate(input):
        match c:
            case "(":
                floor += 1
            case ")":
                floor -= 1
            case _:
                raise RuntimeError("invalid input")

        if floor < 0:
            return i + 1

    raise RuntimeError("no solution found")


if __name__ == "__main__":
    if len(sys.argv) < 2:
        raise RuntimeError("must provide path to input file")
    start = time.time()
    day(sys.argv[1])
    print(f"\tCompleted in {format_duration(time.time() - start)}")

```

Notice I said _almost_ all the boilerplate. When we created our multi-day file, that was the only logic it contained. Rather than have a single entrypoint like C, for Python, each day's file will be able to run it's own puzzles independently. We achieve this by using the common construct `if __name__ == "__main__"`. This statement is true if the Python interpreter starts execution with this file, i.e. it was provided this path as a command-line argument.

In this case, we behave similarly to the single-day macro branch in our C solution: check to make sure we have an argument for the input file, start timing, call the day function, and then print the completion duration.

Now, for the almost-boilerplate.

```python {linenos=table,linenostart=8}
def day(input_file: str) -> None:
    with open(input_file) as f:
        input = f.read()

    print("Day 1: Not Quite Lisp")
    start = time.time()
    print(f"\tPart 1: {_part1(input)} ({format_duration(time.time() - start)})")
    start = time.time()
    print(f"\tPart 2: {_part2(input)} ({format_duration(time.time() - start)})")
```

Each day's module will have a `day` function. Since the modules are effectively namespaces, we can use the same function identifier in each file safely. Like C, we cannot make this a standard boilerplate because some puzzles will require minor changes.

Breaking this down:

```python {linenos=table,linenostart=8}
def day(input_file: str) -> None:
```

Our function definition takes one argument of type `str` and has no return value. The types, again, are hints and do not affect the actual execution, they are just an IDE and static analyzer aid.

```python {linenos=table,linenostart=9}
    with open(input_file) as f:
        input = f.read()
```

This block opens the provided input file path in a _context manager_, and reads the file to a variable. Among other things that aren't relevant right now, the context manager closes the input file once it is read, so we don't need to worry about doing it manually. Note that a context manager does _not_ affect scope, so the `input` variable will be available outside the context manager.

```python {linenos=table,linenostart=12}
    print("Day 1: Not Quite Lisp")
```

As in C, we'll print the puzzle title.

```python {linenos=table,linenostart=13}
    start = time.time()
    print(f"\tPart 1: {_part1(input)} ({format_duration(time.time() - start)})")
    start = time.time()
    print(f"\tPart 2: {_part2(input)} ({format_duration(time.time() - start)})")
```

Also as in C, we'll take a time measurement, run each day's puzzle, and print the result with the formatted measured duration.

Because of the power of Python's standard library, we are able to achieve this with _much_ less boilerplate code than our C solutions. Now let's look at the logic:

#### Part 1

```python {linenos=table,linenostart=19}
def _part1(input: str) -> int:
    floor: int = 0

    for c in input:
        match c:
            case "(":
                floor += 1
            case ")":
                floor -= 1
            case _:
                raise RuntimeError("invalid input")

    return floor
```

This looks nearly identical to the C function. Our `for` construct loops over the input directly instead of needing the index as with C. We utilize Python's new structural pattern matching in the same fashion as the C `switch` statement; `case _` is equivalent to `default` in a C switch statement, matching only when no other cases match. Note that unlike C, Python does NOT execute multiple branches of a `match` statement, therefore there is no `break` statement in the case.

Arguably the biggest departure from C so far is with error handling. Rather than use a canary value, we use Python's built in error-handling mechanism to raise an exception if invalid input is encountered.

Notably, we prefix the function name with `_` as a hint to the IDE/static analyzer that the function should be private to the module. Again, this is not enforced at the interpreter level.

#### Part 2

```python {linenos=table,linenostart=34}
def _part2(input: str) -> int:
    floor: int = 0

    for (i, c) in enumerate(input):
        match c:
            case "(":
                floor += 1
            case ")":
                floor -= 1
            case _:
                raise RuntimeError("invalid input")

        if floor < 0:
            return i + 1

    raise RuntimeError("no solution found")
```

Again, the logic is nearly identical to the C function. However, unlike part 1, we can't just interate directly over the input because we need to know how far into the input we are when we return. In order to accomplish this, we call the `enumerate` function, which transforms the input into an iterator of tuples which contain the current index and the value. We use destructuring to assign these items directly to the variables `i` and `c` within the loop.

### Conclusions

One thing that cannot be argued is that our Python solution and the boilerplate has far fewer lines of code than we've written to enable day 1 in C. Most impactful at this moment, we are able to use the standard library`s file input capabilities to read the file into a string instead of needing to build this ourselves.

However, there is one potentially serious tradeoff: execution time.

```shell
$ ./day1.py ../1.txt
Day 1: Not Quite Lisp
        Part 1: 232 (0.285ms)
        Part 2: 1783 (0.106ms)
        Completed in 0.474ms
```
vs
```shell
$  bin/day1 ../1.txt 
Day 1: Not Quite Lisp
        Part 1: 232 (0.032ms) 
        Part 2: 1783 (0.009ms)
        Completed in 0.106ms
```
Our Python solutions are significantly slower than their C counterparts. Note that this is a second execution of both programs, so this includes the optimization of Python's bytecode caching. However, for simple problems that don't require a lot of low-level compute, this is probably a reasonable trade-off. More serious in my opinion is the lack of static typing. Dynamic typing can be very powerful, but comes with a _huge_ risk of runtime errors undetectable before deployment.

Next up: Go.