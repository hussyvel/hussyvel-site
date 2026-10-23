---
title: "Bitwise Operators in Java: What They Are, Where They Came From, and When to Use Them"
date: 2026-10-22
tech: ["java"]
draft: false
---

Bitwise operators work on each individual bit of an integer instead of treating the number as a single value. Computers already store everything in binary, and these operators give programmers direct access to those ones and zeros.

Java has seven of them:

| Operator | Name | Rule | Example (8 bits) |
|---|---|---|---|
| `&` | AND | 1 only if both bits are 1 | `1100 & 1010 = 1000` |
| `\|` | OR | 1 if at least one bit is 1 | `1100 \| 1010 = 1110` |
| `^` | XOR | 1 if the bits are different | `1100 ^ 1010 = 0110` |
| `~` | NOT (complement) | flips every bit | `~00001100 = 11110011` |
| `<<` | left shift | shifts left and fills with 0 (multiplies by 2ⁿ) | `0011 << 2 = 1100` |
| `>>` | signed right shift | shifts right and copies the sign bit | `-16 >> 2 = -4` |
| `>>>` | unsigned right shift | shifts right and fills with 0 | `-16 >>> 2 = 1073741820` |

The Java Language Specification (§15.19) calls `<<`, `>>` and `>>>` the shift operators. The left operand is the value being shifted, and the right operand is the shift distance.

## Where They Came From

The story has three stages: first the mathematics, then the hardware, and finally the programming languages.

**1847–1854: George Boole invents the algebra of logic.** Boole published "The Mathematical Analysis of Logic" in 1847 and "The Laws of Thought" in 1854, starting what we now call Boolean algebra (Britannica). He treated logical operators as pure mathematics. The modern form of the algebra, with union, intersection and complement, was completed later by Jevons, Peirce, Schröder and Huntington (Wikipedia).

**1937: Claude Shannon brings logic into circuits.** At 21, in his master's thesis at MIT, Shannon realized that Boole's true and false map onto the open and closed states of an electrical circuit (Christie's). Switches in series compute AND, switches in parallel compute OR, and a normally closed relay performs NOT (salivity.github.io). Processing logic with electrical switches is the foundation of every modern digital computer (Wikipedia). From then on, processors came with native AND, OR, XOR and shift instructions that the hardware executes directly.

**1960s–1970s: programming languages.** CPL was designed to allow high-level programming without giving up control over individual bits (University of Michigan). Ken Thompson's B already had the `&`, `|`, `^`, `<<` and `>>` operators (flash-gordon.me.uk). In BCPL and B, however, `&` and `|` changed meaning depending on context. Inside an `if` they acted as sequential logical tests; everywhere else they were bit operations (Bell Labs, BCPL manual). While creating C between 1971 and 1973, Dennis Ritchie took a suggestion from Alan Snyder and split the logical operators `&&` and `||` from the bitwise `&` and `|` (Ritchie, "The Development of the C Language"). That is why Java, which inherited C's syntax, has both pairs.

**1995: Java adds `>>>`.** Java has no unsigned integer types, so it defines two right shifts. `>>` is arithmetic and preserves the sign, while `>>>` is logical and fills with zeros (CERT/SEI). `>>>` also removes an ambiguity from C, where the result of `>>` on negative numbers depends on the compiler.

## Why Use Them

- **Performance:** these are the cheapest operations a processor can run, usually in a single cycle.
- **Memory savings:** one `int` holds 32 flags and one `long` holds 64, where you would otherwise need 32 or 64 `boolean` values.
- **Matching binary formats:** network protocols, file formats (PNG, ZIP), hardware and cryptography all define their data bit by bit.
- **Classic algorithms:** hashing, Bloom filters, compression and random number generation all depend on these operators.

## When to Use Them

| Situation | Java example |
|---|---|
| Flags and permissions | `perms \| WRITE`, `perms & ~WRITE`, `(perms & READ) != 0` |
| Packing several values into one number | ARGB color: `(a << 24) \| (r << 16) \| (g << 8) \| b` |
| Reading unsigned bytes | `byte & 0xFF` in streams, sockets and files |
| Protocols and binary formats | TCP/IP headers, control bits |
| Hashing and table indexing | `h ^ (h >>> 16)` and `hash & (n - 1)` in `HashMap` |
| Overflow-safe midpoint | `(low + high) >>> 1` in binary search |
| Fast checks | even/odd `(n & 1) == 0`; power of two `(n & (n - 1)) == 0` |
| Sets of integers | `BitSet`, prime sieves, graphs and games (chess bitboards) |
| Cryptography and checksums | XOR, rotations (`Integer.rotateLeft`), CRC |

## When to Avoid Them

- **Business rules in APIs:** as Joshua Bloch argues in Item 36 of *Effective Java*, prefer `EnumSet` over bit fields packed into an `int`. `EnumSet` is just as fast, and much more readable and type-safe.
- **Micro-optimizing without measuring:** the JIT compiler already turns `x * 8` into `x << 3` when it pays off. Replace arithmetic with shifts only when it makes the code clearer or a profiler shows a real gain.
- **Tricks that confuse readers:** swapping variables with XOR is a curiosity, not a good practice.
- **Known pitfalls:** the shift distance is reduced modulo 32 (or 64 for `long`), `byte` and `short` are promoted to `int` before the operation, and `&` and `|` on `boolean` values do not short-circuit.
