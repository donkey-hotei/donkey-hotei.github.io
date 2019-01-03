+++
title = "safely exposing unsafe code in Rust"
date = 2018-12-23
+++
In one of my first blog posts, I'll be writing about exposing unsafe code
safely to end users. Rust is a language that prides it self on memory safety and
high-level "zero-cost" abstractions. Though somtimes, to get things done, a programmer has to write
code which is inherently unsafe. This is especially true for those working closer to the
metal.

Even looking at the Rust core library itself we see a variety of data structures, like `Vec` that,
while they can be used by us with utmost safety, are themselves implemented using unsafe
code.

While working though Stanford's CS140e course, to learn Rust and operating systems development,
I've come across a good many use cases where, despite having to write a good amount of
unsafe code to interface with the ARM processor and other peripherals on the Rasperry Pi, we
find ourselves leveraging Rust's type system to lift the unsafe code into safe and usable
libraries, enforcing those type-safe guarantees ourselves.

Today we're met with the need to write a heap allocator from scratch so that we can leverage
types like `Vec`, `HashSet`, & `Box` in our small operating system kernel.. To accomplish this we 
need to get the kernel to read a what is called an [ATAG][2] list that's created by the firmware 
bootloader and placed in RAM on start.

```
        +-----------+
base -> | ATAG_CORE |  |                       a CORE tag always starts the list
        +-----------+  |
        | ATAG_MEM  |  | increasing address    MEM tags describe the memory layout
        +-----------+  |
        | ATAG_CMD  |  |                       CMD tags are commands for the kernel
        +-----------+  |
        | ATAG_NONE |  |                       a NONE tag always terminates the list
        +-----------+  v
```

Each tag in the list contains some information about the specific system that the
kernel is running atop of: memory layout, console info, &c. All of this information is provided
to the kernel by the bootloader without the kernel having any idea on how the setup was performed.
We'll access this information to then build our custom memory allocator, by building an `Iterator`
over these ATAGs.

We start by representing a raw ATAG with a struct:

```rust
#[repr(C)]
pub struct Atag {
    pub dwords: u32,
    pub tag: u32,
    pub kind: Kind,
}
```

Each `Kind` of tag will contain differing types of information and yet exist in within the
same region of memory as another kind. Because of this, it's necessary we use a C-like `union`
so that we can store different struct layouts in the same memory space. Just as in C, the
union will have the maximum size and alignment of any of it's fields.

```rust
#[repr(c)]
pub union Kind {
    pub core: Core,
    pub mem: Mem,
    pub cmd: Cmd,
}
```

With each of the types `Core`, `Mem`, and `Cmd` describing, as structs, the kind of information
the tag contains. All's well, however the compiler cannot make guarantees that the end-user
of this raw `Atag` will always read the correct type of a raw union. Thus, manipulating a raw Atag is
memory unsafe.

Before we start implementing our conversion from raw structures to the safe structures we still
need a way of determining the address of the ATAG following the one pointed to by `self` and
return a reference to it.

The signature is:
```rust
fn next(&self) -> Option<&Atag>
```

Where `&self` is the current `Atag` reference. All ATAGs contain a field that gives it's
size in _double words_ (32-bit words) and we can use this information to know at what
offset from the pointer to `self` the *next* ATAG in the list lives.

To do all of these we'll need to do some fancier type casting to get to the raw pointer,
compute the offset to the next, before eventually casting that to a `&Atag`.

Raw pointers in Rust are described using the type `*const T` and `*mut T`, on our 32-bit
ARM processor our pointers have the shape `T` of `u32`. Since we cannot cast from a reference
type to a raw 32-bit pointer directly, we cast to a raw pointer to the `Atag` type and then to
a `*const u32` like so:

```rust
let curr = self as *const Atag as *const u32
```
Before then calculating the [offset][1] to the next ATAG:
```rust
let next = curr.offset(self.dwords) as *const Atag
```
And finally converting from `*const T` to `&T` by dereferencing the pointer and grabbing
it's reference, giving us:
```rust
impl Atag {
    pub const NONE: u32 = 0x00000000;
    pub const CORE: u32 = 0x54410001;
    pub const MEM: u32 = 0x54410002;
    ...

    /// Returns the ATAG following `self`, if there is one.
    pub fn next(&self) -> Option<&Atag> {
        match self.tag {
            Atag::NONE => None,
            _ => {
                let curr = self as *const Atag as *const u32;
                let next: &Atag = unsafe {
                    &*(curr.offset(self.dwords as isize) as *const Atag)
                };
                Some(next)
            }
        }

    }

```
We now have a way of traversing the ATAG list using some pattern matching on the
tag value and some simple pointer arithemetic. This will later be used to create our
safe `Iterator` implementation.

To expose this raw union type safely, we can leverage the safer tagged-union, or `enum`, type
which gives us the advatage of unions but with the added security of memory safety since
the enumeration will always keep track of whatever particular type it contains.

```rust
#[derive(Debug, Copy, Clone)]
pub enum Atag {
    Core(raw::Core),
    Mem(raw::Mem),
    Cmd(&'static str),
    Unknown(u32),
    None
}
```
With our tagged-union type in place, how to do we go about converting our `raw::Atag` types
to the safer enumerated version?

Rust provides us with a useful trait called [`From`][3] for dealing with concept of performing
type conversions (safely!) from some type `T` into whatever `U` type the trait is being 
implemented for.

Since each variant of the `Atag` enum is simply a wrapper around their raw untagged counterpart
implementing the `From` trait for each `raw::Kind` should be as simple as pattern matching on
the structure of each atag and it's associated `Kind` type before then wrapping it with the
appropriate `enum` variant:

```rust
/// Convert from raw::* types into Atag
impl<'a> From<&'a raw::Atag> for Atag {
    fn from(atag: &raw::Atag) -> Atag {
        unsafe {
            match (atag.tag, &atag.kind) {
                (raw::Atag::CORE, &raw::Kind { core }) => Atag::Core(core),
                (raw::Atag::MEM, &raw::Kind { mem }) => Atag::Mem(mem),
                ...
                (raw::Atag::NONE, _) => Atag::None,
                (id, _) => Atag::Unknown(id),
            }
        }
    }
}
```

Though special attention should be made when converting the struct:

```rust
pub struct Cmd {
    /// The first byte of the command line string.
    pub cmd: u8
}
```

into the `Cmd(&'static str)` variant our enum. Noting that the `cmd` is a single byte _from
the start of the null-terminated C-style string_ we know that `&cmd` is the start address of
that string and that before allocating the string it's size must be found by looping over it
until we hit the null terminator `\0` before then casting the start address and found size
into a slice using [`slice::from_raw_parts()`][4] and then finally into a string using
[`str::from_utf8()`][5]. With a string represented like:

```
 0x0fe  0x102 0x106  0x10a  0x10e  0x112
.-----.------.------.------.------.------.
| "A" |  "B" | "C"  |  "D" |  "E" | "\0" |
`-----'------'------'------'------'------'
 &cmd
```
We want to increments four bytes at a time when we [add][6] to our pointer so, we ensure our 
pointer is of size `u8`, as expected. Incrementing `len` until we hit a null-byte:

```rust
match (atag.tag, &atag.kind) {
    ...

    (raw::Atag::CMDLINE, &raw::Kind { ref cmd }) => {
        let ptr = cmd as *const raw::Cmd as *const u8;
        let mut len = 0;

        while !ptr.add(len).is_null() {
            len += 1;
        }

        let cmd = slice::from_raw_parts(ptr, len);

        Atag::Cmd(str::from_utf8_unchecked(cmd))
    },

    ...
}
```

With all this unsafe business out of the way, we can now finish our ATAG `Iterator` trait
requiring no `unsafe` keyword!

Here is the type we'll be implementing `Iterator` for:

```rust
/// An iterator over the ATAGS on this system.
pub struct Atags {
    ptr: &'static raw::Atag,
}
```

The `ptr` field is a `raw::Atag` that we've already implemented a `next` method for using
unsafe code for pointer arithmetic. Trusting that ATAG list's layout is created as above
by the bootloader, ending with a `NONE` tag, the `next` method on `raw::Atag` should return
`None` if the tag is null, and we can ensure that the iterator will stop at the end of the 
ATAG list. With all this, let's finally put our iterator together as:

```rust
impl Iterator for Atags {
    type Item = Atag;

    fn next(&mut self) -> Option<Atag> {
        if let Some(next) = self.ptr.next() {
            let curr = self.ptr;
            self.ptr = next;
            Some(curr.into())
        } else {
            None
        }
    }
}
```

Making sure we're updating the current pointer, to ensure our iterator moves forward.
Since `Debug` has been derived for our `Atag` types, let's see what ATAGs have been
initialized by the bootloader by looping over our `Atags` iterator in `kmain.rs`:

```rust
#[no_mangle]
pub unsafe extern "C" fn kmain() {
    ...
    for atag in Atags::get() {
        // pretty-print
        kprintln!("{:#?}", atag);
    }

    loop { run_shell() }
    ...
}
```

Booting up the Pi and sending the new kernel binary over [UART][7], we see three ATAGs
printed to the screen:

```
Core(
    Core {
        flags: 0,
        page_size: 0,
        root_dev: 0
    }
)
Mem(
    Mem {
        size: 994050048,
        start: 0
    }
)
Cmd(
    "bcm2708_fb.fbwidth=656 bcm2708_fb.fbheight=416 bcm2708_fb.fbswap=1 dma.dmachans=0x7f35 bcm2709.boardrev=0xa02082 bcm2709.serial=0x91055cb9 bcm2709.uart_clock=48000000 smsc95xx.macaddr=B8:27:EB:05:5C:B9 vc_mem.mem_base=0x3ec00000 vc_mem.mem_size=0x40000000  console=ttyS0,115200 kgdboc=ttyS0,115200 console=tty1 root=/dev/mmcblk0p2 rootfstype=ext4 rootwait"
)
```

The MEM tag reports nearly a GiB of memory, about .074219 GiB less than the Pi's purported
1 GiB of RAM. We also see a long string of kernel parameters passed in via the CMD ATAG.

Part two of this blog post series on will be about writing heap allocators in Rust for
the Raspberry Pi's ARM processor using the ATAG iterator we created for this assignment.


[1]: https://doc.rust-lang.org/std/primitive.pointer.html#method.offset
[2]: http://www.simtec.co.uk/products/SWLINUX/files/booting_article.html#d0e428
[3]: https://doc.rust-lang.org/std/convert/trait.From.html
[4]: https://doc.rust-lang.org/std/slice/fn.from_raw_parts.html
[5]: https://doc.rust-lang.org/std/str/fn.from_utf8.html
[6]: https://doc.rust-lang.org/std/ptr/fn.add.html
[7]: https://mysterious.computer
