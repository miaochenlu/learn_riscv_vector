
# Parameters

| 名称 | 描述                             | 限制                                                 |
| ---- | -------------------------------- | ------------------------------------------------ |
| ELEN | 一个vector element的最大bits数量 | ELEN >= 8 (must be power of 2)                   |
| VLEN | vector register的大小(in bits)   | VLEN >= ELEN (must be power of 2) & VLEN <= $2^{16}$ |

![](rvv_isa/attachments/Pasted%20image%2020231126211839.png)
# Vector Registers

V扩展增加了
* 32个vector寄存器
	* 名称为v0-v31, 每个寄存器的大小是VLEN bits
* 7个unprivileged CSRs
* mstatus/sstatus中增加了vector的控制

## 1. CSRs

| Address | Privilege | Name   | Description                              |
| ------- | --------- | ------ | ---------------------------------------- |
| 0x008   | URW       | vstart | Vector start position                    |
| 0x009   | URW       | vxsat  | Fixed-Point Saturate Flag                |
| 0x00A   | URW       | vxrm   | Fixed-Point Rounding Mode                |
| 0x00F   | URW       | vcsr   | Vector control and status register       |
| 0xC20   | URO       | vl     | Vector length                            |
| 0xC21   | URO       | vtype  | Vector data type register                |
| 0xC22   | URO       | vlenb  | VLEN/8 (vector register length in bytes) |


## 1.1. mstatus

**VS** (vector context status field): `mstatus[10:9]` 以及 `status[10:9]` (这是和FS(floating-point context status field)共用的)。如果有hypervisor mode的话，`vsstatus[10:9]`也是VS

* 当mstatus.VS是OFF的时候，执行任何vector指令，或访问vector CSRs，会产生illegal instruction异常
* 当mstatus.VS是initial或clean，执行任意会改变vector状态的指令（包括vector CSRs），会将mstatus.VS改为dirty。

## 1.2. vtype
### layout Overview

![](rvv_isa/attachments/Pasted%20image%2020231128192333.png)
vill位于XLEN-1的位置


| Bits     | Name       | Description                                     |
|---|---|---|
| XLEN-1   | vill       | Illegal value if set                            |
| XLEN-2:8 | 0          | Reserved if non-zero                            |
| 7        | vma        | Vector mask agnostic                            |
| 6        | vta        | Vector tail agnostic                            |
| 5:3      | vsew[2:0]  | Selected element width (SEW) setting            |
| 2:0      | vlmul[2:0] | Vector register group multiplier (LMUL) setting |

### vsew (vector selected width)

vector register被划分成VLEN/SEW个elements

|vsew[2:0]| | |SEW|
|-|-|-|---|
|0|0|0|8|
|0|0|1|16|
|0|1|0|32|
|0|1|1|64|
|1|X|X|Reserved|

### vlmul
将连续的几个vector register作为一组来进行操作

$$ LMUL=2^{vlmul[2:0]}$$

|vlmul[2:0] | | |LMUL|# groups|VLMAX|Registers grouped with register n|
|-|-|-|---|---|---|---|
|1|0|0|-|-|-|reserved|
|1|0|1|1/8|32|VLEN/SEW/8|$v_n$ (single register in group)|
|1|1|0|1/4|32|VLEN/SEW/4|$v_n$ (single register in group)|
|1|1|1|1/2|32|VLEN/SEW/2|$v_n$ (single register in group)|
|0|0|0|1|32|VLEN/SEW|$v_n$ (single register in group)|
|0|0|1|2|16|2\*VLEN/SEW|$v_n,v_{n+1}$|
|0|1|0|4|8|4\*VLEN/SEW|$v_n,\cdots,v_{n+3}$|
|0|1|1|8|4|8\*VLEN/SEW|$v_n,\cdots,v_{n+7}$|


### vta & vma

|vta|vma|Tail Elements|Inactive Elements|
|---|---|---|---|
|0|0|undisturbed|undisturbed|
|0|1|undisturbed|agnostic|
|1|0|agnostic|undisturbed|
|1|1|agnostic|agnostic|

in-order可以用undisturbed policy
out-of-order可以用tail-agnostic+mask-agnostic或者tail-agnostic+mask-undisturbed
(这是为什么呢)

### vill

记录之前`vset{i}vl{i}`试图在`vtype`写入不支持的值
当`vill`被set的时候，剩余bits都应该是0
如果`vill`处于set状态，则所有依赖于`vtype`执行的vector指令都会触发illegal instruction exception

note:
* `vset{i}vl{i}` 以及whole-register loads, stores, moves不依赖于`vtype`

## 1.3. vl (vector length register)
`vl`记录一条vector指令更新多少elements的结果
可以被`vset{i}vl{i}`以及fault-only-first load指令修改
同时有vlenb (vector byte length)， vlenb=VLEN/8

## 1.4. vstart (vector start index)

![](rvv_isa/attachments/Pasted%20image%2020231127130905.png)

通常，vstart只在vector指令发生trap的时候被修改，代表vstart的位置发生了trap（不包括illegal-instruction exceptions）, 在该位置resume指令执行。
执行时，vstart位置之前的vector register保持undisturbed,执行后vstart重置为0

## 1.5. vxrm (Vector Fixed-Point Rounding Mode Register)
`vxrm`只有两个读写位`vxrm[1:0]`, 表明浮点指令需要saturate
`vxsat[XLEN-1:2]`都是0

|vxrm[1:0] | |Abbreviation|Rounding Mode|Rounding increment, r|
|---|---|---|---|---|
|0|0|rnu|round-to-nearest-up (add +0.5 LSB)|v[d-1]|
|0|1|rne|round-to-nearest-even|v[d-1] & (v[d-2:0]≠0 \| v[d])|
|1|0|rdn|round-down (truncate)|0|
|1|1|rod|round-to-odd (OR bits into LSB, aka "jam")|!v[d] & v[d-1:0]≠0|

## 1.6. vxsat (Vector Fixed-Point Saturation Flag)
`vxsat`只有一个读写位`vxsat[0]`, 表明浮点指令需要saturate
`vxsat[XLEN-1:1]`都是0

## 1.7. vcsr (Vector Control and Status Register)
`vxrm`和`vxsat`也可以用`vcsr`访问

|Bits|Name|Description|
|---|---|---|
|2:1|vxrm[1:0]|Fixed-point rounding mode|
|0|vxsat|Fixed-point accrued saturation flag|

# References
* https://eupilot.eu/wp-content/uploads/2022/11/RISC-V-VectorExtension-1-1.pdf
* https://www.francisz.cn/2022/03/23/riscv-vector/