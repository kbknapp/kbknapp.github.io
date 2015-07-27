---
layout: post
title: "Rusty Utilities"
date: 2015-07-25T06:48:07-04:00
comments: true
categories: rust programming clap cli
---

This is the start of a small series about building CLI utilities in Rust.

 * [Part I - Introductions][part_i]
 * [Part II - Talk to me][part_ii]
 * [Part III - Argument Parsing without Arguments?!][part_iii]

## Part I - Introductions

Rust is excellent language for little command line utilities! The majority of my workload orbits around little scripts and utilities that make my real day job exponentially easier...or at least automated and more interesting. Previously, Python and Go were the tools I'd reach out for in order to accomplish these tasks. But I'm finding myself reaching more and more into this new Rusty toolbox these days.

My goal in this series is share the pain points I've stumbled through and knowledge I've gained while working through building little Rusty utilities. Hopefully you'll find it interesting, and perhaps even insightful.

## Start Your Engines!

In this series I will use an example utility (or two) to demonstrate the thought process one could follow when building these Rusty utilities. As with all things in life, your mileage may vary (YMMV). I am not a Rust expert, so while reading through this series I suggest you do something by boss loves to say, "Trust...but verify" :winky-face: If you find any errors or a better way to accomplish a task, please don't hesitate to let me know!

We'll start by making near "clone" of the Unix `wc` utility which simply counts the lines, bytes, characters, or words in a given file and prints them to `stdout`. We'll call our clone `rwc` because if you view my github repos you'll quickly find that I couldn't name something to save my life! To quote the over-quoted, yet horribly true

{% blockquote Phil Karlton %}
There are only two hard things in computer science: cache invalidation, naming things, and off by one errors.
{% endblockquote %}

We'll move through a few iterations of `rwc` discussing the pros and cons of various ways in which we could build a CLI. Some of those ways may look nothing like the original `wc` but it's all for the sake of learning!

### Defining Functionality

Just like any project, the first thing we need to do is make sure we're clear on what we're making. Sometimes it's equally important to list what we're *not* making. Even if it's just a quick list, this will help us keep track and stay on target. If you're like me and abhor directions, do your best to at least *try* making a quick list.

At this stage in the game, it's a good idea to keep your functionality limited and reasonable. Too much functionality can overwhelm, or even make us scatter-brained and unfocused leaving half implemented features. So for our initial implementation let's keep it really simple and only count lines, we can always add functionality later. Our initial goal is:

 * Reads a file
 * Counts the number of lines
 * Displays the result to stdout

Now we can start the beef! Create a new project with `cargo`

```plain
$ cargo new rwc --bin

$ tree rwc

rwc
├── Cargo.toml
└── src
    └── main.rs

1 directory, 2 files
```

Using the utility `tree`(included in most Unix/Linux package repositories) we can see that `cargo` created an `rwc` directry with a `src/main.rs` file where our code will live, as well as the Rust project meta-data file `Cargo.toml`.

* **NOTE:** Or better yet, we could use [exa][exa_repo], such as `exa -Tl rwc`  which is an `ls` near drop-in replacement written in Rust but also includes some additional excellent features such as tree printing (`-T`).

Let's run the default Rust template just to ensure we have a working `rustc`

```plain
$ cd rwc

$ cargo run

  Compiling rwc v0.1.0 (file:///home/kevin/Projects/rwc)
     Running `target/debug/rwc`
Hello, world!
```

Since the default Rust template simply writes, "Hello, world!" to `stdout` we have a success! (Admin note: that did *not* require use of an exclamation mark...I'm easily excited)

## Know the Unknown

{% pullquote right %}
Unless our program is just a static script, always mindlessly doing the same thing no matter what, chances are there are some things our program must know about in order to do it's job, or make decisions in support of that lofty goal. Sometimes there is a default case where we can make assumptions, but not always. What we need to do is {" stop and determine those things which our program must know, and how it will obtain that knowledge. "} Typically there are three places our utility could obtain information to fill the void: from the system/runtime/environment, or the user (Those off by one errors strike again!). There are actually more, or combinations of several, but those two are the most common. Unsurprisingly, we will be focusing on the latter.
{% endpullquote %}

At this point let's look at what information *must* come from the user, but also keep in the back of your mind what information *could* come from the user. If we look at our `rwc` functionality list, you'll notice there is one thing it *must* know in order to run, which is....? The file! Ok, that wasn't hard, but if you have any gold stars laying around go ahead and stick one on your chest with pride!

For our simple clone, we only have a single input thus far, but we will add others shortly:

 * The file to read and count

There are many, many more things that *could* come from the user, and it's a good idea to note them for future functionality, but let's ensure we get a working product out the door first. If you're singing along, jot down some other things that *could* come from the user, and see if those get answered later.

## Start with Static Input

{% pullquote left %}
We've determined that we need to know which file to count, but in order to develop our utility it's almost always a good idea to {" start with a known input, which can be checked against a known output. "} For this I recommend creating a set of test data, which we can use as our ruler to determine whether or not our utility is functioning properly.
{% endpullquote %}

For `rwc` this means *not* getting the input from the user, but instead using a constant variable until we have a working product. This removes some points of failure and moving parts from the equation. In `rwc` this may seem like overkill, but in more complicated utilities it can be a real ~~life~~ time saver!

Let's add a file `test_input.txt`, which contains some Lorem Ipsum and will be our primary test case.

Place this file in the root of our `rwc` directory:

```plain link_text:"View in Github" url:https://github.com/kbknapp/rwc/blob/32fdeb768c59a2a19f3e52514e9facf94a9639a7/test_input.txt linenos:false
$ cat << EOF > test_input.txt

Lorem ipsum Amet esse consequat quis ea ad magna
adipisicing Duis id do laboris elit Ut
dolore veniam aliqua cillum amet fugiat
reprehenderit exercitation et consequat do cupidatat
voluptate do cillum mollit ullamco ex in in sunt velit Duis
id irure sed fugiat dolore aliquip eu aliqua proident nulla
in ullamco incididunt cupidatat mollit do exercitation
fugiat Excepteur sed veniam irure minim elit non
EOF
```

## Implement Basic (or Stub) Functionality

Now is the part where we either implement the basic functionality with the static input, or if the functionality is complicated we can stub it out with placeholders. Because `rwc` is very simple, let's go ahead and do the basic line counting:

**Disclaimer:** In order to keep examples short and focused, they may *not* be the most efficient or even good Rust code. They're meant solely to display an idea or principal in a larger context.

{% codeblock lang:rust src/main.rs https://github.com/kbknapp/rwc/blob/32fdeb768c59a2a19f3e52514e9facf94a9639a7/src/main.rs %}
use std::fs::File;
use std::io::Read;

const IN_FILE: &'static str = "test_input.txt";

fn main() {
    // Open and read the file
    let mut file = File::open(IN_FILE).ok().unwrap();
    let mut s = String::new();
    file.read_to_string(&mut s).unwrap();

    // Count the lines (types for clarity)
    let lines: u64 = s.lines().map(|_| 1).fold(0, |acc, n| acc + n);

    // Display the result
    println!("{} {}", lines, IN_FILE);
}
{% endcodeblock %}

**Note:** There are many ways to count the lines, and if we didn't care about using only stable Rust there's even a *slightly* faster, more concise `sum()` for iterators in the nightly branch (at the time of this writing stable = 1.1, beta = 1.2, nightly = 1.3).

Now let's test everything to make sure it's working, and check it against `wc` for GP.

```plain
$ cargo run

  Compiling rwc v0.1.0 (file:///home/kevin/Projects/rwc)
     Running `target/debug/rwc`
8 test_input.txt

$ wc -l test_input.txt

8 test_input.txt
```

Yay! Now that we have the base for this series we can get to interesting parts! Come back next time and we'll continue building `rwc` by adding some professional flare!


Rusty Utilities Series:

 * [Part I - Introductions][part_i]
 * [Part II - Talk to me][part_ii]
 * [Part III - Argument Parsing without Arguments?!][part_iii]

[part_i]: http://kbknapp.github.io/2015/07/25/rusty-utilities/index.html
[part_ii]: http://kbknapp.github.io/2015-07/25/talk-to-me/index.html
[part_iii]: http://kbknapp/github.io/2015-07/25/argument-parsing-without-arguments/index.html
[exa_repo]: https://github.com/ogham/exa