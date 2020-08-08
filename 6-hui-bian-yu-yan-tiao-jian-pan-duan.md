---
description: >-
  本章首先介绍了布尔操作，由于能影响 CPU 状态标志，它们是所有条件指令的核心。然后，说明怎样使用演绎 CPU
  状态标志的条件跳转和循环指令。接下来讲解了如何实现理论计算机科学中最根本的结构之一：有限状态机。在本章最后展示的是 MASM 内置的 32
  位编程的逻辑结构。
---

# 6 汇编语言条件判断

## 6.1 布尔和比较指令简介

前面介绍了四种基本的布尔代数操作：AND、OR、XOR 和 NOT。用[汇编语言](http://c.biancheng.net/asm/)指令，这些操作可以在二进制位上实现。同样，这些操作在布尔表达式层次上也很重要，比如 IF 语句。

首先了解按位指令，这里使用的技术也可以用于操作硬件设备控制位，实现通信协议以及加密数据，这里只列举了几种应用。Intel 指令集包含了 AND、OR、XOR 和 NOT 指令，它们能直接在二进制位上实现布尔操作，如下表所示。此外，TEST 指令是一种非破坏性的 AND 操作。

| 操作 | 说明 |
| :--- | :--- |
| AND | 源操作数和目的操作数进行逻辑与操作 |
| OR | 源操作数和目的操作数进行逻辑或操作 |
| XOR | 源操作数和目的操作数进行逻辑异或操作 |
| NOT | 对目标操作数进行逻辑非操作 |
| TEST | 源操作数和目的操作数进行逻辑与操作，并适当地设置 CPU 标志位 |

布尔指令影响零标志位、进位标志位、符号标志位、溢出标志位和奇偶标志位。下面简单回顾一下这些标志位的含义：

* 操作结果等于 0 时，零标志位置 1。
* 操作使得目标操作数的最高位有进位时，进位标志位置 1。
* 符号标志位是目标操作数高位的副本，如果标志位置 1，表示是负数；标志位清 0，表示是正数。（假设 0 为正。）
* 指令产生的结果超出了有符号目的操作数范围时，溢岀标志位置 1。
* 指令使得目标操作数低字节中有偶数个 1 时，奇偶标志位置 1。

接下来分别为大家讲解 AND、OR、XOR 和 NOT 的实际应用。

## 6.2 AND指令：对两个操作数进行逻辑（按位）与操作

AND 指令在两个操作数的对应位之间进行（按位）逻辑与（AND）操作，并将结果存放在目标操作数中：

```text
AND destination,source
```

下列是被允许的操作数组合，但是立即操作数不能超过 32 位：

```text
AND reg, reg
AND reg, mem
AND reg, imm
AND mem, reg
AND mem, imm
```

操作数可以是 8 位、16 位、32 位和 64 位，但是两个操作数必须是同样大小。两个操作数的每一对对应位都遵循如下操作原则：如果两个位都是 1，则结果位等于 1；否则结果位等于 0。

下表展示了两个输入位 X 和 Y，第三列是表达式 X^Y 的值：

| X | Y | X^Y |
| :--- | :--- | :--- |
| 0 | 0 | 0 |
| 0 | 1 | 0 |
| 1 | 0 | 0 |
| 1 | 1 | 1 |

AND 指令可以清除一个操作数中的 1 个位或多个位，同时又不影响其他位。这个技术就称为位屏蔽，就像在粉刷房子时，用遮盖胶带把不用粉刷的地方（如窗户）盖起来。

例如，假设要将一个控制字节从 AL 寄存器复制到硬件设备。并且当控制字节的位 0 和位 3 等于 0 时，该设备复位。那么，如果想要在不修改 AL 其他位的条件下，复位设备，可以用下面的指令：

```text
and AL, 11110110b       ;清除位 0 和位 3 ，其他位不变
```

如，设 AL 初始化为二进制数 1010 1110，将其与 1111 0110 进行 AND 操作后，AL 等于 1010 0110：

```text
mov al,10101110b
and al, 11110110b  ;AL 中的结果 = 1010 0110
```

### 标志位

AND 指令总是清除溢出和进位标志位，并根据目标操作数的值来修改符号标志位、零标志位和奇偶标志位。比如，下面指令的结果存放在 EAX 寄存器，假设其值为 0。在这种情况下，零标志位就会置 1：

```text
and eax,1Fh
```

### 将字符转换为大写

AND 指令提供了一种简单的方法将字符从小写转换为大写。如果对比大写 A 和小写 a 的 ASCII 码，就会发现只有位 5 不同：

```text
0 1 1 0 0 0 0 1 = 61h ('a')
0 1 0 0 0 0 0 1 = 41h ('A')
```

其他的字母字符也是同样的关系。把任何一个字符与二进制数 1101 1111 进行 AND，则除位 5 外的所有位都保持不变，而位 5 清 0。下例中，数组中所有字符都转换为大写：

```text
.data
array BYTE 50 DUP(?)
.code
    mov ecx,LENGTHOF array
    mov esi,OFFSET array
L1: and BYTE PTR [esi], 11011111b       ;清除位 5
    inc esi
    loop L1
```

## 6.3 OR指令：对两个操作数进行逻辑（按位）或操作

OR 指令在两个操作数的对应位之间进行（按位）逻辑或（OR）操作，并将结果存放在目标操作数中：

```text
OR destination, source
```

OR 指令操作数组合与 AND 指令相同：

```text
OR reg,reg
OR reg,mem
OR reg, imm
OR mem,reg
OR mem,imm
```

操作数可以是 8 位、16 位、32 位和 64 位，但是两个操作数必须是同样大小。对两个操作数的每一对对应位而言，只要有一个输入位是 1，则输出位就是 1。下面的真值表展示了布尔运算 x∨y：

| X | Y | X∨Y |
| :--- | :--- | :--- |
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 1 |

当需要在不影响其他位的情况下，将操作数中的 1 个位或多个位置为 1 时，OR 指令就非常有用了。比如，计算机与伺服电机相连，通过将控制字节的位 2 置 1 来启动电机。假设该控制字节存放在 AL 寄存器中，每一个位都含有重要信息，那么，下面的指令就只设置了位 2：

```text
or AL, 00000100b ;位 2 置 1，其他位不变
```

如果 AL 初始化为二进制数 1110 0011，把它与 0000 0100 进行 OR 操作，其结果等于 1110 0111：

```text
mov al,11100011b
or al, 00000100b    ;AL 中的结果 =1110 0111
```

### 标志位

OR 指令总是清除进位和溢出标志位，并根据目标操作数的值来修改符号标志位、零标志位和奇偶标志位。比如，可以将一个数与它自身（或 0）进行 OR 运算，来获取该数值的某些信息：

```text
or al,al
```

下表给出了零标志位和符号标志位对 AL 内容的说明：

| 零标志位 | 符号标志位 | AL 中的值 |
| :--- | :--- | :--- |
| 清0 | 清0 | 大于0 |
| 置1 | 清0 | 等于0 |
| 清0 | 置1 | 小于0 |

## 6.4 位向量（位映射）

有些应用控制的对象是从一个有限全集中选出来的一组项目。就像公司里的雇员，或者气象监测站的环境读数。在这些情景中，二进制位可以代表集合成员。

与 [Java](http://c.biancheng.net/java/) HashSet 用指针或引用指向容器内对象不同，应用可以用位向量（或位映射）把一个二进制数中的位映射为数组中的对象。

如下例所示，二进制数的位从左边 0 号开始，到右边 31 号为止，该数表示了数组元素 0、1、2 和 31 是名为 SetX 的集合成员：

```text
SetX = 10000000 00000000 00000000 00000111
```

（为了提供可读性，字节已经分开。）通过在特定位置与 1 进行 AND 运算，就可以方便地检测出该位是否为集合成员：

```text
mov eax,SetX
and eax, 10000b  ;元素［4］是 SetX 的成员吗？
```

如果本例中的 AND 指令清除了零标志位，那么就可以知道元素［4］是 SetX 的成员。

#### 1\) 补集

补集可以用 NOT 指令生成，NOT 指令将所有位都取反。因此，可以用下面的指令生成上例中 SetX 的补集，并存放在 EAX 中：

```text
mov eax,SetX
not eax         ;Setx的补集
```

#### 2\) 交集

AND 指令可以生成位向量来表示两个集合的交集。下面的代码生成集合 SetX 和 SetY 的交集，并将其存放在 EAX 中：

```text
mov eax,SetX
and eax,SetY
```

SetX 和 SetY 交集生成过程如下所示：

```text
        1000000000000000000000000000111 （SetX）
AND     1000001010100000000011101100011 （SetY）
-------------------------------------------------------------
        1000000000000000000000000000011 （交集）
```

很难想象还有更快捷的方法生成交集。对于更大的集合来说，它所需要的位超过了单个寄存器的容量，因此，需要用循环来实现所有位的 AND 运算。

#### 3\) 并集

OR 指令生成位图表示两个集合的并集。下面的代码产生集合 SetX 和 SetY 的并集，并将其存放在 EAX 中：

```text
mov eax,SetX
or eax,SetY
```

OR 指令生成 SetX 和 SetY 并集的过程如下所示：

```text
       1000000000000000000000000000111 （SetX）
OR     1000001010100000000011101100011 （SetY）
-------------------------------------------------------------
       1000001010100000000011101100111 （并集）
```

## 6.5 XOR指令：对两个操作数进行逻辑（按位）异或操作

XOR 指令在两个操作数的对应位之间进行（按位）逻辑异或（XOR）操作，并将结果存放在目标操作数中：

```text
XOR destination, source
```

XOR 指令操作数组合和大小与 AND 指令及 OR 指令相同。两个操作数的每一对对应位都应用如下操作原则：如果两个位的值相同（同为 0 或同为 1），则结果位等于 0；否则结果位等于 1。

下表描述的是布尔运算 X㊉y：

| x | y | x㊉y |
| :--- | :--- | :--- |
| 0 | 0 | 0 |
| 0 | 1 | 1 |
| 1 | 0 | 1 |
| 1 | 1 | 0 |

与 0 异或值保持不变，与 1 异或则被触发（求补）。对相同操作数进行两次 XOR 运算，则结果逆转为其本身。如下表所示，位 x 与位 y 进行了两次异或，结果逆转为 x 的初始值：

| x | y | x㊉y | \(x㊉y\)㊉y |
| :--- | :--- | :--- | :--- |
| 0 | 0 | 0 | 0 |
| 0 | 1 | 1 | 0 |
| 1 | 0 | 1 | 1 |
| 1 | 1 | 0 | 1 |

异或运算这种“可逆的”属性使其成为简单对称加密的理想工具。

#### 标志位

XOR 指令总是清除溢岀和进位标志位，并根据目标操作数的值来修改符号标志位、零标志位和奇偶标志位。

#### 检查奇偶标志

奇偶检查是在一个二进制数上实现的功能，计算该数中 1 的个数；如果计算结果为偶数，则说该数是偶校验；如果结果为奇数，则该数为奇校验。

x86 处理器中，当按位操作或算术操作的目标操作数最低字节为偶校验时，奇偶标志位置 1。反之，如果操作数为奇校验，则奇偶标志位清 0。一个既能检查数的奇偶性，又不会修改其数值的有效方法是，将该数与 0 进行异或运算：

```text
mov al,10110101b       ;5 个 1,奇校验
xor al, 0                ;奇偶标志位清 0 （奇）
mov al, 11001100b       ;4 个 1，偶校验
xor al, 0                ;奇偶标志位置 1（偶）
```

Visual Studio 用 PE=1 表示偶校验，PE=0 表示奇校验。

#### 16 位奇偶性

对 16 位整数来说，可以通过将其高字节和低字节进行异或运算来检测数的奇偶性：

```text
mov ax,64Clh  ;0110 0100 1100 0001
xor ah, al      ;奇偶标志位置1 （偶）
```

将每个寄存器中的置 1 位（等于 1 的位）想象为一个 8 位集合中的成员。XOR 指令把两个集合交集中的成员清 0，并形成了其余位的并集。这个并集的奇偶性与整个 16 位整数的奇偶性相同。

那么 32 位数值呢？如果将数值的字节进行编号，从 B₀ 到 B₃ 那么计算奇偶性的表达式为:B₀ XOR B₁ XOR B₂ XOR B₃。

## 6.6 NOT（反码）指令：翻转操作数的所有位

NOT 指令触发（翻转）操作数中的所有位。其结果被称为反码。该指令允许的操作数类型如下所示：

```text
NOT reg
NOT mem
```

例如，F0h 的反码是 0Fh：

```text
mov al,11110000b
not al             ;AL = 00001111b
```

> 提示：NOT 指令不影响标志位。

## 6.7 TEST指令：对两个操作数进行逻辑（按位）与操作

TEST 指令在两个操作数的对应位之间进行 AND 操作，并根据运算结果设置符号标志位、零标志位和奇偶标志位。

TEST 指令与《[AND指令](http://c.biancheng.net/view/3553.html)》一节中介绍的 AND 指令唯一不同的地方是，TEST 指令不修改目标操作数。TEST 指令允许的操作数组合与 AND 指令相同。在发现操作数中单个位是否置位时，TEST 指令非常有用。

#### 示例：多位测试

TEST 指令同时能够检查几个位。假设想要知道 AL 寄存器的位 0 和位 3 是否置 1，可以使用如下指令：

```text
test al, 00001001b ;测试位 0 和位 3
```

（本例中的 0000 1001 称为位掩码。）从下面的数据集例子中，可以推断只有当所有测试位都清 0 时，零标志位才置 1：

```text
0 0 1 0 0 1 0 1  <- 输入值
0 0 0 0 1 0 0 1  <- 测试值
0 0 0 0 0 0 0 1  <- 结果：ZF=0

0 0 1 0 0 1 0 0  <- 输入值
0 0 0 0 1 0 0 1  <- 测试值
0 0 0 0 0 0 0 0  <- 结果：ZF=1
```

#### 标志位

TEST 指令总是清除溢出和进位标志位，其修改符号标志位、零标志位和奇偶标志位的方法与 AND 指令相同。

## 6.8 CMP（比较）指令：比较整数

了解了所有按位操作指令后，现在来讨论逻辑（布尔）表达式中的指令。最常见的布尔表达式涉及一些比较操作，下面的伪码片段展示了这种情况：

```text
if A > B ...
while X > 0 and X < 200  ...
if check_for_error(N) = true
```

x86 [汇编语言](http://c.biancheng.net/asm/)用 CMP 指令比较整数。字符代码也是整数，因此可以用 CMP 指令。

CMP（比较）指令执行从目的操作数中减去源操作数的隐含减法操作，并且不修改任何操作数：

```text
CMP destination,source
```

#### 标志位

当实际的减法发生时，CMP 指令按照计算结果修改溢出、符号、零、进位、辅助进位和奇偶标志位。

如果比较的是两个无符号数，则零标志位和进位标志位表示的两个操作数之间的关系如右表所示：

| CMP结果 | ZF | CF |
| :--- | :--- | :--- |
| 目的操作数 &lt; 源操作数 | 0 | 1 |
| 目的操作数 &gt; 源操作数 | 0 | 0 |
| 目的操作数 = 源操作数 | 1 | 0 |

如果比较的是两个有符号数，则符号标志位、零标志位和溢出标志位表示的两个操作数之间的关系如右表所示：

| CMP结果 | 标志位 |
| :--- | :--- |
| 目的操作数 &lt; 源操作数 | SF ≠ OF |
| 目的操作数 &gt; 源操作数 | SF=OF |
| 目的操作数 = 源操作数 | ZF=1 |

CMP 指令是创建条件逻辑结构的重要工具。当在条件跳转指令中使用 CMP 时，汇编语言的执行结果就和 IF 语句一样。

下面用三段代码来说明标志位是如何受到 CMP 影响的。设 AX=5，并与 10 进行比较，则进位标志位将置 1，原因是（5-10）需要借位：

```text
mov ax, 5
cmp ax,10   ; ZF = 0 and CF = 1
```

1000 与 1000 比较会将零标志位置 1，因为目标操作数减去源操作数等于 0：

```text
mov ax,1000
mov cx,1000
cmp cx, ax    ;ZF = 1 and CF = 0
```

105 与 0 进行比较会清除零和进位标志位，因为（105-0）的结果是一个非零的正整数。

```text
mov si,105
cmp si, 0    ; ZF = 0 and CF = 0
```

## 6.9 置位和清除单个CPU标志位

怎样能方便地置位和清除零标志位、符号标志位、进位标志位和溢出标志位？有几种方法，其中的一些需要修改目标操作数。要将零标志位置 1，就把操作数与 0 进行 TEST 或 AND 操作；要将零标志位清零，就把操作数与 1 进行 OR 操作：

```text
test al, 0      ;零标志位置 1
and al, 0      ;零标志位置 1
or al, 1       ;零标志位清零
```

TEST 指令不修改目的操作数，而 AND 指令则会修改目的操作数。若要符号标志位置 1，将操作数的最高位和 1 进行 OR 操作；若要清除符号标志位，则将操作数最高位和 0 进行 AND 操作：

```text
or al, 80h     ;符号标志位置 1
and al, 7Fh    ;符号标志位清零
```

若要进位标志位置 1，用 STC 指令；清除进位标志位，用 CLC 指令：

```text
stc          ;进位标志位置 1
clc          ;进位标志位清零
```

若要溢出标志位置 1，就把两个正数相加使之产生负的和数；若要清除溢出标志位，则将操作数和 0 进行 OR 操作：

```text
mov al,7Fh    ; AL = +127
inc al        ; AL = 80h (-128), OF=1
or eax, 0      ; 溢出标志位清零
```

## 6.10 64位模式下的布尔指令

大多数情况下，64 位模式中的 64 位指令与 32 位模式中的操作是一样的。比如，如果源操作数是常数，长度小于 32 位，而目的操作数是一个 64 位寄存器或内存操作数，那么，目的操作数中所有的位都会受到影响：

```text
.data
allones QWORD 0FFFFFFFFFFFFFFFFh
.code
    mov rax,allones                  ;RAX = FFFFFFFFFFFFFFFF
    and rax,80h                      ;RAX = 0000000000000080
    mov rax,allones                  ;RAX = FFFFFFFFFFFFFFFF
    and rax,8080h                    ;RAX = 0000000000008080
    mov rax,allones                  ;RAX = FFFFFFFFFFFFFFFF
    and rax,808080h                  ;RAX = 0000000000808080
```

但是，如果源操作数是 32 位常数或寄存器，那么目的操作数中，只有低 32 位会受到影响。如下例所示，只有 RAX 的低 32 位被修改了：

```text
mov rax,allones                 ;RAX = FFFFFFFFFFFFFFFF
and rax,80808080h              ;RAX = FFFFFFFF80808080
```

当目的操作数是内存操作数时，得到的结果是一样的。显然，32 位操作数是一个特殊的情况，需要与其他大小操作数的情况分开考虑。

## 6.11 条件跳转简介

x86 指令集中没有明确的高级逻辑结构，但是可以通过比较和跳转的组合来实现它们。

执行一个条件语句需要两个步骤：

* 第一步，用 CMP、AND 或 SUB 操作来修改 CPU 状态标志位；
* 第二步，用条件跳转指令来测试标志位，并产生一个到新地址的分支。

下面是一些例子。

【示例 1】本例中的 CMP 指令把 EAX 的值与 0 进行比较，如果该指令将零标志位置 1，则 JZ（为零跳转）指令就跳转到标号 L1：

```text
    cmp eax, 0
    jz L1          ;如果 ZF=1 则跳转
    .
    .
L1：
```

【示例 2】本例中的 AND 指令对 DL 寄存器进行按位与操作，并影响零标志位。如果零标志位清零，则 JNZ（非零跳转）指令跳转：

```text
    and dl,10110000b
    jnz L2             ;如果 ZF=0 则跳转
    .
    .
L2 :
```

#### Jcond 指令

当状态标志条件为真时，条件跳转指令就分支到目标标号。否则，当标志位条件为假时，立即执行条件跳转后面的指令。语法如下所示：

```text
Jcond destination
```

cond 是指确定一个或多个标志位状态的标志位条件。下面是基于进位和零标志位的例子：

| JC | 进位跳转（进位标志位置 1） |
| :--- | :--- |
| JNC | 无进位跳转（进位标志位清零） |
| JZ | 为零跳转（零标志位置 1） |
| JNZ | 非零跳转（零标志位清零） |

CPU 状态标志位最常见的设置方法是通过算术运算、比较和布尔运算指令。条件跳转指令评估标志位状态，利用它们来决定是否发生跳转。

用 CMP 指令 假设当 EAX=5 时，跳转到标号 L1。在下面的例子中，如果 EAX=5，CMP 指令就将零标志位置 1；之后，由于零标志位为 1，JE 指令就跳转到 L1：

```text
cmp eax,5
je L1         ;如果相等则跳转
```

JE 指令总是按照零标志位的值进行跳转。如果 EAX 不等于 5，CMP 就会清除零标志位，那么，JE 指令将不跳转。

下例中，由于 AX 小于 6，所以 JL 指令跳转到标号 L1：

```text
mov ax, 5
cmp ax, 6
jl L1         ;小于则跳转
```

下例中，由于 AX 大于4，所以发生跳转：

```text
mov ax,5
cmp ax,4
jg L1       ;大于则跳转
```

## 6.12 条件跳转指令汇总

x86 指令集包含大量的条件跳转指令。它们能比较有符号和无符号整数，并根据单个 CPU 标志位的值来执行操作。条件跳转指令可以分为四个类型：

* 基于特定标志位的值跳转
* 基于两数是否相等，或是否等于（E）CX 的值跳转
* 基于无符号操作数的比较跳转
* 基于有符号操作数的比较跳转

下表展示了基于零标志位、进位标志位、溢出标志位、奇偶标志位和符号标志位的跳转。

| 助记符 | 说明 | 标志位/寄存器 | 助记符 | 说明 | 标志位/寄存器 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| JZ | 为零跳转 | ZF=1 | JNO | 无溢出跳转 | OF=0 |
| JNZ | 非零跳转 | ZF=0 | JS | 有符号跳转 | SF=1 |
| JC | 进位跳转 | CF=1 | JNS | 无符号跳转 | SF=0 |
| JNC | 无进位跳转 | CF=0 | JP | 偶校验跳转 | PF=1 |
| JO | 溢出跳转 | OF=1 | JNP | 奇校验跳转 | PF=0 |

### 1\) 相等性的比较

下表列出了基于相等性评估的跳转指令。有些情况下，进行比较的是两个操作数；其他情况下，则是基于 CX、ECX 或 RCX 的值进行跳转。表中符号 leftOp 和 rightOp 分别指的是 CMP 指令中的左（目的）操作数和右（源）操 作数：

| 助记符 | 说明 |
| :--- | :--- |
| JE | 相等跳转 \(leftOp=rightOp\) |
| JNE | 不相等跳转 \(leftOp M rightOp\) |
| JCXZ | CX=0 跳转 |
| JECXZ | ECX=0 跳转 |
| JRCXZ | RCX=0 跳转（64 位模式） |

```text
CMP leftOp,rightOp
```

操作数名字反映了代数中关系运算符的操作数顺序。比如，表达式 X&lt; Y 中，X 被称为 leftOp，Y 被称为 rightOp。

尽管 JE 指令相当于 JZ（为零跳转），JNE 指令相当于 JNZ（非零跳转），但是，最好是选择最能表明编程意图的助记符（JE 或 JZ），以便说明是比较两个操作数还是检查特定的状态标志位。

下述示例使用了 JE、JNE、JCXZ 和 JECXZ 指令。仔细阅读注释，以保证理解为什么条件跳转得以实现（或不实现）。

示例 1：

```text
mov edx, 0A523h
cmp edx, 0A523h
jne L5            ;不发生跳转
je L1             ;跳转
```

示例 2：

```text
mov bx,1234h
sub bx,1234h
jne L5            ;不发生跳转
je L1             ;跳转
```

示例 3：

```text
mov ex, 0FFFFh
inc ex
jexz L2           ;跳转
```

示例4：

```text
xor ecx,ecx
jeexz L2          ;跳转
```

### 2\) 无符号数比较

基于无符号数比较的跳转如下表所示。操作数的名称反映了表达式中操作数的顺序（比如 leftOp &lt; rightOp）。下表中的跳转仅在比较无符号数值时才有意义。有符号操作数使用不同的跳转指令。

| 助记符 | 说明 | 助记符 | 说明 |
| :--- | :--- | :--- | :--- |
| JA | 大于跳转（若 leftOp &gt; rightOp） | JB | 小于跳转（若 leftOp &lt; rightOp） |
| JNBE | 不小于或等于跳转（与 JA 相同） | JNAE | 不大于或等于跳转（与 JB 相同） |
| JAE | 大于或等于跳转（若 leftOp ≥ rightOp） | JBE | 小于或等于跳转（若 leftOp ≤ rightOp） |
| JNB | 不小于跳转（与 JAE 相同） | JNA | 不大于跳转（与 JBE 相同） |

### 3\) 有符号数比较

下表列岀了基于有符号数比较的跳转。下面的指令序列展示了两个有符号数值的比较：

| 助记符 | 说明 | 助记符 | 说明 |
| :--- | :--- | :--- | :--- |
| JG | 大于跳转（若 leftOp &gt; rightOp） | JL | 小于跳转（若 leftOp &lt; rightOp） |
| JNLE | 不小于或等于跳转（与 JG 相同） | JNGE | 不大于或等于跳转（与 JL 相同） |
| JGE | 大于或等于跳转（若 leftOp ≥ rightOp） | JLE | 小于或等于跳转（若 leftOp ≤ rightOp） |
| JNL | 不小于跳转（与 JGE 相同） | JNG | 不大于跳转（与 JLE 相同） |

```text
mov al, +127       ;十六进制数值 7Fh
cmp al, -128       ;十六进制数值 80h
ja Is Above        ;不跳转，因为 7Fh < 80h
jg IsGreater        ;跳转，因为 +127 > -128
```

由于无符号数 7Fh 小于无符号数 80h，因此，为无符号数比较而设计的 JA 指令不发生跳转。另一方面，由于 +127 大于 -128，因此，为有符号数比较而设计的 JG 指令发生跳转。

对下面的代码示例，阅读注释，以保证理解为什么跳转得以实现（或不实现）：

示例 1：

```text
mov edx,-1
cmp edx, 0
jnl L5          ;不发生跳转（-1 ≥ 0 为假）
jnle L5         ;不发生跳转（-1 > 0 为假）
jl L1           ;跳转（-1 < 0 为真）
```

示例 2：

```text
mov bx,+ 32
cmp bx,-35
jng L5         ;不发生跳转（ + 32 ≤ -35 为假）
jnge L5        ;不发生跳转（ + 32 < -35 为假）
jge L1         ;跳转（ + 32 ≥ -35 为真）
```

示例 3：

```text
mov ecx, 0
cmp ecx, 0
jg L5           ;不发生跳转（0 > 0 为假）
jnl L1          ;跳转（0 ≥ 0 为真）
```

示例 4：

```text
mov ecx, 0
cmp ecx, 0
jl L5           ;不发生跳转（0 < 0 为假）
jng L1          ;跳转（0 ≤ 0 为真）
```

## 6.13 条件跳转应用及示例

[汇编语言](http://c.biancheng.net/asm/)做得最好的事情之一就是位测试。通常，不希望改变进行位测试的数值，但是却希望能修改 CPU 状态标志位的值。

条件跳转指令常常用这些状态标志位来决定是否将控制转向代码标号。例如，假设有一个名为 status 的 8 位内存操作数，它包含了与计算机连接的一个外设的状态信息。如果该操作数的位 5 等于 1，表示外设离线，则下面的指令就跳转到标号：

```text
mov al, status
test al, 00100000b     ;测试位 5
jnz DeviceOffline
```

如果位 0、1 或 4 中任一位置 1，则下面的语句跳转到标号：

```text
mov al, status
test al, 00010011b     ;测试位 0、1、4
jnz InputDataByte
```

如果是位 2、3 和 7 都置 1 使得跳转发生，则还需要 AND 和 CMP 指令：

```text
mov al, status
and al,10001100b       ;屏蔽位 2、3 和 7
cmp al, 10001100b      ;所有位都置 1 ?
je ResetMachine        ;是：跳转
```

### 两个数中的较大数

下面的代码比较了 EAX 和 EBX 中的两个无符号整数，并且把其中较大的数送入 EDX：

```text
  mov  edx, eax        ;假设EAX存放较大的数
  cmp eax, ebx          ;若 EAX ≥ EBX
  jae L1                ;跳转到 L1
  mov  edx, ebx        ;否则，将 EBX 的值送入 EDX
L1:                    ;EDX 中存放的是较大的数
```

### 三个数中的最小数

下面的代码比较了分别存放于三个变量 VI、V2 和 V3 的无符号 16 位数值，并且把其中最小的数送入AX：

```text
.data
V1 WORD ?
V2 WORD ?
V3 WORD ?
.code
    mov  ax, V1   ;假设 V1 是最小值
    cmp  ax, V2   ;如果 AX ≤ V2
    jbe  L1       ;跳转到 L1
    mov  ax, V2   ;否则，将 V2 送入 AX
LI:   cmp  ax, V3   ;如果 AX ≤ V3
    jbe L2        ;跳转到L2
    mov  ax, V3   ;否则，将V3送入AX
L2 :
```

### 循环直到按下按键

下面的 32 位代码会持续循环，直到用户按下任意一个标准的字母数字键。如果输入缓冲区中当前没有按键，那么 Irvine32 库中的 ReadKey 函数就会将零标 志位置1:

```text
.data
char BYTE ?
.code
L1: mov eax, 10     
    call Delay       ;创建 10 毫秒的延迟;
    call ReadKey    ;检查按键
    jz L1           ;如果没有按键则循环
    mov char, AL    ;保存字符)
```

上述代码在循环中插入了一个 10 毫秒的延迟，以便 MS-Windows 有时间处理事件消息。如果省略这个延迟，那么按键可能被忽略。

### 【示例 1】：顺序搜索数组

常见的编程任务是在数组中搜索满足某些条件的数值。例如，下述程序就是在一个 16 位数组中寻找第一个非零数值。如果找到，则显示该数值；否则，就显示一条信息，以说明没有发现非零数值：

```text
;扫描数组    （ArrayScan.asm）
;扫描数组寻找第一个非零数值
INCLUDE Irvine32.inc

.data
intArray SWORD 0,0,0,0,1,20,35,-12,66,4,0
;intArray SWORD 1,0,0,0            ;候补测试数据
;intArray SWORD 0,0,0,0            ;候补测试数据
;intArray SWORD 0,0,0,1            ;候补测试数据
noneMsg BYTE "A non-zero value was not found",0

.code
main PROC
    mov ebx,OFFSET intArray        ;指向数组
    mov ecx,LENGTHOF intArray      ;循环计数器
L1: cmp WORD PTR [ebx],0           ;将数值与0比较
    jnz found                      ;寻找数值
    add ebx,2                      ;指向下一个元素
    loop L1                        ;继续循环
    jmp    notFound                ;没有发现非零数值
found:
    movsx eax,WORD PTR[ebx]        ;送人EAX并进行符号扩展
    call WriteInt
    jmp quit

notFound:
    mov edx,OFFSET noneMsg         ;显示“没有发现”消息
    call WriteString

quit:
    call Crlf
    exit
main ENDP
END main
```

本程序包含了可以替换的测试数据，它们已经被注释出来。取消这些注释行，就可 以用不同的数据配置来测试程序。

### 【示例 2】：简单字符串加密

XOR 指令有一个有趣的属性。如果一个整数 X 与 Y 进行 XOR，其结果再次与 Y 进行 XOR，则最后的结果就是 X：

```text
( ( X ㊉ Y ) ㊉ Y) = X
```

XOR 的可逆性为简单数据加密提供了一种方便的途径：明文消息转换成加密字符串，这个加密字符串被称为密文，加密方法是将该消息与被称为密钥的第三个字符串按位进行 XOR 操作。预期的查看者可以用密钥解密密文，从而生成原始的明文。

下面将演示一个使用对称加密的简单程序，即用同一个密钥既实现加密又实现解密的过程。运行时，下述步骤依序发生：

1\) 用户输入明文。

2\) 程序使用单字符密钥对明文加密，产生密文并显示在屏幕上。

3\) 程序解密密文，产生初始明文并显示出来。

程序清单完整的程序清单如下所示：

```text
;加密程序    （Encrypt.asm）

INCLUDE Irvine32.inc
KEY = 239                    ;1-255之间的任一值
BUFMAX = 128                 ;缓冲区的最大容量

.data
sPrompt BYTE "Enter the plain text:",0
sEncrypt BYTE "Cipher text",0
sDecrypt BYTE "Decrypted:",0
buffer BYTE BUFMAX+1 DUP(0)
bufSize DWORD ?

.code
main PROC
    call InputTheString        ;输入明文
    call TranslateBuffer       ;加密缓冲区
    mov edx,OFFSET sEncrypt    ;显示加密消息
    call DisplayMessage
    call TranslateBuffer       ;解密缓冲区
    mov edx,OFFSET sDecrypt    ;显示解密消息
    call DisplayMessage
    exit
main ENDP

;-----------------------------------------
InputTheString PROC
;
;提示用户输入一个纯文本字符串
;保存字符串和它的长度
;接收：无
;返回：无
;-----------------------------------------
    pushad                     ;保存32位寄存器
    mov edx,OFFSET sPrompt     ;显示提示
    call WriteString
    mov ecx,BUFMAX             ;字符计数器最大值
    mov edx,OFFSET buffer      ;指向缓冲区
    call ReadString            ;输入字符串
    mov bufSize,eax            ;保存长度
    call Crlf
    popad
    ret
InputTheString ENDP

;-----------------------------------------
DisplayMessage PROC
;
;显示加密或解密消息
;接收：EDX指向消息
;返回：无
;-----------------------------------------
    pushad
    call WriteString
    mov edx,OFFSET buffer        ;显示缓冲区
    call WriteString
    call Crlf
    call Crlf
    popad
    ret
DisplayMessage ENDP

;-----------------------------------------
TranslateBuffer PROC
;
;字符串的每个字节都与密钥字节进行异或
;实现转换
;接收：无
;返回：无
;-----------------------------------------
    pushad
    mov ecx,bufSize                ;循环计数器
    mov esi,0                      ;缓冲区索引初始值赋0
L1:
    xor buffer[esi],KEY            ;转换一个字节
    inc esi                        ;指向下一个字节
    loop L1
    popad
    ret
TranslateBuffer ENDP
END main
```

## 6.14 LOOPZ（为零跳转）和LOOPE（相等跳转）指令

LOOPZ（为零跳转）指令的工作和 LOOP 指令相同，只是有一个附加条件：为零控制转向目的标号，零标志位必须置 1。指令语法如下：

```text
LOOPZ destination
```

LOOPE（相等跳转）指令相当于 LOOPZ 它们有相同的操作码。这两条指令执行如下任务：

```text
ECX = ECX - 1
if ECX > 0 and ZF = 1, jump to destination
```

否则，不发生跳转，并将控制传递到下一条指令。LOOPZ 和 LOOPE 不影响任何状态标志位。32 位模式下，ECX 是循环计数器；64 位模式下，RCX 是循环计数器。

## 6.15 LOOPNZ（非零跳转）和LOOPNE（不等跳转）指令

LOOPNZ（非零跳转）指令与 LOOPZ 相对应。当 ECX 中无符号数值大于零（减 1 操作之后）且零标志位等于零时，继续循环。指令语法如下：

```text
LOOPNZ destination
```

LOOPNE（不等跳转）指令相当于 LOOPNZ 它们有相同的操作码。这两条指令执行如 下任务：

```text
ECX = ECX - 1
if ECX > 0 and ZF = 0, jump to destination
```

否则，不发生跳转，并将控制传递到下一条指令。

【示例】扫描数组中的每一个数，直到发现一个非负数（符号位为 0）为止。注意，在执行 ADD 指令前要把标志位压入堆栈，因为 ADD 有可能修改标志位。然后在执行 LOOPNZ 指令之前，用 POPFD 恢复这些标志位：

```text
.data
array  SWORD  -3,-6,-1,-10,10,30,40,4
sentinel SWORD  0

.code
main PROC
mov esi,OFFSET array
mov ecx,LENGTHOF array

next:
test WORD PTR [esi],8000h    ; 测试符号位
pushfd                       ; 标志位入栈
add  esi,TYPE array          ; 移动到下一个位置
popfd                        ; 标志位出栈
loopnz next                  ; 继续循环
jnz  quit                    ; 没有发现非负数
sub  esi,TYPE array          ; ESI 指向数值
quit:
```

如果找到一个非负数，ESI 会指向该数值。如果没有找到一个正数，则只有当 ECX=0 时才终止循环。在这种情况下，JNZ 指令跳转到标号 quit，同时 ESI 指向标记值（0），其在内存中的位置正好紧接着该数组。

## 6.16 实现IF语句

IF 结构包含一个布尔表达式，其后有两个语句列表：一个是当表达式为真时执行，另一个是当表达式为假时执行：

```cpp
if( boolean-expression )
  statement-list-1
else
  statement-list-2
```

结构中的 else 部分是可选的。在[汇编语言](http://c.biancheng.net/asm/)中，则是用多个步骤来实现这种结构的。首先，对布尔表达式求值，这样一来某个 CPU 状态标志位会受到影响。然后，根据相关 CPU 状态标志位的值，构建一系列跳转把控制传递给两个语句列表。

【示例 1】下面的 [C++](http://c.biancheng.net/cplus/) 代码中，如果 op1 等于 op2，则执行两条赋值语句：

```cpp
if( op1 = op2 )
{
    X = 1;
    Y = 2;
}
```

在汇编语言中，这种 IF 语句转换为条件跳转和 CMP 指令。由于 op1 和 op2 都是内存操作数（变量），因此，在执行 CMP 之前，要将其中的一个操作数送入寄存器。

下面实现 IF 语句的程序是高效的，当逻辑表达式为真时，它允许代码“通过”直达两条期望被执行的 MOV 指令：

```text
        mov eax, op1
        cmp eax,op2                  ; op1 == op2?
        jne L1                       ; 否：跳过后续指令
        mov X, 1                     ; 是：X, Y 赋值
        mov Y, 2
L1:
```

如果用 JE 来实现 == 运算符，生成的代码就没有那么紧凑了（6 条指令，而非 5 条指令）：

```text
        mov    eax, op1
        cmp    eax,op2              ; op1 == op2?
        je    L1                    ; 是：跳转到 L1
        jmp    L2                   ; 否：跳过赋值语句
LI：    mov X, 1                    ; X, Y 赋值
        mov    Y, 2
L2 :
```

从上面的例子可以看出，相同的条件结构在汇编语言中有多种实现方法。上面给出 的编译代码示例只代表一种假想的编译器可能产生的结果。

【示例 2】NTFS 文件存储系统中，磁盘簇的大小取决于磁盘卷的总容量。如下面的伪代码所示，如果卷大小（用变量 terrabytes 存放）不超过 16TB，则簇大小设置为 4096。否则， 簇大小设置为 8192：

```text
clusterSize = 8192;
if terrabytes < 16
  clusterSize = 4096;
```

用汇编语言实现该伪代码：

```text
        mov clusterSize, 8192                ;假设较大的磁盘簇        cmp terrabytes, 16                   ;小于 16TB?        jae next        mov clusterSize, 4096                ;切换到较小的磁盘簇next:
```

【示例 3】下面的伪代码有两个分支：

```text
if op1 > op2
  call Routine1
else
  call Routine2
end if
```

用汇编语言翻译这段伪代码，设 op1 和 op2 是有符号双字变量。对这两个变量比较时，其中一个必须送入寄存器：

```text
        mov eax, op1                    ; opl送入寄存器
        cmp    eax, op2                 ; opl > op2?
        jg    A1                        ; 是：调用 Routine1
        call    Routine2                ; 否：调用 Routine2
        jmp    A2    ;退出工F语句
A1:     call Routine1
A2:
```

### 白盒测试

复杂条件语句可能有多个执行路径，这使得它们难以进行调试检查（查看代码）。程序员经常使用的技术称为白盒测试，用来验证子程序的输入和相应的输出。

白盒测试需要源代码，并对输入变量进行不同的赋值。对每个输入组合，要手动跟踪源代码，验证其执行路径和子程序产生的输出。下面，通过嵌套 IF 语句的汇编程序来看看这个测试过程：

```text
if op1 == op2
    if X > Y
        call Routine1
    else
        call Routine2
    end if
else
    call Routine3
end if
```

下面是可能的汇编语言翻译，加上了参考行号。程序改变了初始条件（op1 == op2），并立即跳转到 ELSE 部分。剩下要翻译的内容是内层 IF-ELSE 语句：

```text
        mov    eax, op1
        cmp eax, op2                             ;op1 == op2?
        jne    L2                                ;否：调用 Routine3

; 处理内层 IF-ELSE 语句。
        mov    eax, X
        cmp    eax, Y                             ; X > Y?
        jg    L1                                  ; 是：调用 Routine1
        call    Routine2                          ; 否：调用 Routine2
        jmp    L3                                 ; 退出
L1:     call Routine1                             ; 调用 Routine1
        jmp    L3                                 ; 退出
L2:     call    Routine3
L3:
```

下表给出了示例代码的白盒测试结果。前四列对 op1、op2、X 和 Y 进行测试赋值。第 5 列和第 6 列对生成的执行路径进行了验证。

| op1 | op2 | X | Y | 执行行序列 | 调用 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 10 | 20 | 30 | 40 | 1, 2, 3, 11, 12 | Rountine3 |
| 10 | 20 | 40 | 30 | 1, 2, 3, 11, 12 | Rountine3 |
| 10 | 10 | 30 | 40 | 1, 2, 3, 4, 5, 6, 7, 8, 12 | Rountine2 |
| 10 | 10 | 40 | 30 | 1, 2, 3, 4, 5, 6, 9, 10, 12 | Rountine1 |

## 6.17 实现逻辑表达式

在高级编程语言中可以使用复合表达式，但是大家可能并不了解语言编译器是如何将其转化为机器代码的。下面来介绍一下如何使用[汇编语言](http://c.biancheng.net/asm/)来实现复合表达式。

### 逻辑 AND 运算符

汇编语言很容易实现包含 AND 运算符的复合布尔表达式。考虑下面的伪代码，假设其中进行比较的是无符号整数：

```c
if (a1 > b1) AND (b1 > c1)
  X = 1
end if
```

#### 短路求值

下面的例子是短路求值的简单实现，如果第一个表达式为假，则不需计算第二个表达式。高级语言的规范如下：

```text
        cmp a1,b1                  ;第一个表达式…
        ja L1
        jmp next
L1:     cmp b1, c1                 ;第二个表达式…
        ja L2
        jmp next
L2:   mov X, 1                     ;全为真：将 X 置 1
next:
```

如果把第一条 JA 指令替换为 JBE，就可以把代码减少到 5 条：

```text
        cmp    a1,b1                  ; 第一个表达式…
        jbe next                      ; 如果假，则退出
        cmp    b1,c1                  ; 第二个表达式…
        jbe next                      ; 如果假，则退出
        mov    X, 1                   ; 全为真
next:
```

若第一个 JBE 不执行，CPU 可以直接执行第二个 CMP 指令，这样就能够减少 29% 的代码量（指令数从 7 条减少到 5 条）。

### 逻辑 OR 运算符

当复合表达式包含的子表达式是用 OR 运算符连接的，那么只要一个子表达式为真，则整个复合表达式就为真。以如下伪代码为例：

```c
if (a1 > b1) OR (b1 > c1)
  X = 1
```

在下面的实现过程中，如果第一个表达式为真，则代码分支到 L1；否则代码直接执行第二个 CMP 指令。第二个表达式翻转了 &gt; 运算符，并使用了 JBE 指令：

```text
        cmp a1, b1                  ; 1：比较 AL 和 BL
        ja L1                       ; 如果真，跳过第二个表达式
        cmp b1, c1                  ; 2：比较 BL 和 CL
        jbe next                    ; 假：跳过下一条语句
L1:     mov X, 1                    ; 真：将 x 置 1
next:
```

对于一个给定的复合表达式而言，汇编语句有多种实现方法。

## 6.18 实现WHILE循环

WHILE 循环在执行语句块之前先进行条件测试。只要循环条件一直为真，那么语句块就不断重复。下面是用 [C++](http://c.biancheng.net/cplus/) 编写的循环：

```c
while( val1 < val2 )
{
  val1++；
  val2 --；
}
```

用[汇编语言](http://c.biancheng.net/asm/)实现这个结构时，可以很方便地改变循环条件，当条件为真时，跳转到 endwhile。假设 val1 和 val2 都是变量，那么在循环开始之前必须将其中的一个变量送入寄存器，并且还要在最后恢复该变量的值：

```text
        mov eax, val1                  ; 把变量复制到 EAX
beginwhile:
        cmp eax, val2                  ; 如果非 val1 < val2
        jnl     endwhile               ; 退出循环
        inc    eax                     ; val1++;
        dec    val2                    ; val2--;
        jmp    beginwhile              ; 重复循环
endwhile:
        mov    val1, eax                ;保存 val1 的新值
```

在循环内部，EAX 是 val1 的代理（替代品），对 val1 的引用必须要通过 EAX。JNL 的使用意味着 val1 和 val2 是有符号整数。

【示例】循环内的 IF 语句嵌套

高级语言尤其善于表示嵌套的控制结构。如下 C++ 代码所示，在一个 WHILE 循环中有嵌套 IF 语句。它计算所有大于 sample 值的数组元素之和：

```c
int array[] = {10,60,20,33,72,89,45,65,72,18};
int sample  =  50;
int ArraySize = sizeof array / sizeof sample;
int index = 0;
int sum  =  0;
while( index < ArraySize )
{
    if( array[index] > sample )
    {
        sum += array[index];
    }
    index++;
}
```

在用汇编语言编写该循环之前，用下图的流程图来说明其逻辑。为了简化转换过程，并通过减少内存访问次数来加速执行，图中用寄存器来代替变量：EDX-sample, EAX=sum, ESI=index, ECX=ArraySize \( 常数 \)。标号名称也已经添加到逻辑框上。

![&#x5305;&#x542B;IF&#x8BED;&#x53E5;&#x7684;&#x5FAA;&#x73AF;](http://c.biancheng.net/uploads/allimg/190508/4-1Z50Q6415G12.gif)

### 汇编代码

从流程图生成汇编代码最简单的方法就是为每个流程框编写单独的代码。注意流程图标签和下面源代码使用标签之间的直接关系：

```text
array DWORD 10,60,20,33,72,89,45,65,72,18
ArraySize = ($ - Array) / TYPE array

.code
main PROC
    mov    eax,0                           ; 求和
    mov    edx,sample
    mov    esi,0                           ; 索引
    mov    ecx,ArraySize

L1: cmp    esi,ecx                         ; 如果 esi < ecx
    jl    L2
    jmp    L5

L2: cmp    array[esi*4], edx               ; 如果array[esi] > edx
    jg    L3
    jmp    L4
L3: add    eax,array[esi*4]

L4: inc    esi
    jmp    L1

L5: mov    sum,eax
```

## 6.19 表驱动选择

表驱动选择是用查表来代替多路选择结构的一种方法。使用这种方法，需要新建一个表，表中包含查询值和标号或过程的偏移量，然后必须用循环来检索这个表。当有大量比较操作时，这个方法最有效。

例如，下面是一个表的一部分，该表包含单字符查询值，以及过程的地址：

```text
.data
CaseTable BYTE  'A'      ;查询值
    DWORD Process_A  ;过程地址
    BYTE 'B'
    DWORD Process_B
    (etc.)
```

假设 Process\_A、Process\_B、Process\_C 和 Process\_D 的地址分别是 120h、130h、140h 和 150h。上表在内存中的存放如下图所示。

![&#x8FC7;&#x7A0B;&#x504F;&#x79FB;&#x91CF;&#x8868;](http://c.biancheng.net/uploads/allimg/190508/4-1Z50QH955917.gif)

#### 示例程序

用户从键盘输入一个字符。通过循环，该字符与表的每个表项进行比较。第一个匹配的查询值将会产生一个调用，调用对象是紧接在该查询值后面的过程偏移量。每个过程加载到 EDX 的偏移量都代表了一个不同的字符串，它将在循环中显示：

```text
; 过程偏移量表          (ProcTble.asm)

; 本程序包含了过程偏移量表格
; 使用这个表执行间接过程调用

INCLUDE Irvine32.inc
.data
CaseTable  BYTE   'A'                  ; 查询值
           DWORD   Process_A           ; 过程地址
           BYTE   'B'
           DWORD   Process_B
           BYTE   'C'
           DWORD   Process_C
           BYTE   'D'
           DWORD   Process_D
NumberOfEntries = 4

prompt BYTE "Press capital A,B,C,or D: ",0
;为每个过程定义一个单独的消息字串
msgA BYTE "Process_A",0
msgB BYTE "Process_B",0
msgC BYTE "Process_C",0
msgD BYTE "Process_D",0

.code
main PROC
    mov  edx,OFFSET prompt              ; 请求用户输入
    call WriteString
    call ReadChar                       ; 读取字符到AL
    mov  ebx,OFFSET CaseTable           ; 设 EBX 为表指针
    mov  ecx,NumberOfEntries            ; 循环计数器
L1:
    cmp  al,[ebx]                       ; 出现匹配项?
    jne  L2                             ; 否: 继续
    call NEAR PTR [ebx + 1]             ; 是: 调用过程
;这个 CALL 指令调用过程，其地址保存在 EBX+1 指向的内存位置中，像这样的间接调用需要使用 NEAR PTR 运算符
    call WriteString                    ; 显示消息
    call Crlf
    jmp  L3                             ; 推出搜索
    add  ebx,5                          ; 指向下一个表项
    loop L1                             ; 重复直到 ECX = 0

L3:
    exit
main ENDP
;下面的每个过程向EDX加载不同字符串的偏移量
Process_A PROC
    mov  edx,OFFSET msgA
    ret
Process_A ENDP

Process_B PROC
    mov  edx,OFFSET msgB
    ret
Process_B ENDP

Process_C PROC
    mov  edx,OFFSET msgC
    ret
Process_C ENDP

Process_D PROC
    mov  edx,OFFSET msgD
    ret
Process_D ENDP

END main
```

表驱动选择有一些初始化开销，但是它能减少编写的代码总量。一个表就可以处理大量的比较，并且与一长串的比较、跳转和 CALL 指令序列相比，它更加容易修改。甚至在运行时，表还可以重新配置。

## 6.20 有限状态机（FSM）与汇编语言

有限状态机（FSM）是一个根据输入改变状态的机器或程序。用图表示 FSM 相当简明， 下图中的矩形（或圆形）称为节点，节点之间带箭头的线段称为边（或弧）。

![&#x7B80;&#x5355;&#x7684;&#x6709;&#x9650;&#x72B6;&#x6001;&#x673A;](http://c.biancheng.net/uploads/allimg/190509/4-1Z50910224EG.gif)

上图给出了一个简单的例子。每个节点代表一个程序状态，每个边代表从一个状态到另一个状态的转换。一个节点被指定为初始状态，在图中用一个输入箭头指出。其余的状态可以用数字或字母来标示。一个或多个状态可以指定为终止状态，用粗框矩形表示。终止状态表示程序无出错的结束状态。

FSM 是一种被称为有向图的更一般结构的特例。有向图就是一组节点，它们用具有特定方向的边进行连接。

### 验证输入字符串

读取输入流的程序往往要通过执行一定量的错误检查来验证它们的输入。比如，编程语言编译器可以用 FSM 来扫描程序，将文字和符号转换为记号（通常是指关键字、算法运算符和标识符）。

用 FSM 来验证输入字符串时，常常是按字符进行读取。每一个字符都用图中的一条边（转换）来表示。FSM 有两种方法检测非法输入序列：

* 下一个输入字符没有对应到当前状态的任何一个转换。
* 输入已经终止，但是当前状态是非终止状态。

字符串示例现在根据下面两条原则来验证一个输入字符串：

* 该字符串必须以字母“x”开始，以字母“z”结束。
* 第一个和最后一个字符之间可以有零个或多个字母，但其范围必须是 {a,....,y}。

下图的 FSM 显示了上述语法。每一个转换都是由特定类型的输入来标识。比如，仅当从输入流中读取字母 x 时，才能完成状态 A 到状态 B 的转换。输入任何非“z”的字母，都会使得状态 B 转换为其自身。而仅当从输入流中读取字母 z 时，才会发生状态 B 到状态 C 的转换。

![&#x5B57;&#x7B26;&#x4E32;&#x7684;FSM](http://c.biancheng.net/uploads/allimg/190509/4-1Z509102340W8.gif)

如果输入流已经结束，而程序只出现了状态 A 和状态 B，那么就生成出错条件，因为只有状态 C 才能标记终止状态。下述输入字符串能被该 FSM 认可：

```text
xaabcdefgz
xz
xyyqqrrstuvz
```

### 验证有符号整数

下图表示的是 FSM 解析一个有符号整数。输入包括一个可选的前置符号，其后跟一串数字。图中没有对数字个数进行限制。

![&#x6709;&#x7B26;&#x53F7;&#x5341;&#x8FDB;&#x5236;&#x6574;&#x6570;](http://c.biancheng.net/uploads/allimg/190509/4-1Z509102425953.gif)

有限状态机很容易转换为汇编代码。图中的每个状态（A、B、C…）代表了一段有标号的程序。每个标号执行的操作如下：

1\) 调用输入程序读入下一个输入字符。

2\) 如果是终止状态，则检查用户是否按下 Enter 键来结束输入。

3\) 一个或多个比较指令检查从状态发岀的所有可能的转换。每个比较指令后面跟一个条件跳转指令。

比如，在状态 A，如下代码读取下一个输入字符并检查到状态 B 的可能的转换：

```text
StateA:
        Cal1 Getnext                          ;读取下一个字符，并送入 AL
        cmp    al, '+'                        ;前置+ ?
        je    StateB                          ;到状态 b
        cmp    al, '-'                        ;前置 - ?
        je    StateB                          ;到状态 B
        call    IsDigit                       ;如果 AL 包含数字，则 ZF = 1
        jz    StateC                          ;到状态 C
        call    DisplayErrorMsg               ;发现非法输入
        jmp Quit
```

下面来更详细地检查这段代码。首先，代码调用 Getnext，从控制台输入读取下一个字符，送入 AL 寄存器。接着检查前置 + 或 -，先将 AL 的值与符号“+”进行比较，如果匹配，就发生到标号 StateB 的跳转：

```text
StateA:
        call Getnext                         ;读取下一个字符，并送入 al
        cmp al, ' + '                        ;前置 + ?
        je StateB                            ;到状态 B
```

现在，再次查看上图，发现只有输入 + 或 - 时，才发生状态 A 到状态 B 的转换。所以，代码还需检查减号：

```text
cmp al, '-'                                  ;前置 - ？
je StateB                                    ;到状态 B
```

如果无法发生到状态 B 的转换，就可以检查 AL 寄存器中是否为数字，这可以导致到状态 C 的转换。调用 IsDigit 子程序，当 AL 包含数字时，零标志位置 1：

```text
call IsDigit                                 ;如果AL包含数字，贝U ZF=1
jz StateC                                    ;到状态 C
```

最后，状态 A 没有其他可能的转换。如果发现 AL 中的字符既不是前置符号，又不是数字，程序就会调用 DisplayErrorMsg （在控制台上显示一条错误消息）子程序，并跳转到标号 Quit 处：

```text
call DisplayErrorMsg                         ;发现非法输入
jmp Quit
```

标号 Quit 标识程序的出口，位于主程序的结尾：

```text
Quit:
    call Crlf
    exit
main ENDP
```

### 完整的有限状态机程序

如下程序实现上图所示的有符号整数 FSM：

```text
; 有限状态机              (Finite.asm)
INCLUDE Irvine32.inc

ENTER_KEY = 13
.data
InvalidInputMsg BYTE "Invalid input",13,10,0

.code
main PROC
    call Clrscr

StateA:
    call    Getnext               ; 读取下一个字符，并送入AL
    cmp    al,'+'                 ; 前置+ ?
    je    StateB                  ; 到状态 B
    cmp    al,'-'                 ; 前置 - ?
    je    StateB                  ; 到状态 B
    call    IsDigit               ; 如果 AL 包含数字 ，则 ZF = 1
    jz    StateC                  ; 到状态 C
    call    DisplayErrorMsg       ; 发现非法输入
    jmp    Quit

StateB:
    call    Getnext               ; 读取下一个字符，并送入AL
    call    IsDigit               ; 如果AL包含数字，则 ZF = 1
    jz    StateC
    call    DisplayErrorMsg       ; 发现非法输入
    jmp    Quit

StateC:
    call    Getnext               ; 读取下一个字符，并送入AL
    call    IsDigit               ; 如果AL包含数字，则 ZF = 1
    jz    StateC
    cmp    al,ENTER_KEY           ; 按下Enter键?
    je    Quit                    ; 是：Quit
    call    DisplayErrorMsg       ; 否: 发现非法输入
    jmp    Quit

Quit:
    call    Crlf
    exit
main ENDP

;-----------------------------------------------
Getnext PROC
;
; 从标准输入读取一个字符
; 接收: 无
; 返回: 字符保存在AL中
;-----------------------------------------------
     call ReadChar            ; 从键盘输入
     call WriteChar           ; 显示在屏幕上
     ret
Getnext ENDP

;-----------------------------------------------
DisplayErrorMsg PROC
;
; 显示一个错误消息以表示
; 输入流中包含非法输入
; 接收: 无.
; 返回: 无
;-----------------------------------------------
     push  edx
     mov      edx,OFFSET InvalidInputMsg
     call  WriteString
     pop      edx
     ret
DisplayErrorMsg ENDP
END main
```

### IsDigit子程序

有限状态机示例程序调用 IsDigit 子程序，该子程序属于本教程的链接库。现在来看看 IsDigit 的源程序，程序把 AL 寄存器作为输入，其返回值设置零标志位：

```text
;----------------------------------------------------
IsDigit PROC
;
;确定 AL 中的字符是否为有效的十进制数字。
;接收：AL= 字符
;返回：若 AL 为有效的十进制字符，ZF=1;否则，ZF=0
;---------------------------------------------------
        cmp    al,'0'
        jb    ID1                                 ;跳转发生，ZF=0
        cmp    al, '9'
        ja    ID1                                 ;跳转发生，ZF = 0
        test    ax, 0                             ;设置 ZF=1
ID1: ret
IsDigit ENDP
```

在查看 IsDigit 的代码之前，先回顾十进制数字的十六进制 ASCII 码，如下表所示。由于这些值是连续的，因此，只需要检查第一个和最后一个值：

| 字符 | '0' | '1' | '2' | '3' | '4' | '5' | '6' | '7' | '8' | '9' |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| ASCII 码（十六进制） | 30 | 31 | 32 | 33 | 34 | 35 | 36 | 37 | 38 | 39 |

IsDigit 子程序中，开始的两条指令将 AL 寄存器中字符的值与数字 0 的 ASCII 码进行比较。如果字符的 ASCII 码小于 0 的 ASCII 码，程序跳转到标号 ID1：

```text
cmp al, '0'
jb ID1                       ;跳转发生，ZF=0
```

但是有人可能会问了，如果 JB 将控制传递给标号 ID1，那么，怎么知道零标志位的状态呢？答案就在 CMP 指令的执行方式里——它执行一个隐含的减法操作，从 AL 寄存器的字符中减去 0 的 ASCII 码（30h）。如果 AL 中的值较小，那么进位标志位置 1，零标志位清除（你可能想用调试器来单步执行这段代码来验证这个事实）。JB 指令的目的是，当 CF=1 且 ZF=0 时，将控制传递给一个标号。

接下来，IsDigit 子程序代码把 AL 与数字 9 的 ASCII 码进行比较。如果 AL 的值较大，代码跳转到同一个标号：

```text
cmp al, '9'
ja ID1                ;跳转发生，ZF=0
```

如果 AL 中字符的 ASCII 码大于数字 9 的 ASCII 码（39h），清除进位标志位和零标志位。这也正好是使得 JA 指令将控制传递到目的标号的标志位组合。

如果没有跳转发生（JA 或 JE），又假设 AL 中的字符确实是一个数字，则插入一条指令确保将零标志位置 1。将 0 与任何数值进行 test 操作，就意味着执行一次隐含的与全 0 的 AND 运算。其结果必然为 0：

```text
test ax, 0     ; 置 ZF=1
```

前面 IsDigit 中的 JA 和 JB 指令跳转到了 TEST 指令后面的标号。所以，如果发生跳转，零标志位将清零。下面再次给出完整的过程：

```text
Isdigit PROC
    cmp al,'0'
    jb ID1             ;若跳转发生，则 ZF=0
    cmp al,'9'
    ja ID1             ;若跳转发生，则 ZF=0
    test ax,0          ;置 zf=1
ID1: ret
Isdigit ENDP
```

在实时或高性能应用中，程序员常常利用硬件特性的优势，来对其代码进行充分优化。IsDigit 过程就是这种方法的例子，它利用 JB、JA 和 TEST 对标志的设置，实际上返回的是一个布尔结果。

## 6.21 条件控制流伪指令

32 位模式下，MASM 包含了一些高级条件控制流伪指令（conditional control flow directives），这有助于简化编写条件语句。遗憾的是，这些伪指令不能用于 64 位模式。

对程序进行汇编之前，汇编器执行的是预处理步骤。在这个步骤中，汇编器要识别伪指令，如：.CODE、.DATA，以及一些用于条件控制流的伪指令。下表列出了这些伪指令。

| 伪指令 | 说明 |
| :--- | :--- |
| .BREAK | 生成代码终止 .WHILE 或 .REPEAT 块 |
| .CONTINUE | 生成代码跳转到 .WHILE 或 .REPEAT 块的顶端 |
| .ELSE | 当 .IF 条件不满足时，开始执行的语句块 |
| .ELSEIF condition | 生成代码测试 condition，并执行其后的语句，直到碰到一个 .ENDIF 或另一个 .ELSEIF 伪指令 |
| .ENDIF | 终止 .IF、.ELSE 或 .ELSEIF 伪指令后面的语句块 |
| .ENDW | 终止 .WHILE 伪指令后面的语句块 |
| .IF condition | 如果 condition 为真，则生成代码执行语句块 |
| .REPEAT | 生成代码重复执行语句块，直到条件为真 |
| .UNTIL condition | 生成代码重复执行 .REPEAT 和 .UNTIL 伪指令之间的语句块，直到 condition 为真 |
| .UNTILCXZ | 生成代码重复执行 .REPEAT 和 .UNTILCXZ 伪指令之间的语句块，直到 CX 为零 |
| .WHILE condition | 当 condition 为真时，生成代码执行 .WHILE 和 .ENDW 伪指令之间的语句块 |

## 6.22 .IF、.ELSE、.ELSEIF、.ENDIF伪指令

.IF、.ELSE、.ELSEIF 和 .ENDIF 伪指令使得程序员易于对多分支逻辑进行编码。它们让汇编器在后台生成 CMP 和条件跳转指令，这些指令显示在输出列表文件中。语法如下所示：

```text
.IF conditionl
  statements
[.ELSEIF condition2
  statements ]
[.ELSE
  statements ]
.ENDIF
```

方括号表示 .ELSEIF 和 .ELSE 是可选的，而 .IF 和 .ENDIF 则是必需的。condition（条件）是布尔表达式，使用与 [C++](http://c.biancheng.net/cplus/) 和 [Java](http://c.biancheng.net/java/) 相同的运算符 \( 比如：&lt;、&gt;、== 和 !=\)。表达式在运行时计算。下面的例子给出了一些有效的条件，使用的是 32 位寄存器和变量：

```text
eax > 10000h
val1 <= 100
val2 == eax
val3 != ebx
```

下面的例子给出的是复合条件：

```text
(eax > 0) && (eax > 10000h)
(val1 <= 100) || (val2 <= 100)
(val2 != ebx) && !CARRY?
```

下表列出了所有的关系和逻辑运算符。

| 运算符 | 说明 |
| :--- | :--- |
| expr1 == expr2 | 若 expr1 等于 expr2，则返回“真” |
| expr1 != expr2 | 若 expr1 不等于 expr2，则返回“真” |
| expr1 &gt; expr2 | 若 expr1 大于 expr2，则返回"真” |
| expr1 ≥ expr2 | 若 expr1 大于等于 expr2，则返回“真” |
| expr1 &lt; expr2 | 若 expr1 小于 expr2，则返回“真” |
| expr1 ≤ expr2 | 若 expr1 小于等于 expr2，则返回“真” |
| !expr1 | 若 expr 为假，则返回“真” |
| expr1expr2 | 对 expr1 和 expr2 执行逻辑 AND 运算 |
| expr1 \|\| expr2 | 对 1xprl 和 expr2 执行逻辑 OR 运算 |
| expr1 & expr2 | 对 expr1 和 expr2 执行按位 AND 运算 |
| CARR1? | 若进位标志位置 11则返回“真” |
| OVERFLOW ? | 若溢出标志位置 1，则返回“真” |
| PARITY ? | 若奇偶标志位置 1，则返回“真” |
| SIGN ? | 若符号标志位置 1，则返回“真” |
| ZERO ? | 若零标志位置 1，则返回“真” |

在使用 MASM 条件伪指令之前，一定要彻底了解怎样用纯[汇编语言](http://c.biancheng.net/asm/)实现条件分支指令。此外，在包含条件伪指令的程序汇编时，要查看列表文件以确认 MASM 生成的代码确实是编程者所需要的。

### 生成 ASM 代码

当使用如 .IF 和 .ELSE 一样的高级伪指令时，汇编器将为程序员编写代码。例如，编写一条 .IF 伪指令来比较 EAX 与变量 val1：

```text
mov eax,6
.IF eax > val1
    mov result,1
.ENDIF
```

假设 val1 和 result 是 32 位无符号整数，当汇编器读到前述代码时，就将它们扩展为下述汇编语言指令，用 Visual Studio 调试器运行程序时可以查看这些指令，操作为：右键点击, 选择 Go To Disassembly。

```text
    mov eax,6
    cmp eax,val1
    jbe @C0001            ;无符号数比较跳转
    mov result, 1
@C0001:
```

标号名 @C0001 由汇编器创建，这样可以确保同一个过程中的所有标号都具有唯一性。

要控制 MASM 生成代码是否显示在源列表文件中，可以在 Visual Studio 中配置 Project 的属性。步骤如下：在 Project 菜单中，选择 Project Properties，选择 Microsoft Macro Assembler，选择 Listing File，再设置 Enable Assembly Generated Code Listing 为 Yes。

### 有符号数和无符号数的比较

当使用 .IF 伪指令来比较数值时，必须认识到 MASM 是如何生成条件跳转的。如果比较包含了一个无符号变量，则在生成代码中插入一条无符号条件跳转指令。如下还是前面的例子，比较 EAX 和无符号双字变量 val1：

```text
.data
val1 DWORD 5
result DWORD ?
.code
    mov eax,6
    .IF eax > val1
        mov result,1
    .ENDIF
```

汇编器用 JBE（无符号跳转）指令对其进行扩展：

```text
mov eax,6
cmp eax,val1
    jbe @C0001             ;无符号比较跳转
    mov result,1
@C0001：
```

#### 1\) 有符号数比较

如果 .IF 伪指令比较的是有符号变量，则在生成代码中插入一条有符号条件跳转指令。例如，val2 为有符号双字：

```text
.data
val2 SDWORD -1
result DWORD ?
.code
    mov eax,6
    .IF eax > val2
        mov result,1
    .ENDIF
```

因此，汇编器用 JLE 指令生成代码，即基于有符号比较的跳转：

```text
    mov eax,6
    cmp eax,val2
    jle @C0001               ;有符号比较跳转
    mov result,1
@C0001：
```

#### 2\) 寄存器比较

那么，现在可能会有一个问题：如果是两个寄存器进行比较，情况又是怎样的？显然，汇编器无法确定寄存器中的数值是有符号的还是无符号的：

```text
mov eax,6
mov ebx,val2
.IF eax > ebx
    mov result,1
.ENDIF
```

下面生成的代码表示汇编器将其默认为无符号数比较（注意使用的是 JBE 指令）：

```text
    mov eax, 6
    mov ebx,val2
    cmp eax, ebx
    jbe @C0001
    mov result,1
@C0001:
```

### 复合表达式

很多复合布尔表达式使用逻辑 OR 和 AND 运算符。用 .IF 伪指令时，符号 \|\| 表示的是逻辑 OR 运算符：

```text
.IF expression1 || expression2
  statements
.ENDIF
```

同样，符号 && 表示的是逻辑 AND 运算符：

```text
.IF expression1 && expression2
  statements
.ENDIF
```

下面的程序示例中将使用逻辑 OR 运算符。

#### 1\) SetCursorPosition 示例

下例给出的 SetCursorPosition 过程，根据两个输入参数 DH 和 DL，执行范围检查。Y 坐标（DH）范围必须为 0〜24。X 坐标（DL）范围必须为 0〜79。不论发现哪个坐标超出范围，都显示一条错误消息：

```text
SetCursorPosition PROC
; 设置光标位置
; 接收: DL = X坐标, DH = Y坐标
; 检查 DL 和 DH 的范围
; 返回：无
;------------------------------------------------
.data
BadXCoordMsg BYTE "X-Coordinate out of range!",0Dh,0Ah,0
BadYCoordMsg BYTE "Y-Coordinate out of range!",0Dh,0Ah,0
.code
    .IF (DL < 0) || (DL > 79)
       mov  edx,OFFSET BadXCoordMsg
       call WriteString
       jmp  quit
    .ENDIF
    .IF (DH < 0) || (DH > 24)
       mov  edx,OFFSET BadYCoordMsg
       call WriteString
       jmp  quit
    .ENDIF
    call Gotoxy

quit:
    ret
SetCursorPosition ENDP
```

MASM 对 SetCursorPosition 进行预处理时，生成代码如下：

```text
.code
;.IF (dl < 0) || (dl > 79)
    cmp dl, OOOh
    jb @C0002
    cmp dl, 04Fh
    jbe @C0001
@C0002:
    mov edx,OFFSET BadXCoordMsg
    call WriteString
    jmp quit
;.ENDIF
@C0001:
;.IF (dh < 0) || (dh > 24)
    cmp dh, OOOh
    jb @COOO5
    cmp    dh, 018h
    jbe @C0004
@COOO5:
    mov edx,OFFSET BadYCoordMsg
    call WriteString
    jmp quit
;.ENDIF
@C0004:
    call Gotoxy
quit:
    ret
```

#### 2\) 大学注册示例

假设有一个大学生想要进行课程注册。现在用两个条件来决定该生是否能注册：第一个条件是学生的平均成绩，范围为 0〜400，其中 400 是可能的最高成绩；第二个条件是学生期望获得的学分。可以使用多分支结构，包括 .IF、.ELSEIF 和 .ENDIF。示例如下。

```text
.data
TRUE = 1
FALSE = 0
gradeAverage  WORD 275    ; 要检查的数值
credits       WORD 12     ; 要检查的数值
OkToRegister  BYTE ?

.code
main PROC

    mov OkToRegister,FALSE

    .IF gradeAverage > 350
       mov OkToRegister,TRUE
    .ELSEIF (gradeAverage > 250) && (credits <= 16)
       mov OkToRegister,TRUE
    .ELSEIF (credits <= 12)
       mov OkToRegister,TRUE
    .ENDIF
```

汇编器生成的相应代码如下所示，用 Microsoft Visual Studio 调试器的 Dissassembly 窗口可以查看该表。（为了便于阅读，已经对其进行了一些整理。）

```text
    mov byte ptr OkToRegister,FALSE
    cmp word ptr gradeAverage,350
    jbe @C0006
    mov byte ptr OkToRegister,TRUE
    jmp @C0008
@C0006:
    cmp word ptr gradeAverage,250
    jbe @C0009
    cmp word ptr credits,16
    ja  @COOO9
    mov byte ptr OkToRegister,TRUE
    jmp @C0008
@C0009:
    cmp word ptr credits,12
    ja  @C0008
    mov byte ptr OkToRegister,TRUE
@COOO8：
```

汇编程序时，如果使用 /Sg 命令行就可以在源列表文件中显示 MASM 生成代码。被定义常量的大小（如当前代码示例中的 TRUE 和 FALSE）为 32 位。所以，把一个常量送入 BYTE 类型地址时，MASM 会插入 BYTE PTR 运算符。

## 6.23 用.REPEAT和.WHILE伪指令实现循环

除了用 CMP 和条件跳转指令外，.REPEAT 和 .WHILE 伪指令还提供了另一种方法来编写循环。它们可以使用之前由《[.IF伪指令](http://c.biancheng.net/view/3585.html)》一节中关系和逻辑运算符表所列出的条件表达式。

.REPEAT 伪指令执行循环体，然后测试 .UNTIL 伪指令后面的运行时条件：

```text
.REPEAT
  statements
.UNTIL condition
```

.WHILE 伪指令在执行循环体之前测试条件：

```text
.WHILE condition
  statements
.ENDW
```

示例：下述语句使用 .WHILE 伪指令显示数值 1 到 10。循环之前，计数器寄存器 \(EAX\) 被初始化为 0。之后，循环体内的第一条语句将 EAX 加 1。当 EAX 等于 10 时，.WHILE 伪指令将分支到循环体外。

```text
mov eax,0
.WHILE eax < 10
    inc eax
    call WriteDec
    call Crlf
.ENDW
```

下述语句使用 .REPEAT 伪指令显示数值 1 到 10：

```text
mov eax,0
.REPEAT
    inc eax
    call WriteDec
    call Crlf
.UNTIL eax == 10
```

【示例】：含 IF 语句的循环

《[使用汇编语言实现WHILE循环](http://c.biancheng.net/view/3579.html)》一节中展示了如何编写[汇编语言](http://c.biancheng.net/asm/)代码来实现 WHILE 循环嵌套 IF 语句。伪代码如下：

```c
while( op1 < op2 )
{
    op1++;
    if( op1 == op3 )
        X = 2;
    else
        X = 3;
}
```

下面用 .WHILE 和 .IF 伪指令实现这段伪代码。由于 op1、op2 和 op3 是变量，为了避免任何指令出现两个内存操作数，它们被送入寄存器：

```text
.data
X DWORD 0
op1 DWORD 2     ;被检测的数据
op2 DWORD 4     ;被检测的数据
op3 DWORD 5     ;被检测的数据
.code
    mov eax, op1
    mov ebx, op2
    mov ecx, op3
    .WHILE eax < ebx
        inc eax
        .IF eax == ecx
            mov X,2
        .ELSE
            mov X,3
        .ENDIF
    .ENDW
```

