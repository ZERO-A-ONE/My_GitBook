---
description: >-
  大多数程序员不会用汇编语言编写大型程序，因为这将花费相当多的时间。反之，高级语言则隐藏了会减缓项目开发进度的细节。但是汇编语言仍然广泛用于配置硬件驱动器，以及优化程序速度和代码量。本章将重点关注汇编语言和高级编程语言之间的接口或连接。本章主要展示了如何在
  C++ 中编写内联汇编代码，以及如何把 32 位汇编语言模块链接到 C++ 程序中。最后，将说明如何在汇编程序中调用C库函数。
---

# 13 高级语言接口

## 13.1 高级语言调用汇编语言的接口规范

从高级语言中调用汇编过程时，需要解决一些常见的问题。

首先，一种语言使用的命名规范（naming convention）是指与变量和过程命名相关的规则和特性。比如，一个需要回答的重要问题是：汇编器或编译器会修改目标文件中的标识符名称吗？如果是，如何修改？

其次，段名称必须与高级语言使用的名称兼容。

第三，程序使用的内存模式（微模式、小描述、紧凑模式、中模式、大模式、巨模式，或平坦模式）决定了段大小（16 或 32 位），以及调用或引用是近（同一段内）还是远（不同段之间）。

#### 调用规范

调用规范（calling convention）是指调用过程的底层细节。下面列出了需要考虑的细节信息：

* 调用过程需要保存哪些寄存器
* 传递参数的方法：用寄存器、用堆栈、共享内存，或者其他方法
* 主调程序调用过程时，参数传递的顺序
* 参数传递方法是传值还是传引用
* 过程调用后，如何恢复堆栈指针
* 函数如何向主调程序返回结果

#### 命名规范与外部标识符

当从其他语言程序中调用汇编过程时，外部标识符必须与命名规范（命名规则）兼容。外部标识符（external identifier）是放在模块目标文件中的名称，链接器使得这些名称能够被其他程序模块使用。链接器解析对外部标识符的引用，但是仅适用于命名规范一致的情况。

例如，假设 C 程序 Main.c 调用外部过程 ArraySum。如下图所示，C 编译器自动保留大小写，并为外部名称添加前导下划线，将其修改为 \_ArraySum：

![img](http://c.biancheng.net/uploads/allimg/190529/4-1Z529135U4113.gif)

Array.asm 模块用[汇编语言](http://c.biancheng.net/asm/)编写，由于其 .MODEL 伪指令使用的选项为 Pascal 语言，因此输出 ArraySum 过程的名称就是 ARRAYSUM。由于两个输出的名称不同，因此链接器无法生成可执行程序。

早期编程语言，如 COBOL 和 PASCAL，其编译器一般将标识符全部转换为大写字母。近期的语言，如 C、[C++](http://c.biancheng.net/cplus/) 和 [Java](http://c.biancheng.net/java/) 则保留了标识符的大小写。

此外，支持函数重载的语言（如 C++）还使用名称修饰 \(name decoration\) 的技术为函数名添加更多字符。比如，若函数名为 MySub \(int n, double b\)，则其输出可能为 MySub\#int\#double。

在汇编语言模块中，通过 .MODEL 伪指令选择语言说明符来控制大小写。

#### 段名称

汇编语言过程与高级语言程序链接时，段名称必须是兼容的。本章使用 Microsoft 简化段伪指令 .CODE、.STACK 和 .DATA，它们与 Microsoft C++ 编译器生成的段名称兼容。

#### 内存模式

主调程序与被调过程使用的内存模式必须相同。比如，实地址模式下可选择小模式、中模式、紧凑模式、大模式和巨模式。保护模式下必须使用平坦模式。

## 13.2 .MODEL伪指令：确定程序的特性

16 位和 32 位模式中，MASM 使用 .MODEL 伪指令确定若干重要的程序特性：内存模式类型、过程命名模式以及参数传递规则。若汇编代码被其他编程语言程序调用，那么后两者就尤其重要。

.MODEL 伪指令的语法如下：

```text
.MODEL memorymodel [,modeloptions]
```

#### MemoryModel

下表列出了 memorymodel 字段可选择的模式。除了平坦模式之外，其他所有模式都可以用于 16 位实地址编程。

| 模式 | 说明 |
| :--- | :--- |
| 微模式 | 一个既包含代码又包含数据的段。文件扩展名为 .com 的程序使用该模式 |
| 小模式 | 一个代码段和一个数据段。默认情况下，所有代码和数据都为近属性 |
| 中模式 | 多个代码段，一个数据段 |
| 紧凑模式 | 一个代码段，多个数据段 |
| 大模式 | 多个代码段和数据段 |
| 巨模式 | 与大模式相同，但是各个数据项可以大于单个段 |
| 平坦模式 | 保护模式。代码与数据使用 32 位偏移量。所有的数据和代码（包括系统资源）都在一个 32 位段内 |

32 位程序使用平坦内存模式，其偏移量为 32 位，代码和数据最大可达 4GB。比如，Irvine32.inc 文件包含了如下 .MODEL 伪指令：

```text
.model flat, STDCALL
```

#### ModelOptions

.MODEL 伪指令中的 ModelOptions 字段可以包含一个语言说明符和一个栈距离。语言说明符指定过程与公共符号的调用和命名规范。栈距离可以是 NEARSTACK（默认值）或者 FARSTACK。

#### 1\) 语言说明符

伪指令 .MODEL 有几种不同的可选语言说明符，其中的一些很少使用（比如 BASIC、FORTRAN 和 PASCAL）。反之，C 和 STDCALL 则十分常见。结合平坦内存模式，示例如下：

```text
.model flat, C
.model flat, STDCALL
```

语言说明符 STDCALL 用于 Windows 系统函数调用。本章在链接汇编代码和 C 与 [C++](http://c.biancheng.net/cplus/) 程序时，使用 C 语言说明符。

#### 2\) STDCALL

STDCALL 语言说明符将子程序参数按逆序（从后往前）压入堆栈。为了便于说明，首先用高级语言编写如下函数调用：

```c
AddTwo(5, 6);
```

若 STDCALL 被选为语言说明符，则等效的[汇编语言](http://c.biancheng.net/asm/)代码如下：

```text
push 6
push 5
call AddTwo
```

另一个重要的考虑是，过程调用后如何从堆栈中移除参数。STDCALL 要求在 RET 指令中带一个常量操作数。返回地址从堆栈中弹出后，该常数为 RET 执行与 ESP 相加的数值：

```text
AddTwo PROC
  push ebp
  mov ebp,esp
  mov eax, [ebp + 12]  ;第二个参数
  add eax, [ebp + 8]    ;第一个参数
  pod ebp
ret 8         ;清除堆栈
AddTwo ENDPP
```

堆栈指针加上 8 后，就指回了主调程序参数入栈之前指针的位置。

最后，STDCALL 通过将输出（公共）过程名保存为如下格式来修改这些名称：

```text
_name@nn
```

前导下划线添加到过程名，@ 符号后面的整数指定了过程参数的字节数（向上舍入到 4 的倍数）。例如，假设过程 AddTwo 带有两个双字参数，那么汇编器传递给链接器的名称就为 \_AddTwo@8。

Microsoft 链接器是区分大小写的，因此 \_MYSUB@8 和 \_MySub@8 是两个不同的名称。要查看 OBJ 文件中所有的过程名，使用 Visual Studio 中的 DUMPBIN 工具，选项为 /SYMBOLS。

#### 3\) C 说明符

和 STDCALL 一样，C 语言说明符也要求将过程参数按从后往前的顺序压入堆栈。对于过程调用后从堆栈中移除参数的问题，C 语言说明符将这个责任留给了主调方。在主调程序中，ESP 与一个常数相加，将其再次设置为参数入栈之前的位置：

```text
push 6      ;第二个参数
push 5      ;第一个参数
call AddTwo
add esp,8    ;清除堆栈
```

C 语言说明符在外部过程名的前面添加前导下划线。示例如下:

```text
_AddTwo
```

## 13.3 查看C语言/C++编译器生成的汇编语言代码

长久以来，C 和 [C++](http://c.biancheng.net/cplus/) 编译器都会生成[汇编语言](http://c.biancheng.net/asm/)源代码，但是程序员通常看不到。这是因为，汇编语言代码只是产生可执行文件过程的一个中间步骤。幸运的是，大多数编译器都可以应要求生成汇编语言源代码文件。 例如，下表列出了 Visual Studio 控制汇编源代码输出的命令行选项。

| 命令行 | 列表文件内容 |
| :--- | :--- |
| /FA | 仅汇编文件 |
| /FAc | 汇编文件与机器码 |
| /FAs | 汇编文件与源代码 |
| /FAcs | 汇编文件、机器码和源代码 |

检查编译器生成的代码文件有助于理解底层信息，比如堆栈帧结构、循环和逻辑编码，并且还有可能找到低级编程错误。另一个好处是更加便于发现不同编译器生成代码的差异。

现在来看看 C++ 编译器生成优化代码的一种方法。由于是第一个例子，因此先编写一个简单的 C 方法 Array Sum，并在 Visual Studio 2012 中进行编译，其设置如下：

* Optimization=Disabled \( 使用调试器时需要 \)
* Favor Size or Speed=Favor fast code
* Assembler Output=Assembly With Source Code

下面是用 ANSI C 编写的 arraysum 源代码：

```c
int arraySum( int array[], int count )
{
    int i;
    int sum = 0;
    for(i = 0; i < count; i++)
        sum += array[i];
    return sum;
}
```

现在来查看由编译器生成的 arraysum 的汇编代码，如下所示。

```text
_sum$ = -8        ; size = 4
_i$ = -4          ; size = 4
_array$ = 8       ; size = 4
_count$ = 12      ; size = 4
_arraySum PROC    ; COMDAT

;4    : {

    push ebp
    mov    ebp, esp
    sub    esp, 72    ; 00000048H
    push ebx
    push esi
    push edi

;5    : int i;
;6    : int sum = 0;

    mov DWORD PTR _sum$[ebp], 0

;7    :
;8    : for(i =    0; i < count; i++)

    mov DWORD PTR _i$[ebp], 0
    jmp SHORT $LN3@arraySum
$LN2@arraySum:
    mov eax, DWORD PTR _i$[ebp]
    add eax, 1
    mov DWORD PTR _i$[ebp], eax
$LN3@arraySum:
    mov eax, DWORD PTR _i$[ebp]
    cmp eax, DWORD PTR _count$[ebp]
    jge SHORT $LN1@arraySum

;9    : sum += array[i];

    mov eax, DWORD PTR _i$[ebp]
    mov ecx, DWORD PTR _array$[ebp]
    mov edx, DWORDPTR _sum$[ebp]
    add edx, DWORD PTR [ecx+eax*4]
    mov DWORD PTR _sum$[ebp], edx
    jmp SHORT $LN2@arraySum
$LNl@arraySum:

;10    :
;11    : return sum;

    mov eax, DWORD PTR _sum$[ebp]

;12    : }

    pop edi
    pop esi
    pop ebx
    mov esp, ebp
    pop ebp
    ret 0
_arraySum ENDP
```

1〜4 行定义了两个局部变量 \(sum 和 i\) 的负数偏移量，以及输入参数 array 和 count 的正数偏移量：

```text
_sum$ = -8    ; size = 4
_i$ = -4       ; size = 4
_array$ = 8    ; size = 4
_count$ = 12   ; size = 4
```

9〜10 行设置 ESP 为帧指针：

```text
push ebp
mov ebp,esp
```

之后，11〜14 行从 ESP 中减去 72，为局部变量预留堆栈空间。同时，把将会被函数修改的三个寄存器保存到堆栈。

```text
sub esp, 72
push ebx
push esi
push edi
```

19 行把局部变量 sum 定位到堆栈帧，并将其初始化为 0。由于符号 \_sum$ 定义为数值 -8，因此它就位于当前 EBP 下面 8 个字节的位置：

```text
mov DWORD PTR _sum$[ebp],0
```

24 和 25 行将变量 i 初始化为 0，再转移到 30 行，跳过后面循环计数器递增的语句：

```text
mov DWORD PTR _i$[ebp], 0
jmp SHORT $LN3@arraySum
```

26〜29 行标记循环开端以及循环计数器递增的位置。从 C 源代码来看，递增操作 \(i++\) 是在循环末尾执行，但是编译器却将这部分代码移到了循环顶部：

```text
$LN2@arraySum:
  mov eax, DWORD PTR _i$[ebp]
  add eax, 1
  mov DWORD PTR _i$[ebp], eax
```

30〜33 行比较变量 i 和 count，如果 i 大于或等于 count，则跳岀循环：

```text
$LN3@arraySum:
  mov eax, DWORD PTR _i$[ebp]
  cmp eax, DWORD PTR _count$[ebp]
  jge SHORT $LN1@arraySum
```

37〜41 行计算表达式 sum+=array\[i\]。Array\[i\] 复制到 ECX，sum 复制到 EDX，执行加法运算后，EDX 的内容再复制回 sum：

```text
mov eax, DWORD PTR _i$[ebp]
mov ecx, DWORD PTR _array$[ebp]  ; array [i]
mov edx, DWORD PTR _sum$[ebp]   ; sum
add edx, DWORD PTR [ecx+eax*4]
mov DWORD PTR _sum$[ebp], edx
```

42 行将控制转回循环顶部：

```text
jmp SHORT $LN2@arraySum
```

43 行的标号正好位于循环之外，该位置便于作为循环结束时进行跳转的目标地址：

```text
$LN1@arraySum:
```

48 行将变量 sum 送入 EAX，准备返回主调程序。52〜56 行恢复之前被保存的寄存器，其中，ESP 必须指向主调程序在堆栈中的返回地址。

```text
mov eax, DWORD PTR _sum$[ebp]

;  12 : }

pop edi
pop esi
pop ebx
mov esp, ebp
pop ebp
ret 0
_arraySum ENDP
```

可以写出比上例更快的代码，这种想法不无道理。上例中的代码是为了进行交互式调试，因此为了可读性而牺牲了速度。如果针对确定目标编译同样的程序，并选择完全优化，那么结果代码的执行速度将会非常快，但同时，程序对人类而言基本上是无法阅读和理解的。

#### 调试器设置

用 Visual Studio 调试 C 和 C++ 程序时，若想查看汇编语言源代码，就在 Tools 菜单中选择 Options 以显示如下图的对话框窗口，再选择箭头所指的选项。上述设置要在启动调试器之前完成。接着，在调试会话开始后，右键点击源代码窗口，从弹出菜单中选择 Go to Disassembly。

![&#x542F;&#x52A8;VS&#x7684;&#x5730;&#x5740;&#x7EA7;&#x8C03;&#x8BD5;](http://c.biancheng.net/uploads/allimg/190529/4-1Z5291A414Q7.gif)

本章目标是熟悉由 C 和 C++ 编译器产生的最直接和简单的代码生成例子。此外，认识到编译器有多种方法生成代码也是很重要的。比如，它们可以将代码优化为尽可能少的机器代码字节。或者，可以尝试生成尽可能快的代码，即使要用大量机器代码字节来输出结果 \( 常见的情况 \)。

最后，编译器还可以在代码量和速度的优化间进行折中。为速度进行优化的代码可能包含更多指令，其原因是，为了追求更快的执行速度会展开循环。机器代码还可以拆分为两部分以便利用双核处理器，这些处理器能同时执行两条并行代码。

## 13.4 Visual C++ \_\_asm伪指令：C语言/C++内嵌汇编语言代码

内嵌汇编代码 \(inline assembly code\) 是指直接插入高级语言程序中的汇编源代码。大多数 C 和 [C++](http://c.biancheng.net/cplus/) 编译器都支持这一功能。

本节将展示如何在运行于 32 位保护模式，并采用平坦内存模式的 Microsoft Visual C++ 中编写内嵌汇编代码。其他高级语言编译器也支持内嵌汇编代码，但其语法会发生变化。

内嵌汇编代码是把汇编代码编写为外部模块的一种直接替换方式。编写内嵌代码最突岀的优点就是简单性，因为不用考虑外部链接，命名以及参数传递协议等问题。

但使用内嵌汇编代码最大的缺点是缺少兼容性。高级语言程序针对不同目的平台进行编译时，这就成了一个问题。比如，在 Intel Pentium 处理器上运行的内嵌汇编代码就不能在 RISC 处理器上运行。

一定程度上，在程序源代码中插入条件定义可以解决这个问题，插入的定义针对不同目标系统可以启用函数的不同版本。不过，容易看出，维护仍然是个问题。另一方面，外部汇编过程的链接库容易被为不同目标机器设计的相似链接库所代替。

### \_\_asm 伪指令

在 Visual C++ 中，伪指令 \_\_asm 可以放在一条语句之前，也可以放在一个汇编语句块（称为 asm 块）之前。语法如下：

```c
__asm statement

__asm {
  statement-1
  statement-2
  ....
  statement-n
}
```

> 提示：在“asm”的前面有两个下划线。

#### 注释

注释可以放在 asm 块内任何语句的后面，使用汇编语法或 C/C++ 语法。Visual C++ 手册建议不要使用汇编风格的注释，以防与 C 宏混淆，因为 C 宏会在单个逻辑行上进行扩展。下面是可用注释的例子：

```text
mov esi,buf ; initialize index register
mov esi,buf // initialize index register
mov esi,buf /* initialize index register */
```

#### 特点

编写内嵌汇编代码时允许：

* 使用 x86 指令集内的大多数指令。
* 使用寄存器名作为操作数。
* 通过名字引用函数参数。
* 引用在 asm 块之外定义的代码标号和变量。（这点很重要，因为局部函数变量必须在 asm 块的外面定义。）
* 使用包含在汇编风格或 C 风格基数表示法中的数字常数。比如，0A26h 和 0xA26 是等价的，且都能使用。
* 在语句中使用 PTR 运算符，比如 inc BYTE PTR\[esi\]。
* 使用 EVEN 和 ALIGN 伪指令。

#### 限制

编写内嵌汇编代码时不允许：

* 使用数据定义伪指令，如 DB（BYTE）和 DW（WORD）。
* 使用汇编运算符（除了 PTR 之外）。
* 使用 STRUCT、RECORD, WIDTH 和 MASK。
* 使用宏伪指令，包括 MACRO、REPT、IRC、IRP 和 ENDM，以及宏运算符（&lt;&gt;、!、&、% 和 .TYPE）。
* 通过名字引用段。（但是，可以用段寄存器名作为操作数。）

#### 寄存器值

不能在一个 asm 块开始的时候对寄存器值进行任何假设。寄存器有可能被 asm 块前面的执行代码修改。Microsoft Visual C++ 的关键字 **fastcall 会使编译器用寄存器来传递参数，为了避免寄存器冲突，不要同时使用** fastcall 和 \_\_asm。

一般情况下，可以在内嵌代码中修改 EAX、EBX、ECX 和 EDX，因为编译器并不期望在语句之间保留这些寄存器值。但是，如果修改的寄存器太多，那么编译器就无法对同一过程中的 C++ 代码进行完全优化，因为优化要用到寄存器。

虽然不能使用 OFFSET 运算符，但是用 LEA 指令也可以得到变量的偏移地址。比如，下面的指令将 buffer 的偏移地址送入 ESI：

```text
lea esi,buffer
```

#### 长度、类型和大小

内嵌汇编代码还可以使用 LENGTH、SIZE 和 TYPE 运算符。LENGTH 运算符返回数组内元素的个数。按照不同的对象，TYPE 运算符返回值如下：

* 对 C 或 C++ 类型以及标量变量，返回其字节数。
* 对结构，返回其字节数。
* 对数组，返回其单个元素的大小。

SIZE 运算符返回 LENGTH\*TYPE 的值。下面的程序片段演示了内嵌汇编程序对各种 C++ 类型的返回值。

Microsoft Visual C++ 内嵌汇编程序不支持 SIZEOF 和 LENGHTOF 运算符。

### 使用 LENGTH、TYPE 和 SIZE 运算符

下面程序包含的内嵌汇编代码使用 LENGTH、TYPE 和 SIZE 运算符对 C++ 变量求值。每个表达式的返回值都在同一行的注释中给出：

```text
struct Package {
  long originZip;        // 4
  long destinationZip;   // 4
  float shippingPrice;   // 4
};

   char myChar;
   bool myBool;
   short myShort;
   int  myInt;
   long myLong;
   float myFloat;
   double myDouble;
   Package myPackage;

   long double myLongDouble;
   long myLongArray[10];

__asm {
   mov  eax,myPackage.destinationZip;

   mov  eax,LENGTH myInt;         // 1
   mov  eax,LENGTH myLongArray;   // 10

   mov  eax,TYPE myChar;          // 1
   mov  eax,TYPE myBool;          // 1
   mov  eax,TYPE myShort;         // 2
   mov  eax,TYPE myInt;           // 4
   mov  eax,TYPE myLong;          // 4
   mov  eax,TYPE myFloat;         // 4
   mov  eax,TYPE myDouble;        // 8
   mov  eax,TYPE myPackage;       // 12
   mov  eax,TYPE myLongDouble;    // 8
   mov  eax,TYPE myLongArray;     // 4

   mov  eax,SIZE myLong;          // 4
   mov  eax,SIZE myPackage;       // 12
   mov  eax,SIZE myLongArray;     // 40
}
```

## 13.5 C语言/C++内嵌汇编代码实例：文件加密

现在查看的简短程序实现如下操作：读取一个文件，对其进行加密，再将其输出到另一个文件。函数 TranslateBuffer 用一个 \_\_asm 块定义语句，在一个字符数组内进行循环，并把每个字符与预定义值进行 XOR 运算。

内嵌语言可以使用函数形参、局部变量和代码标号。由于本例是由 Microsoft Visual [C++](http://c.biancheng.net/cplus/) 编译的 Win32 控制台应用，因此其无符号整数类型为 32 位：

```c
void TranslateBuffer(char * buf,
    unsigned count, unsigned char eChar)
{
    __asm {
        mov esi, buf
        mov ecx, count
        mov al, eChar
    L1:
        xor [esi],al
        inc esi
        loop L1
    }    // asm
}
```

#### C++ 模块

C++ 启动程序从命令行读取输入和输出文件名。在循环内调用 TranslateBuffer 从文件读取数据块，加密，再将转换后的缓冲区写入新文件：

```c
// ENCODE.CPP    复制并加密文件。
#include <iostream>
#include <fstream>
#include "translat.h"

using namespace std;

int main( int argcount, char * args[] )
{ 
    // 从命令行读取输入和输出文件
    if( argcount < 3 ) {
        cout << "Usage: encode infile outfile" << endl;
        return -1;
    }

    const int BUFSIZE = 2000;
    char buffer[BUFSIZE];
    unsigned int count;            // 字符计算

    unsigned char encryptCode;
    cout << "Encryption code [0-255]? ";
    cin >> encryptCode;

    ifstream infile( args[1], ios::binary );
    ofstream outfile( args[2], ios::binary );

    cout << "Reading " << args[1] << " and creating "
        << args[2] << endl;

    while (!infile.eof() )
    {
        infile.read(buffer, BUFSIZE );
        count = infile.gcount();
        TranslateBuffer(buffer, count, encryptCode);
        outfile.write(buffer, count);
    }
    return 0;
}
```

用命令提示符运行该程序，并传递输入和输岀文件名是最容易的。比如，下面的命令行读取 infile.txt，生成 encoded.txt：

```text
encode infile.txt encoded.txt
```

#### 头文件

头文件 translat.h 包含了 TranslateBuffer 的一个函数原型：

```c
void TranslateBuffer(char * buf, unsigned count, unsigned char eChar);
```

#### 过程调用的开销

如果在调试器调试程序时查看 Disassembly 窗口，那么，看到函数调用和返回究竟有多少开销是很有趣的。下面的语句将三个实参送入堆栈，并调用 TranslateBuffer。在 Visual C++ 的 Disassembly 窗口，激活 Show Source Code 和 Show Symbol Names 选项：

```text
; TranslateBuffer(buffer, count, encryptCode)
mov al,byte ptr [encryptCode]
push eax
mov ecx,dword ptr [count]
push ecx
lea edx,[buffer]
push edx
call TranslateBuffer (4159BFh)
add esp, 0Ch
```

下面的代码对 TranslateBuffer 进行反汇编。编译器自动插入了一些语句用于设置 EBP，以及保存标准寄存器集合，集合内的寄存器不论是否真的会被过程修改，总是被保存。

```text
push ebp
mov ebp, esp
sub esp,40h
push ebx
push esi
push edi
;内嵌代码从这里开始。
mov esi,dword ptr [buf]
mov ecx,dword ptr [count]
mov al,byte ptr [eChar]
L1:
    xor byte ptr [esi],al
    inc esi
    loop L1 (41D762h)
;内嵌代码结束。
pop edi
pop esi
pop ebx
mov esp,ebp
pop ebp
ret
```

若关闭了调试器 Disassembly 窗口的 Display Symbol Names 选项，则将参数送入寄存器的三条语句如下：

```text
mov esi,dword ptr [ebp+8]
mov ecx,dword ptr [ebp+0Ch]
mov al,byte ptr [ebp+10h]
```

编译器按要求生成 Debug 目标代码，这是非优化代码，适合于交互式调试。如果选择 Release 目标代码，那么编译器生成的代码就会更加有效（但易读性更差）。

#### 忽略过程调用

本节前面给出的 TranslateBuffer 中有 6 条内嵌指令，其执行总共需要 8 条指令。

如果函数被调用几千次，那么其执行时间就比较可观了。为了消除这种开销，把内嵌代码插入调用 TranslateBuffer 的循环，得到更为有效的程序：

```c
while (!infile.eof() )
  {
    infile.read(buffer, BUFSIZE );
    count = infile.gcount();

    __asm {
       lea esi,buffer
       mov ecx,count
       mov al, encryptCode
    L1:
       xor [esi],al
       inc  esi
       Loop L1
   } // asm

    outfile.write(buffer, count);
  }
```

## 13.6 C语言/C++调用汇编语言函数

为设备驱动器和嵌入式系统编码的程序员常常需要把 C/[C++](http://c.biancheng.net/cplus/) 模块与用[汇编语言](http://c.biancheng.net/asm/)编写的专门代码集成起来。汇编语言特别适合于直接硬件访问、位映射，以及对寄存器和 CPU 状态标识进行底层访问。

整个应用程序都用汇编语言编写是很乏味的，比较有用的方法是，用 C/C++ 编写主程序，而那些不太好用 C 编写的代码则用汇编语言。现在来讨论一下从 32 位 C/ C++ 程序调用汇编程序的一些标准要求。

C/C++ 程序从右到左传递参数，与参数列表的顺序一致。函数返回后，主调程序负责将堆栈恢复到调用前的状态。这可以采用两种方法：一种是将堆栈指针加上一个数值，该值等于参数大小；还有一种是从堆栈中弹出足够多的数。

在汇编源代码中，需要在 .MODEL 伪指令中指定 C 调用规范，并为外部 C/C++ 程序调用的每个过程创建原型。示例如下：

```text
.586
.model flat,C
IndexOf PROTO,
  srchVal:DWORD, arrayPtr:PTR DWORD, count:DWORD
```

#### 函数声明

在 C 程序中，声明外部汇编过程时要使用 extern 限定符。比如，下面的语句声明了 IndexOf：

```text
extern long IndexOf(long n, long array[], unsigned count);
```

如果过程会被 C++ 程序调用，就要添加“C”限定符，以防止 C++ 的名称修饰：

```text
extern "C" long IndexOf(long n, long array[], unsigned count);
```

名称修饰 \(name decoration\) 是一种标准 C++ 编译技术，通过添加字符来修改函数名，添加的字符指明了每个函数参数的确切类型。任何支持函数重载（多个函数有相同的函数名、不同的参数列表）的语言都需要这个技术。

从汇编语言程序员的角度来看，名称修饰存在的问题是：C++ 编译器让链接器去找的是修饰过的名称，而不是生成可执行文件时的原始名称。

#### IndexOf 示例

现在新建一个简单汇编函数，对数组实现线性搜索，找到与样本整数匹配的第一个实例。如果搜索成功，则返回匹配元素的索引位置；否则，返回 -1。该函数将被 C++ 程序调用。在 C++ 中，编写程序段如下：

```c
long IndexOf(long searchVal, long array[], unsigned count)
{
    for(unsigned i = 0; i < count; i++) {
        if(array[i] == searchVal )
            return i;
    }
    return -1;
}
```

参数包括：希望被找到的数值、一个数组指针，以及数组大小。

用汇编语言编写该函数显然是很容易的。编写好的汇编代码放入自己的源代码文件 IndexOf.asm。这个文件将被编译为目标代码文件 IndexOf.obj。使用 Visual Studio 实现主调 C++ 程序与汇编模块的编译和链接。C++ 项目将用 Win32 控制台作为其输出类型，虽然也没有理由不让它成为图形应用程序。

下面为 IndexOf 模块的源代码清单。

```text
;IndexOf 函数    （IndexOf . asm）
.586
.model flat,C
IndexOf PROTO,
    srchVal:DWORD, arrayPtr:PTR DWORD, count:DWORD

.code
;-------------------------------------------
IndexOf PROC USES ecx esi edi,
    srchVal:DWORD, arrayPtr:PTR DWORD, count:DWORD
;
;对 32 位整数数组执行线性搜索，
;寻找指定数值。如果发现匹配数值，
;用EAX返回该数值的索引位置；
;否则，EAX 返回 -1。
;-------------------------------------------
    NOT_FOUND = -1

    mov    eax, srchVal      ;搜索数值
    mov    ecx, count        ;数组大小
    mov    esi, arrayPtr     ;数组指针
    mov    edi, 0            ;索引

L1:cmp [esi+edi*4],eax
    je found
    inc edi
    loop L1

notFound:
    mov eax,NOT_FOUND
    jmp short exit

found:
    mov eax,edi

exit:
    ret
IndexOf ENDP
END
```

首先，注意到用于测试循环的汇编代码 25〜28 行，虽然代码量小，但是高效。对要执行很多次的循环，应试图使其循环体内的指令条数尽可能少：

```text
L1: cmp [esi+edi*4],eax
  je found
  inc edi
  loop L1
```

如果找到匹配项，程序跳转到 34 行，将 EDI 复制到 EAX，该寄存器用于存放函数返回值。在搜索期间，EDI 为当前索引位置。

```text
found:
  mov eax,edi
```

如果没有找到匹配项，则把 -1 赋值给 EAX 并返回：

```text
notFound:
  mov eax,NOT_FOUND
  jmp short exit
```

下面为主调 C++ 程序清单。

```cpp
#include <iostream>
#include <time.h>
#include "indexof.h"
using namespace std;

int main()  {
    // 用伪随机数填充数组
    const unsigned ARRAY_SIZE = 100000;
    const unsigned LOOP_SIZE = 100000;
    char* boolstr[] = {"false","true"};

    long array[ARRAY_SIZE];
    for(unsigned i = 0; i < ARRAY_SIZE; i++)
     array[i] = rand();

    long searchVal;
    time_t startTime, endTime;
    cout << "Enter an integer value to find: ";
    cin >> searchVal;
    cout << "Please wait...\n";

    // 测试汇编函数
    time( &startTime );
    long count = 0;

    for( unsigned n = 0; n < LOOP_SIZE; n++)
         count = IndexOf( searchVal, array, ARRAY_SIZE );

    bool found = count != -1;

    time( &endTime );
    cout << "Elapsed ASM time: " << long(endTime - startTime)
          << " seconds. Found = " << boolstr[found] << endl;

    return 0;
}
```

首先，用伪随机数值对数组进行初始化：

```cpp
long array [ARRAY_SIZE];
for(unsigned i = 0; i < ARRAY_SIZE; i++)
  array[i] = rand();
```

18〜19 行提示用户输入在数组中搜索的数值：

```cpp
cout << "Enter an integer value to find：";
cin >> searchVal;
```

23 行调用 C 链接库的 time 函数（在 time.h 中），把从 1970 年 1 月 1 日起已经过的秒数保存到变量 startTime：

```c
time(&startTime);
```

26 和 27 行按照 LOOP\_SIZE 的值 \(100 000\)，反复执行相同的搜索：

```c
for(unsigned n = 0; n < LOOP_SIZE; n++)
  count = IndexOf(searchVal, array, ARRAY_SIZE);
```

由于数组大小也约为 100 000，因此执行步骤的总数可以多达 100 000 x 100 000，或 100 亿。31〜33 行再次检查时间，并显示循环运行所耗的秒数：

```c
time(&endTime);
cout << "Elapsed ASM time: " << long(endTime - startTime)
   << "seconds. Found = " << boolstr[found] << endl;
```

在高速计算机上测试时，循环执行时间为 6 秒。对 100 亿次迭代而言，这个时间不算多，每秒约有 16.7 亿次迭代。重要的是，需要意识到程序重复过程调用的开销（参数入栈，执行 CALL 和 RET 指令）也是 100 000 次。过程调用导致了相当多的额外处理。

## 13.7 调用C语言/C++函数

可以编写汇编程序来调用 C 和 [C++](http://c.biancheng.net/cplus/) 函数。这样做的理由至少有两个：

* C 和 C++ 有丰富的输入-输出库，因此输入-输出有更大的灵活性。处理浮点数时，这是相当有用的。
* 两种语言都有丰富的数学库。

调用标准 C 库（或 C++ 库）函数时，必须从 C 或 C++ 的 main\(\) 过程启动程序，以便运行库初始化代码。

#### 1\) 函数原型

[汇编语言](http://c.biancheng.net/asm/)代码调用的 C++ 函数，必须用“C”和关键字 extern 定义。其基本语法如下：

```c
extern "C" returnType funcName(paramlist)
{...}
```

示例如下：

```c
extern "C" int askForlnteger()
{
  cout << "Please enter an integer:";
  //...
}
```

与其修改每个函数定义，把多个函数原型放在一个块内显得更容易。然后，还可以省略单个函数实现的 extern 和“C”：

```c
extern "C" {
  int askForlnteger();
  int showInt(int value, unsigned outwidth);
  //etc.
}
```

#### 2\) 汇编语言模块

如果汇编语言模块调用 Irvine32 链接库过程，就要使用如下 .MODEL 伪指令：

```text
.model flat, STDCALL
```

虽然 STDCALL 与 Win32 API 兼容，但是它与 C 程序的调用规范不匹配。因此，在声明由汇编模块调用的外部 C 或 C++ 函数时，必须给 PROTO 伪指令加上 C 限定符：

```text
INCLUDE Irvine32.inc
askForlnteger PROTO C
showInt PROTO C, value:SDWORD, outWidth:DWORD
```

C 限定符是必要的，因为链接器必须把函数名与 C++ 模块输出的参数列表匹配起来。此外，使用了 C 调用规范，汇编器必须生成正确的代码以便在函数调用后清除堆栈。

C++ 程序调用的汇编过程也必须使用 C 限定符，这样汇编器使用的命名规则将能被链接器识别。比如，下面的 SetTextColor 过程有一个双字参数：

```text
SetTextOutColor PROC C,
color:DWORD
...
SetTextOutColor ENDP
```

最后，如果汇编代码调用其他汇编过程，C 调用规范要求在每个过程调用后，把参数从堆栈中移除。

如果汇编代码不调用 Irvine32 过程，就可以在 .MODEL 伪指令中使用 C 调用规范：

```text
;(do not INCLUDE Irvine32.inc)
.586
.model flat,C
```

此时不再需要为 PROTO 和 PROC 伪指令添加 C 限定符：

```text
askForInteger PROTO
showInt PROTO, value:SDWORD, outWidth:DWORD
SetTextOutColor PROC,
  color:DWORD
  ...
SetTextOutColor ENDP
```

#### 3\) 函数返回值

C++ 语言规范没有提及代码实现细节，因此没有规定标准方法让 C 和 C++ 函数返回数值。当编写的汇编代码调用这些语言的函数时，要检查编译器文件以便了解它们的函数是如何返回数值的。

下面列出了一些可能的情况，但并非全部：

* 整数用单个寄存器或寄存器组返回。
* 主调程序可以在堆栈中为函数返回值预留空间。函数在返回前，可以将返回值存入堆栈。
* 函数返回前，浮点数值通常被压入处理器的浮点数堆栈。

下面列出了 Microsoft Visual C++ 函数怎样返回数值：

* bool 和 char 值用 AL 返回。
* short int 值用 AX 返回。
* int 和 long int 值用 EAX 返回。
* 指针用 EAX 返回。
* float、double 和 long double 值分别以 4 字节、8 字节和 10 字节数值压入浮点堆栈。

## 13.8 调用C语言/C++实例：乘法表

现在编写一个简单的应用程序，提示用户输入整数，通过移位的方式将其与 2 的幕 \(2¹〜2ⁿ\) 相乘，并用填充前导空格的形式再次显示每个乘积。输入-输出使用 [C++](http://c.biancheng.net/cplus/)。汇编模块将调用 3 个 C++ 编写的函数。程序将由 C++ 模块启动。

#### [汇编语言](http://c.biancheng.net/asm/)模块

汇编模块包含一个函数 DisplayTable。它调用 C++ 函数 askForInteger 从用户输入一个整数。它还使用循环结构把整数 intVal 重复左移，并调用 showInt 进行显示。

```text
; C++ 调用ASM函数.

INCLUDE Irvine32.inc

;外部C++函数
askForInteger PROTO C
showInt PROTO C, value:SDWORD, outWidth:DWORD

OUT_WIDTH = 8
ENDING_POWER = 10

.data
intVal DWORD ?

.code
;---------------------------------------------
SetTextOutColor PROC C,
    color:DWORD
;
; 设置文本颜色，并清除控制台窗口
; 调用 Irvine32 库函数
;---------------------------------------------
    mov    eax,color
    call    SetTextColor
    call    Clrscr
    ret
SetTextOutColor ENDP

;---------------------------------------------
DisplayTable PROC C
;
; 输入一个整数 n 并显示范围为 n * 2^1 ~ n * 2^10的乘法表
;----------------------------------------------
    INVOKE askForInteger                 ; 调用 C++ 函数
    mov    intVal,eax                    ; 保存整数
    mov    ecx,ENDING_POWER              ; 循环计数器

L1:    push ecx                          ; 保存循环计数器
    shl  intVal,1                        ; 乘以 2
    INVOKE showInt,intVal,OUT_WIDTH
    call    Crlf
    pop    ecx                           ; 恢复循环计数器
    loop    L1

    ret
DisplayTable ENDP

END
```

在 DisplayTable 过程中，必须在调用 showInt 和 newLine 之前将 ECX 入栈，并在调用后将 ECX 出栈，这是因为 Visual C++ 函数不会保存和恢复通用寄存器。函数 askForInteger 用 EAX 寄存器返回结果。

DisplayTable 在调用 C++ 函数时不一定要用 INVOKE。PUSH 和 CALL 指令也能得到同样的结果。对 showInt 的调用如下所示：

```text
push OUT_WIDTH ;最后一个参数首先入栈
push intVal
call showInt       ;调用函数
add esp,8        ;清除堆栈
```

必须遵守 C 语言调用规范，其参数按照逆序入栈，且主调方负责在调用后从堆栈移除实参。

#### C++ 测试程序

下面查看启动程序的 C++ 模块。其入口为 main\(\)，保证执行所需 C++ 语言的初始化代码。它包含了外部汇编过程和三个输岀函数的原型：

```c
// main.cpp

// 演示C++程序和外部汇编模块的函数调用

#include <iostream>
#include <iomanip>
using namespace std;

extern "C" {
    // 外部 ASM 过程:
    void DisplayTable();
    void SetTextOutColor( unsigned color );

    // 局部 C++ 函数:
    int askForInteger();
    void showInt( int value, int width );
}

// 程序入口
int main()
{
    SetTextOutColor( 0x1E );       // 蓝底黄字
    DisplayTable();                // 调用 ASM 过程
    return 0;
}

// 提示用户输入一个整数

int askForInteger()
{
    int n;
    cout << "Enter an integer between 1 and 90,000: ";
    cin >> n;
    return n;
}

// 按特定宽度显示一个有符号整数

void showInt( int value, int width )
{
    cout << setw(width) << value;
}
```

#### 生成项目

将 C++ 和汇编模块添加到 Visual Studio 项目，并在 Project 菜单中选择 Build Solution。

#### 程序输出

当用户输入为 90 000 时，乘法表程序产生的输出如下：

![img](http://c.biancheng.net/uploads/allimg/190530/4-1Z530140Sb61.gif)

#### Visual Studio 项目属性

如果使用 Visual Studio 生成集成了 C++ 和汇编代码的程序，并且调用 Irvine32 链接库，就需要修改某些项目设置。以 Multiplication\_Table 程序为例。

在 Project 菜单中选择 Properties，在窗口左边的 Configuration Properties 条目下，选择 Linker。在右边面板的 Additional Library Directories 条目中输入 c:\Irvine。

示例如下图所示。点击OK关闭 Project Property Pages 窗口。现在 Visual Studio 就可以找到 Irvine32 链接库了。

![&#x6307;&#x5B9A;Lrvine.lib&#x7684;&#x4F4D;&#x7F6E;](http://c.biancheng.net/uploads/allimg/190530/4-1Z530140913455.gif)

## 13.9 调用C语言/C++库函数

C 语言有标准函数集合，被称为标准 C 库 \(Standard C Library\)。同样的函数还可以用于 [C++](http://c.biancheng.net/cplus/) 程序，因此，也可用于与 C 和 C++ 程序连接的汇编模块。

汇编模块调用 C 函数时，就必须包含函数的原型。一般通过访问 C++ 编译器的帮助系统可以找到 C 函数原型。程序调用 C 函数时，需要先将 C 函数原型转换为[汇编语言](http://c.biancheng.net/asm/)原型。

#### printf 函数

下面是 printf 函数的 C/C++ 语言原型，第一个参数为字符指针，其后跟了一组可变数量的参数：

```c
int printf(
  const char * format [, argument]...
);
```

C/C++ 编译器的帮助库可以查阅到 printf 函数文档。汇编语言中与之等效的函数原型将 char\* 改为 PTR BYTE，将可变长度参数列表的类型改为 VARARG：

```text
printf PROTO C, pString:PTR BYTE, args:VARARG
```

另一个有用的函数是 scanf，用于从标准输入（键盘）接收字符、数字和字符串，并将输入数值分配给变量：

```text
scanf PROTO C, format:PTR BYTE, args:VARARG
```

### 用 printf 函数显示格式化实数

编写汇编函数格式化并显示浮点数不是一件容易的事。与其由程序员自行编码，还不如利用 C 库的 printf 函数。需要创建 C 或 C++ 的启动模块，并将其与汇编代码链接。

下面给出了用 Visual C++.NET 创建这种程序的过程：

1\) 用 Visual C++ 创建一个 Win32 控制台程序。创建文件 main.cpp，并插入函数 main，该函数调用了 asmMain：

```c
extern "C" void asmMain();
int main()
{
  asmMain();
  return 0;
}
```

2\) 在 main.cpp 所在的文件夹中，创建一个汇编模块 asmMain.asm。该模块包含过程 asmMain，并声明使用 C 调用规范：

```text
; asmMain.asm
.386
.model flat,stdcall
.stack 2000
.code
asmMain PROC C
  ret
asmMain ENDP
END
```

3\) 汇编 asmMain.asm（但不进行链接），生成 asmMain.obj。

4\) 将 asmMain.obj 添加到 C++ 项目。

5\) 构建并运行项目。如果修改了 asmMain.asm，则在运行前，需要再一次汇编和构建项目。

一旦程序正确建立，就可以向 asmMain.asm 添加代码来调用 C/C++ 函数。

显示双精度数值 下面是 asmMain 中的汇编代码，它通过调用 printf 输岀了一个类型为 REAL8 的数值：

```text
.data
double1 REAL8 1234567.890123
formatStr BYTE "%.3f", 0dh, 0ah, 0
.code
INVOKE printf, ADDR formatStr, double1
```

相应的输出如下：

```text
1234567.890
```

这里，传递给 printf 的格式化字符串与 C++ 中的略有不同：不是插入转义字符，如 \n，而是必须插入 ASCII 字符（0dh, 0ah）。

传递给 printf 的浮点参数应声明为 REAL8 类型。不过传递的数值也可能是 REAL4 类型，这需要相当的编程技巧。若想了解 C++ 编译器是如何工作的，可以声明一个 float 类型的变量，并将其传递给 printf。编译程序，并用调试器跟踪该程序的反汇编代码。

#### 多参数

printf 函数接收可变数量的参数，因此很容易在一次函数调用中对两个数进行格式化并显示它们：

```text
TAB = 9
.data
formatTwo BYTE "%.2f",TAB,"%.3f",0dh,0ah,0
val1 REAL8 456.789
val2 REAL8 864.231
.code
INVOKE printf, ADDR formatTwo, val1, val2
```

相应的输岀如下：

```text
456.79  864.231
```

### 用 scanf 函数输入实数

调用 scanf 可以从用户输入浮点数。SmallWin.inc（包括在 Irvine32.inc 内）定义的函数原型如下所示：

```text
scanf PROTO C,
  format:PTR BYTE, args:VARARG
```

传递给它的参数包括：格式化字符串的偏移量，一个或多个 REAL4、REAL8 类型变量的偏移量（这些变量存放了用户输入的数值）。调用示例如下：

```text
.data
strSingle BYTE "%f", 0
strDouble BYTE "%lf",0
single1 REAL4 ?
double1 REAL8 ?
.code
INVOKE scanf, ADDR strSingle, ADDR single1
INVOKE scanf, ADDR strDouble, ADDR double1
```

必须从 C 或 C++ 启动程序中调用汇编语言代码。

## 13.10 C/C++调用汇编语言实例：目录表程序

现在编写一个简短的程序，清除屏幕，显示当前磁盘目录，并请求用户输入文件名。程序员可能希望扩展该程序，以打开并显示被选中文件。

#### [C++](http://c.biancheng.net/cplus/) 根模块

C++ 模块只有一个对 asm\_main 的调用，因此可以将其称为根模块 \(stub module\)：

```cpp
// main.cpp
//根模块：启动汇编程序
extern "C" void asm_main() ; // asm 启动过程
void main()
{
  asm_main();
}
```

#### ASM 模块

[汇编语言](http://c.biancheng.net/asm/)模块包括了函数原型、若干字符串和一个 fileName 变量。模块两次调用 system 函数，向其传递“cls”和“dir”命令。然后调用 printf，显示请求文件名的提示行，再调用 scanf，使用户输入文件名。

程序不调用 Irvine32 库中的任何函数，因此可以将 .MODEL 伪指令设置为 C 语言规范：

```text
; 从 C++ 启动的 ASM 程序 (asmMain.asm)

.586
.MODEL flat,C

; 标准 C 库函数
system PROTO, pCommand:PTR BYTE
printf PROTO, pString:PTR BYTE, args:VARARG
scanf  PROTO, pFormat:PTR BYTE,pBuffer:PTR BYTE, args:VARARG
fopen  PROTO, mode:PTR BYTE, filename:PTR BYTE
fclose PROTO, pFile:DWORD

BUFFER_SIZE = 5000
.data
str1 BYTE "cls",0
str2 BYTE "dir/w",0
str3 BYTE "Enter the name of a file: ",0
str4 BYTE "%s",0
str5 BYTE "cannot open file",0dh,0ah,0
str6 BYTE "The file has been opened and closed",0dh,0ah,0
modeStr BYTE "r",0

fileName BYTE 60 DUP(0)
pBuf  DWORD ?
pFile DWORD ?

.code
asm_main PROC

    ; 清除屏幕，显示磁盘目录
    INVOKE system,ADDR str1
    INVOKE system,ADDR str2

    ; 清除文件名
    INVOKE printf,ADDR str3
    INVOKE scanf, ADDR str4, ADDR fileName

    ; 尝试打开文件
    INVOKE fopen, ADDR fileName, ADDR modeStr
    mov pFile,eax

    .IF eax == 0                ; 不能打开文件
      INVOKE printf,ADDR str5
      jmp quit
    .ELSE
      INVOKE printf,ADDR str6
    .ENDIF

    ; 关闭文件
    INVOKE fclose, pFile

quit:
    ret                         ; 返回 C++ 主程序
asm_main ENDP

END
```

函数 scanf 需要两个参数：第一个是格式化字符串（“%s”）的指针，第二个是输入字符串变量（fileName）的指针。因为互联网上有丰富的文档，因此这里不再浪费时间来解释标准 C 函数。

