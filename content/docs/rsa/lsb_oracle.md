---
weight: 999
title: "LSB Oracle"
description: ""
icon: "article"
date: "2023-12-02T16:52:38+05:30"
lastmod: "2023-12-02T16:52:38+05:30"
draft: true
toc: true
---


### Implementation
{{% alert icon="" context="info" %}}
#### Parameters Required:
1. N: the modulus
2. e: the public exponent
3. c: the encrypted message
4. oracle: a function which returns the last bit of a plaintext

***Return: the plaintext***
{{% /alert %}}

```python
from sage.all import ZZ


def attack(N, e, c, oracle):
    left = ZZ(0)
    right = ZZ(N)
    while right - left > 1:
        c = (c * pow(2, e, N)) % N
        if oracle(c) == 0:
            right = (right + left) / 2
        else:
            left = (right + left) / 2

    return int(right)
```