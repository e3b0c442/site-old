---
title: "Advent of Code 2015 Day 1: Go"
date: 2022-05-21T03:45:29.370Z
draft: false
featured: false
tags:
  - advent-of-code-2015
  - advent-of-code
  - go
image:
  filename: featured
  focal_point: Smart
  preview_only: false
---
_The code for these articles can be found at https://github.com/e3b0c442/aoc._

Now that we've set up our Go build environment and boilerplate, we can dive right into the code for the Day 1 puzzle.

# The day

```go {linenos=table}
package aoc2015

import (
	"fmt"
	"os"
	"time"
)

func Day1(filename string) error {
	fmt.Println("Day 1: Not Quite Lisp")

	input, err := os.ReadFile(filename)
	if err != nil {
		return err
	}

	start := time.Now()
	solution, err := day1Part1(string(input))
	end := time.Now()
	if err != nil {
		return err
	}
	fmt.Printf("\tPart 1: %d (%s)\n", solution, end.Sub(start))

	start = time.Now()
	solution, err = day1Part2(string(input))
	end = time.Now()
	if err != nil {
		return err
	}
	fmt.Printf("\tPart 2: %d (%s)\n", solution, end.Sub(start))

	return nil
}

func day1Part1(input string) (int, error) {
	floor := 0
	for _, c := range input {
		switch c {
		case '(':
			floor++
		case ')':
			floor--
		default:
			return 0, fmt.Errorf("invalid input: %c", c)
		}
	}

	return floor, nil
}

func day1Part2(input string) (int, error) {
	floor := 0

	for i, c := range input {
		switch c {
		case '(':
			floor++
		case ')':
			floor--
		default:
			return 0, fmt.Errorf("invalid input: %c", c)
		}
		if floor == -1 {
			return i + 1, nil
		}
	}

	return 0, fmt.Errorf("no solution found")
}
```

As with the other languages, there are three functions in our `day` file, one for the execution of the whole day, and one for each part of the day. Also similar to the other languages, the main day's function is almost-but-not-quite boilerplate.

```go {linenos=table}
package aoc2015

import (
	"fmt"
	"os"
	"time"
)
```

As with any other Go file, we start with a preamble that has the package name at the top, and the imports we are using.

```go {linenos=table,linenostart=9}
func Day1(filename string) error {
	fmt.Println("Day 1: Not Quite Lisp")
```

Our function is named `Day1`. Note the capitalization: this is important; while Go is not object-oriented and does not have inheritance, it does have a concept of exports. Simply speaking, all package level items whose first letters are uppercased are exported; those whose first letters aren't, are not. Because our `main` function is in a different package, we need to export the `Day1` function in order to have it be available to the `main` package.

Our function takes a single string argument, which is the path to the input file, and returns an `error` value.

As with the other languages, we first print the puzzle title, in this case using the `Println` function from the `fmt` package. We use `Println` here instead of `Printf` because there is no interpolation that needs to happen.

```go {linenos=table,linenostart=12}
	input, err := os.ReadFile(filename)
	if err != nil {
		return err
	}
```
Next, we read the input file into memory, using the `ReadFile` convenience function from the `os` package. You'll notice that there are two variable names on the left-hand side. We saw this previously in the boilerplate, but did not dig in. Go is fairly unique amongst general-purpose programming languages in supporting multiple return values. If you look up the API documentation for `os.ReadFile`, you'll notice it has the signature `func ReadFile(name string) ([]byte, error)`, meaning it takes a single string argument, and returns two values: a _byte slice_, and an _error_.

After calling the readfile function, we then check for a return error. The `error` type is an _interface_. We'll get more into interfaces later, but for the current discussion, it is relevant because interface values can be `nil`, thus making it easy to check for an error using `if err != nil`. You will notice this construct occuring very frequently in Go programs; it is the idiomatic way to check and propogate errors. If we receive an error, we propogate it upward by returning it from the calling function.

```go {linenos=table,linenostart=17}
	start := time.Now()
	solution, err := day1Part1(string(input))
	end := time.Now()
	if err != nil {
		return err
	}
	fmt.Printf("\tPart 1: %d (%s)\n", solution, end.Sub(start))
```

This segment of code is very similar to what is in our main package boilerplate. We:
* use `time.Now()` to get a current timestamp;
* run our part 1 function;
* use `time.Now()` again to get an ending timestamp;
* check for and propogate an error if one was returned; and
* print the result from part 1 with the elapsed time, using `time.Sub` to get the duration value.

```go {linenos=table,linenostart=25}
	start = time.Now()
	solution, err = day1Part2(string(input))
	end = time.Now()
	if err != nil {
		return err
	}
	fmt.Printf("\tPart 2: %d (%s)\n", solution, end.Sub(start))
```

We then essentially repeat the same code, calling the part 2 function instead. If you have eagle eyes, you may notice one other small difference: we're assigning our variables with the `=` operator, instead of `:=`. As you may have deduced, `:=` is a shorthand operator for declaring and assigning a variable in one statement.

```go
    foo := "bar"
```
is equivalent to
```go
    var foo string
    foo = bar
```

Go does not allow shadowing variables in the same scope, so the compiler will throw an error if we try to redeclare them using the `:=` operator.

```go {linenos=table,linenostart=33}
	return nil
}
```

As we have successfully gotten to the end of our function without issue, we can return `nil` to indicate that no errors were encountered.

As previously noted, this function is _almost_ boilerplate. There will be some minor modifications in later puzzles where the return value of the first part needs to be passed to the second part, or the returned type is different.

## Part 1
```go {linenos=table,linenostart=38}
func day1Part1(input string) (int, error) {
	floor := 0
	for _, c := range input {
		switch c {
		case '(':
			floor++
		case ')':
			floor--
		default:
			return 0, fmt.Errorf("invalid input: %c", c)
		}
	}

	return floor, nil
}

As with Python, this function is nearly identical to its C counterpart (and even moreso than Python), except for some syntatical differences. First, note our function signature. Our function name is not capitalized, meaning we aren't exporting the function. This is fine, because no other package needs to see this function. That said, unlike the other langages, we do need to prepend the day information to the function name, since, while all our our functions are not in the same file, they _are_ in the same package and thus visible to each other. Our function takes the input string as an argument, and returns an `int` and an `error`. Both values must be returned with any return statement.

Similar to Python, Go allows us to iterate over the items in a collection instead of by index, in Go's case by using the `range` keyword. This makes two values available within the loop: the index of the current iteration, and it's value. For part 1, we don't care about the index, so we use `_` to indicate that we will not be using that variable. In this special case, `range` treats the string as a list of _runes_. A rune in Go represents a single Unicode code point, so loosely one character.

Then, like C, we use a switch statement to branch depending on the character, and return an error if an unexpected character is found. Notice that in this case, we return `0` for the first value; since an integer is not a pointer or an interface, it cannot be `nil`, so we return the _zero value_ instead. Again, this is idiomatic to Go in error situations.

Finally, we return the found solution, along with `nil` to indicate no error was encountered.

## Part 2
```go {linenos=table,linenostart=52}
func day1Part2(input string) (int, error) {
	floor := 0

	for i, c := range input {
		switch c {
		case '(':
			floor++
		case ')':
			floor--
		default:
			return 0, fmt.Errorf("invalid input: %c", c)
		}
		if floor == -1 {
			return i + 1, nil
		}
	}

	return 0, fmt.Errorf("no solution found")
}
```

The part 2 function is quite similar. As with the other languages, we do need to keep track of how far through the input we've advanced, so we will use the index variable provided by the `range` keyword. Like with the C and Python versions, we return an error state if we see an unexpected character, or if we get to the end of the input without a solution. Like in part one, we return `0` as the first value in those cases.

# Conclusion
On the surface, Go seems to pull the best of both of our previous languages in. It is strongly-typed like C, but requires less boilerplate due to a more complete standard library, like Python. It has a more C-like syntax and is compiled, but has quality of life improvments like type inference and automatic memory management. Go brings its own strengths in as well, with code generation being a first class citizen and allowing us to dispense with the boilerplate in one shot, as well as it's modern build toolchain that is part of the language distribution itself.

One of the most shocking benefits, though, comes from our execution times:

```console
$ go run ./cmd/day1 ../1.txt
Day 1: Not Quite Lisp
        Part 1: 232 (26.834µs)
        Part 2: 1783 (7.416µs)
        Completed in 109.583µs
```
Our Go program completes nearly at parity with our C program, speed-wise. Pretty impressive for a language that, while compiled, still has a runtime and the overhead of a garbage collector.

We have one more language in which to complete day 1. Rust is next!