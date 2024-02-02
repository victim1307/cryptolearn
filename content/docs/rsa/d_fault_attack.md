---
weight: 999
title: "D Fault Attack"
description: ""
icon: "article"
date: "2023-12-02T16:50:45+05:30"
lastmod: "2023-12-02T16:50:45+05:30"
draft: true
toc: true
---


```python
import os
import sys

from math import log2


class PartialInteger:


    def __init__(self):

        self.bit_length = 0
        self.unknowns = 0
        self._components = []

    def add_known(self, value, bit_length):
        self.bit_length += bit_length
        self._components.append((value, bit_length))
        return self

    def add_unknown(self, bit_length):
        self.bit_length += bit_length
        self.unknowns += 1
        self._components.append((None, bit_length))
        return self

    def get_known_lsb(self):
        lsb = 0
        lsb_bit_length = 0
        for value, bit_length in self._components:
            if value is None:
                return lsb, lsb_bit_length

            lsb = lsb + (value << lsb_bit_length)
            lsb_bit_length += bit_length

        return lsb, lsb_bit_length

    def get_known_msb(self):
        msb = 0
        msb_bit_length = 0
        for value, bit_length in reversed(self._components):
            if value is None:
                return msb, msb_bit_length

            msb = (msb << bit_length) + value
            msb_bit_length += bit_length

        return msb, msb_bit_length

    def get_known_middle(self):
        middle = 0
        middle_bit_length = 0
        for value, bit_length in self._components:
            if value is None:
                if middle_bit_length > 0:
                    return middle, middle_bit_length
            else:
                middle = middle + (value << middle_bit_length)
                middle_bit_length += bit_length

        return middle, middle_bit_length

    def get_unknown_lsb(self):
        lsb_bit_length = 0
        for value, bit_length in self._components:
            if value is not None:
                return lsb_bit_length

            lsb_bit_length += bit_length

        return lsb_bit_length

    def get_unknown_msb(self):
        msb_bit_length = 0
        for value, bit_length in reversed(self._components):
            if value is not None:
                return msb_bit_length

            msb_bit_length += bit_length

        return msb_bit_length

    def get_unknown_middle(self):
        middle_bit_length = 0
        for value, bit_length in self._components:
            if value is None:
                if middle_bit_length > 0:
                    return middle_bit_length
            else:
                middle_bit_length += bit_length

        return middle_bit_length

    def matches(self, i):
        shift = 0
        for value, bit_length in self._components:
            if value is not None and (i >> shift) % (2 ** bit_length) != value:
                return False

            shift += bit_length

        return True

    def sub(self, unknowns):
        assert len(unknowns) == self.unknowns
        i = 0
        j = 0
        shift = 0
        for value, bit_length in self._components:
            if value is None:
                # We don't shift here because the unknown could be a symbolic variable
                i += 2 ** shift * unknowns[j]
                j += 1
            else:
                i += value << shift

            shift += bit_length

        return i

    def get_known_and_unknowns(self):
        i_ = 0
        o = []
        l = []
        offset = 0
        for value, bit_length in self._components:
            if value is None:
                o.append(offset)
                l.append(bit_length)
            else:
                i_ += 2 ** offset * value

            offset += bit_length

        return i_, o, l

    def get_unknown_bounds(self):
        return [2 ** bit_length for value, bit_length in self._components if value is None]

    def to_int(self):
        assert self.unknowns == 0
        return self.sub([])

    def to_string_le(self, base, symbols="0123456789abcdefghijklmnopqrstuvwxyz"):
        assert (base & (base - 1)) == 0, "Base must be power of two."
        assert base <= 36
        assert len(symbols) >= base
        bits_per_element = int(log2(base))
        chars = []
        for value, bit_length in self._components:
            assert bit_length % bits_per_element == 0, f"Component with bit length {bit_length} can't be represented by base {base} digits"
            for _ in range(bit_length // bits_per_element):
                if value is None:
                    chars.append('?')
                else:
                    chars.append(symbols[value % base])
                    value //= base

        return chars

    def to_string_be(self, base, symbols="0123456789abcdefghijklmnopqrstuvwxyz"):
        return self.to_string_le(base, symbols)[::-1]

    def to_bits_le(self, symbols="01"):
        assert len(symbols) == 2
        return self.to_string_le(2, symbols)

    def to_bits_be(self, symbols="01"):
        return self.to_bits_le(symbols)[::-1]

    def to_hex_le(self, symbols="0123456789abcdef"):
        assert len(symbols) == 16
        return self.to_string_le(16, symbols)

    def to_hex_be(self, symbols="0123456789abcdef"):
        return self.to_hex_le(symbols)[::-1]

    @staticmethod
    def unknown(bit_length):
        return PartialInteger().add_unknown(bit_length)

    @staticmethod
    def parse_le(digits, base):
        assert (base & (base - 1)) == 0, "Base must be power of two."
        assert base <= 36
        bits_per_element = int(log2(base))
        p = PartialInteger()
        rc_k = 0
        rc_u = 0
        value = 0
        for digit in digits:
            if digit is None or digit == '?':
                if rc_k > 0:
                    p.add_known(value, rc_k * bits_per_element)
                    rc_k = 0
                    value = 0
                rc_u += 1
            else:
                if isinstance(digit, str):
                    digit = int(digit, base)
                assert 0 <= digit < base
                if rc_u > 0:
                    p.add_unknown(rc_u * bits_per_element)
                    rc_u = 0
                value += digit * base ** rc_k
                rc_k += 1

        if rc_k > 0:
            p.add_known(value, rc_k * bits_per_element)

        if rc_u > 0:
            p.add_unknown(rc_u * bits_per_element)

        return p

    @staticmethod
    def parse_be(digits, base):
        return PartialInteger.parse_le(reversed(digits), base)

    @staticmethod
    def from_bits_le(bits):
        return PartialInteger.parse_le(bits, 2)

    @staticmethod
    def from_bits_be(bits):
        return PartialInteger.from_bits_le(reversed(bits))

    @staticmethod
    def from_hex_le(hex):
        return PartialInteger.parse_le(hex, 16)

    @staticmethod
    def from_hex_be(hex):
        return PartialInteger.from_hex_le(reversed(hex))

    @staticmethod
    def from_lsb(bit_length, lsb, lsb_bit_length):
        assert bit_length >= lsb_bit_length
        assert 0 <= lsb <= (2 ** lsb_bit_length)
        return PartialInteger().add_known(lsb, lsb_bit_length).add_unknown(bit_length - lsb_bit_length)

    @staticmethod
    def from_msb(bit_length, msb, msb_bit_length):
        assert bit_length >= msb_bit_length
        assert 0 <= msb < (2 ** msb_bit_length)
        return PartialInteger().add_unknown(bit_length - msb_bit_length).add_known(msb, msb_bit_length)

    @staticmethod
    def from_lsb_and_msb(bit_length, lsb, lsb_bit_length, msb, msb_bit_length):
        assert bit_length >= lsb_bit_length + msb_bit_length
        assert 0 <= lsb < (2 ** lsb_bit_length)
        assert 0 <= msb < (2 ** msb_bit_length)
        middle_bit_length = bit_length - lsb_bit_length - msb_bit_length
        return PartialInteger().add_known(lsb, lsb_bit_length).add_unknown(middle_bit_length).add_known(msb, msb_bit_length)

    @staticmethod
    def from_middle(middle, middle_bit_length, lsb_bit_length, msb_bit_length):
        assert 0 <= middle < (2 ** middle_bit_length)
        return PartialInteger().add_unknown(lsb_bit_length).add_known(middle, middle_bit_length).add_unknown(msb_bit_length)

    @staticmethod
    def lsb_of(i, bit_length, lsb_bit_length):
        lsb = i % (2 ** lsb_bit_length)
        return PartialInteger.from_lsb(bit_length, lsb, lsb_bit_length)

    @staticmethod
    def msb_of(i, bit_length, msb_bit_length):
        msb = i >> (bit_length - msb_bit_length)
        return PartialInteger.from_msb(bit_length, msb, msb_bit_length)

    @staticmethod
    def lsb_and_msb_of(i, bit_length, lsb_bit_length, msb_bit_length):
        lsb = i % (2 ** lsb_bit_length)
        msb = i >> (bit_length - msb_bit_length)
        return PartialInteger.from_lsb_and_msb(bit_length, lsb, lsb_bit_length, msb, msb_bit_length)

    @staticmethod
    def middle_of(i, bit_length, lsb_bit_length, msb_bit_length):
        middle_bit_length = bit_length - lsb_bit_length - msb_bit_length
        middle = (i >> lsb_bit_length) % (2 ** middle_bit_length)
        return PartialInteger.from_middle(middle, middle_bit_length, lsb_bit_length, msb_bit_length)


def attack(n, e, sv, sf):
    d_bits = [None] * n.bit_length()
    m = 2
    mi = {pow(m, 2 ** i, n): i for i in range(n.bit_length())}
    for sfi in sf:
        di0 = pow(sv, -1, n) * sfi % n
        di1 = sv * pow(sfi, -1, n) % n
        if di0 in mi:
            d_bits[mi[di0]] = 0
        if di1 in mi:
            d_bits[mi[di1]] = 1

    return PartialInteger.from_bits_le(d_bits)
    ```