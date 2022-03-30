---
title: "Advent of Code 2015 Day 1: C"
subtitle: Not Quite Lisp
date: 2022-03-30T03:29:30.086Z
draft: false
featured: false
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