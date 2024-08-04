---
title: "Fixed timestep update loops for everything!"
date: 2024-08-04
description: "About update loops, interpolation, game state double-buffering, and correct input handling with fixed timestep"
---

[Fix your Timestep!](https://www.gafferongames.com/post/fix_your_timestep/) is an amazing article by Glenn Fiedler about fixed timestep update loops for games and physics simulations.
It explains how to properly accumulate a timer and run ticks with a fixed delta time within your main loop.

I use a slightly modified version of the update loop, here is a simplified version in Odin:

```c
DELTA :: 1.0 / 60.0

accumulator: f32
game: Game
prev_game: Game
prev_time := time.tick_now()

for !quit {
    frame_duration := f32(time.duration_seconds(time.tick_since(prev_time)))
    accumulator += frame_duration
    num_ticks := int(floor(accumulator / DELTA))
    accumulator -= f32(num_ticks) * DELTA
    for tick_index in 0 ..< num_ticks {
        runtime.mem_copy_non_overlapping(&prev_game, &game, size_of(Game))
        game_tick(&game, prev_game, input, DELTA)
    }
    alpha = accumulator / DELTA
    game_draw(game, prev_game, alpha)
}
```

The main difference is that I calculate number of ticks from the accumulator, instead of running a while loop and subtracting delta from the accumulator directly.
I find this a bit nicer but there isn't much difference.

## How engines separate update and physics ticks
It's fairly common for game engines to separate regular ticks and physics ticks. For example, unity components have an `Update` function which runs every frame and `FixedUpdate` function which runs 50 times a second (by default) during physics tick.

I really don't like this separation.

It often makes gameplay code which interacts with physics much more convoluted, because it forces you to split your update logic into two places.
It can be very confusing for beginner programmers.
And in general setting up your simulation this way makes it easier to write frame-rate dependent and less stable gameplay code.

## A single fixed timestep tick for both gameplay and physics
So what's the alternative?

To have one big `tick` procedure which updates _everything_ in the game, including gameplay, physics, etc.
And then run this tick in a fixed update loop.

This way the game simulation is completely independent from "frames" and rendering.

It could be complicated to do this in an already existing engine, or if you use one of the common rigidbody physics engines.
But I have a custom engine and custom collision system, so I can set it up however I like :P

## Interpolating everything
Okay, what about rendering then?

You _could_ you just run your ticks in a fixed update loop, and then rendered the final game state every frame.
However the "tick time" and "frame time" doesn't match perfectly. And in cases when the frame rate is much higher than tick rate, the game could look very laggy.

The solution is interpolation.

Now `game_draw` takes two game states, and interpolates between them based on the alpha factor.
Here is an example of interpolating the camera and enemies:
```c
game_draw :: proc(game, prev_game: Game, alpha: f32) {
    camera := Camera{
        pos = lerp(prev_game.camera.pos, game.camera.pos, alpha),
        rot = slerp(prev_game.camera.pos, game.camera.pos, alpha),
    }
    
    for curr_enemy, i in game.enemies {
        prev_enemy := prev_game.enemies[i]
        enemy := curr_enemy
        enemy.transform = lerp_transform(
            prev_enemy.transform, enemy.transform, alpha)
        draw_model(.Grunt_Enemy, enemy.transform, COLOR_RED)
    }
}
```

The same concept applies to everything else, you can interpolate whatever properties you need.
You could probably even automate it in a proper entity system, or if you use a scripting language.

## Double-buffered game state
As you can see there are two copies of entire game state.
This makes it easy to do interpolation, and you can also use the previous game state for many things in gameplay code as well.

I initially got this idea from Dreams by Media Molecule.
It's a really smart engine architecture, which Alex Evans explained in one of their [streams](https://youtu.be/1Gce4l5orts?si=TMOJP0Bt_jxXazIm).

If you use fixed size datastructures and never dynamically allocate memory like I do, copying the game state is just a single big `memcpy` once per tick.
But in case you want to use dynamic memory, you could use [arena allocators](https://www.rfleury.com/p/untangling-lifetimes-the-arena-allocator) or some kind of memory pools to make copying game state trivial as well.

## What about input?
As you can see the `game_tick` procedure takes an `input` parameter, but where did that come from?
Glenn's article doesn't cover this, probably because he's mostly concerned with physics simulations. But in our case it's crucial to get the correct results.

Just for context, here is how my (simplified) input state looks like. This isn't very important, but note I don't use an "input event queue" in gameplay code. I just use procedures like `input_is_key_down(inp, .Left_Shift)` etc.
```c
Input :: struct {
    cursor_pos:    Vec2,
    cursor_delta:  Vec2,
    scroll_delta:  f32,
    mouse_buttons: [Input_Mouse_Button]bit_set[Input_Digital_Flags],
    keys:          [Input_Key]bit_set[Input_Digital_Flags],
}

Input_Digital_Flags :: enum u8 {
    Down,
    Changed_This_Frame,
    Repeated,
    Multiple_Changes_This_Frame,
}
```

Okay, now let's see how to pass the inputs into the `game_tick` procedure.

The naive solution is to just pass your regular input state like you would normally:
```c
frame_input: Input
// ...

for !quit {
    for event in events {
    case .Key_Down:
        input_key_down(&frame_input, key)
    // handle other (input) events...
    }
    
    // ...
        game_tick(&game, prev_game, frame_input, DELTA)
    // ...
    
    // Clear input flags for things like "pressed", "released" etc.
    // but keep "down" states
    input_clear_temp(&frame_input)
}
```

But, this has a number of problems:
1. Inputs are lost when there are no ticks to run for few frames.
2. Mouse delta is applied multiple times
3. Key states which should only be valid for one frame (like Pressed and Released) are passed into `game_tick` multiple times.

### Solution 1: Poll inputs every tick
This might technically be the "best" solution to this problem, because you also get the most accurate results this way.
However it might not be practical because your app only has one event loop, and you need regular frame inputs too.
You could probably poll events and then "save" non-input ones for later, but IMO it's probably unnecessary for many games.

### Solution 2: Accumulate tick inputs separately
This is what I went with, because it's simple and the accuraccy loss from not polling inputs every tick isn't noticable.

The idea is to keep _two_ separate input states, one for "per frame" stuff and the other one for ticks.
Both are updated with the same events at the start of your frame, but the state is updated in a different way during the frame.

First idea is to discard temporary key flags (like Pressed and Released) after every tick, so they aren't applied multiple times.

Then, we can also divide the mouse delta by number of ticks to apply it over time.
Alternatively you could skip this and set the delta to zero after every tick, but I think splitting the delta over multiple ticks is nicer.
Finally, you clear the rest of temporary input data once you ran all your ticks.

Here is the update loop example code:
```c
DELTA :: 1.0 / 60.0

frame_input: Input
tick_input: Input

accumulator: f32
game: Game
prev_game: Game
prev_time := time.tick_now()

for !quit {
    for event in events {
    case .Key_Down:
        input_key_down(&frame_input, key)
        input_key_down(&tick_input, key)
    // handle other (input) events, as well as mouse buttons and mouse move...
    }
    
    frame_duration := f32(time.duration_seconds(time.tick_since(prev_time)))
    accumulator += frame_duration
    num_ticks := int(floor(accumulator / DELTA))
    accumulator -= f32(num_ticks) * DELTA

    if num_ticks > 0 {
        tick_input.cursor_delta /= f32(num_ticks)
        tick_input.scroll_delta /= f32(num_ticks)
        
        for tick_index in 0 ..< num_ticks {
            runtime.mem_copy_non_overlapping(&prev_game, &game, size_of(Game))
            game_tick(&game, prev_game, tick_input, DELTA)
            // Clear temporary flags
            for &flags in tick_input.keys do flags &= {.Down}
            for &flags in tick_input.mouse_buttons do flags &= {.Down}
        }
        
        // Now clear the rest of the input state
        tick_input.cursor_delta = {}
        tick_input.scroll_delta = {}
    }
    
    alpha = accumulator / DELTA
    game_draw(game, prev_game, alpha)
}
```

And that's it! This is the solution I currently use in my engine and works well so far. Let me know if you run into any problems or if you have a better solution in mind :)

Thanks for reading!
