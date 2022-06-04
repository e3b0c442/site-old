---
title: "Advent of Code 2015 Day 1: Rust"
date: 2022-06-02T15:11:21.470Z
draft: false
featured: false
image:
  filename: featured
  focal_point: Smart
  preview_only: false
---
We're nearly at the end of our day 1 implementations. Now that we have set up our Rust environment, let's dig into the code.

# The Day
```rust {linenos=table}
use simple_error::{bail, SimpleError};
use std::error::Error;
use std::fs::File;
use std::io::prelude::*;
use std::time::Instant;

pub fn day1(input_file: &str) -> Result<(), Box<dyn Error>> {
    println!("Day 1: Not Quite Lisp");

    let mut f = File::open(input_file)?;
    let mut input = String::new();
    f.read_to_string(&mut input)?;

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
    Floor(i32),
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
                    Err(Basement::Floor(step as i32))
                } else {
                    Ok(floor)
                }
            }
            _ => Err(Basement::Err(SimpleError::new("Invalid input").into())),
        }) {
        Err(Basement::Floor(step)) => Ok(step),
        _ => bail!("Solution not found"),
    }
}
```

At the risk of sounding like a broken record, we have three functions in this module: the day's main function `day1`, which is exported via the `pub` keyword, and the `part1` and `part2` functions which are private to the module.

```rust {linenos=table}
use simple_error::{bail, SimpleError};
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

