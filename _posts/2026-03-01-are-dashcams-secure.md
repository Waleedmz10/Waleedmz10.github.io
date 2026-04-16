---
layout: post
title: "Are Dashcams Secure? A Security Perspective"
date: 2026-03-01
categories: Cybersecurity IoT Automotive
---

# Are Dashcams Secure? A Security Perspective

## Introduction

Dashcams have rapidly evolved from simple in-car recording devices into connected IoT systems. Originally designed to document traffic incidents and provide legal evidence, modern dashcams now include Wi-Fi/Cellular connectivity, GPS tracking, cloud storage, and mobile app integration.

While these features improve usability and convenience, they also introduce cybersecurity risks.

This raises a critical question:

> **Are dashcams secure, or are they just another vulnerable IoT device on wheels?**

---

## Dashcam Market

The global dashcam market has seen significant growth in recent years:

- In some regions, over **30–40% of drivers** use dashcams.
- Insurance companies increasingly incentivize dashcam usage.
- Many modern dashcams now support:
  - Wireless connectivity
  - Mobile application control
  - Remote footage download
  - Over-the-air firmware updates

With connectivity comes expanded attack surface.

Like home routers and IP cameras, dashcams now belong to the broader **automotive IoT ecosystem**.

---

# Case Study: Reverse Engineering Ubox Dashcam

In this case study we will be inspecting the Ubox dashcam from Application and features to it's hardware and figure out how secure is it 

## Ubox dashcam features

![dashcam1](https://github.com/user-attachments/assets/754aa091-e103-4fac-9ed7-e8bf790accab)
 
The **Ubox** is a compact automotive dashcam designed for continuous recording and smartphone integration.

### Features

- Full HD video recording
- Loop recording
- MicroSD card storage
- Built-in microphone
- Wide-angle lens
- Motion detection
- G-sensor (collision detection)
- Night vision support

### Connectivity Features

- Built-in Wi-Fi module
- Mobile app configuration
- Video playback and download via smartphone
- Device settings control through app interface

These features enhance user experience — **but also increase potential security exposure.**

## What's next?

As an embedded security researcher, I prefer to approach from the hardware layer and moving upward to the application layer. Beginning with physical interfaces, exposed debug ports, and chipset identification, before progressing to firmware extraction and analysis, network, and mobile application.

Usually I buy two identical devices, so I can go full destricative on the first one if needed. 

first glance from the outside we see

> **TF card is a SD card that we will be using later.**

![car-camera](https://github.com/user-attachments/assets/90a3d522-976a-41f4-a74a-1e7fa9815038)

---

This is what looks like from inside 
![inside1](https://github.com/user-attachments/assets/4c0c1455-581a-400a-a4c6-0be444de922b)

Interesting 4 golden dots

---
![inside2](https://github.com/user-attachments/assets/f808162b-3454-45c2-a0d6-294b30e5939b)


Seems like a UART let's connect to them using [PCBBite](https://sensepeek.com/pcbite-20-1) and [Tigard](https://www.crowdsupply.com/securinghw/tigard)

![connect](https://github.com/user-attachments/assets/85c55f97-3b80-4b95-b17f-f0db711c1591)


---

## UART Access
 
Rather than soldering, I used **PCBite** spring-loaded probes on the four golden pads. It keeps the board intact and, more importantly, lets me move probes around quickly while I am still figuring out which pad is which.
 
After identifying the pinout (GND, TX, RX) and sweeping common baud rates, **115200** gave clean output. Power-cycling the dashcam streams the full boot log over serial.
 
<img width="1086" height="775" alt="UART_log" src="https://github.com/user-attachments/assets/6fa2472c-7f61-4704-93be-aadf0e46f55b" />

A few things immediately jump out of the boot log:
 
```
Linux version 3.10.14-Zeratul-Archon (huanggexiang@dell-Precision
-Tower-3431) (gcc version 5.4.0 (Ingenic r3.3.7.mxu2-gcc540 ...)
 Jun 19 09:11:56 CST 2025
Ingenic Kernel-3.10 version H20250113a
CPU0 revision is: 00d00100 (Ingenic Xburst)
CCLK:1200MHz L2CLK:600Mhz H0CLK:225MHz H2CLK:225MHz PCLK:112Mhz
```

This tells us a lot in a few lines:
 
- The SoC is an **Ingenic Xburst** — a MIPS-based chip that shows up in a lot of low-cost IP cameras, dashcams, and cheap Android devices. Cheap, capable, and not known for strong security defaults.
- The kernel is **Linux 3.10.14**. Linux 3.10 reached end-of-life in **November 2017** — this device is shipping in 2026 on a kernel that has not received upstream security fixes in nearly a decade.


Boot finishes asks for a user just write "root". And then this:

<img width="933" height="180" alt="root" src="https://github.com/user-attachments/assets/df0b9343-5d48-42d7-a57d-863ffb90720f" />

No password. No authentication of any kind. The UART interface drops straight into a **root shell**

UART is a physical interface, so an attacker needs the device in hand. That is a meaningful limitation, however in our case, UART is just the first door. It is how we discovered the real problem. The root shell lets us poke around the filesystem and reverse engineer some interesting binaries — and that is where things get much worse.

---

## Poking Around the Filesystem
 
With a root shell, the next step is the usual embedded-Linux tour: what is running, what is mounted, and what is on filesystem.

Two things to note up front:
 
- The SD card is mounted at `/tmp/mnt/sdcard` as **vfat** — no encryption, no access control, readable and writable by anyone who pops the card into a laptop.
- One userspace process is doing most of the work. It handles recording, the GPS overlay, the Wi-Fi/app interface, button input, and the OBD data pipeline. Basically, it *is* the dashcam.
If there is a vulnerability on this device, it is overwhelmingly likely to live in that single monolithic binary. So let's pull it off the device and open it in IDA.

---

## Reverse Engineering the Main Binary
 
After copying the binary off over the SDcard then went straight for the low-hanging fruit: to `system()`**. On an embedded Linux binary, `system()` calls are almost always where the interesting (and dangerous) stuff lives — shelling out to external tools, running scripts, launching daemons. It is a short list of places to read, and each one is a potential command-injection or arbitrary-execution sink.

Walking through the `system()` xrefs, one large initialization function immediately stood out:

<img width="1370" height="129" alt="system__" src="https://github.com/user-attachments/assets/0df35208-3080-4c3d-b9e6-ea07957ae300" />

### Every branch hands execution to `system()` as root
 
The dashcam's main process runs as root (we confirmed that earlier with `id`). Every `system()` call here inherits that privilege. So does every child process. So does every shell command in every script that those children execute. Full root, no sandboxing, no privilege separation.

**`/tmp/mnt/sdcard/debug.sh`** — the shell-script autorun. Drop a `.sh` with a specific name on the SD card and it runs as root on every boot. This is the one I used for the PoC. 




