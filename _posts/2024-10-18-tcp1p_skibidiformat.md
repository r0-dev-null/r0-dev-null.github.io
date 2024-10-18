---
title: "Skibidi Format Writeup"
date: 2024-10-18 16:30
categories: [TCP1P CTF 2024, Forensics]
tags: [tcp1p, forensics, custom spec, decryption, AES, zstd, image reconstruction]
authors: stefan
description: Decrypt a custom image format using header data, decompress the image data, and reconstruct it to find the flag hidden within.
image:
    path: /assets/img/writeups/tcp1p.png
---

## Challenge Description

So my friend just made a new image format and asked me to give him a test file, so I gave him my favorite png of all time. But the only thing I receive back is just my image with his new format and its "specification" file, don't know what that is. Can you help me read this file?


## Solution

### Step 1: Parsing the Header

The first step is to extract and parse the header of the file to retrieve essential metadata such as image dimensions, color channels, compression method, and AES encryption parameters (key and initialization vector). The `struct` library in Python can be used to decode the binary header.

```
+----------------------+-----------------------+
|       Header         |      Data Section     |
+----------------------+-----------------------+
|  Magic Number (4B)   | Encrypted Data        |
|  Width (4B)          |                       |
|  Height (4B)         |                       |
|  Channels (1B)       |                       |
|  Compression ID (1B) |                       |
|  AES Key (32B)       |                       |
|  AES IV (12B)        |                       |
+----------------------+-----------------------+
```

```python
import struct

def parse_header(header_data):
    # Unpack the header: Magic number, Width, Height, Channels, Compression ID, AES Key, AES IV
    magic, width, height, channels, compression_id, aes_key, aes_iv = struct.unpack('<4sIIbB32s12s', header_data)

    # Check if the magic number matches
    if magic != b'SKB1':
        raise ValueError("Invalid file format: Magic number mismatch.")

    return {
        'width': width,
        'height': height,
        'channels': channels,
        'compression_id': compression_id,
        'aes_key': aes_key,
        'aes_iv': aes_iv
    }

# Example usage to extract the header
with open('suisei.skibidi', 'rb') as file:
    header_data = file.read(58)
    header_info = parse_header(header_data)
    print(header_info)
```

From the header, we extract the width, height, number of channels, compression method (Zstandard), and AES encryption key and IV.

### Step 2: Decrypting the Encrypted Data

The encrypted data is protected using AES-256-GCM, which ensures both confidentiality and integrity with an authentication tag. We extract the tag from the end of the encrypted data and use the key and IV from the header to decrypt the content.

```python
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.backends import default_backend

def decrypt_data(aes_key, aes_iv, encrypted_data):
    # Extract the authentication tag (last 16 bytes)
    auth_tag = encrypted_data[-16:]
    encrypted_data = encrypted_data[:-16]  # Remove the tag from the data

    # Create a decryptor for AES-256-GCM
    decryptor = Cipher(
        algorithms.AES(aes_key),
        modes.GCM(aes_iv, auth_tag),
        backend=default_backend()
    ).decryptor()

    # Perform decryption
    decrypted_data = decryptor.update(encrypted_data) + decryptor.finalize()
    return decrypted_data

# Usage to decrypt data
with open('suisei.skibidi', 'rb') as file:
    file.seek(58)  # Skip the header
    encrypted_data = file.read()
    decrypted_data = decrypt_data(header_info['aes_key'], header_info['aes_iv'], encrypted_data)
```

This step gives us the compressed image data in its raw format.

### Step 3: Decompressing the Data

The decrypted data is compressed using Zstandard (zstd). We use the command-line tool `zstd` to decompress the data into its raw pixel format.

```bash
zstd -d decrypted_data.bin -o decompressed_data.bin
```

This decompresses the data into a binary file containing raw pixel data.

### Step 4: Reconstructing the Image

With the decompressed data available, we can now reconstruct the original image. The image can either be in RGB or RGBA format, depending on the number of channels provided in the header. We use the `PIL` library to handle the raw pixel data and save the final image as a PNG file.

```python
from PIL import Image

def reconstruct_image(decompressed_data, width, height, channels, output_file):
    mode = 'RGB' if channels == 3 else 'RGBA'  # Determine image mode
    image = Image.frombytes(mode, (width, height), decompressed_data)
    image.save(output_file, 'PNG')

# Usage example
with open('decompressed_data.bin', 'rb') as file:
    pixel_data = file.read()
    reconstruct_image(pixel_data, header_info['width'], header_info['height'], header_info['channels'], 'reconstructed_image.png')
```

### Step 5: Retrieving the Flag

Opening the reconstructed image (`reconstructed_image.png`), we find the flag embedded within the image.

## Conclusion

To solve this challenge, we followed these steps:
1. Parsed the file's custom header to extract critical metadata, including AES encryption parameters.
2. Decrypted the image data using the AES key and IV.
3. Decompressed the decrypted data using the Zstandard compression method.
4. Reconstructed the image from the raw pixel data, revealing the flag.

## Flag
`TCP1P{S3ems_L1k3_Sk1b1dI_T0il3t_h4s_C0nsUm3d_My_fr13nD_U72Syd6}`
