---
layout: post
title: "Bitwise Operations"
categories: [Engineering]
tags: [binary, bitwise]
---

I recently ran into a situation where I had to extract the last 2 bits of a number. This was a perfect use case to use **bitwise operations**, which are super damn fast and make you look like a super-duper developer prodigy in no time! So, what are **bitwise operations**?

## Binary System

Before you can understand bitwise operations, you need to understand what the binary system is. I wrote a whole [blog post]() about this, but here is a quick recap: The binary system represents a number with a sequence of 0s and 1s. Each digit represents the number `2` raised to the position of the digit. For example, the number `177` can be represented in binary with `10110001`, which translates to: `2^7 + 2^5 + 2^4 + 2^0 = 128 + 32 + 16 + 1 = 177`. If you are still unclear about what binary is, please take 5 minutes and read [this post](https://medium.com/@LindaVivah/learn-how-to-read-binary-in-5-minutes-dac1feb991e) before continuing.

## Bitwise Operations

### Shift-right `>>` and Shift-left `<<`

There are `13` bitwise operations, but the `2` most common are `>>` and `<<`. These operations shift the bits of a binary sequence either to the right (`>>`) or to the left (`<<`) by a given number of places. But what does this mean?! Let me explain:

As stated above, the number `177` is `10110001` in binary. If we apply a right-shift to this number, all numbers are moved to the right by X positions. So, a right-shift by 2 would be written `177 >> 2` and refer to:

```
10110001 => 00101100 = 44
```

Digits that are moved further than the `0` position are simply removed, whereas `0`s are added to the left. Every bit shift effectivly halves the number and ignores the fractional part (numbers to the right of the `.`). This is also called an [integer division](http://mathworld.wolfram.com/IntegerDivision.html).

In turn, a left-shift moves all bits to the left, which effectively multiplies the number by `2`. So, the operation `177 << 2` would be:

```
10110001 => 1011000100 = 708
```

Note the added `0`s on the right side. This operation can also be written as `177 * 2 * 2 = 708`.

### Other operations

Next to the shift-operations there are `4` other common operations. Here is a quick summary of what they do:

#### `&` or `Bitwise And`

A bit of the output is `1`, if the bit of the given number **AND** the bit of the input are also `1`, otherwise it is `0`. Example:

```
177 & = 10110001 & = 1000001 = 129
133     10000101
```

#### `|` or `Bitwise Or`

A bit of the output is `1`, if either the bit of the given number **OR** the bit of the input is `1`, otherwise it is `0`. Example:

```
177 | = 10110001 | = 10110101 = 181
133     10000101
```

#### `~` or `Complement`

Returns the _complement_ of a number, basically `1`s become `0`s and `1`s become `0`s. Example:

```
~ 177 = ~ 10110001 = 01001110 = 78
```

#### `^` or `Bitwise Exclusive Or`

This is a tricky one. A bit in the output is equal to the given number bit, **if** that bit is `0` in the input. Otherwise, if the bit in the input is `1`, the output bit becomes the _complementary_ to the bit in the given number. Example:

```
177 ^ = 10110001 ^ = 00110100 = 52
133     10000101
```

The `bitwise exclusive or` basically says: _if the input bit is 1, take the complementary to the original bit. Otherwise, take the original bit._

## Further Resources

[Bitwise Operators in Python](https://wiki.python.org/moin/BitwiseOperators)

[All Bitwise Operators explained](https://code.tutsplus.com/articles/understanding-bitwise-operators--active-11301)
