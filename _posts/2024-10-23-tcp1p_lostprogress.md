---
title: "Lost Progress Writeup"
date: 2024-10-23 21:00
categories: [TCP1P CTF 2024, Forensics]
tags: [tcp1p, forensics, volatility, memory forensics, gimp, memdump, visualizing raw memory]
authors: stefan
description: Use GIMP to visualize raw memory data from a memory capture, reveal the passwords, one from an image editor and another from a text editor and combine them to get the flag.
image:
    path: /assets/img/writeups/tcp1p.png
---

## Challenge Description

My friend Andi just crashed his computer and all the progress he made are gone. It was 2 of his secret passwords with each of them being inside an image and a text file. Luckily he has an automatic RAM capture program incase something like this happen, but no idea on how to use it...

## Solution

### Step 1: Extracting Memory Dumps for GIMP and VSCode

The first step involves extracting relevant memory pages from the GIMP and VSCode processes using **Volatility**. We identify the Process IDs (PIDs) for both GIMP and VSCode and dump their memory regions for further analysis.

#### Dumping GIMP's Memory:

```bash
py vol.py -o __OUT -f dumped windows.memmap --pid 5380 --dump
```

The output file is renamed to `gimp.data` for clarity.

#### Dumping VSCode's Memory:

```bash
py vol.py -o __OUT -f dumped windows.memmap --pid 1716 --dump
```

This output is renamed to `vscode.data`.

### Step 2: Recovering the Note from GIMP's Memory Dump

Next, we open the `gimp.data` file using GIMP to attempt recovery of the note. According to this [article](https://beguier.eu/nicolas/articles/security-tips-2-volatility-gimp.html). Here's the process for extracting the note:

1. Open GIMP and import the memory dump file `gimp.data`.
2. Experiment with the **offset** and **width** sliders to adjust how the raw data is interpreted. The format to use is **RGB Alpha** to get the proper color channels.
3. Continue tweaking these values until a readable Notepad-like output begins to form.

After several attempts, we successfully recover the note. Here’s a preview of the reconstructed note:
![Recovered Note](https://i.imgur.com/DYoxBvx.png)

### Step 3: Recovering the Image from VSCode's Memory Dump

Now, we perform a similar process for the VSCode memory dump to retrieve the image:

1. Open GIMP and import the `vscode.data` file.
2. Similar to the previous step, adjust the **offset** and **width** sliders to see the image as a readable format.
3. Eventually, a recognizable image begins to form.

This process allows us to extract a hidden image from the VSCode memory dump.
![Recovered Image](https://i.imgur.com/9vekobS.png)

### Step 4: Assembling the Final Flag

After recovering both the image and the text from the memory dumps, we combine them to form the final flag.

```plaintext
wIeRRRMQqykX6zs3O7KSQY6Xq6z4TKnr_ekxyAH2jIrh0Opyu432tk9y0KdiujkMu
```

## Conclusion

To summarize the steps we took:

1. We extracted memory dumps for GIMP and VSCode using **Volatility**.
2. We recovered the note from GIMP's memory dump by visualizing the raw memory data in GIMP.
3. We recovered the image from VSCode's memory dump using a similar process in GIMP.
4. We combined the recovered note and image to form the final flag.

By carefully following these steps, we were able to recover the hidden data from the memory dumps and retrieve the flag.

## Flag

`TCP1P{wIeRRRMQqykX6zs3O7KSQY6Xq6z4TKnr_ekxyAH2jIrh0Opyu432tk9y0KdiujkMu}`
