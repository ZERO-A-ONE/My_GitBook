---
description: >-
  本章将介绍子程序调用的底层结构，重点集中于运行时堆栈。本章的内容对 C 和 C++
  程序员也是有价值的，因为在调试运行于操作系统或设备驱动程序层的底层子程序时，他们
  也经常必须检查运行时堆栈的内容。大多数现代编程语言在调用子程序之前都会把参数压入堆栈。反过来，子程序也常常把它们的局部变量压入堆栈。本章还将讲解如何以数值或引用的形式来传递参数，如何定义和撤销局部变量，以及如何实现递归。最后介绍了
---

# 8 汇编语言高级过程

## 8.1 汇编语言堆栈帧简介

在之前的章节中，子程序接收的是寄存器参数。比如在 Irvine32 链接库中就是如此。接下来将展示子程序如何用堆栈接收参数。

在 32 位模式下，堆栈参数总是由 Windows API 函数使用。然而在 64 位模式下，Windows 函数可以同时接收寄存器参数和堆栈参数。

堆栈帧 \(stack frame\)\( 或活动记录 \(activation Tecord\)\) 是一块堆栈保留区域，用于存放被传递的实际参数、子程序的返回值、局部变量以及被保存的寄存器。

堆栈帧的创建步骤如下所示：

1\) 被传递的实际参数。如果有，则压入堆栈。

2\) 当子程序被调用时，使该子程序的返回值压入堆栈。

3\) 子程序开始执行时，EEP 被压入堆栈。

4\) 设置 EBP 等于 ESP。从这时开始，EBP 就变成了该子程序所有参数的引用基址。

5\) 如果有局部变量，修改 ESP 以便在堆栈中为这些变量预留空间。

6\) 如果需要保存寄存器，就将它们压入堆栈。

程序内存模式和对参数传递规则的选择直接影响到堆栈帧的结构。

学习用堆栈传递参数有个好理由：几乎所有的高级语言都会用到它们。比如，如果想要在 32 位 Windows 应用程序接口 \(API\) 中调用函数，就必须用堆栈传递参数。而 64 位程序可以使用另一种不同的参数传递规则。

## 8.2 寄存器参数的缺点

多年以来，Microsoft 在 32 位程序中包含了一种参数传递规则，称为 fastcall。如同这个名字所暗示的，只需简单地在调用子程序之前把参数送入寄存器，就可以将运行效率提高一些。相反，如果把参数压入堆栈，则执行速度就要更慢一点。

典型用于参数的寄存器包括 EAX、EBX、ECX 和 EDX，少数情况下，还会使用 EDI 和 ESI。可惜的是，这些寄存器在用于存放数据的同时，还用于存放循环计数值以及参与计算的操作数。

因此，在过程调用之前，任何存放参数的寄存器须首先入栈，然后向其分配过程参数，在过程返回后再恢复其原始值。例如，如下代码从 Irvine32 链接库中调用了 DumpMem：

```text
push ebx               ;保存寄存器值
push ecx
push esi
mov esi,OFFSET array   ;初始 OFFSET
mov ecx,LENGTHOF array ;大小，按元素个数计
mov ebx, TYPE array    ;双字格式
call DumpMem           ;显示内存
pop esi                ;恢复寄存器值
pop ecx
pop ebx
```

这些额外的入栈和出栈操作不仅会让代码混乱，还有可能消除性能优势，而这些优势正是通过使用寄存器参数所期望获得的！此外，程序员还要非常仔细地将 PUSH 与相应的 POP 进行匹配，即使代码存在着多个执行路径。

例如，在下面的代码中，第 8 行的 EAX 如果等于 1，那么过程在第 17 行就无法返回其调用者，原因就是有三个寄存器的值留在运行时堆栈里。

```text
push ebx                   ;保存寄存器值
push ecx
push esi
mov esi, OFFSET array      ;初始 OFFSET
mov ecx, LENGTHOF array    ;大小，按元素个数计
mov ebx, TYPE array        ;双字格式
call DumpMem               ;显示内存
cmp eax, 1                 ;设置错误标志？
je error_exit              ;设置标志后退出

pop esi                    ;恢复寄存器值
pop ecx
pop ebx
ret
error_exit:
mov edx, offset error_msg
ret
```

不得不说，像这样的错误是不容易发现的，除非是花了相当多的时间来检查代码。

堆栈参数提供了一种不同于寄存器参数的灵活方法：只需要在调用子程序之前，将参数压入堆栈即可。比如，如果 DumpMem 使用了堆栈参数，就可以通过如下代码对其进行调用：

```text
push TYPE array
push LENGTHOF array
push OFFSET array
call DumpMem
```

子程序调用时，有两种常见类型的参数会入栈：

* 值参数（变量和常量的值）
* 引用参数（变量的地址）

### 值传递

当一个参数通过数值传递时，该值的副本会被压入堆栈。假设调用一个名为 AddTwo 的子程序，向其传递两个 32 位整数：

```text
.data
val1 DWORD 5
val2 DWORD 6
.code
push val2
push val1
call AddTwo
```

执行 CALL 指令前，堆栈如下图所示：

![img](http://c.biancheng.net/uploads/allimg/190513/4-1Z5131J30TQ.gif)

用 [C++](http://c.biancheng.net/cplus/) 编写相同的功能调用则为

```c
int sum = AddTwo(val1, val2);
```

观察发现参数入栈的顺序是相反的，这是 C 和 C++ 语言的规范。

### 引用传递

通过引用来传递的参数包含的是对象的地址（偏移量）。下面的语句调用了 Swap，并传递了两个引用参数：

```text
push OFFSET val2
push OFFSET val1
call Swap
```

调用 Swap 之前，堆栈如下图所示：

![img](http://c.biancheng.net/uploads/allimg/190513/4-1Z5131J33RY.gif)

在 C/C++ 中，同样的函数调用将传递 val1 和 val2 参数的地址：

```c
Swap(&vail, &val2);
```

### 传递数组

高级语言总是通过引用向子程序传递数组。也就是说，它们把数组的地址压入堆栈。然后，子程序从堆栈获得该地址，并用其访问数组。

不愿意用值来传递数组的原因是显而易见的，因为这样就会要求将每个数组元素分别压入堆栈。这种操作不仅速度很慢，而且会耗尽宝贵的堆栈空间。

下面的语句用正确的方法向子程序 ArrayFill 传递了数组的偏移量：

```text
.data
array DWORD 50 DUP(?)
.code
push OFFSET array
call ArrayFill
```

## 8.3 访问堆栈参数详解

高级语言有多种方式来对函数调用的参数进行初始化和访问。以 C 和 [C++](http://c.biancheng.net/cplus/) 语言为例，它们以保存 EBP 寄存器并使该寄存器指向栈顶的语句为开始 \(prologue\)。

然后，根据实际情况，它们可以把某些寄存器入栈，以便在函数返回时恢复这些寄存器的值。在函数结尾 \(epilogue\) 部分，恢复 EBP 寄存器，并用 RET 指令返回调用者。

#### AddTwo示例

下面是用 C 编写的 AddTwo 函数，它接收了两个值传递的整数，然后返回这两个数之和：

```c
int AddTwo( int x, int y )
{
    return x + y;
}
```

现在用[汇编语言](http://c.biancheng.net/asm/)实现同样的功能。在函数开始的时候，AddTwo 将 EBP 入栈，以保存其当前值：

```text
AddTwo PROC
  push ebp
```

接下来，EBP 的值被设置为等于 ESP，这样 EBP 就成为 AddTwo 堆栈帧的基址指针：

```text
AddTwo PROC
  push ebp
  mov ebp,esp
```

执行了上面两条指令后，堆栈帧的内容如下图所示。而形如 AddTwo\(5, 6\) 的函数调用会先把第一个参数入栈，再把第二个参数入栈：

![img](http://c.biancheng.net/uploads/allimg/190514/4-1Z514100QHU.gif)

AddTwo 在其他寄存器入栈时，不用通过 EEP 来修改堆栈参数的偏移量。数值会改变的是 ESP，而 EBP 则不会。

#### 基址-偏移量寻址

可以使用基址-偏移量寻址 \(base-offset addressing\) 方式来访问堆栈参数。其中，EBP 是基址寄存器，偏移量是常数。通常，EAX 为 32 位返回值。AddTwo 的实现如下所示，参数相加后，EAX 返回它们的和数：

```text
AddTwo PROC
    push ebp
    mov ebp, esp              ;堆栈帧的基址
    mov eax, [ebp + 12]       ;第二个参数
    add eax, [ebp + 8]        ;第一个参数 pop ebp
    ret
AddTwo ENDP
```

#### 显式的堆栈参数

若堆栈参数的引用表达式形如 \[ebp+8\]，则称它们为显式的堆栈参数 \(explicit stack parameters\)。这个名称的含义是：汇编代码显式地说明了参数的偏移量是一个常数。有些程序员定义符号常量来表示显式的堆栈参数，以使其代码更具易读性：

```text
y_param EQU [ebp + 12]
x_param EQU [ebp + 8]

AddTwo PROC
    push ebp
    mov ebp,esp
    mov eax,y_param
    add eax,x_param
    pop ebp
    ret
AddTwo ENDP
```

#### 清除堆栈

子程序返回时，必须将参数从堆栈中删除。否则将导致内存泄露，堆栈就会被破坏。例如，设如下语句在 main 中调用 AddTwo：

```text
push 6
push 5
call AddTwo
```

假设 AddTwo 有两个参数留着堆栈中，下图所示为调用返回后的堆栈：

![img](http://c.biancheng.net/uploads/allimg/190514/4-1Z51410092K63.gif)

main 部分试图忽略这个问题，并希望程序能正常结束。但是，如果循环调用 AddTwo，堆栈就会溢出。因为每次调用都会占用 12 字节的堆栈空间——每个参数需要 4 个字节，再加 4 个字节留给 CALL 指令的返回地址。如果在 main 中调用 Example1，而它又要调用 AddTwo 就会导致更加严重的问题：

```text
main PROC
    call Example1
    exit
main ENDP

Example1 PROC
    push 6
    push 5
    call AddTwo
    ret                   ;堆栈被破坏了！
Example1 ENDP
```

当 Example1 的 RET 指令将要执行时，ESP 指向整数 5 而不是能将其带回 main 的返回地址：

![img](http://c.biancheng.net/uploads/allimg/190514/4-1Z5141009504O.gif)

RET 指令把整数 5 加载到指令指针寄存器，尝试将控制转移到内存地址为 5 的位置。假设这个地址在程序代码边界之外，那么处理器将给出运行时异常，通知 OS 终止程序。

## 8.4 常用32位编程调用规范简介

本节将给出 Windows 环境中两种最常用的 32 位编程调用规范。首先是 C 语言发布的 C 调用规范，该语言用于 Unix 和 Windows。然后是 STDCALL 调用规范，它描述了调用 Windows API 函数的协议。这两种规范都很重要，因为在 C 和 [C++](http://c.biancheng.net/cplus/) 程序中会调用汇编函数, 同时[汇编语言](http://c.biancheng.net/asm/)程序也会调用大量的 Windows API 函数。

### C 调用规范

C 调用规范用于 C 和 C++ 语言。子程序的参数按逆序入栈，因此，C 程序在调用如下函数时，先将 B 入栈，再将 A 入栈：

```c
AddTwo(A, B)
```

C 调用规范用一种简单的方法解决了清除运行时堆栈的问题：程序调用子程序时，在 CALL 指令的后面紧跟一条语句使堆栈指针（ESP）加上一个数，该数的值即为子程序参数所占堆栈空间的总和。下面的例子在执行 CALL 指令之前，将两个参数（5 和 6）入栈：

```text
Example1 PROC
    push 6
    push 5
    call AddTwo
    add esp, 8        ;从堆栈移除参数
    ret
Example1 ENDP
```

因此，用 C/C++ 编写的程序在从子程序返回后，总是能把参数从堆栈中删除。

### STDCALL 调用规范

另一种从堆栈删除参数的常用方法是使用名为 STDCALL 的规范。如下所示的 AddTwo 过程给 RET 指令添加了一个整数参数，这使得程序在返回到调用过程时，ESP 会加上数值 8。这个添加的整数必须与被调用过程参数占用的堆栈空间字节数相等：

```text
AddTwo PROC
    push ebp
    mov ebp,esp                   ;堆栈帧基址
    mov eax, [ebp + 12 ]       ;第二个参数
    add eax, [ebp + 8 ]        ;第一个参数
    pop ebp
ret 8                           ;清除堆栈
AddTwo ENDP
```

要说明的是，STDCALL 与 C 相似，参数是按逆序入栈的。通过在 RET 指令中添加参数，STDCALL 不仅减少了子程序调用产生的代码量（减少了一条指令），还保证了调用程序永远不会忘记清除堆栈。

另一方面，C 调用规范则允许子程序声明不同数量的参数，主调程序可以决定传递多少个参数。C 语言的 printf 函数就是一个例子，它的参数数量取决于初始字符串参数中的格式说明符的个数：

```c
int x = 5;
float y = 3.2;
char z = 'Z';
printf("Printing values: %d, %f, %c", xz y, z);
```

C 编译器按逆序将参数入栈，被调用的函数负责确定要传递的实际参数的个数，然后依次访问参数。这种函数实现没有像给 RET 指令添加一个常数那样简便的方法来清除堆栈，因此，这个责任就留给了主调程序。

调用 32 位 Windows API 函数时，Irvine32 链接库使用的是 STDCALL 调用规范。Irvine64 链接库使用的是 x64 调用规范。

### 保存和恢复寄存器

通常，子程序在修改寄存器之前要将它们的当前值保存到堆栈。这是一个很好的做法，因为可以在子程序返回之前恢复寄存器的原始值。理想情况下，相关寄存器入栈应在设置 EBP 等于 ESP 之后，在为局部变量保留空间之前。这有利于避免修改当前堆栈参数的偏移量。

例如，假设如下过程 MySub 有一个堆栈参数。在 EBP 被设置为堆栈帧基址后，ECX 和 EDX 入栈，然后堆栈参数加载到 EAX：

```text
MySub PROC
    push ebp                ;保存基址指针
    mov ebp,esp             ;堆栈帧基址
    push ecx
    push edx                ;保存 EDX
    mov eax,[ebp+8]         ;取堆栈参数
    .
    .
    pop    edx               ;恢复被保存的寄存器
    pop ecx
    pop    ebp               ;恢复基址指针
    ret                      ;清除堆栈
MySub ENDP
```

EBP 被初始化后，在整个过程期间它的值将保持不变。ECX 和 EDX 的入栈不会影响到已入栈参数与 EBP 之间的位移量，因为堆栈的增长位于 EBP 的下方，如下图所示。

![img](http://c.biancheng.net/uploads/allimg/190514/4-1Z514112121D6.gif)

## 8.5 局部变量应用

高级语言中，在单一子程序内新建、使用和撤销的变量被称为局部变量 \(local variable\)。局部变量创建于运行时堆栈，通常位于基址指针 \(EBP\) 之下。

尽管不能在汇编时给它们分配默认值，但是能在运行时初始化它们。可以使用与 C 和 [C++](http://c.biancheng.net/cplus/) 相同的方法在[汇编语言](http://c.biancheng.net/asm/)中新建局部变量。

【示例】下面的 C++ 函数声明了局部变量 X 和 Y：

```text
void MySub()
{
    int X = 10;
    int Y = 20;
}
```

如果这段代码被编译为机器语言，就能看出局部变量是如何分配的。每个堆栈项都默认为 32 位，因此，每个变量的存储大小都要向上取整保存为 4 的倍数。两个局部变量一共要保留 8 个字节：

| 变量 | 字节数 | 堆栈偏移量 |
| :--- | :--- | :--- |
| X | 4 | EBP-4 |
| Y | 4 | EBP-8 |

MySub 函数（在调试器中）的反汇编展示了 C++ 程序如何创建局部变量，以及如何从堆栈中删除它们。该例使用了 C 调用规则：

```text
MySub PROC
    push ebp
    mov ebp, esp
    sub esp, 8                    ;创建局部变量
    mov DWORD PTR [ebp-4],10      ; X
    mov DWORD PTR [ebp-8],20      ; Y
    mov esp, ebp                  ;从堆栈中删除局部变量
    pop ebp
    ret
MySub ENDP
```

局部变量初始化后，函数的堆栈帧如下图所示。

![img](http://c.biancheng.net/uploads/allimg/190514/4-1Z514125642a3.gif)

在结束前，函数通过将 EBP 的值赋给堆栈指针完成对其的重置，该操作的效果是把局部变量从堆栈中删除：

```text
mov esp, ebp     ;从堆栈中删除局部变量
```

如果省略这一步，那么 POP EBP 指令将会把 EBP 设置为 20，而 RET 指令就会分支到内存地址 10 的位置，从而导致程序因出现处理器异常而终止。下面的 MySub 代码就是这种情况：

```text
MySub PROC
    push ebp
    mov ebp, esp
    sub esp, 8                      ; 创建局部变量
    mov DWORD PTR [ebp-4], 10    ; X
    mov DWORD PTR [ebp-8], 20    ; Y
    pop ebp
    ret                             ; 返回到无效地址！
MySub ENDP
```

### 局部变量符号

为了使程序更加易读，可以为每个局部变量的偏移量定义一个符号，然后在代码中使用这些符号：

```text
X_local EQU DWORD PTR [ebp-4]
Y_local EQU DWORD PTR [ebp-8]

MySub PROC
    push ebp
    mov ebp, esp
    sub esp, 8              ; 为局部变量保留空间
    mov X_local, 10         ; X
    mov Y_local, 20         ; Y
    mov esp, ebp            ;从堆栈中删除局部变量
    pop ebp
    rst
MySub ENDP
```

## 8.6 引用参数简介

引用参数通常是由过程用基址-偏移量寻址（从 EBP）方式进行访问。由于每个引用参数都是一个指针，因此，常常作为一个间接操作数放在寄存器中。例如，假设堆栈地址 \[ebp+12\] 存放了一个数组指针，则下述语句就把该指针复制到 ESP 中：

```text
mov esi, [ebp+12 ]  ;指向数组
```

【示例】下面将要展示的 ArrayFill 过程用 16 位整数的伪随机序列来填充数组。它接收两个参数：数组指针和数组长度，第一个为引用传递，第二个为值传递。调用示例如下：

```text
.data
count = 100
array WORD count DUP(?)

.code
main PROC
    push OFFSET array
    push COUNT
    call ArrayFill
```

在 ArrayFill 中，下面的代码为其开始部分，对堆栈帧指针（EBP）进行初始化：

```text
ArrayFill PROC   
    push ebp
    mov ebp,esp
```

现在，堆栈帧中包含了数组偏移量、数组长度（count）、返回地址以及被保存的 EBP：

![img](http://c.biancheng.net/uploads/allimg/190514/4-1Z514132P3H1.gif)

ArrayFill 保存了通用寄存器，检索参数并填充数组：

```text
ArrayFill PROC   
    push ebp
    mov ebp,esp
    pushad                ; 保存寄存器
    mov esi,              ; 数组偏移量
    mov ecx,[ebp+8]       ; 数组长度
    cmp ecx,0             ; ECX == 0?
    je L2                 ; 是: 跳过循环

L1:
    mov eax,10000h        ; 随机范围 0 - FFFFh
    call RandomRange      ; 从链接库生成随机数
    mov [esi],ax          ; 在数组中插入值
    add esi,TYPE WORD     ; 指向下一个元素
    loop L1

L2:    popad                ; 恢复寄存器
    pop ebp
    ret 8                   ; 清除堆栈
ArrayFill ENDP
```

## 8.7 LEA指令：返回间接操作数的地址

LEA 指令返回间接操作数的地址。由于间接操作数中包含一个或多个寄存器，因此会在运行时计算这些操作数的偏移量。为了演示如何使用 LEA，现在来看下面的 [C++](http://c.biancheng.net/cplus/) 程序，该程序声明了一个局部数组 myString，并引用它来分配数组值：

```c
void makeArray()
{
    char myString[30];
    for ( int i = 0; i < 30; i++ )
        myString[i] = '*';
}
```

与之等效的汇编代码在堆栈中为 myString 分配空间，并将地址（间接操作数）赋给 ESI。虽然数组只有 30 个字节，但是 ESP 还是递减了 32 以对齐双字边界。注意如何使用 LEA 把数组地址分配给 ESI：

```text
makeArray PROC
    push ebp
    mov ebp,esp
    sub esp, 32            ;myString 位于 EBP-30 的位置
    lea esi, [ebp-30]      ;加载 myString 的地址
    mov ecx, 30            ;循环计数器
LI: mov BYTE PTR [esi]     ;填充一个位置
    inc esi                ;指向下一个元素
    loop LI                ;循环，直到 ECX=0
    add esp, 32            ;删除数组(恢复ESP)
    pop ebp
    ret
makeArray ENDP
```

不能用 OFFSET 获得堆栈参数的地址，因为 OFFSET 只适用于编译时已知的地址。下面的语句无法汇编：

```text
mov esi,OFFSET [ebp-30 ]  ;错误
```

## 8.8 ENTER和LEAVE指令：创建和结束堆栈帧

ENTER 指令为被调用过程自动创建堆栈帧。它为局部变量保留堆栈空间，把 EBP 入栈。具体来说，它执行三个操作：

* 把 EBP 入栈 \(push ebp\)
* 把 EBP 设置为堆栈帧的基址 \(mov ebp, esp\)
* 为局部变量保留空间 \(sub esp, numbytes\)

ENTER 有两个操作数：第一个是常数，定义为局部变量保存的堆栈空间字节数；第二个定义了过程的词法嵌套级。

```text
ENTER numbytes, nestinglevel
```

这两个操作数都是立即数。Numbytes 总是向上舍入为 4 的倍数，以便 ESP 对齐双字边界。Nestinglevel 确定了从主调过程堆栈帧复制到当前帧的堆栈帧指针的个数。在示例程序中，nestinglevel 总是为 0。

【示例 1】下面的例子声明了一个没有局部变量的过程：

```text
MySub PROC
  enter 0,0
```

它与如下指令等效：

```text
MySub PROC
  push ebp
  mov ebp, esp
```

【示例 2】ENTER 指令为局部变量保留了 8 个字节的堆栈空间：

```text
MySub PROC
  enter 8,0
```

它与如下指令等效：

```text
MySub PROC
  push ebp
  mov ebp,esp
  sub esp,8
```

下图为执行 ENTER 指令前后的堆栈示意图。

![&#x6267;&#x884C;ENTER&#x6307;&#x4EE4;&#x540E;&#x7684;&#x5806;&#x6808;](http://c.biancheng.net/uploads/allimg/190514/4-1Z514153102338.gif)

如果要使用 ENTER 指令，那么强烈建议在同一个过程的结尾处同时使用 LEAVE 指令。否则，为局部变量保留的堆栈空间就可能无法释放。这将会导致 RET 指令从堆栈中弹出错误的返回地址。

#### LEAVE 指令

LEAVE 指令结束一个过程的堆栈帧。它反转了之前的 ENTER 指令操作：恢复了过程被调用时 ESP 和 EBP 的值。再次以 MySub 过程为例，现在可以编码如下：

```text
MySub PROC
  enter 8,0
  .
  .
  leave
  ret
MySub ENDP
```

下面是与之等效的指令序列，其功能是在堆栈中保存和删除 8 个字节的局部变量：

```text
MySub PROC
  push ebp
  mov ebp, esp
  sub esp, 8
  .
  .
  mov esp, ebp
  pop ebp
  ret
MySub ENDP
```

## 8.9 LOCAL伪指令：声明一个或多个变量名

不难想象，Microsoft 创建 LOCAL 伪指令是作为 ENTER 指令的高级替补。LOCAL 声明一个或多个变量名，并定义其大小属性。（另一方面，ENTER 则只为局部变量保留一块未命名的堆栈空间。）如果要使用 LOCAL 伪指令，它必须紧跟在 PROC 伪指令的后面。

其语法如下所示：

```text
LOCAL varlist
```

varlist 是变量定义列表，用逗号分隔表项，可选为跨越多行。每个变量定义采用如下格式：

```text
label:type
```

其中，标号可以为任意有效标识符，类型既可以是标准类型（WORD、DWORD 等），也可以是用户定义类型。

【示例】MySub 过程包含一个局部变量 var1，其类型为 BYTE：

```text
MySub PROC
LOCAL var1:BYTE
```

BubbleSort 过程包含一个双字局部变量 temp 和一个类型为 BYTE 的 SwapFlag 变量：

```text
BubbleSort PROC
LOCAL temp:DWORD, SwapFlag:BYTE
```

Merge 过程包含一个类型为 PTR WORD 的局部变量 pArray，它是一个 16 位整数的指针：

```text
Merge PROC
LOCAL pArray:PTR WORD
```

局部变量 TempArray 是一个数组，包含 10 个双字。请注意用方括号显示数组大小：

```text
LOCAL TempArray[10]:DWORD
```

### MASM 代码生成

使用 LOCAL 伪指令时，查看 MASM 生成代码是有好处的。下面的过程 Example1 有一个双字局部变量：

```text
Example1 PROC
    LOCAL temp:DWORD

    mov eax,temp
    ret
Example1 ENDP
```

MASM 为 Example1 生成如下代码，展示了 ESP 怎样减去 4，以便为双字变量预留空间：

```text
push ebp
mov ebp, esp
add esp, OFFFFFFFCh     ;ESP 加 -4
mov eax, [ebp-4]
leave
ret
```

Example1 的堆栈帧示意图如下所示：

![img](http://c.biancheng.net/uploads/allimg/190514/4-1Z514155511C9.gif)

## 8.10 Microsoft x64调用规范简介

Microsoft 遵循固定模式实现 64 位编程中的参数传递和子程序调用，该模式被称为 Microsoft x64 调用规范（Microsoft x64 calling convention）。它既用于 C 和 [C++](http://c.biancheng.net/cplus/) 编译器，也用于 Windows API库。

只有在要么调用 Windows 函数，要么调用 C 和 C++ 函数时，才需要使用这个调用规范。它的特点和要求如下所示：

1\) 由于地址长为 64 位，因此 CALL 指令把 RSP（堆栈指针）寄存器的值减去8。

2\) 第一批传递给子程序的四个参数依次存放于寄存器 RCX、RDX、R8 和 R9。因此，如果只传递一个参数，它就会被放入 RCX。如果还有第二个参数，它就会被放入 RDX，以此类推。其他参数按照从左到右的顺序入栈。

3\) 长度不足 64 位的参数不进行零扩展，因此，其高位的值是不确定的。

4\) 如果返回值的长度小于或等于 64 位，那么它必须放在 RAX 寄存器中。

5\) 主调者要负责在堆栈中分配至少 32 字节的影子空间，以便被调用的子程序可以选择将寄存器保存在这个区域中。

6\) 调用子程序时，堆栈指针（RSP）必须对齐 16 字节边界。CALL 指令将 8 字节的返回地址压入堆栈，因此，主调程序除了把堆栈指针减去 32 以便存放寄存器参数之外，还要减去8。

7\) 被调用子程序执行结束后，主调程序需负责从运行时堆栈中移除所有的参数和影子空间。

8\) 大于 64 位的返回值存放于运行时堆栈，由 RCX 指出其位置。

9\) 寄存器 RAX、RCX、RDX、R8、R9、R10 和 R11 常常被子程序修改，因此，如果主调程序想要保存它们的值，就应在调用子程序之前将它们入栈，之后再从堆栈弹出。

10\) 寄存器 RBX、RBP、RDI、RSI、R12、R13、R14 和 R15 的值必须由子程序保存。

## 8.11 递归及应用详解\[附带实例\]

递归子程序（recursive subrountine）是指直接或间接调用自身的子程序。递归，调用递归子程序的做法，在处理具有重复模式的[数据结构](http://c.biancheng.net/data_structure/)时，它是一个强大的工具。例如链表和各种类型的连接图，这些情况下，程序都需要追踪其路径。

### 无限递归

子程序对自身的调用是递归中最显而易见的类型。例如，下面的程序包含一个名为 Endless 的过程，它不间断地重复调用自身：

```text
;无限递归 (Endless, asm)
INCLUDE Irvine32.inc
.data
endlessStr BYTE "This recursion never stops",0
.code
main PROC
    call Endless
    exit
main ENDP

Endless PROC
    mov edx,OFFSET endlessStr
    call WriteString
    call Endless
    ret                           ;从不执行
Endless ENDP
END main
```

当然，这个例子没有任何实用价值。每次过程调用自身时，它会占用 4 字节的堆栈空间让 CALL 指令将返回地址入栈。RET 指令永远不会被执行，仅当堆栈溢出时，程序终止。

### 递归求和

实用的递归子程序总是包含终止条件。当终止条件为真时，随着程序执行所有挂起的 RET 指令，堆栈展开。举例说明，考虑一个名为 CalcSum 的递归过程，执行整数 1 到 n 的加法，其中 n 是通过 ECX 传递的输入参数。CalcSum 用 EAX 返回和数：

```text
;整数求和   (RecursiveSum. asm)
INCLUDE Irvine32.inc
.code
main PROC
    mov  ecx,5          ; 计数值 = 5
    mov  eax,0          ; 保存和数
    call CalcSum        ; 计算和数
L1:    call WriteDec    ; 显示 EAX
    call Crlf           ; 换行
    exit
main ENDP

;--------------------------------------------------------
CalcSum PROC
; 计算整数列表的和数
; 接收: ECX = 计数值
; 返回: EAX = 和数
;--------------------------------------------------------
    cmp  ecx,0         ; 检查计数值
    jz   L2            ; 若为零则推出
    add  eax,ecx       ; 否则，与和数相加
    dec  ecx           ; 计数值递减
    call CalcSum       ; 递归调用
L2:    ret
CalcSum ENDP
END Main
```

CalcSum 的开始两行检查计数值，若 ECX=0 则退出该过程，代码就跳过了后续的递归调用。当第一次执行 RET 指令时，它返回到前一次对 CalcSum 的调用，而这个调用再返回到它的前一次调用，依序前推。

下表给出了 CALL 指令压入堆栈的返回地址（用标号表示），以及与之相应的 ECX（计数值）和 EAX（和数）的值。

| 入栈的返回地址 | ECX的值 | EAX的值 | 入栈的返回地址 | ECX的值 | EAX的值 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| L1 | 5 | 0 | L2 | 2 | 12 |
| L2 | 4 | 5 | L2 | 1 | 14 |
| L2 | 3 | 9 | L2 | 0 | 15 |

即使是一个简单的递归过程也会使用大量的堆栈空间。每次过程调用发生时最少占用 4 字节的堆栈空间，因为要把返回地址保存到堆栈。

### 计算阶乘

递归子程序经常用堆栈参数来保存临时数据。当递归调用展开时，保存在堆栈中的数据就有用了。下面要查看的例子是计算整数 n 的阶乘。阶乘算法计算 n!，其中 n 是无符号整数。第一次调用 factorial 函数时，参数 n 就是初始数字。下面给出的是用 C/[C++](http://c.biancheng.net/cplus/)/[Java](http://c.biancheng.net/java/) 语法编写的代码：

```c
int function factorial(int n)
{
    if(n == 0)
        return 1;
    else
        return n * factorial(n-1);
}
```

假设给定任意 n，即可计算 n-1 的阶乘。这样就可以不断减少 n，直到它等于 0 为止。根据定义，0!=l。而回溯到原始表达式 n! 的过程，就会累积每次的乘积。比如计算 5! 的递归算法如下图所示，左列为算法递进过程，右列为算法回溯过程。

![&#x9636;&#x4E58;&#x51FD;&#x6570;&#x7684;&#x9012;&#x5F52;&#x8C03;&#x7528;](http://c.biancheng.net/uploads/allimg/190514/4-1Z5141KZ5929.gif)

【示例】下面的[汇编语言](http://c.biancheng.net/asm/)程序包含了过程 Factorial，递归计算阶乘。通过堆栈把 n\(0〜12 之间的无符号整数 \) 传递给过程 Factorial，返回值在 EAX 中。由于 EAX 是 32 位寄存器，因此，它能容纳的最大阶乘为 12!\(479 001 600 \)。

```text
; 计算阶乘 (Fact.asm)
INCLUDE Irvine32.inc
.code
main PROC
    push 5                ; 计算 5!
    call Factorial        ; 计算阶乘 (eax)
    call WriteDec         ; 显示结果
    call Crlf
    exit
main ENDP

Factorial PROC
    push ebp
    mov  ebp,esp
    mov  eax,[ebp+8]       ; 获取 n
    cmp  eax,0             ; n < 0?
    ja   L1                ; 是: 继续
    mov  eax,1             ; 否: 返回0!的值 1
    jmp  L2                ; 并返回主调程序

L1:    dec  eax
    push eax               ; Factorial(n-1)
    call Factorial

; 每次递归调用返回时
; 都要执行下面的指令

ReturnFact:
    mov  ebx,[ebp+8]       ; 获取 n
    mul  ebx               ; EDX:EAX = EAX*EBX

L2:    pop  ebp            ; 返回 EAX
    ret  4                 ; 清除堆栈
Factorial ENDP
END main
```

现在通过跟踪初始值 N=3 的调用过程，来更加详细地查看 Factorial。按照其说明中的记录，Factorial 用 EAX 寄存器返回结果：

```text
push 3
call Factorial      ; EAX = 3!
```

Factorial 过程接收一个堆栈参数 N 为初始值，以决定计算哪个数的阶乘。主调程序的返回地址由 CALL 指令自动入栈。Factorial 的第一个操作是把 EBP 入栈，以便保存主调程序堆栈的基址指针：

```text
Factorial PROC
  push ebp
```

之后，它必须把 EBP 设置为当前堆栈帧的起始地址：

```text
mov ebp resp
```

现在，EBP 和 ESP 都指向栈顶，运行时堆栈的堆栈帧如下图所示。其中包含了参数 N、主调程序的返回地址和被保存的 EBP 值：

![img](http://c.biancheng.net/uploads/allimg/190514/4-1Z5141P025562.gif)

由上图可知，要从堆栈中取出 N 并加载到 EAX，代码需要把 EBP 加 8 后进行基址-偏移量寻址：

```text
mov eax, [ebp+8]   ; 获取 n
```

然后，代码检查基本情况 \(base case\)，即停止递归的条件。如果 N \(EAX 当前值 \) 等于 0，函数返回 1，也就是 0! 的定义值。

```text
cmp  eax,0    ; n>0  ?
ja  L1         ; 是：继续
mov  eax, 1    ; 否：返回o  !的结果1
jmp  L2       ; 并返回主调程序
```

由于当前 EAX 的值为 3，Factorial 将递归调用自身。首先，它从 N 中减去 1，并把新值入栈。该值作为参数传递给新调用的 Factorial:

```text
L1: dec eax
  push eax     ; Factorial(n - 1)
  call Factorial
```

现在，执行转向 Factorial 的第一行，计算数值为 N 的新值：

```text
Factorial PROC
  push ebp
  mov ebp,esp
```

运行时堆栈又包含了第二个堆栈帧，其中 N 等于 2：

![img](http://c.biancheng.net/uploads/allimg/190514/4-1Z5141P12E91.gif)

现在 N 的值为 2，将其加载到 EAX，并与 0 比较：

```text
mov eax, [ebp+8]   ;当前 N=2
cmp  eax, 0       ; N 与 0 比较
ja  L1            ;仍然大于0
mov  eax, 1       ;不执行
jmp  L2          ;不执行
```

N 大于 0，因此，继续执行标号 L1。

大家可能已经注意到之前的 EAX，即第一次调用时分配给 Factorial 的值，被新值覆盖了。这说明了一个重要的事实：在过程进行递归调用时，应该小心注意哪些寄存器会被修改。如果需要保存这些寄存器的值，就需要在递归调用之前将其入栈，并在调用返回之后将其弹出堆栈。幸运的是，对 Factorial 过程而言，在递归调用之间保存 EAX 并不是必要的。

执行 L1 时，将会用递归过程调用来计算 N-1 的阶乘。代码将 EAX 减 1，并将结果入栈，再调用 Factorial：

```text
L1: dec  eax      ; N = 1
  push eax       ; Factorial(1)
  call Factorial
```

现在，第三次进入 Factorial，堆栈中也有了三个活动的堆栈帧：

![img](http://c.biancheng.net/uploads/allimg/190514/4-1Z5141P230H7.gif)

Factorial 程将 N 与 0 比较，发现 N 大于 0，则再次调用 Factorial，此时 N=0。当最后一次进入 Factorial 过程时，运行时堆栈出现了第四个堆栈帧：

![img](http://c.biancheng.net/uploads/allimg/190514/4-1Z5141P246294.gif)

在 N=0 时调用 Factorial，情况变得有趣了。下面的语句产生了到标号 L2 的分支。由于 0! =1，因此数值 1 赋给 EAX，而 EAX 必须包含 Factorial 的返回值：

```text
mov eax,[ebp+8]     ; EAX = 0
cmp eax,0          ; n < 0?
ja L1               ; 是: 继续
mov eax,1          ; 否: 返回0!的值 1
jmp L2             ; 并返回主调程序
```

标号 L2 的语句如下，它使得 Factorial 返回到前一次的调用：

```text
L2: pop ebp      ; 返回 EAX
  ret 4          ; 清除堆栈
```

此时，如下图所示，最近的帧已经不在运行时堆栈中，且 EAX 的值为 1（零的阶乘）。

![img](http://c.biancheng.net/uploads/allimg/190514/4-1Z5141P3291S.gif)

下面的代码行是 Factorial 调用的返回点。它们获取 N 的当前值（保存于堆栈 EBP+8 的位置），将其与 EAX（Factorial 调用的返回值）相乘。那么，EAX 中的乘积就是 Factorial 本次迭代的返回值：

```text
ReturnFact:
  mov ebx,[ebp+8]    ; 获取 n
  mul ebx           ; EDX:EAX = EAX*EBX

L2:  pop ebp        ; 返回 EAX
  ret 4              ; 清除堆栈
Factorial ENDP
```

（EDX 中的乘积高位部分为全 0，可以忽略。）由此，上述代码行第一次得以执行，EAX 保存了表达式 1 x 1 的乘积。随着 RET 指令的执行，又一个堆栈帧从堆栈中删除：

![img](http://c.biancheng.net/uploads/allimg/190514/4-1Z5141P406204.gif)

再次执行 CALL 指令后面的语句，将 N（现在等于 2）与 EAX 的值（等于 1）相乘：

```text
ReturnFact:
  mov ebx,[ebp+8]    ; 获取 n
  mul ebx           ; EDX:EAX = EAX*EBX

L2:  pop ebp        ; 返回 EAX
  ret 4             ; 清除堆栈
Factorial ENDP
```

EAX 中的乘积现在等于 2，RET 指令又从堆栈中移除一个堆栈帧：

![img](http://c.biancheng.net/uploads/allimg/190514/4-1Z5141P444K9.gif)

现在，最后一次执行 CALL 指令后面的语句，将 N（等于 3）与 EAX 的值（等于 2）相乘：

```text
ReturnFact:
  mov ebx,[ebp+8]    ; 获取 n
  mul ebx           ; EDX:EAX = EAX*EBX

L2:  pop ebp        ; 返回 EAX
  ret 4              ; 清除堆栈
Factorial ENDP
```

EAX 的返回值为 6，是 3! 的计算结果，也是第一次调用 Factorial 时希望进行的计算。当 RET 指令执行时，最后一个堆栈帧也从堆栈中移除。

## 8.12 INVOKE伪指令：将参数入栈并调用过程

INVOKE 伪指令，只用于 32 位模式，将参数入栈（按照 MODEL 伪指令的语言说明符所指定的顺序）并调用过程。INVOKE 是 CALL 指令一个方便的替代品，因为，它用一行代码就能传递多个参数。常见语法如下：

```text
INVOKE procedureName [, argumentList]
```

ArgumentList 是可选项，它用逗号分隔传递给过程的参数。例如，执行若干 PUSH 指令后调用 DumpArray 过程，使用 CALL 指令的形式如下：

```text
push TYPE array
push LENGTHOF array
push OFFSET array
call DumpArray
```

使用等效的 INVOKE 则将代码减少为一行，列表中的参数逆序排列（假设遵循 STDCALL 规范）：

```text
INVOKE DumpArray, OFFSET array, LENGTHOF array, TYPE array
```

INVOKE 对参数数量几乎没有限制，每个参数也可以独立成行。下面的 INVOKE 语句包含了有用的注释：

```text
INVOKE DumpArray,     ;显示数组
OFFSET array,          ;指向数组
LENGTHOF array,       ;数组长度
TYPE array             ;数组元素的大小类型
```

参数类型如下表所示。

| 类型 | 例子 | 类型 | 例子 |
| :--- | :--- | :--- | :--- |
| 立即数 | 10, 3000h, Offset mylist, TYPE array | 寄存器 | eax, bl, edi |
| 整数表达式 | \(10\*20\), COUNT | ADDR name | ADDR myList |
| 变量 | myLIst, array, my Word, myDword | OFFSET name | OFFSET myList |
| 地址表达式 | \[myList+2\], \[ebx+esi\] |  |  |

### 覆盖 EAX 和 EDX

如果向过程传递的参数小于 32 位，那么在将参数入栈之前，INVOKE 为了扩展参数常常会使得汇编器覆盖 EAX 和 EDX 的内容。有两种方法可以避免这种情况：

* 其一，传递给 INVOKE 的参数总是 32 位的；
* 其二，在过程调用之前保存 EAX 和 EDX，在过程调用之后再恢复它们的值。

## 8.13 ADDR运算符：传递指针参数

ADDR 运算符同样可用于 32 位模式，在使用 INVOKE 调用过程时，它可以传递指针参数。比如，下面的 INVOKE 语句给 FillArray 过程传递了 myArray 的地址：

```text
INVOKE FillArray, ADDR myArray
```

传递给 ADDR 的参数必须是汇编时常数。下面的例子就是错误的：

```text
INVOKE mySub, ADDR [ebp+12 ]     ;错误
```

ADDR 运算符只能与 INVOKE 一起使用，下面的例子也是错误的：

```text
mov esi, ADDR myArray             ;错误
```

【示例】下例中的 INVOKE 伪指令调用 Swap，并向其传递了一个双字数组前两个元素的地址：

```text
.data
Array DWORD 20 DUP(?)
.code
...
INVOKE Swap,
  ADDR Array,
  ADDR [Array+4]
```

假设使用 STDCALL 规范，那么汇编器生成的相应代码如下所示：

```text
push OFFSET Array+4
push OFFSET Array
call Swap
```

## 8.14 PROC伪指令：过程定义

32 位模式中，PROC 伪指令基本语法如下所示：

```text
label PROC [attributes] [USES reglist], parameter_list
```

Label 是按照《[LABEL伪指令](http://c.biancheng.net/view/3519.html)》一节中说明的标识符规则、由用户定义的标号。Attributes 是指下述任一内容：

```text
[distance] [langtype] [visibility] [prologuearg]
```

下表对这些属性进行了说明。

| 属性 | 说明 |
| :--- | :--- |
| distance | NEAR 或 FAR。指定汇编器生成的 RET 指令（RET 或 RETF）类型 |
| langtype | 指定调用规范（参数传递规范），如 C、PASCAL 或 STDCALL。能覆盖由 .MODEL 伪指令指定的语言 |
| visibility | 指明本过程对其他模块的可见性。选项包括 PRIVATE、PUBLIC （默认项）和 EXPORT。若可见性为 EXPORT，则链接器把过程名放入分段可执行文件的导出表。EXPORT 也使之具有了 PUBLIC 可见性 |
| prologuearg | 指定会影响开始和结尾代码生成的参数 |

### 参数列表

PROC 伪指令允许在声明过程时，添加上用逗号分隔的参数名列表。代码实现可以用名称来引用参数，而不是计算堆栈偏移量，如 \[ebp+8\]:

```text
label PROC [attributes] [USES reglist],
  parameter_1,
  parameter_2,
  ...
  parameter_n
```

如果参数列表与 PROC 在同一行，则 PROC 后面的逗号可以省略：

```text
label PROC [attributes], parameter_1, parameter_2, ..., parameter_n
```

每个参数的语法如下：

```text
paramName: type
```

ParamName 是分配给参数的任意名称，其范围只限于当前过程（称为局部作用域（local scope））。同样的参数名可以用于多个过程，但却不能作为全局变量或代码标号的名称。

Type 可以在这些类型中选择：BYTE、SBYTE、WORD、SWORD、DWORD、SDWORD、FWORD、QWORD 或 TBYTE。此外，type 还可以是限定类型（qualified type），如指向现有类型的指针。

下面是限定类型的例子：

```text
PTR BYTE     PTR SBYTE
PTR WORD   PTR SWORD
PTR DWORD  PTR SDWORD
PTR QWORD  PTR TBYTE
```

虽然可以在这些表达式中添加 NEAR 和 FAR 属性，但它们只与更加专用的应用程序相关。限定类型还能够用 TYPEDEF 和 STRUCT 伪指令创建。

【示例 1】AddTwo 过程接收两个双字数值，用 EAX 返回它们的和数：

```text
AddTwo PROC,
    val1:DWORD,
    val2:DWORD
    mov eax,val1
    add eax,val2
    ret
AddTwo ENDP
```

AddTwo 汇编时，MASM 生成的汇编代码显示了参数名是如何被转换为 EBP 偏移量的。由于使用的是 STDCALL，因此 RET 指令附加了一个常量操作数：

```text
AddTwo PROC
    push ebp
    mov ebp, esp
    mov eax, dword ptr [ebp+8]
    add eax, dword ptr [ebp+OCh]
    leave
    ret 8
AddTwo ENDP
```

用指令 ENTERO, 0 来代替下面的语句，AddTwo 过程也一样正确：

```text
push ebp
mov ebp,esp
```

【示例 2】FillArray 过程接收一个字节数组的指针：

```text
FillArray PROC,
  pArray:PTR BYTE
  ...
FillArray ENDP
```

【示例 3】Swap 过程接收两个双字指针：

```text
Swap PROC,
    pValX:PTR DWORD,
    pValY:PTR DWORD
Swap ENDP
```

【示例 4】Read\_File 过程接收一个字节指针 pBuffer，有一个局部双字变量 fileHandle，并把两个寄存器保存入栈（EAX 和 EBX）：

```text
Read_File PROC USES eax ebx,
  pBuffer:PTR BYTE
  LOCAL fileHandle:DWORD

  mov esi,pBuffer
  mov fileHandle,eax
  ...
  ret
Read_File ENDP
```

MASM 为 Read\_File 生成的代码显示了在 EAX 和 EBX 入栈（由 USES 子句指定）前，如何为局部变量（fileHandle）预留堆栈空间：

```text
Read_File PROC
    push ebp
    mov ebp,esp
    add esp, OFFFFFFFCh          ;创建 fileHandle
    push eax                     ;保存 EAX
    push ebx                     ;保存 EBX
    mov esi, dword ptr [ebp+8]   ; pBuffer
    mov dword ptr [ebp-4],eax    ; fileHandle
    pop ebx
    pop eax
    leave
    ret 4
Read_File ENDP
```

注意：尽管 Microsoft 没有采用这种方法，但 Read\_File 生成代码的开始部分还可以是这样的：

```text
Read_File PROC
  enter 4,0
  push eax
  (etc.)
```

ENTER 指令首先保存 EBP，再将它设置为堆栈指针的值，并为局部变量保留空间。

### 由 PROC 修改的 RET 指令

当 PROC 有一个或多个参数时，STDCALL 是默认调用规范。假设 PROC 有 n 个参数，MASM 将生成如下入口和出口代码：

```text
push ebp
mov ebp,esp
...
leave
ret (n*4)
```

RET 指令中的常数是参数个数乘以 4 \( 因为每个参数都是一个双字 \)。若使用了 INCLUDE Irvine32.inc，则 STDCALL 是默认规范，它是所有 Windows API 函数调用使用的调用规范。

### 指定参数传递协议

一个程序可以调用 Irvme32 链接库过程，反之，也可以包含能被 [C++](http://c.biancheng.net/cplus/) 程序调用的过程。为了提供这样的灵活性，PROC 伪指令的属性域允许程序指定传递参数的语言规范，并且能覆盖 .MODEL 伪指令指定的默认语言规范。

下例声明的过程采用了 C 调用规范：

```text
Examplel PROC C,
  parm1:DWORD, parm2:DWORD
```

若用 INVOKE 执行 Examplel，汇编器将生成符合 C 调用规范的代码。同样，如果用 STDCALL 声明 Examplel，INVOKE 的生成代码也会符合这个语言规范：

```text
Examplel PROC STDCALL,
  parm1:DWORD, parm2:DWORD
```

## 8.15 PROTO伪指令：指定程序的外部过程

64 模式中，PROTO 伪指令指定程序的外部过程，示例如下：

```text
ExitProcess PROTO
.code
mov ecx, 0
call ExitProcess
```

然而在 32 位模式中，PROTO 是一个更有用的工具，因为它可以包含过程参数列表。可以说，PROTO 伪指令为现有过程创建了原型 \(prototype\)。原型声明了过程的名称和参数列表。它还允许在定义过程之前对其进行调用，并验证参数的数量和类型是否与过程的定义相匹配。

MASM 要求 INVOKE 调用的每个过程都有原型。PROTO 必须在 INVOKE 之前首先岀现。换句话说，这些伪指令的标准顺序为：

```text
MySub PROTO    ;过程原型
.
INVOKE MySub    ;过程调用
.
MySub PROC     ;过程实现
..
MySub ENDP
```

还有一种情况也是可能的：过程实现可以出现在程序的前面，先于调用该过程的 INVOKE 语句。在这种情况下，PROC 就是它自己的原型：

```text
MySub PROC      ;过程定义
..
MySub ENDP
.
INVOKE MySub    ;过程调用
```

假设已经编写了一个特定的过程，创建其原型也是很容易的，即复制 PROC 语句并做如下修改：

* 将关键字 PROC 改为 PROTO。
* 如有 USES 运算符，则把该运算符连同其寄存器列表一起删除。

比如，假设已经创建了 ArraySum 过程：

```text
ArraySum PROC USES esi ecx,
    ptrArray:PTR DWORD,        ;指向数组
    szArray:DWORD              ;数组大小
;省略其余代码行……
ArraySum ENDP
```

下面是与之对应的 PROTO 声明：

```text
ArraySum PROTO,
  ptrArray:PTR DWORD,     ;指向数组
  szArray:DWORD          ;数组大小
```

PROTO 伪指令可以覆盖 .MODEL 伪指令中的参数传递协议。但它必须与过程的 PROC 声明一致：

```text
Example1 PROTO C,
  parm1:DWORD, parm2:DWORD
```

### 汇编时参数检查

PROTO 伪指令帮助汇编器比较过程调用和过程定义的参数列表。但是这个错误检查没有如 C 和 [C++](http://c.biancheng.net/cplus/) 语言中那样重要。相反，MASM 检查参数正确的数量，并在某些情况下，匹配实际参数和形式参数的类型。比如，假设 Sub1 的原型声明如下：

```text
Sub1 PROTO, p1:BYTE, p2:WORD, p3:PTR BYTE
```

现在定义变量：

```text
.data
byte_1 BYTE 10h
.word_1 WORD 2000h
word_2 WORD 3000h
dword_1 DWORD 12345678h
```

那么，下面是 Sub1 的一个有效调用：

```text
INVOKE Sub1, byte_1, word_1, ADDR byte_1
```

MASM 为这个 INVOKE 生成的代码显示了参数按逆序压入堆栈：

```text
push 404000h                  ;指向 byte_1 的指针
sub  esp, 2                    ;在栈项填充两个字节
push word ptr ds:[00404001h]     ;word_1 的值
mov  al, byte ptr ds:[00404000h]  ;byte_1 的值
push eax
call 00401071
```

EAX 被覆盖，sub esp,2 指令填充接下来的两个堆栈单元，以扩展到 32 位。

#### MASM 会检测的错误

如果实际参数超过了形式参数声明的大小，MASM 就会产生一个错误：

```text
INVOKE Sub1, word_1, word_2, ADDR byte_1   ;参数 1 错误
```

如果调用 Sub1 时参数个数太少或太多，则 MASM 会产生错误：

```text
INVOKE Sub1, byte_1, word_2               ;错误：参数个数太少
INVOKE Sub1, byte_1,                      ;错误：参数个数太多
  word_2, ADDR byte_1, word_2
```

#### MASM 不会检测的错误

如果实际参数的类型小于形式参数的声明，那么 MASM 不会检测出错误：

```text
INVOKE Sub1, byte_1, byte_1, ADDR byte_1
```

相反，MASM 会把实际参数扩展为形式参数声明的类型大小。下面是 INVOKE 示例生成的代码，其中第二个实际参数 \(byte\_1\) 入栈之前，在 EAX 中进行了扩展：

```text
push 404000h                  ;byte_1 的地址
mov al,byte ptr ds:[00404000h]     ;byte_1
movzx eax,al                   ;在 EAX 中扩展
push eax                      ;入栈
mov al,byte ptr ds:[00404000h]    ;byte_1 的值
push eax                      ;入栈
call 00401071                  ;调用 Sub1
```

如果在想要传递指针时传递了一个双字，则不会检测出任何错误。当子程序试图把这个堆栈参数用作指针时，这种情况通常会导致一个运行时错误：

```text
INVOKE Sub1, byte_1, word_2, dword_1 ;无错误检出
```

### ArraySum 示例

过程用寄存器传递参数，现在，可以用 PROC 伪指令来声明堆栈参数：

```text
ArraySum PROC USES esi ecx,
    ptrArray:PTR DWORD,                  ;指向数组
    szArray:DWORD                        ;数组大小
    mov esi, ptrArray                    ;数组地址
    mov ecx, szArray                     ;数组大小
    mov eax, 0                           ;和数清零
    cmp ecx, 0                           ;数组长度=0?
    je L2                                ;是：退出
L1: add eax, [esi]                       ;将每个整数加到和数中
    add esi, 4                           ;指向下一个整数
    loop L1                              ;按数组大小重复
L2: ret                                  ;和数保存在EAX中
ArraySum ENDP
```

INVOKE 语句调用 ArraySum，传递数组地址和元素个数：

```text
.data
array DWORD 10000h, 20000h, 30000h, 40000h, 50000h
theSum DWORD ?
.code
main PROC
    INVOKE ArraySum,
        AD DR array,                     ;数组地址
        LENGTHOF array                   ;元素个数
    mov theSum, eax                      ;保存和数
```

## 8.16 过程参数简介

过程参数一般按照数据在主调程序和被调用过程之间传递的方向来分类：

#### 1\) 输入类

输入参数是指从主调程序传递给过程的数据。被调用过程不会被要求修改相应的参数变量，即使修改了，其范围也只能局限在自身过程中。

#### 2\) 输出类

当主调程序向过程传递变量地址，就会产生输岀参数。过程用地址来定位变量，并为其分配数据。比如，Win32 控制台库中的 ReadConsole 函数，其功能为从键盘读入一个字符串。用户键入的字符由 ReadConsole 保存到缓冲区中，而主调程序传递的就是这个字符串缓冲区的指针：

```text
.data
buffer BYTE 80 DUP(?)
inputHandle DWORD ?
.code
INVOKE ReadConsole, inputHandle, ADDR buffer,
  (etc.)
```

#### 3\) 输入输出类

输入输出参数与输出参数相同，只有一个例外：被调用过程预期参数引用的变量中会包含一些数据，并且能通过指针来修改这些变量。

【示例】下面的例子实现两个 32 位整数的交换。Swap 过程有两个输入输出参数 pValX 和 pValY，它们是交换数据的地址：

```text
; Swap 过程示例   (Swap.asm)
INCLUDE Irvine32.inc

Swap PROTO, pValX:PTR DWORD, pValY:PTR DWORD

.data
Array DWORD 10000h,20000h

.code
main PROC
    ; 显示交换前的数组
    mov  esi,OFFSET Array
    mov  ecx,2                     ; 计数值 = 2
    mov  ebx,TYPE Array
    call DumpMem                   ; 显示数组

    INVOKE Swap,ADDR Array, ADDR [Array+4]
    ; 显示交换后的数组
    call DumpMem
    exit
main ENDP

;-------------------------------------------------------
Swap PROC USES eax esi edi,
    pValX:PTR DWORD,              ; 第一个整数的指针
    pValY:PTR DWORD               ; 第二个整数的指针
; 交换两个32位整数的值
; 返回: 无
;-------------------------------------------------------
    mov esi,pValX                 ; 获得指针
    mov edi,pValY
    mov eax,[esi]                 ; 取第一个整数
    xchg eax,[edi]                ; 与第二个数交换
    mov [esi],eax                 ; PROC 在这里生成 RET 8
    ret
Swap ENDP
END main
```

Swap 程的两个参数 pValX 和 pValY 都是输入输出参数。它们的当前值要输入到过程，而它们的新值也会从过程输出。由于使用的 PROC 带有参数，汇编器把 Swap 过程末尾的 RET 指令改为 RET 8（假设调用规范是 STDCALL）。

### 调试提示

这里提醒编程者要注意的一些常见错误是[汇编语言](http://c.biancheng.net/asm/)在传递过程参数时会遇到的，希望编程者永远不要犯这些错误。

#### 1\) 参数大小不匹配

数组地址以其元素的大小为基础。比如，一个双字数组第二个元素的地址就是其起始地址加 4。假设调用 Swap 过程，并传递 DoubleArray 前两个元素的指针。如果错误地把第二个元素的地址计算为 DoubleArray+1，那么调用 Swap 后，DoubleArray 中的十六进制结果值也不正确：

```text
.data
DoubleArray DWORD 10000h,20000h
.code
INVOKE Swap, ADDR [DoubleArray+0], ADDR [DoubleArray+1]
```

#### 2\) 传递错误类型的指针

在使用 INVOKE 时，要记住汇编器不会验证传递给过程的指针类型。例如，Swap 过程期望接收到两个双字指针，假若不小心传递的是指向字节的指针：

```text
.data
ByteArray BYTE 10h,20h,30h,40h,50h,60h,70h,80h
.code
INVOKE Swap, ADDR [ByteArray+0], ADDR [ByteArray+1]
```

程序可以汇编运行，但是当 ESI 和 EDI 解引用时，就会交换两个 32 位数值。

#### 3\) 传递立即数

如果过程有一个引用参数，就不要向其传递立即数参数。考虑下面的过程，它只有一个引用参数：

```text
Sub2 PROC, dataPtr:PTR WORD
    mov esi, dataPtr         ;获得地址
    mov WORD PTR [esi], 0    ;解引用，分配零
    ret
Sub2 ENDP
```

汇编下面的 INVOKE 语句将导致一个运行时错误。Sub2 过程接收 1000h 作为指针的值，并解引用到内存地址 1000h：

```text
INVOKE Sub2, 1000h
```

上例很可能会导致一般性保护故障，因为内存地址1000h不大可能在该程序的数据段中。

## 8.17 WriteStackFrame过程：显示当前过程堆栈帧的内容

Irvine32 链接库有个很有用的过程 WriteStackFrame，用于显示当前过程堆栈帧的内容，其中包括过程的堆栈参数、返回地址、局部变量和被保存的寄存器。

该过程由太平洋路德大学 \(Pacific Lutheran University\) 的詹姆斯·布林克 \(James Brink\) 教授慷慨提供，原型如下：

```text
WriteStackFrame PROTO,
  numParam:DWORD,        ;传递参数的数量
  numLocalVal: DWORD,      ;双字局部变量的数量
  numSavedReg: DWORD     ;被保存寄存器的数量
```

下面的代码节选自WriteStackFrame的演示程序：

```text
main PROC
    mov eax, 0EAEAEAEAh
    mov ebx, 0EBEBEBEBh
    INVOKE myProc, 1111h, 2222h ;传递两个整数参数
    exit
main ENDP

myProc PROC USES eax ebx,
    x: DWORD, y: DWORD
    LOCAL a:DWORD, b:DWORD

    PARAMS = 2
    LOCALS = 2
    SAVED_REGS = 2
    mov a, 0AAAAh
    mov b, 0BBBBh
    INVOKE WriteStackFrame, PARAMS, LOCALS, SAVED_REGS
```

该调用生成的输岀如下所示：

![img](http://c.biancheng.net/uploads/allimg/190515/4-1Z5151J03S33.gif)

还有一个过程名为 WriteStackFrameName，增加了一个参数，保存拥有该堆栈帧的过程名：

```text
WriteStackFrameName PROTO,
  numParam:DWORD,         ;传递参数的数量
  numLocalVal: DWORD,       ;双字局部变量的数量
  numSavedReg: DWORD,     ;被保存寄存器的数量
  procName:PTR BYTE        ;空字节结束的字符串
```

Irvine32 链接库的源代码保存在安装目录的 \Examples\Lib32 子目录下，文件名为 Irvine32.asm。Irvine32 链接库安装文件可以从我的网盘（[https://pan.baidu.com/s/1yQBqLbViAgs8mCImheCe6g](https://pan.baidu.com/s/1yQBqLbViAgs8mCImheCe6g) 提取码:6kvk）获取。

## 8.18 多模块程序简述

大型源文件难于管理且汇编速度慢，可以把单个文件拆分为多个子文件，但是，对其中任何子文件的修改仍需对所有的文件进行整体汇编。更好的方法是把一个程序按模块（module）（汇编单位）分割。每个模块可以单独汇编，因此，对一个模块源代码的修改就只需要重汇编这个模块。

链接器将所有汇编好的模块（OEJ 文件）组合为一个可执行文件的速度是相当快的，链接大量目标模块比汇编同样数量的源代码文件花费的时间要少得多。

新建多模块程序有两种常用方法：

* 其一是传统方法，使用 EXTERN 伪指令，基本上它在不同的 x86 汇编器之间都可以进行移植。
* 其二是使用 Microsoft 的高级伪指令 INVOKE 和 PROTO，这能够简化过程调用，并隐藏一些底层细节。

### 隐藏和导出过程名

默认情况下，MASM 使所有的过程都是 public 属性，即允许它们能被同一程序中任何其他模块调用。使用限定词 PRIVATE 可以覆盖这个属性：

```text
mySub PROC PRIVATE
```

使过程为 private 属性，可以利用封装原则将过程隐藏在模块中，如果其他模块有相同过程名，就还需避免潜在的重名冲突。

### OPTION PROC:PRIVATE 伪指令

在源模块中隐藏过程的另一个方法是，把 OPTION PROC:PRIVATE 伪指令放在文件顶部。则所有的过程都默认为 private，然后用 PUBLIC 伪指令指明那些希望其可见的过程：

```text
OPTION PROC:PRIVATE
PUBLIC mySub
```

PUBLIC 伪指令用逗号分隔过程名：

```text
PUBLIC sub1, sub2, sub3
```

或者，也可以单独指定过程为 public 属性：

```text
mySub PROC PUBLIC
.
mySub ENDP
```

如果程序的启动模块使用了 OPTION PROC:PRIVATE，那么就应该将它（通常为 main）指定为 PUBLIC，否则操作系统加载器无法发现该启动模块。比如：

```text
main PROC PUBLIC
```

## 8.19 EXTERN伪指令：调用外部过程

用当前模块之外的过程时使用EXTERN伪指令，它确定过程名和堆栈帧大小。下面的示例程序调用了 sub1，它在一个外部模块中：

```text
INCLUDE Irvine32.inc
EXTERN sub1@0:PROC
.code
main PROC
    call subl@0
    exit
main ENDP
END main
```

当汇编器在源文件中发现一个缺失的过程时（由 CALL 指令指定），默认情况下它会产生错误消息。但是，EXTERN 伪指令告诉汇编器为该过程新建一个空地址。在链接器生成程序的可执行文件时再来确定这个空地址。

过程名的后缀 @n 确定了已声明参数占用的堆栈空间总量。如果使用的是基本 PROC 伪指令，没有声明参数，那么 EXTERN 中的每个过程名后缀都为 @0。若用扩展 PROC 伪指令声明一个过程，则每个参数占用 4 字节。假设现在声明的 AddTwo 带有两个双字参数：

```text
AddTwo PROC,
  val1:DWORD,
  val2:DWORD
  ...
AddTwo ENDP
```

则相应的 EXTERN 伪指令为 EXTERN AddTwo@8 : PROC。或者，也可以用 PROTO 伪指令来代替 EXTERN：

```text
AddTwo PROTO,
  val1:DWORD,
  val2:DWORD
```

## 8.20 跨模块使用变量和标号

默认情况下，变量和符号对其包含模块是私有的（private）。可以使用 PUBLIC 伪指令输出指定过程名，如下所示：

```text
PUBLIC count, SYM1
SYM1 = 10
.data
count DWORD 0
```

### 访问外部变量和符号

使用 EXTERN 伪指令可以访问在外部过程中定义的变量和符号：

```text
EXTERN name:type
```

对符号（由 EQU 和 = 定义）而言，type 应为 ABS。对变量而言，type 是数据定义属性，如 BYTE、WORD、DWORD 和 SDWORD，可以包含 PTR。例子如下：

```text
EXTERN one:WORD, two:SDWORD, three:PTR BYTE, four:ABS
```

### 使用带 EXTERNDEF 的 INCLUDE 文件

MASM 中一个很有用的伪指令 EXTERNDEF 可以代替 PUBLIC 和 EXTERN。它可以放在文本文件中，并用 INCLUDE 伪指令复制到每个程序模块。比如，现在用如下声明定义文件 vars.inc：

```text
;var s.inc
EXTERNDEF count:DWORD, SYM1:ABS
```

接着，新建名为 sub1.asm 的源文件，其中包含了 count 和 SYM1，以及一条用于把 vars.inc 复制到编译流中的 INCLUDE 语句。

```text
;sub1.asm
.386
.model flat,STDCALL
INCLUDE vars.inc
SYM1 = 10
.data
count DWORD 0
END
```

因为不是程序启动模块，因此 END 伪指令省略了程序入口标号，并且不用声明运行时堆栈。

现在再新建一个启动模块 main.asm，其中包含 vars.inc，并使用了 count 和 SYM1：

```text
; main.asm
.386
.model flat,stdcall
.stack 4096
ExitProcess proto, dwExitCode:dword
INCLUDE vars.inc
.code
main PROC
    mov count,2000h
    mov eax,SYM1
    INVOKE ExitProcess,0
main ENDP
END main
```

## 8.21 用Extern伪指令新建模块\[附带实例\]

本节主要讲解使用传统的 EXTERN 伪指令引用位于不同模块中的函数。

#### PromptForIntegers

jprompt.asm 是 PromptForIntegers 过程的源代码文件。它显示提示要求用户输入三个整数，调用 Readlnt 获取数值，并将它们插入数组：

```text
;提示整数输入请求    (_prompt.asm)
INCLUDE Irvine32.inc
.code
;--------------------------------
PromptForIntegers PROC
;提示用户为数组输入整数，并用
;用户输入填充该数组。
;接收：
;    ptrPrompt:PTR BYTE    ;提示信息字符串
;    ptrArray:PTR DWORD    ;数组指针
;    arraySize:DWORD       ;数组大小
;返回：无
;--------------------------------
arraySize EQU [ebp+16]
ptrArray EQU [ebp+12]
ptrPrompt EQU [ebp+8]
    enter 0,0
    pushad                     ;保存全部寄存器
    mov ecx,arraySize
    cmp    ecx,0               ;数据大小WO?
    jle    L2                  ;是：退出
    mov edx,ptrPrompt          ;提示信息的地址
    mov esi,ptrArray
L1: call WriteString           ;显示字符串
    call ReadInt               ;将整数读入EAX
    call Crlf                  ;换行
    mov [esi],eax              ;保存入数组
    add esi,4                  ;下一个整数
    loop L1
L2: popad                      ;恢复全部寄存器
    leave 
    ret 12                     ;恢复堆栈
PromptForIntegers ENDP
END
```

#### ArraySum

\_arraysum.asm 模块为 ArraySum 过程，计算数组元素之和，并用 EAX 返回计算结果：

```text
;ArraySumit程    (_arrysum.asm)
INCLUDE Irvine32.inc
.code
;----------------------------------------------   
ArraySum PROC
;计算32位整数数组之和。
;接收：
;    ptrArray    ;数组指针
;    arraySize    ;数组大小(DWROD)
;返回：EAX = 和数
;----------------------------------------------   
ptrArray EQU [ebp+8]
arraySize EQU [ebp+12]
    enter 0,0
    push ecx                      ;EAX 不入栈
    push esi

    mov    eax, 0                 ;和数清零
    mov    esi, ptrArray
    mov    ecx,arraySize
    cmp    ecx, 0                 ;数组大小WO?
    jle    L2                     ;是：退出
L1: add    eax, [esi]             ;将每个整数加到和数中
    add    esi,4                  ;指向下一个整数
    loop L1                       ;按数组大小重复

L2: pop esi
    pop ecx                       ;用EAX返回和数
    leave
    ret    8                      ;恢复堆栈
ArraySum ENDP
END
```

#### DisplaySum

\_display.asm 模块为 DisplaySum 过程，显示标号和和数的结果：

```text
;DisplaySum 过程    (_display.asm)
INCLUDE Irvine32.inc
.code
;-----------------------------------------
DisplaySum PROC
;在控制台显示和数。
;接收：
;    ptrPrompt      ;提示字符串的偏移量
;    theSum         ;数组和数(DWROD)
;返回：无
;-----------------------------------------
theSum EQU [ebp+12]
ptrPrompt EQU [ebp+8]
    enter 0,0
    push eax
    push edx

    mov edx,ptrPrompt                           ;提示字符串的指针
    call WriteString
    mov eax,theSum
    call Writelnt                               ;显示 EAX
    call Crlf

    pop edx
    pop eax
    leave
    ret 8                                        ;恢复堆栈
DisplaySum ENDP
END
```

#### Startup 模块

Sum\_main.asm 模块为启动过程 \(main\)。其中的 EXTERN 伪指令指定了三个外部过程。为了使源代码更加友好，用 EQU 伪指令再次定义了过程名：

```text
ArraySum      EQU ArraySum@0
PromptForIntegers EQU PromptForIntegers@0
DisplaySum     EQU DisplaySum@0
```

每次过程调用之前，用注释说明了参数顺序。该过程使用 STDCALL 参数传递规范：

```text
;整数求和过程(Sum_main.asm)
;多模块示例
;本程序由用户输入多个整数，
;将它们存入数组，计算数组之和，
;并显示和数。
INCLUDE Irvine32.inc

EXTERN PromptForIntegers@0:PROC
EXTERN ArraySum@0:PROC, DisplaySum@0:PROC
;为方便起见，重新定义外部符号
ArraySum          EQU ArraySum@0
PromptForIntegers EQU PromptForIntegers@0
DisplaySum        EQU DisplaySum@0
;修改 Count 来改变数组大小：
Count = 3

.data
prompt1 BYTE "Enter a signed integer: ",0
prompt2 BYTE "The sum of the integers is: ",0
array DWORD Count DUP(?)
sum DWORD ?
.code
main PROC
    call Clrscr
;PromptForIntegers (addr prompt1, addr array, Count)
    push Count
    push OFFSET array
    push OFFSET prompt1
    call PromptForIntegers
;sum = ArraySum(addr array, Count)
    push Count
    push OFFSET array
    call ArraySum
    mov sum,eax
;DisplaySum(addr prompt2, sum)
    push sum
    push OFFSET prompt2
    call DisplaySum
    call Crlf
    exit
main ENDP
END main
```

## 8.22 用INVOKE和PROTO新建模块

32 位模式中，可以用 Microsoft 的 INVOKE、PROTO 和扩展 PROC 伪指令新建多模块程序。与更加传统的 CALL 和 EXTERN 相比，它们的主要优势在于：能够将 INVOKE 传递的参数列表与 PROC 声明的相应列表进行匹配。

现在用 INVOKE、PROTO 和高级 PROC 伪指令重新编写 ArraySum。为每个外部过程创建含有 PROTO 伪指令的头文件是很好的开始。每个模块都会包含这个文件 \( 用 INCLUDE 伪指令\) 且不会增加任何代码量或运行时开销。

如果一个模块不调用特定过程，汇编器就会忽略相应的 PROTO 伪指令。

sum.inc 头文件本程序的 sum.inc 头文件如下所示：

```text
; (sum.inc)
INCLUDE Irvine32.inc

PromptForIntegers PROTO,
    ptrPrompt:PTR BYTE,        ; 提示字符串
    ptrArray:PTR DWORD,        ; 数组指针
    arraySize:DWORD            ; 数组大小

ArraySum PROTO,
    ptrArray:PTR DWORD,        ; 数组指针
    arraySize:DWORD            ; 数组大小

DisplaySum PROTO,
    ptrPrompt:PTR BYTE,        ; 提示字符串
    theSum:DWORD               ; 数组之和
```

#### \_prompt 模块

\_prompt.asm 文件用 PROC 伪指令为 PromptForIntegers 过程声明参数，用 INCLUDE 将 sum.inc 复制到本文件：

```text
; 提示整数输入请求          (_prompt.asm)
INCLUDE sum.inc        ; 获得过程原型
.code
;-----------------------------------------------------
PromptForIntegers PROC,
  ptrPrompt:PTR BYTE,        ; 提示字符串
  ptrArray:PTR DWORD,        ; 数组指针
  arraySize:DWORD            ; 数组大小
;
; 提示用户输入数组元素值，并用用户输入
; 填充数组
; 返回：无
;-----------------------------------------------------
    pushad                 ; 保存所有寄存器

    mov  ecx,arraySize
    cmp  ecx,0             ; 数组大小 <= 0?
    jle  L2                ; 是: 退出
    mov  edx,ptrPrompt     ; 提示信息的地址
    mov  esi,ptrArray

L1:    call WriteString    ; 显示字符串
    call ReadInt           ; 把整数读入EAX
    call Crlf              ; 换行
    mov  [esi],eax         ; 保存入数组
    add  esi,4             ; 下一个整数
    loop L1

L2:    popad               ; 恢复所有寄存器
    ret
PromptForIntegers ENDP
END
```

与前面的 PromptForIntegers 版本比较，语句 enter 0，0 和 leave 不见了，这是因为当 MASM 遇到 PROC 伪指令及其声明的参数时，会自动生成这两条语句。同样，RET 指令也不需要自带常数参数了，PROC 会处理好。

#### \_arraysum 模块

接下来，\_arraysum.asm 文件包含了 ArraySum 过程：

```text
; ArraySum 过程                 (_arrysum.asm)
INCLUDE sum.inc
.code
;-----------------------------------------------------
ArraySum PROC,
    ptrArray:PTR DWORD,    ; 数组指针
    arraySize:DWORD        ; 数组大小
;
; 计算 32 位整数数组之和
; 返回:  EAX = 和数
;-----------------------------------------------------
    push ecx              ; EAX 不入栈
    push esi

    mov  eax,0            ; 和数清零
    mov  esi,ptrArray
    mov  ecx,arraySize
    cmp  ecx,0            ; 数组大小 <= 0?
    jle  L2               ; 是: 退出

L1:    add  eax,[esi]     ; 将每个整数加到和数中
    add  esi,4            ; 指向下一个整数
    loop L1               ; 按数组大小重复

L2:    pop esi
    pop ecx               ; 用 EAX 返回和数
    ret
ArraySum ENDP
END
```

#### \_display 模块

\_display.asm 文件包含了 DisplaySum 过程：

```text
; DisplaySum 过程        (_display.asm)
INCLUDE Sum.inc
.code
;-----------------------------------------------------
DisplaySum PROC,
    ptrPrompt:PTR BYTE,    ; 提示字符串
    theSum:DWORD           ; 数组之和
;
; 控制台显示和数
; 返回：无
;-----------------------------------------------------
    push    eax
    push    edx

    mov    edx,ptrPrompt        ; 提示信息的指针
    call    WriteString
    mov    eax,theSum
    call    WriteInt            ; 显示 EAX
    call    Crlf

    pop    edx
    pop    eax
    ret
DisplaySum ENDP
END
```

#### Sum\_main 模块

Sum\_main.asm \( 启动模块 \) 包含主程序并调用所有其他的过程。它使用 INCLUDE 从 sum.inc 复制过程原型：

```text
; 整数求和程序         (Sum_main.asm)
INCLUDE sum.inc
Count = 3
.data
prompt1 BYTE  "Enter a signed integer: ",0
prompt2 BYTE  "The sum of the integers is: ",0
array   DWORD  Count DUP(?)
sum     DWORD  ?

.code
main PROC
    call Clrscr

    INVOKE PromptForIntegers, ADDR prompt1, ADDR array, Count
    INVOKE ArraySum, ADDR array, Count
    mov    sum,eax
    INVOKE DisplaySum, ADDR prompt2, sum

    call Crlf
    exit
main ENDP
END main
```

小结 本节与上一节《[用Extern伪指令新建模块](http://c.biancheng.net/view/3665.html)》展示了在 32 位模式中新建多模块程序的两种方法：

* 第一种使用 的是更传统的EXTERN伪指令；
* 第二种使用的是INVOKE. PROTO和PROC的高级功能。

后一种中的伪指令简化了很多细节，并为 Windows API 函数调用进行了优化。此外，它们还隐藏了一些细节，因此，编程者可能更愿意使用显式的堆栈参数和 CALL 及 EXTERN 伪指令。

## 8.23 使用USES运算符注意事项

在《[USES运算符](http://c.biancheng.net/view/3540.html)》一节中列出了在过程开始保存、结尾恢复的寄存器名。汇编器自动为每个列出的寄存器生成相应的 PUSH 和 POP 指令。

但是必须注意的是：如果过程用常数偏移量访问其堆栈参数，比如 \[ebp+8\]，那么声明该过程时不能使用 USES 运算符。现在举例说明其原因。下面的 MySub1 过程用 USES 运算符保存和恢复 ECX 和 EDX：

```text
MySub1 PROC USES ecx edx
  ret
MySub1 ENDP
```

当 MASM 汇编 MySub1 时，生成代码如下：

```text
push ecx
push edx
pop edx
pop ecx
ret
```

假设在使用 USES 的同时还使用了堆栈参数，如 MySub2 过程所示，该参数预期保存的堆栈地址为 EBP+8：

```text
MySub2 PROC USES ecx edx
    push ebp                 ;保存基址指针
    mov ebp, esp             ;堆栈帧基址
    mov eax, [ebp+8]         ;取堆栈参数
    pop ebp                  ;恢复基址指针
    ret 4                    ;清除堆栈
MySub2 ENDP
```

则 MASM 为 MySub2 生成的相应代码如下：

```text
push ecx
push edx
push ebp
mov ebp,esp
mov eax, dword ptr [ebp+8]   ;错误地址！
pop ebp
pop edx
pop ecx
ret 4
```

由于汇编器在过程开头插入了 ECX 和 EDX 的 PUSH 指令，使得堆栈参数的偏移量发生变化，从而导致结果错误。

下图说明了为什么堆栈参数现在必须以［EBP+16］来引用。USES 在保存 EBP 之前修改了堆栈，破坏了子程序常用的标准开始代码。

![img](http://c.biancheng.net/uploads/allimg/190516/4-1Z5161453054W.gif)

提示：前面介绍了 PROC 伪指令声明堆栈参数的高级语法。在那种情况下，USES 运算符不会带来问题。

## 8.24 向堆栈传递8位和16位参数

32 位模式中，向过程传递堆栈参数时，最好是压入 32 位操作数。虽然也可以将 16 位操作数入栈，但是这样会使得 EBP 不能对齐双字边界，从而可能导致出现页面失效、降低运行时性能。因此，在入栈之前，要把操作数扩展为 32 位。下面的 Uppercase 过程接收一个字符参数，并用 AL 返回其大写字母：

```text
Uppercase PROC
    push ebp
    mov ebp, esp
    mov    al, [esp+8 ]          ;AL=字符
    cmp    al, 'a'               ;小于'a' ?
    jb L1                        ;是：什么都不做
    cmp    al, 'z'               ;大于'z' ?
    ja L1                        ;是：什么都不做
    sub    al, 32                ;否：转换字符
L1:
    pop ebp
    ret 4                        ;清除堆栈
Uppercase ENDP
```

当向 Uppercase 传递一个字母字符时，PUSH 指令自动将其扩展为 32 位：

```text
push 'x'
call Uppercase
```

如果传递的是字符变量就需要更小心一些，因为 PUSH 指令不允许操作数为 8 位：

```text
.data
charVal BYTE 'x'
.code
push charVal                 ;语法错误！
call Uppercase
```

相反，要用 MOVZX 把字符扩展到 EAX：

```text
movzx eax,charVal             ;扩展并传送
push eax
call Uppercase
```

### 16 位参数示例

假设现在想向之前给出的 AddTwo 过程传递两个 16 位整数。由于该过程期望的数值为 32 位，所以下面的调用会发生错误：

```text
.data
word1 WORD 1234h
word2 WORD 4111h
.code
push word1
push word2
call AddTwo                    ;错误！
```

因此，可以在每个参数入栈之前进行全零扩展。下面的代码将会正确调用 AddTwo：

```text
movzx eax,word1
push eax
movzx eax,word2
push eax
call AddTwo                    ; EAX 为和数
```

一个过程的主调者必须保证它传递的参数与过程期望的参数是一致的。对堆栈参数而言，参数的顺序和大小都很重要！

## 8.25 32位模式下传递64位参数

32 位模式中，通过堆栈向子程序传递 64 位参数时，先将参数的高位双字入栈，再将其低位双字入栈。这样就使得整数在堆栈中是按照小端顺序（低字节在低地址）存放的，因而子程序容易检索到这些数值，如同下面的 WriteHex64 过程操作一样。该过程用十六进制显示 64 位整数：

```text
WriteHex64 PROC
    push ebp
    mov ebp, esp
    mov eax, [ebp+12]     ;高位双字
    call WriteHex
    mov eax, [ebp+8]      ;低位双字
    call WriteHex
    pop ebp
    ret 8
WriteHex64 ENDP
```

WriteHex64 的调用示例如下，它先把 longVal 的高半部分入栈，再把 longVal 的低半部分入栈：

```text
.data
longVal QWORD 1234567800ABCDEFh
.code
push DWORD PTR longVal + 4            ;高位双字
push DWORD PTR longVal                ;低位双字
call WriteHex64
```

下图显示的是在 EBP 入栈，并把 ESP 复制给 EBP 之后，WriteHex64 的堆栈帧示意图。

![img](http://c.biancheng.net/uploads/allimg/190516/4-1Z516162546462.gif)

## 8.26 非双字局部变量

在声明不同大小的局部变量时，LOCAL 伪指令的操作会变得很有趣。每个变量都按照其大小来分配空间：8 位的变量分配给下一个可用的字节，16 位的变量分配给下一个偶地址（字对齐），32 位变量分配给下一个双字对齐的地址。

现在来看几个例子。首先，Example 过程含有一个局部变量 var1，类型为 BYTE：

```text
Example1 PROC
    LOCAL var1:byte
    mov al,var1       ;[EBP-1]
    ret
Example1 ENDP
```

由于 32 位模式中，堆栈偏移量默认为 32 位，因此，var1 可能被认为会存放于 EBP-4 的位置。实际上，如下图所示，MASM 将 EBP 减去 4，但是却把 var1 存放在 EBP-1，其下面的三个字节并未使用（用 nu 标记，表示没有使用）。图中，每个方块表示一个字节。

![&#x4E3A;&#x5C40;&#x90E8;&#x53D8;&#x91CF;&#x4FDD;&#x7559;&#x7A7A;&#x95F4;](http://c.biancheng.net/uploads/allimg/190516/4-1Z5161GF34M.gif)

过程 Example2 含一个双字局部变量和一个字节局部变量：

```text
Example2 PROC
  local temp:dword, SwapFlag:BYTE
  ...
  ret
Example2 ENDP
```

汇编器为 Example2 生成的代码如下所示。ADD 指令将 ESP 加 -8，在 ESP 和 EBP 之间为这两个局部变量预留了空间：

```text
push ebp
mov ebp, esp
add esp,0FFFFFFF8h     ; ESP+（-8）
mov eax,[ebp-4]        ; temp
mov bl,[ebp-5]         ; SwapFlag
leave
ret
```

虽然 SwapFlag 只是一个字节变量，但是 ESP 还是会下移到堆栈中下一个双字的位置。下图以字节为单位详细展示了堆栈的情况：SwapFlag 确切的位置以及位于其下方的三个没有使用的空间（用 nu 标记）。图中，每个方块表示一个字节。

![Example2&#x4E2D;&#x4E3A;&#x5C40;&#x90E8;&#x53D8;&#x91CF;&#x4FDD;&#x7559;&#x7A7A;&#x95F4;](http://c.biancheng.net/uploads/allimg/190516/4-1Z5161GK54I.gif)

如果要创建超过几百字节的数组作为局部变量，那么一定要确保为运行时堆栈预留足够的空间。此时可以使用 STACK 伪指令。比如，在 Irvine32 链接库中，要预留 4096 个字节的堆栈空间：

```text
.stack 4096
```

对嵌套调用来说，不论程序执行到哪一步，运行时堆栈都必须大到能够容纳下全部的活跃局部变量。比如在下面的代码中，Sub1 调用 Sub2，Sub2 调用 Sub3，每个过程都有一个局部数组变量：

```text
Sub1 PROC
local array1 [50]:dword    ; 200 字节
callSub2
...
ret
Sub1 ENDP

Sub2 PROC
local array2 [80]:word     ; 160 字节
callSub3
...
ret
Sub2 ENDP

Sub3 PROC
local array3 [300]:dword    ; 1200 字节
...
ret
Sub3 ENDP
```

当程序进入 Sub3 时，运行时堆栈中有来自 Sub1、Sub2 和 Sub3 的全部局部变量。那么堆栈总共需要：1560 个字节保存局部变量，加上两个过程的返回地址（8 字节），还要加上在过程中入栈的所有寄存器占用的空间。若过程为递归调用，则堆栈空间大约为其局部变量与参数总的大小乘以预计的递归次数。

## 8.27 Java虚拟机（JVM）工作原理

虽然本教程的内容为 x86 处理器的原生[汇编语言](http://c.biancheng.net/asm/)，但是了解其他机器架构如何工作也是有益的。JVM 是基于堆栈机器的首选示例。JVM 用堆栈实现数据传送、算术运算、比较和分支操作，而不是用寄存器来保存操作数（如同 x86 一样）。

### [Java](http://c.biancheng.net/java/) 虚拟机

Java 虚拟机（JVM）是执行已编译 Java 字节码的软件。它是 Java 平台的重要组成部分，包括程序、规范、库和[数据结构](http://c.biancheng.net/data_structure/)，让它们协同工作。Java 字节码是指编译好的 Java 程序中使用的机器语言的名字。

JVM 执行的编译程序包含了 Java 字节码。每个 Java 源程序都必须编译为 Java 字节码（形式为 .class 文件）后才能执行。包含 Java 字节码的程序可以在任何安装了 Java 运行时软件的计算机系统上执行。

例如，一个 Java 源文件名为 Account.java，编译为文件 Account.class。这个类文件是该类中每个方法的字节码流。JVM 可能选择实时编译（just-in-time compilation）技术把类字节码编译为计算机的本机机器语言。

正在执行的 Java 方法有自己的堆栈帧存放局部变量、操作数栈、输入参数、返回地址和返回值。操作数区实际位于堆栈顶端，因此，压入这个区域的数值可以作为算术和逻辑运算的操作数，以及传递给类方法的参数。

在局部变量被算术运算指令或比较指令使用之前，它们必须被压入堆栈帧的操作数区域。通常把这个区域称为操作数栈（operand stack）。

Java 字节码中，每条指令包含 1 字节的操作码、零个或多个操作数。操作码可以用 Java 反汇编工具显示名字，如 iload、istore、imul 和 goto。每个堆栈项为 4 字节（32 位）。

#### 查看反汇编字节码

Java 开发工具包（JDK）中的工具 javap.exe 可以显示 java.class 文件的字节码，这个操作被称为文件的反汇编。命令行语法如下所示：

```text
javap -c classname
```

比如，若类文件名为 Account.class，则相应的 javap 命令行为：

```text
javap -c Account
```

安装 Java 开发工具包后，可以在 \bin 文件夹下找到 javap.exe 工具。

### 指令集

#### 1\) 基本数据类型

JVM 可以识别 7 种基本数据类型，如下表所示。和 x86 整数一样，所有有符号整数都是二进制补码形式。但它们是按照大端顺序存放的，即高位字节位于每个整数的起始地址（x86 的整数按小端顺序存放）。

| 数据类型 | 所占字节 | 格式 | 数据类型 | 所占字节 | 格式 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| char | 2 | Unicode 字符 | long | 8 | 有符号整数 |
| byte | 1 | 有符号整数 | float | 4 | IEEE 单精度实数 |
| short | 2 | 有符号整数 | double | 8 | IEEE 双精度实数 |
| int | 4 | 有符号整数 |  |  |  |

#### 2\) 比较指令

比较指令从操作数栈的顶端弹出两个操作数，对它们进行比较，再把比较结果压入堆栈。现在假设操作数入栈顺序如下所示：

![img](http://c.biancheng.net/uploads/allimg/190517/4-1Z51G01G62c.gif)

下表给出了比较 op1 和 op2 之后压入堆栈的数值：

| op1 和 op2 比较的结果 | 压入操作数栈的数值 |
| :--- | :--- |
| op1 &gt; op2 | 1 |
| op1 = op2 | 0 |
| op1 &lt; op2 | -1 |

dcmp 指令比较双字，fcmp 指令比较浮点数。

#### 3\) 分支指令

分支指令可以分为有条件分支和无条件分支。Java 字节码中无条件分支的例子是 goto 和 jsr。

goto 指令无条件分支到一个标号：

```text
goto label
```

jsr 指令调用用标号定义的子程序。其语法如下：

```text
jsr label
```

条件分支指令通常检测从操作数栈顶弹出的数值。根据该值，指令决定是否分支到给定标号。比如，ifle 指令就是当弹出数值小于等于 0 时跳转到标号。其语法如下：

```text
ifle label
```

同样，ifgt 指令就是当弹出数值大于等于 0 时跳转到标号。其语法如下：

```text
ifgt label
```

### Java 反汇编示例

为了帮助理解 Java 字节码是如何工作的，本节将给出用 Java 编写的一些短代码例子。在这些例子中，请注意不同版本 Java 的字节码清单细节会存在些许差异。

【示例 1】两个整数相加

下面的 Java 源代码行实现两个整数相加，并将和数保存在第三个变量中：

```text
int A = 3;
int B = 2;
int sum = 0;
sum = A + B;
```

该 Java 代码的反汇编如下：

```text
iconst_3
istore_0
iconst_2
istore_l
iconst_0
istore_2
iload_0
iload_l
iadd
istore_2
```

每个编号行表示一条 Java 字节码指令的字节偏移量。本例中，可以发现每条指令都只占一个字节，因为指令偏移量的编号是连续的。

尽管字节码反汇编一般不包括注释，这里还是会将注释添加上去。虽然局部变量在运行时堆栈中有专门的保留区域，但是指令在执行算术运算和数据传送时还会使用另一个堆栈，即操作数栈。为了避免在这两个堆栈间产生混淆，将用索引值来指代变量位置，如 0、1、2 等。

现在来仔细分析刚才的字节码。开始的两条指令将一个常数值压入操作数栈，并把同一个值弹出到位置为 0 的局部变量：

```text
iconst_3 //常数（3）压入操作数栈
istore_0 //弹出到局部变量0
```

接下来的四行将其他两个常数压入操作数栈，并把它们弹岀到位置分别为 1 和 2 的局部变量：

```text
iconst_2 //常数（2）压入操作数栈
istore_1 //弹出到局部变量1
iconst_0 //常数（0）压入操作数栈
istore_2 //弹出到局部变量2
```

由于已经知道了该生成字节码的 Java 源代码，因此，很明显下表列出的是三个变量的位置索引：

| 位置索引 | 变量名 |
| :--- | :--- |
| 0 | A |
| 1 | B |
| 2 | sum |

接下来，为了实现加法，必须将两个操作数压入操作数栈。指令 iload\_0 将变量 A 入栈，指令 iload\_1 对变量 B 进行相同的操作：

```text
iload_0 // （A 入栈）
iload_1 // （B 入栈）
```

现在操作数栈包含两个数：

![img](http://c.biancheng.net/uploads/allimg/190517/4-1Z51G01K1456.gif)

这里并不关心这些例子的实际机器表示，因此上图中的运行时堆栈是向上生长的。每个堆栈示意图中的最大值即为栈顶。

指令 iadd 将栈顶的两个数相加，并把和数压入堆栈：

```text
iadd
```

操作数栈现在包含的是 A、B 的和数：

![img](http://c.biancheng.net/uploads/allimg/190517/4-1Z51G01P9505.gif)

指令 istore\_2 将栈顶内容弹出到位置为 2 的变量，其变量名为 sum：

```text
istore_2
```

操作数栈现在为空。

【示例 2】两个 Double 类型数据相加

下面的 Java 代码片段实现两个 double 类型的变量相加，并将和数保存到 sum。它执行的操作与两个整数相加示例相同，因此这里主要关注的是整数处理与 double 处理的差异：

```text
double A = 3.1;
double B = 2;
double sum = A + B;
```

本例的反汇编字节码如下所示，用 javap 实用程序可以在右边插入注释：

```text
ldc2_w #20; // double 3.Id
dstore_0
ldc2_w #22; // double 2.Od
dstore_2
dload_0
dload_2
dadd
dstore_4
```

下面对这个代码进行分步讨论。偏移量为 0 的指令 ldc2\_w 把一个浮点常数（3.1）从常数池压入操作数栈。ldc2 指令总是用两个字节作为常数池区域的索引：

```text
ldc2_w #20; // double 3.ld
```

偏移量为 3 的 dstore 指令从堆栈弹出一个 double 数，送入位置为 0 的局部变量。该指令起始偏移量（3）反映出第一条指令占用的字节数（操作码加上两字节索引）：

```text
dstore_0 //保存到 A
```

同样，接下来偏移量为 4 和 7 的两条指令对变量 B 进行初始化：

```text
ldc2_w #22; // double 2.Od
dstore_2 // 保存到 B
```

指令 dload\_0 和 dload\_2 把局部变量入栈，其索引指的是 64 位位置（两个变量栈项），因为双字数值要占用 8 个字节：

```text
dload_0
dload_2
```

接下来的指令（dadd）将栈顶的两个 double 值相加，并把和数入栈：

```text
dadd
```

最后，指令 dstore\_4 把栈顶内容弹出到位置为 4 的局部变量：

```text
dstore_4
```

### JVM 条件分支

了解 JVM 怎样处理条件分支是理解 Java 字节码的重要一环。比较操作总是从堆栈栈顶弹出两个数据，对它们进行比较后，再把结果数值入栈。条件分支指令常常跟在比较操作的后面，利用栈顶数值决定是否分支到目标标号。比如，下面的 Java 代码包含一个简单的 IF 语句，它将两个数值中的一个分配给一个布尔变量：

```text
double A = 3.0;
boolean result = false;
if( A > 2.0 )
result = false;
else
result = true;
```

该 Java 代码对应的反汇编如下所示：

```text
ldc2_w #26; // double 3.Od
dstore_0 // 弹出到 A
iconst_0 // false = 0
istore_2 //保存到 result
dload_0
ldc2_w #22; // double 2.0d
dcmpl
ifle 19     //如果 A ≤ 2.0,转到 19
iconst_0 // false
istore_2 // result = false
goto 21     //跳过后面两条语句
iconst_l // true
istore_2 // result = true
```

开始的两条指令将 3.0 从常数池复制到运行时堆栈，再把它从堆栈弹岀到变量 A：

```text
ldc2_w #26; // double 3.0d
dstore_0 // 弹出至A
```

接下来的两条指令将布尔值 false （等于 0）从常量区复制到堆栈，再把它弹出到变量 result：

```text
iconst_0 // false = 0
istore_2 // 保存到 result
```

A 的值（位置 0）压入操作数栈，数值 2.0 紧跟其后入栈：

```text
dload_0   //A 入栈
ldc2_w #22; // double 2.0d
```

操作数栈现在有两个数值：

![img](http://c.biancheng.net/uploads/allimg/190517/4-1Z51G01RAc.gif)

指令 dcmpl 将两个 double 数弹出堆栈进行比较。由于栈顶的数值（2.0）小于它下面的数值（3.0），因此整数 1 被压入堆栈。

```text
dcmpl
```

如果从堆栈弹出的数值小于等于 0，则指令 ifle 就分支到给定的偏移量：

```text
ifle 19  //如果 stack.pop() <= 0，转到 19
```

这里要回顾一下之前给出的 Java 源代码示例，若 A&gt;2.0，其分配的值为 false：

```text
if( A > 2.0 )    result = false;else    result = true;
```

如果 A &lt;= 2.0，Java 字节码就把 IF 语句转向偏移量为 19 的语句，为 result 分配数值 true。与此同时，如果不发生到偏移量 19 的分支，则由下面几条指令把 false 赋给 result：

```text
iconst_0   // false
istore_2   // result = false
goto 21   //跳过后面两条指令
```

偏移量 16 的指令 goto 跳过后面两行代码，它们的作用是给 result 分配 true：

```text
iconst_l // true
istore_2 // result = true
```

Java 虚拟机的指令集与 x86 处理器系列的指令集有很大的不同。它采用面向堆栈的方法实现计算、比较和分支，与 x86 指令经常使用寄存器和内存操作数形成了鲜明的对比。

虽然字节码的符号反汇编不如 x86 汇编语言简单，但是，编译器生成字节码也是相当容易的。每个操作都是原子的，这就意味着它只执行一个操作。

若 JVM 使用的是实时编译器，则 Java 字节码只要在执行前转换为本地机器语言即可。就这方面来说，Java 字节码与基于精简指令集（RISC）模型的机器语言有很多共同点。

