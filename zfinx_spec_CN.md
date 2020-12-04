# Zfinx 技术规格

## 总览

`Zfinx` 拓展改变了全部已有和未来的拓展，将使用`F`的浮点寄存器替换为`X`寄存器。
因而得名`F-in-X`。
这并不影响Vector(`v`)部分的浮点指令。
此外，`Zfinx`删除了全部的：

1. 浮点加载指令（例如`FLW`）
2. 浮点存储指令（例如`FSW`）
3. 整数 到/从 浮点 寄存器移动指令（例如`FMV.X.W`）

在全部的情形下，都用整形版本来进行替代。

在`Zfinx`内核中，浮点指令的汇编语法变成了指向`X`寄存器。
因而，在 RV32F 内核上，合法的语法：

```assembly
FLW     fa4, 12(sp)        // 加载浮点数据
FMADD.S fa1, fa2, fa3, fa4 // RV32F的浮点算术指令
```

换到`Zfinx`内核上，因为`F`寄存器不存在，因而`FLW`是不支持的指令。

```assembly
LW      a4, 12(sp)     // 加载整形数据
FMADD.S a1, a2, a3, a4 // RV32F Zfinx的浮点算术指令
```

**注意** 上述两个`FMADD.S`间只有汇编语法的差异，编码是相同的。

汇编语法的改变，是为了避免代码移植的bug，因而寄存器必须被更新，而非只是复用非`Zfinx`代码。

`Zfinx`可用于任何使用`F`寄存器的拓展。
寄存器数量对`Zfinx`（`I`或`E`拓展）并无影响，虽然`XLEN`和`FLWN`相对大小对本规格确有影响。

本规格使用 D 表示64位浮点，使用 F 表示32位浮点，使用 Zfh 表示16位浮点。
`Zfinx`的行为只受 数据宽度 影响，因而未来的编码格式是隐式支持的，例如 POSIT的64或32位编码格式。

表1. 受支持的`Zfinx`配置

| 结构              | 评论          |
| ----------------- | ------------- |
| RV32IFD Zfinx     | XLEN  <  FLEN |
| RV32IFD Zfh Zfinx | XLEN  <  FLEN |
| RV32F Zfinx       | XLEN == FLEN  |
| RV32F Zfh Zfinx   | XLEN == FLEN  |
| RV64FD Zfinx      | XLEN == FLEN  |
| RV64FD Zfh Zfinx  | XLEN == FLEN  |
| RV64F Zfinx       | XLEN  >  FLEN |
| RV64F Zfh Zfinx   | XLEN  >  FLEN |

**注意** RV32FD Zfinx和RV32FD Zfh Zfinx需要寄存器对，因而比其他情况更为复杂。

RV128和`Q`拓展并未被本规格所覆盖，但很容易拓展本规格以包含他们。



## 语义差异

浮点算术指令的NaN-boxing行为被修改了以抑制资源检查。浮点的结果总被NaN-box到 `XLEN`位。

*当整数加载并未NaN-box其结果，并且加载少于`XLEN`位时，NaN-boxing检查被删除（例如使用`LW`以加载浮点在RV64内核上）。其它情况因此需要NaN-boxing在软件中，浪费了性能与代码体积。*

在`Zfinx`和`非Zfinx`间，对于浮点指令行为，没有其他语义差异了。但还有一些特殊用例（例如`x0`处理）的差异在本规格后续列出。

## 探索

如果 `Zfinx` 如果被制定了，将有下述**#define**：

```assembly
__riscv_zfinx
```

软件可用此来在`Zfinx`和常规浮点代码间进行选择。

特权指令，可以通过检查if来判断`Zfinx`是否被实现。

1. `mstatus.FS` 被硬连线到 0
2. `misa.F`被重置时为 1，或是可写的

非特权指令，可以通过下列代码来判断`Zfinx`是否被实现。

```assembly
li a0, 0 # 将a0赋为0

#ifdef __riscv_zfinx

fneg.s a0, a0 # 反转a0

#else

fneg.s fa0, fa0 # 反转fa0

#endif
```

如果a0非0，则其为`Zfinx`内核，否则其为`非Zfinx`内核。此二分支编码相同，但是汇编语法对于变量是不同的。

## mstatus.fs

For `Zfinx` cores `mstatus.fs` is hardwired to zero, because all the integer registers already form part of the current context. Note however that `fcsr` needs to be saved and restored. This gives a performance advantage when saving/restoring contexts.

Floating point instructions and `fcsr` accesses do *not* trap if `mstatus.fs`=0. This is different to non-`Zfinx` cores.

## Register pair handling for XLEN < FLEN

For `RV32D`, all D-extension instructions which are implemented with `Zfinx` will access register pairs:

1. The specified register must be even, odd registers will cause an illegal instruction exception
2. Even registers will cause an even/odd pair to be accessed
   1. Accessing Xn will cause the {Xn+1, Xn} pair to be accessed. For example if n = 2
      1. X2 is the least significant half (bits [31:0]) for little endian mode
      2. X3 the most significant half (bits [63:32]) for little endian mode
   2. For big endian mode the register mapping is reversed, so X2 is the most significant half, and X3 is the least significant half.
3. X0 has special handling
   1. Reading {X1, X0} will read all zeros
   2. Writing {X1, X0} will discard the entire result, it will not write to X1

The register pairs are *only* used by the floating point arithmetic instructions. All integer loads and stores will only access `XLEN` bits, not `FLEN`.

Note:

1. **Zp64** from the P-extension specifies consistent register pair handling.
2. Big endian mode is enabled in M-mode if `mstatus.MBE`=1, in S-mode if `mstatus.SBE`=1, or in U-mode if `mstatus.UBE`=1

## x0 register target

If a floating point instruction targets x0 then it will still execute, and will set any required flags in `fcsr`. It will not write to a target register. This matches the non-`Zfinx` behaviour for

```
fcvt.w.s x0, f0
```

If the floating point source is invalid then it will set the `fflags.NV` bit, regardless of whether `Zfinx` is implemented. The target register is not written as it is x0.

If `fcsr.RM` is in an illegal state then floating point instruction behaviour is the same whether the target register is x0 is not, i.e. targetting x0 doesn’t disable any execution side effects.

In the case of `RV32D Zfinx`, register pairs are used. See above for x0 handling.

## NaN-boxing

For `Zfinx` the NaN-boxing is limited to `XLEN` bits, not `FLEN` bits. Therefore a `FADD.S` executed on an `RV64D` core will write a 64-bit value (the MSH will be all 1’s). On an `RV32D Zfinx` core it will write a 32-bit register, i.e. a single X register only. This means there is semantic difference between these code sequences:

```
#ifdef __riscv_zfinx

fadd.s x2, x3, x4 # only write x2 (32-bits), x3 is not written

#else

fadd.s f2, f3, f4 # NaN-box 64-bit f2 register to 64-bits

#endif
```

NaN-box generation is supported by `Zfinx` implementations. NaN-box checking is not supported by scalar floating point instructions. For example for `RV64F`:

```
#ifdef __riscv_zfinx

lw[u] x1, 0(sp)   # load 32-bits into x1 and sign / zero extend upper 32-bits
fadd.s x1, x1, x1 # use x1 but do not check source is Nan-boxed, NaN-box output

#else

flw.s  f1, 0(sp)  # load 32-bits into f1 and NaN-box to 64-bits (set upper 32-bits to 0xFFFFFFFF)
fadd.s f2, f1, f1 # check f1 is NaN-boxed, NaN-box output

#endif
```

Floating point loads are not supported on `Zfinx` cores so x1 is not NaN-boxed in the example above, therefore the `FADD.S` instruction does *not* check the input for NaN-boxing. The result of `FADD.S` *is* NaN-boxed, which means setting the upper half of the output register to all 1’s.

The table shows the effect of writing each possible width of value to the register file for all supported combinations. Note that Verilog syntax is used in the final column.

Table 2. NaN-boxing for supports configurations

| XLEN          | Width of write to Xreg from FP instruction | Value written to Xreg                                        |
| ------------- | ------------------------------------------ | ------------------------------------------------------------ |
| 64            | 16                                         | {48{1’b1}, result[15:0]}                                     |
| 32            | 16                                         | {16{1’b1}, result[15:0]}                                     |
| 64            | 32                                         | {32{1’b1}, result[31:0]}                                     |
| 32            | 32                                         | result[31:0]                                                 |
| 64            | 64                                         | result[63:0]                                                 |
| Little endian |                                            |                                                              |
| 32            | 64                                         | EvenXreg: result[31:0]Odd Xreg: result[63:32]special handling Xreg={0, 1} |
| Big endian    |                                            |                                                              |
| 32            | 64                                         | Odd Xreg: result[31:0]EvenXreg: result[63:32]special handling Xreg={0, 1} |

Therefore, for example, if an `FADD.S` instruction is issued on an `RV64F` core then the upper 32-bits will be set to one in the target integer register, or an `FADD.H` (floating point add half-word) instruction will set the upper 48-bits to one.

## Assembly Syntax and Code Porting

Any references to `F` registers, or removed instructions will cause assembler errors.

For example, the encoding for

```
FMADD.S <1>, <2>, <3>, <4>
```

will disassemble and execute as

```
FMADD.S f1, f2, f3, f4
```

on a non-`Zfinx` core, or

```
FMADD.S x1, x2, x3, x4
```

on a `Zfinx` core.

*We considered allowing pseudo-instructions for the deleted instructions for easier code porting. For example allowing FLW to be a pseudo-instruction for LW, but decided not to. Because the register specifiers must change to integer registers, it makes sense to also remove the use of FLW etc. In this way the user is forced to rewrite their code for a `Zfinx` core, reducing the chance of undiscovered porting bugs. This only affects assembly code, high level language code is unaffected as the compiler will target the correct architecture.*

## Replaced Instructions

All floating point loads, stores and floating point to integer moves are removed on a `Zfinx` core. The following three tables give suggested replacements.

Table 3. replacements for floating point load instructions

| **Instruction**               | **RV32F Zfh Zfinx**                    | **RV32D Zfh Zfinx**            | **RV64F Zfh Zfinx** | **RV64D Zfh Zfinx**            | **RV32F Zfinx** | **RV32D Zfinx** | **RV64F Zfinx** | **RV64D Zfinx** |
| ----------------------------- | -------------------------------------- | ------------------------------ | ------------------- | ------------------------------ | --------------- | --------------- | --------------- | --------------- |
| **loads**                     | **suggested replacement instructions** |                                |                     |                                |                 |                 |                 |                 |
| FLD **f**rd, offset(xrs1)     | *reserved*                             | LW,LW                          | LD                  | *reserved*                     | LW, LW          | LD              |                 |                 |
| FLW **f**rd, offset(xrs1)     | LW                                     | LW[U] and NaN-box in software  | LW                  | LW[U] and NaN-box in software  |                 |                 |                 |                 |
| FLH **f**rd, offset(xrs1)     | LH[U] and NaN-box in software          | *reserved*                     |                     |                                |                 |                 |                 |                 |
| C.FLD **f**rd’, offset(xrs1’) | *reserved*                             | [C.]LW,[C.]LW                  | [C.]LD              | *reserved*                     | [C.]LW,[C.]LW   | [C.]LD          |                 |                 |
| C.FLDSP **f**rd, uimm(x2)     | *reserved*                             | C.LWSP,C.LWSP                  | C.LDSP              | *reserved*                     | C.LWSP,C.LWSP   | C.LDSP          |                 |                 |
| C.FLW **f**rd, offset(xrs1)   | C.LW                                   | C.LW and NaN-box in software   | C.LW                | C.LW and NaN-box in software   |                 |                 |                 |                 |
| C.FLWSP **f**rd, uimm(x2)     | C.LWSP                                 | C.LWSP and NaN-box in software | C.LWSP              | C.LWSP and NaN-box in software |                 |                 |                 |                 |

Table 4. replacements for floating point store instructions

| **Instruction**               | **RV32F Zfh Zfinx**                    | **RV32D Zfh Zfinx** | **RV64F Zfh Zfinx** | **RV64D Zfh Zfinx** | **RV32F Zfinx** | **RV32D Zfinx** | **RV64F Zfinx** | **RV64D Zfinx** |
| ----------------------------- | -------------------------------------- | ------------------- | ------------------- | ------------------- | --------------- | --------------- | --------------- | --------------- |
| **stores**                    | **suggested replacement instructions** |                     |                     |                     |                 |                 |                 |                 |
| FSD **f**rd, offset(xrs1)     | *reserved*                             | SW,SW               | SD                  | *reserved*          | SW, SW          | SD              |                 |                 |
| FSW **f**rd, offset(xrs1)     | SW                                     |                     |                     |                     |                 |                 |                 |                 |
| FSH **f**rd, offset(xrs1)     | SH                                     | *reserved*          |                     |                     |                 |                 |                 |                 |
| C.FSD **f**rd’, offset(xrs1’) | *reserved*                             | [C.]SW,[C.]SW       | [C.]SD              | *reserved*          | [C.]SW,[C.]SW   | [C.]SD          |                 |                 |
| C.FSDSP **f**rd, uimm(x2)     | *reserved*                             | C.SWSP,C.SWSP       | C.SDSP              | *reserved*          | C.SWSP,C.SWSP   | C.SDSP          |                 |                 |
| C.FSW **f**rd, offset(xrs1)   | C.SW                                   |                     |                     |                     |                 |                 |                 |                 |
| C.FSWSP **f**rd, uimm(x2)     | C.SWSP                                 |                     |                     |                     |                 |                 |                 |                 |

Table 5. replacements for floating point move instructions

| **Instruction**       | **RV32F Zfh Zfinx**                    | **RV32D Zfh Zfinx**            | **RV64F Zfh Zfinx** | **RV64D Zfh Zfinx**            | **RV32F Zfinx** | **RV32D Zfinx** | **RV64F Zfinx** | **RV64D Zfinx** |
| --------------------- | -------------------------------------- | ------------------------------ | ------------------- | ------------------------------ | --------------- | --------------- | --------------- | --------------- |
| **moves**             | **suggested replacement instructions** |                                |                     |                                |                 |                 |                 |                 |
| FMV.X.D xrd, **f**rs1 | *reserved*                             | MV,MV                          | *reserved*          | MV                             | *reserved*      | MV,MV           | *reserved*      | MV              |
| FMV.D.X **f**rd, xrs1 | *reserved*                             | MV,MV                          | *reserved*          | MV                             | *reserved*      | MV,MV           | *reserved*      | MV              |
| FMV.X.W xrd, **f**rs1 | MV                                     | MV and sign extend in software | MV                  | MV and sign extend in software |                 |                 |                 |                 |
| FMV.W.X **f**rd, xrs1 | MV                                     | MV and NaN-box in software     | MV                  | MV and NaN-box in software     |                 |                 |                 |                 |
| FMV.X.H xrd, **f**rs1 | MV and sign extend in software         | *reserved*                     |                     |                                |                 |                 |                 |                 |
| FMV.H.X **f**rd, xrs1 | MV and NaN-box in software             | *reserved*                     |                     |                                |                 |                 |                 |                 |

Notes:

1. Where a floating point load loads fewer than `XLEN` bits then software NaN-boxing in software is required to get the same semantics as a non-`Zfinx` core
2. Where a floating point move moves fewer than `XLEN` bits then either sign extension (if the target is an `X` register) or NaN-boxing (if the target is an `F` register) is required in software to get the same semantics

The B-extension is useful for sign extending and NaN-boxing.

To sign-extend using the B-extension:

```
FMV.X.H rd, rs1
```

is replaced by

```
SEXT.H rd, rs1
```

Without the B-extension two instructions are required: shift left 16 places, then arithmetic shift right 16 places.

NaN boxing in software is more involved, as the upper part of the register must be set to 1. The B-extension is also helpful in this case.

```
FMV.H.X a0, a1
```

is replaced by

```
C.ADDI a2, zero, -1
PACK a0, a1, a2
```

## Emulation

A non-`Zfinx` core can run a `Zfinx` binary. M-mode software can do this:

1. Set `mstatus.fs`=0 to cause every floating point instruction to trap
2. When a floating point instruction traps, move the source operands from the X registers to the equivalent F registers (i.e. the same register numbers)
3. Set `mstatus.fs` to be non-zero
4. Execute the original instruction which caused the trap
5. Move the result from the destination `F` register to the `X` register / `X` register pair (For `RV32D`)
6. Set `mstatus.fs`=0
7. `MRET`

There are corner cases around the use of x0 and register pairs for `RV32D`

1. Two 32-bit `X` registers must be transferred to a single 64-bit F register to set up the source operands. This must be done by saving each `X` register to consecutive memory locations, and using a 64-bit floating point load (`FLD` or `C.FLD`) to load the data
2. One 64-bit F register must be transferred to two 32-bit `X` registers to receive the result. This must be done with a 64-bit floating point store (`FSD` or `C.FSD`) and then two 32-bit loads (such as `LW` or `C.LW`).
3. If the source register pair is {x1,x0}, the source data will read as all zeroes. Therefore f0 must be loaded with a 64-bit zero constant from memory.
4. If the destination register pair is {x1,x0} then the full output is discarded, do not transfer the resulting data to the {x1,x0} register pair which would result in the upper half being written to x1

A `Zfinx` core cannot trap on floating point instructions by setting `mstatus.fs`=0, so the reverse emulation isn’t possible. The code must be recompiled (or ported for assembler).

## ABI

For details of the current calling conventions see:

[*https://github.com/riscv/riscv-elf-psabi-doc/blob/master/riscv-elf.md*](https://github.com/riscv/riscv-elf-psabi-doc/blob/master/riscv-elf.md)

The ABI when using `Zfinx` must be one of the the standard integer calling conventions as listed below:

- ilp32e
- ilp32
- lp64

## Floating Point Configurations To Reduce Area

To reduce the area overhead of FPU hardware new configurations will make the `F[N]MADD.*, F[N]MSUB.*` and `FDIV.*, FSQRT.*`` instructions optional in hardware. This then gives the choice of implementing them in software instead by:

1. Taking an illegal instruction trap, and calling the required software routine in the trap handler. This requires that the opcodes are not reallocated and gives binary compatibility between cores with/without hardware support for `F[N]MADD.*, F[N]MSUB.*` and `FDIV.*, FSQRT.*`, but is lower performance than option 2
2. Use the GCC options below so that a software library is used to execute them

This argument already exists for RISCV

```
gcc -mno-fdiv
```

This argument exists for other architectures (e.g. MIPs) but not for RISCV, so it needs to be added

```
gcc -mno-fused-madd
```

To achieve this we break all current and future floating point extensions into three parts: `Zf*base`, `Zfma` and `Zfdiv`. `Zfinx` is orthogonal, and so is an additional modifier to these as described below.

| Options, all start with **Zf** | Meaning                                                      |
| ------------------------------ | ------------------------------------------------------------ |
| Zfhbase                        | Support half precision base instructions                     |
| Zffbase                        | Support single precision base instructions                   |
| Zfdbase                        | Support double precision base instructions                   |
| Zfqbase                        | Support quad precision base instructions                     |
| Zfldstmv                       | Support load,store and integer to/from FP move for all FP extensions |
| Zfma                           | Support multiply-add for all FP extensions                   |
| Zfdiv                          | Support div/sqrt for all FP extensions                       |
| Zfinx                          | Share the integer register file for all FP extensions        |

So the `Zfldstmv`, `Zfma`, `Zfdiv`, `Zfinx` options apply to all floating point extensions, including future ones. This keeps the support regular across the different options.

Therefore `RV32FD Zfh Zfinx` can also be expressed as:

```
rv32_Zfhbase_Zffbase_Zfdbase_Zfma_Zfdiv_Zfinx
```

Also `RV32FD Zfh` can be expressed as:

```
rv32_Zfhbase_Zffbase_Zfdbase_Zfldstmv_Zfma_Zfdiv
```

The options are designed to be additive, none of them remove instructions.

## Rationale, why implement Zfinx?

Small embedded cores which need to implement floating point extensions have some options:

1. Use software emulation of floating point instructions, so don’t implement a hardware FPU which gives minimum core area
   1. The floating point library can be large, and expensive in terms of ROM or flash storage, costing power and energy consumption
   2. The performance of this solution is very low
2. Low core area floating point implementations
   1. Share the integer registers for floating point instructions (`Zfinx`)
      1. Will cause more register spills/fills than having a separate register file, but the effect of this is application dependant
      2. No need for special instructions such as load and stores to access floating point registers, and moves between integer and floating point registers
   2. There are still performance/area tradeoffs to make for the FPU design itself
      1. e.g. pipelined versus iterative
   3. Optionally remove multiply-add instructions to save area in the FPU and a register file read port
   4. Optionally remove divide/square root instructions to to save area in the FPU
3. Dedicated FPU registers, and higher performance FPU implementations use the most area
   1. Separate floating point registers allow fewer register spills/fills, and can also be used for integer code to prevent spilling to memory
   2. There are the same performance/area tradeoffs for the FPU design

`Zfinx` is implemented to allow core area reduction as the area of the `F` register file is significant, for example:

1. `RV32IF Zfinx` saves 1/2 the register file state compared to `RV32IF`
2. `RV32EF Zfinx` saves 2/3 the register file state compared to `RV32EF`

Therefore `Zfinx` should allow for small embedded cores to support floating point with

1. Minimal area increase
2. Similar context switch time as an integer only core
   1. there are no `F` registers to save/restore
3. Reduced code size by removing the floating point library

