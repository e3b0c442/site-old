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

* We start on the ground floor, which is numbered `0`
* We are given an input with instructions for going up (`(`) and down (`)`)
* The solution is the floor number we're on at the end of the input

Let's get to work.

### Boilerplate... again

When setting up our top-level boilerplate, we defined a function signature for each day's puzzles of `int (const char *)`, where the single argument is the filename for the puzzle input. For this puzzle, we could simply open the file and read directly, but generally speaking reading from the filesystem a byte at a time is much less efficient than reading large chunks, so we will read the whole input file into memory and operate on it in memory.

Unfortunately (?) for us, we're starting with C, and there is no single function in the standard library that will do this for us, so we get to create our own!

Because we don't know the size of our file at compile time, we will need to dynamically allocate memory for this, meaning we need to be able to return a pointer. Easy, right? Remember that in C, the length of an array is not stored with an array, we need to return the length (or force the end-user to trust that we correctly end our C string with the null terminator, which as a consumer of C library rings all the alarm bells). This is a problem for us because we can only return one value in C.

There are a couple of ways we could handle this. We could create a custom `struct` type which includes the length and a pointer to the array. In fact, this is how most high-level programming language handle dynamic arrays under the hood. However, we're going to use a mechanism that is common in C programming: we will return the length of the array, and pass a *pointer to pointer* as an argument. This allows us to pass a memory address into the function wherein we can store *another* memory address that actually points at the data we care about, and have this available once the execution of the function has ended.

We'll first add our declaration to `lib.h`:

```c {linenos=table,linenostart=10}
int read_file_to_buffer(char **buf, const char *filename);
```

Our function will have two arguments: A pointer to a pointer where we can place the pointer to the allocated buffer, and the string with the filename. It will return the length of the buffer. Here's our implementation that will be added to `lib.c`:

```c {linenos=table,linenostart=43}
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

```c {linenos=table,linenostart=43}
    *buf = NULL;
    int rval;
```

The first thing we do is set the *value* behind the provided pointer (so, the pointer being pointed to) to `NULL`. It will remain `NULL` unless the operation succeeds. We also predeclare our return value here.

```c {linenos=table,linenostart=48}
    // open the file
    FILE *f = fopen(filename, "r");
    if (f == NULL)
    {
        set_err_msg("unable to open file: %s", strerror(errno));
        goto err_cleanup;
    }
```

Now we are going to attempt to open the file. `fopen` is included from `stdio.h`, and it returns `NULL` and sets `errno` if the file cannot be opened. If that is the case, we set our own error message with a little more detail using our `set_err_msg` utility, and make a jump to do more cleanup (more on this in a bit).

```c {linenos=table,linenostart=56}
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

```c {linenos=table,linenostart=70}
    // allocate the buffer
    *buf = calloc(filesize + 1, sizeof(char));
    if (*buf == NULL)
    {
        set_err_msg("unable to allocate buffer for file read: %s", strerror(errno));
        goto err_cleanup;
    }
```

Now we allocate and initialize the buffer. `calloc` is included from `stdlib.h`, and takes two arguments: a number of objects, and the size of each object. In order to be extra clear here, we use sizeof(char) even though this is always one byte. We allocate `filesize+1` bytes to ensure we have a valid C string with a null terminator. `calloc` returns a pointer to the *initialized* buffer (all bytes are set to 0), saving us the step of manually initializing. If calloc returns `NULL` it was not able to allocate the memory. In practice, if this were to happen the system probably wouldn't even be able to get to a point where you could see the error output, but we'll handle the error case for consistency and because it's a good habit.

```c {linenos=table,linenostart=78}
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

```c {linenos=table,linenostart=91}
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

```c
#include <limits.h>
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
            return INT_MIN;
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

    int rval = 11;

    char *input;
    int filesize = read_file_to_buffer(&input, filename);
    if (filesize < 0)
        return rval;

    clock_t start = clock();
    int solution = part1(input, filesize);
    clock_t end = clock();
    double duration = ((double)(end - start)) / CLOCKS_PER_SEC;
    if (solution == INT_MIN)
        goto cleanup;

    char dur_buf[8] = {0};
    format_duration(dur_buf, 8, duration);
    printf("\tPart 1: %d (%s) \n", solution, dur_buf);

    start = clock();
    solution = part2(input, filesize);
    end = clock();
    duration = ((double)(end - start)) / CLOCKS_PER_SEC;
    format_duration(dur_buf, 8, duration);
    if (solution < 0)
        goto cleanup;

    printf("\tPart 2: %d (%s)\n", solution, dur_buf);
    rval = 0

cleanup:
    free(input);
    return rval;
}
```

There are three functions in our `day` file: one matching the signature `int (const char *)` we utilize in our entrypoint, and two *static* functions for each of the two parts of the days puzzle. We make these static functions so that they are not exported from the file, allowing us to reuse the identifiers in other files.

At first glance, it seems that the `day1` function is also a good candidate for boilerplate. Unfortunately as we'll see in later days, that is not always the case. In some puzzles, the second part is dependent on the result of the first part, for example.

Let's walk through what we're doing. We won't rehash this for subsequent days, unless there are changes.

```c
int day1(const char *filename)
{
    printf("Day 1: Not Quite Lisp\n");
```

The first thing we do is print the puzzle's title. We pull this directly from the puzzle page, and it's just to keep the output sane.

```c
    int rval = 1;
```

Next we'll predeclare our return value. This isn't strictly necessary except in ANSI C, but having the variable declared and initialized to an error value allows us to use our cleanup jump, and saves some lines of code later.

```c
    char *input;
    int filesize = read_file_to_buffer(&input, filename);
    if (filesize < 0)
        return rval;
```

Now we'll read our input file. We declare a pointer, and pass a pointer to that pointer to the `read_file_to_buffer` function, which will allocate and change the value of the pointer to point at the allocated string. `read_file_to_buffer` returns `<0` on failure, so we'll check for that and return an error value if the read fails.

```c
    clock_t start = clock();
    int solution = part1(input, filesize);
    clock_t end = clock();
    double duration = ((double)(end - start)) / CLOCKS_PER_SEC;
    if (solution == INT_MIN)
        goto cleanup;

    char dur_buf[8] = {0};
    format_duration(dur_buf, 8, duration);
    printf("\tPart 1: %d (%s) \n", solution, dur_buf);
```

These next few lines of code look pretty familiar from our entrypoint boilerplate. We take a monotonic clock reading, run the first part of the puzzle passing in the input string and the file size and returning the solution, and then take another clock reading, using the two readings to determine the duration. For this part of the problem, since the answer could ostensibly be either negative or positive, we use a *canary value* of the `INT_MIN` constant which is defined in `limits.h` to indicate an error. Rather than immediately returning on error, we jump to cleanup -- we'll see why shortly.

If our solution is good, we declare a short character buffer to hold the formatted duration and pass that along with the duration to the `format_duration` function in our library. Then we print the solution along with the duration.

```c
    start = clock();
    solution = part2(input, filesize);
    end = clock();
    duration = ((double)(end - start)) / CLOCKS_PER_SEC;
    format_duration(dur_buf, 8, duration);
    if (solution < 0)
        goto cleanup;

    printf("\tPart 2: %d (%s)\n", solution, dur_buf);
```

The second part is almost identical to the first part. The differences are that we do not need to redeclare the duration format buffer, and that for part two we can use a negative value to determine an error state (we'll see why shortly).

```c
    rval = 0;

cleanup:
    free(input);
    return rval;
}
```

If we've made it to this point, there were no errors so we set our return value to a non-error value and continue to the cleanup.

Our `read_file_to_buffer` function allocates and returns a character array on the heap. Remember that in C, heap memory management is our responsibility; therefore, we must `free` the allocated buffer or risk leaking memory.

Finally, we return the `rval`. Structuring the function in this way allows us to save a lot of repeated code on error cases, and reuse the variable regardless of the error state of the function, making for a more concise (and in my opinion readable) function.

Now that we're finally done with the boilerplate, let's solve some puzzles!

#### Part 1

It's been awhile since we looked at the puzzle, so let's recap what we distilled from the problem statement:

* We start on the ground floor, which is numbered `0`
* We are given an input with instructions for going up (`(`) and down (`)`)
* The solution is the floor number we're on at the end of the input

Solving this puzzle is relatively easy. We'll need to set an initial value of `0`, iterate over the input, and mutate the value accordingly. Here's our solution:

```c
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
            return INT_MIN;
        }
    }

    return floor;
}
```

Let's break this down.

```c
    int floor = 0;
```

First, we declare a variable that holds our solution value, and set it to `0` as specified in the puzzle.

```c
    for (int i = 0; i < input_len; i++)
    {
```

Now we set up a `for` loop. Remember that a string is simply an array of `char`. Remember also that arrays are zero-indexed, so the first element of `input` has an index of `0`, and the last element has an index of `input_len - 1`. 

A `for` loop declaration has three clauses: an initialization, a predicate, and an increment. First, we initialize our loop variable to `0`, which is the first index of the input array. Note that in ANSI C, we must predeclare this variable outside the for loop.

Next, we set our predicate; that is, our requirement that must be met for the loop to continue executing. We state that `i` must be less than `input_len`, which holds because the last index of the input array is `input_len - 1`.

Finally, we set our increment. In this case it is simple, just add `1` to `i` for each loop, allowing us to operate on every element of the array.

```c
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
            return INT_MIN;
        }
```

Here, we operate on each element in the array. While it would be appropriate to use `if...else` here, I prefer to use a `switch` statement for readability. `switch` takes an input value and chooses where to `begin` executing based on that value. So in our case, if the character is `(`, we increment `floor` as specified in the problem statement. We then have a `break` statement, which ends execution of the `switch`. This is important; as I said, the value comparison determines where to `begin` executing; if not for the `break`, the switch statement would continue executing the next case. This is a very powerful behavior, but also tends to catch new C programmers off-guard.

Continuing on, if the character is `)`, we decrement `floor`, again as specified in the problem statement.

Finally, we have a case labeled `default`, which is matched if no other case matches the switch. Since we are expecting our input to only have `(` or `)`, we consider it an error case if another character is encountered, and so return our error canary value of `INT_MIN`. 

```c
    }

    return floor;
}
```

At the end of our loop, `floor` contains the floor number we are on at the end of the input, so we return this value. When we enter this solution into the Advent of Code solution page, we are presented with part 2.

### Part 2

> Now, given the same instructions, find the *position* of the first character that causes him to enter the basement (floor `-1`). The first character in the instructions has position `1`, the second character has position `2`, and so on.
>
> For example:
>
> * `)` causes him to enter the basement at character position `1`.
> * `()())` causes him to enter the basement at character position `5`.
>
> What is the *position* of the character that causes Santa to first enter the basement?

This puzzle is subtly different from part 1, as most Advent of Code puzzles are. In this problem, rather than iterating the whole input, the solution is how far along we are in the input when we hit a condition. Here's our solution.

```c {linenos=table,linenostart=30}
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
```

The solution looks very similar to the first part, so we won't break it down as we did with part 1, rather, we'll look at the differences. First, you'll notice our error returns are `-1` instead of `INT_MIN`. This is acceptable because our return value must be `>0`.

Secondly, notice that we are checking for a condition after each switch statement -- specifically, we're looking for `floor == -1`, which indicates we are in the basement. If we encounter that, then we've found the solution to our puzzle, `i + 1`, and return that. Note that the problem statement specifies that the first input character has value `1`. Since our arrays are zero-indexed, we need to add `1` to the index to get the actual puzzle solution.

Finally, if we get all the way through the array and have never hit `floor == -1`, then there is no solution to our puzzle, and we set the error message and return appropriately.

### Conclusion

We've coded the solutions, now we need to build and run them. Using our Makefile, we can easily clean and build the specific day, then run it:

```console
$ make clean
rm -rf obj
rm -rf bin
$ make day1
mkdir -p obj
mkdir -p bin
gcc -c -o obj/day1.o src/day1.c -Wall -g -pedantic -std=c17 -Og
gcc -c -o obj/mainday1.o src/main.c -D DAYNUM=day1 -Wall -g -pedantic -std=c17 -Og
gcc -c -o obj/lib.o src/lib.c -Wall -g -pedantic -std=c17 -Og
gcc -o bin/day1 obj/day1.o obj/mainday1.o obj/lib.o -Wall -g -pedantic -std=c17 -Og 
$ bin/day1 ../1.txt 
Day 1: Not Quite Lisp
        Part 1: 232 (0.081ms) 
        Part 2: 1783 (0.021ms)
        Completed in 0.001s
$
```
You should see similar output.

So, evaluating where we are at. We were able to complete this problem in C, but there was a significant amount of boilerplate and setup required. That said, the problems ran reasonably fast, less than a millisecond on a fresh compilation on an M1 Mac mini. In subsequent chapters, we're going to explore a few more modern and/or high-level languages, build the solutions in those languages, and discuss the tradeoffs. Until then, have fun coding!