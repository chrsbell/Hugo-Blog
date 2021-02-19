---
title: "GameBoy Emulator Log #1"
subtitle: ""
author: "Chris"
description: ""
date: 2021-02-18T18:15:02.751Z
draft: true
tags: ["python", "emulator", "chip-8"]
categories: ["Chip-8"]
DisableComments: false
featuredImage: "featured-image.jpeg"
---

I started working on a new GameBoy emulator project a couple of months ago that I've been meaning to document. Here, I'll talk about the decisions I made along the way, technical hurdles involved in emulating the GameBoy, and upcoming challenges.<!--more--></br>

---

## Sections

### Chip-8 Differentiation

### Fetching Instructions

Before beginning, I knew that GameBoy would be a more difficult system to emulate than Chip-8. By researching the system, I've truly come to appreciate how sophisticated and powerful its hardware actually is.

On paper, the GameBoy has an SM83 microprocessor and executes opcodes using a fetch-and-execute loop. This fetch-and-execute loop is unique from the Chip-8's in that the currently fetched opcode execution is done in parallel with the next opcode fetch instead of serially. This means that it requires at least 2 machine cycles to complete a single instruction, and generally can be treated as such if writing a serial fetch-and-execute loop.

####