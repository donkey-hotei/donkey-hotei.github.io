+++
title = "dynamic memory allocation"
date = 2018-01-03
+++
Welcome to a cs140e blog post series, where I write down my notes as I work
through Stanford's [experimental course on operating systems in Rust][1]. Today, we're going
to walk through creating a dynamic memory allocator in Rust so that the kernel can use data structures
that live in the heap like the `Vec` and `HashSet` types. Think `malloc` and `free`: Rust-style.

Here are the signatures for `malloc` and `free` in C:

```c
void * malloc(size_t size);
void free(void * pointer);
```

And the related signatures of `alloc` and `dealloc` in Rust ([GlobalAlloc][3]):

```rust
unsafe fn alloc(&self, _layout: Layout) -> *mut u8;
unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout);
```

Later, we'll dive deeper into these type signatures.

Before writing a heap allocator, "What is the heap?".
The [Book][6] has a very good description of the characteristics of the heap and the stack
in the ownership chapter. In summary, we can say the heap is where we put objects that are 
_potentially_ variable and large in size and the stack consists of those objects whose size is typically
smaller and known at runtime. Since many collection types like `HashMap` and `Vec` can
change size during runtime, only by having a something to allocate them dynamically can
we unlock their powers.

![alt text][heap]

The details will vary from system to system, but on the Raspberry Pi the heap will live
directly after the kernel's binary in RAM. We assume that it's an area of demand-zero[^1]
memory that begins after the uninitialized `.bss`[^2] area and moves upwards (towards higher
addresses).

## Memory Alignment

```rust
/// Align `addr` downwards to the nearest multiple of `align`.
///
/// The returned usize is always <= `addr.`
///
/// # Panics
///
/// Panics if `align` is not a power of 2.
pub fn align_down(addr: usize, align: usize) -> usize {
    assert!(align.is_power_of_two());

    (addr / align) * align
}

/// Align `addr` upwards to the nearest multiple of `align`.
///
/// The returned `usize` is always >= `addr.`
///
/// # Panics
///
/// Panics if `align` is not a power of 2.
pub fn align_up(addr: usize, align: usize) -> usize {
    align_down(addr + align - 1, align)
}
```


The `Layout` type describes the particular layout of a block of memory. This type has two
getter methods `layout.size()` and `layout.align()` that return the requested size of our
block and the alignment of that memory block's address.

`GlobalAlloc` is an unsafe trait (like `Send` and `Sync`) for a variety of reasons:

1. `alloc` can cause undefined behavior if the caller does not ensure that the `layout`
   argument has non-zero size.
2. That allocated block of memory may or may not be initialized.
3. `dealloc` will cause undefined behavior the caller does not ensure that
    the `ptr` denotes a block of memory returned by `alloc` (allocated via this
    allocator) and that the memory `layout` is the same as that used to allocate
    the block of memory we're freeing.

In writing this allocator one make sure that the addresses returned are properly aligned.
A difference from C we see here is that Rust split the responsiblities of dynamic memory 
allocation such that the caller of `alloc` and `dealloc` must specify the the alignment.
`alloc` puts the burden of responsiblity on the allocator to return an address in memory
that is properly aligned while `dealloc` puts the onus on the caller to keep track of the
memory layout that was previous used to allocate the address being free'd. Rust has more
restrictions on memory alignment than C does. The benefits of this are that all common
code are in the same location avoiding the same code being written for every allocator.
This goes well with Rust's power to abstract away details from the user and additionally
allows the Rust compiler to perform some langauge-level optimizations without involving
the allocator.

Back to memory alignment: we need to write two utility functions to aide us, `align_up`
and `align_down` which will find the closest memory address upwards or downwards to
the nearest multiple of some power of two. The power of two thing is _just a convention_,
but a good one at that because it's how computers are built! Multiplication, division,
addition, and subtraction, are all much easier to do (i.e: faster) than attempting to do
so with non-binary powers.

In a few simple lines, including an assertion that the `align` argument is a power of [two][4].

## Thread Safety

There is already extensive literature on creating thread-safe memory allocators. The reason
is because most allocators are _global_ to the scope of a program and `dealloc` and `alloc`
are certainly *not* reentrant functions, as they operate on a slab of shared data. What happens
if two separate threads (which we don't support yet) calls `alloc` at the sime time? Could
they receive the same address? If so, how large would the allocated region be? Rust, as a
language, takes these questions of concurrency very seriously. It is difficult to write
programs in Rust that isn't thread-safe. For our part the instructor has simply wrapped the 
allocator in a `Mutex` ensuring thread-safety by mutual-exclusion.

```rust
#[path = "bump.rs"]
mod imp;

/// Thread-safe (locking) wrapper around a particular memory allocator.
#[derive(Debug)]
pub struct Allocator(Mutex<Option<imp::Allocator>>);
```

Note here that the `imp` module is _virtual_ and not actually backed by a file of the same
name on the file-system but instead given a path to the allocator of our choosing. In this
lab we're going to write two different kinds of allocators so having an ability to easily
switch between implementations is really convienent. Our first allocator is called a "bump
allocator" and our second is a "bin allocator". We'll go into more detail as to how these
two memory allocators work further along in the post.


## Mapping Memory

Here is the implementation of our generic memory allocator:

```rust
impl Allocator {
    /// Returns an uninitialized `Allocator`.
    ///
    /// The allocator must be initialized by calling `initialize()` before the
    /// first memory allocation. Failure to do will result in panics.
    pub const fn uninitialized() -> Self {
        Allocator(Mutex::new(None))
    }

    /// Initializes the memory allocator.
    ///
    /// # Panics
    ///
    /// Panics if the system's memory map could not be retrieved.
    pub fn initialize(&self) {
        let (start, end) = memory_map().expect("failed to find memory map");
        *self.0.lock() = Some(imp::Allocator::new(start, end));
    }
}
```

The `initialize` method constructs an instance of the internal `imp::Allocator` structure
for later allocations and deallocations. It in turn calls `memory_map` to find out the start and 
end point of a region the region memory that has been mapped by the Raspberry Pi's bootloader
on start. We can implement this using the `Atags` implementation of the [previous post][5]!

Since the MEM ATAG gives us the amount of memory available in RAM, we can just take the
start address in the tag (likely 0x0000000) and then add the size to that number to get
the end address before returning them both as a tuple `(start, end)`.

```rust
// Holds the first address after the kernel's binary.
extern "C" {
    static _end: u8;
}

use pi::atags::Atags;
/// Returns the (start address, end address) of the available memory on this
/// system if it can be determined. If it cannot, `None` is returned.
///
/// This function is expected to return `Some` under all normal cirumstances.
fn memory_map() -> Option<(usize, usize)> {
    let binary_end = unsafe { (&_end as *const u8) as u32 };

    for atag in Atags::get() {
        if let Some(mem) = atag.mem() {
            return Some(
                (binary_end as usize, (binary_end + mem.size) as usize)
            );
        }
    }

    None
}
```

It's imporant to note that the MEM ATAG reports the amount of *total system memory* in RAM,
however, and not the amount of actually free memory. Our instructor has helpfully defined
for us `binary_end` that holds the first address of memory after the kernel

## Bump Allocator

Our first allocator will be the *dumbest of all allocators*. Whose behavior is specfified as:

1. Initialized with the pointer at the beginning of our heap space
2. When we request to `alloc` n-bytes of memory, the `current` pointer is bumped forward
   n-bytes (plus some alignment) and the old value of the pointer is returned.
3. When we `dealloc` an address, nothing actually happens.

Here's a diagram from the assignment's page that depicts what happens to the `current` pointer
after a `1k` byte allocation and a subsequent `512` byte allocation. Though alignment concerns
are abset in the diagram.

![alt text][bump]

We're tasked with implementing the `new()`, `alloc()`, and `dealloc()` methods of the
`bump::Allocator` using our `align_up` and `align_down` utility functions to ensure
proper alignment of the return address.

The instructor wrote a good number of tests to run our solution against, though this is one
of those times where the Rust core API has changed considerably enough over the past year
that we have to make some changes to get them running. I'll document this process here for
others taking the class years after it was first created.

The first bug we run into is related to a missing module `raw_vec` that supposedly contains `RawVec`:

```rust
error[E0432]: unresolved import `core::alloc::raw_vec`
  --> src/allocator/tests.rs:63:22
     |
     63 |     use core::alloc::raw_vec::RawVec;
        |                      ^^^^^^^ could not find `raw_vec` in `alloc`
```

Looking for `RawVec` in the standard documentation yielded no results though checking out
the official Rust repo we see a `RawVec` [implementation][8] in liballoc with the `#![doc(hidden)]`
attribute, which is why it doesn't appear in the doucmentation.

## Using the Allocator

Rust 1.28 introduced a stable `#[global_allocator]` attribute that allows Rust programs to either
set the allocator used to the system allocator as well as create new allocators which implment
the `GlobalAlloc` trait. This is very nice for us because this means that just by properly
[implementing][3] `alloc` and `dealloc` within our custom allocator, we'll be able to manage
memory without having to know exactly how much memory our kernel will need at runtime.



[^1]: A region (or page, though we haven't implemented paging yet) of memory that's mapped
to an anonymous file. It's _demand-zero_ because no data is actually transferred and s.t.
their sizes are initially zero.
```
[Sections]
00 0x00000000     0 0x00000000     0 -----
01 0x000000c0 20860 0x00080000 20860 --r-x .text
02 0x00005240  6066 0x00085180  6066 --r-- .rodata
03 0x00006a00  2848 0x00086940  2848 --rw- .data
04 0x00000000     0 0x00087460     0 --rw- .bss       <-- demand-zero
05 0x00007520  5976 0x00000000  5976 ----- .symtab
06 0x00008c78  6233 0x00000000  6233 ----- .strtab
07 0x0000a4d1    52 0x00000000    52 ----- .shstrtab
08 0x000000c0 29792 0x00080000 29792 m-rwx LOAD0
09 0x00000000     0 0x00000000     0 m-rw- GNU_STACK  <-- demand-zero
10 0x00000000    64 0x00080000    64 m-rw- ehdr

```

[^2]: As per [wikipedia][7]: "The name .bss is used by many compilers and linkers for the portion
of the executable file containing _statically-allocated variables_ that are not explicitly initialized
to any value."

[1]: https://web.stanford.edu/class/cs140e
[2]: https://github.com/donkey-hotei/cs140e/blob/1102a214a1f4d2254a761ac8551db45db9c7f216/1-shell/stack-vec/src/lib.rs
[3]: https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html
[4]: https://doc.rust-lang.org/std/primitive.usize.html#method.is_power_of_two
[5]: https://mysterious.computer/safely-exposing-unsafe-api
[6]: https://doc.rust-lang.org/book/ch04-01-what-is-ownership.html?highlight=stack,and,heap#the-stack-and-the-heap 
[7]: https://en.wikipedia.org/wiki/.bss
[8]: https://github.com/rust-lang/rust/blob/master/src/liballoc/raw_vec.rs

[heap]: https://i.ibb.co/BCzK1cN/Selection-367.png#center
[bump]: https://web.stanford.edu/class/cs140e/assignments/2-fs/images/bump-diagram.svg