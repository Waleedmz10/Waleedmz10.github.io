---
layout: post
title: "Root on Wheels: Hacking a Connected Dashcam"
date: 2026-04-16
categories: Cybersecurity IoT Automotive
---

> ## TL;DR
>
> - A popular connected dashcam runs unsigned code from its SD card, as root, every time the car starts.
> - I demonstrated the attack by getting the dashcam to send its own photos — including embedded GPS coordinates, speed, and timestamp — to a Discord channel I control.
> - Most dashcams ship with an SD card already in the box. Almost no one checks it before use. That is the delivery mechanism.
> - The same device is wired into the car's OBD-II / CAN bus, meaning a compromised dashcam is a foothold inside the vehicle's internal network (if the data lines are connected).
> - **The takeaway:** A connected dashcam is not a camera. It is a Linux computer with your GPS, your microphone, and a cable into your car's nervous system — and this one has no meaningful security boundary.

## Introduction

Dashcams have rapidly evolved from simple in-car recording devices into connected IoT systems. Originally designed to document traffic incidents and provide legal evidence, modern dashcams now include Wi-Fi/Cellular connectivity, GPS tracking, cloud storage, and mobile app integration.

While these features improve usability and convenience, they also introduce cybersecurity risks.

> **"These are no longer just cameras. They are small Linux computers, wired into your car, watching and listening." **

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

In this case study, we inspect the Ubox dashcam from the application layer down to the hardware — and find out how secure it really is.

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

As an embedded security researcher, I prefer to start at the hardware layer and work upward toward the application layer. Beginning with physical interfaces, exposed debug ports, and chipset identification, before progressing to firmware extraction and analysis, network, and mobile application.

Usually I buy two identical devices so I can go fully destructive on the first one if needed.

At first glance from the outside, we see:

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


When boot finishes, it asks for a username. Just type root — no password. And then this:

<img width="933" height="180" alt="root" src="https://github.com/user-attachments/assets/df0b9343-5d48-42d7-a57d-863ffb90720f" />

No password. No authentication of any kind. The UART interface drops straight into a **root shell**.

UART is a physical interface, so an attacker needs the device in hand. That is a meaningful limitation, however in our case, UART is just the first door. It is how we discovered the real problem. The root shell lets us poke around the filesystem and reverse engineer some interesting binaries — and that is where things get much worse.

---

## Poking Around the Filesystem
 
With a root shell, the next step is the usual embedded-Linux tour: what is running, what is mounted, and what is on the filesystem.

Two things to note up front:
 
- The SD card is mounted at `/tmp/mnt/sdcard` as **vfat** — no encryption, no access control, readable and writable by anyone who pops the card into a laptop.
- One userspace process is doing most of the work. It handles recording, the GPS overlay, the Wi-Fi/app interface, button input, and the OBD data pipeline. Basically, it *is* the dashcam.
If there is a vulnerability on this device, it is overwhelmingly likely to live in that single monolithic binary. So let's pull it off the device and open it in IDA.

---

## Reverse Engineering the Main Binary
 
After copying the binary over the SDcard then went straight for the low-hanging fruit: to `system()`**. On an embedded Linux binary, `system()` calls are almost always where the interesting (and dangerous) stuff lives — shelling out to external tools, running scripts, launching daemons. It is a short list of places to read, and each one is a potential command-injection or arbitrary-execution sink.

Walking through the `system()` xrefs, one large initialization function immediately stood out:

<img width="1370" height="129" alt="system__" src="https://github.com/user-attachments/assets/0df35208-3080-4c3d-b9e6-ea07957ae300" />

### Every branch hands execution to `system()` as root
 
The dashcam's main process runs as root (we confirmed that earlier with `id`). Every `system()` call here inherits that privilege. So does every child process. So does every shell command in every script that those children execute. Full root, no sandboxing, no privilege separation.

**`/tmp/mnt/sdcard/debug.sh`** — the shell-script autorun. Drop a `.sh` with a specific name on the SD card and it runs as root on every boot. This is the one I used for the PoC. 

---

## Attack Scenarios
 
Once you have "drop a file on the SD card and it runs as root on boot," the question is no longer *whether* the device can be compromised — it is *what* an attacker chooses to do with it. The SD card is the canvas. The dashcam's own hardware is the paint.

### The PoC: Screenshot Exfiltration
 
To demonstrate impact, I wrote a short shell script, saved it to the SD card under the magic filename, and put the card back in the dashcam. On the next boot, the main process executed the script as root with zero user interaction.
 
The payload is deliberately simple: wait for the device to finish initializing, grab the most recent snapshot it has captured, and send it to a Discord channel I control via a webhook.

<img width="1296" height="783" alt="POC" src="https://github.com/user-attachments/assets/293a36e1-0182-42d3-8ee2-da7dbdc521ff" />

Within seconds of the dashcam booting, the latest photo lands in my Discord.
 
And this is where the dashcam itself does most of the attacker's work. Look at the top of the exfiltrated image — the device stamps every frame with its own overlay:
 
- **Exact date and time** (`2026-04-16 21:18:46`)
- **GPS coordinates** (latitude and longitude, burned directly into the pixels)
- **Vehicle speed** (`0 MPH` in the stationary test, but it updates live)
  
No EXIF parsing, no metadata extraction, nothing fancy. The dashcam helpfully renders the victim's exact location onto every image it saves, and then saves those images to an SD card it runs unsigned code from. A single frame is enough to know **where the car is, when it was there, and how fast it was going** — all without the owner ever noticing the SD card was touched.

### How a Real Victim Would Actually Get Hit
 
The natural reaction here is: *"fine, but how does the attacker's SD card end up in someone else's dashcam?"*
 
The answer is simpler than people expect. **Most dashcams ship with the SD card already included in the box.** The buyer opens the package, slides the card into the slot, mounts the dashcam on the windshield, and drives off. Almost nobody formats a brand-new SD card that came with the device. Almost nobody plugs it into a laptop to inspect what is on it. Why would they? It came in the box.
 
That is the entire attack. A malicious actor anywhere in the supply chain — a reseller, a repackager, a compromised warehouse, a "refurbished" listing on a marketplace — drops the payload file onto the bundled SD card before the box is sealed. The victim does the rest for free.
 
And because the payload survives on the SD card across every boot, the dashcam is compromised from the moment the car is first driven until the day the user thinks to reformat the card. Which, for most users, is **never**.
 
---

### Why This PoC Is the *Small* Version
 
The screenshot script is the proof, not the ceiling. Root code execution on the device that owns the car's camera, microphone, GPS, Wi-Fi radio, and OBD-II connection opens up a much bigger set of possibilities. I did not build out every one of these — but the primitive is the same for all of them.
 
#### Continuous video exfiltration
 
The dashcam is already recording loop video to the SD card. A slightly longer script could tar up the most recent clip, or tail the active recording, and push it to the attacker. Instead of a single snapshot, the attacker gets **a live feed of everywhere the car goes** — with the GPS/timestamp overlay baked into every frame, exactly like the PoC image above.
 
#### Live audio and the in-car microphone
 
The device has a built-in microphone, and it is already wired into the recording pipeline for video audio. An attacker running as root can re-open that audio device, dump raw samples, and stream them out — turning the dashcam into a **covert in-cabin listening device**. Conversations in the car, phone calls on speaker, anything audible inside the cabin.
 
#### GPS tracking without needing a photo
 
The GPS module is continuously feeding the main process for the on-video overlay. A payload does not have to wait for a snapshot — it can read GPS coordinates directly and report them on an interval, turning the dashcam into a **real-time tracker** for whoever owns the webhook.

#### The CAN bus — the scariest possibility
 
This is where this research stops being a privacy story and starts being a safety story.
 
This dashcam is powered from the vehicle's **OBD-II port**. OBD-II is not just a power connector — it is a physical tap into the **CAN bus**, the network that carries messages between the engine control unit, transmission, brakes, airbags, and, on modern cars, driver-assistance systems. The dashcam legitimately uses this connection to read vehicle data (speed, RPM, etc.) for its overlays.
 
That means the device has, by design, a hardware path to the internal vehicle network. And we just showed that the device has **no meaningful boundary** between "what the vendor runs" and "what a file on the SD card runs."
 
I have **not tested CAN injection on this device.** I did not trace the OBD wiring through the board, I did not reverse the CAN transceiver path, and I have not sent a single frame onto a live vehicle bus. That is a separate research project with its own ethical and legal weight, and it belongs on a test vehicle in a controlled environment — not on a public road in someone else's car.

The industry spent the last decade — from the 2015 Jeep Cherokee remote hack onwards — learning that **anything wired into the CAN bus is part of the car's trust boundary**. A sub-$100 IoT device with an unauthenticated autorun from removable storage should not be sitting inside that boundary. And yet here we are.
 
> **A dashcam that can be compromised with a file on an SD card, and that is wired into the car's internal network, is not a recording device. It is a foothold.**

---

## What This Means

For drivers: treat the SD card that came in the box as untrusted. Format it before first use. If the dashcam offers an OBD-II cable, consider whether you actually need it — the cigarette-lighter adapter trades some features for isolation from the CAN bus.

For vendors: do not execute unsigned code from removable storage. Do not ship 2026 products on a Linux kernel that has been end-of-life since 2017. Publish a security contact.

For the industry: a sub-$100 IoT device with no authentication should not be sitting inside the trust boundary of a moving vehicle. We spent the 2010s learning this lesson with infotainment systems. Dashcams are the next chapter.
 
---



