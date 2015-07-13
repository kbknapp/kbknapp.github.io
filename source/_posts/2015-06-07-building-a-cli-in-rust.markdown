---
layout: post
title: "Building Command Line Utilities in Rust"
date: 2015-06-10 21:48:08 -0400
comments: true
categories: rust cli programming clap
---

## Introduction

I've always been fascinated by command line applications. They're like little treasures hiding in the open. When you take the time to search their depths, you'll often find seemingly infinite flexibility an utility. You may use a little application for years in a particular way, and then at some point find a totally new way in which to use it making you question how you've gone for so long without knowing about or using said "new" feature. Many times, your own imagination is the limit on new and useful ways in which to use, and even combine this little tools.

A good command line utility is somewhat of a paradox; simple and intuitive, yet flexible, complex, and deep. But when we say command line, or GUI application what are we really talking about? Usually, we're talking about the front end, or interface (the "I" in GUI or CLI) by which the end-user communicates with the *real* application (the backend, or core).

Giving your application a CLI in particular, has a few nice side effects. In many circumstances your applicaiton is then capable of being incorporated into scripts or other such automated tasks. And also it could make it accessible via low bandidth remote sessions, such as SSH.

Although I may be disillusioned about many things, command line applications aren't one of them. They *do* have their thorns, and aren't perfect in all situations. There are some tasks for which GUIs are unquesitonably the way to go. But that doesn't mean a good terminal program won't forever hold a special place in my heart.

This post is about adding a good CLI to your application's core. Unfortunately, a poor CLI can really hurt an otherwise amazing program.

## Disclaimer

First, I am no expert, nor do I claim to be. Some things in this post are subjective and clearly up for debate or personal prefernce. Take everything with a grain of salt :)

Second, for the sake of brevity I'm going to assume you're already familiar with the basics of Rust, and it's package manager `cargo`. If not, there are some wonderful introductions such as [The Book]() and [Rust by Example]().

## Rust?!

Perhaps your core application or library is already written in Rust, so writing the CLI in Rust is only natural. But if your application is still on the drawing board, hopefully this post will help demonstrate that Rust isn't just a low level systems language but also a very viable option for writing command line utilities that you want to be *fast* and *safe*!

Some may ask, aren't there other languages more suited to this area such as Python, Go, Ruby, etc? Maybe. Those are all great languages, and I'm actually quite fond of them, but there are a few things Rust offers me that the others simply can't match.

Many of the little utilities I write end up leaving my desk, never to be seen or heard from again. When I write in Rust, I"m far more confident that my programs will work as intended and won't blow up in the users face...or at least if they do it'll be more of a "poof..." and not a, "BOOOOM!" :) Having the option of a single statically compiled binary is quite nice as well!

## Semantics

The first part we need to talk about is difference between the CLI and the application itself, which we previously alluded to. Although many applications that run in a terminal are called CLIs, the CLI is actually just a small part of the whole pogram. As the 'I' in CLI stands for, it is the *interface* by which the user communicates with your core program.

## Types of Interfaces

There are three popular methods for developing said interface.

* Explictly asking question
* Creating a console, or sub-shell
* Letting the user *attempt* to speak your application's language

All have some pros and cons, but typically the latter is preferred except in very specific cases.

### Asking for Input

Asking the user explicitly may seem like a good choice at first, and it's typically the method taught first (think of most guessing game examples when learning to program). There are some big draw backs to this method. Imagine that we made a program which took as file as input, computed some values, and spit out a result.

A typical run of this program may look like this...

```
$ myprogram
 > Please specify an input file: myfile.txt
 > The answer is 3!
```
Looks fine, right? Perhaps it suffices. But then imagine running this program over an over again, perhaps you need to compute the value of all the files in a directory!

```
$ myprogram
 > Please specify an input file: myfile.txt
 > The answer is 3!
$ myprogram
 > Please specify an input file: myotherfile.txt
 > The answer is 287!
$ myprogram
 > Please specify an input file: myotherotherfile.txt
 > The answer is 12!
```

This will quickly begin to annoy a user. It would be *so* much nicer if the user could just specify the value up front!

This problem is compounded when asking for multiple input values, or several "optional" values. I remeber using an application which asked for several values, then had a few optional values (where you just hit `enter`), then asked for a few more values. If you got impatiant (as I often do) and hit `enter` one too many times during the "optional" values, you would miss a required value and have to start all over re-entering the values you'd already entered. This was infuriating!

Another downside is that their must be a human to answer the questions. These types of CLIs aren't typically scriptable (without some form of configuration file).

### A Custom Console

The second method, of creating a custom console can actually be a great solution when your application is very complex, and has many seemingly unrelated or optional capabilities. It can be severe over-kill for smaller applications. And you run into the same situation where these types of CLIs aren't very scriptable (unless you should you decide to implement such a feature yourself).

Custom consoles are somewhat of a hybrid between the first, and third form of CLI as they make heavy use of argument parsing.

### Argument Parsing

With argument parsing, the user specifies all the values she wishes to set, up front. As an added benefit, these types of applications are usually scriptable, becuase you don't need a human to sit down and manually enter all the values.

The downside to argument parsing is that the developer must then *validate* that all the required input values have been established. With complex programs using multiple options, some of which having intertwined relationships, this can be difficult!

Another difficulty in designing a good CLI is making the options and input types work well together and in the most intuitive way possible. This post will discuss how to overcome those difficulties using argument parsing. Perhaps a future post will discuss building a console as well once we've establish this base :)

## Argument Parsing in Rust

There are a few ways that we can go about parsing command line arguments in Rust. We could use the bare minimum functionality built into the Rust standard library, or use one of several third party libraries. I almost always recommend using a third party library; writing the code to parse arguments and handle edge cases is tedious. Add to that, in most cases it's not at all related to the functionality of the core application.

Using a third party library greatly alliviates the tedium, making it easier to write a *good* CLI without having to worry about the details.

Luckily for us Rust has quite a few third party libraries for doing just this. While the specific code examples here may be tied to a single library, the principals can be applied to any of them. The following examples focus on [clap]() because it is the one I am most familiar with. Some other great options include [docopt](), [getopts](), and recently [pirate]() (There are a few others as well, but I haven't personally tried them). All have various pros and cons, but largely come down to personal prefernce or use case. The over simplified comparison is that `getopts` and `pirate` are very minimalistic and lightweight, whereas `clap` and `docopt` are more full featured, but also more complex.

As with all things, try them all and pick the one fits your style or use case :)

## Finally! Making the CLI...

The first step in designing a CLI determining what your program needs to know from the user. The easiest way I've found to do this is to list what our application is supposed to do...as in actually write it out. Even if we already *think* we know what sorts of input we may need, it may help to write it, or say it out loud.

For this post's example, let's build a simple partial clone of the unix `wc` command called `rwc`. Some of it may be contrived, but it'll give us some concrete examples. Let's define our programs functionality:

* Counts the words, lines, bytes, and characters of a given file (files?) and prints the result to standard out

From this, we can see that at a minimum, we'll need to know the file (or files) to work on.

### Write the Core (or even stub) Functionality

It's a good idea to write the application, or at least stubs *without* any user input. This gives you a chance to validate a workflow and determine whether there are actually other peices of information you'll need from the user or not. In these cases I like to use `const` values to simulate input until I'm sure what I'll need.

If the program is complex, we can also write stubs for the parts that have no interaction with input from the user.

Let's create the basic `rwc` now...

**Note:** Ignore the inefficiences and `.unwrap()`s as that's not the focus of this post ;)

```
$ cargo new --bin rwc
$ cd rwc
$ vim src/main.rs
```

```rust
// src/main.rs
#![feature(core)]
use std::fs::File;
use std::io::Read;

const in_file: &'static str = "my_file.txt";

fn main() {
        let mut file = File::open(in_file).ok().unwrap();
        let mut s = String::new();
        file.read_to_string(&mut s).unwrap();
        let lines: u64 = s.lines().map(|_| 1).sum();
        let words: u64 = s.split_whitespace().map(|_| 1).sum();
        let bytes: u64 = s.bytes().map(|_| 1).sum();
        let chars: u64 = s.chars().map(|_| 1).sum();
        println!("{} {} {} {} {}", lines, words, bytes, chars, file_name);
}
```

Assuming we have a file called, `my_file.txt` in our directory we can run it

```
$ rwc
114 804 6059 6059 my_file.txt
```

...and everything works!

### Initial Considerations

Now that we have a working (or stubbed) program, and we know about at least one argument that we want to add, there is a **very** important question that needs to be asked. Unfortunately this question is often over-looked; what kind of argument will this be? Most command line argument parsers allow for differnt types of arguments, the most common types are:

* flags or switches (fixed yes/no or true/false fields)
* positional arguments (user supplied values based off an index from the program name)
* options (user supplied values preceded by a special identifier)
* sub-commands (mini-applications that may have their own arguments)

We'll get into the details of each shortly. Since we know our program needs to accept a value (presumably some type of `String`) that leaves two choices; positional arguments or options.

As with any software decision there are multiple ways to accomplish this task, and not necssarily a "right" answer. Sure, there are ways that are better than others, but it can come down to personal prefernce. In times like these, I move on to the next step...

### Enumerate the Options

When deciding how to best implement an interface, I find it very helpful to write out each permutation of the input and see what *feels* most ergonomic, or intuitve. Take this with a grain of salt, though. Becuase we are intimately familiar with the how the program does and should function, we may arrive at certain conclusions whereas an end user having no experience with our program may try something that seems totally illogical to us. **Consider reading the program description to a colleage, and asking them how they would intuitively invoke the program.**

For `rwc` our options are limited to only three possibilities:

```
$ rwc my_file.txt
$ rwc -i my_file.txt
$ rwc --input my_file.txt
```

**Note:** most libraries will allow you parse the third option alternatively as `rwc --input=my_file.txt` but there's no need to write both styles.

It's usually a good idea to side with less typing, unless it hinders readabiltiy *or* memorizing. In this case, becasue we only have a single argument, using a posisitional argument appears to be the way to go. Let's use a positional argument...

### Positional Arguments

There are a few circumstances where positional arguments are a good choice

* The argument is required at all times (such as with `rwc`)
* The total number of positional arguments is relatively low (< 2 or 3)
* The total number of possible arguments is relatively low (< 4)
* The order of arguments matters, or makes the intention easier to understand (such as `cp <from> <to>`)
* If the value is limited to a specifc set of values *and* one of the other rules applies (such as a required "mode", and this "mode" is the only argument)
* Multiple values  for a single field are valid (such as `git add <file1> <file2> <file3> <file4>`)

There are also a few times when positional arguments are *not* a good idea

* When the number of positional arguments is relatively large (> 3) (Although you may remember a command such as `make_user <name> <birth_day> <age> <mobile> <address>` a user may have a very difficult time remembering the order)
* When there are a large number of possible (or optional) arguments
* If a positional argument is required, all *previous* positional arguments must also be required
* When the order of arguments shouldn't matter (See the first bullet example)

Since we're sure using a possitional argument is both ergonomic and not breaking any usability rules, let's add a single positional argument using `clap` now. Let's also start by only accepting a single file, we can add more later.

```toml
# in Cargo.toml
[dependencies]
clap = "*"
```

**Note:** We are using a very verbose method of using `clap` in order to demonstrate what's going on behind the scenes so that you don't have to be familiar with `clap`s ins and outs to follow the examples. For other less verbose ways to use `clap` see the [project page on github](https://github.com/kbknapp/clap-rs).

```rust
// src/main.rs
extern crate clap;
use clap::{App, Arg};

fn main() {
	let matches = App::new("rwc")
    				.arg(Arg::with_name("file") // The name by which we will access the value
                    	.index(1)               // Positional index's start at 1
                    	.required(true))        // This argument *must* be present
                    .get_matches();
    // .unwrap() is safe here, because "file" is required
    let in_file = matches.value_of("file").unwrap();

	// ... same as before
}
```
Now if we run our program:

```
$ rwc my_file.txt
114 804 6059 6059 my_file.txt
$ rwc my_other_file.txt
64 321 2456 2456 my_other_file.txt
```

Yay! Now we could also run `rwc` against an entire directory almost for free!

```
$ find . -type f -exec rwc {} \;
26 54 639 639 ./Cargo.lock
3 5 24 24 ./.gitignore
114 804 6059 6059 ./my_file.txt
7 16 114 114 ./Cargo.toml
```

**Note:** Because this is just an example, if you run the above against a directory with binary files or non UTF-8 data it will panic.

### Validation

If you're testing the examples here, try running `rwc` with more than one argument, or no arguments at all. Because we told the argument parser that one value, and only one value, was required, it ensures that is the case.

This is *very* basic level of input validation. It's almost *always* a good idea to further validate user-input before using it. In our simple `rwc` program, `clap` ensures the proper number of arguments were specified, but it does not ensure that *what* the user entered was a valid file. In a real program we should also validation that the string input entered by the user is a valid file that we have Read permission to. Some argument parsers, such as `docopt` can perform further validation or serialization of certain types.

### Accepting More Files

So we're happy with what we've got, but come across a use case where we only want to run `rwc` against a few files, our current implementation isn't too much better than the version that explicitly asks for a file each time (except that now we could use `rwc` in a script). Let's make it more ergonomic to run `rwc` against more files, but perhaps not against a whole directory.

Most arguemnt parsers will allow us to specify a single argument that accepts multiple values, but there are a few considerations when doing so. If we're doing so with positional arguments, only the *last* positional argument can accept multiple values (unless the number of values is specifically limited to a set number).

This seems acceptable, since we only have one psositional argument anyways. Let's change our implementation:

```rust
fn main() {
	let matches = App::new("rwc")
    				.arg(Arg::with_name("file")
                    	.index(1)
                        .multiple(true)         // Allow mutliple values
                    	.required(true))
                    .get_matches();
    // Note the 's' in values
    for file_name in matches.values_of("file").unwrap().iter() {
        // ... same as before
    }
}
```

Let's test it:

```
$ rwc my_file.txt Cargo.toml .gitignore
3 5 24 24 ./.gitignore
114 804 6059 6059 ./my_file.txt
7 16 114 114 ./Cargo.toml
```

Awesome! ...But can we make it better?

### Aditional Arguments

At this point we could pat ourselves on the back and call it a day. But being the developer we are, we are constantly striving to improve our program. So we go back to our original definition, and see if there are any additional things the user may wish to tell our program.

* Counts the words, lines, bytes, and characters of the given files and prints the results to standard out

Perhaps the user *only* wants to see words, lines, bytes, or chacacters (or a combination)? Again we look the different ways we could implement this, so write the options (remember to write it exactly as the user would)...

With specific values using positional arguments, meaning a positional argument that accepts *only* the values `words`, `lines`, `bytes`, or `characters` (remember because the files accepts multiple values, it must be last):

```
$ rwc words my_file.txt
$ rwc bytes my_file.txt
$ rwc characters my_file.txt
$ rwc lines my_file.txt
```
But there is are a few problems with this. Can you see them? If you said, the order of the arguments or that we can't combine them, find a gold star sticker place it proudly on your chest!

Users may not use your program every day (sad face), and may forget which order the arguments go...was it `rwc <file>... [how]` or `rwc [how] <file>...`? As a side note, when we physically write it down it's obvious which version is correct, because an optional positional argument cannot come *before* a required one (how would you know which is `index 1` if the optional one is omitted?). And a multiple value positional argument must be last (because how else would you know when the other one started?). But users rarely write down their thoughts before trying them. Also, we couldn't combine methods, say if the user wanted to output lines *and* words, because only one positional argument may accept multiple values (again, because we can't know when one stops and the other starts).

So using a positional argument is out...Let's try option arguments, with specific values:

**Note:** Because it's a good idea to write each permutation, if feasible, and using options allows us not to worry about position there are many duplicates functionality-wise:

```
$ rwc my_file.txt --what words
$ rwc --what words my_file.txt
$ rwc my_file.txt --what characters
$ rwc --what characters my_file.txt
...etc
```
Since the `--what` is long version, and option arguments support long or short version, we could use a short version `-w` (either in addition to, or instead of)

```
$ rwc my_file.txt -w words
$ rwc -w words my_file.txt
$ rwc my_file.txt -w characters
$ rwc -w characters my_file.txt
...etc
```

But there is still a problem here! This implementation requires the user to remember, and correctly spell the type. If the user is anything like me, you can bet money that they will forget if it's `characters` or `chars`, or even accidently  (maybe not so...) type `charactres`. Libraries like `clap` can help with this by offering spelling suggestions on error, but it's still a burden on the user to remember. Learning a new CLI can be difficult, the goal is to lighten the burden on the user where feasible.

Also, we're now requiring the user to remember the identifier `--what` or `-w`, which may not be what they think of...maybe the think `--how`, or `--type`, etc.

As a last ditch effort, lets try flags (or switches...whichever you'd like to call them):

```
$ rwc my_file.txt --lines
$ rwc --lines my_file.txt
$ rwc my_file.txt --bytes
$ rwc --bytes my_file.txt
...etc
```

This isn't bad! But it's still a lot to type, and requires the user to spell somewhat correctly. Let's add a short version too

```
$ rwc my_file.txt -l
$ rwc -l my_file.txt
$ rwc my_file.txt -b
$ rwc -b my_file.txt
...etc
```
It's a safe assumption the user can *at least* remember what the word starts with, or so we hope. Flags can also be combined easily because they are separte arguments (such as `--words --lines` or `-w -l`. In addition, most libraries will allow you combine short version further such as `-wl`).

Let's discuss flags for a minute...

### Flags (or Switches)

Flags are simple yes/no, or true/false style fields. They're either present, or they aren't. The only "value" they bring is being present or not. Well, almost...flags can also be present either single or multiple times as well. Such as the typical Unix `-v` or `--verbose` flag which some programs allow either `-v` meaning "be verbose" or `-vv` (or `-v -v`) meaning "be *very* verbose." That is to say, their "number of occurrences" can be a sort of "value" as well.

Flags are a good choice when:

* The option is binary (yes/no, true/false, etc.)
* There is a large number of binary options (flags with short version can be combined, such as `-v -Z -g -R` being the same as `-vZgR`)
* If you only care about a relatively low optional number (< 4) which could be represented incrementally (An "optimize" flag where `-o` or `-oo` or `-ooo` could be three vary levels of optimization)
* The order doesn't matter


There are also pitfalls of flags:

* Flags **cannot** be required, that would be the same as saying something is always true (There are ways you can require a flag from a set of flags though, more on this later...)
* Like options, and specific values, the long version requires users to remember how to spell (maybe this is only a pitfall for me...)

Before we implement our new version, we need to define what should happen with each type of invocation (**Note:** we leave off the long version for brevity):

```
$ rwc <file>
	> Outputs lines, words, bytes, chars, and file name

$ rwc -l <file>
	> Outputs lines and file name

$ rwc -w <file>
	> Outputs words and file name

$ rwc -b <file>
	> Outputs bytes and file name

$ rwc -c <file>
	> Outputs characters and file name
```

Should we allow combinations? Sure, why not!

```
$ rwc -lw <file>
	> Outputs lines, words, and file name

$ rwc -cbl <file>
	> Outputs characters, bytes, lines, and file name

...etc
```

Now let's implement our `rwc` with these new flags (We're not going to follow the `wc` conventions, cause we like going against the grain!):

```rust
// src/main.rs
fn main() {
	let matches = App::new("rwc")
    				.arg(Arg::with_name("file")
                    	.index(1)
                        .multiple(true)
                    	.required(true))
                    .arg(Arg::with_name("words")
                    	.short("w")
                        .long("words"))
                    .arg(Arg::with_name("lines")
                    	.short("l")
                        .long("lines"))
                    .arg(Arg::with_name("bytes")
                    	.short("b")
                        .long("bytes"))
                    .arg(Arg::with_name("characters")
                    	.short("c")
                        .long("characters"))
                    .get_matches();

	let (l, c, b, w)  = (matches.is_present("lines"),
               		  matches.is_present("characters"),
                		 matches.is_present("bytes"),
                		 matches.is_present("words"));
	let empty = !l && !c && !b && !w;

	for file_name in matches.values_of("file").unwrap().iter() {
    	// Same as before...

        if l || empty { print!("{} ", s.lines().map(|_| 1).sum::<u64>()); }
        if w || empty { print!("{} ", s.split_whitespace().map(|_| 1).sum::<u64>()); }
        if b || empty { print!("{} ", s.bytes().map(|_| 1).sum::<u64>()); }
        if c || empty { print!("{} ", s.chars().map(|_| 1).sum::<u64>()); }
        println!("{}", file_name);
    }
}
```

Now we can run it and get only the ouput we care about against multiple files!

```
$ rwc my_file.txt Cargo.toml
114 804 6059 6059 ./my_file.txt
7 16 114 114 ./Cargo.toml

$ rwc Cargo.lock .gitignore -wl
26 54 ./Cargo.lock
3 5 ./.gitignore
```

Sweet :)

### Extra Niceties

Our program now has a small ammount of arguments. When creating a CLI, it's nearly always a good idea to follow certain conventions such has providing at a minimum `--help`, `--version`. Luckily for us, by using a third party library that's automatic! If we run `--help` with `rwc` we'll see it already has this functionality.

```
$ rwc --help
rwc

USAGE:
	rwc <file>... [FLAGS]

FLAGS:
    -b, --bytes
    -c, --characters
    -h, --help          Prints help information
    -l, --lines
    -v, --version       Prints version information
    -w, --words

POSITIONAL ARGUMENTS:
    file...
```

This is a little bit of polish that helps our CLI really elevate! If we run `rwc --version` all we see is `rwc`...what gives? Giving additional details is a great idea, but those details aren't automatic, we have to add them. Let's add those finishing touches now:

```rust
#[macro_use]
extern crate clap;

// same as before...

fn main() {
	let matches = App::new("rwc")
    				.version(&format!("v{}", crate_version!())[..]))
                    .about("Counts lines, words, characters, or bytes of a file")
                    .author("Kevin K.")
                    .after_help("Defaults to counting all lines, words, characters, and bytes if you don't specify \
                    			 any flags. If only a few of those fields are desired, use the flags in any \
                                 combination such as 'rwc -lw <file>' to count only lines and words")
    				.arg(Arg::with_name("file")
                    	.help("The file to count")
                    	.index(1)
                        .multiple(true)
                    	.required(true))
                    .arg(Arg::with_name("words")
                    	.help("Counts the words")
                    	.short("w")
                        .long("words"))
                    .arg(Arg::with_name("lines")
                    	.help("Counts the lines")
                    	.short("l")
                        .long("lines"))
                    .arg(Arg::with_name("bytes")
                    	.help("Counts the bytes")
                    	.short("b")
                        .long("bytes"))
                    .arg(Arg::with_name("characters")
                    	.help("Counts the characters")
                    	.short("c")
                        .long("characters"))
                    .get_matches();
	// Same as before...
}
```

Running `rwc --version` or `rwc --help` now displays all those nice details. It's important to note here, the various third party libraries all implement this functionality very differently and to varying degrees. It's a good idea to try them all and find the one that suits your needs best.

### What about options?!

The third common type of argument is options, or those that are like a hybrid of flags and positional arguments. There are a few circumstances in which options are a good idea to use:

* When there are many differnet optional arguments
* When there is a larger number of required arguments (> 2, or 3)
* When the order of arguemnts doesn't, or shouldn't matter
* When the invocation of the program should be self-documenting
* When there are optional values a user could present, but there are already optional or required positional arguments

As with positional arguments, or flags, there are also a few draw backs to options as well:

* Extra typing if the number of total arguments is few ( < 3)
* Spelling...again

Its also worth noting that many third party libraries also allow options to specified in several different manners. `clap` uses `--long <value>`, `--long=value`, or `-l value` (as well as bunch of seperate options you can explore).

## Wrapping Up

We have a good understanding of the basics at this point. We know when to use a flag, or a positional argument, or an option. We haven't yet touched requirements, relationships, or subcommands. Check back soon for those additions!