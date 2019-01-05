+++
title = "hardware peripherals and typestate analysis"
date = 2019-01-02
+++

In this post we'll walk though the code that makes up the GPIO driver in subphase C of
[assignment one][2] of Stanford's cs140e course, where we see hardware peripherals modeled as a
kind of [finite state machine][5], with the type system being used to enforce certain static guarantees
that our code itself doesn't cause the hardware to go into some nonsensical or "[invalid state][6]".
This is a combination exploration of type systems and their use when developing drivers for the
peripherals of embedded devices.

To get there we should explore each of the ideas that help us understand what this all means from the beginning 
and then zoom into our final goal.

## Types

Like many concepts in computer programming, "types" are an older mathematical concept that
finds it's existence reified in the type systems of modern day programming languages.
In a nutshell, the word "type" has similar meaning to when one says "this type of thing or another",
denoting that two, maybe similar, things have different properties. For our case these  _types_ are given
to the _terms_ in the language we are writing in.

In the language of arithmetic we can say `41 + 1`, or `42`, or `21 * 2` are all valid terms
with their type as natural numbers, abbreviated as `nat` - maybe written with a colon to denote
the type like `42: nat`[^2]. Whereas
a term like `1 / 2` is an invalid expression for `nat` types but a valid rational number, which
is another type.

For a natural language, like English, we have terms with types whose composition is defined
by the grammar. We can say that `cold` is of the adjective type and `beer` is of the `noun`
type with our grammar saying we must put adjectives _before_ the nouns, thus "cold beer" is
a valid expression while "beer cold" is a nonsensical one.

The mathematical notion of the "type" grew out of some apparent inconsistencies in the foundations
of mathematics. Bertrand Russel wrote of the "theories of type" back in 1802 as a way to patch a version of
[naive set theory][8] that was afflicted with [Russel's paradox][9]. The details of this are beyond
the scope of this article, but we'll address the more common usage of what we mean by a "type theory" when we
explore Alonzo Church's simply typed lambda calculus in [another post][11]

The theory of types as created by Russel was a new logical language different from set theory
where concepts like "and" and "or" and "maybe", found in the predicate logic set theory was 
built upon, can be encoded into the types of the type system itself. See the wikipedia post on
[type theory][12] for how concepts in set theory find their counterpart in the theory of types.

It should suffice to say that when we are giving a *type* to a term, variable, or object we are determining
the set of things we can *do* to that object and that a language with a type system can check statements
in the language for their _type correctness_ to enforce that operations applied to operands are of the correct type.
This is a powerful concept that helps us create large systems that are easier to reason about, in which the programs
when compiled and type checked are themselves sort of proofs of their correctness.

Cool. So, what are some sorts of things that can be encoded in a type system?

## Type States

There are many things we can encode into a type system, here we're going to jump beyond numbers, pairs, booleans,
sum and product and types, and the like, and talk about something a little more arcane: type states.

So, what exactly is a  *type state*? Informally said, they are, as you might think, the idea that we can ascribe
a given set of states to some type. Where we say that some object with some type is always in one of the type states 
associated with that type. To move from one type state to another we perform some operation thats _allowable_ on that
type in the given type state that it's in. These operations[^3], for a type with type states, when done one after the
other effectively act a series of *state transitions* (coercions) between states (types) and thus form a kind of finite state machine,
allowing us to keep track of the state that some statefully typed object is in.

In a large sense, *type states* also allow us to keep track of the _order_ in which certain operations are made on an object.

To illustrate why this is a useful idea, let's make this more concrete with a practical example:

Say we have a `File` type, with three states: "uninitialized", "opened" and "closed". The state machine could be drawn something like:

```
        .---------------. open  .--------. close  .--------.
        | uninitialized | ----> | opened | -----> | closed |
        '---------------'       '--------'        '--------'
                                  ^    |
                                  `----'
                                 seek/read
```

A `File` can't be read unless it has been opened, we also can't seek to a part of the
file unless it is open, nor can we close an unopened file. Though, once the file is
open we can do those things, and then close when finished. To borrow from David Teller's 
[example][3], let's show what this would look like in Rust:

```rust
fn read_file(path: &Path) -> String {
    let mut file = File::open(path);
    // opening the file _could_ fail
    file.open();
    // invalid read() if file failed to open
    let data = file.read();
    file.close();
    file.seek(Seek::Start); // ERROR! Already closed.

    data
}
```

## Typed move semantics

How can we ensure that this kind of behavior is caught by the Rust compiler? Luckily for us
we can exploit Rust's [affline type system][13] to leverage the use of it's special *move semantics*.
If you haven't already read the [Book][14], at least as much of it s.t. you understand
the basic ideas behind ownership and borrowing, and what the `Drop` trait is - now would be a good
time to do so.

Here's the implementation of our `File`:

```rust
impl File {
    // NOTE: Here we're using the venerable `Result` type which gives the postfix "?" operator
    //       that allows us to propagate errors if, say, any operations on the file fail.
    pub fn open(path: &Path) -> Result<File, Error> {
        // ...
    }

    // NOTE: Here `self` is "moved" into `close()`, by being passed by value and not
    //       by reference, consuming the file and effectively making any operations
    //       on `self` after this call to `close()` invalid.
    pub fn close(self) -> Result<(), Error> {
        // ...
    }

    // NOTE: &mut self, is a mutable reference and doesn't actually consume the file,
    //       making operations on self after reading, valid.
    pub fn read(&mut self) -> Result<String, Error> {
        // ...
    }

    pub fn seek(&mut self) -> Result<(), Error> {
        // ...
    }
}
```

As you can see, depending on how we use references and Rust's move semantics by passing `self`,
`&self`, or `&mut self` to the various `File` operations we are trivially informing Rust of
a `File`'s type states and they can be enforced at compile-time.

Let's go about implementing `Drop` for our `File` before re-evaluating the type checker's reaction to `read_file()`:

```rust
impl Drop for File {
    // NOTE: The Drop trait is basically like a "destructor" that can be used to
    //       automatically close the file when it falls out of scope.
    fn drop(&mut self) {
        self.close()
    }
}
```

Now here's the expected behavior we'd like to see from compiler, totally eliminating the
possiblity of `read()`'ing a file that hasn't been opened or `seek()`'ing a file that has 
already been closed:

```rust
fn read_file(path: &Path) -> String {
    let mut file = File::open(path);
    // opening the file _could_ fail, so we now use "?" to catch the exception.
    file.open()?;
    // if opening the file failed, then we never reach this line.
    let data = file.read()?;

    file.close();
    // ERROR! Which is now caught by the compiler.
    file.seek(Seek::Start);

    data
}
```

The error we'd see when `seek()`'ing the file would look something like:

```rust
r[E0382]: use of moved value: `file`
 --> src/main.rs:9:14
     |
   8 |     file.close();
     |          - value moved here
   9 |     file.seek(Seek::Start);
     |        ^ value used here after move
     |
     = note: move occurs because `file` has type `File`, which does not implement the `Copy` trait
```

This is Rust's ownership-centric view of the world at work. Since `File` is a type
that doesn't implement `Copy` it is an *_affline type_* where:

> Affine types are a version of linear types allowing to discard (i.e. not use) a resource, corresponding to affine logic. An affine resource _can only be used once_, while a linear one must be used once.

And thus, by passing the `File` resource by value, we have effectively _moved_ it, or *transferred ownership* of that resource
to the `close()` method. As a perhaps trivial way of using this unique (and yet pragmatic) type system to enforce type states,
it's possible to ensure that no `File` is ever acted on after it's been closed.

Let's finally move into a more complex example involving the the hardware target 

The GPIO subsystem of the Raspberry Pi is documented on page 89 (section 6) of the
[BCM2837 ARM Peripherals Manual][1]

## Shout outs

Thanks to those who wrote the Rust [embedded book][4] on peripherals as FSMs, hoverbear's
["Pretty State Machine Patterns in Rust"][4] post, to mozilla
researcher [Dale Teller][3] and their article on this subject, to the instructors who build
this [class][7] which introduced me to many interesting subjects, and of course to the original
[paper][10] on the typestate by Robert E. Strom and Shaula Yemini. I didn't understand typestates
and their use until reading all of these resources. Affline type systems have never been so fascinating!

[gpio]: https://web.stanford.edu/class/cs140e/assignments/1-shell/images/gpio-diagram.svg

[struct]: https://upload.wikimedia.org/wikipedia/commons/thumb/e/ea/Hasse_diagram_of_powerset_of_3.svg/1280px-Hasse_diagram_of_powerset_of_3.svg.png

[^2]: Example adapted from https://en.wikipedia.org/wiki/Type_theory

[^3]: From [wikipedia][12], "For each two type states `t1 < t2`, a unique _typestate coercion operation_ 
needs to be provided which, when applied to a typestate `t2`, reduces it to a typestate of `t1`..."

[1]: https://web.stanford.edu/class/cs140e/docs/BCM2837-ARM-Peripherals.pdf
[2]: https://web.stanford.edu/class/cs140e/assignments/1-shell/#subphase-c-gpio
[3]: https://yoric.github.io/post/rust-typestate/
[4]: https://hoverbear.org/2016/10/12/rust-state-machine-pattern/
[5]: https://en.wikipedia.org/wiki/Finite-state_machine
[6]: http://www.merkur-online.de/bilder/2009/04/21/216591/1389401738-toaster-explosion-feuer-2409.jpg
[7]: https://web.stanford.edu/class/cs140e
[8]: https://en.wikipedia.org/wiki/Naive_set_theory
[9]: https://en.wikipedia.org/wiki/Russell%27s_paradox
[10]: http://www.cs.cmu.edu/~aldrich/papers/classic/tse12-typestate.pdf
[11]: https://mysterious.computer/simply-type-lambda-calculus
[12]: https://en.wikipedia.org/wiki/Typestate_analysis
[13]: https://en.wikipedia.org/wiki/Substructural_type_system#Affine_type_systems
[14]: https://doc.rust-lang.org/book/second-edition/ch04-00-understanding-ownership.html
[15]: https://github.com/rust-lang/rfcs/blob/master/text/0243-trait-based-exception-handling.md
