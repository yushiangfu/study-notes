## Index

- [Introduction](#introduction)
- [Flags](#flags)
- [Instruction *cmp* - compare](#instruction-cmp)
- [Instruction *mov* - move](#instruction-mov)
- [Instruction *stm* - store multiple](#instruction-stm)
- [Instruction *tst* - test](#instruction-tst)
- [Suffix *!*](#suffix-!)
- [Suffix *^*](#suffix-^)
- [Reference](#reference)

## <a name="introduction"></a> Introduction

I'm sick of starting over to learn assembly again and again, and decided to make the note here.

## <a name="flags"></a> Flags

| Flag | Note 1   | Note 2 |
| ---  | ---      | ---    |
| N    | negative |        |
| Z    | zero     |        |
| C    | carry    | rarely used |
| V    | overflow | rarely used |

## <a name="instruction-cmp"></a> Instruction *cmp* - compare

- mp r5, #0

```
tmp = r5 - 0
if (tmp < 0)
    flag_N = 1
else (tmp == 0)
    flag_Z = 1
```

## <a name="instruction-mov"></a> Instruction *mov* - move

- movne   r0, r4

```
if (!flag_Z)
    r0 = r4
```

## <a name="instruction-stm"></a> Instruction *stm* - store multiple

- stmia   sp, {r0 - r12}
    - **ST**ore **M**ultiple, **I**ncrement **A**fter

```
tmp = sp
*tmp = r0; tmp += 4
*tmp = r1; tmp += 4
...
*tmp = r12; tmp += 4
```

<br />

- stmib   sp, {r0 - r12}
    - **ST**ore **M**ultiple, **I**ncrement **B**efore

```
tmp = sp
tmp += 4; *tmp = r0
tmp += 4; *tmp = r1
...
tmp += 4; *tmp = r12
```

<br />

- stmda   sp, {r0 - r12}
    - **ST**ore **M**ultiple, **D**ecrement **A**fter

```
tmp = sp
*tmp = r12; tmp -= 4
*tmp = r11; tmp -= 4
...
*tmp = r10; tmp -= 4
```

<br />

- stmdb   sp, {r0 - r12}
    - **ST**ore **M**ultiple, **D**ecrement **B**efore

```
tmp = sp
tmp -= 4; *tmp = r12
tmp -= 4; *tmp = r11
...
tmp -= 4; *tmp = r0
```

## <a name="instruction-tst"></a> Instruction *tst* - test

- tst r10, #1

```
res = r10 & 1
if (!res)
    flag_Z = 1
```

## <a name="suffix-!"></a> Suffix *!*

write back to register

- stmia   sp!, {r0 - r12}

```
*sp = r0; sp += 4
*sp = r1; sp += 4
...
*sp = r12; sp += 4
```

## <a name="suffix-^"></a> Suffix *^*

- set S bit to load CPSR and PC

(TBD)

- get user bank register when we are in privileged mode
  - stmdb   r8, {sp, lr}^

```
tmp = r8
tmp -= 4; *tmp = lr_usr
tmp -= 4; *tmp = sp_usr
```

## <a name="reference"></a> Reference

- [Load/Store Multiple Instructions](http://www-mdp.eng.cam.ac.uk/web/library/enginfo/mdp_micro/lecture5/lecture5-3-2.html)
- [ARM Assembly Language Tools](https://www.ti.com/lit/ug/spnu118y/spnu118y.pdf?ts=1639549704916)
- [NZCV, Condition Flags](https://developer.arm.com/documentation/ddi0601/2021-06/AArch64-Registers/NZCV--Condition-Flags)


