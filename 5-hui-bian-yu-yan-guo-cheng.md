---
description: >-
  本章介绍了汇编语言中的过程，也称为子程序或函数。任何具有一定规模的程序都需要被划分为几个部分，其中某些部分要被使用多次。通过本章的学习大家会发现寄存器可以传递参数，也将了解为了追踪过程的调用位置，CPU
  使用的运行时堆栈。最后，本章会介绍本教程提供的两个代码库，分别称为 Irvine 32 和 Irvine 64，其中包含了有用的工具来简化输入输出。
---

# 5 汇编语言过程

## 5.1 汇编语言堆栈简介

如下图所示，如果把 10 个盘子垒起来，其结果就称为堆栈。虽然有可能从这个堆栈的中间移出一个盘子，但是，更普遍的是从顶端移除。新的盘子可以叠加到堆栈顶部，但不能加在底部或中部。

![&#x76D8;&#x5B50;&#x6784;&#x6210;&#x7684;&#x5806;&#x6808;](http://c.biancheng.net/uploads/allimg/190505/4-1Z50514301TQ.gif)

堆栈[数据结构](http://c.biancheng.net/data_structure/)（stack data structure）的原理与盘子堆栈相同：新值添加到栈顶，删除值也在栈顶移除。通常，对各种编程应用来说，堆栈都是有用的结构，并且它们也容易用面向对象的编程方法来实现。

如果大家已经学习过使用数据结构的编程课程，那么就应该已经用过堆栈抽象数据类型（stack abstract data type）。

堆栈也被称为 LIFO 结构（后进先出，Last-In First-Out），,其原因是，最后进入堆栈的值也是第一个出堆栈的值。

## 5.2 汇编语言运行时堆栈（内存数组）

运行时堆栈是内存数组，CPU 用 ESP（扩展堆栈指针，extended stack pointer）寄存器对其进行直接管理，该寄存器被称为堆栈指针寄存器（stack pointer register）。

32位模式下，ESP 寄存器存放的是堆栈中某个位置的 32 位偏移量。ESP 基本上不会直接被程序员控制，反之，它是用 CALL、RET、PUSH 和 POP 等指令间接进行修改。

ESP 总是指向添加，或压入（pushed）到栈顶的最后一个数值。为了便于说明，假设现有一个堆栈，内含一个数值。如下图所示，ESP 的内容是十六进制数 0000 1000，即刚压入堆栈数值（0000 0006）的偏移量。在图中，当堆栈指针数值减少时，栈顶也随之下移。

![&#x5305;&#x542B;&#x4E00;&#x4E2A;&#x503C;&#x5F97;&#x5806;&#x6808;](http://c.biancheng.net/uploads/allimg/190505/4-1Z505154RRX.gif)

上图中，每个堆栈位置都是32位长，这 是32位模式下运行程序的情形。

运行时堆栈工作于系统层，处理子程序调用。堆栈 ADT 是编程结构，通常用高级编程语言编写，如 [C++](http://c.biancheng.net/cplus/) 或 [Java](http://c.biancheng.net/java/)。它用于实现基于后进先出操作的算法。

### 入栈操作

32 位入栈操作把栈顶指针减 4，再将数值复制到栈顶指针指向的堆栈位置。下图展示了把 0000 00A5 压入堆栈的结果，堆栈中已经有一个数值（0000 0006）。注意，ESP 寄存器总是指向最后压入堆栈的数据项。

![&#x5C06;&#x6574;&#x6570;&#x538B;&#x5165;&#x5806;&#x6808;](http://c.biancheng.net/uploads/allimg/190505/4-1Z505154Zb64.gif)

上图中显示的堆栈顺序与之前示例给出的盘堆栈顺序相反，这是因为运行时堆栈在内存中是向下生长的，即从高地址向低地址扩展。入栈之前， ESP=0000 1000h；入栈之后，ESP=0000 0FFCh。下图显示了同一个堆栈总共压入 4 个整数之后的情况。

![&#x538B;&#x5165;&#x6570;&#x503C;00000001&#x548C;00000002&#x7684;&#x5806;&#x6808;](http://c.biancheng.net/uploads/allimg/190505/4-1Z505154942144.gif)

### 出栈操作

出栈操作从堆栈删除数据。数值弹岀堆栈后，栈顶指针增加（按堆栈元素大小），指向堆栈中下一个最高位置。下图展示了数值 0000 0002 弹出前后的堆栈情况。

![&#x4ECE;&#x8FD0;&#x884C;&#x65F6;&#x5806;&#x6808;&#x5F39;&#x51FA;&#x4E00;&#x4E2A;&#x6570;&#x503C;](http://c.biancheng.net/uploads/allimg/190505/4-1Z50515503c32.gif)

ESP 之下的堆栈域在逻辑上是空白的，当前程序下一次执行任何数值入栈操作指令都可以覆盖这个区域。

### 堆栈应用

运行时堆栈在程序中有一些重要用途：

* 当寄存器用于多个目的时，堆栈可以作为寄存器的一个方便的临时保存区。在寄存器被修改后，还可以恢复其初始值。
* 执行 CALL 指令时，CPU 在堆栈中保存当前过程的返回地址。
* 调用过程时，输入数值也被称为参数，通过将其压入堆栈实现参数传递。
* 堆栈也为过程局部变量提供了临时存储区域。

## 5.3 PUSH和POP指令（压栈和出栈）

汇编里把一段内存空间定义为一个栈，栈总是先进后出，栈的最大空间为 64K。由于 "栈" 是由高到低使用的，所以新压入的数据的位置更低，ESP 中的指针将一直指向这个新位置，所以 ESP 中的地址数据是动态的。

### PUSH 指令

PUSH 指令首先减少 ESP 的值，再将源操作数复制到堆栈。操作数是 16 位的，则 ESP 减 2，操作数是 32 位的，则 ESP 减 4。PUSH 指令有 3 种格式：

```text
PUSH reg/mem16
PUSH reg/mem32
PUSH inm32
```

### POP指令

POP 指令首先把 ESP 指向的堆栈元素内容复制到一个 16 位或 32 位目的操作数中，再增加 ESP 的值。如果操作数是 16 位的，ESP 加 2，如果操作数是 32 位的，ESP 加 4：

```text
POP reg/mem16
POP reg/mem32
```

### PUSHFD 和 POPFD 指令

PUSHFD 指令把 32 位 EFLAGS 寄存器内容压入堆栈，而 POPFD 指令则把栈顶单元内容弹出到 EFLAGS 寄存器：

```text
pushfd
popfd
```

不能用 MOV 指令把标识寄存器内容复制给一个变量，因此，PUSHFD 可能就是保存标志位的最佳途径。有些时候保存标志寄存器的副本是非常有用的，这样之后就可以恢复标志寄存器原来的值。通常会用 PUSHFD 和 POPFD 封闭一段代码：

```text
pushfd      ;保存标志寄存器
;
;任意语句序列
;
popfd      ;恢复标志寄存器
```

当用这种方式使用入栈和出栈指令时，必须确保程序的执行路径不会跳过 POPFD 指令。当程序随着时间不断修改时，很难记住所有入栈和出栈指令的位置。因此，精确的文档就显得至关重要！

一种不容易出错的保存和恢复标识寄存器的方法是：将它们压入堆栈后，立即弹出给一个变量：

```text
.data
saveFlags DWORD ?
.code
pushfd                    ;标识寄存器内容入栈
pop saveFLags             ;复制给一个变量
```

下述语句从同一个变量中恢复标识寄存器内容：

```text
push saveFlags            ;被保存的标识入栈
popfd                     ;复制给标识寄存器
```

### PUSHAD，PUSHA，POPAD 和 POPA

PUSHAD 指令按照 EAX、ECX、EDX、EBX、ESP（执行 PUSHAD 之前的值）、EBP、ESI 和 EDI 的顺序，将所有 32 位通用寄存器压入堆栈。

POPAD 指令按照相反顺序将同样的寄存器弹出堆栈。与之相似，PUSHA 指令按序（AX、CX、DX、BX、SP、BP、SI 和 DI）将 16 位通用寄存器压入堆栈。

POPA 指令按照相反顺序将同样的寄存器弹出堆栈。在 16 位模式下，只能使用 PUSHA 和 POPA 指令。

如果编写的过程会修改 32 位寄存器的值，则在过程开始时使用 PUSHAD 指令，在结束时使用 POPAD 指令，以此保存和恢复寄存器的内容。示例如下列代码段所示：

```text
MySub PROC
    pushad                 ;保存通用寄存器的内容
    .
    .
    mov eax,...
    mov edx,...
    mov ecx,...
    .
    .
    popad                   ;恢复通用寄存器的内容
    ret
MySub ENDP
```

必须要指岀，上述示例有一个重要的例外：过程用一个或多个寄存器来返回结果时，不应使用 PUSHA 和 PUSHAD。假设下述 ReadValue 过程用 EAX 返回一个整数；调用 POPAD 将会覆盖 EAX 中的返回值：

```text
ReadValue PROC
    pushad                    ;保存通用寄存器的内容
    .
    .
    mov eax rreturn_value
    .
    .
    popad                    ;覆盖 EAX !
    ret
ReadValue ENDP
```

### 示例：字符串反转

现在查看名为 RevStr 的程序：在一个字符串上循环，将每个字符压入堆栈，再把这些字符从堆栈中弹出（相反顺序），并保存回同一个字符串变量。由于堆栈是 LIFO（后进先出）结构，字符串中的字母顺序就发生了翻转：

```text
;字符串翻转（Revstr.asm）
.386
.model flat,stdcall
.stack 4096
ExitProcess PROTO,dwExitCode:DWORD

.data
aName BYTE "Abraham Lincoln",0
nameSize = ($-aName)-1

.code
main PROC
;将名字压入堆栈
    mov ecx,nameSize
    mov esi,0

L1:    movzx eax,aName[esi]        ;获取字符
    push eax                       ;压入堆栈
    inc esi
    loop L1
;将名字逆序弹出堆栈
;并存入aName数组
    mov ecx,nameSize
    mov esi,0

L2:    pop eax                        ;获取字符
    mov aName[esi],al                 ;存入字符串
    inc esi
    loop L2

    INVOKE ExitProcess,0
main ENDP
END main
```

## 5.4 PROC和ENDP伪指令：定义一个过程

如果大家已经学过了高级编程语言，那么就会知道将程序分割为子过程（subroutine）是多么有用。一个复杂的问题常常要分解为相互独立的任务，这样才易于被理解、实现以及有效地测试。

在[汇编语言](http://c.biancheng.net/asm/)中，通常用术语过程（procedure）来指代子程序。在其他语言中，子程序也被称为方法或函数。

就面向对象编程而言，单个类中的函数或方法大致相当于封装在一个汇编语言模块中的过程和数据集合。汇编语言出现的时间远早于面向对象编程，因此它不具备面向对象编程中的形式化结构。汇编程序员必须在程序中实现自己的形式化结构。

### 定义过程

过程可以非正式地定义为：由返回语句结束的已命名的语句块。过程用 PROC 和 ENDP 伪指令来定义，并且必须为其分配一个名字（有效标识符）。到目前为止，所有编写的程序都包含了一个名为 main 的过程，例如：

```text
main PROC
.
.
main ENDP
```

当在程序启动过程之外创建一个过程时，就用 RET 指令来结束它。RET 强制 CPU 返回到该过程被调用的位置：

```text
sample PROC
  .
  .
  ret
sample ENDP
```

### 过程中的标号

默认情况下，标号只在其被定义的过程中可见。这个规则常常影响到跳转和循环指令。在下面的例子中，名为 Destination 的标号必须与 JMP 指令位于同一个过程中：

```text
jmp Destination
```

解决这个限制的方法是定义全局标号，即在名字后面加双冒号 \(::\)。

```text
Destination::
```

就程序设计而言，跳转或循环到当前过程之外不是个好主意。过程用自动方式返回并调整运行时堆栈。如果直接跳出一个过程，则运行时堆栈很容易被损坏。

### 示例：三个整数求和

现在创建一个名为 SumOf 的过程计算三个 32 位整数之和。假设在过程调用之前，整数已经分配给 EAX、EBX 和 ECX。过程用 EAX 返回和数：

```text
SumOf PROC
    add eax,ebx
    add eax,ecx
    ret
SumOf ENDP
```

### 过程说明

要培养的一个好习惯是为程序添加清晰可读的说明。下面是对放在每个过程开头的信息的一些建议：

* 对过程实现的所有任务的描述。
* 输入参数及其用法的列表，并将其命名为 Receives \( 接收 \)。如果输入参数对其数值有特殊要求，也要在这里列岀来。
* 对过程返回的所有数值的描述，并将其命名为 Returns \( 返回 \)。
* 所有特殊要求的列表，这些要求被称为先决条件 \(preconditions\)，必须在过程被调用之前满足。列表命名为 Requires。例如，对一个画图形线条的过程来说，一个有用的先决条件是该视频显示适配器必须已经处于图形模式。

上述选择的描述性标号，如 ReceivesReturns 和 Requires，不是绝对的；其他有用的名字也常常被使用。

有了这些思想，现在对 SumOf 过程添加合适的说明：

```text
;-------------------------------------------------------
; sumof
; 计算 3 个 32 位整数之和并返回和数。
; 接收：EAX、EBX和ECX为3个整数，可能是有符号数，也可能是无符号数。
; 返回：EAX=和数
;------------------------------------------------------
SumOf PROC
    add eax,ebx
    add eax,ecx
    ret
SumOf ENDP
```

用高级语言，如 C 和 [C++](http://c.biancheng.net/cplus/)，编写的函数，通常用 AL 返回 8 位的值，用 AX 返回 16 位的值，用 EAX 返回 32 位的值。

## 5.5 CALL和RET指令：调用一个过程

CALL 指令调用一个过程，指挥处理器从新的内存地址开始执行。过程使用 RET（从过程返回）指令将处理器转回到该过程被调用的程序点上。

从物理上来说，CALL 指令将其返回地址压入堆栈，再把被调用过程的地址复制到指令指针寄存器。当过程准备返回时，它的 RET 指令从堆栈把返回地址弹回到指令指针寄存器。32 位模式下，CPU 执行的指令由 EIP（指令指针寄存器）在内存中指岀。16 位模式下，由 IP 指出指令。

### 调用和返回示例

假设在 main 过程中，CALL 指令位于偏移量为 0000 0020 处。通常，这条指令需要 5 个字节的机器码，因此，下一条语句（本例中为一条 MOV 指令）就位于偏移量为 0000 0025 处：

```text
   main PROC
00000020 call MySub
00000025 mov eax,ebx
```

然后，假设 MySub 过程中第一条可执行指令位于偏移量 0000 0040 处：

```text
  MySub PROC
00000040 mov eaxz edx
   .
   .
   ret
  MySub ENDP
```

当 CALL 指令执行时如下图所示，调用之后的地址（0000 0025）被压入堆栈，MySub 的地址加载到 EIP。

![&#x6267;&#x884C;&#x4E00;&#x5929;CALL&#x6307;&#x4EE4;](http://c.biancheng.net/uploads/allimg/190505/4-1Z5051K14A60.gif)

执行 MySub 中的全部指令直到 RET 指令。当执行 RET 指令时，ESP 指向的堆栈数值被弹岀到 EIP（如下图所示，步骤 1）。在步骤 2 中，ESP 的数值增加，从而指向堆栈中的前一个值（步骤 2）。

![&#x6267;&#x884C;RET&#x6307;&#x4EE4;](http://c.biancheng.net/uploads/allimg/190505/4-1Z5051K214956.gif)

## 5.6 过程调用嵌套

被调用过程在返回之前又调用了另一个过程时，就发生了过程调用嵌套。假设 main 调用了过程 Sub1。当 Sub1 执行时，它调用了过程 Sub2。当 Sub2 执行时，它调用了过程 Sub3。步骤如下图所示。

![&#x8FC7;&#x7A0B;&#x8C03;&#x7528;&#x5D4C;&#x5957;](http://c.biancheng.net/uploads/allimg/190506/4-1Z506100T5F3.gif)

当执行 Sub3 末尾的 RET 指令时，将 stack\[ESP\]（堆栈段首地址 +ESP 给岀的偏移量）中的数值弹出到指令指针寄存器中，这使得执行转回到调用 Sub3 后面的指令。下图显示的是执行从 Sub3 返回操作之前的堆栈：

![img](http://c.biancheng.net/uploads/allimg/190506/4-1Z506100913457.gif)

返回之后，ESP 指向栈顶下一个元素。当 Sub2 末尾的 RET 指令将要执行时，堆栈如下所示：

![img](http://c.biancheng.net/uploads/allimg/190506/4-1Z50610093U01.gif)

最后，执行 Sub1 的返回，stack\[ESP\] 的内容弹出到指令指针寄存器，继续在 main 中执行：

![img](http://c.biancheng.net/uploads/allimg/190506/4-1Z506100952C3.gif)

显然，堆栈证明了它很适合于保存信息，包括过程调用嵌套。一般说来，堆栈结构用于程序需要按照特定顺序返回的情况。

### 向过程传递寄存器参数

如果编写的过程要执行一些标准操作，如整数数组求和，那么，在过程中包含对特定变量名的引用就不是一个好主意。如果这样做了，该过程就只能作用于一个数组。更好的方法是向过程传递数组的偏移量以及指定数组元素个数的整数。这些内容被称为参数（或输入参数）。在[汇编语言](http://c.biancheng.net/asm/)中，经常用通用寄存器来传递参数。

在《[PROC和ENDP伪指令](http://c.biancheng.net/view/3536.html)》一节中创建了一个简单的过程 SumOf，计算 EAX、EBX 和 ECX 中的整数之和。在 main 调用 SumOf 之前，将数值分配给 EAX、EBX 和 ECX：

```text
.data
theSum DWORD ?
.code
main PROC
mov eax, 10000h          ;参数
mov ebx, 20000h          ;参数
mov ecx, 30000h          ;参数
call Sumof               ;EAX=(EAX+EEX+ECX)
mov theSum,eax           ;保存和数
```

在 CALL 语句之后，选择了将 EAX 中的和数复制给一个变量。

## 5.7 示例：整数数组求和

程序员在 [C++](http://c.biancheng.net/cplus/) 或 [Java](http://c.biancheng.net/java/) 中编写过的非常常见的循环类型是计算整数数组之和。这在[汇编语言](http://c.biancheng.net/asm/)中很容易实现，它可以被编码为按照尽可能快的方式来运行。比如，在循环内可以使用寄存器而非变量。

现在创建一个过程 ArraySum，从一个调用程序接收两个参数：一个指向 32 位整数数组的指针，以及一个数组元素个数的计数器。该过程计算和数，并用 EAX 返回数组之和：

```text
;------------------------------------
;ArraySum
;计算32位整数数组元素之和
;接收：ESI = 数组偏移量
;      ECX = 数组元素的个数
;返回：EAX = 数组元素之和
;-------------------------------------
ArraySum PROC
    push esi                ;保存ESI和ECX
    push ecx
    mov eax,0               ;设置和数为0

L1：    add eax,[esi]       ;将每个整数与和数相加
    add esi,TYPE DWORD      ;指向下一个整数

    loop L1                 ;按照数组大小重复

    pop ecx                 ;恢复ECX和ESI
    pop esi               
    ret                     ;和数在EAX中
ArraySum ENDP
```

这个过程没有特别指定数组名称和大小，它可以用于任何需要计算32位整数数组之和的程序。只要有可能，编程者也应该编写具有灵活性和适应性的程序。

### 测试 ArraySum 过程

下面的程序通过传递一个 32 位整数数组的偏移量和长度来测试 ArraySum 过程。调用 ArraySum 之后，程序将过程的返回值保存在变量 theSum 中。

```text
;测试ArraySum过程
.386
.model flat,stdcall
.stack 4096
ExitProcess PROTO,dwExitCode:DWORD

.data
array DWORD 10000h,20000h,30000h,40000h,50000h
theSum DWORD ?

.code
main PROC
    mov esi,OFFSET array          ;ESI指向数组
    mov ecx,LENGTHOF array        ;ECX = 数组计算器
    call ArraySum                 ;计算和数
    mov theSum,eax                ;用EAX返回和数

    INVOKE ExitProcess,0
main ENDP
;------------------------------------
;ArraySum
;计算32位整数数组元素之和
;接收：ESI = 数组偏移量
;       ECX = 数组元素的个数
;返回：EAX = 数组元素之和
;-------------------------------------
ArraySum PROC
    push esi                 ;保存ESI和ECX
    push ecx
    mov eax,0                ;设置和数为0

L1：    add eax,[esi]        ;将每个整数与和数相加
    add esi,TYPE DWORD      ;指向下一个整数

    loop L1                 ;按照数组大小重复

    pop ecx                 ;恢复ECX和ESI
    pop esi               
    ret                     ;和数在EAX中
ArraySum ENDP
END  main
```

## 5.8 USES运算符：保存和恢复寄存器

在上一节《[整数数组求和](http://c.biancheng.net/view/3539.html)》的 ArraySum 示例中，ECX 和 ESI 在过程开始时被压入堆栈，在过程结束时被弹出堆栈。这是大多数过程修改寄存器的典型操作。总是保存和恢复被过程修改的寄存器，将使得调用程序确保自己的寄存器值不会被覆盖。但是对用于返回数值的寄存器应该例外，通常是指 EAX，不要将它们压入和弹出堆栈。

### USES 运算符

USES 运算符与 PROC 伪指令一起使用，让程序员列出在该过程中修改的所有寄存器名。USES 告诉汇编器做两件事情：第一，在过程开始时生成 PUSH 指令，将寄存器保存到堆栈；第二，在过程结束时生成 POP 指令，从堆栈恢复寄存器的值。

USES 运算符紧跟在 PROC 之后，其后是位于同一行上的寄存器列表，表项之间用空格符或制表符（不是逗号）分隔。

在 ArraySum 过程使用 PUSH 和 POP 指令来保存和恢复 ESI 和 ECX。 USES 运算符能够更加容易地实现同样的功能：

```text
ArraySum PROC USES esi ecx
  mov eax, 0                   ;置和数为0
L1:
  add eax,[esi]                ;将每个整数与和数相加
  add esi, TYPE DWORD          ;指向下个整数
  loop L1                      ;按照数组大小重复
  ret                          ;和数在 EAX 中
ArraySum ENDP
```

汇编器生成的相应代码展示了使用 USES 的效果：

```text
ArraySum PROC
  push esi
  push ecx
  mov eax, 0                      ;置和数为0
L1:
  add eax, [esi]                  ;将每个整数与和数相加
  add esi, TYPE DWORD             ;指向下一个整数
  loop L1                         ;按照数组大小重复
  pop ecx
  pop esi
  ret
ArraySum ENDP
```

> 调试提示：使用 Microsoft Visual Studio 调试器可以查看由 MASM 高级运算符和伪指令生成的隐藏机器指令。在调试窗口中右键点击，选择 Go To Disassembly。该窗口显示程序源代码，以及由汇编器生成的隐藏机器指令。

当过程利用寄存器（通常用 EAX）返回数值时，保存使用寄存器的惯例就岀现了一个重要的例外。在这种情况下，返回寄存器不能被压入和弹出堆栈。例如下述 SumOf 过程把 EAX 压入、弹出堆栈，就会丢失过程的返回值：

```text
SumOf PROC                             ;三个整数之和
  push eax                             ;保存EAX
  add eax, ebx
  add eax, ecx                         ;计算EAX、EBX和ECX之和
  pop eax                              ;和数丢失！
  ret
SumOf ENDP
```

## 5.9 链接库简介

如果编程者花时间的话，就可以用[汇编语言](http://c.biancheng.net/asm/)编写岀详细的输入输岀代码。就好比自己从头开始搭建汽车，然后可以驾车出行一样。这个工作很有趣但也很耗时。接下来我们来了解一下什么是链接库。

### 背景知识

链接库是一种文件，包含了已经汇编为机器代码的过程（子程序）。链接库开始时是一个或多个源文件，这些文件再被汇编为目标文件。目标文件插入到一个特殊格式文件，该文件由链接器工具识别。

假设一个程序调用过程 WriteString 在控制台窗口显示一个字符串。该程序源代码必须包含 PROTO 伪指令来标识 WriteString 过程：

```text
WriteString proto
```

之后，CALL 指令执行 WriteString：

```text
call WriteString
```

当程序进行汇编时，汇编器将不指定 CALL 指令的目标地址，它知道这个地址将由链接器指定。链接器在链接库中寻找 WriteString，并把库中适当的机器指令复制到程序的可执行文件中。同时，它把 WriteString 的地址插入到 CALL 指令。

如果被调用过程不在链接库中，链接器就发出错误信息，且不会生成可执行文件。

### 链接命令选项

链接器工具把一个程序的目标文件与一个或多个目标文件以及链接库组合在一起。比如，下述命令就将 hello.obj 与 irvine32.lib 和 kernel32.lib 库链接起来：

```text
link hello.obj irvine32.lib kernel32.lib
```

### 32位程序链接

kernel32.lib 文件是 Microsoft Windows 平台软件开发工具（Software Development Kit）的一部分，它包含了 kernel32.dll 文件中系统函数的链接信息。kernel32.dll 文件是 MS-Windows 的一个基本组成部分，被称为动态链接库（dynamic link library）。它含有的可执行函数实现基于字符的输入输出。

下图展示了为什么 kernel32.lib 是通向 kernel32.dll 的桥梁。

![32&#x4F4D;&#x7A0B;&#x5E8F;&#x94FE;&#x63A5;](http://c.biancheng.net/uploads/allimg/190506/4-1Z50613391L52.gif)

## 5.10 Irvine32链接库

[汇编语言](http://c.biancheng.net/asm/)编程没有 Microsoft 认可的标准库。在 20 世纪 80 年代早期，程序员第一次开始为 x86 处理器编写汇编语言时，MS-DOS 是常用的操作系统。这些 16 位程序可以调用 MS-DOS 函数（即 INT 21h 服务）来实现简单的输入输出。

即使是在那个时代，如果想在控制台上显示一个整数，也需要编写一个相当复杂的程序，将整数的内部二进制表示转换为可以在屏幕上显示的 ASCII 字符序列。这个过程被称为 WriteInt，下面是其抽象为伪代码的逻辑：

初始化：

```text
let n equal the binary value
let buffer be an array of char[size]
```

算法：

```text
i = size -1                      ;缓冲区最后一个位置
repeat
  r = n mod 10                   ;余数
  n = n / 10                     ;整数除法
  digit = r OR 30h               ;将工转换为 ASCII 数字
  bufferf[i--] = digit           ;保存到缓冲区
until n = 0
if n is negative
  buffer[i] = "-"                ;插入负号
while i > 0
  print buffer[i]
  i++
```

注意，数字是按照逆序生成，插入缓冲区，从后往前移动。然后，数字按照正序写到控制台。虽然这段代码简单到足以用 C/[C++](http://c.biancheng.net/cplus/) 实现，但是如果是在汇编语言中，它还需要一些高级技巧。

专业程序员通常更愿意自己建立库，这是一种很好的学习经验。在 Windows 的 32 位模式下，输入输出库必须能直接调用操作系统的内容。这个学习曲线相当陡峭，对编程初学者提出了一些挑战。因此，Irvine32 链接库被设计成给初学者提供简单的输入输岀接口。

随着学习的推进，我们将能获得自己创建库的知识和技术。只要成为库的创建者，就能自由地修改和重用库。

下表列出了 Irvine32 链接库的全部过程。

| 过程 | 说明 |
| :--- | :--- |
| CloseFile | 关闭之前已经打开的磁盘文件 |
| Clrscr | 清除控制台窗口，并将光标置于左上角 |
| CreateOutputFile | 为输出模式下的写操作创建一个新的磁盘文件 |
| Crlf | 在控制台窗口中写一个行结束的序列 |
| Delay | 程序执行暂停指定的 n 毫秒 |
| DumpMem | 以十六进制形式，在控制台窗口写一个内存块 |
| DumpRegs | 以十六进制形式显示 EAX、EEX、ECX、EDX、ESI、EDI、EBP、ESP、EFLAGS 和 EIP 寄存器。也显示最常见的 CPU 状态标志位 |
| GetCommandTail | 复制程序命名行参数（称为命令尾）到一个字节数组 |
| GetDateTime | 从系统获取当前日期和时间 |
| GetMaxXY | 返回控制台窗口缓冲器的行数和列数 |
| GetMseconds | 返回从午夜开始经过的毫秒数 |
| GetTextColor | 返回当前控制台窗口的前景色和背景色 |
| Gotoxy | 将光标定位到控制台窗口内指定的位置 |
| IsDigit | 如果 AL 寄存器中包含了十进制数字（0-9）的 ASCII 码，则零标志位置 1 |
| MsgBox | 显示一个弹出消息框 |
| MsgBoxAsk | 在弹出消息框中显示 yes/no 问题 |
| OpenlnputFile | 打开一个已有磁盘文件进行输入操作 |
| ParseDecimal32 | 将一个无符号十进制整数字符串转换为 32 位二进制数 |
| Parselnteger32 | 将一个有符号十进制整数字符串转换为 32 位二进制数 |
| Random32 | 在 0〜FFFFFFFFh 范围内，生成一个 32 位的伪随机整数 |
| Randomize | 用一个值作为随机数生成器的种子 |
| RandomRange | 在特定范围内生成一个伪随机整数 |
| ReadChar | 等待从键盘输入一个字符，并返回该字符 |
| ReadDec | 从键盘读取一个无符号 32 位十进制整数，用回车符结束 |
| ReadFromFile | 将一个输入磁盘文件读入缓冲区 |
| ReadHex | 从键盘读取一个 32 位十六进制整数，用回车符结束 |
| Readlnt | 从键盘读取一个有符号 32 位十进制整数，用回车符结束 |
| ReadKey | 无需等待输入即从键盘输入缓冲区读取一个字符 |
| ReadString | 从键盘读取一个字符串，用回车符结束 |
| SetTextColor | 设置控制台输出字符的前景色和背景色 |
| Str\_compare | 比较两个字符串 |
| Str\_copy | 将源字符串复制到目的字符串 |
| Str\_length | 用 EAX 返回字符串长度 |
| Str\_trim | 从字符串删除不需要的字符 |
| Str\_ucase | 将字符串转换为大写字母 |
| WaitMsg | 显示信息并等待按键操作 |
| WriteBin | 用 ASCII 二进制格式，向控制台窗口写一个无符号 32 位整数 |
| WriteBinB | 用字节、字或双字格式向控制台窗口写一个二进制整数 |
| WriteChar | 在控制台窗口写一个字符 |
| WriteDec | 用十进制格式，向控制台窗口写一个无符号 32 位整数 |
| WriteHex | 用十六进制格式，向控制台窗口写一个 32 位整数 |
| WriteHexB | 用十六进制格式，向控制台窗口写一个字节、字或双字整数 |
| Writelnt | 用十进制格式，向控制台窗口写一个有符号 32 位整数 |
| WriteStackFrame | 向控制台窗口写当前过程的堆栈帧 |
| WriteStackFrameName | 向控制台窗口写当前过程的名称和堆栈帧 |
| WriteString | 向控制台窗口写一个以空字符结束的字符串 |
| WriteToFile | 将缓冲区内容写入一个输出文件 |
| WriteWindowsMsg | 显示一个字符串，包含 MS-Windows 最近一次产生的错误 |

## 5.11 Irvine32链接库过程详细说明

本节将逐一介绍《[Irvine32链接库](http://c.biancheng.net/view/3543.html)》一节中 Irvine32 链接库中的过程是如何使用的，同时也会忽略一些更高级的过程，它们将在后续章节中进行解释。

### CloseFile

CloseFile 过程关闭之前已经创建或打开的文件（参见 CreateOutputFile 和 OpenlnputFile）。该文件用一个 32 位整数的句柄来标识，句柄由 EAX 传递。如果文件成功关闭，EAX 中的返回值就是非零的。示例如下：

```text
mov eax,fileHandle
call CloseFile
```

### Clrscr

Clrscr 过程清除控制台窗口。该过程通常在程序开始和结束时被调用。如果在其他时间调用这个过程，就需要先调用 WaitMsg 来暂停程序，这样就可以让用户在屏幕被清除之前，阅读屏幕上的信息。调用示例如下：

```text
call WaitMsg       ; "Press any key..."
call Clrscr
```

### CreateOutputFile

CreateOutputFile 过程创建并打开一个新的磁盘文件，进行写操作。调用该过程时，将文件名的偏移量送入 EDX。过程返回后，如果文件创建成功则 EAX 将包含一个有效文件句柄（32 位整数），否则，EAX 将等于 INVALID\_HANDLE\_VALUE（一个预定义的常数）。调用示例如下：

```text
.data
filename BYTE "newfile.txt",0
.code
mov edx,OFFSET filename
call CreateOutputFile
```

下面的伪代码描述的是调用 CreateOutputFile 之后，可能会出现的结果：

```text
if EAX = INVALID_HANDLE_VALUE
    the file was not created successfully
else
    EAX = handle for the open file
endif
```

### Crlf

Crlf 过程将光标定位在控制台窗口下一行的开始位置。它写的字符串包含了 ASCII 字符代码 ODh 和 OAh。调用示例如下：

```text
call Crlf
```

### Delay

Delay 过程按照特定毫秒数暂停程序。在调用 Delay 之前，将预定时间间隔送入 EAXO 调用示例如下：

```text
mov eax,1000              ;1 秒
call Delay
```

### DumpMen

DumpMen 过程在控制台窗口中用十六进制的形式显示一段内存区域。ESI 中存放的是内存区域首地址；ECX 中存放的是单元个数；EBX 中存放的是单元大小（1 = 字节，2 = 字，4 = 双字\)。下述调用示例用十六进制形式显示了包含 11 个双字的数组：

```text
.data
array DWORD 1,2,3,4,5,6,7,8,9,0Ah,0Bh
.code
main PROC
        mov esi,OFFSET array                   ;首地址偏移量
        mov ecx, LENGTHOF array                ;单元个数
        mov ebx,TYPE array                     ;双字格式
        call DumpMen
```

```text
00000001  00000002  00000003  00000004  00000005  00000006
00000007  00000008  00000009  0000000A  0000000B
```

### DumpRegs

DumpRegs 过程用十六进制形式显示 EAX、EBX、ECX、EDX、ESI、EDI、EBP、ESP、EIP 和 EFL（EFLAGS）的内容，以及进位标志位、符号标志位、零标志位、溢出标志位、辅助进位标志位和奇偶标志位的值。调用示例如下：

```text
call DumpRegs
```

示例输出如下所示：

```text
EAX=00000613  EBX=00000000  ECX=000000FF  EDX=00000000
ESI=00000000  EDI=00000100  EBP=0000091E  ESP=000000F6
EIP=00401026  EFL=00000286  CF=0  SF=1  ZF=0  OF=0  AF=0  PF=1
```

EIP 显示的数值是调用 DumpRegs 的下一条指令的偏移量。DumpRegs 在调试程序时很有用，因为它显示了 CPU 快照。该过程没有输入参数和返回值。

### GetCommandTail

GetCommandTail 过程将程序命令行复制到一个空字节结束的字符串。如果命令行是空，则进位标志位置 1 ；否则进位标志位清零。该过程的作用在于能让程序用户通过命令行传递参数。假设有一程序 Encrypt.exe 读取输入文件 filel.txt，并产生输出文件 file2.txt。程序运行时，用户可以通过命令行传递这两个文件名：

```text
Encrypt filel.txt file2.txt
```

当 Encrypt 程序启动时，它可以调用 GetCommandTail，检索这两个文件名。调用 GetCommandTail 时，EDX 必须包含一个数组的偏移量，该数组至少要有 129 个字节。调用示例如下：

```text
.data
cmdTail BYTE 129 DUP ( 0 )          ;空缓冲区
.code
mov edx,OFFSET cmdTail
call GetCommandTail                 ;填充缓冲区
```

在 Visual Studio 中运行应用程序时，有一种方法可以传递命令行参数。在 Project 菜单中，选择 Properties。在 Property Pages 窗口，展开 Configuration Properties 选项，选择 Debugging。然后，在右边 Command Arguments 面板的编辑行中输入程序的命令参数。

### GetMaxXY

GetMaxXY 过程获取控制台窗口缓冲区的大小。如果控制台窗口缓冲区大于可视窗口尺寸，则自动显示滚动条。GetMaxXY 没有输入参数。当过程返回时，DX 寄存器包含了缓冲区的列数，AX 寄存器包含了缓冲区的行数。每个数值的可能范围都不超过 255，这也许会小于实际窗口缓冲区的大小。调用示例如下：

```text
.data
rows BYTE ?
cols BYTE ?
.code
call GetMaxXY
mov rows, al
mov cols,dl
```

### GetMseconds

GetMseconds 过程获取主机从午夜开始经过的毫秒数，并用 EAX 返回该值。在计算事件间隔时间时，这个过程是非常有用的。过程不需要输入参数。

下面的例子调用了 GetMseconds，并保存了返回值。执行循环之后，代码第二次调用 GetMseconds，并将两次返回的时间值相减，结果就是执行循环的大致时间：

```text
.data
startTime DWORD ?
.code
call GetMseconds
mov startTime,eax
LI :
;(loop body)
loop LI
call GetMseconds
sub eax, startTime      ;EAX = 循环时间，按毫秒计
```

### GetTextColor

GetTextColor 过程获取控制台窗口当前的前景色和背景色，它没有输入参数。返回时，AL 中的高四位是背景色，低四位是前景色。调用示例如下：

```text
.data
color byte ?
.code
call GetTextColor
mov color,AL
```

### Gotoxy

Gotoxy 过程将光标定位到控制台窗口的指定位置。默认情况下，控制台窗口的X轴范围为 0〜79，Y 轴范围为 0〜24。调用 Gotoxy 时，将 Y 轴（行数）传递到 DH 寄存器，X 轴（列数）传递到 DL 寄存器。调用示例如下：

```text
mov dh, 10              ;第 10 行
mov dl, 20              ;第 20 列
call Gotoxy             ;定位光标
```

用户可能会修改控制台窗口大小，因此可以调用 GetMaxXY 获取当前窗口的行列数。

### IsDigit

IsDigit 过程确定 AL 中的数值是否是一个有效十进制数的 ASCII 码。过程被调用时，将一个 ASCII 字符传递到 AL。如果 AL 包含的是一个有效十进制数，则过程将零标志位置 1；否则，清除零标志位。调用示例如下：

```text
mov AL,somechar
call IsDigit
```

### MsgBox

MsgBox 过程显示一个带选择项的图形界面弹出消息框。（当程序运行于控制台窗口时有效。）过程用 EDX 传递一个字符串的偏移量，该字符串将显示在消息框中。还可以用 EBX 传递消息框标题字符串的偏移量，如果标题为空，则 EBX 为 0。调用示例如下：

```text
.data
caption BYTE "Survey Completed",0
question BYTE "Thank you for completing the survey."
BYTE 0dh,0ah
BYTE "Would you like to receive the results?",0
.code
mov ebx,OFFSET caption
mov edx,OFFSET question
call MsgBoxAsk                    ;查看 EAX 中的返回值
```

### MsgBoxAsk

MsgBoxAsk 过程显示带有 Yes 和 No 按钮的图形弹岀消息框。（当程序运行于控制台窗口时有效。）过程用 EDX 传递问题字符串的偏移量，该问题字符串将显示在消息框中。还可以用 EBX 传递消息框标题字符串的偏移量，如果标题为空，则 EBX 为 0。

MsgBoxAsk 用 EAX 中的返回值表示用户选择的是哪个按钮，返回值有两个选择，都是预先定义的 Windows 常数：IDYES （值为 6）或 IDNO（值为 7）。调用示例如下：

```text
.data
caption BYTE "Survey Completed",0
question BYTE "Thank you for completing the survey."
BYTE 0dh,0ah
BYTE "Would you like to receive the results?",0
.code
mov ebx,OFFSET caption
mov edx,OFFSET question
call MsgBoxAsk                    ;查看 EAX 中的返回值
```

### OpenlnputFile

OpenlnputFile 过程打开一个已存在的文件进行输入。过程用 EDX 传递文件名的偏移量。当从过程返回时，如果文件成功打开，则 EAX 就包含有效的文件句柄。 否则，EAX 等于 INVALID\_HANDLE\_VALUE（一个预定义的常数）。

调用示例如下：

```text
.data
filename BYTE "myfile.txt",0
.code
mov edx,OFFSET filename
call OpenlnputFile
```

下述伪代码显示了调用 OpenlnputFile 后可能的结果：

```text
if EAX = INVALID_HANDLE_VALUE
    the file was not opened successfully
else
    EAX = handle for the open file
endif
```

### ParseDecimal32

ParseDecimal32 过程将一个无符号十进制整数字符串转换为 32 位二进制数。非数字符号之前所有的有效数字都要转，前导空格要忽略。过程用 EDX 传递字符 串的偏移量，用 ECX 传递字符串的长度，用 EAX 返回二进制数值。

调用示例如下：

```text
.data
buffer BYTE "8193"
bufSize = ($ - buffer)
.code
mov edx,OFFSET buffer
mov ecx, bufSize
call Pars eDecimal32     ;返回 EAX
```

* 如果整数为空，则 EAX=0 且 CF=1
* 如果整数只有空格，则 EAX=0 且 CF=1
* 如果整数大于（2³²-1），则 EAX=0 且 CF=1
* 否则，EAX 为转换后的数，且 CF=0

  参阅 ReadDec 过程的说明，详细了解进位标志位是如何受到影响的。

### Parselnteger32

Parselnteger32 过程将一个有符号十进制整数字符串转换为32位二进 制数。字符串开始到第一个非数字符号之间所有的有效数字都要转，前导空格要忽略。过程用 EDX 传递字符串的偏移量，用 ECX 传递字符串的长度，用 EAX 返回二进制数值。调用示例如下：

```text
.data
buffer byte ,'-8193"
bufSize = ($ - buffer)
.code
mov edx,OFFSET buffer
mov ecx,bufSize
call Parselnteger32            ;返回 EAX
```

字符串可能包含一个前导加号或减号，但其后只能跟十进制数字。如果数值不能表示为 32 位有符号整数（范围：-2 147 483 648 到 +2 147 483 647），则溢出标志位置 1，且在控制 台显示一个错误信息。

### Random32

Random32 过程生成一个 32 位随机整数并用 EAX 返回该数。当被反复调用时，Random32 就会生成一个模拟的随机数序列，这些数由一个简单的函数产生，该函数有一个输入称为种子（seed）。

函数利用公式里的种子生成一个随机数值，并且每次都使用前次生成的随机数作为种子，来生成后续随机数。下述代码段展示了一个调用 Random32 的例子：

```text
.data
randVal DWORD ?
.code
call Random32
mov randVal, eax
```

### Randomize

Randomize 过程对 Random32 和 RandomRange 过程的第一个种子进行初始化。种子等于一天中的时间，精度为 1/100 秒。每当调用 Random32 和 RandomRaiige 的程序运行时，生成的随机数序列都不相同。而 Randomize 程只需要在程序开头调用一次。 下面的例子生成了 10 个随机整数：

```text
call Randomize
mov ecx,10
L1: call Random32
;在此使用或显示 EAX 中的随机数
loop L1
```

### RandomRange

RandomRange 过程在范围 0〜n-1 内生成一个随机整数，其中 n 是用 EAX 寄存器传递的输入参数。生成的随机数也用 EAX 返回。下面的例子在 0 到 4999 之间生成一个随机整数，并将其放在变量 randVal 中。

```text
.data
randVal DWORD ?
.code
mov eax,5000
call RandomRange
mov randVal, eax
```

### ReadChar

ReadChar 过程从键盘读取一个字符，并用 AL 寄存器返回，字符不在控制台窗口中回显。调用示例如下：

```text
.data
char BYTE ?
.code
call ReadChar
mov char,al
```

如果用户按下的是扩展键，如功能键、方向键、Ins 键或 Del 键，则过程就把 AL 清零，而 AH 包含的是键盘扫描码。EAX 的高字节没有使用。下述伪代码描述了调用 ReadChar 之后可能产生的结果：

```text
if an extended key was pressed
    AL = 0
    AH = keyboard scan code
else
    AL = ASCII key value
endif
```

### ReadDec

ReadDec 过程从键盘读取一个 32 位无符号十进制整数，并用 EAX 返回该值，前导空格要忽略。返回值为遇到第一个非数字字符之前的所有有效数字。比如，如果用户输入123AEC，则 EAX 中的返回值为 123。下面是一个调用示例：

```text
.data
intVal DWORD ?
.code
call ReadDec
mov intVal,eax
```

ReadDec 会影响进位标志位：

* 如果整数为空，则 EAX=0 且 CF=1
* 如果整数只有空格，则 EAX=0 且 CF=1
* 如果整数大于（2³²-1），则 EAX=0 且 CF=1
* 否则，EAX 为转换后的数，且 CF=0

### ReadFromFile

ReadFromFile 过程读取存储缓冲区中的一个输入磁盘文件。当调用 ReadFromFile 时，用 EAX 传递打开文件的句柄，用 EDX 传递缓冲区的偏移量，用 ECX 传递读取的最大字节数。

ReadFromFile 返回时要查看进位标志位的值：如果 CF 清零，则 EAX 包含了从文件中读取的字节数；如果 CF 置 1，则 EAX 包含了数字系统错误代码。调用 WriteWindowsMsg 程就可以获得该错误的文本。在下面的例子中，从文件读取的 5000 个字节复制到了缓冲区变量：

```text
.data
BUFFER_SIZE = 5000
buffer BYTE BUFFER_SIZE DUP(?)
bytesRead DWORD ?
.code
mov edx,OFFSET buffer         ;指向缓冲区
mov ecx,BUFFER_SIZE           ;读取的最大字节数
call ReadFromFile             ; 读文件 }
```

如果此时进位标志位清零，则可以执行如下指令：

```text
mov bytesRead, eax            ;实际读取的字节数
```

但是，如果此时进位标志位置 1，就可以调用 WriteWindowsMsg 过程，显示错误代码以及该应用最近产生错误的说明：

```text
call WriteWindowsMsg
```

### ReadHex

ReadHex 过程从键盘读取一个 32 位十六进制整数，并用 EAX 返回相应的二进制数。对无效字符不进行任何错误检查。字母 A 到 F 的大小写都可以使用。最多能够输入 8 个数字（超出的字符将被忽略），前导空格将被忽略。调用示例如下：

```text
.data
hexVal DWORD ?
.code
call ReadHex
mov hexVal,eax
```

### Readlnt

Readlnt 过程从键盘读取一个 32 位有符号整数，并用 EAX 返回该值。用户可以键入前置加号或减号，而其后跟的只能是数字。

Readlnt 设置溢出标志位，如果输入数值无法表示为 32 位有符号数（范围：-2 147 483 648 至 +2 147 483 647），则显示一个错误信息。返回值包括所有的有效数字，直到遇见第一个非数字字符。例如，如果用户输入 +123ABC，则返回值为 +123。调用示例如下：

```text
.data
intVal SDWORD ?
.code
call Readlnt
mov intVal,eax
```

### ReadKey

ReadKey 过程执行无等待键盘检查。换句话说，它检查键盘输入缓冲区以查看用户是否有按键操作。如果没有发现键盘数据，则零标志位置 1。如果 ReadKey 发现有按键，则清除零标志位，且向 AL 送入 0 或 ASCII 码。若 AL 为 0，表示用户可能按下了一个特殊键（功能键、方向键等）。

AH 寄存器为虚拟扫描码，DX 为虚拟键码，EBX 为键盘标志位。下述伪代码说明了调用 ReadKey 时的各种结果：

```text
if no_keyboard_data then
    ZF = 1
else
    ZF = 0
    if AL = 0 then
        extended key was pressed, and AH = scan code, DX = virtual
           key code, and EBX = keyboard flag bits
    else
        AL = the key's ASCII code
    endif
endif
```

当调用 ReadKey 时，EAX 和 EDX 的高 16 位会被覆盖。

### ReadString

ReadString 过程从键盘读取一个字符串，直到用户键入回车键。过程用 EDX 传递缓冲区的偏移量，用 ECX 传递用户能键入的最大字符数加 1（保留给终止空字节），用 EAX 返回用户键入的字符数。示例调用如下：

```text
.data
buffer BYTE 21 DUP(0)          ;输入缓冲区
byteCount DWORD ?              ;定义计数器
.code
mov edx,OFFSET buffer           ;指向缓冲区
mov ecxz SIZEOF buffer          ;定义最大字符数
call ReadString                 ;输入字符串
mov byteCount, eax              ;字符数
```

ReadString 在内存中字符串的末尾自动插入一个 null 终止符。用户输入“ABCDEFG”后，buffer 中前 8 个字节的十六进制形式和 ASCII 形式如下所示：

```text
41 42 43 44 45 46 47 00 ABCDEFG
```

变量 byteCoun t等于 7。

### SetTextColor

SetTextColor 过程（仅在 Irvine32 链接库中）设置输出文本的前景色和背景色。调用 SetTextColor 时，给 EAX 分配一个颜色属性。下列预定义的颜色常数都可以用于前景色和背景色：

| black = 0 | red = 4 | gray = 8 | lightRed = 12 |
| :--- | :--- | :--- | :--- |
| blue = 1 | magenta = 5 | lightBlue = 9 | light Magenta = 13 |
| green = 2 | brown = 6 | light Green = 10 | yellow = 14 |
| cyan = 3 | lightGray = 7 | lightCyan = 11 | white = 15 |

颜色常量在 Irvine32.inc 文件中进行定义。要获得完整的颜色字节数值，就将背景色乘以 16 再加上前景色。例如，下述常量表示在蓝色背景上输出黄色字符：

```text
yellow + (blue * 16)
```

下列语句设置为蓝色背景上输出白色字符：

```text
mov eax,white + (blue * 16)   ; 蓝底白字
call SetTextColor
```

另一种表示颜色常量的方法是使用 SHL 运算符，将背景色左移 4 位再加上前景色。

```text
yellow + (blue SHL 4)
```

位移是在汇编时执行的，因此它只能用常数作操作数。

### Str\_length

Str\_length 过程返回空字节结束的字符串的长度。过程用 EDX 传递字符串的偏移量，用 EAX 返回字符串的长度。调用示例如下：

```text
.data
buffer BYTE "abcde",0
bufLength DWORD ?
.code
mov edx, OFFSET buffer          ;指向字符串
call Str_length                 ;EAX=5
mov bufLength, eax              ;保存长度
```

### WaitMsg

WaitMsg 过程显示“Press any key to continue…”消息，并等待用户按键。当用户想在数据滚动和消失之前暂停屏幕显示时，这个过程就很有用。过程没有输入参数。 调用示例如下：

```text
call WaitMsg
```

### WriteBin

WriteBin 过程以 ASCII 二进制格式向控制台窗口输出一个整数。过程用 EAX 传递该整数。为了便于阅读，二进制位以四位一组的形式进行显示。调用示例如下：

```text
mov eax,12346AF9h
call WriteBin
```

示例代码显示如下：

```text
0001 0010 0011 0100 0110 1010 1111 1001
```

### WriteBinB

WriteBinB 过程以 ASCII 二进制格式向控制台窗口输出一个 32 位整数。过程用 EAX 寄存器传递该整数，用 EDX 表示以字节为单位的显示大小（1、2，或 4）。为了便于阅读，二进制位以四位一组的形式进行显示。调用示例如下：

```text
mov eax,00001234h
mov ebx,TYPE WORD           ; 两个字节
call WriteBinB              ; 显示 0001 0010 0011 0100
```

### WriteChar

WriteChar 过程向控制台窗口写一个字符。过程用 AL 传递字符（或其 ASCII 码）。调用示例如下：

```text
mov al, 'A'
call WriteChar           ;显示："A"
```

### WriteDec

WriteDec 过程以十进制格式向控制台窗口输出一个 32 位无符号整数，且没有前置 0。过程用 EAX 寄存器传递该整数。调用示例如下：

```text
mov eax,295
call WriteDec            ;显示："295"
```

### WriteHex

WriteHex 过程以 8 位十六进制格式向控制台窗口输出一个 32 位无符号整数，如果需要，应插入前置 0。过程用 EAX 传递整数。调用示例如下：

```text
mov eax,7FFFh
call WriteHex           ;显示："00007FFF"
```

### WriteHexB

WriteHexB 过程以十六进制格式向控制台窗口输岀一个 32 位无符号整数，如果需要，应插入前置 0。过程用 EAX 传递整数，用 EBX 表示显示格式的字节数（1、2，或 4）。调用示例如下：

```text
mov eax, 7FFFh
mov ebx, TYPE WORD        ;两个字节
call WriteHexB            ;显示："7FFF"
```

### Writelnt

Writelnt 过程以十进制向控制台窗口输岀一个 32 位有符号整数，有前置符号，但没有前置 0。过程用 EAX 传递整数。调用示例如下：

```text
mov eax, 216543
call Writelnt             ;显示："+216543"
```

### WriteString

WriteString 过程向操作台窗口输出一个空字节结束的字符串。过程用 EDX 传递字符串的偏移量。调用示例如下：

```text
.data
prompt BYTE "Enter your name: ",0
.code
mov edx,OFFSET prompt
call WriteString
```

### WriteToFile

WriteToFile 过程向一个输出文件写入缓冲区内容。过程用 EAX 传递有效的文件句柄，用 EDX 传递缓冲区偏移量，用 ECX 传递写入的字节数。当过程返回时，如果 EAX 大于 0，则其包含的是写入的字节数；否则，发生错误。下述代码调用了 WriteToFile：

```text
BUFFER_SIZE = 5000
.data
fileHandle DWORD ?
buffer BYTE BUFFER_SIZE DUP(?)
.code
mov eax, fileHandle
mov edx, OFFSET buffer
mov ecx, BUFFER SIZE
call WriteToFile
```

下面的伪代码说明了调用 WriteToFile 之后对 EAX 返回值的处理：

```text
if EAX = 0 then
    error occurred when writing to file
    call WriteWindowsMessage to see the error
else
    EAX = number of bytes written to the file
endif
```

### WriteWindowsMsg

WriteWindowsMsg 过程向控制台窗口输出应用程序在调用系统函数时最近产生的错误信息。调用示例如下：

```text
call WriteWindowsMsg
```

下面的例子展示了一个消息字符串：

```text
Error 2: The system cannot find the file specified.
```

## 5.12 Irvine64链接库

本教程提供了一个能支持 64 位编程的最小链接库，其中包含了如下过程：

* Crlf：向控制台写一个行结束的序列。
* Random64：在0〜2⁶⁴-1 内，生成一个 64 位的伪随机整数。随机数值用 RAX 寄存器返回。
* Randomize：用一个值作为随机数生成器的种子。
* Readlnt64：从键盘读取一个 64 位有符号整数，用回车符结束。数值用 RAX 寄存器返回。
* ReadString：从键盘读取一个字符串，用回车符结束。过程用 RDX 传递输入缓冲器偏移量；用 RCX 传递用户可输入的最大字符数加 1（用于 unll 结束符字节）。返回值（用 RAX）为用户实际输入的字符数。
* Str\_compare：比较两个字符串。过程将源串指针传递给 RSI，将目的串指针传递给 RDIO 用与 CMP（比较）指令一样的方式设置零标志位和进位标志位。
* Str\_copy：将一个源串复制到目标指针指定的位置。源串偏移量传递给 RSI，目标偏移量传递给 RDI。
* Strjength：用 RAX 寄存器返回一个空字节结束的字符串的长度。过程用 RCX 传递字符串的偏移量。
* Writelnt64：将 RAX 寄存器中的内容显示为 64 位有符号十进制数，并加上前置加号或减号。过程没有返回值。
* WriteHex64：将 RAX 寄存器中的内容显示为 64 位十六进制数。过程没有返回值。
* WriteHexB：将 RAX 寄存器中的内容显示为 1 字节、2 字节、4 字节或 8 字节的十六进制数。将显示的大小（1、2、4 或 8\) 传递给 RBX 寄存器。过程没有返回值。
* WriteString：显示一个空字节结束的 ASCII 字符串。将字符串的 64 位偏移量传递给 RDX。过程没有返回值。

尽管这个库比 32 位链接库小很多，它还是包含了许多重要工具能使得程序更具互动性。随着学习的深入，大家可以用自己的代码来扩展这个链接库。Irvine64 链接库会保留 RBX、RBP、RDI、RSI、R12、R13、R14 和 R15 寄存器的值，反之，RAX、RCX、RDX、R8、R9、R10 和 R11 寄存器的值则不会保留。

### 调用 64 位子程序

如果想要调用自己编写的子程序，或是 Irvine64 链接库中的子程序，则程序员需要做的就是将输入参数送入寄存器，并执行 CALL 指令。比如：

```text
mov rax,12345678h
call WriteHex64
```

还有一件小事也需要完成，即程序员要在自己程序的顶部用 PROTO 伪指令指定所有在本程序之外同时又将会被调用的过程：

```text
ExitProcess PROTO         ;位于 Windows API
WriteHex64 PROTO          ;位于 Irvine64 链接库
```

### x64 调用规范

Microsoft 在 64 位程序中使用统一模式来传递参数并调用过程，称为 Microsoft x64 调用规范。该规范由 C/[C++](http://c.biancheng.net/cplus/) 编译器和 Windows 应用编程接口（API）使用。

程序员只有在调用 Windows API 的函数或用 C/C++ 编写的函数时，才会使用这个调用规范。该调用规范的一些基本特性如下所示：

1\) CALL 指令将 RSP（堆栈指针）寄存器减 8，因为地址是 64 位的。

2\) 前四个参数依序存入 RCX、RDX、R8 和 R9 寄存器，并传递给过程。如果只有一个参数，则将其放入 RCX。如果还有第二个参数，则将其放入 RDX，以此类推。其他参数，按照从左到右的顺序压入堆栈。

3\) 调用者的责任还包括在运行时堆栈分配至少 32 字节的影子空间（shadow space），这样，被调用的过程就可以选择将寄存器参数保存在这个区域中。

4\) 在调用子程序时，堆栈指针（RSP）必须进行 16 字节边界对齐（16 的倍数）。CALL 指令把 8 字节的返回值压入堆栈，因此，除了已经减去的影子空间的 32 之外，调用程序还必须从堆栈指针中减去 8。后面的示例将显示如何实现这些操作。

> 提示：调用 Irvine64 链接库中的子程序时，不需使用 Microsoft x64 调用规范；只在调用 Windows API 函数时使用它。

### 调用过程示例

现在编写一段小程序，使用 Microsoft x64 调用规范来调用子程序 AddFour。这个子程序将四个参数寄存器（RCX、RDX、R8 和 R9）的内容相加，并将和数保存到 RAX。

由于过程通常使用 RAX 返回结果，因此，当从子程序返回时，调用程序也期望返回值在这个寄存器中。这样就可以说这个子程序是一个函数，因为，它接收了四个输入并（确切地说）产生了一个输出。

```text
;在64模式下调用子程序

ExitProcess PROTO
WriteInt64 PROTO          ;Irvine64链接库
Crlf PROTO                ;Irvine64链接库

.code
main PROC
    sub    rsp,8            ;对准堆栈指针
    sub    rsp,20h          ;为影子参数保留32个字节

    mov    rcx,1            ;依序传递参数
    mov    rdx,2
    mov    r8,3
    mov    r9,4
    call AddFour            ;在RAX中查找返回值
    call WriteInt64         ;显示数字
    call Crlf               ;输出回车换行符

    mov    ecx,0
    call ExitProcess
main ENDP

AddFour PROC
    mov rax,rcx
    add    rax,rdx
    add    rax,r8
    add    rax,r9            ;和数保存在RAX中
    ret
AddFour ENDP

END
```

现在来看看本例中的其他细节：第 10 行将堆栈指针对齐到 16 字节的偶数边界。为什么要这样做？在 OS 调用主程序之前，假设堆栈指针是对齐 16 字节边界的。然后，当 OS 调用主程序时，CALL 指令将 8 字节的返回地址压入堆栈。将堆栈指针再减去 8，使其减少成一个 16 的倍数。

可以在 Visual Studio 调试器中运行该程序，并查看 RSP 寄存器（堆栈指针）改变数值。通过这个方法，能够看到用图形方式在下图中展示的十六进制数值。

![&#x7A0B;&#x5E8F;&#x7684;&#x8FD0;&#x884C;&#x65F6;&#x5806;&#x6808;](http://c.biancheng.net/uploads/allimg/190507/4-1Z50G02034253.gif)

上图只展示了每个地址的低 32 位，因为高 32 位为全零：

1\) 执行第 10 行前，RSP=01AFE48。这表示在 OS 调用本程序之前，RSP 等于 01AFE50。（ CALL 指令使得堆栈指针减 8。）

2\) 执行第 10 行后，RSP=01AFE40，表示堆栈正好对齐到 16 字节边界。

3\) 执行第 11 行后，RSP=01AFE20，表示 32 个字节的影子空间位置从 01AFE20 到 01AFE3F。

4\) 在 AddFour 过程中，RSP=01AFE18，表示调用者的返回地址已经压入堆栈。

5\) 从 AddFour 返回后，RSP 再一次等于 01AFE20，与调用 AddFour 之前的值相同。

与调用 ExitProcess 来结束程序相比，本程序选择的是执行 RET 指令，这将返回到启动本程序的过程。但是，这也就要求能将堆栈指针恢复到其在 main 程开始执行时的位置。下面的代码行能替代 CallProc\_64 程序的第 20 和 21 行：

```text
add rsp,28         ;恢复堆栈指针
mov ecx,0          ;过程返回码
ret                ;返回 OS
```

> 提示：要使用 Irvine64 链接库，将 Irvine64.obj 文件添加到用户的 Visual Studio 项目中。Visual Studio 中的操作步骤如下：在 Solution Explorer 窗口中右键点击项目名称，选择 Add，选择 Existing Item，再选择 Irvine64.obj 文件名。

