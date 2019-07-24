---
layout: post
title: "Binary and Hexadecimal Numeral Systems"
categories: [Engineering]
tags: [binary, hex, hexadecimal]
---

The **binary** numerical system forms the foundation on which our modern computers run. I recently played with Least Significant Bit Stenography and found myself diving into this topic again. So, if you ever wondered how the binary or hexademical system works, this is for you.

## Radix or Base

**Binary** or **Hexadecimal** are both [Numeral Systems](https://en.wikipedia.org/wiki/Numeral_system) that differ by their **base** or **radix**. A **base** is the number of unique digits that a numeral system contains. For example, the ubiquitous _Decimal System_ contains 10 unique digits: `0, 1, ..., 9`, so every number is described with a combination of these 10 digits. Therefore, the _Decimal System_ is a _base 10_ system. The **binary** system only contains 2 unique digits, 0 and 1, and is therefore a **base 2** system. **Hexadecimal** is a **base 16** system, with uses all digits `0 - 9` followed by 6 letters `A, B, ..., F` (or `a, b, ...f`).

## Binary System

The binary system represents a number with a sequence of 0s and 1s, where a digit represents the number 2 raised to the position number of the digit. Let me explain:

If I wanted to represent the number `14` in binary, it would look like `1110`. This stems from the fact that the number `14` can be written as a sum of smaller numbers like this: `14 = 8 + 4 + 2`. Which smaller numbers are summed up is determined by the 0s and 1s in the binary sequence.

For example, the sequence `1110` tells us that the following numbers should be summed up: `8 + 4 + 2`. These numbers are all results of raising the number `2` to a certain exponent, like `2^3 + 2^2 + 2^1`. The exponent is equal to the position of a digit, read from **right to left** and starting at `0`. This table explains this further:

```
Position:  7   6   5   4   3   2   1   0
Exponent:  2^7 2^6 2^5 2^4 2^3 2^2 2^1 2^0
Value:     128 64  32  16  8   4   2   1
```

So, when you want to represent a decimal number as a binary number, you have to break down that number into smaller numbers taken from the table above. At the positions of every number you use, you put a 1 into the binary sequence, otherwise you put a 0. This way you can translate any number from and to binary. All numbers between `0` and `255` can be represented by only 8 binary digits. In the computer field, such digits are called `bits` and a group of 8 `bits` are called 1 `byte`.

## Hexadecimal System

The **hexadecimal** system works similar to the **binary** system, with the difference that digits represent the number `16` raised to the position of the digit **multiplied** with the value of the digit. So, the number `14` in hexadecimal would be simply `E`, since it translates to `14 * 16 ^ 0 = 14`. Whereas the number `177` would be: `B1 = 11 * 16 ^ 1 + 1 * 16 ^ 0 = 176 + 1`.

The hexadecimal system is usually used to present numbers in a very condensed way. If you want want to see how other numbers translate into from binary or hexadecimal, try out the [BinaryHex Converter](https://www.binaryhexconverter.com/).
