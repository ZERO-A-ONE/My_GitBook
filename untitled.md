---
description: >-
  本章介绍了数据传送和算术运算的若干必要指令，用大量的篇幅说明了基本寻址模式，如直接寻址、立即寻址和可以用于处理数组的间接寻址。同时，还展示了怎样创建循环和怎样使用一些基本运算符，如
  OFFSET， PTR 和 LENGTHOF。通过本章的学习，将会了解除条件语句之外的汇编语言的基本工作知识。
---

# 4 汇编语言数据相关的运算符、指令和算术运算

## 4.1 汇编语言操作数类型

x86 的指令格式为：

```text
[label:] mnemonic [operands][ ;comment ]
```

指令包含的操作数个数可以是：0 个，1 个，2 个或 3 个。这里，为了清晰起见，省略掉标号和注释：

```text
mnemonic
mnemonic [destination]
mnemonic [destination] , [source]
mnemonic [destination] , [source-1] , [source-2]
```

操作数有 3 种基本类型：

* 立即数——用数字文本表达式
* 寄存器操作数——使用 CPU 内已命名的寄存器
* 内存操作数——引用内存位置

下表说明了标准操作数类型，它使用了简单的操作数符号（32 位模式下），这些符号来自 Intel 手册并进行了改编。本教程将用这些符号来描述每条指令的语法。

| 操作数 | 说明 |
| :--- | :--- |
| reg8 | 8 位通用寄存器：AH、AL、BH、BL、CH、CL、DH、DL |
| reg16 | 16 位通用寄存器：AX、BX、CX、DX、SI、DI、SP、BP |
| reg32 | 32 位通用寄存器：EAX、EEX、ECX、EDX、ESI、EDI、ESP、EBP |
| reg | 通用寄存器 |
| sreg | 16 位段寄存器：CS、DS、SS、ES、FS、GS |
| imm | 8 位、16 位或 32 位立即数 |
| imm8 | 8 位立即数，字节型数值 |
| imm16 | 16 位立即数，字类型数值 |
| imm32 | 32 位立即数，双字型数值 |
| reg/mem8 | 8 位操作数，可以是 8 位通用寄存器或内存字节 |
| reg/mem16 | 16 位立即数，可以是 16 位通用寄存器或内存字 |
| reg/mem32 | 32 位立即数，可以是 32 位通用寄存器或内存双字 |
| mem | 8位、16 位或 32 位内存操作数 |

### 直接内存操作数

变量名引用的是数据段内的偏移量。例如，如下变量 varl 的声明表示，该变量的大小类型为字节，值为十六进制的10：

```text
.data
var1 BYTE 10h
```

可以编写指令，通过内存操作数的地址来解析（查找）这些操作数。假设 var1 的地址偏移量为 10400h。如下指令将该变量的值复制到 AL 寄存器中：

```text
mov al var1
```

指令会被汇编为下面的机器指令：

```text
A0 00010400
```

这条机器指令的第一个字节是操作代码（即操作码（opcode））。剩余部分是 var1 的 32 位十六进制地址。虽然编程时有可能只使用数字地址，但是如同 var1 一样的符号标号会让使用内存更加容易。

另一种表示法。一些程序员更喜欢使用下面这种直接操作数的表达方式，因为，括号意味着解析操作：

```text
mov al, [var1]
```

MASM 允许这种表示法，因此只要愿意就可以在程序中使用。由于多数程序（包括 Microsoft 的程序）印刷时都没有用括号，所以，本书只在出现算术表达式时才使用这种带括号的表示法：

```text
mov al,[var1 + 5]
```

## 4.2 MOV指令：将源操作数复制到目的操作数

MOV 指令将源操作数复制到目的操作数。作为数据传送（data transfer）指令，它几乎用在所有程序中。在它的基本格式中，第一个操作数是目的操作数，第二个操作数是源操作数：

```text
MOV destination,source
```

其中，目的操作数的内容会发生改变，而源操作数不会改变。这种数据从右到左的移动与 [C++](http://c.biancheng.net/cplus/) 或 [Java](http://c.biancheng.net/java/) 中的赋值语句相似：

```text
dest = source;
```

在几乎所有的[汇编语言](http://c.biancheng.net/asm/)指令中，左边的操作数是目标操作数，而右边的操作数是源操作数。只要按照如下原则，MOV 指令使用操作数是非常灵活的。

* 两个操作数必须是同样的大小。
* 两个操作数不能同时为内存操作数。
* 指令指针寄存器（IP、EIP 或 RIP）不能作为目标操作数。

下面是 MOV 指令的标准格式：

```text
MOV reg, reg
MOV mem, reg
MOV reg, mem
MOV mem, imm
MOV reg, imm
```

#### 内存到内存

单条 MOV 指令不能用于直接将数据从一个内存位置传送到另一个内存位置。相反，在将源操作数的值赋给内存操作数之前，必须先将该数值传送给一个寄存器：

```text
.data
var1 WORD ?
var2 WORD ?
.code
mov ax,var1
mov var2,ax
```

> 提示：在将整型常数复制到一个变量或寄存器时，必须考虑该常量需要的最少字节数。

#### 覆盖值

下述代码示例演示了怎样通过使用不同大小的数据来修改同一个 32 位寄存器。当 oneWord 字传送到 AX 时，它就覆盖了 AL 中已有的值。当 oneDword 传送到 EAX 时，它就覆盖了 AX 的值。最后，当 0 被传送到 AX 时，它就覆盖了 EAX 的低半部分。

```text
.data
oneByte BYTE 78h
oneWord WORD 1234h
oneDword DWORD 12345678h
.code
mov eax,0                                 ;EAX=OOOOOOOOh
mov al,oneByte                            ;EAX=00000078h
mov ax,oneWord                            ;EAX=00001234h
mov eax,oneDword                          ;EAX=12345678h
mov ax, 0                                 ;EAX=12340000h
```

## 4.3 MOVZX和MOVSX指令

尽管 MOV 指令不能直接将较小的操作数复制到较大的操作数中，但是程序员可以想办法解决这个问题。假设要将 count（无符号，16 位）传送到 ECX（32 位），可以先将 ECX 设置为 0，然后将 count 传送到 CX：

```text
.data
count WORD 1
.code
mov ecx,0
mov cx,count
```

如果对一个有符号整数 -16 进行同样的操作会发生什么呢？

```text
.data
signedVal SWORD -16      ; FFF0h （-16）
.code
mov ecx,0
mov cx,signedVal         ; ECX = 0000FFF0h（+ 65,52 0）
```

ECX 中的值（+65 520）与 -16 完全不同。但是，如果先将 ECX 设置为 FFFFFFFFh，然后再把 signedVal 复制到 CX，那么最后的值就是完全正确的：

```text
mov ecx,0FFFFFFFFh
mov cx,signedVal    ;ECX = FFFFFFF0h（-16）
```

本例的有效结果是用源操作数的最高位（1）来填充目的操作数 ECX 的高 16 位，这种技术称为符号扩展（sign extension）。当然，不能总是假设源操作数的最高位是 1。幸运的是，Intel 的工程师在设计指令集时已经预见到了这个问题，因此，设置了 MOVZX 和 MOVSX 指令来分别处理无符号整数和有符号整数。

### MOVZX 指令

MOVZX 指令（进行全零扩展并传送）将源操作数复制到目的操作数，并把目的操作数 0 扩展到 16 位或 32 位。这条指令只用于无符号整数，有三种不同的形式：

```text
MOVZX reg32,reg/mem8
MOVZX reg32,reg/mem16
MOVZX reg16,reg/mem8
```

在三种形式中，第一个操作数（寄存器）是目的操作数，第二个操作数是源操作数。注意，源操作数不能是常数。下例将二进制数 1000 1111 进行全零扩展并传送到 AX：

```text
.data
byteVal BYTE 10001111b
.code
movzx ax,byteVal ;AX = 0000000010001111b
```

下图展示了如何将源操作数进行全零扩展，并送入 16 位目的操作数。

![img](http://c.biancheng.net/uploads/allimg/190429/4-1Z4291K50I28.gif)

下面例子的操作数是各种大小的寄存器：

```text
mov bx, 0A69Bh
movzx eax, bx     ;EAX = 0000A69Bh
movzx edx, bl     ;EDX = 0000009Bh
movzx cx, bl     ;CX = 009Bh
```

下面例子的源操作数是内存操作数，执行结果是一样的：

```text
.data
byte1 BYTE  9Bh
word1 WORD 0A69Bh
.code
movzx eax, word1 ;EAX = 0000A69Bh
movzx edx, byte1 ;EDX = 0000009Bh
movzx ex, byte1     ;CX = 009Bh
```

### MOVSX 指令

MOVSX 指令（进行符号扩展并传送）将源操作数内容复制到目的操作数，并把目的操作数符号扩展到 16 位或 32 位。这条指令只用于有符号整数，有三种不同的形式：

```text
MOVSX reg32, reg/mem8
MOVSX reg32, reg/mem16
MOVSX reg16, reg/mem8
```

操作数进行符号扩展时，在目的操作数的全部扩展位上重复（复制）长度较小操作数的最高位。下面的例子是将二进制数 1000 1111b 进行符号扩展并传送到 AX：

```text
.data
byteVal BYTE 10001111b
.code
movsx ax,byteVal      ;AX = 1111111110001111b
```

如下图所示，复制最低 8 位，同时，将源操作数的最高位复制到目的操作数高 8 位的每一位上。

![img](http://c.biancheng.net/uploads/allimg/190429/4-1Z4291K614217.gif)

如果一个十六进制常数的最大有效数字大于 7，那么它的最高位等于 1。如下例所示，传送到 BX 的十六进制数值为 A69B，因此，数字“A”就意味着最高位是 1。（A69B 前面的 0 是一种方便的表示法，用于防止汇编器将常数误认为标识符。）

## 4.4 LAHF和SAHF指令

LAHF（加载状态标志位到 AH）指令将 EFLAGS 寄存器的低字节复制到 AH。被复制的标志位包括：符号标志位、零标志位、辅助进位标志位、奇偶标志位和进位标志位。使用这条指令，可以方便地把标志位副本保管在变量中：

```text
.data
saveflags BYTE ?
.code
lahf                      ;将标志位加载到 AH
mov saveflags, ah         ;用变量保存这些标志位
```

SAHF（保存 AH 内容到状态标志位）指令将 AH 内容复制到 EFLAGS（或 RFLAGS）寄存器低字节。例如，可以检索之前保存到变量中的标志位数值：

```text
mov ah, saveflags  ;加载被保存标志位到 AH
sahf                        ;复制到 FLAGS 寄存器
```

## 4.5 XCHG指令：交换两个操作数内容

XCHG（交换数据）指令交换两个操作数内容。该指令有三种形式：

```text
XCHG reg, reg
XCHG reg, mem
XCHG mem, reg
```

除了 XCHG 指令不使用立即数作操作数之外，XCHG 指令操作数的要求与《[MOV指令](http://c.biancheng.net/view/3493.html)》一节中介绍的 MOV 指令操作数要求是一样的。

在数组排序应用中，XCHG 指令提供了一种简单的方法来交换两个数组元素。下面是几个使用 XCHG 指令的例子。

```text
xchg ax,bx      ;交换 16 位寄存器内容
xchg ah,al      ;交换 8 位寄存器内容
xchg var1,bx    ;交换 16 位内存操作数与 BX 寄存器内容
xchg eax,ebx    ;交换 32 位寄存器内容
```

如果要交换两个内存操作数，则用寄存器作为临时容器，把 MOV 指令与 XCHG 指令一起使用：

```text
mov ax,val1
xchg ax,val2
mov val1,ax
```

## 4.6 直接偏移量操作数

变量名加上一个位移就形成了一个直接 - 偏移量操作数。这样可以访问那些没有显式标记的内存位置。假设现有一个字节数组 arrayB：

```text
arrayB BYTE 10h,20h,30h,40h,50h
```

用该数组作为 MOV 指令的源操作数，则自动传送数组的第一个字节：

```text
mov al,arrayB     ;AL = 10h
```

通过在 arrayB 偏移量上加 1 就可以访问该数组的第二个字节：

```text
mov al,[arrayB+1]      ;AL = 20h
```

如果加 2 就可以访问该数组的第三个字节：

```text
mov al, [arrayB+2]      ;AL = 30h
```

形如 arrayB+1 一样的表达式通过在变量偏移量上加常数来形成所谓的有效地址。有效地址外面的括号表明，通过解析这个表达式就可以得到该内存地址指示的内容。汇编器并不要求在地址表达式之外加括号，但为了清晰明了，建议使用括号。

MASM 没有内置的有效地址范围检查。在下面的例子中，假设数组 arrayB 有 5 个字节，而指令访问的是该数组范围之外的一个内存字节。其结果是一种难以发现的逻辑错误，因此，在检查数组引用时要非常小心：

```text
mov al, [arrayB+20]        ; AL = ??
```

### 字和双字数组

在 16 位的字数组中，每个数组元素的偏移量比前一个多 2 个字节。这就是为什么在下面的例子中，数组 ArrayW 加 2 才能指向该数组的第二个元素：

```text
.data
arrayW WORD 100h,200h,300h
.code
mov ax, arrayW                               ;AX = 100h
mov ax,[arrayW+2]                            ;AX = 200h
```

同样，如果是双字数组，则第一个元素偏移量加 4 才能指向第二个元素：

```text
.data
arrayD DWORD l0000h,20000h
.code
mov eax, arrayD                            ;EAX = 10000h
mov eax,[arrayD+4]                         ;EAX = 20000h
```

## 4.7 数据传送示例

该程序中包含了本章迄今介绍的所有指令，包括：MOV、XCHG、MOVSX 和 MOVZX，展示了字节、字和双字是如何受到它们的影响。同时，程序中还包括了一些直接 - 偏移量操作数。

```text
;数据传送示例
.386
.model flat,stdcall
.stack 4096
ExitProcess PROTO,dwExitCode:DWORD
.data
val1 WORD 1000h
val2 WORD 2000h
arrayB BYTE 10h,20h,30h,40h,50h
arrayW WORD 100h,200h,300h
arrayD DWORD 10000h,20000h

.code
main PROC
;演示 MOVZX 指令
    mov bx,0A69Bh
    movzx eax,bx        ;EAX = 0000A69Bh
    movzx edx,bl        ;EDX = 0000009Bh
    movzx cx,bl         ;CX     = 009Bh
;演示 MOVSX 指令
    mov bx,0A69Bh
    movsx eax,bx        ;EAX = FFFFA69Bh
    movsx edx,bl        ;EDX = FFFFFF9Bh
    mov bl,7Bh
    movsx cx,bl         ;CX = 007Bh
;内存-内存的交换
    mov ax,val1         ;AX = 1000h
    xchg ax val2        ;AX = 2000h,val2 = 1000h
    mov val1,ax         ;val1 = 2000h
;直接-偏移量寻址（字节数组）
    mov al,arrayB        ;AL = 10h
    mov al,[arrayB+1]    ;AL = 20h
    mov al,[arrayB+2]    ;AL = 30h
;直接-偏移量寻址（字数组）
    mov ax,arrayW        ;AX = 100h
    mov ax,[arrayW+2]    ;AX = 200h
;直接-偏移量寻址（双字数组）
    mov eax,arrayD        ;EAX = 10000h
    mov eax,[arrayD+4]    ;EAX = 20000h
    mov eax,[arrayD+4]    ;EAX = 20000h

    INVOKE ExitProcess,0
main ENDP
END main
```

该程序不会产生屏幕输出，但是可以用调试器（debugger）运行。

### 在 Visual Studio 调试器中显示 CPU 标志位

在调试期间显示 CPU 状态标志位时，在 Debug 菜单中选择 Windows 子菜单，再选择 Register。在 Register 窗口，右键选择下拉列表中的 Flags。要想查看这些菜单选项，必须调 试程序。下表是 Register 窗口中用到的标志位符号：

| 标志名称 | 溢岀 | 方向 | 中断 | 符号 | 零 | 辅助进位 | 奇偶 | 进位 |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| 符号 | OV | UP | EI | PL | ZR | AC | PE | CY |

每个标志位有两个值：0（清除）或 1（置位）。示例如下:

```text
OV = 0   UP = 0   EI = 1
PL = 0   ZR = 1   AC = 0
PE = 1   CY = 0
```

调试程序期间，当逐步执行代码时，指令只要修改了标志位的值，则标志位就会显示为红色。这样就可以通过单步执行来了解指令是如何影响标志位的，并可以密切关注这些标志位值的变化。

## 4.8 加法和减法详解

算术运算是[汇编语言](http://c.biancheng.net/asm/)中一个大得令人惊讶的主题！本节重点在于加法和减法的运算。

先从最简单、最有效的指令开始：INC（增加）和 DEC（减少）指令，即加 1 和减 1。然后是能提供更多操作的 ADD、SUB 和 NEG（非）指令。最后，将讨论算术运算指令如何影响 CPU 状态标志位（进位位、符号位、零标志位等）。请记住，汇编语言的细节很重要。

### INC 和 DEC 指令

INC（增加）和DEC（减少）指令分别表示寄存器或内存操作数加 1 和减 1。语法如下所示：

```text
INC reg/mem
DEC reg/mem
```

下面是一些例子：

```text
.data
myWord WORD 1000h
.code
inc myWord                         ; myWord = 1001h
mov bx,myWord
dec bx                             ; BX = 1000h
```

根据目标操作数的值，溢岀标志位、符号标志位、零标志位、辅助进位标志位、进位标志位和奇偶标志位会发生变化。INC 和 DEC 指令不会影响进位标志位（这还真让人吃惊）。

### ADD 指令

ADD 指令将长度相同的源操作数和目的操作数进行相加操作。语法如下：

```text
ADD dest,source
```

在操作中，源操作数不能改变，相加之和存放在目的操作数中。该指令可以使用的操作数与 MOV 指令相同。下面是两个 32 位整数相加的短代码示例：

```text
.data
var1 DWORD 10000h
var2 DWORD 20000h
.code
mov eax,var1    ； EAX = 10000h
add eax,var2    ； EAX = 30000h
```

标志位：进位标志位、零标志位、符号标志位、溢出标志位、辅助进位标志位和奇偶标 志位根据存入目标操作数的数值进行变化。

### SUB 指令

SUB 指令从目的操作数中减去源操作数。该指令对操作数的要求与 ADD 和 MOV 指令相同。指令语法如下：

```text
SUB dest, source
```

下面是两个 32 位整数相减的短代码示例：

```text
.data
var1 DWORD 30000h
var2 DWORD 10000h
.code
mov eax,var1         ;EAX = 30000h
sub eax,var2         ;EAX = 20000h
```

标志位：进位标志位、零标志位、符号标志位、溢出标志位、辅助进位标志位和奇偶标 志位根据存入目标操作数的数值进行变化。

### NEG 指令

NEG（非）指令通过把操作数转换为其二进制补码，将操作数的符号取反。下述操作数可以用于该指令：

```text
NEG reg
NEG mem
```

> 提示：将目标操作数按位取反再加 1，就可以得到这个数的二进制补码。

标志位：进位标志位、零标志位、符号标志位、溢出标志位、辅助进位标志位和奇偶标志位根据存入目标操作数的数值进行变化。

### 执行算术表达式

使用 ADD、SUB 和 NEG 指令，就有办法来执行汇编语言中的算术表达式，包括加法、减法和取反。换句话说，当有下述表达式时，就可以模拟 [C++](http://c.biancheng.net/cplus/) 编译器的行为：

```text
Rval = -Xval + (Yval - Zval);
```

现在来看看，使用如下有符号 32 位变量，汇编语言是如何执行上述表达式的。

```text
Rval SDWORD ?
Xval SDWORD 26
Yval SDWORD 30
Zval SDWORD 40
```

转换表达式时，先计算每个项，最后再将所有项结合起来。首先，对 Xval 的副本进行取反，并存入寄存器：

```text
; first term： -Xval
mov eax,Xval
neg eax                            ; EAX = -26
```

然后，将 Yval 复制到寄存器中，再减去 Zval：

```text
; second term： （Yval - Zval）
mov ebx,Yval
sub ebx,Zval                    ; EBX = -10
```

最后，将两个项（EAX 和 EBX 的内容）相加：

```text
; add the terms and store:
add eax,ebx
mov Rval,eax                    ; -36
```

### 加减法影响的标志位

执行算术运算指令时，常常想要了解结果。它是负数、正数还是零？对目的操作数来说，它是太大，还是太小？这些问题的答案有助于发现计算错误，否则可能会导致程序的错误行为。

检查算术运算结果使用的是 CPU 状态标志位的值，同时，这些值还可以触发条件分支指令，即基本的程序逻辑工具。下面是对状态标志位的简要概述：

* 进位标志位意味着无符号整数溢出。比如，如果指令目的操作数为 8 位，而指令产生的结果大于二进制的 1111 1111，那么进位标志位置 1。
* 溢出标志位意味着有符号整数溢出。比如，指令目的操作数为 16 位，但其产生的负数结果小于十进制的 -32 768，那么溢出标志位置 1。
* 零标志位意味着操作结果为 0。比如，如果两个值相等的操作数相减，则零标志位置 1。
* 符号标志位意味着操作产生的结果为负数。如果目的操作数的最高有效位（MSE）置 1，则符号标志位置 1。
* 奇偶标志位是指，在一条算术或布尔运算指令执行后，立即判断目的操作数最低有效字节中 1 的个数是否为偶数。
* 辅助进位标志位置 1，意味着目的操作数最低有效字节中位 3 有进位。

要在调试时显示 CPU 状态标志位，打开 Register 窗口，右键点击该窗口，并选择 Flags。

#### 1\) 无符号数运算：零标志位、进位标志位和辅助进位标志位

当算术运算结果等于 0 时，零标志位置 1。下面的例子展示了执行 SUB、INC 和 DEC 指令后，目的寄存器和零标志位的状态：

```text
mov ecx,1
sub ecx,1                          ;ECX = 0, ZF = 1
mov eax,0FFFFFFFFh
inc    eax                         ;EAX =    0,    ZF    =    1
inc    eax                         ;EAX =    1,    ZF    =    0
dec    eax                         ;EAX =    0,    ZF    =    1
```

加法和进位标志位，如果将加法和减法分开考虑，那么进位标志位的操作是最容易解释的。两个无符号整数相加时，进位标志位是目的操作数最高有效位进位的副本。直观地说，如果和数超过了目的操作数的存储大小，就可以认为 CF = 1。在下面的例子里，ADD 指令将进位标志位置 1，原因是，相加的和数（100h）超过了 AL 的大小：

```text
mov al,0FFh
add al,1              ; AL = 00, CF = 1
```

下图演示了在 0FFh 上加 1 时，操作数的位是如何变化的。AL 最高有效位的进位复制到进位 标志位。

![&#x5728; 0FFh &#x4E0A;&#x52A0; 1 &#x65F6;&#xFF0C;&#x64CD;&#x4F5C;&#x6570;&#x7684;&#x4F4D;&#x662F;&#x5982;&#x4F55;&#x53D8;&#x5316;&#x7684;](http://c.biancheng.net/uploads/allimg/190430/4-1Z430145AOb.gif)

另一方面，如果 AX 的值为 00FFh，则对其进行加 1 操作后，和数不会超过 16 位，那么进位标志位清 0：

```text
mov ax,00FFh
add ax, 1           ; AX = 0100h, CF = 0
```

但是，如果 AX 的值为 FFFFh，则对其进行加 1 操作后，AX 的高位就会产生进位：

```text
mov ax,0FFFFh
add ax, 1           ; AX = 0000, CF = 1
```

减法和进位标志位，从较小的无符号整数中减去较大的无符号整数时，减法操作就会将进位标志位置 1。下图说明了，操作数为 8 位时，计算（1-2）会出现什么情况。下面是相应的汇编代码：

```text
mov al, 1
sub al,2            ; AL = FFh, CF = 1
```

![&#xFF08;1-2&#xFF09;&#x4F7F;&#x8FDB;&#x4F4D;&#x6807;&#x5FD7;&#x4F4D;&#x7F6E;1](http://c.biancheng.net/uploads/allimg/190430/4-1Z430145PEE.gif)

> 提示：INC 和 DEC 指令不会影响进位标志位。在非零操作数上应用 NEG 指令总是会将进位标志位置 1。

辅助进位标志位，辅助进位（AC）标志位意味着目的操作数位 3 有进位或借位。它主要用于二进制编码的十进制数（BCD）运算，也可以用于其他环境。现在，假设计算（1+0Fh），和数在位 4 上为 1，这是位 3 的进位：

```text
mov al,0Fhadd al, 1           ; AC = 1
```

计算过程如下：

```text
  00001111
+ 00000001
--------------
  00010000
```

奇偶标志位，目的操作数最低有效字节中 1 的个数为偶数时，奇偶（PF）标志位置 1。下例中，ADD 和 SUB 指令修改了 AL 的奇偶性：

```text
mov al,10001100b
add al,00000010b    ; AL = 10001110, PF = 1
sub al,10000000b    ; AL = 00001110, PF = 0
```

执行了 ADD 指令后，AL 的值为 1000 1110 （4 个 0, 4 个 1）, PF=1。执行了 SUB 指令后，AL 的值包含了奇数个 1，因此奇偶标志位等于 0。

#### 2\) 有符号数运算：符号标志位和溢出标志位

符号标志位，有符号数算术操作结果为负数，则符号标志位置 1。下面的例子展示的是小数（4）减去大数（5）：

```text
mov eax, 4
sub eax,5        ; EAX = -1, SP = 1
```

从机器的角度来看，符号标志位是目的操作数高位的副本。下面的例子表示产生了负数结果后，BL 中的十六进制的值：

```text
mov bl,1        ; BL = 01h
sub bl,2        ; BL = FFh（-1）, SF = 1
```

溢出标志位，有符号数算术操作结果与目的操作数相比，如果发生上溢或下溢，则溢出标志位置 1。例如，8 位有符号整数的最大值为 +127，再加 1 就会溢出：

```text
mov al,+127
add al, 1       ; OF = 1
```

同样，最小的负数为-128,再减1就发生下溢。如果目的操作数不能容纳一个有效算 术运算结果，那么溢出标志位置 1：

```text
mov al,-128
sub al,1        ;OF = 1
```

加法测试，两数相加时，有个很简单的方法可以判断是否发生溢出。溢出发生的情况有：

* 两个正数相加，结果为负数
* 两个负数相加，结果为正数

如果两个加数的符号相反，则不会发生溢出。

硬件如何检测溢出，加法或减法操作后，CPU 用一种有趣的机制来检测溢出标志位的状态。计算结果的最高有效位产生的进位与结果的最高位进行 异或操作，异或的结果存入溢岀标志位。如下图所示，两个 8 位二进制数 1000 0000 和 1111 1110 相加，产生进位 CF=1，和数最高位（位 7）= 0，即 1 XOR 0=1，则 OF=1。

![&#x8BBE;&#x7F6E;&#x6EA2;&#x51FA;&#x6807;&#x5FD7;&#x4F4D;](http://c.biancheng.net/uploads/allimg/190430/4-1Z430150145416.gif)

NEG 指令，如果 NEG 指令的目的操作数不能正确存储，则该结果是无效的。例如， AL 中存放的是 -128，对其求反，正确的结果为 +128，但是这个值无法存入 AL。则溢出标志位置 1 就表示 AL 中存放的是一个无效的结果：

```text
mov al,-128         ;AL = 10000000b
neg al              ;AL = 10000000b, OF = 1
```

反之，如果对 +127 求反，结果是有效的，则溢出标志位清 0：

```text
mov al,+127         ;AL = 01111111b
neg al              ;AL = 10000001b, OF = 0
```

CPU 如何知道一个算术运算是有符号的还是无符号的？答案看上去似乎有点愚蠢：它不知道！在算术运算之后，不论标志位是否与之相关，CPU 都会根据一组布尔规则来设置所有的状态标志位。程序员要根据执行操作的类型，来决定哪些标志位需要分析，哪些可以忽略。

### 示例程序（AddSubTest）

AddSubTest 程序利用 ADD、SUB、INC、DEC 和 NEG 指令执行各种算术运算表达式，并展示了相关状态标志位是如何受到影响的：

```text
;加法和减法   （AddSubTest.asm）
.386
.model flat,stdcall
.stack 4096
ExitProcess proto,dwExitCode:dword
.data
Rval    SDWORD ?
Xval    SDWORD 26
Yval    SDWORD 30
Zval    SDWORD 40

.code
main PROC
    ;INC和DEC
    mov    ax,1000h
    inc    ax        ;1001h
    dec    ax        ;1000h
    ;表达式：Rval=-Xval+(Yval-Zval)
    mov    eax,Xval
    neg    eax        ;-26
    mov    ebx,Yval
    sub    ebx,Zval   ;-10
    add    eax,ebx
    mov    Rval,eax;36
    ;零标志位示例
    mov    cx,1
    sub    cx,1       ;ZF = 1
    mov    ax,0FFFFh
    inc    ax         ;ZF = 1
    ;符号标志位示例
    mov    cx,0
    sub    cx,1       ;SF = 1
    mov    ax,7FFFh
    add    ax,2       ;SF = 1
    ;进位标志位示例
    mov    al,0FFh
    add    al,1       ;CF = 1,AL = 00
    ;溢出标志位示例
    mov    al,+127
    add    al,1       ;OF = 1
    mov    al,-128
    sub    al,1       ;OF = 1

    INVOKE ExitProcess,0
main ENDP
END main
```

## 4.9 OFFSET运算符：返回数据标号的偏移量

OFFSET 运算符返回数据标号的偏移量。这个偏移量按字节计算，表示的是该数据标号距离数据段起始地址的距离。如下图所示为数据段内名为 myByte 的变量。

![&#x540D;&#x4E3A;myBye&#x7684;&#x53D8;&#x91CF;](http://c.biancheng.net/uploads/allimg/190430/4-1Z430160124c6.gif)

## OFFSET 示例

在下面的例子中，将用到如下三种类型的变量：

```text
.data
bVal BYTE ?
wVal WORD ?
dVal DWORD ?
dVal2 DWORD ?
```

假设 bVal 在偏移量为 0040 4000（十六进制）的位置，则 OFFSET 运算符返回值如下：

```text
mov esi,OFFSET bVal             ; ESI = 00404000h
mov esi,OFFSET wVal             ; ESI = 00404001h
mov esi,OFFSET dVal             ; ESI = 00404003h
mov esi,OFFSET dVal2            ; ESI = 00404007h
```

OFFSET 也可以应用于直接 - 偏移量操作数。设 myArray 包含 5 个 16 位的字。下面的 MOV 指令首先得到 myArray 的偏移量，然后加 4，再将形成的结果地址直接传送给 ESI。因此，现在可以说 ESI 指向数组中的第 3 个整数。

```text
.data
myArray WORD 1,2,3,4,5
.code
mov esi,OFFSET myArray + 4
```

还可以用一个变量的偏移量来初始化另一个双字变量，从而有效地创建一个指针。如下例所示，pArray 就指向 bigArray 的起始地址：

```text
.data
bigArray DWORD 500 DUP (?)
pArray DWORD bigArray
```

下面的指令把该指针的值加载到 ESI 中，因此，这个 ESI 寄存器就可以指向数组的起始地址：

```text
mov esi,pArray
```

## 4.10 ALIGN伪指令：对齐一个变量

ALIGN 伪指令将一个变量对齐到字节边界、字边界、双字边界或段落边界。

语法如下：

```text
ALIGN bound
```

Bound 可取值有：1、2、4、8、16。当取值为 1 时，则下一个变量对齐于 1 字节边界（默认情况）。当取值为 2 时，则下一个变量对齐于偶数地址。当取值为 4 时，则下一个变量地址为 4 的倍数。当取值为 16 时，则下一个变量地址为 16 的倍数，即一个段落的边界。

为了满足对齐要求，汇编器会在变量前插入一个或多个空字节。为什么要对齐数据？因为，对于存储于偶地址和奇地址的数据来说，CPU 处理偶地址数据的速度要快得多。

下述例子中，bVal 处于任意位置，但其偏移量为 0040 4000。在 wVal 之前插入 ALIGN 2 伪指令，这使得 wVal 对齐于偶地址偏移量：

```text
bVal BYTE ?           ;00404000h
ALIGN 2 
wVal WORD ?           ;00404002h
bVal2 BYTE ?          ;00404004h
ALIGN 4 
dVal DWORD ?          ;00404008h
dVal2 DWORD ?         ;0040400Ch
```

请注意，dVal 的偏移量原本是 0040 4005，但是 ALIGN 4 伪指令使它的偏移量成为 0040 4008。

## 4. 11 PTR运算符：重写操作数的大小类型

PTR 运算符可以用来重写一个已经被声明过的操作数的大小类型。只要试图用不同于汇编器设定的大小属性来访问操作数，那么这个运算符就是必需的。

例如，假设想要将一个双字变量 myDouble 的低 16 位传送给 AXO 由于操作数大小不匹配，因此，汇编器不会允许这种操作：

```text
.data
myDouble DWORD 12345678h
.code
mov ax,myDouble
```

但是，使用 WORD PTR 运算符就能将低位字（5678h）送入 AX：

```text
mov ax,WORD PTR myDouble
```

为什么送入 AX 的不是 1234h ？因为，x86 处理器采用的是小端存储格式，即低位字节存放于变量的起始地址。如下图所示，用三种方式表示 myDouble 的内存布局：第一列是一个双字，第二列是两个字（5678h、1234h），第三列是四个字节（78h、56h、34h、12h）。

![myDouble&#x7684;&#x5185;&#x5B58;&#x5206;&#x5E03;](http://c.biancheng.net/uploads/allimg/190430/4-1Z4301AA1B3.gif)

不论该变量是如何定义的，都可以用三种方法中的任何一种来访问内存。比如，如果 myDouble 的偏移量为 0000，则以这个偏移量为首地址存放的 16 位值是 5678h。同时也可以检索到 1234h，其字地址为 myDouble+2，指令如下：

```text
mov ax,WORD PTR [myDouble+2]     ; 1234h
```

同样，用 BYTE PTR 运算符能够把 myDouble 的单个字节传送到 BL：

```text
mov b1,BYTE PTR myDouble       ; 78h
```

注意，PTR 必须与一个标准汇编数据类型一起使用，这些类型包括：BYTE、SEYTE、WORD、SWORD、DWORD、SDWORD、FWORD、QWORD 或 TBYTE。

### 将较小的值送入较大的目的操作数

程序可能需要将两个较小的值送入一个较大的目的操作数。如下例所示，第一个字复制到 EAX 的低半部分，第二个字复制到高半部分。而 DWORD PTR 运算符能实现这种操作：

```text
.data
wordList WORD 5678h,1234h
.code
mov eax, DWORD PTR wordList      ; EAX = 12345678h
```

## 4.12 TYPE运算符：返回变量的大小

TYPE 运算符返回变量单个元素的大小，这个大小是以字节为单位计算的。比如，TYPE 为字节，返回值是 1；TYPE 为字，返回值是 2；TYPE 为双字，返回值是 4；TYPE 为四字，返回值是 8。示例如下：

```text
.data
var1 BYTE ?
var2 WORD ?
var3 DWORD ?
var4 QWORD ?
```

下表是每个 TYPE 表达式的值。

| 表达式 | 值 | 表达式 | 值 |
| :--- | :--- | :--- | :--- |
| TYPE var1 | 1 | TYPE var3 | 4 |
| TYPE var2 | 2 | TYPE var4 | 8 |

## 4.13 LENGTHOF运算符：计算数组中元素的个数

LENGTHOF 运算符计算数组中元素的个数，元素个数是由数组标号同一行出现的数值来定义的。示例如下：

```text
.data 
byte1 BYTE  10,20,30
array1 WORD  30 DUP (?),0,0
array2 WORD 5 DUP(3 DUP(?))
array3 DWORD 1,2,3,4
digitStr  BYTE "12345678",0
```

如果数组定义中出现了嵌套的 DUP 运算符，那么 LENGTHOF 返回的是两个数值的乘积。下表列出了每个 LENGTHOF 表达式返回的数值。

| 表达式 | 值 | 表达式 | 值 |
| :--- | :--- | :--- | :--- |
| LENGTHOF byte1 | 3 | LENGTHOF array3 | 4 |
| LENGTHOF array1 | 30+2 | LENGTHOF digitStr | 9 |
| LENGTHOF array2 | 5\*3 |  |  |

如果数组定义占据了多个程序行，那么 LENGTHOF 只针对第一行定义的数据。比如有如下数据，则 LENGTHOF myArray 返回值为 5 :

```text
myArray BYTE 10,20,30,40,50
        BYTE 60,70,80,90,100
```

另外，也可以在第一行结尾处用逗号，并在下一行继续进行数组初始化。若有如下数据定义， LENGTHOF myArray 返回值为 10：

```text
myArray BYTE 10,20,30,40,50,
             60,70,80,90,100
```

## 4.14 LABEL伪指令

LABEL 伪指令可以插入一个标号，并定义它的大小属性，但是不为这个标号分配存储空间。LABEL 中可以使用所有的标准大小属性，如 BYTE、WORD、DWORD、QWORD 或 TBYTE。

LABEL 常见的用法是，为数据段中定义的下一个变量提供不同的名称和大小属性。如下例所示，在变量 val32 前定义了一个变量，名称为 val16 属性为 WORD：

```text
.data
val16 LABEL WORD
val32 DWORD 12345678h
.code
mov ax,val16          ; AX = 5678h
mov dx,[val16+2]      ; DX = 1234h
```

val16 与 val32 共享同一个内存位置。LABEL 伪指令自身不分配内存。

有时需要用两个较小的整数组成一个较大的整数，如下例所示，两个 16 位变量组成一个 32 位变量并加载到 EAX 中：

```text
.data
LongValue LABEL DWORD
val1 WORD 5678h
val2 WORD 1234h
.code
mov eax,LongValue         ; EAX = 12345678h
```

## 4.15 间接寻址

直接寻址很少用于数组处理，因为，用常数偏移量来寻址多个数组元素时，直接寻址不实用。反之，会用寄存器作为指针（称为间接寻址）并控制该寄存器的值。如果一个操作数使用的是间接寻址，就称之为间接操作数。

### 间接操作数

#### 保护模式

任何一个 32 位通用寄存器（EAX、EBX、ECX、EDX、ESI、EDI、EBP 和 ESP）加上括号就能构成一个间接操作数。

寄存器中存放的是数据的地址。示例如下，ESI 存放的是 byteVal 的偏移量，MOV 指令使用间接操作数作为源操作数，解析 ESI 中的偏移量，并将一个字节送入 AL：

```text
.data
byteVal BYTE 10h
.code
mov esi,OFFSET byteVal
mov al,[esi]                              ; AL = 10h
```

如果目的操作数也是间接操作数，那么新值将存入由寄存器提供地址的内存位置。在下面的例子中，BL 寄存器的内容复制到 ESI 寻址的内存地址中：

```text
mov [esi],bl
```

#### PTR 与间接操作数一起使用

一个操作数的大小可能无法从指令中直接看出来。下面的指令会导致汇编器产生“operand must have size（操作数必须有大小）”的错误信息：

```text
inc [esi]    ;错误：operand must have size
```

汇编器不知道 ESI 指针的类型是字节、字、双字，还是其他的类型。而 PTR 运算符则可以确定操作数的大小类型：

```text
inc BYTE PTR [esi]
```

### 数组

间接操作数是步进遍历数组的理想工具。下例中，arrayB 有 3 个字节，随着 ESI 不断加 1，它就能顺序指向每一个字节：

```text
.data
arrayB BYTE 10h,20h,30h
.code
mov esi,OFFSET arrayB
mov alz [esi]                        ;AL = lOh
inc esi
mov al, [esi]                        ;AL = 20h
inc esi
mov al, [esi]                        ;AL = 30h
```

如果数组是 16 位整数类型，则 ESI 加 2 就可以顺序寻址每个数组元素：

```text
.data
arrayW WORD 1000h,2000h,3000h
.code
mov esi,OFFSET arrayW
mov ax,[esi]                         ; AX = 1000h
add esi, 2
mov ax,[esi]                         ; AX = 2000h
add esi, 2
mov axz [esi]                        ; AX = 3000h
```

假设 arrayW 的偏移量为 10200h，下图展示的是 ESI 初始值相对数组数据的位置。

![img](http://c.biancheng.net/uploads/allimg/190505/4-1Z505111230G8.gif)

示例：32 位整数相加下面的代码示例实现的是 3 个双字相加。由于双字是 4 个字节的，因此，ESI 要加 4 才能顺序指向每个数组数值：

```text
.data
arrayD DWORD 10000h,20000h,30000h
.code
mov esi,OFFSET arrayD
mov eax, [esi]                  ;（第一个数）
add esi, 4
add eax, [esi]                   ;（第二个数）
add esi, 4
add eax, [esi]                   ;（第三个数）
```

假设 arrayD 的偏移量为 10200h。下图展示的是 ESI 初始值相对数组数据的位置：

![img](http://c.biancheng.net/uploads/allimg/190505/4-1Z505111300619.gif)

### 变址操作数

变址操作数是指，在寄存器上加上常数产生一个有效地址。每个 32 位通用寄存器都可以用作变址寄存器。MASM 可以用不同的符号来表示变址操作数（括号是表示符号的一部分）：

```text
constant [reg]
[constant + reg]
```

第一种形式是变量名加上寄存器。变量名由汇编器转换为常数，代表的是该变量的偏移量。下面给岀的是两种符号形式的例子：

| arrayB\[esi\] | \[arrayB + esi\] |
| :--- | :--- |
| arrayD\[ebx\] | \[arrayD + ebx\] |

变址操作数非常适合于数组处理。在访问第一个数组元素之前，变址寄存器需要初始化为 0：

```text
.data
arrayB BYTE 10h,20h,30h
.code
mov esi, 0
mov al, arrayB[esi]                ; AL = 10h
```

最后一条语句将 ESI 和 arrayB 的偏移量相加，表达式 \[arrayB+ESI\] 产生的地址被解析，并将相应内存字节的内容复制到AL。

增加位移量变址寻址的第二种形式是寄存器加上常数偏移量。变址寄存器保存数组或结构的基址，常数标识各个数组元素的偏移量。下例展示了在一个 16 位字数组中如何使用这种形式：

```text
.data
arrayW WORD 1000h,2000h,3000h
.code
mov esi,OFFSET arrayW
mov ax, [esi]                   ;AX = 1000h
mov ax, [esi+2]                 ;AX = 2000h
mov ax, [esi+4]                 ;AX = 3000h
```

#### 使用 16 位寄存器

在实地址模式中，一般用 16 位寄存器作为变址操作数。在这种情况下，能被使用的寄存器只有 SI、DI、BX 和 BP：

```text
mov al,arrayB[si]mov ax,arrayW[di]mov eax,arrayD[bx]
```

如果有间接操作数，则要避免使用 BP 寄存器，除非是寻址堆栈数据。

#### 变址操作数中的比例因子

在计算偏移量时，变址操作数必须考虑每个数组元素的大小。比如下例中的双字数组，下标（3 ）要乘以 4（一个双字的大小）才能生成内容为 400h 的数组元素的偏移量：

```text
.dataarrayD DWORD 100h, 200h, 300h, 400h.codemov esi , 3 * TYPE arrayD                            ; arrayD [ 3 ]的偏移量mov eax,arrayD[esi]                                  ; EAX = 400h
```

Intel 设计师希望能让编译器编写者的常用操作更容易，因此，他们提供了一种计算偏移量的方法，即使用比例因子。比例因子是数组元素的大小（字 = 2，双字 =4，四字 =8）。现在对刚才的例子进行修改，将数组下标（3）送入 ESI，然后 ESI 乘以双字的比例因子（4）：

```text
.dataarrayD DWORD 1,2,3,4.code mov esi, 3                               ;下标mov eax,arrayD[esi*4]                    ;EAX = 4
```

TYPE 运算符能让变址更加灵活，它可以让 arrayD 在以后重新定义为别的类型：

```text
mov esi, 3                         ;下标 mov eax,arrayD[esi*TYPE arrayD]    ;EAX = 4
```

### 指针

如果一个变量包含另一个变量的地址，则该变量称为指针。指针是控制数组和[数据结构](http://c.biancheng.net/data_structure/)的重要工具，因为，它包含的地址在运行时是可以修改的。比如，可以使用系统调用来分配（保留）一个内存块，再把这个块的地址保存在一个变量中。

指针的大小受处理器当前模式（32位或64位）的影响。下例为 32 位的代码，ptrB 包含了 arrayB 的偏移量：

```text
.data
arrayB byte 10h,20h,30h,40h
ptrB dword arrayB
```

还可以用 OFFSET 运算符来定义 ptrB，从而使得这种关系更加明确：

```text
ptrB dword OFFSET arrayB
```

32 位模式程序使用的是近指针，因此，它们保存在双字变量中。这里有两个例子：ptrB 包含 arrayB 的偏移量，ptrW 包含 arrayW 的偏移量：

```text
arrayB BYTE 10h,20h,30h,40h
arrayW WORD 1000h,2000h,3000h
ptrB    DWORD arrayB
ptrW    DWORD arrayW
```

同样，也还可以用 OFFSET 运算符使这种关系更加明确：

```text
ptrB DWORD OFFSET arrayB
ptrW DWORD OFFSET arrayW
```

高级语言刻意隐藏了指针的物理细节，这是因为机器结构不同，指针的实现也有差异。[汇编语言](http://c.biancheng.net/asm/)中，由于面对的是单一实现，因此是在物理层上检查和使用指针。这样有助于消除围绕着指针的一些神秘感。

#### 使用 TYPEDEF 运算符

TYPEDEF 运算符可以创建用户定义类型，这些类型包含了定义变量时内置类型的所有状态。它是创建指针变量的理想工具。比如，下面声明创建的一个新数据类型 PBYTE 就是一个字节指针：

```text
PBYTE TYPEDEF PTR BYTE
```

这个声明通常放在靠近程序开始的地方，在数据段之前。然后，变量就可以用 PBYTE 来定义：

```text
.data
arrayB BYTE 10h,20h,30h,40h
ptr1 PBYTE ?                              ;未初始化
ptr2 PBYTE arrayB                     ;指向一个数组
```

#### 示例程序：Pointers

下面的程序（pointers.asm）用 TYPEDEF 创建了 3 个指针类型（PBYTE、PWORD、PDWORD）。此外，程序还创建了几个指针，分配了一些数组偏移量，并解析了这些指针：

```text
TITLE Pointers            (Pointers.asm)
.386
.model flat,stdcall
.stack 4096
ExitProcess proto,dwExitCode:dword
;创建用户定义类型
PBYTE TYPEDEF PTR BYTE                ;字节指针
PWORD TYPEDEF PTR WORD                ;字指针
PDWORD TYPEDEF PTR DWORD            ;双字指针

.data
arrayB BYTE 10h,20h,30h
arrayW WORD 1,2,3
arrayD DWORD 4,5,6
;创建几个指针变量
ptr1 PBYTE arrayB
ptr2 PWORD arrayW
ptr3 PDWORD arrayD

.code
main PROC
;使用指针访问数据
    mov esi,ptr1
    mov al,[esi]                    ;10h
    mov esi,ptr2
    mov ax,[esi]                    ;1
    mov esi,ptr3
    mov eax,[esi]                    ;4
    invoke ExitProcess,0
main ENDP
END main
```

## 4.16 JMP和LOOP（转移）指令

默认情况下，CPU 是顺序加载并执行程序。但是，当前指令有可能是有条件的，也就是说，它按照 CPU 状态标志（零标志、符号标志、进位标志等）的值，把控制转向程序中的新位置。

[汇编语言](http://c.biancheng.net/asm/)程序使用条件指令来实现如 IF 语句的高级语句与循环。每条条件指令都包含了一个可能的转向不同内存地址的转移（跳转）。控制转移，或分支，是一种改变语句执行顺序的方法，它有两种基本类型：

* 无条件转移：无论什么情况都会转移到新地址。新地址加载到指令指针寄存器，使得程序在新地址进行执行。JMP 指令实现这种转移。
* 条件转移：满足某种条件，则程序出现分支。各种条件转移指令还可以组合起来，形成条件逻辑结构。CPU 基于 ECX 和标志寄存器的内容来解释真 / 假条件。

### JMP 指令

JMP 指令无条件跳转到目标地址，该地址用代码标号来标识，并被汇编器转换为偏移 量。语法如下所示：

```text
JMP destination
```

当 CPU 执行一个无条件转移时，目标地址的偏移量被送入指令指针寄存器，从而导致迈从新地址开始继续执行。

JMP 指令提供了一种简单的方法来创建循环，即跳转到循环开始时的标号：

```text
top:
    .
    .
    jmp top     ;不断地循环
```

JMP 是无条件的，因此循环会无休止地进行下去，除非找到其他方法退岀循环。

### LOOP 指令

LOOP 指令，正式称为按照 ECX 计数器循环，将程序块重复特定次数。ECX 自动成为计数器，每循环一次计数值减 1。语法如下所示：

```text
LOOP destination
```

循环目标必须距离当前地址计数器 -128 到 +127 字节范围内。LOOP 指令的执行有两个步骤：

* 第一步，ECX 减 1。
* 第二步，将 ECX 与 0 比较。

如果 ECX 不等于 0，则跳转到由目标给岀的标号。否则，如果 ECX 等于 0，则不发生跳转，并将控制传递到循环后面的指令。

实地址模式中，CX 是 LOOP 指令的默认循环计数器。同时，LOOPD 指令使用 ECX 为循环计数器，LOOPW 指令使用 CX 为循环计数器。

下面的例子中，每次循环是将 AX 加 1。当循环结束时，AX=5, ECX=0：

```text
    mov ax,0
    mov ecx,5
L1:
    inc ax
    loop L1
```

一个常见的编程错误是，在循环开始之前，无意间将 ECX 初始化为 0。如果执行了这个操作，LOOP 指令将 ECX 减 1 后，其值就为 FFFFFFFFh，那么循环次数就变成了 4 294 967 296！如果计数器是 CX （实地址模式下），那么循环次数就为 65 536。

有时，可能会创建一个太大的循环，以至于超过了 LOOP 指令允许的相对跳转范围。下面给出是 MASM 产生的一条错误信息，其原因就是 LOOP 指令的跳转目标太远了：

```text
error A2075: jump destination too far : by 14 byte(s)
```

基本上，在一个循环中不用显式的修改 ECX，否则，LOOP 指令可能无法正常工作。下例中，每次循环 ECX 加 1。这样 ECX 的值永远不能到 0，因此循环也永远不会停止：

```text
top:
    .
    .
    inc ecx
    loop top
```

如果需要在循环中修改 ECX，可以在循环开始时，将 ECX 的值保存在变量中，再在 LOOP 指令之前恢复被保存的计数值：

```text
.data
count DWORD ?
.code
    mov ecx, 100        ;设置循环计数值
top:
    mov count, ecx      ;保存计数值
    .
    mov ecx, 20         ;修改 ECX
    .
    mov ecx, count      ;恢复计数值
    loop top
```

#### 循环嵌套

当在一个循环中再创建一个循环时，就必须特别考虑外层循环的计数器 ECX，可以将它保存在一个变量中：

```text
.data
count DWORD ?
.code
    mov ecx, 100    ;设置外层循环计数值
L1:
    mov count, ecx  ;保存外层循环计数值
    mov ecx, 20     ;设置内层循环计数值
L2 :
    loop L2         ;重复内层循环
    mov ecx, count  ;恢复外层循环计数值
    loop L1         ;重复外层循环
```

> 提示：作为一般规则，多于两重的循环嵌套难以编写。如果使用的算法需要多重循环，则将一些内层循环用子程序来实现。

### 在 Visual Studio 调试器中显示数组

在调试期间，如果想要显示数组的内容，步骤如下：选择 Debug 菜单 -&gt; 选择 Windows -&gt; 选择 Memory -&gt; 选择Memory 1。则出现内存窗口，可以用鼠标拖动并停靠在 Visual Studio 工作区的任何一边。还可以右键点击该窗口的标题栏，表明要这个窗口浮动在编辑窗口之上。

在内存窗口上端的 Address 栏里， 键入 & 符号和数组名称，然后点击 Enter。比如，&myArray 就是一个有效的地址表达式。内存窗口将显示从这个数组地址开始的内存块，如下图所示。

![&#x4F7F;&#x7528;&#x8C03;&#x8BD5;&#x5185;&#x5B58;&#x7A97;&#x53E3;&#x663E;&#x793A;&#x6570;&#x7EC4;](http://c.biancheng.net/uploads/allimg/190505/4-1Z5051329291H.gif) 如果数组的值是双字，可以在内存窗口中，点击右键并在弹出菜单里选择 4-byte integer。还有不同的格式可供选择，包括 Hexadecimal Display,Signed Display（有符号显示），和 Unsigned Display（无符号显示）。下图显示了所有的选项。

![&#x8C03;&#x8BD5;&#x5668;&#x5185;&#x5B58;&#x7A97;&#x53E3;&#x7684;&#x53F3;&#x952E;&#x83DC;&#x5355;](http://c.biancheng.net/uploads/allimg/190505/4-1Z5051330132F.gif)

### 整数数组求和

在刚开始编程时，几乎没有任务比计算数组元素总和更常见了。汇编语言实现数组求和步骤如下：

* 指定一个寄存器作变址操作数，存放数组地址。
* 循环计数器初始化为数组的长度。
* 指定一个寄存器存放累积和数，并赋值为0。
* 创建标号来标记循环开始的地方。
* 在循环体内，将和数与一个数组元素相加。
* 指向下一个数组元素。
* 用LOOP指令重复循环。

步骤 1 到步骤 3 可以按照任何顺序执行。下面的短程序实现对一个 16 位整数数组求和。

```text
; 数组求和（SumArray. asm）
.386
.model flat,stdcall
.stack 4096
ExitProcess proto,dwExitCode:dword
.data
intarray DWORD 10000h,20000h,30000h,40000h
.code
main PROC

    mov edi, OFFSET intarray   ; 1: EDI=intarray 地址
    mov ecx, LENGTHOF intarray ; 2 :循环计数器初始化
    mov    eax,0               ; 3:    sum=0
L1:                            ; 4：标记循环开始的地方
    add    eax,    [edi]       ; 5：加一个整数
    add edi, TYPE intarray     ; 6：指向下一个元素
    loop    L1                 ; 7：重复，直到 ECX=0
    invoke ExitProcess, 0
main ENDP
END main
```

### 复制字符串

程序常常要将大块数据从一个位置复制到另一个位置。这些数据可能是数组或字符串，但是它们可以包括任何类型的对象。

现在看看在汇编语言中如何实现这种操作，用循环来复制一个字符串，而字符串表示为带有一个空终止值的字节数组。变址寻址很适合于这种操作，因为可以用同一个变址寄存器来引用两个字符串。目标字符串必须有足够的空间来接收 被复制的字符，包括最后的空字节：

```text
;复制字符串    （CopyStr.asm）
.386
.model flat,stdcall
.stack 4096
ExitProcess proto,dwExitCode:dword
.data
source BYTE "This is the source string", 0
target BYTE SIZEOF source DUP(0)
.code
main PROC
    mov    esi, 0                      ;变址寄存器
    mov ecx, SIZEOF source             ;循环计数器
L1：                                   ;从源字符串获取一个字符
    mov    al, source [esi]            ;保存到目标字符串
    mov target [esi] , al              ;指向下一个字符
    inc esi                            ;重复，直到整个字符串完成
    loop L1
    invoke ExitProcess,0
main ENDP
END main
```

MOV 指令不能同时有两个内存操作数，所以，每个源字符串字符送入 AL，然后再从 AL 送入目标字符串。

## 4.17 64位MOV指令

64 位模式下的 MOV 指令与 32 位模式下的有很多共同点，只有几点区别，现在讨论一下。立即操作数（常数）可以是 8 位、16 位、32 位或 64 位。下面为一个 64 位示例：

```text
mov rax, 0ABCDEF0AFFFFFFFFh ; 64 位立即操作数
```

当一个 32 位常数送入 64 位寄存器时，目标操作数的高 32 位（位 32—位 63）被清除（等于 0）：

```text
mov rax, 0FFFFFFFFh ;rax = 00000000FFFFFFFF
```

向 64 位寄存器送入 16 位或 8 位常数，其高位也要清零：

```text
mov rax, 06666h  ;清位 16—位 63
mov rax, 055h      ;清位 8—位 63
```

如果将内存操作数送入 64 位寄存器，则结果是确定的。比如，传送一个 32 位内存操作数到 EAX（RAX 寄存器的低半部分），就会清除 RAX 的高 32 位：

```text
.data
myDword DWORD 80000000h
.code
mov rax,0FFFFFFFFFFFFFFFFh
mov eax,myDword                     ; RAX = 0000000080000000
```

但是，如果是将 8 位或 16 位内存操作数送入 RAX 的低位，那么，目标寄存器的高位不受影响：

```text
.data
myByte BYTE 55h
myWord WORD 6666h
.code
mov ax,myWord                ;位 16—位 63 不受影响
mov al, myByte                  ;位 8—位 63 不受影响
```

MOVSXD 指令（符号扩展传送）允许源操作数为 32 位寄存器或内存操作数。下面的指令使得 RAX 的值为 FFFFFFFFFFFFFFFFh：

```text
mov ebx, 0FFFFFFFFh
movsxd rax,ebx
```

OFFSET 运算符产生 64 位地址，必须用 64 位寄存器或变量来保存。下例中使用的是 RSI 寄存器：

```text
.data
myArray WORD 10,20,30,40
.code
mov rsi,OFFSET myArray
```

64 位模式中，LOOP 指令用 RCX 作为循环计数器。

有了这些基本概念，就可以编写许多 64 位模式程序了。大多数情况下，如果一直使用 64 位整数变量或 64 位寄存器，那么编程比较容易。ASCII 码字符串是一种特殊情况，因为它们总是包含字节。一般在处理时，采用间接或变址寻址。

## 4.18 64位加法和减法

如同 32 位模式下一样，ADD、SUB、INC 和 DEC 指令在 64 位模式下，也会影响 CPU 状态标志位。在下面的例子中，RAX 寄存器存放一个 32 位数，执行加 1，每一位都向左产生一个进位，因此，在位 32 生成 1：

```text
mov rax, 0FFFFFFFFh ;低 32 位是全 1add rax,1           ; RAX = 100000000h
```

需要时刻留意操作数的大小，当操作数只使用部分寄存器时，要注意寄存器的其他部分是没有被修改的。如下例所示，AX 中的 16 位总和翻转为全 0，但是不影响 RAX 的高位。这是因为该操作只使用 16 位寄存器（AX 和 BX）：

```text
mov rax,0FFFFh        ; RAX = 000000000000FFFFmov bx, 1add ax,bx             ; RAX = 0000000000000000
```

同样，在下面的例子中，由于 AL 中的进位不会进入 RAX 的其他位，所以执行 ADD 指令后，RAX 等于 0：

```text
mov rax,0FFh         ; RAX = 00000000000000FFmov bl, 1add al,bl            ; RAX = 0000000000000000
```

减法也使用相同的原则。在下面的代码段中，EAX 内容为 0，对其进行减 1 操作，将会使得 RAX 低 3 2位变为 -1（FFFFFFFFh）。同样，AX 内容为 0，对其进行减 1 操作，使得 RAX 低 16 位等于 -1（FFFFh）。

```text
mov rax,0               ; RAX = 0000000000000000
mov ebx, 1sub eax,ebx             ; RAX = 00000000FFFFFFFF
mov rax,0               ; RAX = 0000000000000000
mov bx,1sub ax,bx               ; RAX = 000000000000FFFF
```

当指令包含间接操作数时，必须使用 64 位通用寄存器。记住，一定要使用 PTR 运算符来明确目标操作数的大小。下面是一些包含了 64 位目标操作数的例子：

```text
dec BYTE PTR [rdi]              ;8 位目标操作数
inc WORD PTR [rbx]              ;16 位目标操作数
inc QWORD PTR [rsi]             ;64 位目标操作数
```

64 位模式下，可以对间接操作数使用比例因子，就像在 32 位模式下一样。如下例所示，如果处理的是 64 位整数数组，比例因子就是 8：

```text
.data
array QWORD 1,2,3,4
.code
mov esi, 3                   ;下标
mov rax,array[rsi*8]         ; RAX = 4
```

64 位模式的指针变量包含的是 64 位偏移量。在下面的例子中，ptrB 变量包含了数组 B 的偏移量：

```text
.data
arrayB BYTE 10h, 20h, 30h, 40h
ptrB QWORD arrayB
```

或者，还可以用 OFFSET 运算符来定义 ptrB，使得这个关系更加明确：

```text
ptrB QWORD OFFSET arrayB
```

