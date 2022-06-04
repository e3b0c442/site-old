---
title: "Advent of Code 2015: Rust setup"
date: 2022-06-02T04:44:51.242Z
draft: false
featured: false
image:
  filename: featured
  focal_point: Smart
  preview_only: false
---
_The code for these articles can be found at [https://github.com/e3b0c442/aoc](https://github.com/e3b0c442/aoc)._

Let's get our repo set up for the Rust side of our project.

The astute reader may have noticed that I've titled this post "setup" and not "boilerplate". This is because our setup will obviate the need for boilerplate! Honestly it's quite similar to the code generation setup Go uses, except it's using a Rust _build script_, which does not place the generated source in the repo like our Go generator does.

## Cargo

`cargo` is Rust's first-class build and dependency manager. One _can_ run the Rust compiler individually, but it generally doesn't make sense to. The first thing we are going to do is run `cargo new aoc2015` to make our package. This creates a simple hello world application with the necessary files. In particular, we're interested in `Cargo.toml`. Here's what the generated file looks like:

```toml {linenos=table}
[package]
name = "aoc2015"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
```

This file includes our package metadata and list of dependencies. While Rust does have a good standard library, it does not cover as much functionality as the Go standard library so we will need to use some external dependencies. We are going to immediately add to this file to include a few dependencies (which we'll cover in a bit):

```toml {linenos=table}
[package]
name = "aoc2015"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
simple-error = "0.2"

[build-dependencies]
tinytemplate = "1"
serde = { version = "1.0", features = ["derive"] }
```

We've added `simple-error`to our dependencies, and also added a new `build-dependencies` section with `tinytemplate` and `serde`. For the former two, we've used a simplified value which is just a version number. This behaves as a `*.x`, so for `simple-error` the latest `0.2.x` version will be retrieved, for tinytemplate the latest `1.x` version. This allows you to pin to the extent you feel is appropriate.

We'll discuss what each of these dependencies is for as we encounter them.

## src/main.rs

The other file generated by `cargo new` is `src/main.rs`. This is exactly what you think it might be: the entrypoint for the program. We're going to replace the contents of this file as follows:

```rust {linenos=table}
use simple_error::SimpleError;
use std::env;
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> {
    let args = env::args().collect::<Vec<String>>();
    if let Some(path) = args.get(1) {
        return aoc2015::run_all(path);
    }
    Err(Box::new(SimpleError::new("No input path provided")))
}
```

Let's break this down.

```rust {linenos=table,linenostart=1}
use simple_error::SimpleError;
use std::env;
use std::error::Error;
```

The preamble to our file has our imports. Rust has a set of imports called the _prelude_ which are imported into every file. Anything one wishes to use that is not in the prelude must be imported. For our entrypoint, we are using:
* the `env` module from the `std` crate. `std` contains the Rust standard library, while `env` contains functions to inspect the process environment;
* the `Error` trait from the `error` module in `std`. `error` only contains `Error`, which is a _trait_ that any type must implement to be treated as an error.
* the `SimpleError` type from the `simple_error` crate, which is an implementation of the `Error` trait on the `String` type. We'll use this (as well as some of the supporting _macros_) throughout our implementations, since we are generally looking to report any errors as text. Notice that `simple_error` corresponds to the `simple-error` dependency we added to `Cargo.toml`.

```rust {linenos=table,linenostart=5}
fn main() -> Result<(), Box<dyn Error>> {
```

This is the function declaration for our `main` function, which like C and Go is the entrypoint for the program. Spelled out, this line defines a function `main` that has no arguments, and returns a `Result<(), Box<dyn Error>>`

The `Result` type is very commonly used in Rust. It is a _generic enum_, and is more correctly noted as `Result<T,E>`. Rust `enum`s are very powerful, as one can assign values of arbitrary types to them. The `Result<T, E>` enum has two variants: `Ok(T)` and Err(E)`. `T` and `E` have no bounds, and thus can be satisfied by any type. In our case, our `Ok` variant is the Rust `unit` type `()`, and our `Err` variant is `Box<dyn Error>`, which is simply speaking a heap-allocated container for any type which implements the `Error` trait. We'll see how we use these in just a bit.

```rust {linenos=table,linenostart=6}
    let args = env::args().collect::<Vec<String>>();
```

This line retrieves the command line arguments and `collect`s them into a `Vec<String>`. `env::args()` returns an _iterator_, rather than a collection. Using `collect`, we can consume the iterator to create a function of the specified type. The `::<>` syntax is colloquially called the _turbofish_, and is not frequently used – only where the types cannot be inferred by the compiler.

`Vec` is the dynamically-allocated growable array type provided by the Rust standard library. We use this instead of a nominal array because we don't know how many arguments will be provided at compile time.

```rust {linenos=table,linenostart=7}
    if let Some(path) = args.get(1) {
        return aoc2015::run_all(path);
    }
    Err(Box::new(SimpleError::new("No input path provided")))
}
```

This stanza retrieves the path argument from the provided command and calls the top level `run_all` (yet to be defined) function from our crate using the argument if it is provided. `args.get()` returns a `Option` `enum`, which is similar to the `Result` enum we just discussed. `Option` has two variants: `None` and `Some(T)`, where `T` has no bounds. `if let` is a shorthand construct for checking if a returned `Option` returns `Some` and extracting the value. If the assigned value is the `Some` variant, the value from the `Option` is placed into the `path` variable and the code inside the braces is executed. If the assigned value is the `None` variant, the code in the braces is not executed. This gives us a very clean way to check that the argument has been provided. `run_all` returns `Result<(), Box<dyn Error>>` as well so we will directly return the result if we execute; if not, we return the `Err` variant for our result, which is a `Box`ed `SimpleError`. Note that line 10 does not start with `return`. Also note that there is no semicolon. Rust automatically returns the value of the last expression in a function. A semicolon literally takes the value of the previous expression, discards its, and evaluates to `()`. By omitting the semicolon, the last expression is `Err(Box::new(SimpleError::new("No input path provided")))`, and is thus returned.

So far, we have a basic _binary crate_. This crate offers a single module with a single executable. Our previous progams have multiple executables, and we'll want to repeat the pattern here, so we'll need multiple executables. Since we don't want to duplicate code across those executables, we'll also want the crate to be a _library crate_. Finally, we want to eliminate as much boilerplate as possible. So, let's add the rest of our "boilerplate" pieces.

## src/lib.rs

```rust {linenos=table}
include!(concat!(env!("OUT_DIR"), "/lib.rs"));
```

What on earth is this? It's an include macro, for a file that doesn't exist yet. Remember how we hadn't yet defined `run_all`? Well, we still haven't, but we're getting closer.

`lib.rs` has special meaning, just like `main.rs`. The existence of `lib.rs` indicates that this crate is a library crate. Crates can be both library and binary crates, and since we have both `main.rs` and `lib.rs`, this is the case. Anything that we want exposed by the crate must be exposed through lib.rs.

## bin/day1.rs
```rust {linenos=table}
include!(concat!(env!("OUT_DIR"), "/bin/day1.rs"));
```

Why, this looks almost exactly the same as `lib.rs`!. This file represents our single-day executable, and we'll need to create a new one for each day. Our build script will handle populating the file, just as with making updates to `lib.rs`, but we must have the file shell there for it to be recognized as an executable. Each file in the `bin` directory is built as a separate executable by the Rust build system.


## build.rs

Here's where the magic happens:

```rust {linenos=table}
use serde::Serialize;
use std::{env, fs, path::Path, path::PathBuf};
use tinytemplate::TinyTemplate;

static DAYS: [u8; 1] = [1];

#[derive(Serialize)]
struct LibContext {
    days: &'static [u8],
    cwd: PathBuf,
}

#[derive(Serialize)]
struct MainContext {
    day: u8,
}

static LIB_RS_TEMPLATE: &'static str = r#"
use std::error::Error;
use std::time::Instant;

{{ for day in days }}
#[path = "{cwd}/src/day{day}.rs"]
mod day{day};
{{ endfor }}

{{ for day in days }}
pub use day{day}::day{day};
{{ endfor }}
pub fn run_all(input_path: &str) -> Result<(), Box<dyn Error>> \{
    let funcs = [
        {{ for day in days -}}day{day},{{ endfor -}}
    ];
    let start = Instant::now();
    for (i, func) in funcs.iter().enumerate() \{
        func(&format!("\{}/\{}.txt", input_path, i + 1))?;
    }
    println!("All puzzles completed in \{:?}", start.elapsed());
    Ok(())
}
"#;

static DAY_MAIN_RS_TEMPLATE: &'static str = r#"
use simple_error::SimpleError;
use std::env;
use std::error::Error;

fn main() -> Result<(), Box<dyn Error>> \{
    let args = env::args().collect::<Vec<String>>();
    if let Some(path) = args.get(1) \{
        return aoc2015::day{day}(path);
    }
    Err(Box::new(SimpleError::new("No input file provided")))
}
"#;

fn main() {
    let out_dir = env::var_os("OUT_DIR").unwrap();

    let mut tt = TinyTemplate::new();
    tt.add_template("lib.rs", LIB_RS_TEMPLATE).unwrap();
    tt.add_template("day/main.rs", DAY_MAIN_RS_TEMPLATE)
        .unwrap();

    let ctx = LibContext {
        days: &DAYS,
        cwd: env::current_dir().unwrap(),
    };
    let lib_rs = tt.render("lib.rs", &ctx).unwrap();
    let lib_rs_path = Path::new(&out_dir).join("lib.rs");
    fs::write(&lib_rs_path, &lib_rs).unwrap();

    fs::create_dir_all(Path::new(&out_dir).join("bin")).unwrap();
    for day in DAYS {
        let ctx = MainContext { day };
        let main_rs = tt.render("day/main.rs", &ctx).unwrap();
        let main_rs_path = Path::new(&out_dir)
            .join("bin")
            .join(format!("day{}.rs", day));
        fs::write(&main_rs_path, &main_rs).unwrap();
    }
    println!("cargo:rerun-if-changed=build.rs");
}
```
`build.rs` at the same level as `Cargo.toml` indicates a _build script._ This is arbitrary code which can affect the build process. In our case, we are using it to generate our boilerplates and add them to the built executable. Let's walk through this file.

```rust {linenos=table}
use serde::Serialize;
use std::{env, error::Error, fs, path::Path, path::PathBuf};
use tinytemplate::TinyTemplate;
```

As with `main.rs`, our preamble has our imports. We are using several items from the standard library, as well as items from `serde` and `tinytemplate` that we added to our `[build-dependencies]` section of `Cargo.toml`. We'll go over the usages as we encounter them.

```rust {linenos=table,linenostart=5}
static DAYS: [u8; 1] = [1];
```

Here, we are defining a `static` variable which is an array containing `1` item of type `u8`, and assigning it a value which is `[1]`. This is our list of days as in the previous languages, and one of two places we'll need to make a change every time we do a new day's puzzle.

```rust {linenos=table,linenostart=7}
#[derive(Serialize)]
struct LibContext {
    days: &'static [u8],
    cwd: PathBuf,
}

#[derive(Serialize)]
struct MainContext {
    day: u8,
}
```

In this section of code, we are defining two `struct`s which will be used to pass data to our templates. `tinytemplate` requires structs that implement the `serde::Serialize` trait, unlike our Go templates where we could pass any value in as the context. The `#[derive(Serialize)]` _attribute_ before each struct definition allow this implementation to be taken care of automatically, and are a part of the `serde` crate.

We are generating two different types of files, so we have a context struct for each one. The `LibContext` struct has a _reference_ to the static days array we defined. Since we are referring to static data, we need to assign a static _lifetime_ to the reference, which is done with `'static`. The `static` lifetime is a reserved lifetime and indicates that the reference is valid for the entire lifetime of the running program. `LibContext` also contains a `PathBuf` which is an OS-independent file path.

`MainContext` contains a single `u8` variable which represents the specific day being built.

```rust {linenos=table,linenostart=18}
static LIB_RS_TEMPLATE: &'static str = r#"
use std::error::Error;
use std::time::Instant;

{{ for day in days }}
#[path = "{cwd}/src/day{day}.rs"]
mod day{day};
{{ endfor }}

{{ for day in days }}
pub use day{day}::day{day};
{{ endfor }}
pub fn run_all(input_path: &str) -> Result<(), Box<dyn Error>> \{
    let funcs = [
        {{ for day in days -}}day{day},{{ endfor -}}
    ];
    let start = Instant::now();
    for (i, func) in funcs.iter().enumerate() \{
        func(&format!("\{}/\{}.txt", input_path, i + 1))?;
    }
    println!("All puzzles completed in \{:?}", start.elapsed());
    Ok(())
}
"#;
```

Here we have the template for the contents of our `lib.rs` file. Remember the `include!` macro in the actual file? This is the template for what ultimately what will be included. As string literals in Rust are references to `str`, we need to define the `static` lifetime on the reference. We are also using the raw string syntax `r#"..."#`, much like we used raw strings in Go.

Let's dig a bit deeper into our template. 

```
use std::error::Error;
use std::time::Instant;
```

In our preamble, we are importing `std::error::Error` and `std::time::Instant`. `Instant` will be used for measuring our runtimes.

```
{{ for day in days }}
#[path = "{cwd}/src/day{day}.rs"]
mod day{day};
{{ endfor }}
```

In this stanza, our template will loop over the days provided in the context, and add a `mod dayX;` line for each day. `mod` tells the file that there is a _module_ of the same name at the provided path. In our case, the day modules will all be at the top level of the crate, so we don't need any further path. In front of each `mod` statement, there is also a `#[path]` attribute. Remember that unlike Go, our generated code will not be placed into the repository, so we need to tell the compiler where those files actually are. This is done using the `cwd` field in the context.

```
{{ for day in days }}
pub use day{day}::day{day};
{{ endfor }}
```

In this stanza, we again loop over all the days and create a `pub use dayX::dayX` statement for each day. Like other `use` statements, these are imports, but with an additional `pub` modifier which re-exports them. This would make the day functions available as `aoc2015::day1` if anybody were importing our crate.

```
pub fn run_all(input_path: &str) -> Result<(), Box<dyn Error>> \{
    let funcs = [
        {{ for day in days -}}day{day},{{ endfor -}}
    ];
    let start = Instant::now();
    for (i, func) in funcs.iter().enumerate() \{
        func(&format!("\{}/\{}.txt", input_path, i + 1))?;
    }
    println!("All puzzles completed in \{:?}", start.elapsed());
    Ok(())
}
```

Here we have the template for our `run_all` function that we've already referred to. This is functionally identical to the other languages' boilerplate; given a list of days, run each day's function with the provided input path, and then print the elapsed time at the end. We'll see each of the usages in this function later on, so I won't go into detail on them here. Note that the backslashes are necessary to escape the brackets for the templating engine and won't be part of the generated code.

```rust {linenos=table,linenostart=43}
static DAY_MAIN_RS_TEMPLATE: &'static str = r#"
use simple_error::SimpleError;
use std::env;
use std::error::Error;

fn main() \{
    let args = env::args().collect::<Vec<String>>();
    if let Some(path) = args.get(1) \{
        return aoc2015::day{day}(path);
    }
    Err(Box::new(SimpleError::new("No input file provided")))
}
"#;
```

This statement defines the variable for our single-day template. We've already seen everything here so I won't deep-dive on it.

```rust {linenos=table,linenostart=57}
fn main() {
```

Here we define the main function for our build script. A build script is just a Rust binary, so we use `main` just like any other Rust binary. We don't provide a return value here as any error we would encounter during the build is unrecoverable, so we will panic for those errors.

```rust {linenos=table,linenostart=58}
    let out_dir = env::var_os("OUT_DIR").unwrap();
```

The Rust build system provides the `OUT_DIR` environment variable to the build script, which is the folder in which the generated files should be placed. Here, we grab that variable from the OS. `env::var_os("OUT_DIR")` returns `Option`, so we chain the return into `unwrap()`. `unwrap()` returns the raw value for the `Some` variant, and _panics_ for the `None` variant. Since any errors here are errors in coding and thus unrecoverable, it is valid to panic and end execution.

```rust {linenos=table,linenostart=60}
    let mut tt = TinyTemplate::new();
    tt.add_template("lib.rs", LIB_RS_TEMPLATE).unwrap();
    tt.add_template("day/main.rs", DAY_MAIN_RS_TEMPLATE)
        .unwrap();
```

In these lines, we initialize our template engine and add the templates we already defined. The `add_template` method returns `Result`, which we chain into `unwrap()`. `unwrap()` has the same behavior on `Result` as on `Option`, panicking if the `Err` variant is returned, or returning the raw `Ok` value.

```rust {linenos=table,linenostart=65}
    let ctx = LibContext {
        days: &DAYS,
        cwd: env::current_dir().unwrap(),
    };
    let lib_rs = tt.render("lib.rs", &ctx).unwrap();
    let lib_rs_path = Path::new(&out_dir).join("lib.rs");
    fs::write(&lib_rs_path, &lib_rs).unwrap();
```

This section of code is generating the contents of our `lib.rs` file. First, we create a context object with a reference to the static `DAYS` array and the current working directory (which is always the top-level directory of the crate when run from the build system). Note that `env::current_dir()` again returns `Result`, so we `unwrap()`.

Next, we render the template into a `String` using the templating engine's `render()` method and `unwrap`ping the returned `Result`. The template name and context are passed into the `render()` method.

Finally, we build the output file path using the `Path` module, and write the rendered string to the file, again `unwrap`ping the returned `Result`.

```rust {linenos=table,linenostart=73}
    fs::create_dir_all(Path::new(&out_dir).join("bin")).unwrap();
    for day in DAYS {
        let ctx = MainContext { day };
        let main_rs = tt.render("day/main.rs", &ctx).unwrap();
        let main_rs_path = Path::new(&out_dir)
            .join("bin")
            .join(format!("day{}.rs", day));
        fs::write(&main_rs_path, &main_rs).unwrap();
    }
```

In this section, we generate our single-day executables. First, we create the "bin" subfolder in our output directory, then loop over the days as we have done previously. From there, the steps are the same as the single-day section: render, create the path, and write.

```rust {linenos=table,linenostart=82}
    println!("cargo:rerun-if-changed=build.rs");
}
```

The last thing we do is print a directive that is picked up by Cargo, telling it to re-run the script if the script itself has changed.

This is our setup. It is both similar and different to the code generation we used with Go; in Go, the generator added the files to the repo; with Rust, the generated files are only used during the build process and are not committed to the repo. The end result is that for each day we add, we only need to update the `DAYS` variable in `build.rs` and add a one-line `bin/dayX.rs` file with the include directive to update our runners. Not too shabby!

In the next article we'll implement our day 1 solution in Rust.