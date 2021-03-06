---
title: "Designing a Serial Keyboard Circuit"
subtitle: "How I designed the circuit for a mini keyboard"
author: "Chris"
description: ""
date: 2020-08-23T20:02:36-07:00
draft: false
tags: ["serial", "keyboard", "low-latency", "pcb", "design"]
categories: ["Serial Keyboard"]
featuredImage: "featured-image.jpeg"
math:
  enable: true
---

{{< load-photoswipe >}}

Excited to try and get back into blogging!

This will be the first post in a _hopefully_ not super long series documenting my progress in building a very special mini keyboard.

<!--more-->

---

## About

{{< admonition note "What is this?" >}}

A three-key keyboard, just like the one you use to type. The primary goal of this keyboard is to achieve the lowest input latency I possibly can using standard Cherry MX switches.

{{< /admonition >}}

{{< admonition note "Why only three keys?" >}}

I personally only need 2-3 keys to make good use of this keyboard in the rhythm game I play, osu! It should be possible to scale the design up to a full-size keyboard, but this would take longer and cost much more.

{{< /admonition >}}

This serial keyboard project was based on some work I did last year on a [very similar project](https://github.com/chrsbell/PS-2-Mini). The major difference is that this keyboard communicates with the PC, or host, using RS232. My previous project used PS/2; if you are interested, a great resource for the PS/2 protocol is available [here](https://www.avrfreaks.net/sites/default/files/PS2%20Keyboard.pdf).</br></br>

---

## Circuitry

My first step in designing the circuit for my keyboard was to pick a CAD program. There are many free and commercial programs available for this purpose, but the ones I'm most familiar with are KiCAD and EAGLE. KiCAD has great support for hierarchical sheets and it's free so I went with that. At first, I designed the circuit without using hierarchical sheets but later added them since I had extra time. Here are some before and after pics of the full circuit!</br></br>

{{< gallery hover-effect="grow" caption-effect="fade">}}
{{< figure src="/images/blog/8-20/sch-before.jpg" caption="Designing the circuit without hierarchical sheets " >}}
{{< figure src="/images/blog/8-20/sch-after.jpg" caption="The completed circuit, using hierarchical sheets for sub circuits" >}}
{{< /gallery >}}</br></br>

There were a few new features I wanted to include in my serial mini keyboard which required me to make good use of Google. The biggest feature I wanted to add was bootloader support.

{{< admonition note "What's a bootloader?" >}}

Basically, a bootloader is an on-chip program that handles loading a target application. This allows convenient firmware updates over USB, as the bootloader can communicate with a PC host application to change the on-keyboard firmware.

{{< /admonition >}}

The microprocessor I chose for my keyboard, a [16-bit dsPIC](http://ww1.microchip.com/downloads/en/DeviceDoc/dsPIC33CK64MP105-Family-Data-Sheetg-DS70005363D.pdf), was a mostly arbitrary pick...I had many leftovers from my previous projects. The main thing to note is that this microprocessor includes 3 on-chip UARTs. These are devices used for serial communication, the whole purpose of the serial keyboard. I decided to use one on-chip UART device to communicate with the PC for firmware updates, and another UART to send keystrokes.</br></br>

---

### USB-to-UART

To receive and send USB data for keyboard firmware updates, I used the [FT230XQ](https://www.ftdichip.com/Support/Documents/DataSheets/ICs/DS_FT230X.pdf) USB-to-UART IC. The finished circuit for USB communication was pretty straightforward, I just used the suggested circuit from the datasheet with an added LED status indicator and additional ESD protection.

{{< figure src="/images/blog/8-20/usb-uart.jpg" caption="The completed USB-UART circuit" >}}

---

### RGB Backlighting

The next new feature I attempted to implement was adjustable RGB backlighting. The first step in designing this circuit was to refresh my understanding of transistors. It took a bit of time, but I ended up using [IRLML2060](https://www.infineon.com/dgdl/irlml2060pbf.pdf?fileId=5546d462533600a401535664b7fb25ee) N-MOSFETs for each color. This transistor is operable with TTL logic levels and has a short enough time delay to drive the LEDs with PWM. I added [two potentiometers](https://www.mouser.com/datasheet/2/54/3352-776447.pdf) as well; one to control the hue of the LEDs, and one to control the brightness. I had a tough time deciding on LEDs but ended up using [these fairly bright ones](https://www.cree.com/led-components/media/documents/ds-CLVBA-FKA.pdf).

{{< figure src="/images/blog/8-20/backlight.jpg" caption="The completed backlighting circuit" >}}

---

### RS232 Converter

While the on-chip UART devices allow for easy serial communication between TTL logic levels (typically around 3.3V), serial communication with a PC's serial port requires voltage levels that swing from ~$\pm$12V. The famous [MAX232 IC](http://www.ti.com/lit/ds/symlink/max232.pdf) takes care of this conversion, and I once again used the preexisting example circuit with no modifications needed.

{{< figure src="/images/blog/8-20/rs232-converter.jpg" caption="The RS232 conversion circuit">}}

---

### Debouncing

This circuit is responsible for _attempting_ to reduce the largest source of keyboard latency for Cherry MX switches, switch bouncing.
On debouncing:

> "When the contacts of any mechanical switch bang together they rebound a bit before
> settling, causing bounce. **Debouncing**, of course, is the process of removing the bounces,
> of converting the brutish realities of the analog world into pristine ones and zeros. Both
> hardware and software solutions exist, though by far the most common are those done in
> a snippet of code."
>
> > [Jack Ganssle](https://my.eng.utah.edu/~cs5780/debouncing.pdf)

My goal with this circuit was to use hysteresis to reduce/eliminate logic changes due to switch bouncing. My previous project attempted to use hysteresis but was flawed because of incorrect RC values and an incorrect hysteresis circuit. The updated circuit should behave as expected although I have yet to test it, and it was mostly taken from [this simple hysteresis circuit](https://hackaday.com/2015/12/09/embed-with-elliot-debounce-your-noisy-buttons-part-i/). I adjusted the debounce time here to 0.1 ms versus the recommended 5 ms time for Cherry switches...we'll see how that goes.

{{< figure src="/images/blog/8-20/debounce.jpg" caption="The circuit responsible for the bulk of debouncing">}}

---

### Power and Oscillator

These circuits are pretty unexciting and take care of the basics of providing 3.3V to the ICs and 100MHz to the microcontroller. After playing around with MPLAB Code Configurator, however, I think I might have misinterpreted how the microcontroller should be run at 100MHz. My initial assumption was that the microcontroller would play fine with a 100MHz external clock, but as shown in the picture below the max frequency for an external clock is 64MHz. The apparent solution to this would be to use a clock source with a PLL; an example is shown in the second figure using the internal FRC clock. So I will probably need to use the internal FRC clock for this one.

{{< gallery hover-effect="grow" caption-effect="fade">}}
{{< figure src="/images/blog/8-20/clock/external-clock.png" caption="The maximum external clock frequency..." >}}
{{< figure src="/images/blog/8-20/clock/pll.png"caption="FRC with PLL can provide the expected 100MHz" >}}
{{< /gallery >}}

---

### Complete PCB

So here is the completed PCB! I skipped a few steps to get to this point, but hopefully, in another post, I can talk about the obstacles and decisions I made while designing the PCB.

{{< gallery hover-effect="grow" caption-effect="fade">}}
{{< figure src="/images/blog/8-20/pcb-3d/pcb-3d-top.jpeg" caption="Top of PCB" >}}
{{< figure src="/images/blog/8-20/pcb-3d/pcb-3d-bottom.jpeg"caption="Bottom of PCB" >}}
{{< /gallery >}}
