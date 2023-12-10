见riscv-v-spec chapter 7
![](rvv_isa/attachments/Pasted%20image%2020231204213650.png)

# 1. Overview

vector loads stores指令分为以下几种
- Unit stride
- Strided
- indexed

在load/store指令中，还可以指定EEW，也就是本次需要读写的element宽度，通过公式可以得到EMUL。

**EMUL = (EEW / SEW) * LMUL**

也就是，根据原SEW/LMUL计算得到本次要操作的个数 vl = VLEN/SEW\*LMUL，即使重新指令EEW，也就是重新指定了每个元素的宽度，但是本次要操作的个数vl是不变的。

指令format如下所示
![](rvv_isa/attachments/Pasted%20image%2020231204213719.png)


|Field|Description|
|---|---|
|rs1[4:0]|specifies x register holding base address|
|rs2[4:0]|specifies x register holding stride|
|vs2[4:0]|specifies v register holding address offsets|
|vs3[4:0]|specifies v register holding store data|
|vd[4:0]|specifies v register destination of load|
|vm|specifies whether vector masking is enabled (0 = mask enabled, 1 = mask disabled)|
|width[2:0]|specifies size of memory elements, and distinguishes from FP scalar|
|mew|extended memory element width. See Vector Load/Store Width Encoding|
|mop[1:0]|specifies memory addressing mode|
|nf[2:0]|specifies the number of elds in each segment, for segment load/stores|
|lumop[4:0]/sumop[4:0]|are additional fields encoding variants of unit-stride instructions|


# 2. Addressing Modes

## 2.1. Unit stride
固定间隔为一个element width的步长，load element0, load element1, load element2
load指令格式如下所示

![](rvv_isa/attachments/Pasted%20image%2020231204214029.png)

- rs1为base address
- 从rs1开始，连续地址访存将数据放入vd寄存器
  
![](rvv_isa/attachments/Pasted%20image%2020231205212009.png)

## 2.2. Strided
![](rvv_isa/attachments/Pasted%20image%2020231204214113.png)

- rs1为base address, rs2为stride的值
- 访问rs1到第0个vd, rs1 + stride到第1个vd, rs2 + 2 * stride到第2个vd ...

spec里面原话说的是
> Vector constant-strided operations access the first memory element at the base effective address, and then access subsequent elements at address increments given by the **byte offset** contained in the x register specified by rs2

![](rvv_isa/attachments/Pasted%20image%2020231205212023.png)
## 2.3. Indexed

![](rvv_isa/attachments/Pasted%20image%2020231204214129.png)

- rs1为base address, vs2存放了每个element对应的addr offset
- 用base addr + offset为每一个元素访存

spec里面原话说的是
> The vector offset operand is treated as a vector of byte-address offsets
> 
> If the vector offset elements are narrower than XLEN, they are **zero-extended to XLEN** before adding to the base effective address. If the vector offset elements are wider than XLEN, **the least-significant XLEN bits** are used in the address calculation.

![](rvv_isa/attachments/Pasted%20image%2020231205212035.png)
# 3. 指令Fields拆解
忽略rs1, rs2, vs2, vs3, vd这些寄存器
## 3.1. Mop

区分了vector访存的类型，其中unit stride和strided访存顺序是随机的，但indexed提供了两种：ordered和unordered。ordered要求按顺序访问memory，
![](rvv_isa/attachments/Pasted%20image%2020231205212046.png)

### A. strided
```ruby
# Vector strided loads and stores
# vd destination, rs1 base address, rs2 byte stride
vlse8.v vd, (rs1), rs2, vm # 8-bit strided load
vlse16.v vd, (rs1), rs2, vm # 16-bit strided load
vlse32.v vd, (rs1), rs2, vm # 32-bit strided load
vlse64.v vd, (rs1), rs2, vm # 64-bit strided load
# vs3 store data, rs1 base address, rs2 byte stride
vsse8.v vs3, (rs1), rs2, vm # 8-bit strided store
vsse16.v vs3, (rs1), rs2, vm # 16-bit strided store
vsse32.v vs3, (rs1), rs2, vm # 32-bit strided store
vsse64.v vs3, (rs1), rs2, vm # 64-bit strided store
```

stride为负以及stride为0都要支持

## 3.2. lumop和sumop -- unit-stride专属

根据lumop和sumop，分成了很多种Unit-stride的访存类型

- The whole register transfers ignore the configured vector length and just load or store the whole vector register, as their name implies. They are meant for OS kernels and debuggers.
- The mask instructions are for generating masks
![](rvv_isa/attachments/Pasted%20image%2020231205212056.png)
### A. 关于whole register load/store

当vtype vl未知的时候，或者vtype vl改起来代价比较大的时候，可以用whole register指令

Whole register指令读写完整的几个寄存器。NFields指定读写多少个vector寄存器，目前只支持1/2/4/8, load可以指定EEW，store的EEW固定是8。

evl = NFields * VLEN / EEW。vstart >= vl就不操作element在这里会变成vstart >= evl

```Ruby
# Format of whole register load and store instructions
vl<NFIELDS>re<EEW>.v vd, (rd)
vs<NFIELDS>r.v vd, (rd)
```

```ruby
# Format of whole register load and store instructions.
vl1r.v v3, (a0) # Pseudoinstruction equal to vl1re8.v

vl1re8.v v3, (a0) # Load v3 with VLEN/8 bytes held at address in a0
vl1re16.v v3, (a0) # Load v3 with VLEN/16 halfwords held at address in a0
vl1re32.v v3, (a0) # Load v3 with VLEN/32 words held at address in a0
vl1re64.v v3, (a0) # Load v3 with VLEN/64 doublewords held at address in a0
vl2r.v v2, (a0) # Pseudoinstruction equal to vl2re8.v v2, (a0)

vl2re8.v v2, (a0) # Load v2-v3 with 2*VLEN/8 bytes from address in a0
vl2re16.v v2, (a0) # Load v2-v3 with 2*VLEN/16 halfwords held at address in a0
vl2re32.v v2, (a0) # Load v2-v3 with 2*VLEN/32 words held at address in a0
vl2re64.v v2, (a0) # Load v2-v3 with 2*VLEN/64 doublewords held at address in a0

vl4r.v v4, (a0) # Pseudoinstruction equal to vl4re8.v

vl4re8.v v4, (a0) # Load v4-v7 with 4*VLEN/8 bytes from address in a0
vl4re16.v v4, (a0)
vl4re32.v v4, (a0)
vl4re64.v v4, (a0)

vl8r.v v8, (a0) # Pseudoinstruction equal to vl8re8.v

vl8re8.v v8, (a0) # Load v8-v15 with 8*VLEN/8 bytes from address in a0

vl8re16.v v8, (a0)
vl8re32.v v8, (a0)
vl8re64.v v8, (a0)

vs1r.v v3, (a1) # Store v3 to address in a1
vs2r.v v2, (a1) # Store v2-v3 to address in a1
vs4r.v v4, (a1) # Store v4-v7 to address in a1
vs8r.v v8, (a1) # Store v8-v15 to address in a1
```

### B. Maksed load/store

用来Load/store mask value，固定EEW为8

evl=ceil(vl/8) (也就是每个element占一个bit), vstart不变

```Ruby
# Vector unit-stride mask load
vlm.v vd, (rs1) # Load byte vector of length ceil(vl/8) # Vector unit-stride mask store
vsm.v vs3, (rs1) # Store byte vector of length ceil(vl/8)
```

### B. 关于unit-stride fault-only-first

只在第一个元素上产生fault
如果element 0出现异常，则修改vl, 进入trap
如果element > 0 出现异常，不进入trap, vl会被修改成出现exception的element的index

如果过程中遇到中断，不改变vl, 而是改变vstart的值

## 3.3. Width

Mem bits代表了在Memory访存时每个element的size, data reg bits代表了register中每个element的size
可以看到unit-stride/strided都是根据指令中的width来决定EEW的
而indexed根据指令中的width决定Index的EEW，访存时element size由SEW决定，也就是source eew和dest eew时不一样的

![](rvv_isa/attachments/Pasted%20image%2020231205212943.png)

## 3.4. Nf

* 对于unit-stride中的whole-register指令，Nf指定了要load/store多少个v寄存器
* 对于其他指令，Nf代表了segment指令以及要写入多少个v寄存器

nf\[2:0]的encoding含义如下所示

| nf[2:0] |     |     | NFIELDS |
| ------- | --- | --- | ------- |
| 0       | 0   | 0   | 1       |
| 0       | 0   | 1   | 2       |
| 0       | 1   | 0   | 3       |
| 0       | 1   | 1   | 4       |
| 1       | 0   | 0   | 5       |
| 1       | 0   | 1   | 6       |
| 1       | 1   | 0   | 7       |
| 1       | 1   | 1   | 8       |


### a. Segment Unit Stride Load/Store
segment指令会将连续的内存地址数据竖着放入v寄存器，如下所示：mem0放入v0 element0, mem1放入v1 element0, mem2放入v2 element0, mem3放入v3 element0, mem4放入v0 element1 ...
效果类似于做了4次strided load

(fault-only-first也是支持的)

![](rvv_isa/attachments/Pasted%20image%2020231210203049.png)

### b. Segment Strided Load/Store
![](rvv_isa/attachments/Pasted%20image%2020231210203419.png)

### c. Segment Index Load/Store

![](rvv_isa/attachments/Pasted%20image%2020231210205423.png)

# 4. Memory Alignment

自由设计
# References

- https://www.francisz.cn/2022/03/23/riscv-vector/
- https://www.andestech.com/wp-content/uploads/Andes-RVV-Webinar-III.pdf