---
title: "EncryptDecryptFile Writeup"
date: 2024-10-17 22:00
categories: [TCP1P CTF 2024, Forensics]
tags: [tcp1p, forensics, mercurial, hg, repository, decryption]
authors: stefan
description: Recover a deleted file from a repository that uses Mercurial (hg) for version control, decrypt it using a given script, and obtain the flag.
image:
    path: /assets/img/writeups/tcp1p.png
---

## Challenge Description

My brother deleted an important file from the encrypt-decrypt-file repository, help me recover it.

## Solution

### Step 1: Analyze the Repository with `hg log`

The challenge hints that the file we need has been deleted from a repository called **encrypt-decrypt-file**. Since the repository uses **Mercurial**, we start by looking at the commit history.

To view the commit history, we use the following command:

```bash
hg log
```

This command displays the repository's commit log, showing all changes made to the files, including deletions. By reviewing the log, we can identify which commit deleted the important file. The commit logs also provide useful metadata such as commit hashes, dates, and descriptions of changes made.

### Step 2: Revert the Repository to a Specific Revision

Once we’ve identified the commit where the file still existed, we can use the `hg revert` command to restore the repository to that state. In this case, the specific commit to revert to is `8fdb18e9618d`.

We run the following command to revert the file:

```bash
hg revert --rev 0:8fdb18e9618d -- flag.enc
```

Here’s what each part of this command does:
- `hg revert` reverts changes made to the repository or specific files.
- `--rev 0:8fdb18e9618d` specifies the revision range or the exact revision we want to restore (in this case, revision 0 through commit `8fdb18e9618d`).
- `-- flag.enc` tells Mercurial to only revert the file named `flag.enc`.

This restores the deleted file, **flag.enc**, from that specific commit.

### Step 3: Decrypt the Recovered File

After recovering the file, we need to decrypt it. According to the challenge, the encryption and decryption are handled by the **main.py** script included in the repository.

To decrypt the file, we use the following command:

```bash
python main.py --decrypt --input flag.enc --output out
```

Once the decryption is complete, the file is written to **out**, and it contains the decrypted content.

### Step 4: Retrieve the Flag

After decrypting the file, we notice that it is an image file. By opening the file, we find the flag embedded within the image.

## Conclusion

To summarize the steps we took:

1. **Checked the commit history** using `hg log` to identify the commit where the file was deleted.
2. **Reverted the repository** to that specific revision using `hg revert`, restoring the deleted file, **flag.enc**.
3. **Decrypted the file** using the Python script provided, with the `--decrypt` option.
4. **Retrieved the flag** from the decrypted file, which was an image.

## Flag
`TCP1P{introduction_to_hg_a82ffbe612}`
