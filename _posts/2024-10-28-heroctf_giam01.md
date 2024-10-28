---
title: "Giam v0.01 Writeup"
date: 2024-10-28 11:00
categories: [HeroCTF 2024, Game Hacking]
tags: [heroctf, game hacking, rev, ida, breakpoints, jump]
authors: stefan
description: Use IDA to find the function responsible for displaying the flag and jump to it for an unintended solve.
image:
    path: /assets/img/writeups/heroctf.png
---

## Challenge Description

A giam you must hax

## Solution

### Step 1: Analyzing the Binary

We start by loading the binary into IDA and analyzing the code, which, unfortunately, is compiled from Rust. Looking at the rdata binary, we can see an interesting string:

![Interesting String](/assets/img/writeups/giam01/interesting_string.png)

### Step 2: Identifying the flag function

By scrolling back up to the start of the diassembly and searching for "Congratz" (Ctrl + F), it will take us to the function that interacts with it:

![Disassembly](/assets/img/writeups/giam01/disass.png)

Switching to decompiler view, we can see that it's part of a switch:

![Switch](/assets/img/writeups/giam01/switch.png)

Looking at the full function, it has a beautiful number of 1488 lines, which indicates that it's the main loop or `FrameHandler`.

### Step 3: Setting breakpoints and jumping

Going to the start of the function, we can find the switch instruction and we set a breakpoint to it:

![Breakpoint](/assets/img/writeups/giam01/breakpoint.png)

Running the binary and hitting the breakpoint, I pressed F9 a few times to let the menu fully initialize. Afterwards, we go
to the case that displays the flag and `Set IP` to jump to it:

{: .prompt-info }
> We can synchronize the decompiler view with the disassembly view (as we can only set IP from disassembly view) by right-clicking on the decompiler view and selecting `Synchronize with` -> `IDA View-RIP`.

![Target_jump_decomp](/assets/img/writeups/giam01/target_jump_decompiled.png)

![Target_jump](/assets/img/writeups/giam01/target_jump.png)

We right click the first instruction, `Set IP` and, by pressing F9, we can see the flag being displayed:

![Flag](/assets/img/writeups/giam01/flag.png)

## Conclusion

To summarize the steps we took:

1. We loaded the binary into IDA and analyzed the code to find a string that contains Hero{}.
2. We identified the function responsible for displaying the flag by searching for the string "Congratz" in the disassembly.
3. We set breakpoints at the start of the function and ran the binary to hit the breakpoint.
4. We synchronized the decompiler view with the disassembly view and set the instruction pointer to the case displaying the flag.
5. By pressing F9, we were able to see the flag being displayed.

By carefully following these steps, we were able to use IDA to find the function responsible for displaying the flag and jump to it for an unintended solve.

## Flag

`Hero{&1Kb,6Kb08Lb-3Jb}`
