---
weight: 41
title: "Caesar Cipher"
description: ""
icon: "article"
date: "2023-12-02T14:07:13+05:30"
lastmod: "2023-12-02T14:07:13+05:30"
draft: true
toc: true
---

The Caesar cipher is one of the oldest and simplest methods of encryption. It's a type of substitution cipher, meaning each letter in the original message (plaintext) is replaced with a different letter from the alphabet. This replacement follows a fixed rule: shifting each letter a certain number of positions down the alphabet.

#### How It Works

* Shift (or Key): The core parameter of a Caesar cipher is the shift or key. This is a number that determines how many places each letter will be moved.
* Encryption: To encrypt a message, each letter is shifted down the alphabet by the key value. For example, with a shift of 3:
    * 'A' becomes 'D'
    * 'B' becomes 'E'
    * 'Z' becomes 'C' (wrapping around from the end)
* Decryption: The decryption process is simply the reverse. Each letter is shifted backwards by the key value.