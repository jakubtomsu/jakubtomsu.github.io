---
title: "Simple Undo/Redo System in Odin"
date: 2024-05-28
description: "A Grug Brained approach to level editor history"
---

This is a short blog post about something I've had to implement recently: editor undo/redo functionality.

A few weeks ago I started working on a new project, it's a 3D FPS game inspired by Quake and other '90s shooters.
The world is a 16x16x16 uniform grid of blocks, and it loops infinitely. Here is a clip of the game prototype:
{{< twitter user="jakubtomsu_" id="1794059547146936800" >}}

This clip uses procedurally generated levels. Those are cool, but don't have that much structure and interesting stuff going on. So I quickly decided I need to hand-craft levels for them to reach the full potential.

So I wrote a simple level editor. It was surprisingly quick, I had all of the basic functionality implemented within a few hours. One of the problems I had to solve was undo/redo system, which seems very challenging but doesn't have to be. I found a very simple way to structure the code to make the implementation trivial.

My implementation is inspired by rxi's [Simple Undo System](https://rxi.github.io/a_simple_undo_system.html) and Dennis Gustaffson's [Undo for Lazy Programmers](https://blog.voxagon.se/2018/07/10/undo-for-lazy-programmers.html)

## Level Representation
First, let's define the data for our level. In my case it's very simple, but it's easy to extend if necessary.
```odin
Level :: struct {
    using info:   Level_Info,
    cells:        Level_Cells,
    detail_cells: Level_Detail_Cells,
}

Level_Cells :: [LEVEL_BOUNDS_X][LEVEL_BOUNDS_Y][LEVEL_BOUNDS_Z]Cell // Cell is a u8 enum

Level_Detail_Cells :: [LEVEL_BOUNDS_X][LEVEL_BOUNDS_Y][LEVEL_BOUNDS_Z]Detail_Cell

Level_Info :: struct {
    size:      IVec3,
    fog_color: Vec3,
}
```

The goal here is to organize the level representation into a bunch of "chunks" (mostly by size and importance). All miscellaneous level metadata goes into `Level_Info` but other big chunks of data are a separate member.

Note: this assumes all of your level data is statically allocated and trivially copyable. I do this for _all_ of my data anyway, I think there isn't a reason to use any dynamic allocation for the use cases I care about. I might write a blog about this another time, it's a very useful way to think about data and not many people talk about it. It's always borrow checker/RAII/Arenas/Custom Allocators...

## Save Points
Now, let's define a way to store a single "change" to the data. This acts as a save point the user can go back to.

```odin
Editor_Undo_Item :: union {
    Level_Info,
    Level_Cells,
    Level_Detail_Cells,
}
```

This stores a change to a part of a level. It's a [tagged union](https://odin-lang.org/docs/overview/#unions), so the total size is the size of the largest item.

This allows me to easily store a change to any part of a level. That's why I separated the level into chunks in the previous step, it makes it easier to organize changes into groups. In theory you _could_ save the entire level on every change. But this is almost as simple and can save a lot of memory.

This way every change takes up memory for the "worst-case", which might seem bad at first but it's actually completely fine. In my case one undo item is 4 kilobytes, which is almost nothing these days. This system also scales really well in cases where you modify _most or all_ of the cells, and doesn't create unexpected spikes. But the reasons for focusing on worst-case computation is a big topic, let's leave that for another blog post :)

## History Buffers
Let's define a way to store the actual edits within our editor state.

```odin
Editor :: struct {
    level: Level, // current level data
    undo:  Queue(2048, Editor_Undo_Item),
    redo:  Queue(2048, Editor_Undo_Item),
    // other editor state...
}
```

The editor stores two queues (ring buffers) of edits, one for Undo and one for Redo. The reason why I use a queue is to have the ability to "force push" an edit at the end. If I used a regular array/stack, I would need to shift all other items down by one slot. The queue I use comes from my own small library for static datastructures, but you could use something like `core:container/queue` as well.

```odin
editor_undo_push :: proc(ed: ^Editor, item: Editor_Undo_Item) {
    // Makes sure the item is always pushed back into the queue, even if it's full.
    queue_push_back_force(&ed.undo, item)
}
```

This is the procedure for pushing save points before any edits. I pass the state which will be changed, and then perform the change. Here is an example of placing wall blocks on a mouse click:

```odin
if input_pressed(.Mouse_Left) {
    editor_undo_push(ed, ed.level.cells)
    // Do any modifications to level.cells...
    ed.level.cells[cursor.x][cursor.y][cursor.z] = .Wall
}
```

## Ctrl+Z and Ctrl+Shift+Z
This is all I need to implement the actual undo/redo functionality. This logic is the same as in rxi's article. Pop from one buffer, and before applying the data push the current state to the other buffer.

```odin
block: if input_pressed(inp, .Z) || input_repeated(inp, .Z) {
    modifiers: bit_set[Input_Modifier] = input_modifiers_down(inp)
    
    // Pop the data from a change buffer depending on the operation
    // Breaks out of this entire scope if it's empty
    change: Editor_Undo_Item
    switch modifiers {
    case {.Left_Control}:
        change = queue_pop_back_safe(&ed.undo) or_break block
    case {.Left_Control, .Left_Shift}:
        change = queue_pop_back_safe(&ed.redo) or_break block
    case:
        break block
    }
    
    // Prepare a save point for the current data
    // Write the change to the current state
    prev: Editor_Undo_Item
    switch v in change {
    case Level_Info:
        prev = ed.level.info
        ed.level.info = v
    case Level_Cells:
        prev = ed.level.cells
        ed.level.cells = v
    case Level_Detail_Cells
        prev = ed.level.detail_cells
        ed.level.detail_cells = v
    }
    
    // Push the current data into the _other_ buffer
    switch modifiers {
    case {.Left_Control}:
        queue_push_back_force(&ed.redo, prev)
    case {.Left_Control, .Left_Shift}:
        queue_push_back_force(&ed.undo, prev)
    }
}
```

And that's it! Turns out this entire system is not many lines of code at all, and is efficient even when you use worst-case-sized statically allocated data structures. That said, it's not a one-size-fits-all method. If you have gigantic scenes you probably need to look into other approaches (maybe XOR and RLE compressed delta states? LZ4? idk).

But this is fine for basically anything an indie developer might need in 99% of cases. Hope this helps, thank you for reading!