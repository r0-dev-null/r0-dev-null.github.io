---
title: "Forevncrypt Writeup"
date: 2024-10-21 19:30
categories: [TCP1P CTF 2024, Forensics]
tags: [tcp1p, forensics, xor, pycdas, bash history, decryption, .img, pyinstxtractor]
authors: stefan
description: Extract .img file to get files, find the "compressor" binary and the bash history, use pyinstxtractor and pycdas to get partial source code, and "decompress" (decrypt) the target file to retrieve the flag.
image:
    path: /assets/img/writeups/tcp1p.png
---

## Challenge Description

After reading this article, I'm now hesitant to use xz-utils for compression:
https://www.openwall.com/lists/oss-security/2024/03/29/4
As a result, I've been searching for alternative compression tools. I found one online, but unfortunately this tools are trash  because it has limited features. Specifically, I’m unable to decompress files that were compressed with a password. Could you please help me recover them?

## Solution

### Step 1: Extracting the Disk Image

First, we extract the `.img` file with PowerISO to explore its contents.

Once inside, we find a bash history file and a suspicious file called `superimportantfile.xyz`.

### Step 2: Investigating the `bash_history`

In the extracted files, we find the `bash_history` file, which provides us some useful information on how the encryption was performed:

```bash
ls -lah
cd Desktop
../Downloads/forevncrypt note.txt
rm note.txt
cd Document
../Downloads/forevncrypt mydesign.odg
rm mydesign.odg
cd Music
../Downloads/forevncrypt myfavsong.mp3
rm myfavsong.mp3
cd Videos
../Downloads/forevncrypt superimportantfile.xyz -p thiswillbenotinrockyoubro
cd
```

This reveals that the file `superimportantfile.xyz` was encrypted using a tool called `forevncrypt` with the password `thiswillbenotinrockyoubro`.

### Step 3: Decompiling the Forevncrypt Python Executable

The encryption tool `forevncrypt` is not a common utility, so we extract it to inspect how it works. Using [PyInstxtractor](https://pyinstxtractor-web.netlify.app/), we decompile the Python bytecode (`.pyc` files) and analyze the partial source code using **PyCDAS**:

```bash
./pycdas app.pyc
```

The decompiled source code reveals an encryption process based on XOR and LZMA compression. Key sections of the code include:

```python
def xor(self, data, key):
    return bytes(a ^ b for a, b in zip(data, (key * (len(data) // len(key) + 1))[:len(data)]))

def encrypt_compress(self):
    compressed = self.compress()
    keygen = os.urandom(2)
    compressed = self.xor(compressed, keygen)
    result = self.xor(compressed, self.password.encode('utf-8'))
    return result
```

The encryption process involves two rounds of XOR: the first using a random key generated by `os.urandom(2)`, and the second using the password from `bash_history`.

### Step 4: Writing a Decryption Script

Using the insights gained from decompiling the source, we write a Python script to reverse the encryption. Here's the final decryption script:

```python
from Crypto.Cipher import AES
import os

def xor(data, key):
    """XOR operation between data and key."""
    return bytes(a ^ b for a, b in zip(data, (key * (len(data) // len(key) + 1))[:len(data)]))

def decrypt_file(encrypted_file, password):
    # Read the encrypted file
    with open(encrypted_file, 'rb') as f:
        file_content = f.read()

    # Skip the first 37 bytes (header)
    data_after_header = file_content[37:]

    # Known random value generated by os.urandom(2)
    urandom_xor_value = bytes.fromhex('80FE')  # Based on analysis

    # XOR with the password
    xor_with_password = xor(data_after_header, password.encode('utf-8'))

    # XOR the result with the known urandom XOR value
    decrypted_data = xor(xor_with_password, urandom_xor_value)

    return decrypted_data

if __name__ == '__main__':
    password = 'thiswillbenotinrockyoubro'
    decrypted_content = decrypt_file('superimportantfile.xyz', password)

    # Save the decrypted content
    with open('decrypted_superimportantfile.mp4', 'wb') as f:
        f.write(decrypted_content)

    print("Decryption successful! Decrypted file saved as decrypted_superimportantfile.mp4")
```

### Step 5: Running the Decryption Script

We execute the script to decrypt the file and find out that it's an MP4 video file.

### Step 6: Recovering the Flag

Upon playing the decrypted video file, we find that the flag is embedded within the video.

## Conclusion

To summarize the steps we took:

1. We extracted the `.img` file to explore its contents and found a `bash_history` file and a suspicious file called `superimportantfile.xyz`.
2. We investigated the `bash_history` file, which revealed the password used for encryption.
3. We decompiled the `forevncrypt` Python executable using **PyInstxtractor** and **PyCDAS** to understand the encryption mechanism.
4. We wrote a decryption script based on the insights gained from the decompiled source code.
5. We executed the decryption script to decrypt the file, which turned out to be an MP4 video.
6. Upon playing the decrypted video file, we found the flag embedded within the video.

By carefully following these steps, we were able to decrypt the contents and retrieve the flag.

## Flag
`TCP1P{3_challenge_in_one_category_ummm_hehe}`