---
title: "Advent of Code 2015 Day 1: Rust"
date: 2022-06-02T15:11:21.470Z
draft: false
featured: false
tags:
  - advent-of-code
  - advent-of-code-2015
  - rust
image:
  filename: featured
  focal_point: Smart
  preview_only: false
---
_The code for these articles can be found at [https://github.com/e3b0c442/aoc](https://github.com/e3b0c442/aoc)._

We're nearly at the end of our day 1 implementations. Now that we have set up our Rust environment, let's dig into the code.

# The Day
```rust {linenos=table}
use simple_error::{bail, simple_error};
use std::error::Error;
use std::fs;
use std::time::Instant;

pub fn day1(input_file: &str) -> Result<(), Box<dyn Error>> {
    println!("Day 1: Not Quite Lisp");

    let input = fs::read_to_string(input_file)?;

    let start = Instant::now();
    println!("\tPart 1: {} ({:?})", part1(&input)?, start.elapsed());
    let second = Instant::now();
    println!("\tPart 2: {} ({:?})", part2(&input)?, second.elapsed());
    println!("\t\t Completed in {:?}", start.elapsed());
    Ok(())
}

fn part1(input: &str) -> Result<i32, Box<dyn Error>> {
    input.chars().try_fold(0, |floor, c| match c {
        '(' => Ok(floor + 1),
        ')' => Ok(floor - 1),
        _ => bail!("Invalid input: {}", c),
    })
}

#[derive(Debug)]
enum Basement {
    Step(i32),
    Err(Box<dyn Error>),
}
impl std::error::Error for Basement {}
impl std::fmt::Display for Basement {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "")
    }
}

fn part2(input: &str) -> Result<i32, Box<dyn Error>> {
    match input
        .chars()
        .enumerate()
        .try_fold(0, |mut floor, (step, c)| match c {
            '(' => Ok(floor + 1),
            ')' => {
                floor -= 1;
                if floor < 0 {
                    Err(Basement::Step(step as i32))
                } else {
                    Ok(floor)
                }
            }
            _ => Err(Basement::Err(simple_error!("Invalid input").into())),
        }) {
        Err(Basement::Step(step)) => Ok(step),
        Err(Basement::Err(err)) => Err(err),
        _ => bail!("Solution not found"),
    }
}

```

At the risk of sounding like a broken record, we have three functions in this module: the day's main function `day1`, which is exported via the `pub` keyword, and the `part1` and `part2` functions which are private to the module.

```rust {linenos=table}
use simple_error::{bail, simple_error};
use std::error::Error;
use std::fs;
use std::time::Instant;
```

Our preamble lists the imports. We already saw most of these in the setup, but I'll touch on a couple of new ones.

* `simple_error::bail` is a shorthand macro for constructing and returning a `SimpleError`.
* `std::io::prelude::*` is a _prelude_ import. If you'll remember during our setup, we briefly discussed the standard library prelude, which is a set of imports that are implicitly available to every Rust module. The idea of the prelude has been extended to other parts of the standard library and to other crates as well. `std::io::prelude` contains commonly used imports for programs that read and write input and output. Rather than import each of the needed items individually, it is convenient to use the prelude in our case.

```rust {linenos=table,linenostart=6}
pub fn day1(input_file: &str) -> Result<(), Box<dyn Error>> {
    println!("Day 1: Not Quite Lisp");
```

This is our function declaration for the day main function. As previously noted, the `pub` keyword exports the function from the module and makes it available to other modules. Note that since we are not using a `mod` block, the file is implicitly understood to be a module of the same name as the stem of the file (so in our case, `day1`). We could use a `mod` block to change the name, or multiple blocks to add multiple modules to a file or spread modules between files, however I prefer to maintain one module per file if feasible.

Our function takes an input filename as a reference to a string slice, and returns `Result`, with the `Ok` variant containing the unit type, and `Err` variant containing the heap-allocated error.

Following the same logic as with the other languages, we print the puzzle day and name to the output. `println!` is a macro provided by the standard library for printing to the standard output.

```rust {linenos=table,linenostart=9}
    let input = fs::read_to_string(input_file)?;
```

Next, we read our input file into a string. Rust has provided a convenience function for this, `fs::read_to_string`. This returns `io::Result<String>`. `io::Result` is a shorthand result type; since all `io` functions return `io::Error` for the error variant, it isn't necessary to specify it as a parameter on each `io` function. As we have seen previously, we use the shorthand `?` to immediately return an `Err` variant or to unwrap an `Ok` variant into the variable. Note that this is why we use `Box<dyn Error>` for the Err variant in our functions instead of `SimpleError`. We would need to add additional code to map the error, and we would lose the original error, neither of which I consider to be good practice.

```rust {linenos=table,linenostart=11}
    let start = Instant::now();
    println!("\tPart 1: {} ({:?})", part1(&input)?, start.elapsed());
    let second = Instant::now();
    println!("\tPart 2: {} ({:?})", part2(&input)?, second.elapsed());
    println!("\t\t Completed in {:?}", start.elapsed());
    Ok(())
}
```

Similar to the other languages, we take timestamps, run our puzzle logic, and then print the solutions along with the elapsed time. The `println!` macro accepts a format string and a number of arguments equal to the number of parameters in the format string. In Rust, `{}` represents a format parameter, which may have additional information inside. Notice that the second parameter has `:?` inside; this tells the formatter to use the `Debug` formatter instead of the `Display` formatter. Any argument assigned to a `{:?}` parameter must implement `fmt::Debug`.

Once the function has reached the end without encountering an error, the `Ok(())` statement sets the automatic return value.

As with the other days, we expect this function to change minimally across the days, but enough that we can't factor it out.

## Part 1

```rust {linenos=table,linenostart=19}
fn part1(input: &str) -> Result<i32, Box<dyn Error>> {
    input.chars().try_fold(0, |floor, c| match c {
        '(' => Ok(floor + 1),
        ')' => Ok(floor - 1),
        _ => bail!("Invalid input: {}", c),
    })
}
```

Surprise! After three days of very similar implementations, our Rust implementation is completely different. Rust's power and flexibility enable functional programming to a degree the other  languages don't _(ed note: Yes, I know Python has functional programming support, but it isn't as idiomatic as it is with JavaScript or Rust)_, so we are going to take advantage of iterators and make a concise solution.

Remember our solution is the floor on which we arrive at the end of the instructions.

Rather than taking this line-by-line, we're going to go expression-by-expression.

```rust
fn part1(input: &str) -> Result<i32, Box<dyn Error>> {
```
Our part function takes a string slice argument with the input, and returns a `Result` that matches the call in the exported function.

```rust
input.chars()
```

`str::chars()` returns an `Iterator` over the characters in `str`.

```rust
.try_fold(0, |floor, c| 
```

`Iterator::try_fold()` takes two arguments: an initialization value (we start at floor 0), and a closure with two parameters: a _mutable_ running value, and the current character. The closure is called once per item in the iterator (so in our case, once per character), and returns a `Result`. If the `Result` is `Err`, the iteration immediately stops and propogates the error. Otherwise, the `Ok` value is populated with the solution.

```rust
match c {
    '(' => Ok(floor + 1),
    ')' => Ok(floor - 1),
    _ => bail!("Invalid input: {}", c),
}
```
The body of our closure is a simple match statement. If the character is `(`, we return `floor + 1` (which becomes the `floor` argument on the next iteration, or the solution if the iterator is exhausted); if the character is `(`, we return `floor - 1`. If any other character is encountered, we use the `bail!` macro from `simple-error` to immediately return a `SimpleError` with the formatted message.

When the iterator completes, it returns a `Result` of the same type as our function, so we can just use the statement value as the automatic return. 

## Part 2

```rust {linenos=table,linenostart=27}
#[derive(Debug)]
enum Basement {
    Step(i32),
    Err(Box<dyn Error>),
}
impl std::error::Error for Basement {}
impl std::fmt::Display for Basement {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "")
    }
}

fn part2(input: &str) -> Result<i32, Box<dyn Error>> {
    match input
        .chars()
        .enumerate()
        .try_fold(0, |mut floor, (step, c)| match c {
            '(' => Ok(floor + 1),
            ')' => {
                floor -= 1;
                if floor < 0 {
                    Err(Basement::Step(step as i32))
                } else {
                    Ok(floor)
                }
            }
            _ => Err(Basement::Err(simple_error!("Invalid input").into())),
        }) {
        Err(Basement::Step(step)) => Ok(step),
        Err(Basement::Err(err)) => Err(err),
        _ => bail!("Solution not found"),
    }
}
```

Ok, there's a bit more going on for part 2. As a reminder, we are looking for the one-based index of the step when we enter a basement floor (<`0`). In order to achieve this using an iterator, we are going to use an `enum` to contain either an error value or the value of the floor.

```rust {linenos=table,linenostart=27}
#[derive(Debug)]
enum Basement {
    Step(i32),
    Err(Box<dyn Error>),
}
impl std::error::Error for Basement {}
impl std::fmt::Display for Basement {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        write!(f, "")
    }
}
```

Our `enum` has two variants: a `Floor` variant which contains the step number where we hit the basement, and an `Err` variant which wraps an actual error. In order to short-circuit the iterator, we need to return an `Err` variant on our `Result`, and this allows us to do this while distinguishing between the value we're trying to get and an actual error. 

The remaining lines are all necessary to have a type which satisfies the `std::error::Error` trait; specifically, it must implement both the `std::fmt::Debug` and `std::fmt::Display` traits. We can use the `#[derive(Debug)]` attribute to automatically implement `Debug`, but we must manually implement `Display`.

This is done using an `impl` block. For any trait we are promising to satisfy with our type, we need an `impl` block for that trait. In the case of `std::error::Error`, since there are no methods outside of the `Debug` and `Display` traits that need to be implemented, we can use an empty block. 

Since we don't care about displaying a message (no matter what we're unwrapping the underlying value and propagating that), we just have a simple implementation of the `fmt` method that writes an empty string.

This code satisfies `std::error::Error` so we can use our type anywhere `Error` is accepted.

Now that we have our types, we'll walk through statement by statement again.

```rust
match
```

Since we are ultimately looking for a specific variant of an `enum`, we will use a match statement to branch on that variant.

```rust
input.chars()
```

As in part 1, this returns an `Iterator` over the characters in the input string.

```rust
.enumerate()
```

This method returns a new iterator based on the original, which yields a tuple of the index and the value. We need the index to track our position for the puzzle solution.

```rust
try_fold(0, |mut floor, (step, c)|
```

As before, we'll use `try_fold` so that we can short-circuit on error. Note that the second argument _deconstructs_ the tuple into discrete variables that can be accessed in the closure.

```rust
match c {
    '(' => Ok(floor + 1),
    ')' => {
        floor -= 1;
        if floor < 0 {
            Err(Basement::Step(step as i32))
        } else {
            Ok(floor)
        }
    }
    _ => Err(Basement::Err(simple_error!("Invalid input").into())),
}
```

Our in-closure match statement is similar to part 1, but there are two major differences. In the `)` arm, we check to see if we've entered the basement, and if so, return the `Err` variant with `Basement::Step` populated with our position in the input. In our `_` arm, we return a `Basement::Err` populated with our SimpleError (which we use the convenience macro `simple_error!` to create). The `into()` method `Box`es the `SimpleError` so that it correctly matches our return type.

```rust
) {
        Err(Basement::Step(step)) => Ok(step),
        Err(Basement::Err(err)) => Err(err),
        _ => bail!("Solution not found"),
    }
}
```
Now that we have a value from `try_from`, we can check the value with our outer `match` statement. If the returned Result matches `Err(Basement::Step)`, the value of the `match` statement is a `Result` with the `Ok` variant containing the step number. If `Err(Basement::Err)` matches, then the value of the `match` statement is a `Result` with the `Err` variant containing the error message. Finally, if anything else is returned, we immediately `bail` with "Solution not found", as we would only encounter this branch if the input ran to the end  with no solution.

There is no semicolon following the match statement, so the value of the expression is automatically returned.

# Conclusion
Rust is a very unique and powerful language. Unlike Go, there are frequently many ways to do things. We absolutely could have implemented our Rust solution using loops like our previous implementations. However, with Rust, we can have the advantage of the conciseness of the functional syntax without a performance impact. 

```console
$ cargo build
$ target/debug/day1 ../1.txt
Day 1: Not Quite Lisp
        Part 1: 232 (442.333µs)
        Part 2: 1782 (122.666µs)
                 Completed in 576.125µs
```

This is very similar to Go and actually quite a bit faster than our C debug build. If we do a release build, things get significantly faster:

```console
$ cargo build --release
$ target/release/day1 ../1.txt
Day 1: Not Quite Lisp
        Part 1: 232 (38.083µs)
        Part 2: 1782 (10.666µs)
                 Completed in 75.375µs
```

This is quite a bit faster than even an optimized C build.

The only "problem" with Rust is that it is also by far the most difficult language we're testing. As noted, there are a lot of different ways to do the same thing in Rust. There is also a lot of grammar and syntax to learn, and the concept of variable ownership comes to the forefront. The tradeoff of course is highly performant code that is guaranteed (barring use of `unsafe` of course) to not have any undefined behavior.

We've completed our Day 1 series. I'd love to hear your comments about how things are going so far. My subsequent articles will likely lump the different language implementations for each day into one article, to improve ease of comparison and reduce verbosity.