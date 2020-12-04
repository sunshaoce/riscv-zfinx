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

RV128和`Q`拓展并未被本规格所覆盖，但很容易就可以拓展本规格以包含他们。
