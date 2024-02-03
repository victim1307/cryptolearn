---
weight: 999
title: "Extended Wiener Attack"
description: ""
icon: "article"
date: "2023-12-02T16:51:38+05:30"
lastmod: "2023-12-02T16:51:38+05:30"
draft: true
toc: true
---

### Implementation
{{% alert icon="" context="info" %}}
#### Parameters Required:
1. max_s: the amount of s values to try (default: 20000)
2. max_r: the amount of r values to try for each s value (default: 100)
3. N: the modulus
4. e: the public exponent
5. max_t: the amount of t values to try for each s value (default: 100)

***Return: a tuple containing the prime factors and the private exponent, or None if the private exponent was not found***
{{% /alert %}}

```python
import os
import sys

from sage.all import RR
from sage.all import ZZ
from sage.all import continued_fraction

from math import gcd
from math import isqrt
from random import randrange

from sage.all import is_prime


def factorize(N, phi):
    s = N + 1 - phi
    d = s ** 2 - 4 * N
    p = int(s - isqrt(d)) // 2
    q = int(s + isqrt(d)) // 2
    return p, q if p * q == N else None



def attack(n, e, max_s=20000, max_r=100, max_t=100):
    i_n = ZZ(n)
    i_e = ZZ(e)
    threshold = i_e / i_n + (RR(2.122) * i_e) / (i_n * i_n.sqrt())
    convergents = continued_fraction(i_e / i_n).convergents()
    for i in range(1, len(convergents) - 2, 2):
        if convergents[i + 2] < threshold < convergents[i]:
            m = i
            break
    else:
        return None

    for s in range(max_s):
        for r in range(max_r):
            k = r * convergents[m + 1].numerator() + s * convergents[m + 1].numerator()
            d = r * convergents[m + 1].denominator() + s * convergents[m + 1].denominator()
            if pow(pow(2, e, n), d, n) != 2:
                continue

            phi = (e * d - 1) // k
            factors = known_phi.factorize(n, phi)
            if factors:
                return *factors, int(d)

        for t in range(max_t):
            k = s * convergents[m + 2].numerator() - t * convergents[m + 1].numerator()
            d = s * convergents[m + 2].denominator() - t * convergents[m + 1].denominator()
            if pow(pow(2, e, n), d, n) != 2:
                continue

            phi = (e * d - 1) // k
            factors = known_phi.factorize(n, phi)
            if factors:
                return *factors, int(d)
```