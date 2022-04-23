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

```go
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