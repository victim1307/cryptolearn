---
weight: 999
title: "Hastad Attack"
description: ""
icon: "article"
date: "2023-12-02T16:51:57+05:30"
lastmod: "2023-12-02T16:51:57+05:30"
draft: true
toc: true
---

### Implementation

{{% alert icon="" context="info" %}}
#### Parameters Required:
1. N: the modulus
2. e: the public exponent
3. c: the ciphertexts

***Return: the plaintext***
{{% /alert %}}

```python
import os
import sys
from math import gcd
from sage.all import ZZ

from sage.all import crt
from math import lcm


def fast_crt(X, M, segment_size=8):
    assert len(X) == len(M)
    assert len(X) > 0
    while len(X) > 1:
        X_ = []
        M_ = []
        for i in range(0, len(X), segment_size):
            if i == len(X) - 1:
                X_.append(X[i])
                M_.append(M[i])
            else:
                X_.append(crt(X[i:i + segment_size], M[i:i + segment_size]))
                M_.append(lcm(*M[i:i + segment_size]))
        X = X_
        M = M_

    return X[0], M[0]

def low_exponent_attack(e, c):
    return int(ZZ(c).nth_root(e))

def attack(N, e, c):
    """
    Recovers the plaintext from e ciphertexts, encrypted using different moduli and the same public exponent.
    :param N: the moduli
    :param e: the public exponent
    :param c: the ciphertexts
    :return: the plaintext
    """
    assert e == len(N) == len(c), "The amount of ciphertexts should be equal to e."

    for i in range(len(N)):
        for j in range(len(N)):
            if i != j and gcd(N[i], N[j]) != 1:
                raise ValueError(f"Modulus {i} and {j} share factors, Hastad's attack is impossible.")

    c, _ = fast_crt(c, N)
    return low_exponent_attack(e, c)
```