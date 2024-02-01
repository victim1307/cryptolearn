---
weight: 999
title: "Cherkaoui Semmouni"
description: ""
icon: "article"
date: "2023-12-02T16:49:59+05:30"
lastmod: "2023-12-02T16:49:59+05:30"
draft: true
toc: true
---


{{< embed-pdf url="../files/cherkaoui_semmouni.pdf" >}}


### Implementation
{{% alert icon="" context="info" %}}
#### Parameters Required:
1. N: the modulus
2. e: the exponent
3. beta: the parameter beta such that |p - q| <= N^beta
4. delta: the parameter delta such that d <= N^delta
5. m: the m value to use for the small roots method (default: 1)
6. t: the t value to use for the small roots method (default: automatically computed using m)
7. check_bounds: perform bounds check (default: True)

***Return: a tuple containing the prime factors and the private exponent, or None if the factors could not be found***
{{% /alert %}}


```python
import logging
import os
import sys
from math import isqrt
from math import log
from math import sqrt

from sage.all import RR
from sage.all import ZZ

path = os.path.dirname(os.path.dirname(os.path.dirname(os.path.realpath(os.path.abspath(__file__)))))
if sys.path[1] != path:
    sys.path.insert(1, path)


def modular_bivariate(f, e, m, t, X, Y, roots_method="groebner"):
    f = f.change_ring(ZZ)

    pr = ZZ["x", "y", "u"]
    x, y, u = pr.gens()
    qr = pr.quotient(1 + x * y - u)
    U = X * Y

    logging.debug("Generating shifts...")

    shifts = []
    for k in range(m + 1):
        for i in range(m - k + 1):
            g = x ** i * f ** k * e ** (m - k)
            g = qr(g).lift()
            shifts.append(g)

    for j in range(1, t + 1):
        for k in range(m // t * j, m + 1):
            h = y ** j * f ** k * e ** (m - k)
            h = qr(h).lift()
            shifts.append(h)

    L, monomials = small_roots.create_lattice(pr, shifts, [X, Y, U])
    L = small_roots.reduce_lattice(L)

    pr = f.parent()
    x, y = pr.gens()

    polynomials = small_roots.reconstruct_polynomials(L, f, None, monomials, [X, Y, U], preprocess_polynomial=lambda p: p(x, y, 1 + x * y))
    for roots in small_roots.find_roots(pr, polynomials, method=roots_method):
        yield roots[x], roots[y]

def attack(N, e, beta, delta, m=1, t=None, check_bounds=True):
    alpha = log(e, N)
    assert not check_bounds or delta < 2 - sqrt(2 * alpha * beta), f"Bounds check failed ({delta} < {2 - sqrt(2 * alpha * beta)})."

    x, y = ZZ["x", "y"].gens()
    A = -(N - 1) ** 2
    f = x * y + A * x + 1
    X = int(2 * e * RR(N) ** (delta - 2))  # Equivalent to 2N^(alpha + delta - 2)
    Y = int(RR(N) ** (2 * beta))
    t = int((2 - delta - 2 * beta) / (2 * beta) * m) if t is None else t
    logging.info(f"Trying {m = }, {t = }...")
    for x0, y0 in modular_bivariate(f, e, m, t, X, Y):
        s = isqrt(y0)
        d = s ** 2 + 4 * N
        p = int(-s + isqrt(d)) // 2
        q = int(s + isqrt(d)) // 2
        d = int(f(x0, y0) // e)
        return p, q, d

    return None
```