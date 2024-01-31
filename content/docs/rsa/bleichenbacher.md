---
weight: 999
title: "Bleichenbacher"
description: ""
icon: "article"
date: "2023-12-02T16:49:27+05:30"
lastmod: "2023-12-02T16:49:27+05:30"
draft: true
toc: true
---


The attack exploits the way RSA ciphertext is formatted in PKCS#1 v1.5 padding. In this padding scheme, the plaintext is prefixed with random non-zero pad bytes before encrypting.

The steps are as follows:
1. The attacker sends manipulated ciphertexts to the target and observes the response to deduce information about the padding.
2. The ciphertext is multiplied by a blinding factor and sent to the server. Based on whether the ciphertext is accepted or rejected, the possible interval for the plaintext is narrowed down.
3. By interacting with the server multiple times, the interval is narrowed down until only one possible plaintext remains. This reveals the original plaintext.
4. The attacker never needs the private key as the padding format allows deducing the plaintext by observing the target's responses.

So in brief, the adaptive chosen ciphertext attack exploits flaws in the PKCS#1 v1.5 padding to decrypt the ciphertext without requiring the private key. Only the public key and access to the target server is needed to iteratively recover the plaintext.


{{< embed-pdf url="../files/bleichenbacher.pdf" >}}

### Implementation
{{% alert icon="" context="info" %}}
#### Parameters Required:
1. padding_oracle: the padding oracle taking integers, returns True if the PKCS #1 v1.5 padding is correct, False otherwise
2. n: the modulus
3. e: the public exponent
4. c: the ciphertext (integer)

***Return: the plaintext (integer)***
{{% /alert %}}


```python
import logging
import os
import sys
from random import randrange

def ceil_div(a, b):
    return a // b + (a % b > 0)

def floor_div(a, b):
    return a // b

def _insert(M, a, b):
    for i, (a_, b_) in enumerate(M):
        if a_ <= b and a <= b_:
            a = min(a, a_)
            b = max(b, b_)
            M[i] = (a, b)
            return

    M.append((a, b))
    return


# Step 1.
def _step_1(padding_oracle, n, e, c):
    s0 = 1
    c0 = c
    while not padding_oracle(c0):
        s0 = randrange(2, n)
        c0 = (c * pow(s0, e, n)) % n

    return s0, c0


# Step 2.a.
def _step_2a(padding_oracle, n, e, c0, B):
    s = ceil_div(n, 3 * B)
    while not padding_oracle((c0 * pow(s, e, n)) % n):
        s += 1

    return s


# Step 2.b.
def _step_2b(padding_oracle, n, e, c0, s):
    s += 1
    while not padding_oracle((c0 * pow(s, e, n)) % n):
        s += 1

    return s


# Step 2.c.
def _step_2c(padding_oracle, n, e, c0, B, s, a, b):
    r = ceil_div(2 * (b * s - 2 * B), n)
    while True:
        left = ceil_div(2 * B + r * n, b)
        right = floor_div(3 * B + r * n, a)
        for s in range(left, right + 1):
            if padding_oracle((c0 * pow(s, e, n)) % n):
                return s

        r += 1


# Step 3.
def _step_3(n, B, s, M):
    M_ = []
    for (a, b) in M:
        left = ceil_div(a * s - 3 * B + 1, n)
        right = floor_div(b * s - 2 * B, n)
        for r in range(left, right + 1):
            a_ = max(a, ceil_div(2 * B + r * n, s))
            b_ = min(b, floor_div(3 * B - 1 + r * n, s))
            _insert(M_, a_, b_)

    return M_


def attack(padding_oracle, n, e, c):
    k = ceil_div(n.bit_length(), 8)
    B = 2 ** (8 * (k - 2))
    logging.info("Executing step 1...")
    s0, c0 = _step_1(padding_oracle, n, e, c)
    M = [(2 * B, 3 * B - 1)]
    logging.info("Executing step 2.a...")
    s = _step_2a(padding_oracle, n, e, c0, B)
    M = _step_3(n, B, s, M)
    logging.info("Starting while loop...")
    while True:
        if len(M) > 1:
            s = _step_2b(padding_oracle, n, e, c0, s)
        else:
            (a, b) = M[0]
            if a == b:
                m = (a * pow(s0, -1, n)) % n
                return m
            s = _step_2c(padding_oracle, n, e, c0, B, s, a, b)
        M = _step_3(n, B, s, M)
```

## Bleichenbacher Signature Forgery
The attack exploits properties of the PKCS#1 v1.5 signature padding scheme. The signature is computed as:

s = md^d mod n

Where m is the padded message, d is the private key and n is the RSA modulus.

The padding has a specific format that includes random non-zero bytes. The forged signature s' is iteratively constructed such that:

1. The padding conforms to the expected format.
2. When verified with the public key, the verification equation holds: (s')^e = m' mod n

Where e is the public exponent and m' is the message chosen by the attacker.

By observing the verification responses for manipulated signatures, the attacker can deduce boundary conditions on the padded message m. This allows constructing a properly formatted m' that verifies correctly with the public key but is not the actual signed message.

The attack relies on flaws in the deterministic PKCS#1 v1.5 padding scheme rather than mathematical weaknesses in RSA itself. Using probabilistic padding schemes like PSS can mitigate the risk of forgery.

Bleichenbacher's adaptive chosen ciphertext attack can be adapted to forge RSA signatures without knowing the private key by exploiting properties of the padding scheme.

### Implementation
{{% alert icon="" context="info" %}}
#### Parameters Required:
1. suffix: the suffix
2. suffix_bit_length: the bit length of the suffix

***Return: the number s***
{{% /alert %}}

```python
def attack(suffix, suffix_bit_length):
    assert suffix % 2 == 1, "Target suffix must be odd"

    s = 1
    for i in range(suffix_bit_length):
        if (((s ** 3) >> i) & 1) != ((suffix >> i) & 1):
            s |= (1 << i)

    return s
```