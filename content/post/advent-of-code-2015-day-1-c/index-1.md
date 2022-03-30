---
title: "Advent of Code 2015 Day 1: C"
subtitle: Not Quite Lisp
date: 2022-03-30T03:58:16.625Z
draft: false
featured: false
authors:
  - nick-meyer
tags:
  - advent-of-code
  - advent-of-code-2015
  - c
image:
  filename: featured
  focal_point: Smart
  preview_only: false
---
We've set up our C build environment and boilerplate, and we're ready to dive into some problem solving. Let's take a look at the Day 1 puzzle.

> Santa is trying to deliver presents in a large apartment building, but he can't find the right floor - the directions he got are a little confusing. He starts on the ground floor (floor `0`) and then follows the instructions one character at a time.
>
> An opening parenthesis, `(`, means he should go up one floor, and a closing parenthesis, `)`, means he should go down one floor.
>
> The apartment building is very tall, and the basement is very deep; he will never find the top or bottom floors.
>
> For example:
>
> * `(())` and `()()` both result in floor `0`.
> * `(((` and `(()(()(` both result in floor `3`.
> * `))(((((` also results in floor `3`.
> * `())` and `))(` both result in floor `-1` (the first basement level).
> * `)))` and `)())())` both result in floor `-3`.
>
> To *what floor* do the instructions take Santa?

The first thing we need to do is fully understand the problem. Reading through the description, we can distill the following:
- We start on the ground floor, which is numbered `0`
- We are given an input with instructions for going up (`(`) and down (`)`)
- The solution is the floor number we're on at the end of the input

Let's get to work.

### Boilerplate... again

When setting up our top-level boilerplate, we defined a function signature for each day's puzzles of `int (const char *)`, where the single argument is the filename for the puzzle input. For this puzzle, we could simply open the file and read directly, but generally speaking reading from the filesystem a byte at a time is much less efficient than reading large chunks, so we will read the whole input file into memory and operate on it in memory.

Unfortunately (?) for us, we're starting with C, and there is no single function in the standard library that will do this for us, so we get to create our own!

Because we don't know the size of our file at compile time, we will need to dynamically allocate memory for this, meaning we need to be able to return a pointer. Easy, right? Remember that in C, the length of an array is not stored with an array, we need to return the length (or force the end-user to trust that we correctly end our C string with the null terminator, which as a consumer of C library rings all the alarm bells). This is a problem for us because we can only return one value in C.

There are a couple of ways we could handle this. We could create a custom `struct` type which includes the length and a pointer to the array. In fact, this is how most high-level programming language handle dynamic arrays under the hood. However, we're going to use a mechanism that is common in C programming: we will return the length of the array, and pass a _pointer to pointer_ as an argument. This allows us to pass a memory address into the function wherein we can store _another_ memory address that actually points at the data we care about, and have this available once the execution of the function has ended.

We'll first add our declaration to `lib.h`:

```c {linenos=table,linenostart=9}
int read_file_to_buffer(char **buf, const char *filename);
```
