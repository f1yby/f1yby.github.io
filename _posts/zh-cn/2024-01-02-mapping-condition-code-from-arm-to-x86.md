---
title: "arm [N, Z, C, V] 映射至 x86 [EFLAGS]"
date: 2024-01-02
categories:
  - zh-cn
  - gossip

tags:
  - arm
  - assembly
  - x86
---

## arm 中的条件码 (Condition Code)

arm 中 后缀带 `S` (flag-setting *设置标志*) 的计算指令会设置 `CPSR` (Current Program Status Register *当前程序状态寄存器*) ，比如 `ADDS`，而对应的`ADD`指令则不会设置条件码。

而 `CPSR` 中涉及条件判断的四位分别是：

- N (Negative): 结果是否为负数即最高位是否为 1
- Z (Zero): 结果是否为 0
- C (Carry): 结果是否出现了无符号溢出
- V (Overflow): 结果是否出现了有符号溢出

## x86 中的条件码

x86 中，涉及计算的指令都会设置 `EFLAGS` 寄存器

`EFLAGS` 中涉及条件判断的分别是：

- CF (Carry Flag): 结果是否出现无符号溢出
- PF (Parity Flag): 结果的二进制表示中 1 的个数是否为偶数
- ZF (Zero Flag): 结果是否为 0
- SF (Sign Flag): 结果是否为负数
- OF (Overflow Flag): 结果是否出现有符号溢出

## arm 映射至 x86

这看起来似乎非常简单:

```json
{
  N: SF,
  Z: ZF,
  C: CF,
  V: OF
}
```

然而如果你分别参阅 arm，intel 文档，就会发现一些小小的问题: arm 的 `C` 对应的似乎应该是 x86 的 `!CF`，也就是 arm 中 `C = true`，对应到 x86 中应该是 `CF = false`。

我们拿 `JBE`(Jump Below or Equal 如果 lhs 小于或者等于 rhs，那么就跳转) 这个指令举例:

- intel: true on `CF == 1 || ZF == 1`
- arm: true on `C == 0 || Z == 1`

### 减法的两种视角

- intel: `a - b -> a - b`
- arm: `a - b -> a + (-b)`

考虑 8 位 下 `2 - 1`, 即 `0x02 - 0x01`：

- 按照人类计算的思维：`0x2 - 0x1 = 0x1`，不需要借位，计算中止，因此 `Carry = 0`，这是 intel 的实现，考虑到 `cmp` 是以 lhs - rhs 的结果设置条件码寄存器，比较 `0x2` 和 `0x1`，得到的结果应该是 `NBE` (`CF == 0`) 与 `Carry = 0` 一致。
- 按照计算机的思维：目前计算机对于负数的实现基于二的补码表示，减法实际上是转为负数再相加，`0x02 - 0x01 = 0x02 + (-0x01) = 0x02 + 0xff = 0x101`，因此 `Carry = 1`。这是 arm 的实现。

***然而，计算机和人类对于加法的理解是一样的: `0x1 + 0x2` 无论在 arm 还是在 x86 下`Carry` 均为 0***

**因此，如果需要将 arm 条件码正确的映射到 x86 上，可以选择将所有 `ADDS` 映射到 `SUBS`，并且将 `C` 映射到 `!CF`**