---
title: "bandit Writeup"
date: 2024-10-17 21:00
categories: [TCP1P CTF 2024, OSINT]
tags: [tcp1p, osint, license plate, exiftool, wikipedia]
authors: stefan
description: Look at the EXIF data of the image to find the location and use Wikipedia to determine the location the license plate belongs to.
image:
    path: /assets/img/writeups/tcp1p.png
---

## Challenge Description

An Jieyab as informant took a photo of a vehicle, can you find the location?

The flag is name the location and date example TCP1P{Town, Coutry. Month Year}

Example : TCP1P{Yogyakarta, Indonesia. June 2010}

## Solution

### Step 1: Analyze the Image Metadata Using EXIFTOOL

The first clue in this challenge comes from analyzing an image file. To extract metadata from an image, we use **ExifTool**, a tool specifically designed for this purpose.

```bash
exiftool suspect.jpg
```

In this case, running **ExifTool** reveals the following:

```plaintext
Date/Time Original              : 2019:10:25 17:00:00
```

The timestamp shows that the image was taken on **October 25, 2019, at 5:00 PM**.

### Step 2: Identifying the License Plate

Upon examining the image, we observe that a license plate is visible, and it contains the letter **N**.

This letter might be crucial in narrowing down the geographical region of the license plate, so we turn to external resources to investigate further.

### Step 3: Research Vehicle Registration Plates

Since the organizers of the CTF event are from **Indonesia**, we hypothesize that the license plate follows the Indonesian vehicle registration system.

After conducting some research on **Wikipedia**, we find a page about [Vehicle registration plates of Indonesia](https://en.wikipedia.org/wiki/Vehicle_registration_plates_of_Indonesia).

According to the page, license plates with the letter **N** are registered in **East Java**, particularly in the following regions:

- **Malang Residency**
- **Malang Regency**
- **City of Malang**
- **Pasuruan Regency**
- **City of Pasuruan**
- **Lumajang**
- **Batu**

Based on this information, we can reasonably conclude that the license plate originates from the **Malang** region in East Java, Indonesia. After trying, it turns out that **Malang** was the correct answer.

## Conclusion

To summarize the steps we took:

1. Used **ExifTool** to extract metadata from the image, finding the timestamp when it was captured.
2. Identified the letter **N** on the license plate and researched its significance.
3. Concluded that the license plate belongs to a vehicle from **Malang**, East Java, Indonesia.

## Flag
`TCP1P{Malang, Indonesia. October 2019}`
