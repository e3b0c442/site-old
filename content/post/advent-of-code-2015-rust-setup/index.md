---
title: "Advent of Code 2015: Rust setup"
date: 2022-06-01T14:14:42.987Z
draft: false
featured: false
image:
  filename: featured
  focal_point: Smart
  preview_only: false
---
Let's get our repo set up for the Rust side of our project.

The astute reader may have noticed that I've titled this post "setup" and not "boilerplate". This is because our setup will obviate the need for boilerplate! Honestly it's quite similar to the code generation setup Go uses, except it's using a Rust _build script_, which does not place the generated source in the repo like our Go generator does.

## Cargo

`cargo` is Rust's first-class build and dependency manager. One _can_ run the Rust compiler individually, but it generally doesn't make sense to. The first thing we are going to do is run `cargo new aoc2015` to make our package. This creates a simple hello world application with the necessary files. In particular, we're interested in `Cargo.toml`. Here's what the generated file looks like:

```toml
[package]
name = "aoc2015"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
```