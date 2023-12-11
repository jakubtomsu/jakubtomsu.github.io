---
title: "Reverse engineering Unity games"
date: 2022-03-24
description: "few tips on modding games made with Unity Engine..."
---

Few months ago, I made a level editor mod for [Shady Knight](https://store.steampowered.com/app/1155650/Shady_Knight/),
an action game with lots of movement, parkour and combat. I thought I could share a few things related to reverse engineering the game and modding.


## accessing source code
For a game, all of the gameplay source code is stored in `\SomeGame\SomeGame_Data\Managed\Assembly-CSharp.dll`. This is a **dll** (dynamically-linked library), which is generated from source files when the game is built.
To get the original source code, you need to decompile this dll. Fortunately, C# is quite easy to decompile, and there are many free and open-source tools on the internet.  
I personally really like [dnSpy](https://github.com/dnSpy/dnSpy/releases/tag/v6.1.8),
since it's also capable of compiling C# code (this means you can just change code from the original game)


## injecting custom code
It is of course possible to develop an entire mod by editing the `Assembly-CSharp.dll`, but it's easier to just create it separately and just load it at startup time.
This way your game can support multiple mods at the same time, because games can have one common loader for all mods.

If you want to inject the code yourself, here is a simple example how to load and run a C# dll:
```csharp
using System.Reflection; // loading dll's at runtime

// just loop all types and find the entry point method
// there are probably much better ways of doing this, but this worked for me
foreach(Type type in Assembly.LoadFile("some dll path").GetTypes()) {
	MethodInfo method = type.GetMethod("some method with unique name");
	if (!(method == null)) {
		object obj = Activator.CreateInstance(type);
		method.Invoke(obj, new object[0]);
	}
}
```
This code sould ideally be placed in some method that runs *as early as possible*.
Probably `Awake` function on a component somewhere on the first loading screen, main menu, etc. is good enough.

## inspecting the renderer
With tools like [NVIDIA Nsight](https://developer.nvidia.com/nsight-graphics), [RenderDoc](https://renderdoc.org/), etc.
you can easily inspect what your GPU is doing, and how the game actually renders each frame.
You can also take a look at all kinds of resources in the game - shaders, textures, models, ...

Here is a screenshot from Shady Knight when using RenderDoc:
![shady knight screenshot](/shady_knight_renderdoc.png)
as you can see, it shows everything you might need