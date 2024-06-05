---
title: "Mapping Condition Code from arm [N, Z, C, V] to x86 [EFLAGS]"
lang: en
ref: mapping-condition-code-from-arm-to-x86
date: 2024-01-02
categories:
  - en
  - gossip
tags:
  - arm
  - assembly
  - x86
---

## CC (Condition Code) in arm [N, Z, C,V]

Operations in arm suffixed with `S` (flag-setting) and `TST`, `CMP` will set `CPSR` (Current Program Status Register). e.g. `ADDS` will change the `CPSR`, and `ADD` won't.

CC in `CPSR` are:

- N (Negative): result is negative(the msb(most significant bit) is 1)
- Z (Zero): result is zero
- C (Carry): unsigned overflow occurs in the computation of the op
- V (Overflow): signed overflow occurs in the computation of the op

## CC in x86 [EFLAGS]

In x86, every arithmetic op will change the CC in `EFLAGS` register.

CC in `EFLAGS`

- CF (Carry Flag): unsigned overflow occurs in the computation of the op
- PF (Parity Flag): the binary expression of the result has even bits of `1`
- ZF (Zero Flag): result is zero
- SF (Sign Flag): result is negative
- OF (Overflow Flag): signed overflow occurs in the computation of the op

## Mapping CC from arm to x86

It seems easy:

```text
{
  N: SF,
  Z: ZF,
  C: CF,
  V: OF
}
```

But if you consult the docs of arm and x86, you may find an issue: ***`C` is mapped to `!CF`***

Taking `JBE`(Jump Below or Equal) for example:

- x86: jump on `CF == 1 || ZF == 1`
- arm: jump on `C == 0 || Z == 1`

It's strange after all since they both have the same description on the truth condition.

### Two views of subtraction

- x86: `a - b -> a - b`
- arm: `a - b -> a + (-b)`

Considering `2 - 1` in 8 bit, i.e, `0x02 - 0x01`：

- In human's way: calculating from the lsb, and conduct the borrow: `0x2 - 0x1 = 0x1` no borrow needed, `0x0 - 0x0`, no borrow needed, and then the `Carry` is `0`. It's the view x86 takes on `CF`.
- In computer's way: subtraction is implemented on addition: add the negative subtracting number: `0x02 - 0x01 = 0x02 + (-0x01) = 0x02 + 0xff = 0x101`, so the `Carry` is `1`. It's the view arm takes on `C`.

***However, human and computer share the same way how the addition is done: `0x01 + 0x02 = 0x03` will set `Carry` to `0` on both arm and x86***

**Therefore, to map CC from arm to x86, one can map `ADDS` to `SUB`, and map  `C` to `!CF`**
