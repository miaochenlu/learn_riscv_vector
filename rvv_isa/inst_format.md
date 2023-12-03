
vector指令利用了两类已有的opcodes (LOAD-FP & STORE-FP), 增加了一类新的opcode(OP-V)

* vector load指令利用了LOAD-FP的opcode
![](rvv_isa/attachments/Pasted%20image%2020231203191559.png)

* vector store指令利用了STORE-FP的opcode
![](rvv_isa/attachments/Pasted%20image%2020231203191627.png)

* vector arithmetic指令使用了OP-V的opcode
![](rvv_isa/attachments/Pasted%20image%2020231203191719.png)

* vector configuration指令也用了OP-V的opcode
![](rvv_isa/attachments/Pasted%20image%2020231203191755.png)
![](rvv_isa/attachments/Pasted%20image%2020231203191804.png)


# 1. Scalar Operands
scalar operands可以来自
* 立即数
* x registers
* f registers
* vector register的element 0

scalar results可以写入
* x registers
* f registers
* vector register的element 0

任何一个vector register都可以用来承载scalar值

# 2. Vector Operands

在vector operand中存在两种element width和multiplier， 用来决定vector register group的size以及element的粒度
* SEW & LMUL
* EEW & EMUL (effective element width & effective LMUL)

一般来说EEW=SEW, LMUL=EMUL
但也存在不相等的情况，比如widening的arith指令:
* 源寄存器：EEW=SEW, EMUL=LMUL
* 目的寄存器：EEW=2\*SEW, EMUL=2\*LMUL
注意，不相等的情况下要满足操作的length是一样的，也就是EEW/SEW=EMUL/LMUL


目的寄存器和源寄存器可以overlap, 但是为了保证exception发生的时候还能恢复，需要满足以下任一条件
* 目标寄存器EEW = 源寄存器EEW
* 目标寄存器EEW < 源寄存器EEW 但是目标寄存器只能与源寄存器的lowest-numbered part overlap
* 目标寄存器EEW > 源寄存器EEW, EMUL至少是1，overlap只能在目标寄存器的highest-numbered part发生
![](rvv_isa/attachments/Pasted%20image%2020231203202126.png)


另外还有一个限制，EMUL <= 8
EMUL > 8是reserved
# 3. Masking

对于支持mask的指令来说，`inst[25]`是一个vm位，表示是否启用mask操作，v0寄存器存放了mask值。如果vm为1，则不开启mask; 如果vm为0，则根据v0寄存器存的mask进行操作

被masked off的那些element不会产生exceptions, 并且在目标寄存器中，这些位置会根据mask-undisturbed或者mask-agnostic policy选择值进行写入

# 4. Prestart, Active, Inactive, Body, Tail Element

```
for element index x
prestart(x) = (0 <= x < vstart)
body(x) = (vstart <= x < vl)
tail(x) = (vl <= x < max(VLMAX,VLEN/SEW))
mask(x) = unmasked || v0.mask[x] == 1
active(x) = body(x) && mask(x)
inactive(x) = body(x) && !mask(x)
```

如果`vstart>=vl`: dest reg没有elements会更新，并且tail elements也不会用agnostic value更新，x/f register也不会更新


举个例子
假设VLEN=32，LMUL=2，SEW=16，那么这条指令需要操作4个元素。如果vstart设置为1，vl设置为3，那这些概念对应的分别是如图所示。
![](rvv_isa/attachments/Pasted%20image%2020231203204631.png)