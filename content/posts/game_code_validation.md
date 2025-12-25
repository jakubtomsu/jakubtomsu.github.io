---
title: "Gameplay code validation and catching NaNs"
date: 2025-06-04
description: "A super simple, brute force method of validating gameplay code instead of unit testing"
---

I recently ran into some NaN propagation bug in my gameplay code, which required a bit more effort than usual.

Often, those problems are straightforward to solve: it's just a forgotten zero-check before dividing, normalizing zero-length vector, etc.
I almost always find it immediately by looking through my most recent changes. But this time I had no idea!

So I came up with a simple way to *help* catching these bugs in general (in fact, it's so simple it's probably the first thing you would think of too).
But it's a nice pattern, so I thought I would share it anyway.

## Gameplay testing in general
Testing games can be really complicated, since the final output is just pixels and audio.
I have few "real unit tests" in my codebase, mostly for things like file formats.
Those usually look like this:
1. Setup state
2. Execute some logic
3. Assert correct results

That's just *not useful* for most gameplay code.

Most of the bugs happen when the setup gets complex enough and the multiple systems interact together.
And the expected results change all the time as you're iterating on your game design.

So, what if instead of writing isolated tests, I just inserted the correctness validation (step 3) throughout the codebase?
It limits what you can do, but IMO it's still useful.

## Validation
First I created validation procedures for common data types, e.g. a float:

```odin
validate_f32 :: proc(x: f32) {
    assert(!math.is_nan(x))
    assert(!math.is_inf(x))
}
```

Then it's easy to use this for validating other data:
```odin
validate_vec :: proc(v: [$N]f32) {
    for f in v {
        validate_f32(f)
    }
}
```

And then validating actual entities:
```odin
validate_entity :: proc(ent: ^Entity) {
    validate_vec(ent.position)
    validate_vec(ent.velocity)
    // arbitrary limits depending on what's reasonable for the game
    assert(ent.position.y > -1000)
    assert(linalg.length(ent.position) < 10000)
    assert(linalg.length(ent.velocity) < 10000)
}
```

The validation procedure asserts the data isn't in a completely unreasonable state.

> I also add a `@(disabled = !DEV_BUILD)` to all those procedures since I don't really need validation in release mode.

## Instrumenting code with validation
Now, let's just slap the validation in the most important parts of the codebase.
I know this is really grug-brained, but just a few calls in the right places got me a lot of coverage!

The point is to *reliably* catch possible errors soon after they happen, rather than letting them propagate for multiple frames before the issue becomes visible.
This is about *debuggability* - not necessarily 100% correctness.

Here is a pattern I use a lot: validate every entity at the start and the end of interface procs like `entity_tick`, `entity_damage`, `entity_interact` etc.
The defer makes sure it's called on every return.

```odin
entity_tick :: proc(ent: ^Entity, delta: f32) {
    validate_entity(ent^)
    defer validate_entity(ent^)

    // gameplay logic...
}
```

This is kinda like the "assert everywhere" advice, just in a slightly different form.

With this in place, I'm significantly more confident I can debug issues with NaN etc much easier.

## Failed approach: catching FP exceptions

Before this approach I also tried to catch FP exceptions directly at the source.
Turns out this isn't really viable due to SIMD codegen of modern compilers (I won't go into details here...)

But in case you wanna try it yourself, here is a [Github gist](https://gist.github.com/jakubtomsu/361f9319f73ed0ae9c0e36df887226b9) for enabling the exceptions on win32 with Odin.
