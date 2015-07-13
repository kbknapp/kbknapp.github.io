---
layout: post
title: "Argument Parsing without Arguments"
date: 2015-07-12 12:54:00 -0400
comments: true
categories: rust clap programming cli
---

Even if we did not intend on using command line arguments, I **always** recommend following the next step when building any sort of CLI utility. By adding an argument parsing library (in this case `clap`) we give our utility an extra layer of polish that can really go a long way. Let's demonstrate...

Add `clap` do your `cargo` dependencies

```toml
# in Cargo.toml
[package]
name = "rwc"
version = "0.1.0"
authors = ["Kevin K. <kbknapp@gmail.com>"]

[dependencies]
clap = "*"
```

Now do three simple things, create new `App` struct, give it the version of your project, and start the `clap` parsing process:

```
// in src/main.rs
#[macro_use]
extern crate clap;

use clap::App;

fn main() {
	App::new("rwc")
    	.version(&*crate_version!())
        .get_matches();
   println!("Hello, world!");
}
```