---
layout: post
title:  "Interoperability - calling Rust code from C"
description: Guide to calling a Rust code from within a C program
img:
date: 2018-10-04  +1545
---

1. Step 1: Create a Rust library
```
cargo new addition
```

In Cargo.toml, write:
```
[lib]
name = "addition"
crate-type = ["dylib"]
```

dylib stands for dynamic library

In src/lib.rs
```
#[no_mangle]
extern pub fn(a: i32, b: i32) -> i32 {
	println!("Hello from Rust");
	a + b
}
```
