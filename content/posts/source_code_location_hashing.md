---
title: "Source Code Location Hashing for immediate-mode APIs"
date: 2025-06-04
description: "An alternative to name strings for ImGUIs and other immediate-mode APIs"
---

If you've ever worked with an immediate-mode GUI system (e.g. Dear ImGui), you've most certainly seen something like this:

```odin
if imgui.Button("Do Action") {
    // action
}
```

The GUI library uses the `"Do Action"` string is both as the visible text for showing the button, but also, it generates
a unique integer ID by hashing it.
This ID is then pushed to a stack of IDs, from which it also calculates an ID for the current scope by XORing it with the previous scope ID.

## What about alternatives?
Using name strings in a GUI system for IDs is really nice, but are there other ways?
I like to use the immediate-mode paradigm for more than just GUI, and in many cases using string doesn't seem ideal.

For example, consider an immediate-mode collider system. Each frame, every entity can call `collider`