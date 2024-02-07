---
weight: 999
title: "Known D"
description: ""
icon: "article"
date: "2023-12-02T16:52:21+05:30"
lastmod: "2023-12-02T16:52:21+05:30"
draft: true
toc: true
---


### Implementation
{{% alert icon="" context="info" %}}
#### Parameters Required:
1. N: the modulus
2. e: the public exponent
3. d: the private exponent

***Return: a tuple containing the prime factors***
{{% /alert %}}

```python
from math import gcd
from random import randrange


def attack(N, e, d):
    k = e * d - 1
    t = 0
    while k % (2 ** t) == 0:
        t += 1

    while True:
        g = randrange(1, N)
        for s in range(1, t + 1):
            x = pow(g, k // (2 ** s), N)
            p = gcd(x - 1, N)
            if 1 < p < N and N % p == 0:
                return p, N // p

```