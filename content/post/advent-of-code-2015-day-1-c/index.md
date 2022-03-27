---
title: "Advent of Code 2015 Day 1: C"
date: 2022-03-27T02:24:14.468Z
draft: true
featured: false
image:
  filename: featured
  focal_point: Smart
  preview_only: false
---
Welcome to my kickoff of Advent of Code 2015.

Now, before we start any coding in C, we need to do some setup. Unlike the other languages we use, C does not have a built-in toolchain, so we need to do a little housekeeping to make sure we can build our project.

Since I live and work in a UNIX-based world, we'll be using GNU Make to build our project. I won't be going too far into detail in discussing this, except to note that we are trying to build binaries for each individual day as well as a binary that will run all days, and trying to do so in a fashion that will allow the fewest updates to the `Makefile` with each day.

Here's our `Makefile`:

```makefile {title=Makefile}
CC 					:= gcc
ifdef OPT
override CCFLAGS	+= -Wall -pedantic -std=c17 -O3
else
override CCFLAGS	+= -Wall -g -pedantic -std=c17 -Og
endif
override LDFLAGS	+= 


DAYNUMS 	:= 1 
DAYS		:= $(addprefix day, $(DAYNUMS))
DAY_BINS	:= $(addprefix bin/, $(DAYS))
DAY_OBJS	:= $(addprefix obj/, $(addsuffix .o, $(DAYS)))
DAY_MAINS	:= $(addprefix obj/main, $(addsuffix .o, $(DAYS)))
LIBS		:= common
LIB_OBJS    := $(addprefix obj/, $(addsuffix .o, $(LIBS)))

.PHONY: clean all aoc2015 $(DAYS)

all: aoc2015 $(DAYS)

aoc2015: obj bin bin/aoc2015

$(DAYS): % : obj bin bin/%

$(DAY_BINS): bin/% : obj/%.o obj/main%.o $(LIB_OBJS)
	$(CC) -o $@ $^ $(CCFLAGS) $(LDFLAGS)

$(DAY_MAINS): src/main.c
	$(CC) -c -o $@ $< -D DAYNUM=$(@:obj/main%.o=%) $(CCFLAGS)

bin/aoc2015: obj/aoc2015.o $(LIB_OBJS) $(DAY_OBJS)
	$(CC) -o $@ $^ $(CCFLAGS) $(LDFLAGS)

obj/aoc2015.o: src/main.c
	$(CC) -c -o $@ $< $(CCFLAGS)

$(DAY_OBJS): obj/%.o : src/%.c
	$(CC) -c -o $@ $< $(CCFLAGS)

$(LIB_OBJS): obj/%.o : src/%.c
	$(CC) -c -o $@ $< $(CCFLAGS)

obj:
	mkdir -p obj

bin:
	mkdir -p bin

clean:
	rm -rf obj
	rm -rf bin
```
The gist of this is, we track any common build flags and libraries that need to be added to the build up at the top, and then use some `make` functions to build variables based on the day name. This allows us to only come back in and update the `DAYNUMS` variable for each day we build. This `Makefile` allows us to build everything, or just a specific day. This setup also allows us to factor out reusable code into library files that are not specific to a day.

Now that we have our build setup, we need to add a little boilerplate. In particular, we need to set up our entrypoint for the builds, and a header file from which we can include the code for each day's puzzles.

```c {title=main.c}
#include <stdio.h>
#include <string.h>
#include <time.h>
#include "days.h"
#include "lib.h"

#ifdef DAYNUM
int main(int argc, char const *argv[])
{
    if (argc < 2)
    {
        fprintf(stderr, "%s\n", "must provide path to input file");
        return 1;
    }

    clock_t start = clock();
    int rval = DAYNUM(argv[1]);
    clock_t end = clock();
    double duration = ((double) (end - start)) / CLOCKS_PER_SEC;

    if (rval)
    {
        fprintf(stderr, "%s\n", err_msg);
        return 1;
    } else {
        printf("\tCompleted in %gms\n", duration * 1000.0);
    }

    return 0;
}
#else // #ifdef DAYNUM
typedef int (*day_f)(const char *);

static const int days_len = 11;
static const day_f days[] = {
    day1,
};
static char path[FILENAME_MAX];

int main(int argc, char const *argv[])
{
    if (argc < 2)
    {
        fprintf(stderr, "%s\n", "must provide path to input files");
        return 1;
    }

    if (strlen(argv[1]) > FILENAME_MAX - 8)
    {
        fprintf(stderr, "%s\n", "input path too long");
        return 1;
    }

    for (int i = 0; i < days_len; i++)
    {
        sprintf(path, "%s/%d.txt", argv[1], i + 1);
        int rval = days[i](path);
        if (rval)
        {
            fprintf(stderr, "%s\n", err_msg);
            return rval;
        }
    }

    return 0;
}
#endif // #ifdef DAYNUM
```

Let's walk through this file one chunk at a time.

```c 
#include <stdio.h>
#include <string.h>
#include "days.h"
#include "lib.h"
```

At the top of the file, we have our includes. We'll discuss why these includes are necessary as we encounter code that uses the included functions.

```c
#ifdef DAYNUM
```

Because we want to keep a single entrypoint file, we are going to use `#ifdef` macros to determine which code should be compiled for a particular set of conditions. The code between `#ifdef DAYNUM` and `#else // #ifdef DAYNUM` (note I'm using comments to help keep track of which `#ifdef` the `#else` belongs to) will be compiled in the steps which are building the executable for a single day's puzzle. The code between `#else // #ifdef DAYNUM` and `#endif // #ifdef DAYNUM` will be compiled in the step where we are building the executable to run all the puzzles. `DAYNUM` is defined in the `Makefile` for the appropriate steps, and is set to a string `day<#>` where `<#>` is replaced with the day number we are building.

```c
int main(int argc, char const *argv[])
{
```

The entrypoint for a single-day executable. As is standard, our `main` function has two arguments, `argc` which is the total count of arguments passed (including the executable itself), and `argv` which is an array of strings representing the arguments.

```c
    if (argc < 2)
    {
        fprintf(stderr, "%s\n", "must provide path to input file");
        return 1;
    }
```

Here, we have some input validation. Any single day's executable requires a single argument which contains the path to the input file for that day. If that argument is not provided, we return with a non-zero exit code. In C, the return value of the `main` function is the exit code provided to the operating system.

```c
    clock_t start = clock();
    int rval = DAYNUM(argv[1]);
    end = clock();
    duration = ((double) (end - start)) / CLOCKS_PER_SEC;

```

Here is the meat of our function. We first get a monotonic time from the operating system by calling the `clock` function included from `time.h`. We then run the function for the day we are building. Notice the use of the `DAYNUM` macro; this is replaced by the actual function name, e.g. `day1` for day 1, allowing us to use the same entrypoint for each day's puzzle even if the functions are named differently. Finally, we take another monotonic time at the end, find the difference, and divide by `CLOCKS_PER_SEC` -- also included from `time.h` -- to determine how many seconds were required to execute the puzzle function.

```c
    if (rval)
    {
        fprintf(stderr, "%s\n", err_msg);
        return 1;
    } else {
        printf("\tCompleted in %gms\n", duration * 1000.0);
    }

    return 0;
}
```

We wrap up our single-day entrypoint by checking the return value of the called function. Using the standard C convention, we'll return a non-zero value if there was an error in the function. If this occurs, we print an error message that has been stored in `err_msg`, included from `lib.h` -- more on this in a bit -- and return non-zero to the operating system. Otherwise we simply return 0 to note that the execution was successful.

Now, let's look at the multi-day entrypoint on the other side of `#else // #ifdef DAYNUM`.

```c
#else // #ifdef DAYNUM
typedef int (*day_f)(const char *);
```

The first thing we do is make a type alias of pointer to a function with the signature each day\* function is going to have, in this case `int (const char*)`, meaning each day's function is going to be passed a C string and return an `int`.

```
static const int days_len = 1;
static const day_f days[] = {
    day1,
};
```

Then we define the array of functions along with a length, remembering that C arrays only store a pointer to the front of the array, and do not store the length. Each time we add a day, we'll need to 
