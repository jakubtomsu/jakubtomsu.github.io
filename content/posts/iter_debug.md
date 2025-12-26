---
title: "Coroutine-less way to visualize iterations"
date: 2025-12-26
---

Let's say you're working on a procgen system, or a similar problem where you combine multiple iterative algorithms to get visual results.
It could look something like this:

```odin
// Imagine this is for a tile-based 2D game or something
generate_level :: proc(some_params: ...) {
    some_temp_state_1, some_temp_state_2: int
    for i in 0..<N {
        // generate land tiles ...
    }
    for iter in 0..<4 {
        // run cellular automata ...
    }
    for x in 0..<SIZE do for y in 0..<SIZE {
        // spawn entity ...
    }
}
```

> All the logic being contained within one big procedure (possibly with a few helpers) is quite nice, especially for prototyping.

But, what if something goes wrong, and the debugger, logging or 2D/3D visuals don't help?

You might want to inspect what the actual ***iterations*** are doing.

## The usual solution

If you're using a managed language with coroutines, you can just `yield` from the procedure.

In other cases, you likely need to completely refactor your state management, put in a state machine, and split up the code a lot. I really don't want to do that.

## No-refactoring approach

So here's my 1-million-IQ solution.

What if you just **re-ran the entire procedure**, stopping at a different point each time? The observable effect is just like with coroutines.

So let's try that! Here's a helper proc to mark each "step" of the algorithm:
```odin
max_iter: int
curr_iter: int

_iter :: proc(n := 1) -> (ok: bool) {
    ok = curr_iter < max_iter
    curr_iter += n
    return ok
}
```

Now all you need is to instrument your `generate_level` procedure:
```odin
generate_level :: proc(some_params: ...) {
    some_temp_state_1, some_temp_state_2: int
    for i in 0..<N {
        // generate land tiles ...
        _iter() or_return
    }
    for iter in 0..<4 {
        // run cellular automata ...
        _iter() or_return
    }
    for x in 0..<SIZE do for y in 0..<SIZE {
        // spawn entity ...
        _iter() or_return
    }
}
```

> In case you're not using Odin, you can use `if (!_iter()) return` or another equivalent instead of `or_return`

And a way to control `max_iter`:

```odin
max_iter = 0
for app_main_loop_running {
    max_iter += 1
    generate_level(...)
    debug_draw_level()
}
```

And that's it. Of course it works only if `generate_level` is reasonably fast, otherwise you'd have use other methods.

Thanks for reading, I hope it saves you some refactoring time.