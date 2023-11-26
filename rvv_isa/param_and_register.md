
# Parameters

| 名称 | 描述                             | 限制                                                 |
| ---- | -------------------------------- | ------------------------------------------------ |
| ELEN | 一个vector element的最大bits数量 | ELEN >= 8 (must be power of 2)                   |
| VLEN | vector register的大小(in bits)   | VLEN >= ELEN (must be power of 2) & VLEN <= $2^{16}$ |

![](attachments/Pasted%20image%2020231126211839.png)
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


# References
* https://eupilot.eu/wp-content/uploads/2022/11/RISC-V-VectorExtension-1-1.pdf
* https://www.francisz.cn/2022/03/23/riscv-vector/