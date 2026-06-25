---
layout: post
title: "One Connected Camera, 15,000 Eyes: How One Hardware Bug Spreads Across the Internet"
date: 2026-06-23
tags: [hardware, iot, connected-camera, uart, uboot, firmware, rtsp, reverse-engineering]
---


<img width="800" height="600" alt="image" src="https://github.com/user-attachments/assets/a7f1213c-853c-4c53-9f9b-45346c131b0b" />



> ## TL;DR
>
> - I tested an internet-connected camera. From the its app and network, I found no vulnerabilities.
> - So I opened it up and worked on the hardware parts. I pulled the memory chip off the board and copied everything on it.
> - Inside that copy I found the bootloader (the first code that runs) was locked with a password — but the password was written right into the code, the same on every device.
> - That password let me unlock the device and, with a small trick on a writable part of the memory, change the admin password and get full control (root).
> - With full control I read the main program and found a hidden video link that streams live with no password at all.
> - A small spelling mistake in the device's web server made it easy to spot online. A passive internet scan found the same software on 15,000+ devices.
> - **One bug on my bench turns into 15,000 exposed devices. That scale is the whole point of this post.**

## Why I keep coming back to these devices

After [Root on Wheels](https://waleedmz10.github.io/cybersecurity/iot/automotive/2026/04/16/Root-on-Wheels-Hacking-a-Connected-Dashcam.html)  I wanted to use the same approach on something that sits on millions of walls instead of car dashcams: the connected camera. If one hardware weakness in a single unit also exists in every other unit built on the same software, then the problem isn't one camera — it's every device like it.

That jump from "one device" to "the whole product line" is the main idea of this post. The hardware work is how I got there; the scale is the point.

(I'm keeping the brand and model private until the fix)

## Step 0 — The boring part: testing from the app & network (nothing found)

I always start where a normal remote attacker would: the web page it serves, the phone app, and the network traffic.

- I looked at the web admin page, the video endpoints, and the app's traffic.
- I tested the login for the usual easy wins — default passwords, command injection, hidden file access, login bypass, weak sessions.
- I checked how the device pairs with the cloud for the classic "guessable ID, no real binding" mistake.

**Result: nothing.** No easy remote shell, no default password, no obvious login bypass from outside. On the surface, a fairly solid device.

That clean result is exactly why the hardware matters. "We couldn't break in over the network" is not the same as "it's safe." It usually just means the interesting bug is one layer down — hidden behind the assumption that nobody will ever open the case. So I opened the case.

## Step 1 — Opening it up: looking at the board

Off came the screws. On the board, the parts that matter:

<img width="781" height="1200" alt="Open_Camera" src="https://github.com/user-attachments/assets/4dc2ba28-09a7-4d49-bfc7-9dee6e731b66" />


From the main PCB (Top view)

<img width="1000" height="918" alt="MAin_PCB_TOP" src="https://github.com/user-attachments/assets/df9b7601-4f91-4a9f-8e34-34dfa34c6d80" />

- **The main chip (SoC) [Red rectangle]** — the brain. It runs Linux and handles the video, the network, and the streaming.
- **A UART pins [Yellow rectangle]** — a small set of pins that lets you watch the device "talk" as it starts up, like a debug window.

From the main PCB (bottom view)

<img width="1000" height="1400" alt="Main_PCB_Bottom" src="https://github.com/user-attachments/assets/9a35e5cd-4aae-457f-99d2-28a464769073" />


- **The memory chip (flash) [Red rectangle]** — holds the startup code, the system, and the files.


I connected to the UART and watched the startup messages. Two things were clear, and both were annoying:

<img width="960" height="1280" alt="UART_connection" src="https://github.com/user-attachments/assets/e1cad180-2af6-4a15-ad14-2ba292767930" />

- The bootloader (U-Boot) was locked with a password when interrupting it.

  <img width="615" height="936" alt="uboot_password" src="https://github.com/user-attachments/assets/225b1b10-fb81-4052-ac55-ddf3154b32d0" />

- The normal login was also locked with a password I didn't have.

  <img width="1002" height="1012" alt="root_shell" src="https://github.com/user-attachments/assets/25f02719-c330-4aee-a3c6-be1883983137" />

So the easy path was closed. The device was basically telling me to go get the firmware!

## Step 2 — Pulling the firmware from flash chip 

If the device won't talk, take its firmware. I carefully unsoldered the flash chip, put it on a reader, and copied everything off it into one file.

<img width="831" height="773" alt="Desoldring" src="https://github.com/user-attachments/assets/993b7dcc-feaf-4238-b51e-b29e26bba0d5" />

<img width="1508" height="836" alt="reading_flash" src="https://github.com/user-attachments/assets/4746e41d-3bb0-4127-a0d9-df52efc36a53" />

A clean, complete copy matters a lot here — a bad copy will look like random errors later and waste hours. So I read it twice and checked that both copies matched before moving on.

## Step 3 — Rebuilding the files (binwalk + the startup log)

With the raw copy in hand, I used a tool called binwalk to split it into its parts. I lined those parts up against the startup log from the UART (which helpfully prints the names and locations of each part) so I wasn't guessing:

From the UART while booting:

<img width="657" height="412" alt="MTD_sectors" src="https://github.com/user-attachments/assets/93e87b77-4683-4900-a1af-4ff9481a7dac" />

From the binwalk after reading the firmware:

<img width="1422" height="661" alt="Binwalk_Desoldered" src="https://github.com/user-attachments/assets/64e28790-96f6-41d1-87a5-7f44d4b633b7" />

- the bootloader (U-Boot),
- the system core (kernel),
- the main read-only files (a SquashFS image),
- and a small writable area for settings (important later, in Step 6).

The startup log is underrated — the device basically hands you a map of its own memory every time you power it on. Pairing that map with binwalk removed almost all the guesswork.


## Step 4 — Finding the bootloader password

Now the system files and the bootloader were sitting on my computer, fully readable. I opened the bootloader in a disassembler and looked for the part that checks that startup password.

It was about as bad as it gets: the password was written straight into the code — the same on every single device of this type. No per-device value, nothing unique. One password, everywhere.

<img width="1378" height="857" alt="RE_password" src="https://github.com/user-attachments/assets/87762c61-22ae-4e45-a0cc-1426b505b308" />

## Step 5 — Getting in, but only halfway

I reconnected the UART, typed the password I'd found, and the bootloader let me in.

From there I changed the startup settings to drop me straight into a basic shell when the device powered on. That worked — but it was a weak shell. Skipping the normal startup means most of the device isn't really running: no network, no video services, lots of missing pieces. Enough to look around the files, but not enough to actually be the working device.

I needed the real thing: a proper login on a fully running device.

<img width="1102" height="111" alt="booting_to_shell" src="https://github.com/user-attachments/assets/d5c2aa30-a004-414f-b529-4e6db0edf101" />


## Step 6 — The trick: replacing the password file

Here's the neat part. The flash memory had a small writable settings area that held both the device's settings and a startup script that runs every time it boots.

The main system files are read-only, so I couldn't edit the real password file (`/etc/shadow`) directly. But I didn't need to. From my weak shell, I dropped my own password file into that writable area and used the existing startup script to copy my file over the real one (mount it on top) — early, before the login screen even loads.

So I set the admin password to one I picked, saved it to the writable area, and rebooted normally. Now the device boots all the way — everything running, video working, network up — and the admin login accepts my password.

The shadow file shown in the screenshot below is the file that I created containing the modified password (created with the same structure as the Linux shadow file).

The run file is the file that runs at boot and can be modified (modified to mount the shadow file on top of the original shadow file of the system).

<img width="616" height="327" alt="config_files" src="https://github.com/user-attachments/assets/2091e498-7ccd-411f-b54e-4c17a71ba149" />

Also you can see in the screenshot above configuration files that contain a lot of useful information such as **(WiFi credentials, user credentials, ...etc)**

## Step 7 — Full control, full understanding

With a real admin login on a fully working device, it stopped being a black box. I could see everything it was running, all its settings, the startup steps, and — most importantly — the main program that runs the device and its network features.

That main program is where a physical break-in turns into a remote one.

## Step 8 — The bug that spreads: a video link with no password

Reverse engineering the main program, I found how it handles its video links — and one hidden link that asks for no password at all. The normal links require a login. This one doesn't. Open a video player, point it at that link, and it just shows the live video. No username, no password.

This is the moment everything changes. Up to here I needed a screwdriver, a soldering iron, and the device in my hands. This needs only a network connection and the link. The hardware work is how I found the bug; the bug itself can be used by anyone who can reach the device over the network/internet.

**Note on what I'm sharing:** I'll describe the kind of bug (a hidden video link with no password), but I'm not publishing the exact link while devices are still online.

I opened VLC media player and connected to my camera (That is protected with a password) with the link I found:

<img width="896" height="657" alt="VLC_open" src="https://github.com/user-attachments/assets/17415f73-50c7-4afa-9909-680a640527d3" />

<img width="1662" height="932" alt="live_screenshot" src="https://github.com/user-attachments/assets/d8c3a700-81da-42b1-9dbb-352c1d76e921" />

## Step 9 — One spelling mistake, fifteen thousand devices

While reading the device's web server, I noticed something small and very useful: a spelling mistake — an odd, misspelled word in the server's responses that is specific to this exact software and almost certainly unique on the internet.

That mistake is a fingerprint. I built a search around it and ran it against a public internet scan. The result:

**15,000+ devices online running the same software.**

<img width="612" height="315" alt="internetscan" src="https://github.com/user-attachments/assets/f7094ec3-0966-471a-9acf-5a3aa93cb16b" />


Every one of them has the same hardcoded startup password, the same read-only system, the same writable settings area, and — the part that needs no hardware at all — the same video link with no password. The hardware work gave me the understanding; the software bug plus the fingerprint turned that understanding into a 15,000-device problem.

## What I did not do

To be totally clear: every step involving full control, the no-password video link, or changing passwords was done only on my own device — one I bought and own. The 15,000+ number comes purely from a passive internet scan (just matching public fingerprints). I did not connect to, log into, open video from, or touch a single one of those exposed devices — and I will not. Counting devices that share a fingerprint is not the same as accessing them, and that line is the line between research and breaking in. The number is there to measure how big the problem is, not to point at anyone.

## Why this matters: these devices are a real, ongoing danger

The point of this post isn't "a camera device had a bug." The point is that this kind of connected camera is a real, large-scale danger to the people/entities who own it — and most owners have no idea. Multiply that by 15,000 identical devices and it stops being a curiosity and becomes a privacy disaster at a huge scale.

The danger is in the shape of the failure, not just the number:

- A **hardware weakness** (a shared, built-in password) hands one person with one device total control and a full view of how the whole product line works.
- That view reveals a **software weakness** (a no-password video link) that needs no hardware at all to use.
- A **fingerprint** (the spelling mistake) turns "I can do this to my device" into "this is true for 15,000 devices I'll never physically touch."

Each step is ordinary on its own. Stacked together, they turn one quiet bench project into an internet-wide problem. Anyone who stops at "we passed the network test" misses this completely — because the root cause sits below the layer they tested, and the damage shows up above it.

## Lessons

**For the people who build these devices:**

- A startup password is useless if it's the same value baked into every unit. Lock down access, and if you must use a password, make it unique per device and not recoverable from the memory chip.
- Every video link is a door. "Hidden" is not the same as "locked."
- Unique strings — even typos — are free fingerprints for attackers. When all your devices are identical, that sameness is what lets one bug hit all of them.

**For other researchers:** a clean network test is the start of the interesting work on these devices, not the end. The bug that spreads is often the one you only find after you've earned full control the hard way.

## Wrap-up

One device on a bench, a soldering iron, and some patient reading turned into 15,000 exposed video feeds across the internet — not because any single step was fancy, but because when every device is identical, one shared flaw is every device's flaw. The hardware was the key; the scale was the door it opened.

Main tools: a UART connection, pulling and copying the firmware, binwalk, the device's read-only and writable file areas, the bootloader, a disassembler, and a public internet scan.

