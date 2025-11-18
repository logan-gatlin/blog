+++
title = "Results are not (just) for error reporting"
date = 2025-11-17
slug = "rust-ctrl-flow"
+++

I have struggled to understand how to effectively use `Result` in Rust for years.
`Result` is one of the most deceptively simple features I've seen in a language.
It wasn't until I did a deep dive into how Rust internally uses this data structure that it started to really click.
Learning more about `Result` has changed the way I think about structuring my code.

There are at least three levels to understanding `Result`.
None of them are wrong exactly, but each level is a little more *complete* than the last.

# Level 1: Results with error messages
If you go visit the [official documentation](https://doc.rust-lang.org/std/result/) for `Result`, this is the first example code you'll see:
```rust
#[derive(Debug)]
enum Version { Version1, Version2 }

fn parse_version(header: &[u8]) -> Result<Version, &'static str> {
    match header.get(0) {
        None => Err("invalid header length"),
        Some(&1) => Ok(Version::Version1),
        Some(&2) => Ok(Version::Version2),
        Some(_) => Err("invalid version"),
    }
}

let version = parse_version(&[1, 2, 3, 4]);
match version {
    Ok(v) => println!("working with version: {v:?}"),
    Err(e) => println!("error parsing header: {e:?}"),
}
```

Using strings as the error type in a `Result` is a classic beginner move.
It seems at first like a sensible thing to do; the user needs to know when something goes wrong, so why not tell them directly?
The problem is that a string is an error's **final form**.
They are easy for end users to digest, but very difficult to interpret, modify, or extend from within code.
When you represent your error as a string, you are effectively giving up on trying to handle it or obtain more context for a better error message later.
You may as well `panic!()` or `std::process::exit(1)` in that case.

It is best to use the most specific type possible to represent your data.
There are an infinite number of strings, but only two possible errors from `parse_version`.
Using an `enum` here is the much better choice.
Most errors in `std` use this approach.

# Level 2: Results with structured errors
The next level of understanding is that your program is a two way street.
Crates like [`anyhow`](https://docs.rs/anyhow/latest/anyhow/) and [`thiserror`](https://docs.rs/thiserror/latest/thiserror/) help write structured errors that gain context as they climb up the call stack.

Most programs will have two paths, success and failure.
The successful path travels *down* and then *up* the callstack.
It executes some algorithm by breaking it up piecewise.
Functions call other functions with increasingly specific purposes.
Pieces of the puzzle are returned back up the call stack, where they are assembled into some final product.

The fail path travels *up* the callstack.
The `Result` type and `?` operator are off-ramps from the successful path to the fail path.
When an error happens, its exact cause may not be very specific or useful to report.
As the error is propagated up the call stack, more context can be provided.
By the time the error reaches `main`, there is enough info to provide a contextually relevant error message.

To continue with the previous example, imagine you are parsing some binary file format.
Using the `anyhow` crate's custom `Result` type, you can attach additional context to the `Err` variant before it gets passed to the caller.
```rust
use anyhow::{Context, Result};

fn parse_file(reader: &mut impl Iterator<Item = u8>) -> Result<BinaryFile> {
  let version = parse_version(reader)?;
  let header = parse_header(version, reader).context(
    format!("Wrong header format for version {version}")
  )?;
  let contents = parse_content(version, &header, reader).context(
    "Content does not match the header, it may be truncated or corrupted"
  )?;
  Ok(BinaryFile {
    version,
    header,
    contents,
  })
}
```

# Level 3: Results as control flow
While hacking on my toy programming language [Halcyon](https://halcyon-lang.dev/), I ran into a problem.
When I compiled a program, only the first error encountered would be reported.
Most compilers try to recover from errors when they happen so they can keep checking the source code.
However, a `Result` can only be `Ok` or `Err`, not both, or something in-between.
I decided to look at how the Rust compiler handles this problem, and I found something unexpected.

From what I gather reading the source code, the parser handles errors in two ways: using `Result` and a mutable `context` variable.
Errors that can't be immediately recovered from are propagated as a `Result` as usual, while minor mistakes are added to the `context`.
The `Err` variant doesn't just contain what went wrong, but also how to best recover from the error.
For example, if an error happens inside of an open parenthesis, the parser may seek to the next closing parenthesis before continuing.
Once handled, these errors will end up in the `context` as well.

The third level of understanding is realizing `Result` is just another control flow mechanism.
It is a combination of a branch and an early return.
Error handling is just one (very common) case where branched early returns are useful.
However, tying error reporting together with control flow is sometimes a mistake.
Using `anyhow` will make your errors more comprehendable, but this is only half the battle; sometimes quantity is better than quality.
Rather than allowing errors to halt your program, consider logging them and continuing instead (if it is safe to do so).

Here is an excerpt from my new parser, which divorces error reporting and control flow:
```rust
/// How to recover from a parsing error
#[derive(Debug, Clone, PartialEq, Eq)]
pub enum RecoveryBehavior {
    /// No recovery necessary
    NoRecovery,
    /// Skip until this exact token is next
    UntilKind(TokenKind),
    /// Skip until this category of token is next
    UntilCategory(TokenCategory),
    /// Until the beginning of the next module statement
    UntilNextStatement,
}

type Result<T> = std::result::Result<T, RecoveryBehavior>;

pub struct Parser<'a, I: Iterator<Item = Token>> {
    // Buffer containing errors encountered so far
    logger: &'a mut LoggerT,
    iter: I,
}

impl<'a, I: Iterator<Item = Token>> Parser {
    // Error reporting without `Result`
    fn error_expected(
        &mut self,
        token: &TokenKind,
    ) -> LogBuilder<'_, usize> {
        self.logger
            .error(ERR_MSG)
            .context(format!("Expected `{token}` here"))
    }

    fn parse_expr(&mut self) -> Result<Expression> {
      /* ... */
    }
}
```

## Unconventional uses of Result
Sometimes a function wants to pause execution, and come back later.
This can happen because it is waiting on I/O, or for some resource to become available.
I believe the Jai compiler does this when the parser must wait a metaprogram to generate code.
Such a function could return a `Result` or `Option`.
The `Err` or `None` variants don't represent an error, rather they are a mechanism to return control flow to the caller without providing a return value.
The `std::io::Result` type has some `Err` variants that work this way:
```rust
// std::io::ErrorKind
enum ErrorKind {
  ResourceBusy,
  Interrupted,
  InProgress,
  /* ... */
}
```
If this reminds you of asynchronous programming, you are absolutely correct.
Rust's `Future` is really just a wrapped callback that returns something like an `Option`.
The `async` and `.await` syntax is not necessary for writing asynchronous code, it just provides a convenient and standardized abstraction for it.
```rust
// std::future::Future
pub trait Future {
  type Output;
  fn poll(
    self: Pin<&mut Self>,
    cx: &mut Context<'_>
  ) -> Poll<Self::Output>;
}

// std::task::Poll
pub enum Poll<T> {
  Ready(T),
  Pending,
}
```

So what does a resumable function without `async` look like?
The trick is to return a `Result` where the `Err` variant contains the current state of the function.
The state can be passed as a parameter later to resume where you left off.
In the example below, I am reading, processing, and writing a file in steps using a resumable function.
```rust
use std::fs::read_to_string;

enum ParseFnState<'a> {
  // Failure state
  Failure(String),
  // The name of the file to open
  Initial(&'a str),
  // The contents of the file after it is read
  FileRead(String),
  // The file processed into binary form
  ProcessedFile(Vec<u8>),
}

impl<'a> ProcessFnState<'a> {
  pub fn new(file_name: &'a str) -> Self {
    Self::Initial(file_name)
  }
}

fn process_file<'a>(
  state: ProcessFnState<'a>
) -> Result<(), ProcessFnState<'a>>  {
  use ParseFnState::*;
  match state {
    Failure(e) => Err(Failure(e)),
    Initial(file_name ) => Err(
      FileRead(
        read_to_string(file_name).map_err(
          |e| Failure(format!("Failed to read file: {e:?}"))
        )?
      )
    ),
    FileRead(file_contents) => Err(
      process_file(file_contents).map_err(
        |e| Failure(format!("Failed to parse file: {e:?}"))
      )?
    ),
    ProcessedFile(file) => {
      std::fs::write("out.bin", file).map_err(
        |e| Failure(format!("Failed to write file: {e:?}"))
      )?;
      Ok(())
    }
  }
}
```

Resumable functions are difficult to read and write.
Rust has an [upcoming feature](https://rust-lang.github.io/rfcs/3513-gen-blocks.html) called `gen` blocks, which are an abstraction for this pattern.
```rust
let mut generator = gen {
    for x in [0, 1, 2, 3] {
        yield x;
    }
};

assert_eq!(generator.next(), Some(0));
assert_eq!(generator.next(), Some(1));
assert_eq!(generator.next(), Some(2));
assert_eq!(generator.next(), Some(3));
assert_eq!(generator.next(), None);
```

# Closing thoughts
This article is about `Result`, but you need not restrict yourself to just this type.
There is also the lesser known [`ControlFlow`](https://doc.rust-lang.org/std/ops/enum.ControlFlow.html) enum with variants `Continue` and `Break`.
`Result` is simple enough that its trivial to write your own variation, with more specific functionality.
The `?` operator can even be implemented for these types using the `std::ops::Try` trait.

