---
title: "Intro to Odin's core libraries for gamedev beginners"
date: 2024-08-15
description: "A beginner-friendly overview of all the useful packages you can find in Odin's base/core/vendor library collections."
draft: true
---

Hey! I decided to write down a list of packages from Odin's base, core and vendor libraries I use most often when developing games.
And also explain why I use them and which parts in particular are useful to me.
I also included short code examples to get you going.

When I first started using Odin few years ago it took me no time at all to learn _the language_, but I struggled with core library for a bit.
It's not that complicated, but there are many things which can be a bit overwhelming for a newcomer.
The package organization can also seem a bit chaotic at times.

In case you want to take a look at the complete list of packages, check out [Odin Package Docs](https://pkg.odin-lang.org/)

## core:os

OS is a package for interacting with the operating system in a cross platform way.
The most common use-case is reading and writing files, which can be done like so:
```odin
// Temp allocator makes sure you don't need to explicitly free.
// read_entire_file also returns a bool ok value, let's handle that with or_return
data := os.read_entire_file("foo.bin", context.temp_allocator) or_return
// Do something with data...
// Don't truncate binary data.
os.write_entire_file("bar.bin", data, truncate = false) or_return
```

You can also do things more directly with just `os.open`, `os.read` and `os.write`. Also don't forget to call `os.close` on file handles!

Currently there is an `os2` package in the works, because the `os` pacakge is one of the oldest packages in Odin's core library and it really needed a rewrite.
But unless you're doing something advanced it's good enough.

## core:fmt
This is probably the first package you ever used, since it's nice for writing a simple hello world program, like so:

```odin
package main
import "core:fmt"

main :: proc() {
    fmt.println("Hellope!")
}
```

But there are a few other useful procs you might want to use as well.

```odin
my_value: f32 = 1.234
some_string := fmt.tprint("Hello", 123, my_value)
// Alternatively the 'f' variant for more control.
// Read more about the options in fmt docs.
other_string := fmt.tprintf("my value = %.3f", my_value)
// There is also this helper for generating a C string,
// this is very useful when using Raylib for example.
some_cstring := fmt.ctprint("Hello", "World!")
```

## core:math
Odin has a lot of built-in math procs, like `abs`, `clamp`, etc.
The math package contains all the other things which aren't available by default.

Here is a list of the ones I use most often:
- sin, cos
- pow, sqrt
- round
- floor (round towards negative)
- ceil (round towards positive)
- trunc (round towards zero)
- fract (float fractional part)
- exp
- to_degrees, to_radians
- saturate (clamp between 0 and 1)

### core:math/linalg
This package implements many math procs which you might find useful when doing 2D/3D computations with vectors, matrices and quaternions.

There are euqivalents of all the procs I mentioned in the `math` package but which work on entire vectors.
So for example, you can do `linalg.floor([3]f32{1.2, 3.6, -1.3})` to floor all the components in a 3D vector.

Again, here is a list of the most useful procs for vectors:
- length
- normalize (makes sure length = 1)
- dot, cross
- reflect
- lerp

Here are the common procs for quaternions, which represent a rotation in 3D space:
- quaternion_inverse
- quaternion_nlerp/quaternion_slerp
- quaternion_angle_axis

### core:math/rand

The rand package, as the name suggests, exists for generating pseud-random numbers.

### core:math/noise

## core:mem

## core:reflect

## core:strings

## core:encoding/json

## core:log

## core:time

## core:path/filepath

## core:slice