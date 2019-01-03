+++
title = "dynamic memory allocators in Rust"
date = 2018-01-03
+++
Welcome to part two of my CS140e blog post series, where I write down my notes as I work
through Stanford's [experimental course on operating systems in Rust][1]. Today, we're going
to write a dynamic memory allocator in Rust so that the kernel can use data structures
that are dynamically allocated during runtime  such as the collections `Vec` and `HashSet`. 
Basically this is `malloc` and `free`: Rust-style. This will be less constraining than only
being able the [stack][2] which is _bounded_ by a user-supplied slice to `StackVec`.

Rust 1.28 introduced a stable `#[global_allocator]` attribute that allows Rust programs to either
set the allocator used to the system allocator as well as create new allocators which implment
the `GlobalAlloc` trait. This is very nice for us because this means that just by properly
[implementing][3] `alloc` and `dealloc` within our custom allocator, we'll be able to manage
memory without having to know exactly how much memory our kernel will need at runtime.

## Memory Alignment

To set the stage for our memory allocators, let's first talk about a little thing that
most programmers don't have to know about: memory-alignment. Memory alignment is a sort
of convention for where objects are placed in memory. We say a memory address *k* is *n*-byte
aligned if `k mod n == 0`. C's default allocator guarantees 8-byte alignment on 32-bit systems
and 16-byte aligned on 64-bit systems. This is not without good reason. (TODO: Write reason)

Signatures for `malloc` and `free` in C:

```c
void * malloc(size_t size);
void free(void * pointer);
```

And the related signatures of `alloc` and `dealloc` in Rust's unsafe `GlobalAlloc` trait:

```rust
unsafe fn alloc(&self, _layout: Layout) -> *mut u8;
unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout);
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
                (mem.start as usize, (binary_end + mem.size) as usize)
            );
        }
    }

    None
}
```

It's imporant to note that the MEM ATAG reports the amount of *total system memory* in RAM,
however, and not the amount of actually free memory. Our instructor has helpfully defined
for us `binary_end` that holds the first address of memory after the kernel

[1]: https://web.stanford.edu/class/cs140e
[2]: https://github.com/donkey-hotei/cs140e/blob/1102a214a1f4d2254a761ac8551db45db9c7f216/1-shell/stack-vec/src/lib.rs
[3]: https://doc.rust-lang.org/std/alloc/trait.GlobalAlloc.html
[4]: https://doc.rust-lang.org/std/primitive.usize.html#method.is_power_of_two
[5]: https://mysterious.computer/safely-exposing-unsafe-api