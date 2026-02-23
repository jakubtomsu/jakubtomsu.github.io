---
title: "You don't need free lists!"
date: 2026-02-20
description: "An alternative to free-list-based pools using multi-level bit sets"
---

Pool-like datastructures are incredibly practical, especially when [combined with generational handles](https://floooh.github.io/2018/06/17/handles-vs-pointers.html).
They let you have an array of reusable item slots, but crucially all actions such as insertions, removals and lookups are *always O(1)*, including worst case! (Assuming you preallocate all buffers)

The usual way to implement the slot-reuse mechanism is to store some kind of a list of currently unused slots.
There are a bunch of different ways to implement this list, from a dumb array with slot indices, an intrusive/non-intrusive singly-linked-list.
They essentially do the same, except minor memory overhead differences.

Here's how the datastructure internals could look like:
```c
Pool :: struct($N: int, $T: typeid) {
    data:   [N]T,
    next:   [N]u32, // per-item free list 'next'
    free:   u32, // free list head
    max:    u32, // max used slot
}
```

But I have been using a very different approach lately, which is *NOT* based on any kind of free-list-like system.
It has a few nice performance properties:
- only ~1 bit overhead per slot
- significantly better slot allocation strategy with improved memory locality

My approach is inspired by the [TLSF memory allocator](http://www.gii.upv.es/tlsf/), though it should be a lot simpler to understand and implement from scratch.

## Demo

{{< x user="jakubtomsu_" id="2025892087451816092" >}}

# Bits

The idea is to just store a single bit, which determines whether a slot is in use or not.
Let's start with this, though we will improve it a lot later:
```c
Pool :: struct($N: int, $T: typeid)
    where N % 64 == 0
{
    data:   [N]T,
    used:   [N/64]u64,
}
```
> For the rest of this article we assume all pools store a multiple of 64 elements.

This of course has a fatal flaw - once you have thousands of items, you need to search through all the individual bits.

## *tzcnt* is your friend

It's wasteful to search the 64 bits of each `used` field individually with something like a loop of shr+mask+branch.

There is `tzcnt`, which stands for trailing zero count. It's an x64 instruction which has been supported on all consumer chips for many years. It's exposed to the programmer in the form of compiler intrinsics, and it lets you find the index of a lowest binary 1 in essentially a [single cycle](https://uops.info/html-instr/TZCNT_R16_R16.html).

And while a smart optimizing compiler could convert a tiny loop into a single `tzcnt` call, using it yourself explicitly makes it clearer what's going on and it will be fast even in debug builds.

> In Odin, I use a procedure called `count_trailing_zeros` from the `base:intrinsics` package, which implements this in a cross-platform way.

## Searching slots with *tzcnt*

All you need to do to find an empty slot in a single `u64` is to call `count_trailing_zeros(~used[index])`. We're negating the used mask to actually *count trailing ones* (used slots) to *find the first zero* (unused) bit index.

The code could look like this:
```c
find_empty_slot :: proc(pool: $T/$Pool($N, $V)) -> (result: int, ok: bool) {
    for i in 0..<N/64 {
        bit := int(intrinsics.count_trailing_zeros(~pool.used[i]))
        // check for full block
        if bit != 64 {
            return i * 64 + bit
        }
    }
    // Pool is full
    return 0, false
}
```

Okay, this is quite a nice improvement, but you still have to search `N/64` u64s in the worst case.
Can we improve this? Can we somehow quickly search among all those u64s and find one that isn't full?


# 2-level bit arrays

The answer is yes. The solution is another, but smaller bit array.
While the original bit array stores 0/1 for each slot, we can add another array to store bit per each block, which is 1 when a block is full.

The actual struct ends up something like this. I'll only implement the bit pool part without the actual data, since that's how I use it in my code.
```c
Bit_Pool :: struct($N: int)
    where N % 64 == 0
{
    // round up div by 64*64
    l1: [(N + 4095) / 4096]u64,
    // 'used' array from before
    l0: [N / 64]u64,
}
```

Each field in the L1 array stores info about 64 fields in L0. This way each `tzcnt` instruction effectively searches 4096 slots.

The arrows in this illustration show the implicit mappings between the data:
```
data             L0               L1
┌──────────┐   ┌──────┐      ┌────────┐
│0..63     │◄──┤0     │◄─────┤0       │
├──────────┤   ├──────┤      ├────────┤
│64..127   │◄──┤1     │   ┌──┤1       │
└──────────┘   ├──────┤   │  ├────────┤
               │...   │   │  │...     │
┌──────────┐   ├──────┤   │  │────────│
│4032..4095│◄──┤63    │   │  │N/4096-1│
└──────────┘   └──────┘   │  └────────┘
┌──────────┐   ┌───────┐  │
│4096..8191│◄──┤64..127│◄─┘
└──────────┘   └───────┘
```

## Search

This is how the code looks like. There is a loop so while the "theoretical asymptotic complexity" is still based on N.
In practice, the N is either quite small so this loop is taken only a few times, or you [add more levels](#more-levels), which scales logarithmically in memory footprint and with a tiny constant overhead at each additional level.

```c
find_0 :: proc(bp: Bit_Pool($N)) -> (index: int, ok: bool) {
    l0_index := N > 64 ? -1 : 0 // ignore the L1 in small pools
    // Find suitable L0 block by searching L1
    for used, i in bp.l1 {
        l1_slot := int(intrinsics.count_trailing_zeros(~used))
        if l1_slot != 64 {
            l0_index = 64 * i + l1_slot
            break
        }
    }

    if l0_index == -1 || l0_index >= (N / 64) {
        return -1, false // Pool is full
    }

    // Find the actual slot within the L0 block
    l0_slot := int(intrinsics.count_trailing_zeros(~bp.l0[l0_index]))
    if l0_slot != 64 {
        return l0_index * 64 + l0_slot, true
    }

    return -1, false // Pool is full
}
```

As you can see this has a nice "front loading" property, where it always allocates slots with the lowest index first. This is *great* for memory locality, especially in burst-like slot allocation patterns.
A regular list-based pool would scatter the values all across the pool!

The generated code also looks reasonably nice, LLVM can unroll the loop in many cases and even collapse it.
Check out this [Godbolt example](https://godbolt.org/z/hq88fGrdG) where N = 8192.

### Code

A full implementation is on [Github](https://github.com/jakubtomsu/raven/blob/main/base/base_bit_pool.odin), as a part of my Raven engine. It includes procedures for actually setting bits and lookups.

## More levels

If your **N** gets *significantly* larger than 4096, you could trivially add more levels. It's still all O(1) in every case. The only reason I don't do this is because my pools tend to be relatively small, tens of thousands of items at most.

For example, if you added `l2` bit array, a single `tzcnt` instruction would effectively search 64^3 (262144) slots. That is a lot of slots.

## AoSoA

This block-based approach lends itself to SIMD AoSoA (arrays of structures of arrays).

The L0 mask is the "ground truth" determining free slots, but since it's a bitmask of packed elements it's exactly the layout you would use as a SIMD op mask or something similar.

I quite enjoy keeping datastructures simple, instead of wrapping everything in multiple levels of generic containers. Here's a an example of AoSoA combined with Bit_Pool, for something like a particle system:
```c
MAX_FOOS :: 1024 * 16

Foo_Batch :: struct {
    pos:    [3]#simd[64]f32, // xyz 64 times
    vel:    [3]#simd[64]f32, // xyz 64 times
    timer:  #simd[64]f32,
    flags:  #simd[64]u16,
}

// Each value in used.l0 corresponds to one batch
Foos :: struct {
    used:       Bit_Pool(MAX_FOOS),
    batches:    [MAX_FOOS / 64]Foo_Batch,
}
```

Explaining this in more detail is out of scope for this article, but I'll leave this here just as a vague idea for inspiration.

## Possibly atomic?

I haven't really looked into this yet, but I wonder if insertion/removal could be completely lockless using atomics.
It seems relatively trivial in the one-level case, maybe I could order the insertion and removal memory writes to eliminate data races?

In case you figure something out, please do let me know! I would be very interested in experiments and also correctness proof.

# Conclusion

This approach is a bit more complicated than a dumb free slot array, but I really like the fact it's just a basic bit array, no super complicated metadata to take care of.

I'm not necessarily trying to convince anyone to use this approach, however I think these kinds of algorithms are still relatively uncommon, so it was worth writing about. It's certainly nothing novel, but I hope it's valuable!

Thank you for reading!

Jakub

> 2026-02-22: Minor updates, correct free list memory overhead
> 2026-02-23: Typo fixes, demo