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

For a natural language, like English, we have terms with types whose composition in a sentence is defined
by the English grammar. We can say that `cold` is of the adjective type and `beer` is of the `noun`
type with our grammar saying we must put adjectives _before_ the nouns, so "cold beer" is
a valid expression while "beer cold" is a "nonsensical".

The mathematical notion of the "type" grew out of some apparent inconsistencies in the foundations
of mathematics. Bertrand Russel wrote of the "theories of type" back in 1902 as a way to remediate a version of
[naive set theory][8] that was afflicted with [Russel's paradox][9]. The details of all this are beyond
the scope of this article, but we'll address the more common usage of what we mean by a "type theory" when we
explore Alonzo Church's simply typed lambda calculus in [another post][11]. [This][15] is another great resource
for you type-freaks out there.

What's interesting is that the theory of types as created by Russel was a *_new logical language different from set theory_*
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
algebraic types, and the like, and instead talk about something a little more arcane: type states.

So, what exactly is a  *type state*? If you've ever studied a couple of nice [finite state machines][17] then the concept is
just that: types states are a way for us encode finite state automata into a type system. They are, as 
you might think, the idea that we can ascribe a given set of states to some type. Where we say that some object with some type 
is always in one of the type states 
associated with that type. To move from one type state to another we perform some operation thats _allowable_ on that
type in the given type state that it's in. These operations[^3] *when done one after the
other* effectively act a series of *state transitions* (coercions) between (type)states and thus form a kind of finite state machine,
allowing us to keep track of the state that some statefully typed object is in.

In brief, *type states* also allow us to keep track of the _order_ in which certain operations are made on an object.

To illustrate why this is a useful idea, let's make this more concrete with a practical example borrowed from David Teller's
[article][3] on type states in Rust:

Say we have a `File` type, with three states: "initialized", "opened" and "closed". The state machine could be drawn something like:

```
        .---------------. open  .--------. close  .--------.
        |  initialized  | ----> | opened | -----> | closed |
        '---------------'       '--------'        '--------'
                                  ^    |
                                  `----'
                                 seek/read
```

A `File` can't be read unless it has been opened, we also can't seek to a part of the
file unless it is open, nor can we close an unopened file. Though, once the file is
open we can do those things, and then close when finished. Here's the Rust code:

```rust
fn read_file(path: &Path) -> String {
    let mut file = File::new(path);
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
the basic ideas behind ownership and borrowing, that's always a good thing
to do but, it's not totally necessary here minus some syntax.

Here's the implementation of our `File`:

```rust
struct File { path: Path, cursor: usize, is_open: bool }

impl File {
    pub fn new(path: &Path) { File { path, cursor: 0, is_open: false } }

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
`&self`, or `&mut self` to the various `File` operations we are trivially informing the Rust compiler,
through the [borrow checker][18], of a `File`'s type states and how they should be enforced at compile-time.

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
    let mut file = File::new(path);
    // opening the file _could_ fail, so we now use "?" to catch the exception.
    file.open()?;
    // if opening the file failed, then we never reach this line.
    let data = file.read()?;

    {
        let mut scoped_file = File::new(path);
        file.open()?;
        file.read()?; // valid if no runtime error, still open
    } // scoped_file falls out of scope, but is Drop so closes

    file.close(); // valid, as file's in the "open" state.

    // ERROR! Now statically caught by the compiler.
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
to the `close()` method. As a perhaps trivial way of using this unique and pragmatic type system to enforce type states,
it's possible to ensure that no `File` is ever acted on after it's been closed.

Though, let's note, that we've encoded the type system *implicitly* in Rust's move semantics for a simple example that was amenable to that.
Can we be more explicit that what we're dealing with are *type states*?

We can. Let's finally move into a more complex example, which calls for something a little more than typed moves and
offers a more real-life example where one would not want to live without type states which
involves our beloved little hardware target: the Raspberry Pi.

## BCM2835 ARM Peripherals

Before diving back into Rust-world we'll try to cultivate some basic way of thinking about hardware as a state machine as well as
go into some details on the GPIO subsystem of the BCM2835 ARM processor seen on the Raspberry Pi 3.

The "General Purpose Input Output" subsystem of the Raspberry Pi is documented on page 89 (section 6) of the
[BCM2837 ARM Peripherals Manual][1]. These are the external pins that folks usually use to interface with
other computers and electronic gizmos.

The vertical pin layout of the GPIO pins, courtesy of [tvierb][18] :

```
                                J8
                               .___.
                      +3V3---1-|O O|--2--+5V
              (SDA)  GPIO2---3-|O O|--4--+5V
             (SCL1)  GPIO3---5-|O O|--6--_
        (GPIO_GLCK)  GPIO4---7-|O O|--8-----GPIO14 (TXD0)
                          _--9-|O.O|-10-----GPIO15 (RXD0)
        (GPIO_GEN0) GPIO17--11-|O O|-12-----GPIO18 (GPIO_GEN1)
        (GPIO_GEN2) GPIO27--13-|O O|-14--_
        (GPIO_GEN3) GPIO22--15-|O O|-16-----GPIO23 (GPIO_GEN4)
                      +3V3--17-|O O|-18-----GPIO24 (GPIO_GEN5)
         (SPI_MOSI) GPIO10--19-|O.O|-20--_
         (SPI_MOSO) GPIO9 --21-|O O|-22-----GPIO25 (GPIO_GEN6)
         (SPI_SCLK) GPIO11--23-|O O|-24-----GPIO8  (SPI_C0_N)
                          _-25-|O O|-26-----GPIO7  (SPI_C1_N)
           (EEPROM) ID_SD---27-|O O|-28-----ID_SC Reserved for ID EEPROM
                    GPIO5---29-|O.O|-30--_
                    GPIO6---31-|O O|-32-----GPIO12
                    GPIO13--33-|O O|-34--_
                    GPIO19--35-|O O|-36-----GPIO16
                    GPIO26--37-|O O|-38-----GPIO20
                          _-39-|O O|-40-----GPIO21
                               '---'
                           40W 0.1" PIN HDR
```

As mentioned in the manual, there are 54 pins total split into two banks. It's said that each of these pins
have at least two "alternative functions", which, in plain English, is really just a way of saying "each of these pins can act
as an *input* or an *output*, but some can do other things like UART, SPI, &c".

You can see which pins above likely support some of the other special alternative functions in the pinout above.

To select a pin we'd like to use, and it's function, we look at a the right `FSEL` (function select) 
register for our pin and flip the right bits. The layout of an `FSEL` register looks like this:

```
pin #:        9   9   9   8   8   8         0   0   0
            .___.___.___.___.___.___.____ .___.___.___.
            | 1 | 0 | 1 | 0 | 0 | 0 | ... | 0 | 0 | 0 |
            '---'---'---'---'---'---'-----'---'---'---'
```

With three bits being dedicated to each pin, allowing allows at most 10 pins per `GPFSELn` register (32-bits total) 
with at least two left unused.

With the three bits associated to our pin found, we set the function of this pin by using the corresponding bit
pattern. As per the manual we have:

* `000` for output
* `001` for input`
* `100` for alternate function zero
* `101` for alternate function one
* so and and so forth like, `110`, `111`, `011`, or `010` for the rest up to alt. function 5.

To set and clear pins (on or off) that have been set as outputs  we check out the `SET` and `CLR` registers (two of each to
cover every pin) and flip the corresponding bit (1st pin has 1st rightmost bit in the first
register, the 33rd pin as the 1st rightmost bit in the second register).

To check the level of a pin that's been set as an input we check the `LVL` register to check if the voltage on that
pin is low or high.

Now the state machine is begining to float out of the screen... Output pins can be *set* and *cleared* and input pins
can only be used to *read* the voltage level at that pin. We wouldn't want to break the specification by reading the
levels of an output pin, or attempting to set and clear a pin the ARM thinks is an input, would we? We'd like to be able to write
our driver with these concerns out of the way, and to allow others to use the public API of our driver to play with the
Pi's GPIO at a low-level without worry that they may be about to run software that puts the hardware into an invalid state.

All of the code we'll see here lives in my [solution repo][21] for the course. Let's break it down piece-by-piece and take a
deep-dive into Rust and driver internals:

## GPIO Driver Implementation

### Top-level imports

```rust
use common::{IO_BASE, states};
use volatile::prelude::*;
use volatile::{Volatile, WriteVolatile, ReadVolatile, Reserved};
```

Hardware devices communicate with software through *memory-mapped I/O* where their functionality
is revealed through a certain set of memory addresses. The data-sheet we were reading above gives
us a specification for what will happen to the device when we read or write to certain addresses
in memory. `IO_BASE` gives us the base, or start, address of where the Pi's I/O peripherals are
mapped to. The GPIO registers exist at an offset from that base addresss.

`Volatile`, `WriteVolatile`, `ReadVolatile`, and `Reserved` are all pointer types that we'll go into
the details of in another post that will be about exposing unsafe code safely in Rust. Suffice to say, these are pointer
types that come with certain static guarantees about what you can and cannot do to the pointer-type (address) that it wraps (points-to). 
For example, the compiler will complain if you try to apply a bit-mask to a `ReadVolatile` pointer since it's
a read-only pointer and doesn't have any operations on it that allow writes.

### Functions and Registers

Here we encode the available functions into Rust's tagged-union type: the `enum`.

```rust
/// An alternative GPIO function.
#[repr(u8)]
pub enum Function {
    Input  = 0b000,
    Output = 0b001,
    Alt0   = 0b100,
    Alt1   = 0b101,
    Alt2   = 0b110,
    Alt3   = 0b111,
    Alt4   = 0b011,
    Alt5   = 0b010
}
```

Now we can use the various `Function`s like `Function::Input` or `Function::Output` as, for example arguments to
functions that take the `Function` type as an argument, and bit patterns will be substitued in. Nice start!

`repr` is a macro described in the [nomicon][22] that affects the data layout, or alignment,
of the type we're defining. Here, with `repr(u8)`, we're saying, "any variant of `Function` must be aligned to 8 bits"
(i.e: each variant lives 8-bits apart from the next). There are [many other][24] `repr`s and the one's being
used here come from [RFC2195][23] on the feature called "Really Tagged Unions". This is cool because enums aren't a
concept in C, yet we can still leverage high-level programming with a strongly typed language on bare metal...

Next we write out the relevant GPIO registers and type their pointers with the access-controlled
pointer types we got from the `volatile` crate, as per the spec:

```rust
#[repr(C)]
#[allow(non_snake_case)]
struct Registers {
    FSEL: [Volatile<u32>; 6],      // function select register
    __r0: Reserved<u32>,
    SET: [WriteVolatile<u32>; 2],  // set register
    __r1: Reserved<u32>,
    CLR: [WriteVolatile<u32>; 2],  // clear register
    __r2: Reserved<u32>,
    LEV: [ReadVolatile<u32>; 2],   // level register
    __r3: Reserved<u32>,
    EDS: [Volatile<u32>; 2],
    __r4: Reserved<u32>,
    REN: [Volatile<u32>; 2],
    __r5: Reserved<u32>,
    FEN: [Volatile<u32>; 2],
    __r6: Reserved<u32>,
    HEN: [Volatile<u32>; 2],
    __r7: Reserved<u32>,
    LEN: [Volatile<u32>; 2],
    __r8: Reserved<u32>,
    AREN: [Volatile<u32>; 2],
    __r9: Reserved<u32>,
    AFEN: [Volatile<u32>; 2],
    __r10: Reserved<u32>,
    PUD: Volatile<u32>,
    PUDCLK: [Volatile<u32>; 2],
}
```

`repr(C)` tells Rust compiler to do what C does - the layout seen will be exactly as you'd
expect in C(++) with each field stored right-after-the-other and not optimized all over the place.

`allow(non_snake_case)` disables the snake-case lint on field-members so we can use all capitol letters to
name our registers - feels appropriate.

The layout of the registers is just as is described in the manual. The syntax `[Volatile<u32>; 6]`
says, "here lives six 32-bit pointers that we expect to be able to read and write to".

### Stateful GPIO

So, far we know that a pin can be either an `input` or an `output` or one of the `alt` functions.
Okay so that means when a pin is in one of these states it can only do certain things. In what order
can change the state of a pin, and what can we do at each state? Let's draw out the simple-transition 
diagram of our upcoming `Gpio` type.

```
                                              .~-----.
    .---------------.  Function::Input   .---------. | level
    | uninitialized |------------------> |  Input  |<'
    '---------------'                    `---------'
      |       |   Function::Output    .---------.
      |        `--------------------> |  Output | -.
      |   Function::Alt  .-----.      `---------'  |  set/clear
       `---------------> | Alt |             ^.____/
                         `-----'
```


It would be pretty difficult to encode this state machine using a series of typed moves only,
we need this *and something more*. Here is our `Gpio` type, as of now, plain and simple:

```rust
pub struct Gpio {
    pin: u8,
    registers: &'static mut Registers,
}
```

We have a byte-size `pin` field and mutable reference to the GPIO `Registers` type that has a `static`
lifetime. This is to say that this mutable reference lives for entire life time of the program.

In Rust we can parameterize container types with the other generic types it contains. For example,
`Vec<T>` is a vector-type in rust that takes some generic type `T` so it could be concretized as
either `Vec<u8>`, for a vector of `u8`'s or as `Vec<Vec<char>>` as a vector of vectors of `char`s.

What if we could parameterize the `Gpio` type with its associated state? Just as marker, not be
used. Like so,

```rust
pub struct Gpio<State> {
    pin: u8,
    registers: &'static mut Registers,
}
```

When running this definition through the Rust compiler, we'd receive the error:

```
error[E0392]: parameter `State` is never used
--> src/lib.rs:1:17
  |
1 | pub struct Gpio<State> {
  |                 ^^^^^ unused type parameter
  |
  = help: consider removing `State` or using a marker such as `std::marker::PhantomData`
```

Ah, interesting! So, we really do have to *use* the type parameter and the compiler will
always complain if we paramterize something on a type and don't make use of it. Nice, but
we still want to *mark* or `Gpio` pin with it's state and... and what's this... `PhantomData`!?

### PhantomData of the opera

Many of those who lived a life programming in strongly-typed functional languages will be
familiar with the concept of [phantom data types][25]. The word "phantom" seems to imply something
that is seen but doesn't actually exists. In fact, this is exactly the kind of phantom types do for
our type system. It's a parameterized "marker" type that isn't actually used, so we can use it
to inform the type system so that we can parameterize our own types and not use the types their
parameterized on for anything other than as a marker. Very cool.

Our new definition of the `Gpio` type as it's parameterized on `State`:

```rust
/// A GPIO pin in state `State`.
///
/// The `State` generic always corresponds to an uninstantiatable type that is
/// use solely to mark and track the state of a given GPIO pin. A `Gpio`
/// structure starts in the `Uninitialized` state and must be transitioned into
/// one of `Input`, `Output`, or `Alt` via the `into_input`, `into_output`, and
/// `into_alt` methods before it can be used.
pub struct Gpio<State> {
    pin: u8,
    registers: &'static mut Registers,
    _state: PhantomData<State>
}
```

Okay - let's write our states. Since we plan for these to just be markers, we'll never
instantiate them. Sounds like a use case for a fieldless enum with no variants like:

```rust
pub enum Uninitialized { };
pub enum Input { };
pub enum Output { };
put enum Alt { };
```

Or, with a macro in place to generate our enums for us:
```rust
/// Generates `pub enums` with no variants for each `ident` passed in.
pub macro states($($name:ident),*) {
    $(pub enum $name {  })*
}
```

We can define our states cleanly like:

```rust
/// Possible states for a GPIO pin.
states! {
    Uninitialized, Input, Output, Alt
}
```

And the methods that correspond to each state:

```rust
impl Gpio<Output> {
    /// Sets (turns on) the pin.
    pub fn set(&mut self) {
        // two banks, anything below pin 32 is in the first bank (either 0 or 1).
        let index = (self.pin / 32) as usize;
        // if in the first bank then pin 0 is in bit 0, if in bank two then
        // 32nd bit is at bit 0, and (32 - 32 == 0).
        let shift = self.pin as usize - index * 32;

        self.registers
            .SET[index]
            .write(1 << shift);
    }

    /// Clears (turns off) the pin.
    pub fn clear(&mut self) {
        let index = (self.pin / 32) as usize;
        let shift = self.pin as usize - index * 32;

        self.registers
            .CLR[index]
            .write(1 << shift);
    }
}

impl Gpio<Input> {
    /// Reads the pin's value. Returns `true` if the level is high and `false`
    /// if the level is low.
    pub fn level(&mut self) -> bool {
        let index = (self.pin / 32) as usize;
        let shift = self.pin as usize - index * 32;

        self.registers
            .LEV[index]
            .has_mask(1 << shift)
    }
}
```

The code above follow from our discussion about what registers do what and the pins they
correspond to. The comments should clarify the arithmetic needed to select the correct bit(s)
for a given pin.

And then, boom - we suddenly have a way of tagging `Gpio` typed objects with the state they're
currently in by wrapping the type we're parameterizing on with the phantom data type.
This is how we'll keep track of states, but how will we define our state-transitions? 
Like before, we can use *typed moves* to define transitions between states. We know how to do that.

```rust
impl Gpio<Uninitialized> {
    fn transition(self) -> Gpio<Output> {
        Gpio {
            pin: self.pin,
            registers: self.registers,
            _state: PhantomData
        }
    }
}
```

`self`, not being a reference, is moved into `transition()` and out comes a new output pin of type 
`Gpio<Output>` with the old `self` dropped.

However, transitions can exists on any one of the states (types) the `Gpio` type itself is
parmaterized on and we should have a way of generically describing a transition so that we
don't have to enumerate each and every state a given state has transitions to. To be able to
say something like, "Transition GPIO pin 42 from state T to some state S".


Let's use these generic type parameters in place of `Uninitialized` and `Output`:

```rust
impl<T> Gpio<T> {
    /// Transitions `self` to state `S`, consuming `self` and returning a new
    /// `Gpio` instance in state `S`. This method should _never_ be exposed to
    /// the public!
    #[inline(always)]
    fn transition<S>(self) -> Gpio<S> {
        Gpio {
            pin: self.pin,
            registers: self.registers,
            _state: PhantomData
        }
    }
}
```

This is the *only* method on `Gpio`-typed objects we should hide from the user of our driver
compeltely because it is generic on the states it can transition to and from, working for
all `S` and `T`. As implementors of this stateful `Gpio` type, we must use `transition()` carefully
to spec otherwise the driver itself will be incorrect and will have none of the guarantees
the user of our driver would expect.

To transition we'll tell the Rust compiler what type to expect, or at least enough information
so that it can infer:

```rust
/// Sets this pin to be an _output_ pin. Consumes self and returns a `Gpio`
/// structure in the `Output` state.
pub fn into_output(self) -> Gpio<Output> {
    self.into_alt(Function::Output).transition()
}

/// Sets this pin to be an _input_ pin. Consumes self and returns a `Gpio`
/// structure in the `Input` state.
pub fn into_input(self) -> Gpio<Input> {
    self.into_alt(Function::Input).transition()
}
```

Where `into_alt` is a function on `Gpio<Uninitialized>` that enables an alternative function for
a `self` (pin), by `OR`-masking the three bits associated with that pin with the bit pattern
that corresponds to the `Function` type variant we're using:

```rust
/// Enables the alternative function `function` for `self`. Consumes self
/// and returns a `Gpio` structure in the `Alt` state.
pub fn into_alt(self, function: Function) -> Gpio<Alt> {
    let fsel_index = self.pin as usize / 10;
    // 10 pins per GPFSELn register, 3-bits per FSEL{n} field (i.e: per pin)
    let pin_offset = (self.pin as usize % 10) * 3;

    // clear function bits for pin
    self.registers.FSEL[fsel_index]
        .and_mask(!(0b111 << pin_offset));

    // write new function bits
    self.registers.FSEL[fsel_index]
        .or_mask((function as u32) << pin_offset);

    self.transition()
}
```

Rust will infer, for example,  when calling `transition()` within `.into_output()` on a pin that 
the `T` type is a `Gpio<Alt>` being transformed into a `S` which is a `Gpio<Output>`.

For completion here's the full `Gpio<Uninitialized>` implementation:

```rust
impl Gpio<Uninitialized> {
    /// Returns a new `GPIO` structure for pin number `pin`.
    ///
    /// # Panics
    ///
    /// Panics if `pin` > `53`.
    pub fn new(pin: u8) -> Gpio<Uninitialized> {
        if pin > 53 {
            panic!("Gpio::new(): pin {} exceeds maximum of 53", pin);
        }

        Gpio {
            registers: unsafe { &mut *(GPIO_BASE as *mut Registers) },
            pin: pin,
            _state: PhantomData
        }
    }

    pub fn into_alt(self, function: Function) -> Gpio<Alt> { ... }

    pub fn into_output(self) -> Gpio<Output> { ... }

    pub fn into_input(self) -> Gpio<Input> { ... }
}
```

The type `Gpio<Uninitialized>` is the only state that has a `new` method so saying `Gpio::new(42)`
will create an uninitialized pin of type `Gpio<Uninitialized>` and the user can't fabricate an
invalid initial state and accidentally violate the hardware specification. 

## Conclusion

Wow, with a combination of *phantom data types* and *typed moves* we are able to create type states for
our types! Rust is unique in allowing us to do this, though we should expect many languages down the
road to use some inspration from Rust's type system to allow programmers to encode *perfectly* the semantics
of a type state in their programs. With this, we can create a world with more secure hardware drivers.

Just Beautiful.

## Shout outs

Thanks to those who wrote the Rust [embedded book][4] on peripherals as FSMs, hoverbear's
["Pretty State Machine Patterns in Rust"][4] post, to mozilla
researcher [Dale Teller][3] and their article on this subject, to the instructors who build
this [class][7] which introduced me to many interesting subjects, and of course to the original
[paper][10] on the typestate by Robert E. Strom and Shaula Yemini. I didn't understand typestates
and their use until reading all of these resources. Affline type systems have never been so fascinating!

Also, at the Chaos Communications Congress this year was a great talk by Paul Emmerich, Simon Ellmann and Sebastian Voit
on writing safe and secure drivers in high-level languages with Rust as a primary example. Here's the [link][20].

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
[16]: https://plato.stanford.edu/entries/type-theory/
[17]: http://foldoc.org/finite+state+machine
[18]: https://www.reddit.com/r/rust/comments/5ny09j/tips_to_not_fight_the_borrow_checker/dcf9zdv
[19]: https://github.com/tvierb
[20]: https://media.ccc.de/v/35c3-9670-safe_and_secure_drivers_in_high-level_languages
[21]: https://github.com/donkey-hotei/cs140e/blob/master/os/pi/src/gpio.rs
[22]: https://doc.rust-lang.org/nomicon/repr-rust.html
[23]: https://github.com/rust-lang/rfcs/blob/master/text/2195-really-tagged-unions.md
[24]: https://doc.rust-lang.org/nomicon/other-reprs.html
[25]: https://wiki.haskell.org/Phantom_type
