---
title: "Game Boy Emulator Log #2"
subtitle: "Testing Strategy"
author: "Chris"
description: ""
date: 2021-03-20T20:03:30.088Z
draft: false
tags: ["typescript", "emulator", "game boy"]
categories: ["Game Boy"]
DisableComments: false
featuredImage: "featured-image.jpg"
# [blackfriday]
#   extensions = ["hardLineBreak"]
---

Time for an overview of my testing strategy.

<!--more-->

---

[In my last post](/2021/02/gameboy-emulation-1/) I spent time talking about how the Game Boy CPU works using a fetch-and-execute loop. I also gave some examples of what my code looks like. It's already been over a month so I think it's time for a new post talking about the most important and frustrating part of emulation. But before that here is everything I worked on over the past month:
</br></br>

# What I've done

</br>

- Finished writing the other half of the instruction set
- Cleaned up my codebase
- Replaced my class-based primitive type system with one based on [TypeScript flavoring](https://spin.atomicobject.com/2018/01/15/typescript-flexible-nominal-typing/)
- Started working on the LCD interface
- Made a test suite for the CPU
  </br></br>

# Testing

</br>

How do you do that? I need to make sure the emulator behaves like a Game Boy...

</br>

```typescript
it("emulates", () => {
  expect(CPU).toBe(GameBoy);
});
```

</br>
{{< figure src="http://wp.production.patheos.com/blogs/exploringourmatrix/files/2014/10/wpid-Photo-20141013234900.jpg" caption="When you try to write an emulator" >}}

</br></br>

There are a couple of testing strategies that I'm aware of at the moment. One is **unit testing**, or testing how components behave in isolation from other components. I also know of **integration testing**, or testing for expected behavior when components interact with each other out in the wild.
</br></br>

The biggest blocker that I've faced over the past month, is how do you write effective tests for a CPU? I was able to reduce my options to two:
</br></br>

1. Writing tests for each instruction in isolation
2. Integrating some sort of suite of pre-written tests
   </br></br>

Both choices have their pros and cons.
</br></br>

Writing a test for each instruction seems ideal, as I can isolate each instruction and test for expected behavior while decoupling other components such as the Memory module. But then, I would need to write at least 500 tests for the CPU. And that is assuming only one test per instruction.
</br></br>

Using a suite of pre-written tests sounds nice at first, but then you have to think about how to integrate it with your emulator.
</br></br>

After a few minutes of consideration, I decided to try and find a suite of pre-written tests.
</br></br>

## Z80 comparison method

My first idea was to use a suite of z80 tests, like [z80-test](https://github.com/lkesteloot/z80-test). I quickly realized this would take a lot of time. Although the Game Boy CPU resembles a z80, it differs in ways that would make repurposing a z80 test suite annoying. Mostly due to me not wanting to write assembly.
</br></br>

## Emulator state comparison method

I was browsing stack overflow trying to come up with a good way to compare my CPU to some kind of expected state, and came across a post suggesting to compare using another emulator. That sounded like a quick and easy idea! (Spoiler: it wasn't). Searching for emulators to compare my CPU's state to, I settled upon using [the PyBoy emulator by Baekalfen](https://github.com/Baekalfen/PyBoy).
</br></br>

Next, I needed a test suite to run on both emulators. Thankfully, there's a [thorough suite of testing roms](https://github.com/retrio/gb-test-roms) that I used for my emulator.
</br></br>

PyBoy has a great separation of concerns, and it was pretty straightforward to generate expected values. I essentially wrote a Python script that initializes a modified version of PyBoy, and then saves the CPU state to a file each time it executes an instruction. Here is what the script looks like:
</br></br>

```python
import os
import sys
import io
from pyboy.utils import IntIOWrapper
from pyboy import PyBoy

dirname = os.path.dirname(__file__)
out_dir = os.path.join(dirname, "generated")

def runCPUTest(rom_path, rom_name):
    print("Generating expected for " + rom_name)
    rom_file = os.path.join(rom_path, rom_name)
    pyboy = PyBoy(rom_file, bootrom_file=None, disable_renderer=False, sound=False)

    pyboy.set_emulation_speed(0)
    state_file = os.path.join(dirname, "generated", rom_name + ".state")
    if os.path.exists(state_file):
      os.remove(state_file)
    cpu_state = IntIOWrapper(
        open(state_file, "wb")
    )
    while not pyboy.tick(cpu_state):
        pass
    pyboy.stop()
    print("Finished")


def main():
    # Create generated dir
    if not os.path.exists(out_dir):
        os.makedirs(out_dir)
    # Dir pointing to cpu instruction test gameboy roms
    cpu_test_dir = os.path.join(dirname, "gb-test-roms", "cpu_instrs", "individual")
    runCPUTest(cpu_test_dir, "01-special.gb")
    # runCPUTest(cpu_test_dir, "02-interrupts.gb")
    runCPUTest(cpu_test_dir, "03-op sp,hl.gb")
    runCPUTest(cpu_test_dir, "04-op r,imm.gb")
    runCPUTest(cpu_test_dir, "05-op rp.gb")
    runCPUTest(cpu_test_dir, "06-ld r,r.gb")
    # runCPUTest(cpu_test_dir, "07-jr,jp,call,ret,rst.gb")
    runCPUTest(cpu_test_dir, "08-misc instrs.gb")
    runCPUTest(cpu_test_dir, "09-op r,r.gb")
    runCPUTest(cpu_test_dir, "10-bit ops.gb")
    runCPUTest(cpu_test_dir, "11-op a,(hl).gb")


if __name__ == "__main__":
    main()
```

</br></br>

There are a couple of issues with this script...as you can see I've commented out two tests. The reason being that PyBoy itself hangs when I run them. There are some other Game Boy test suites available that I may look into when I get there. If that doesn't work, I may need to try using another emulator.
</br></br>
{{< figure src="https://i.imgur.com/cfs2o9F.png" caption="Modified PyBoy running a test rom" >}}
</br></br>

I also face the question of when to stop running the test ROMs since the ROMs just render a message on the screen to show whether the emulator passed or failed the test. At the moment I just quit out of each instance of PyBoy when I see the message but I'd love to automate that process.
</br></br>

In my emulator, I'm using the Jest testing library to check for expected CPU values. The flow of each test looks like this:
</br></br>

1. Load a test ROM
2. Load the corresponding file that stores expected CPU values
3. Iterate over the file contents, comparing my CPU state at each step to the expected state
   </br></br>

I also have a custom matcher that I use to log the expected state of the emulator and recently executed instructions when a test fails, for example here is what a failed test looks like:
</br></br>
{{< figure src="https://i.imgur.com/GXyrylJ.png" caption="A failed CPU test" >}}
</br></br>
I'd think that this is more of an integration testing strategy since the CPU needs access to the Memory module to pass these tests. This method has still been critical for me to find and fix failed instructions.
</br></br>

# Next steps

The good news is that I was able to fix enough instructions so that the first test now fails at instruction `0xF0`. This is an `LDH` instruction intended to write values to memory addresses offset by `0xff00`.
</br></br>
Inside the Game Boy, this region is used to communicate with external peripherals like the LCD and controller. So I've started looking into how the Game Boy's LCD works and plan to work on that part of the emulator next to make sure the test suite passes.
</br></br>
I'd also love to write tests for each class method and helper functions.
</br></br>
If you're interested in taking a look, the source is available here https://github.com/chrsbell/GameBoy-Emulator.
</br></br>
Don't hesitate to create an issue or a PR/comment if you have thoughts or feedback!
</br></br>
