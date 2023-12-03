这一文档主要介绍configuration-setting指令`vsetvli` `vsetivli` `vsetvl`

```
vsetvli rd, rs1, vtypei    #rd = new vl, rs1 = AVL, vtypei = new vtype setting
vsetivli rd, uimm, vtypei  #rd = new vl, uimm = AVL, vtypei = new vtype setting
vsetvl rd, rs1, rs2        #rd = new vl, rs1 = AVL, rs2 = new vtype setting
```

指令Formats如下
![](rvv_isa/attachments/Pasted%20image%2020231202193738.png)

# 1. Configuration-setting -> vtype
vtype的fields在[param and register](param_and_register.md)中解释过了

| Bits     | Name       | Description                                     |
| -------- | ---------- | ----------------------------------------------- |
| XLEN-1   | vill       | Illegal value if set                            |
| XLEN-2:8 | 0          | Reserved if non-zero                            |
| 7        | vma        | Vector mask agnostic                            |
| 6        | vta        | Vector tail agnostic                            |
| 5:3      | vsew[2:0]  | Selected element width (SEW) setting            |
| 2:0      | vlmul[2:0] | Vector register group multiplier (LMUL) setting |

`vsetvl`指令通过rs2中的值设置`vtype`; `vsetvli`指令通过zimm设置vtype。

zimm也就是vtypei的一些值的名字如下所示

```
Suggested assembler names used for vset{i}vli vtypei immediate

e8 # SEW=8b
e16 # SEW=16b 
e32 # SEW=32b 
e64 # SEW=64b

mf8 # LMUL=1/8 
mf4 # LMUL=1/4 
mf2 # LMUL=1/2
m1 # LMUL=1, assumed if m setting absent
m2 # LMUL=2 
m4 # LMUL=4 
m8 # LMUL=8

Examples:
vsetvli t0, a0, e8 # SEW= 8, LMUL=1
vsetvli t0, a0, e8, m2 # SEW= 8, LMUL=2
vsetvli t0, a0, e32, mf2 # SEW=32, LMUL=1/2
```

- [!] 如果vtype设定的值不合法，vtype中的vill会被set, vtype其余位置以及vl会被设定成0

# 2. AVL编码

新的vl的设置依赖于AVL，在`vsetvli`和`vsetvl`指令中，AVL是通过`rs1`和`rd`的值进行计算的

|rd|rs1|AVL value|Effect on vl|
|---|---|---|---|
|-|!x0|Value in x[rs1]|Normal stripmining|
|!x0|x0|~0|Set vl to VLMAX|
|x0|x0|Value in vl register|Keep existing vl (of course, vtype may change)|


对于`vsetivli`指令，AVL被encode在rs1 field中
# 3. vl设定的限制

1. `vl=0 if AVL=0`
2. `vl>0 if AVL>0`
3. `vl<=VLMAX`
4. `vl<=AVL`
5. `vl=AVL` if `AVL<= VLMAX`
6. `ceil(AVL/2) <= vl <= VLMAX if AVL<(2*VLMAX)`
7. `vl=VLMAX` if `AVL>=(2*VLMAX)`

# 4. 以spike代码为例

set vtype和vl的过程在`set_vl`函数中完成
```cpp
reg_t vectorUnit_t::vectorUnit_t::set_vl(int rd, int rs1, reg_t reqVL, reg_t newType);
```
接收rd寄存器编号, rs1寄存器编号, reqVL(AVL), newType(新的vtype值)，返回vl的值

下面这段代码显示了这三种vsetvl指令使用`set_vl`时传入的参数，并且将rd设置为vl
* `vsetvl`传入rd, rs1的编号，reqVL传入rs1寄存器值，newType为rs2寄存器值
* `vsetvli`传入rd, rs1的编号，reqVL传入rs1寄存器值，newType传入zimm11立即数的值
* `vsetivli`传入rd, 没有rs1寄存器，reqVL传入rs1位置形成的uimm立即数，newType传入zimm10立即数的值
```cpp
// rs1 - regidx 
// RS1 - reg value
// vsetvl.h
WRITE_RD(P.VU.set_vl(insn.rd(), insn.rs1(), RS1, RS2));

// vsetivli.h
WRITE_RD(P.VU.set_vl(insn.rd(), -1. insn.rs1(), insn.v_zimm10()));

// vsetvli.h
WRITE_RD(P.VU.set_vl(insn.rd(), insn.rs1(), RS1, insn.v_zimm11()));
```

接下来是具体的`vset_vl`的实现。代码逻辑主要分成两个部分，一个是set vtype, 一个是set vl，见下方代码注释
```cpp
reg_t vectorUnit_t::vectorUnit_t::set_vl(int rd, int rs1, reg_t reqVL, reg_t newType)
{
  int new_vlmul = 0;
  // set vtype
  if (vtype->read() != newType) {
	//sew, lmul, ta, ma, 以及 vlmax的设置
	vsew = 1 << (extract64(newType, 3, 3) + 3);
	new_vlmul = int8_t(extract64(newType, 0, 3) << 5) >> 5;
	vflmul = new_vlmul >= 0 ? 1 << new_vlmul : 1.0 / (1 << -new_vlmul);
	vlmax = (VLEN/vsew) * vflmul;
	vta = extract64(newType, 6, 1);
	vma = extract64(newType, 7, 1);

	// 如果vtype设定不合法，则set vill bit
	vill = !(vflmul >= 0.125 && vflmul <= 8)
		   || vsew > std::min(vflmul, 1.0f) * ELEN
		   || (newType >> 8) != 0;

    if (vill) {
	  // vtype除了vill都设置为0，vl也设置为0(通过将vlmax设置为0)
	  vlmax = 0;
	  vtype->write_raw(UINT64_MAX << (p->get_xlen() - 1));
    } else {
	  // 合法的话，将newType写入vtype
      vtype->write_raw(newType);
    }
  }

  // set vl
  if (vlmax == 0) {
    vl->write_raw(0);
  } else if (rd == 0 && rs1 == 0) {
	// 如果rd和rs1都是0，保留当前vl的值
	// 如果因为vsetvl指令改变了vlmax, 则也需要判断vlmax和原vl值的大小
    vl->write_raw(std::min(vl->read(), vlmax));
  } else if (rd != 0 && rs1 == 0) {
    // 如果rs1是0，将vl设置为vlmax
    vl->write_raw(vlmax);
  } else if (rs1 != 0) {
    // 如果rs1非0，则将vl设置为min(AVL, vlmax)
    vl->write_raw(std::min(reqVL, vlmax));
  }

  vstart->write_raw(0);
  setvl_count++;
  return vl->read();
}
```

