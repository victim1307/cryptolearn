---
weight: 999
title: "Boneh Durfee"
description: ""
icon: "article"
date: "2023-12-02T16:49:42+05:30"
lastmod: "2023-12-02T16:49:42+05:30"
draft: true
toc: true
---
The attack relies on the fact that if d is small, the prime factors p and q of the RSA modulus n must lie in a relatively narrow range that can be determined from d and n.

The steps of the Boneh-Durfee attack to factor RSA moduli when the private exponent d is small:

1. Estimate the number of bits k in the prime factors p and q of the RSA modulus n. This is approximately k = ceil(log(n, 2)) / 2.
2. Use the knowledge that d is small to get an upper bound on p and q. Since p and q are roughly 2^k, we know p < 2^k and q < 2^k.
3. Also use d to derive a lower bound on p and q. Based on the relationship between p, q, d and the Euler's totient function φ(n), we get: p ≥ 2^(k-d/2) and q ≥ 2^(k-d/2).
4. Generate all prime numbers in the range [lower_bound, upper_bound]. This gives us a set of candidate values for p and q.
5. Test each candidate p by computing q = n/p and checking if p*q = n. Additionally check that p > q as required for RSA.
6. If a valid factorization is found, we have recovered p and q. These can then be used to compute the private exponent and decrypt the ciphertext.

The attack relies on d being small to significantly reduce the search space for p and q. By generating primes in this smaller range and testing candidates, the factorization can be recovered in feasible time.


{{< embed-pdf url="../files/boneh_durfee.pdf" >}}

### Implementation

{{% alert icon="" context="info" %}}
#### Parameters Required:
1. N: the modulus
2. e: the public exponent
3. factor_bit_length: the bit length of the prime factors
4. partial_p: the partial prime factor p (PartialInteger) (default: None)
5. delta: a predicted bound on the private exponent (d < N^delta) (default: 0.25)
6. m: the m value to use for the small roots method (default: 1)
7. t: the t value to use for the small roots method (default: automatically computed using m)

***Return: a tuple containing the prime factors, or None if the factors were not found***
{{% /alert %}}


```python
import logging
import os
import sys
from math import gcd
from math import isqrt
from random import randrange
from sage.all import RR
from sage.all import ZZ
from sage.all import is_prime


def factorize(N, phi):
    s = N + 1 - phi
    d = s ** 2 - 4 * N
    p = int(s - isqrt(d)) // 2
    q = int(s + isqrt(d)) // 2
    return p, q if p * q == N else None


def factorize_multi_prime(N, phi):
    prime_factors = set()
    factors = [N]
    while len(factors) > 0:
        # Element to factorize.
        N = factors[0]

        w = randrange(2, N - 1)
        i = 1
        while phi % (2 ** i) == 0:
            sqrt_1 = pow(w, phi // (2 ** i), N)
            if sqrt_1 > 1 and sqrt_1 != N - 1:
                # We can remove the element to factorize now, because we have a factorization.
                factors = factors[1:]

                p = gcd(N, sqrt_1 + 1)
                q = N // p

                if is_prime(p):
                    prime_factors.add(p)
                elif p > 1:
                    factors.append(p)

                if is_prime(q):
                    prime_factors.add(q)
                elif q > 1:
                    factors.append(q)

                # Continue in the outer loop
                break

            i += 1

    return tuple(prime_factors)

def attack(N, e, factor_bit_length, partial_p=None, delta=0.25, m=1, t=None):
    # Use additional information about factors to speed up Boneh-Durfee.
    p_lsb, p_lsb_bit_length = (0, 0) if partial_p is None else partial_p.get_known_lsb()
    q_lsb = (pow(p_lsb, -1, 2 ** p_lsb_bit_length) * N) % (2 ** p_lsb_bit_length)
    A = ((N >> p_lsb_bit_length) + pow(2, -p_lsb_bit_length, e) * (p_lsb * q_lsb - p_lsb - q_lsb + 1))

    x, y = ZZ["x", "y"].gens()
    f = x * (A + y) + pow(2, -p_lsb_bit_length, e)
    X = int(RR(e) ** delta)
    Y = int(2 ** (factor_bit_length - p_lsb_bit_length + 1))
    t = int((1 - 2 * delta) * m) if t is None else t
    logging.info(f"Trying {m = }, {t = }...")
    for x0, y0 in herrmann_may.modular_bivariate(f, e, m, t, X, Y):
        z = int(f(x0, y0))
        if z % e == 0:
            k = pow(x0, -1, e)
            s = (N + 1 + k) % e
            phi = N - s + 1
            factors = known_phi.factorize(N, phi)
            if factors:
                return factors

    return None


def attack_multi_prime(N, e, factor_bit_length, factors, delta=0.25, m=1, t=None):
    x, y = ZZ["x", "y"].gens()
    A = N + 1
    f = x * (A + y) + 1
    X = int(RR(e) ** delta)
    Y = int(2 ** ((factors - 1) * factor_bit_length + 1))
    t = int((1 - 2 * delta) * m) if t is None else t
    logging.info(f"Trying {m = }, {t = }...")
    for x0, y0 in herrmann_may.modular_bivariate(f, e, m, t, X, Y):
        z = int(f(x0, y0))
        if z % e == 0:
            k = pow(x0, -1, e)
            s = (N + 1 + k) % e
            phi = N - s + 1
            factors = known_phi.factorize_multi_prime(N, phi)
            if factors:
                return factors

    return None
```