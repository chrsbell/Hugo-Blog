---
title: "Serial Keyboard Update #1"
subtitle: "Time for a project update"
author: "Chris"
description: ""
date: 2020-12-07T02:09:08.307Z
draft: false
tags: ["serial", "keyboard", "low-latency", "pcb", "design", "linux"]
categories: ["Serial Keyboard"]
featuredImage: "featured-image.jpeg"
math:
  enable: true
---

{{< load-photoswipe >}}

Retroactive performance data and PCB design decisions. <!--more--></br>

[In my previous post](https://chrsbell.github.io/2020/08/designing-a-serial-keyboard/) I discussed the circuitry of my mini serial keyboard, so I thought it would be a good idea to give a brief overview of some of the decisions I made while designing the PCB. I also talk about my initial latency measurements from last year. Note that I am not an electrical engineer by trade and am open to suggestions for improvement.

---

## Contents

</br>

### Size

One of the major flaws of my old mini PS/2 keyboard was its size. Here is what the first edition looked like:

{{< gallery hover-effect="grow" caption-effect="fade">}}
{{< figure src="/images/blog/12-6/board-back.jpg" caption="The back" >}}
{{< figure src="/images/blog/12-6/board-front.jpg" caption="The front" >}}
{{< /gallery >}}

</br>

While it was pretty fun and challenging to design such a small keyboard, using it was difficult. The case I printed for it was not stable and the keyboard would slide around while typing. So I nearly doubled the height and increased the width of my PCB to a size I found to be suitably stable.

</br>

{{< figure src="/images/blog/12-6/size-comparison.jpg" caption="Size comparison between the first and latest mini keyboard iterations, from left to right">}}

</br>

---

### PCB Layout

For the PCB, I decided to use a single ground plane on the opposite side of my power and signal traces with minimal cross-over to ensure each signal had a clear return path. I wanted to better follow the return path of each trace and isolate areas of potential noise and interference, especially on the USB and TTL signals.

</br>

The LEDs make up the bulk of the power usage in this design, so I connected them all to the USB's 5V line with calculated resistors. This way, I could minimize the strain on the 300mA 3.3V regulator. The MAX232, FT230, dsPIC, and Schmitt inverter all draw a minimal amount of current comparatively, but I used a 12 mil width on all of my power traces anyways to distinguish them from my signal traces. Lastly, I placed decoupling capacitors close to each IC and the ESD protection close to the USB port. For the curious, [here is a nice article I referenced in my design](https://www.signalintegrityjournal.com/blogs/12-fundamentals/post/1207-seven-habits-of-successful-2-layer-board-designers).

---

### Retroactive measurements

Although I'm working on a dedicated PCB for my serial keyboard, I actually first tested the idea by altering my old PS/2 mini keyboard. Instead of sending PS/2 scancodes, I encoded information regarding which key was pressed/released into a single byte and wrote it using `putc`. I then prepared a Linux kernel module to process the byte and send corresponding key information to the OS using `input_report_key` and `input_sync`.

</br>

After confirming that my idea worked, I wanted to measure communication time between the keyboard and the kernel module at different serial baud rates. Here is the very rough and shortened firmware and kernel module I used in my measurements:

</br>

</div>

{{< gist chrsbell 5a5f2e51e210bb2d8a8454d5c25a346c >}}

{{< gist chrsbell cd3360761b24843514e7b0d53316edb3 >}}

<div>

</br>

When a key is pressed/released, I send a `1` or `2` to the kernel module and begin recording time in increments of 10 microseconds within the firmware. Once the data is received, the kernel module sends an acknowledgment to the keyboard and the keyboard stops recording. Finally, the keyboard sends the recorded time to the kernel module, and I cut the time in half to approximate the one-way communication speed. Here is what I recorded using various baud rates:

</br>

{{< gallery hover-effect="grow" caption-effect="fade">}}
{{< figure src="/images/blog/12-6/19200.png" caption="Around 1515 microseconds at 19200 baud" >}}
{{< figure src="/images/blog/12-6/38400.png" caption="Around 790 microseconds at 38400 baud" >}}
{{< figure src="/images/blog/12-6/57600.png" caption="Around 520 microseconds at 57600 baud" >}}
{{< figure src="/images/blog/12-6/115200.png" caption="Around 275 microseconds at 115200 baud" >}}
{{< /gallery >}}

</br>

{{< figure src="/images/blog/12-6/scatter-plot.svg" caption="As baud rate doubles, latency is cut by around half" >}}

</br>

Unsurprisingly, latency is inversely proportional to round-trip latency. The maximum baud rate I could achieve using my PC was 115200, though there are serial cards that can go much higher.

---

</br>

### Next steps

Well, now that I have the latest PCB in my hands, you might be thinking to yourself "start working on it again!". Sadly I did try to build it and ran into issues hand soldering the FT230. I also forgot to add larger vias on the exposed pads for the dsPIC and FT230 so I could hand solder them as well as make the pads longer on the ESD protection. Basically, I'm bad at soldering and had to make it more friendly to hand solder. Once the new boards arrive and I find the time I plan to continue rewriting the firmware. Stay tuned!
