---
title: "Advent of Code 2015 Day 1: Go"
date: 2022-04-23T11:51:16.153Z
draft: false
featured: false
image:
  filename: featured
  focal_point: Smart
  preview_only: false
---
_The code for these articles can be found at <https://github.com/e3b0c442/aoc>._

We've look at day 1 in C and Python, two very different languages, both in syntax and operation. Today, we add a third: Go.

Go's main strengths are around developer productivity. Like C, it uses brackets to delimit blocks, is strongly-typed, and is compiled. Like Python, it has a fantastic standard library (albeit with a few holes, but that's an article for another time). Go's biggest benefit is around its toolset; it is truly "batteries included," and we'll be able to set up our builds just using the included toolset.

## Boilerplate

You didn't think we were going to get away without any boilerplate, did you? Thankfully, unlike Python we'll be able to generally avoid needing to copy and paste it for each day, thanks to the power of Go's first-class code generation. 

The first thing we are going to do is set up our module:

```terminal
$ go mod init github.com/e3b0c442/aoc/2015/go
```

This will create a `go.mod` file in the folder, which tells the Go compiler that this code is one module, and tracks all of the necessary dependencies. Now let's start writing some code.

### gen.go

```go {linenos=table}
//go:build ignore

package main

import (
	"fmt"
	"os"
	"path"
	"text/template"
)

var singleTmpl = `package main

import (
	"fmt"
	"log"
	"os"
	"time"

	aoc2015 "github.com/e3b0c442/aoc/2015/go"
)

func main() {
	if len(os.Args) < 2 {
		log.Fatal("must provide path to input file")
	}

	start := time.Now()
	if err := aoc2015.Day{{ . }}(os.Args[1]); err != nil {
		log.Fatal(err)
	}
	fmt.Printf("\tCompleted in %s\n", time.Now().Sub(start))
}
`

var allTmpl = `package main

import (
	"fmt"
	"log"
	"os"
	"path"
	"time"

	aoc2015 "github.com/e3b0c442/aoc/2015/go"
)

var days = []func(string) error{
	{{ range . -}}aoc2015.Day{{.}},{{- end }}
}

func main() {
	if len(os.Args) < 2 {
		log.Fatal("must provide path to input files")
	}

	start := time.Now()
	for i, day := range days {
		dayStart := time.Now()
		err := day(path.Join(os.Args[1], fmt.Sprintf("%d.txt", i+1)))
		dayEnd := time.Now()
		if err != nil {
			log.Fatal(err)
		}
		fmt.Printf("\tCompleted in %s\n", dayEnd.Sub(dayStart))
	}
	fmt.Printf("All puzzles completed in %s\n", time.Now().Sub(start))
}
`

func main() {
	single := template.New("single")
	single = template.Must(single.Parse(singleTmpl))

	fmt.Println("generating mains")
	days := os.Args[1:]
	wd, err := os.Getwd()
	if err != nil {
		panic(err)
	}

	for _, day := range days {
		dayPath := path.Join(wd, "cmd", fmt.Sprintf("day%s", day))
		os.MkdirAll(dayPath, 0755)
		f, err := os.Create(path.Join(dayPath, "main.go"))
		if err != nil {
			panic(err)
		}
		if err := single.Execute(f, day); err != nil {
			panic(err)
		}
	}

	all := template.New("all")
	all = template.Must(all.Parse(allTmpl))

	allPath := path.Join(wd, "cmd", "aoc2015")
	os.MkdirAll(allPath, 0755)
	f, err := os.Create(path.Join(allPath, "main.go"))
	if err != nil {
		panic(err)
	}
	if err := all.Execute(f, days); err != nil {
		panic(err)
	}
}
```

This file contains (almost) all the logic required to generate our main packages for each day and the whole puzzle output. Let's look at this bit by bit.

```go {linenos=table}
//go:build ignore
```

At the very top of the file we place a build tag. This tells the Go compiler to ignore this file when doing the regular build, which is exactly what we want; code generation will happen with its own step.

```go {linenos=table,linenostart=3}
package main
```

Next, we have our package declaration. Since we're using Go itself to build the generation, this file is essentially a self-contained program. Since it is the entrypoint, the package name must be `main`. 

```go {linenos=table,linenostart=5}
import (
	"fmt"
	"os"
	"path"
	"text/template"
)
```

Continuing on, we come to our imports. These are the packages from which we need to use code in this file. In Go, a folder is a package, so depending on how your project is organized, you may need to explictly import from other packages in your module, as well as the standard library and any external dependencies you pull in.

For the generation program, we're using the `fmt` package, which has text formatting routines; the `os` packages which has routines for writing our generated files to disk; the `path` package which has routines for sanely assembling filesystem paths, and the `text/template` package, which gives us access to Go's standard template language and compiler.

```go {linenos=table,linenostart=12}
var singleTmpl = `package main

import (
	"fmt"
	"log"
	"os"
	"time"

	aoc2015 "github.com/e3b0c442/aoc/2015/go"
)

func main() {
	if len(os.Args) < 2 {
		log.Fatal("must provide path to input file")
	}

	start := time.Now()
	if err := aoc2015.Day{{ . }}(os.Args[1]); err != nil {
		log.Fatal(err)
	}
	fmt.Printf("\tCompleted in %s\n", time.Now().Sub(start))
}
`
```

This is our template for the entrypoint for a single day's puzzles. This template is a _raw string_, which is specified by using backticks (```) instead of quotation marks. This means that everything in the string will be taken literally, including line breaks and anything that would be an escape sequence in a regular string.

In our template string, you can see the package declaration and imports just like the containing file. Note that this file uses an import alias to avoid package conflicts: `aoc "github.com/e3b0c442/aoc/2015/go"`. By default, Go uses the last portion of the package path as the import name.

We then have our `main` function. Just like in `C`, `main` is the entrypoint for the program. As previously noted, it must also be in a package named `main`. Let's dig into this `main` function:

```go {linenos=table,linenostart=24}
	if len(os.Args) < 2 {
		log.Fatal("must provide path to input file")
	}
```
Similar to our C and Python boilerplates, we check that we have at least two command line arguments: the first is the command itself, and the second is the path to the input files. In Go, command line arguments are provided via the `Args` variable in the `os` package, which is a _slice_ of strings with the arguments. We'll dig into slices later, but for now, think of them similarly to Python lists. If we don't have the correct number of arguments, we use the `Fatal` function from the `log` package print the error message to the screen and exit the program with a non-zero status.

```go {linenos=table,linenostart=28}
	start := time.Now()
```

Once we've verified the arguments, we'll get the current time so that we can measure the duration at the end. This is done with the `Now` function from the `time` package.

```go {linenos=table,linenostart=29}
	if err := aoc2015.Day{{ . }}(os.Args[1]); err != nil {
		log.Fatal(err)
	}
```
There are a couple of interesting things going on here. The first thing to note is that we have the one and only template tag for our file, `{{ . }}`. The dot in this case is the _context_, i.e. the data that is passed into the template executor along with the template itself. In our case, this is simply going to be a day number, so this tag would be replaced with a numeral when the template is executed.

As far as what the generated code does: this is what actually pulls in our day logic and runs it. Our day functions will be in a separate package `aoc2015`, and they will all have the signature `func (string) error`, meaning they take a single string argument, and return an error. We know our function is successful if the returned value is `nil`, that is, no error. Here, we use a shorthand which allows us to assign and evaluate `err` on the same line; this is frequently used for functions that return only `err` in order to error check. If we do get an error -- that is, `err` is not `nil` -- we will print and exit as before. The day function itself is responsible for printing its results if there is no error.

```go {linenos=table,linenostart=32}
	fmt.Printf("\tCompleted in %s\n", time.Now().Sub(start))
}
```

Here, we take another time measurement and print the duration. There are a few things going on here. We are using the `Printf` function from the `fmt` package to print a formatted string, similar to C. Notice that we have a `%s` token in the format string, indicating that it is expecting a string argument.

Taking a look at our argument, we are first getting another instantateous time measurement with `time.Now`. This returns a `time.Time` struct. We take advantage of this to chain into the `Sub` method (a function that is tied to a type, as opposed to a first-class function) on `time.Time`, which takes another `time.Time` struct as its argument, and returns a `time.Duration`.

But wait, you say. Our format string is expecting a string for the token, but we're passing it `time.Duration`. Won't that break?

The answer is no, because of a special _interface_ satisfied by `time.Duration`. Interfaces are integral to go, and they are best described as a contract: any type that satisfies an interface implements all of the methods specified by that interface. In our case, the %s token will accept a string, _or a value of a type that implements the `fmt.Stringer` interface_. `fmt.Stringer` specifies one method, `String() string`, which `time.Duration` implements. This allows us to pass the value, which formats itself for presentation.

Unlike in C, `main` has no return value in Go, so we simply end execution of the function.

Next, we have our all days template:
```go {linenos=table,linenostart=36}
var allTmpl = `package main

import (
	"fmt"
	"log"
	"os"
	"path"
	"time"

	aoc2015 "github.com/e3b0c442/aoc/2015/go"
)

var days = []func(string) error{
	{{ range . -}}aoc2015.Day{{.}},{{- end }}
}

func main() {
	if len(os.Args) < 2 {
		log.Fatal("must provide path to input files")
	}

	start := time.Now()
	for i, day := range days {
		dayStart := time.Now()
		if err := day(path.Join(os.Args[1], fmt.Sprintf("%d.txt", i+1))); err != nil {
			log.Fatal(err)
		}
		dayEnd := time.Now()
		fmt.Printf("\tCompleted in %s\n", dayEnd.Sub(dayStart))
	}
	fmt.Printf("All puzzles completed in %s\n", time.Now().Sub(start))
}
`
```

Similar to the single-day template, we have our package declaration and imports, so we won't go over those again.

```go {linenos=table,linenostart=48}
var days = []func(string) error{
	{{ range . -}}aoc2015.Day{{.}},{{- end }}
}
```

Here, we are defining a package variable, which is a slice of day functions. We use the `range` template action to iterate over the values of our context, which in this case is a slice of numbers, so the end result in the generated file will be one line per day with the function name.

Now we go into our main function. The time measurement is nearly identical, so we won't cover that again, but here we use a `for` loop to loop over the slice of day functions:

```go {linenos=table,linenostart=58}
	for i, day := range days {
		dayStart := time.Now()
		err := day(path.Join(os.Args[1], fmt.Sprintf("%d.txt", i+1)))
		dayEnd := time.Now()
		if err != nil {
			log.Fatal(err)
		}
		fmt.Printf("\tCompleted in %s\n", dayEnd.Sub(dayStart))
	}
```

For loops can be created in the same fashion as in C, but in Go they can also use the `range` keyword to iterate over a container, similar to the `in` keyword in Python. When that container is a slice, each iteration has access to the index (`i` in our loop) and the value (`day` in our loop). 

```go {linenos=table,linenostart=60}
		if err := day(path.Join(os.Args[1], fmt.Sprintf("%d.txt", i+1))); err != nil {
			log.Fatal(err)
		}
```
Here, we're making the call to the day function, which is passed as the variable `day` by the loop. Functions in Go are first class and can be assigned to variables directly, making it easy for us to pass them around in this manner. We provide our argument, which is the generated path to the input file -- the folder provided on the command line, plus the numbered filename. We use the same one-line error checking shorthand as in the single-day file.

These two templates are the boilerplate for our main files. How do we use them to generate the actual files? This is where the `main` function in gen.go comes in (not to be confused with the `main` function in the templates).

```go {linenos=table,linenostart=70}
func main() {
	single := template.New("single")
	single = template.Must(single.Parse(singleTmpl))
```

This starts out simply enough. First, we create a new template using the `text/template` package. Then, we parse our template string for the single day template. This requires two lines because `Parse` is a method on `*text.Template`. We use the `Must` function here to eliminate an unnecessary error check; the template is part of the code itself, so it is expected to be correct. `Must` will panic if the template cannot be parsed. In the end, we end up with a pointer to a `text.Template` struct containing a parsed template.

```go {linenos=table,linenostart=74
	fmt.Println("generating mains")
	days := os.Args[1:]
	wd, err := os.Getwd()
	if err != nil {
		panic(err)
	}
```

After printing some output to let the runner know things are proceeding as expected, we grab our command-line arguments. Remember that the first item in `os.Args` is the command name, so we slice that first argument off to get the rest of the arguments. Our generator is expecting one numeric argument for each single day we are generating the main package for. Finally, we attempt retrieve the current working directory with `os.Getwd`, and panic if we can't, because that's not recoverable.

```go {linenos=table,linenostart=81}
	for _, day := range days {
		dayPath := path.Join(wd, "cmd", fmt.Sprintf("day%s", day))
		os.MkdirAll(dayPath, 0755)
		f, err := os.Create(path.Join(dayPath, "main.go"))
		if err != nil {
			panic(err)
		}
		if err := single.Execute(f, day); err != nil {
			panic(err)
		}
		f.Close()
	}
```

Next is the actual logic that generates the boilerplates. For each argument representing a day number, we:
* Determine the path for the main package for that day by joining the current working directory, the `cmd` subdirectory, and the `dayX` directory within `cmd` where X is the provided day number;
* Create the directory for the main package for that day using the path with `os.MkdirAll`, using `os.MkdirAll` instead of `os.Mkdir` to cover the case that the `cmd` folder doesn't yet exist;
* Create and open file called "main.go" within the package folder;
* Execute the template with the `.Execute` method on the template, using the created file as the first (`io.Writer`) argument, and the day number as the second (context) argument, which populates the template and writes it to disk;
* Close the opened file.