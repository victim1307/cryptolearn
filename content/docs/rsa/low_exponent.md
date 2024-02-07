---
weight: 999
title: "Low Exponent"
description: ""
icon: "article"
date: "2023-12-02T16:52:29+05:30"
lastmod: "2023-12-02T16:52:29+05:30"
draft: true
toc: true
---

### Implementation
{{% alert icon="" context="info" %}}
#### Parameters Required:
1. e: the public exponent
2. c: the ciphertext

***Return: the plaintext***
{{% /alert %}}

```python
from sage.all import ZZ

def attack(e, c):
    return int(ZZ(c).nth_root(e))
```