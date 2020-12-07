---
title: "Designing a Chip-8 Emulator in Python"
subtitle: "How I wrote a simple emulator"
author: "Chris"
description: ""
date: 2020-11-10T04:47:09.883Z
draft: false
tags: ["python", "emulator", "chip-8"]
categories: ["Chip-8"]
DisableComments: false
featuredImage: "featured-image.jpeg"
---

{{< load-photoswipe >}}

Emulating in Python!<!--more--></br>

Note: I am by no means an expert or fluent with Python, and prior to this project I hadn't used it since taking a computer science class many years ago.
</br></br>
Before starting my immersive at Hack Reactor, I decided to try and build an emulator! <!--more-->I got the idea for the project from [this large collection of project ideas](https://github.com/danistefanovic/build-your-own-x), and while GameBoy was a tempting prospect to emulate I quickly realized how much time it would take to complete one. So I went with the much simpler Chip-8.
</br>

---

## Project Phases

</br>

### Rendering Graphics

This was my first time really diving into how emulation worked and doing it in a relatively new language for me was also a challenge. I spent a few days refreshing my knowledge of Python by watching tutorial videos and once I felt that I had the essential fundamentals down, I started by writing a rendering class.
</br></br>
I'd done a lot of OpenGL based rendering in high school so I decided to go with what I was familiar with and write a simple class which could toggle virtual 'pixels' on the screen. The Chip-8 language uses a 64x32 monochrome display, so I mapped each corresponding section of the emulator window to a pixel.
</br></br>
Since I was using Modern OpenGL (i.e. shaders and vertex buffer objects), I was able to access and toggle each pixel as needed by obtaining an offset into the virtual screen's color VBO and then writing either '0' or '1' with `glBufferSubData` to represent that pixel's state.
I decided to keep it simple by representing each virtual pixel as a set of 4 vertices and rendering the data as `GL_QUADS`, but further optimization using `GL_TRIANGLES` or `GL_TRIANGLE_STRIP` could significantly reduce the vertex count.
</br></br>

</div>

{{< admonition note "The set pixel function" >}}

{{< gist chrsbell a8f448fef0e71da4f67d4ffb8b419cff >}}

{{< /admonition >}}

### Interpreting Instructions

After having a solid rendering class, I jumped straight into writing the Chip-8 interpreter.
<br/></br>
Taken straight from [Cowgod's Chip-8 technical reference](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM), the structure of the Chip-8's RAM should look like this:

{{< admonition note "The RAM structure of the Chip-8 interpreter" >}}

```
+---------------+= 0xFFF (4095) End of Chip-8 RAM
|               |
|               |
|               |
|               |
|               |
| 0x200 to 0xFFF|
|     Chip-8    |
| Program / Data|
|     Space     |
|               |
|               |
|               |
+- - - - - - - -+= 0x600 (1536) Start of ETI 660 Chip-8 programs
|               |
|               |
|               |
+---------------+= 0x200 (512) Start of most Chip-8 programs
| 0x000 to 0x1FF|
| Reserved for  |
|  interpreter  |
+---------------+= 0x000 (0) Start of Chip-8 RAM
```

{{< /admonition >}}

</br></br>
Each Chip-8 instruction is 2-bytes long, however, so to successfully load a ROM I had to cut each opcode in half and insert it into the emulator's memory starting at 0x200 that way.

{{< admonition note "The opcode loading function" >}}

{{< gist chrsbell b8e754e29d22361933bdd55570543c07 >}}

{{< /admonition >}}

Next, I had to figure out a way to map each of the 36 opcodes to a corresponding function. This turned out to be challenging in that different instructions contain variables in their names to represent targeted bits of the instruction.
</br></br>
For example, one instruction might be **8FA0** and need to be mapped to **8xy0**, where x represents the lowest 4 bits of the upper half of the opcode and y represents the upper 4 bits of the lower half of the opcode. Creating a hashing function that worked for all 36 opcodes took a good while to figure out, but once I had it working I was able to simply write out all of the necessary opcode functions.
</br></br>
{{< figure src="/images/blog/11-8/hash.png" caption="Mapping each opcode to a function for easy instruction lookup" >}}
</br></br>
The most difficult instruction to write was definitely **Dxyn** or the draw instruction. Chip-8 represents sprites, or pictures, as bytes of data such that an individual bit corresponds to a pixel on the virtual screen. Rendering these sprites involves performing bitwise logic on the screen itself as well as checking for sprite collisions and setting the appropriate bit.
</br></br>
The remainder instructions were mostly math and performing bitwise logic, so they were fairly straightforward. The instructions handling subroutines were a bit confusing for me at first, but I'd say they really solidified my understanding of how a simple call stack works.

### Timing

The first iteration of my emulator was...super slow. Even when I increased my FPS limiter to 240, I found every game to be unplayable. I spent more time than I'd like to admit troubleshooting this problem, but it turned out the way I coupled interpreter logic with rendering was not viable at all. I solved the issue by adding a `num_cycles` variable to my interpreter to adjust the number of instructions executed before re-rendering the window.
</br></br>
{{< admonition note "The fix to my playability issues" >}}

{{< gist chrsbell 2ac5cb699307fceb1e6e802a8a39fbbf >}}

{{< /admonition >}}

</br></br>

Another challenge I faced was generating audio of a variable length for the sound timer opcode. While I could have used an audio file and stopped/started playing it as necessary, I decided to use the [https://pypi.org/project/simpleaudio/](simpleaudio) library with NumPy to generate and play square waves of variable length.

{{< admonition note "This is the whole class" >}}

{{< gist chrsbell e50ea6179d49e9b44ddc9f8ee261eb62 >}}

<div>
{{< /admonition >}}

### Closing Comments

There are some other features I touched on while working on the emulator that I didn't fully implement, such as remappable keyboard input and saving/loading ROM states. All in all, this was an incredibly fun mini project and I would love to tackle Game Boy emulation sometime, preferably in JavaScript!
</br></br>
Finally, here is a quick demo video:
</br></br>

{{< youtube iDq2hOtcp-c >}}
</br></br>
[The full source is available here!](https://github.com/chrsbell/Chip8Emulator)
