---
title: "Advent of Code 2015 Day 1: C"
subtitle: Not Quite Lisp
date: 2022-03-30T04:05:58.657Z
draft: false
featured: false
authors:
  - nick-meyer
tags:
  - advent-of-code-2015
  - advent-of-code
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

Our function will have two arguments: A pointer to a pointer where we can place the pointer to the allocated buffer, and the string with the filename. It will return the length of the buffer. Here's our implementation that will be added to `lib.c`:

```c {linenos=table,linenostart=18}
int read_file_to_buffer(char **buf, const char *filename)
{
    *buf = NULL;
    int rval;

    // open the file
    FILE *f = fopen(filename, "r");
    if (f == NULL)
    {
        set_err_msg("unable to open file: %s", strerror(errno));
        goto err_cleanup;
    }

    // get the length
    if (fseek(f, 0L, SEEK_END))
    {
        set_err_msg("unable to seek file");
        goto err_cleanup;
    }
    long filesize = ftell(f);
    if (filesize < 0)
    {
        set_err_msg("unable to get file size: %s", strerror(errno));
        goto err_cleanup;
    }
    rewind(f);

    // allocate the buffer
    *buf = calloc(filesize + 1, sizeof(char));
    if (*buf == NULL)
    {
        set_err_msg("unable to allocate buffer for file read: %s", strerror(errno));
        goto err_cleanup;
    }

    // read the file into the buffer
    size_t rd = fread(*buf, 1, filesize, f);
    if (rd < filesize)
    {
        if (ferror(f))
        {
            set_err_msg("unable to read entire file: %s", strerror(errno));
            goto err_cleanup;
        }
        set_err_msg("read %d bytes, expected %d", rd, filesize);
        goto err_cleanup;
    }

    rval = rd;
    goto cleanup;

err_cleanup:
    rval = -1;
    if (*buf != NULL)
        free(*buf);
    *buf = NULL;
cleanup:
    if (f != NULL)
        fclose(f);
    return rval;
}
```

Let's break this down.

```c {linenos=table,linenostart=10}
    *buf = NULL;
    int rval;
```

The first thing we do is set the _value_ behind the provided pointer (so, the pointer being pointed to) to `NULL`. It will remain `NULL` unless the operation succeeds. We also predeclare our return value here.

```c {linenos=table, linenostart=13}
    // open the file
    FILE *f = fopen(filename, "r");
    if (f == NULL)
    {
        set_err_msg("unable to open file: %s", strerror(errno));
        goto err_cleanup;
    }
```

Now we are going to attempt to open the file. `fopen` is included from `stdio.h`, and it returns `NULL` and sets `errno` if the file cannot be opened. If that is the case, we set our own error message with a little more detail using our `set_err_msg` utility, and make a jump to do more cleanup (more on this in a bit).

```c {linenos=table,linenostart=31}
    // get the length
    if (fseek(f, 0L, SEEK_END))
    {
        set_err_msg("unable to seek file");
        goto err_cleanup;
    }
    long filesize = ftell(f);
    if (filesize < 0)
    {
        set_err_msg("unable to get file size: %s", strerror(errno));
        goto err_cleanup;
    }
    rewind(f);
```

Remember that this is C, and we do not have a dynamically-resizing array. Therefore, we need to figure out how large our buffer needs to be to hold the entire file. While there are slightly more concise ways outside of the standard library to determine the file size, in particular the POSIX `stat` function), this method is easy to understand and allows us to stick with the C standard library. In order to find our file size, we `fseek` to the end (`SEEK_END`) of the file, and then `ftell` to get the position of the file pointer (which is now at the end of the file, effectively giving is our filesize). `fseek` returns 0 on success, so we can use a shorthand in our `if` statement to handle errors. `ftell` returns <0 if an error occurs, and sets `errno`.

Finally, we `rewind` the file pointer to the beginning of the file to prepare for reading it.

```c {linenos=table,linenostart=45}
    // allocate the buffer
    *buf = calloc(filesize + 1, sizeof(char));
    if (*buf == NULL)
    {
        set_err_msg("unable to allocate buffer for file read: %s", strerror(errno));
        goto err_cleanup;
    }
```

Now we allocate and initialize the buffer. `calloc` is included from `stdlib.h`, and takes two arguments: a number of objects, and the size of each object. In order to be extra clear here, we use sizeof(char) even though this is always one byte. We allocate `filesize+1` bytes to ensure we have a valid C string with a null terminator. `calloc` returns a pointer to the _initialized_ buffer (all bytes are set to 0), saving us the step of manually initializing. If calloc returns `NULL` it was not able to allocate the memory. In practice, if this were to happen the system probably wouldn't even be able to get to a point where you could see the error output, but we'll handle the error case for consistency and because it's a good habit.

```c {linenos=table,linenostart=53}
    // read the file into the buffer
    size_t rd = fread(*buf, 1, filesize, f);
    if (rd < filesize)
    {
        if (ferror(f))
        {
            set_err_msg("unable to read entire file: %s", strerror(errno));
            goto err_cleanup;
        }
        set_err_msg("read %d bytes, expected %d", rd, filesize);
        goto err_cleanup;
    }
```

Now we'll read the file into the buffer. `fread` is included from `stdio.h` and takes four arguments: the buffer into which we're reading, the size of each object to read, the number of objects to read, and the file pointer to read from, and returns the number of bytes read. We use this for our error check; if the number of bytes read is less than what we were expecting, we check for a read error with `ferror` and then set our error message based on this. If `ferror` returns 0, there was another issue but we know we don't have the whole file, so we'll still return as an error.

```c {linenos=table,linenostart=66}
    rval = rd;
    goto cleanup;

err_cleanup:
    rval = -1;
    if (*buf != NULL)
        free(*buf);
    *buf = NULL;
cleanup:
    if (f != NULL)
        fclose(f);
    return rval;
}
```

Now that we have completed our logic, we're going to clean up and return. The use of `goto` functions can be controversial in C development, but I find that it offers a sane and easily understandable way to ensure cleanup occurs in all cases.

If we've gotten this far without an error, we can set our return value to the number of bytes read (which is the length of the C string in our buffer), then jump over the error cleanup and do our all-cases cleanup, which is ensuring we've closed our file.

If we've encountered an error at any other point in the function, we'll have jumped to `err_cleanup`. In this case, we set our return value to `-1`, `free` our buffer to ensure there is no memory leaking, and set the buffer pointer back to `NULL`. We then continue to the all-cases cleanup which closes the file.

We now have a straightforward, reusable function for reading each day's input into memory and returning a pointer to the buffer. Maybe now we can look at solving the problem.

### The day

```c {linenos=table}
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "lib.h"

static int part1(const char *input, size_t input_len)
{
    int floor = 0;
    for (int i = 0; i < input_len; i++)
    {
        switch (input[i])
        {
        case '(':
            floor++;
            break;
        case ')':
            floor--;
            break;
        default:
            set_err_msg("invalid input: %c", input[i]);
            return -1;
        }
    }

    return floor;
}

static int part2(const char *input, size_t input_len)
{
    int floor = 0;
    for (int i = 0; i < input_len; i++)
    {
        switch (input[i])
        {
        case '(':
            floor++;
            break;
        case ')':
            floor--;
            break;
        default:
            set_err_msg("invalid input: %c", input[i]);
            return -1;
        }
        if (floor == -1)
            return i + 1;
    }

    set_err_msg("solution not found");
    return -1;
}

int day1(const char *filename)
{
    printf("Day 1: Not Quite Lisp\n");

    int rval = 0;

    char *input;
    int filesize = read_file_to_buffer(&input, filename);
    if (filesize < 0)
        return 1;

    clock_t start = clock();
    int solution = part1(input, filesize);
    clock_t end = clock();
    double duration = ((double)(end - start)) / CLOCKS_PER_SEC;
    if (solution < 0)
    {
        rval = 1;
        goto cleanup;
    }
    char dur_buf[8] = {0};
    format_duration(dur_buf, 8, duration);
    printf("\tPart 1: %d (%s) \n", solution, dur_buf);

    start = clock();
    solution = part2(input, filesize);
    end = clock();
    duration = ((double)(end - start)) / CLOCKS_PER_SEC;
    format_duration(dur_buf, 8, duration);
    if (solution < 0)
    {
        rval = 1;
        goto cleanup;
    }
    printf("\tPart 2: %d (%s)\n", solution, dur_buf);

cleanup:
    free(input);
    return rval;
}
```

There are three functions in our `day` file: one matching the signature `int (const char *)` we utilize in our entrypoint, and two _static_ functions for each of the two parts of the days puzzle. We make these static functions so that they are not exported from the file, allowing us to reuse the identifiers in other files.

At first glance, it seems that the `day1` function is also a good candidate for boilerplate. Unfortunately as we'll see in later days, that is not always the case. In some puzzles, the second part is dependent on the result of the first part, for example.

#### Part 1



