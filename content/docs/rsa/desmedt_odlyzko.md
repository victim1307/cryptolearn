---
weight: 999
title: "Desmedt Odlyzko"
description: ""
icon: "article"
date: "2023-12-02T16:50:57+05:30"
lastmod: "2023-12-02T16:50:57+05:30"
draft: true
toc: true
---

In Desmedt and Odlyzko's attack, which is a type of existential forgery, the forger requests signatures on messages of their choosing from the legitimate owner of the private key d. After obtaining these signatures, the forger can then compute a signature on a new, arbitrary message that was never actually signed by the real owner of d. This allows the forger to produce a valid signature on any message, even a meaningless one. In the case of Rabin-Williams signatures, the attack could enable the forger to completely break the scheme by factoring the modulus N.

{{< alert context="warning" text="Note that this selective forgery attack is much slower than the existential forgery attack. However, it is also more applicable to real world scenarios." />}}

{{< embed-pdf url="../files/desmedt_odlyzko.pdf">}}

### Implementation
{{% alert icon="" context="info" %}}
#### Parameters Required:
1. hash_oracle: the oracle taking integer messages, returns an integer representation of the hashed message
2. sign_oracle: the oracle taking integer messages, returns an integer representation of the signature
3. N: the modulus
4. e: the public exponent
5. target_m: the target message to sign (integer)

***Return: the signature of the target message (integer)***
{{% /alert %}}


```python
import logging

from sage.all import GF
from sage.all import ZZ
from sage.all import factor
from sage.all import matrix
from sage.all import next_prime
from sage.all import vector


def attack(hash_oracle, sign_oracle, N, e, target_m):
    target_factors = factor(hash_oracle(target_m))
    B, _ = target_factors[-1]

    logging.info(f"Computing all primes <= {B}...")
    primes = {}
    p = 2
    i = 0
    while p <= B:
        primes[p] = i
        p = next_prime(p)
        i += 1

    l = len(primes)

    Vt = vector(GF(e), l, sparse=True)
    for p, v in target_factors:
        Vt[primes[p]] = v

    logging.info(f"Generating initial {l}x{l} matrix...")
    M = matrix(GF(e), l, sparse=True)
    m = []
    mi = 0
    i = 0
    while i < l:
        factors = factor(hash_oracle(mi))
        if all(p <= B for p, _ in factors):
            for p, v in factors:
                M[i, primes[p]] = v
            m.append(mi)
            i += 1
        mi += 1

    rank = M.rank()
    logging.info(f"Extending initial matrix with rank {rank} (required = {l})...")
    while rank < l:
        Vi = vector(GF(e), l, sparse=True)
        factors = factor(hash_oracle(mi))
        if all(p <= B for p, _ in factors):
            for p, v in factors:
                Vi[primes[p]] = v
            M_ = M.stack(Vi)
            rank_ = M_.rank()
            if rank_ > rank:
                M = M_
                rank = rank_
                m.append(mi)
                logging.debug(f"New rank: {rank}...")
        mi += 1

    logging.info(f"Found {M.nrows()}x{M.ncols()} basis matrix")

    b = M.solve_left(Vt)

    logging.info(f"Found linear combination of target vector")

    Vt = Vt.change_ring(ZZ)
    M = M.change_ring(ZZ)
    b = b.change_ring(ZZ)
    G = (Vt - b * M) / e
    delta = 1
    for pj, gj in zip(primes.keys(), G):
        delta = delta * pow(int(pj), int(gj), N) % N

    st = delta
    for bi, mi in zip(b, m):
        if bi > 0:
            si = sign_oracle(mi)
            st = st * pow(int(si), int(bi), N) % N

    return st

```