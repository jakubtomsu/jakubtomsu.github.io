---
title: "Fixed timestep without interpolation"
date: 2024-08-04
description: "Temp render tick as a simple interpolation alternative for singleplayer games with fixed timestep ticks"
draft: true
---

Some time ago I wrote a [blog post](https://jakubtomsu.github.io/posts/input_in_fixed_timestep/) about the tick setup in my engine, why and how I use fixed timestep update loops. It also explained how to actually use it in practice when dealing with input etc. The game would keep a accumulator timer and only simulate steps with a constant delta time, which is great for stability and predictability.

This was mostly inspired by server code for multiplayer games, so I naturally also implemented game state interpolation, which multiplayer games use as well.

Turns out there are better ways to do it! I'm making a singleplayer game, so it's much easier to do what multiplayer games might call _client side prediction_. In our case it literally just boils down to storing an alternative, temporary game state and simulate this until the fixed time matches the real time.

# The "old" way

Here is what the **old** game loop looked like. The game kept track of the game state from the previous tick, and then used this for interpolation with the time remainder.
```odin
DELTA :: 1.0 / 60.0

accumulator: f32
game: Game
prev_game: Game
prev_time := time.tick_now()

for !quit {
    frame_duration := f32(time.duration_seconds(time.tick_since(prev_time)))
    prev_time = time.tick_now()
    accumulator += frame_duration
    for ;accumulator > DELTA; accumulator -= DELTA {
        runtime.mem_copy_non_overlapping(&prev_game, &game, size_of(Game))
        game_tick(&game, input, DELTA)
    }
    // Remainder to go from fixed time to real time.
    // Always in 0..1 range.
    alpha = accumulator / DELTA
    // Interpolates between T-1 and T game state with the alpha factor.
    game_draw(game, prev_game, alpha)
}
```

# Can we do better?
The fundamental issue is that the fixed timestep simulation does as many timesteps as possible, however because the frame-rate is variable the real time doesn't always match the fixed time.

So the solution is very simple, after the fixed timestep simulation is done, we need to simulate one more additional tick with the remainding time. However this additional tick could mess up our game state, because the time step is no longer fixed.

For this reason we **duplicate** the entire game state into a new, temporary render-only game state, and simulate the tick on this one. That way the fixed timestep game state stays untouched.

```odin
DELTA :: 1.0 / 60.0

accumulator: f32
game: Game
temp_game: Game
prev_time := time.tick_now()

for !quit {
    frame_duration := f32(time.duration_seconds(time.tick_since(prev_time)))
    prev_time = time.tick_now()
    accumulator += frame_duration
    for ;accumulator > DELTA; accumulator -= DELTA {
        // Simulate the game state (without keeping T-1 game state!)
        game_tick(&game, input, DELTA)
    }
    // Remainder to go from fixed time to real time.
    // Always in 0..1 range.
    alpha = accumulator / DELTA
    // "Create" a duplicate version of the game state so our
    // "render tick" doesn't affect the real game state.
    runtime.mem_copy_non_overlapping(&temp_game, &game, size_of(Game))
    // Simulate the temp state forward in time to match reality.
    game_tick(&temp_game, alpha)
    // No need to do any interpolation now, just render the game state as it is.
    // It matches real time pretty much perfectly.
    game_draw(temp_game)
}
```

> Note:
> This method probably won't work very well in cases when `game_tick` can take a very significant amount of time, or the game state is gigantic, e.g. in a big AAA game which needs to run the physics engine and whatnot.

In my case the _entire_ game state is trivially copyable, because:
1. I never store any pointers. ever. Only integer indexes or handles.
2. All of the datastructures are fixed size, no dynamic allocations.

This means I can easily do `memcpy` and get a duplicate game state without any issues. This makes things a whole lot easier to deal with.

# What about input?
Just like the last time, the `game_tick` needs to get the input from somewhere. This is easy, because the main update loop is pretty much the same like in the interpolation case! So the solution from my original post still applies, you can just use a separate `frame_input` and a `tick_input`.

But what about the temp render tick?

This is a bit trickier. To be honest I'm not sure what is the perfectly correct solution, especially if you implement the stateful render game state I mentioned in the previous section.

But I found re-applying the `tick_input` in exactly the same way as in the fixed update loop works decently well. You also want to clear the "pressed" and "released" flags from the input, because those were already applied in the fixed update loop. If you use my implementation from previous article you don't need to do this directly, the fixed update loop does this on it's own anyway.

# Tangent: Large delta
In case you wanted to run only a very small number of fixed update ticks per second, the method described above has some limitations. The temporary render tick always uses the last fixed tick, which means as your fixed delta increases your render tick might be doing a single, very large time step.

> This is not an issue in most singleplayer PC games. That's because you _probably_ want to simulate >30 ticks per second anyway, for stability reasons and just because there is no reason to push TPS lower.

The solution is to copy game state into temp game state _only_ if you simulated a fixed tick that frame. Otherwise you keep simulating the old temp game state. This also means you need to keep another separate remainder timer, because alpha is no longer valid for each render tick.

I don't do this in my game, simply because I didn't run into issues with the current, simpler approach. But I thought I should mention it anyway.

# Tangent: determinism and replay
This isn't related to the main topic of this post, because it applies to any kind of gameplay where the timestep is fixed (it doesn't matter whether you use prediction or interpolation). But if your gameplay is deterministic, meaning you always get the same game state T+1 by simulating game state T with the same input, there is a neat trick to do game replay.

Regular replay systems would have to periodically store the game state, which ends up consuming a lot of memory, even if you try to compress it. But if your game is deterministic you only need to store the _starting_ game state (or even just the starting settings to generate the game state), and then inputs for all the fixed timesteps. Then when you can just simulate the original game state N times to get any tick you want, once you need to do the replay.

By the way, this is what Media Molecule does in Dreams. The Trackmania racing games do this as well, to verify runs and make sure people aren't cheating. Even their 3d physics engine is fully deterministic! very cool stuff.

> Notes:
> 1. You need to make sure entities are always updated in the same order. This means deterministic O(1) datastructures like pools are your friend.
> 2. If you use random numbers then you need to make sure the seeds match at the start of every tick as well. You can probably get by storing only one seed along with the first
> 3. The stored replay gets invalidated once you change your gameplay logic, so this method is generally useful for debugging only.

# That's all!
Thank you for reading, I hope this can be useful in your engine/game project! Also let me know if you have any feedback:)

## Credit

After I published the original blog post, a number of people messaged me it helped them with use fixed timestep update loops in their game/engine as well. However some people complained specifically about the interpolation step, it can be kinda cumbersome to do, especially if you want to interpolate more properties than just the entity transform.

I was told about this method by `poyepolomix` on discord, a fellow handmade programmer. He uses it in his engine as well! He found out about it from an old [Jonathan Blow Q&A stream](https://youtu.be/fdAOPHgW7qM?si=chCqgUTNoOLuHDvy) which is a much more in-depth explanation of this entire topic.
