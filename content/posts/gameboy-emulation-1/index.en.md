---
title: "Game Boy Emulator Log #1"
subtitle: "The Game Boy CPU"
author: "Chris"
description: ""
date: 2021-02-18T18:15:02.751Z
draft: false
tags: ["typescript", "emulator", "game boy"]
categories: ["Game Boy"]
DisableComments: false
featuredImage: "featured-image.jpg"
---

Here, I'll give a brief overview of the Game Boy's processor and how I'm implementing it.<!--more-->

---

### CPU Overview

Before beginning, I knew that the Game Boy would be a more difficult system to emulate than Chip-8. While researching the system, I've come to appreciate how surprisingly sophisticated its hardware is. This video does a great job of explaining it in detail: </br></br>

{{< youtube RZUDEaLa5Nw >}}</br></br>

The Game Boy system uses an SM83 microprocessor and decodes/executes opcodes using a fetch-and-execute loop. This fetch-and-execute loop differs from the Chip-8's in that the next opcode is fetched in *parallel* with the execution of the current opcode instead of serially.</br></br>

This means that it requires at least 2 machine cycles to complete a single instruction if writing a serial fetch-and-execute loop, like the one I'm writing.</br></br>

The SM83's instruction set is very similar to a Z80's, with a few specialized instructions added and removed. In total, the SM83 contains 256 standard instructions and another 256 special CB-prefixed instructions compared to the Chip-8's 36 instructions.</br></br>

For my emulator, I've mostly referenced these documentation sources for the instruction set:</br></br>

- [Gbdev's Opcode Table](https://gbdev.io/gb-opcodes/optables/)
- [Gekkio's Game Boy Technical Reference](https://gekkio.fi/files/gb-docs/gbctr.pdf)
- [Z80 Instruction Documentation](https://gist.github.com/sifton/4471555)
- [Imran Nazar's Game Boy](http://imrannazar.com/GameBoy-Emulation-in-JavaScript:-The-CPU)
</br></br>

The half-carry flag and DAA instructions were difficult for me to learn, so I used these resources as well:
</br></br>

- [Visual Guide to the Gameboy's Half-Carry Flag](https://robdor.com/2016/08/10/gameboy-emulator-half-carry-flag/)
- [Explanation of Half-Carry Flag](https://stackoverflow.com/questions/57958631/game-boy-half-carry-flag-and-16-bit-instructions-especially-opcode-0xe8)
- [Explanation of Decimal Adjust](https://forums.nesdev.com/viewtopic.php?t=15944)
</br></br>

---

</br>

#### RLA/RRA instructions

The bit rotation instructions were pretty fun to learn about and there were some subtleties when I implemented them.</br></br>

The goals of the RLA and RRA instructions are to rotate the affected register (A) left or right through the carry bit.</br></br>

{{< admonition note "About register A" >}}

Register A serves as the "accumulator" register and is used to store the results of and perform mathematical operations. I thought this was interesting as in JavaScript, the reduce method of an array takes an argument by the same name and uses it for the same purpose.

{{< /admonition >}}</br>

At first, I thought left/right shifting using `<<` or `>>` would be enough, but the goal here is to incorporate the carry bit as a 9th bit.</br></br>

In the case of RLA, for example:
 1. A copy of the carry bit is stored.
 2. Bit 7 becomes the new carry bit.
 3. All bits are left-shifted using the `<<` operator.
 4. Bit 0 is set to the old carry bit.</br></br>

Here is some example code from my emulator, and an example from the [Gambatte emulator](https://github.com/sinamas/gambatte).</br></br>

##### My RLA
<iframe frameborder="0" width="100%" height="500px" src="https://repl.it/@chrsbell/RLA?lite=true"></iframe></br></br>

##### Gambatte's RLA
<iframe frameborder="0" width="100%" height="500px" src="https://repl.it/@chrsbell/Gambatte-RLA?lite=true"></iframe></br></br>

---

</br>

### Emulating the CPU

I made the decision early on to use TypeScript since there are a lot of data types required by the emulator. For example, I have types and interfaces for a *Byte*, *Word*, *ByteArray*, *Register*, *Flag*, and more.</br></br>

This was helpful when writing instructions as I found myself regularly double-checking the data type of every address/register and read/write value from memory.</br></br>

Once I verify the functionality of my CPU, I'd like to turn some of the primitive types into simple functions. My primitive types rely on `new` instantiation which increases the memory usage of my emulator, I think.</br></br>

For example:
```typescript
const UnsignedByte = (reg) => reg & 255;
```
</br></br>
Instead of:
```typescript
export class Byte extends Uint8Array implements Primitive
```
</br></br>
Compared to my Chip-8 emulator, I did not need to write a hash function to decode each opcode and only needed to map each opcode's hex value to an instruction.</br></br>

To help accomplish this, I made an [intermediate function generator](https://github.com/chrsbell/gameboy-instruction-generator) that automates the process while also allowing me to make changes to the documentation in the future.</br></br>

The function generator uses a [JSON of the instruction set](https://gbdev.io/gb-opcodes/Opcodes.json) to create functions that return the number of machine cycles each opcode requires and increments the program counter accordingly.</br></br>

Keeping track of the number of m-cycles each opcode uses is important to properly emulate rendering and other timing functions...I'll save that explanation for when I get there.</br></br>

Here's an example of the auto-generated opcode `RLA`:

```typescript
  /**
   * Rotate A left through Carry flag.
   * @param - CPU class.
   * @returns - Number of system clock ticks used.
   * Affected flags: Z, N, H, C
   */
   function RLA (this: CPU): number {
     OpcodeMap['0x17'].call(this);
     this.PC.add(1);
     return 4;
   };
```
</br></br>
And here is the opcode itself:

```typescript
  '0x17': function (this: CPU): void {
    // need to rotate left through the carry flag
    // get the old carry value
    const oldCY = this.R.F.CY.value();
    // set the carry flag to the 7th bit of A
    this.R.F.CY.set(this.R.AF.upper().value() >> 7);
    // rotate left
    let shifted = this.R.AF.upper().value() << 1;
    // combine old flag and shifted, set to A
    this.R.AF.setUpper(new Byte(shifted | oldCY));
  },
```
</br></br>

Finally, here is what my fetch-execute cycle looks like at the moment:

```typescript
  public executeInstruction(): number {
    if (Memory.inBios) {
      // fetch
      const opcode: number = Memory.readByte(this.PC.value());
      // not doing any execution of bios instructions for now
      this.PC.add(1);
      // check if finished bios execution
      if (!Memory.inBios) {
        this.initPowerSequence();
      }
      return 2;
    } else {
      // normal execution
      // fetch
      const opcode: number = Memory.readByte(this.PC.value());
      // execute
      const numCycles: number = this.opcodes[opcode].call(this);
      return numCycles;
    }
  }
```
</br></br>

---

</br>

### Next steps

Right now, I've written the core 256 instructions and am working on testing. Writing tests for the emulator will be challenging and I'll talk about it in another post.</br></br>

I also have a Memory class that supports reading/writing bytes and words using an MBC0 cartridge. I may talk about this later after I've tested it.</br></br>

[The full source is available here, I'm working on the opcodes and test-suite branches right now.](https://github.com/chrsbell/GameBoy-Emulator)
