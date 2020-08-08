---
description: >-
  结构（structure）是一组逻辑相关变量的模板或模式。结构中的变量被称为字段（fields）。程序语句可以把结构作为整体进行访问，也可以访问其中的单个字段。结构常常包含不同类型的字段。联合（union）也会把多个标识符组织在一起，但是这些标识符会在内存同一区域内相互重叠。汇编语言中的结构与
  C 和 C++ 中的结构同样重要。只需要一点转换，就可以从 MS-WindowsAPI 库中获得任何
---

# 10 汇编语言结构和宏

## 10.1 STRUCT和ENDS伪指令：定义结构

定义结构使用的是 STRUCT 和 ENDS 伪指令。在结构内，定义字段的语法与一般的变量定义是相同的。结构对其包含字段的数量几乎没有任何限制：

```text
name STRUCT
  field-declarations
name ENDS
```

字段初始值若结构字段有初始值，那么在创建结构变量时就要进行赋值。字段初始值可以使用各种类型：

* 无定义：运算符？使字段初始值为无定义。
* 字符串文本：用引号括起的字符串。
* 整数：整数常数和整数表达式。
* 数组：DUP 运算符可以初始化数组元素。

下面的 Employee 结构描述了雇员信息，其包含字段有 ID 号、姓氏、服务年限，以及薪酬历史信息数组。结构定义如下所示，定义必须在声明 Employee 变量之前：

```text
Employee STRUCT
    IdNum BYTE "000000000"
    LastName BYTE 30 DUP(0)
    Years WORD 0
    SalaryHistory DWORD 0,0,0,0
Employee ENDS
```

该结构内存保存形式的线性表示如下：

![img](http://c.biancheng.net/uploads/allimg/190520/4-1Z520142Z3V7.gif)

#### 对齐结构字段

为了获得最好的内存 I/O 性能，结构成员应按其数据类型进行地址对齐。否则，CPU 将会花更多时间访问成员。例如，一个双字成员应对齐到双字边界。下表列岀了 Microsoft C 和 [C++](http://c.biancheng.net/cplus/) 编译器，以及 Win32 API 函数的对齐方式。[汇编语言](http://c.biancheng.net/asm/)中的 ALIGN 伪指令会使其后的字段或变量按地址对齐：

```text
ALIGN datatype
```

| 成员类型 | 对齐方式 | 成员类型 | 对齐方式 |
| :--- | :--- | :--- | :--- |
| BYTE, SBYTE | 对齐到 8 位（字节）边界 | REAL4 | 对齐到 32 位（双字）边界 |
| WORD, SWORD | 对齐到 16 位（字）边界 | REAL8 | 对齐到 64 位（四字）边界 |
| DWORD, SDWORD | 对齐到 32 位（双字）边界 | structure | 所有成员的最大对齐要求 |
| QWORD | 对齐到 64 位（四字）边界 | union | 第一个成员的对齐要求 |

比如，下面的例子就把 myVar 对齐到双字边界：

```text
.data
ALIGN DWORD
myVar DWORD ?
```

现在正确地定义 Employee 结构，利用 ALIGN 将 Years 按字（WORD）边界对齐，SalaryHistory 按双字（DWORD）边界对齐。注释为字段大小：

```text
Employee STRUCT
    IdNum BYTE "000000000"              ; 9
    LastName BYTE 30 DUP(0)             ; 30
    ALIGN WORD                          ; 加 1 字节
    Years WORD 0                        ; 2
    ALIGN DWORD                         ; 加 2 字节
    SalaryHistory DWORD 0,0,0,0         ; 16
Employee ENDS                           ;共 60 字节
```

## 10.2 声明结构变量

结构变量可以被声明，并能选择为是否用特定值进行初始化。语法如下，其中 structureType 已经用 STRUCT 伪指令定义过了：

```text
identifier structureType <initializer-list>
```

identifier 的命名规则与 MASM 中其他变量的规则相同。initializer-list 为可选项，但是如果选择使用，则该项就是一个用逗号分隔的汇编时常数列表，需要与特定结构字段的数据类型相匹配：

```text
initializer [, initializer] ...
```

空括号 &lt;&gt; 使结构包含的是结构定义的默认字段值。此外，还可以在选定字段中插入新值。结构字段中的插入值顺序为从左到右，与结构声明中字段的顺序一致。这两种方法的示例如下，使用的结构是 COORD 和 Employee：

```text
.data
point1 COORD <5,10>             ; X = 5, Y = 10
point2 COORD <20>               ; X = 20, Y = ?
point3 COORD <>                 ; X = ?, Y = ?
worker Employee <>              ; 默认初始值
```

可以只覆盖选定字段的初始值。下面的声明只覆盖了 Employee 结构的 IdNum 字段，而其他字段仍为默认值：

```text
person1 Employee <"555223333">
```

还有一种形式是使用大括号 {…} 而不是尖括号：

```text
person2 Employee {"555223333"}
```

若字符串字段初始值的长度少于字段的定义，则多出的位置用空格填充。空字节不会自动插到字符串字段的尾部。通过插入逗号作为位置标记可以跳过结构字段。例如，下面的语句就跳过了 IdNum 字段，初始化了 LastName 字段：

```text
person3 Employee <, "dJones">
```

数组字段使用 DUP 运算符来初始化某些或全部数组元素。如果初始值比字段位数少，则多出的位置用零填充。下面的语句只初始化了前两个 SalaryHistory 的值，而其他的值则为 0：

```text
person4 Employee <, , ,2 DUP(20000)>
```

DUP 运算符能够用于定义结构数组，如下所示，AllPoints 中每个元素的 X 和 Y 字段都被初始化为 0：

```c
NumPoints = 3
AllPoints COORD NumPoints DUP(<0,0>)
```

#### 对齐结构变量

为了最好的处理器性能，结构变量在内存中的位置要与其最大结构成员的边界对齐。Employee 结构包含双字 \(DWORD\) 字段，因此，下面的定义使用了双字对齐：

```text
.data
ALIGN DWORD
person Employee <>
```

## 10.3 TYPE和SIZEOF运算符：引用结构变量和结构名称

使用 TYPE 和 SIZEOF 运算符可以引用结构变量和结构名称。例如，现在回到之前的 Employee 结构：

```text
Employee STRUCT
    IdNum BYTE "000000000"      ; 9
    LastName BYTE 30 DUP(0)     ; 30
    ALIGN WORD                  ; 加 1 字节
    Years WORD 0                ; 2
    ALIGN DWORD                 ; 加 2 字节
    SalaryHistory DWORD 0,0,0,0 ; 16
Employee ENDS                   ; 共 60 字节
```

给定数据定义：

```text
.data
worker Employee <>
```

则下列所有表达式返回的值都相同：

```text
TYPE Employee        ; 60
SIZEOF Employee      ; 60
SIZEOF worker        ; 60
```

TYPE 运算符返回的是标识符存储类型（BYTE、WORD、DWORD 等）的字节数。LENGTHOF 运算符返回的是数组元素的个数。SIZEOF 运算符则为 LENGTHOF 与 TYPE 的乘积。

#### 1\) 引用成员

引用已命名的结构成员时，需要用结构变量作为限定符。以 Employee 结构为例，在汇编时能生成下述常量表达式：

```text
TYPE Employee.SalaryHistory            ; 4
LENGTHOF Employee.SalaryHistory       ; 4
SIZEOF Employee.SalaryHistory           ; 16
TYPE Employee.Years                   ; 2
```

以下为对 worker（一个 Employee）的运行时引用：

```text
.data
worker Employee <>
.code
mov dx,worker.Years
mov worker.SalaryHistory, 20000              ;第一个工资
mov [worker.SalaryHistory+4 ], 30000         ;第二个工资
```

使用 OFFSET 运算符能获得结构变量中一个字段的地址：

```text
mov edx,OFFSET worker.LastName
```

#### 2\) 间接和变址操作数

间接操作数用寄存器（如 ESI）对结构成员寻址。间接寻址具有灵活性，尤其是在向过程传递结构地址或者使用结构数组的情况下。引用间接操作数时需要 PTR 运算符：

```text
mov esi,OFFSET worker
mov ax,(Employee PTR [esi]).Years
```

下面的语句不能汇编，原因是 Years 自身不能表明它所属的结构：

```text
mov ax, [esi].Years  ;无效
```

#### 变址操作数

用变址操作数可以访问结构数组。假设 department 是一个包含 5 个 Employee 对象的数组。下述语句访问的是索引位置为 1 的雇员的 Years 字段：

```text
.data
department Employee 5 DUP(<>)
.code
mov esi, TYPE Employee              ; 索引 = 1
mov department[esi].Years, 4
```

#### 数组循环

带间接或变址寻址的循环可以用于处理结构数组。下面的程序 \(AllPoints.asm\)为 AllPoints 数组分配坐标：

```text
; 数组循环       (AllPoints.asm)
INCLUDE Irvine32.inc
NumPoints = 3
.data
ALIGN WORD
AllPoints COORD NumPoints DUP(<0,0>)

.code
main PROC
    mov edi,0                    ; 数组索引
    mov ecx,NumPoints            ; 循环计数器
    mov ax,1                     ; 起始 X, Y 的值

L1:    mov (COORD PTR AllPoints[edi]).X,ax
    mov (COORD PTR AllPoints[edi]).Y,ax
    add edi,TYPE COORD
    inc ax
    loop L1

    exit
main ENDP
END main
```

#### 3\) 对齐的结构成员的性能

之前已经断言，处理器访问正确对齐的结构成员时效率更高。那么，非对齐字段会对性能产生多大影响呢？现在使用本章介绍的 Employee 结构的两种不同版本，进行一个简单的测试。测试将对第一个版本进行重命名，以便两种版本能在同一个程序中使用：

```text
EmployeeBad STRUCT   
    Idnum    BYTE "000000000"
    Lastname BYTE 30 DUP(0)
    Years    WORD 0
    SalaryHistory DWORD 0,0,0,0
EmployeeBad ENDS               

Employee STRUCT   
    Idnum    BYTE "000000000"
    Lastname BYTE 30 DUP(0)
    ALIGN    WORD
    Years    WORD 0
    ALIGN    DWORD
    SalaryHistory DWORD 0,0,0,0
Employee ENDS
```

下面的代码首先获取系统时间，再执行循环以访问结构字段，最后计算执行花费的时 间。变量 emp 可以声明为 Employee 对象或者 EmployeeBad 对象：

```text
.data
ALIGN DWORD
startTime DWORD ?                ; 对齐 startTime
emp EmployeeBad <>               ; 或: EmployeeBad

.code
    call    GetMSeconds          ; 获取系统时间
    mov    startTime,eax

    mov    ecx,0FFFFFFFFh        ; 循环计数器
L1:    mov    emp.Years,5
    mov    emp.SalaryHistory,35000
    loop    L1

    call    GetMSeconds        ; 获取开始时间
    sub    eax,startTime
    call    WriteDec           ; 显示执行花费的时间
```

在这个简单的测试程序中，使用正确对齐的 Employee 结构的执行时间为 6141 毫秒，而使用 EmployeeBad 结构的执行时间为 6203 毫秒。两者相差不大 \(62 毫秒\)，可能是因为处理器的内存 cache 将对齐问题最小化了。

## 10.4 实例：显示系统时间

MS-Windows 提供了设置屏幕光标位置和获取系统时间的控制台函数。要使用这些函数，先为两个预先定义的结构 COORD 和 SYSTEMTIME 创建实例：

```text
COORD STRUCT
    X WORD ?
    Y WORD ?
COORD ENDS

SYSTEMTIME STRUCT
    wYear WORD ?
    wMonth WORD ?
    wDayOfWeek WORD ?
    wDay WORD ?
    wHour WORD ?
    wMinute WORD ?
    wSecond WORD ?
    wMilliseconds WORD ?
SYSTEMTIME ENDS
```

这两个结构都在 SmallWin.inc 中进行了定义，这个文件位于汇编器的 INCLUDE 目录下，并且由 Irvine32.inc 引用。首先获取系统时间（调整本地时间），调用 MS-Windows 的 GetLocalTime 函数，并向其传递 SYSTEMTIME 结构的地址：

```text
.data
sysTime SYSTEMTIME <>
.code
INVOKE GetLocalTime, ADDR sysTime
```

接着，从 SYSTEMTIME 结构检索相应的数值：

```text
movzx eax,sysTime.wYear
call WriteDec
```

当 Win32 程序产生屏幕输出时，它要调用 MS-Windows GetStdHandle 函数来检索标准控制台输出句柄（一个整数）：

```text
.data
consoleHandle DWORD ?
.code
INVOKE GetStdHandle, STD_OUTPUT_HANDLE
mov consoleHandle,eax
```

设置光标位置要调用 MS-Windows SetConsoleCursorPosition 函数，并向其传递控制台输岀句柄，以及包含 X、Y 字符坐标的 COORD 结构变量：

```text
.data
XYPos COORD <10,5>
.code
INVOKE SetConsoleCursorPosition, consoleHandle, XYPos
```

#### 程序清单

下面的程序检索系统时间，并将其显示在指定的屏幕位置。该程序只在保护模式下运行：

```text
; 结构    （ShowTime.asm）
INCLUDE Irvine32.inc
.data
sysTime SYSTEMTIME <>
XYPos   COORD <10,5>
consoleHandle DWORD ?
colonStr BYTE ":",0

.code
main PROC
; 获取 Win32 控制台的标准输出句柄
    INVOKE GetStdHandle, STD_OUTPUT_HANDLE
    mov consoleHandle,eax

; 设置光标位置并获取系统时间
    INVOKE SetConsoleCursorPosition, consoleHandle, XYPos
    INVOKE GetLocalTime,ADDR sysTime
; 显示系统时间 (hh:mm:ss).
    movzx eax,sysTime.wHour          ; 小时
    call  WriteDec
    mov   edx,offset colonStr        ; ":"
    call  WriteString
    movzx eax,sysTime.wMinute        ; 分钟
    call  WriteDec
    call  WriteString                ; ":"
    movzx eax,sysTime.wSecond        ; 秒
    call  WriteDec
    call Crlf
    exit
main ENDP
END main
```

SmallWin.inc（自动包含在 Irvine32.inc 中）中的上述程序采用如下定义：

```text
STD_OUTPUT_HANDLE EQU -11
SYSTEMTIME STRUCT ...
COORD STRUCT ...
GetStdHandle PROTO,
  nStdHandle:DWORD
GetLocalTime PROTO,
  lpSystemTime:PTR SYSTEMTIME
SetConsoleCursorPosition PROTO,
  nStdHandle:DWORD,
  coords:COORD
```

下面是示例程序输出，执行时间为下午 12:16：

```text
12:16:35
Press any key to continue...
```

## 10.5 结构嵌套简述\[附带实例\]

结构还可以包含其他结构的实例。例如，Rectangle 可以用其左上角和右下角来定义，而它们都是 COORD 结构：

```text
Rectangle STRUCT
  UpperLeft COORD <>
  LowerRight COORD <>
Rectangle ENDS
```

Rectangle 变量可以被声明为不覆盖或者覆盖单个 COORD 字段。各种表达形式如下所示：

```text
rect1 Rectangle < >
rect2 Rectangle { }
rect3 Rectangle { {10,10}, {50,20} }
rect4 Rectangle < <10,10>, <50,20> >
```

下面是对其一个结构字段的直接引用：

```text
mov rect1.UpperLeft.X, 10
```

也可以用间接操作数访问结构字段。下例用 ESI 指向结构，并把 10 送人该结构左上角的 Y 坐标：

```text
mov esi,OFFSET rect1
mov (Rectangle PTR [esi]).UpperLeft.Y, 10
```

OFFSET 运算符能返回单个结构字段的指针，包括嵌套字段：

```text
mov edi,OFFSET rect2.LowerRight
mov (COORD PTR [edi]).X, 50
mov edi,OFFSET rect2.LowerRight.X
mov WORD PTR [edi], 50
```

#### 示例：醉汉行走

现在来看一个使用结构的小程序将会有所帮助。下面完成一个“醉汉行走”练习，用程序模拟一个不太清醒的教授从计算机科学假期聚会回家的路线。利用随机数生成器，选择该教授每一步行走的方向。假设教授处于一个虚构的网格中心，其中的每个方格代表的是北、南、东、西方向上的一步。现在按照随机路径通过网格，如下图所示。

![&#x9189;&#x6C49;&#x884C;&#x8D70;&#x7684;&#x793A;&#x4F8B;&#x8DEF;&#x5F84;](http://c.biancheng.net/uploads/allimg/190520/4-1Z5201G929123.gif)

本程序将使用 COORD 结构追踪这个人行走路径上的每一步，它们被保存在一个 COORD 对象数组中。

```text
WalkMax == 50
DrunkardWalk STRUCT
  path COORD WalkMax DUP(<0, 0>)
  pathsUsed WORD 0
DrunkardWalk ENDS
```

Walkmax 是一个常数，决定在模拟中教授能够行走的总步数。pathsUsed 字段表示在程序循环结束后，一共行走了多少步。教授每走一步，其位置就被记录在 COORD 对象中，并插入 path 数组下一个可用的位置。程序将在屏幕上显示这些坐标。

以下是完整的程序清单, 需在 32 位模式下运行：

```text
; 醉汉行走    (Walk. asm)
; 醉汉行走程序。教授的起点坐标为（25，25），并在周围徘徊
INCLUDE Irvine32.inc
WalkMax = 50
StartX = 25
StartY = 25

DrunkardWalk STRUCT
    path COORD WalkMax DUP(<0,0>)
    pathsUsed WORD 0
DrunkardWalk ENDS

DisplayPosition PROTO currX:WORD, currY:WORD

.data
aWalk DrunkardWalk <>

.code
main PROC
    mov esi,OFFSET aWalk
    call TakeDrunkenWalk
    exit
main ENDP

;-------------------------------------------------------
TakeDrunkenWalk PROC
    LOCAL currX:WORD, currY:WORD
;
; 向随机方向行走(北, 南, 东, 西)
; 接收: ESI 为 DrunkardWalk 结构的指针
; 返回:  结构初始化为随机数
;-------------------------------------------------------
    pushad

; 用 OFFSET 运算符获取 path，COORD 对象数组的地址，并将其复制到 EDI.
    mov edi,esi
    add edi,OFFSET DrunkardWalk.path
    mov ecx,WalkMax            ; 循环计数器
    mov currX,StartX           ; 当前 X 的位置
    mov currY,StartY           ; 当前 Y 的位置

Again:
    ; 把当前位置插入数组
    mov ax,currX
    mov (COORD PTR [edi]).X,ax
    mov ax,currY
    mov (COORD PTR [edi]).Y,ax

    INVOKE DisplayPosition, currX, currY

    mov      eax,4      ; 选择一个方向 (0-3)
    call  RandomRange

    .IF eax == 0        ; 北
      dec currY
    .ELSEIF eax == 1    ; 南
      inc currY
    .ELSEIF eax == 2    ; 西
      dec currX
    .ELSE               ; 东 (EAX = 3)
      inc currX
    .ENDIF

    add    edi,TYPE COORD    ; 指向下一个 COORD
    loop    Again

Finish:
    mov (DrunkardWalk PTR [esi]).pathsUsed, WalkMax
    popad
    ret
TakeDrunkenWalk ENDP

;-------------------------------------------------------
DisplayPosition PROC currX:WORD, currY:WORD
; 显示当前 X 和 Y 的位置
;-------------------------------------------------------
.data
commaStr BYTE ",",0
.code
    pushad
    movzx eax,currX                ; 当前 X 的位置
    call     WriteDec
    mov     edx,OFFSET commaStr    ; "," 字符串
    call     WriteString
    movzx eax,currY                ; 当前 Y 的位置
    call     WriteDec
    call     Crlf
    popad
    ret
DisplayPosition ENDP
END main
```

现在进一步查看 TakeDrunkenWalk 过程。过程接收指向 DrunkardWalk 结构的指针 \(ESI\)，利用 OFFSET 运算符计算 path 数组的偏移量，并将其复制到 EDI：

```text
mov edi,esi
add edi,OFFSET DrunkardWalk.path
```

教授初始位置的 X 和 Y 值 \(StartX 和 StartY\) 都被设置为 25，位于 50 x 50 虚拟网格的中点。循环计数器也进行了初始化：

```text
mov ecx, WalkMax ;循环计数器
mov currX, StartX  ;当前 X 的位置
mov currY, StartY  ;当前 Y 的位置
```

循环开始时，对 path 数组的第一项进行初始化：

```text
Again:
  ; 把当前位置插入数组
  mov ax,currX
  mov (COORD PTR [edi]).X,ax
  mov ax,currY
  mov (COORD PTR [edi]).Y,ax
```

路径结束时，在 pathsUsed 字段插入一个计数值，表示总共走了多少步：

```text
Finish:
  mov (DrunkardWalk PTR [esi]).pathsUsed, WalkMax
```

在当前的程序中，pathsUsed 总是等于 WalkMaX。不过，若在行走过程中发现障碍，如湖泊或建筑物，情况就会发生变化，循环将会在达到 WalkMax 之前结束。

## 10.6 联合 \(union\) 的声明和使用

结构中的每个字段都有相对于结构第一个字节的偏移量，而联合 \(union\) 中所有的字段则都起始于同一个偏移量。一个联合的存储大小即为其最大字段的长度。如果不是结构的组成部分，那么需要用 UNION 和 ENDS 伪指令来定义联合：

```text
unionname UNION
  union-fields
unionname ENDS
```

如果联合嵌套在结构内，其语法会有一点不同：

```text
structname STRUCT
  structure-fields
  UNION unionname
    union-fields
  ENDS
structname ENDS
```

除了其每个字段都只有一个初始值之外，联合字段声明的规则与结构的规则相同。例如，Integer 联合对同一个数据声明了 3 种不同的大小属性，并将所有的字段都初始化为 0：

```text
Integei; UNION
  D DWORD 0
  W WORD 0
  B BYTE 0
Integer ENDS
```

#### 一致性

如果使用初始值，那么它们必须为相同的数值。假设 Integer 声明了 3 个不同的初始值：

```text
Integer UNION
  D DWORD 1
  W WORD 5
  B BYTE 8
Integer ENDS
```

同时还假设声明了一个 Integer 变量 mylnt 使用默认初始值：

```text
.data
mylnt Integer <>
```

结果发现，myInt.D、myInt.W 和 myInt.B 都等于 1。字段 W 和 B 中声明的初始值会被汇编器忽略。

#### 结构包含联合

在结构声明中使用联合的名称，就可以使联合嵌套在这个结构中。方法如同下面在 Fileinfo 结构中声明 FilelD 字段一样：

```text
Fileinfo STRUCT
  FilelD Integer <>
  FileName BYTE 64 DUP(?)
Fileinfo ENDS
```

还可以直接在结构中定义联合，方法如同下面定义 FilelD 字段一样：

```text
Fileinfo STRUCT
  UNION FilelD
    D DWORD ?
    W WORD ?
    B BYTE ?
  ENDS
  FileName BYTE 64 DUP(?)
Fileinfo ENDS
```

#### 声明和使用联合变量

联合变量的声明和初始化方法与结构变量相同，只除了一个重要的差异：不允许初始值多于一个。下面是 Integer 类型变量的例子：

```text
val1 Integer <12345678h>
val2 Integer <100h>
val3 Integer <>
```

在可执行指令中使用联合变量时，必须给出字段的一个名称。下面的例子把寄存器的值赋给了 Integer 联合字段。注意其可以使用不同操作数大小的灵活性：

```text
mov val3.B, al
mov val3.W, ax
mov val3.D, eax
```

联合还可以包含结构。有些 MS-Windows 控制台输入函数会使用如下 INPUT\_RECORD 结构，它包含了一个名为 Event 的联合，这个联合对几个预定义的结构类型进行选择。EventType 字段表示联合中出现的是哪种 record。每一种结构都有不同的布局和大小，但是一次只能使用一种：

```text
INPUT_RECORD STRUCT
  EventType WORD ?
  ALIGN DWORD
  UNION Event
    KEY_EVENT_RECORD <>
    MOUSE_EVENT_RECORD <>
    WINDOW_BUFFER_SIZE_RECORD <>
    MENU_EVENT_RECORD <>
    FOCUS_EVENT_RECORD <>
  ENDS
INPUT_RECORD ENDS
```

#### Win32 API

在命名结构时，常常使用单词 RECORD。KEY\_EVENT\_RECORD 结构的定义如下所示：

```text
KEY_EVENT_RECORD STRUCT
  bKeyDown DWORD ?
  wRepeatCount WORD ?
  wVirtualKeyCode WORD ? wVirtualScanCode WORD ?
  UNION uChar
    UnicodeChar WORD ?
    AsciiChar BYTE ?
  ENDS
  dwControlKeyState DWORD ?
KEY_EVENT_RECORD ENDS
```

## 10.7 宏过程（macro procedure）简述

宏过程 \(macro procedure\) 是一个命名的汇编语句块。一旦定义好了，它就可以在程序中多次被调用。在调用宏过程时，其代码的副本将被直接插入到程序中该宏被调用的位置。

这种自动插入代码也被称为内联展开\(inline expansion\)。尽管从技术上来说没有 CALL 指令，但是按照惯例仍然说调用 \(calling\) 宏过程。

> 提示：Microsoft 汇编程序手册中的术语宏过程是指无返回值的宏。还有一种宏函数 \(macro function\) 则有返回值。在程序员中，单词宏 \(macro\) 通常被理解为宏过程。在下面的讲解中将使用宏这个简短的称呼。

位置宏定义一般出现在程序源代码开始的位置，或者是放在独立文件中，再用 INCLUDE 伪指令复制到程序里。

宏在汇编器预处理 \(preprocessing\) 阶段进行扩展。在这个阶段中，预处理程序读取宏定义并扫描程序剩余的源代码。每到宏被调用的位置，汇编器就将宏的源代码复制插入到程序中。

汇编器在调用宏之前，必须先找到宏定义。如果程序定义了宏但却没有调用它，那么在编译好的程序中不会出现宏代码。

在下例中，宏 PrintX 调用了 Irvine32 链接库的 WriteChar 过程。这个定义通常会被放置在数据段之前：

```text
PrintX MACRO
  mov al,'X'
  call WriteChar
ENDM
```

接着，在代码段中调用这个宏：

```text
.code
PrintX
```

当预处理程序扫描这个程序并发现对 PrintX 的调用后，它就用如下语句替换宏调用：

```text
mov al, 'X'
call WriteChar
```

这里发生的是文本替换。虽然宏有点不灵活，但后面很快就会展示如何向宏传递实参，使它们变得更有用。

## 10.8 MACRO和ENDM伪指令：定义宏

定义一个宏使用的是 MACRO 和 ENDM 伪指令，其语法如下所示：

```text
macroname MACRO parameter-1, parameter-2...
  statement-list
ENDM
```

关于缩进没有硬性规定，但是还是建议对 macroname 和 ENDM 之间的语句进行缩进。 同时，还希望在宏名上使用前缀 m，形成易识别的名称，如 mPutChar，mWriteString 和 mGotoxy。

除非宏被调用，否则 MACRO 和 ENDM 伪指令之间的语句不会被汇编。宏定义中还可以有多个形参，参数之间用逗号隔开。

#### 参数

宏形参 \(macro parameter\) 是需传递给调用者的文本实参的命名占位符。实参实际上可能是整数、变量名或其他值，但是预处理程序把它们都当做文本。

形参不包含类型信息，因此，预处理程序不会检查实参类型来看它们是否正确。如果发生类型不匹配，它将会在宏展开之后，被汇编器捕获。

#### mPutChar 示例

下面宏 mPutChar 接收一个名为 char 的输入形参，通过调用本教程链接库的 WriteChar 将其显示在控制台：

```text
mPutchar MACRO char
    push eax
    mov al,char
    call WriteChar
    pop eax
ENDM
```

## 10.9 宏的调用简述

调用宏的方法是把宏名插入到程序中，后面可能跟有宏的实参。宏调用语法如下：

```text
macroname argument-1, argument-2,
```

Macroname 必须是源代码中在此之前被定义宏的名称。每个实参都是文本值，用以替换宏的一个形参。实参的顺序要与形参一致，但是两者的数量不须相同。如果传递的实参数太 多，则汇编器会发出警告。如果传递给宏的实参数太少，则未填充的形参保持为空。

#### 调用 mPutChar

上一节《[MACRO和ENDM伪指令](http://c.biancheng.net/view/3714.html)》中定义了宏 mPutChar。调用 mPutChar 时，可以传递任何字符或 ASCII 码。下面的语句调用了 mPutChar，并向其传递了字母 “A”：

```text
mPutchar 'A'
```

汇编器的预处理程序将这条语句展开为下述代码，以列表文件的形式展开如下：

```text
1 push eax
1 mov al,'A'
1 call WriteChar
1 pop eax
```

左侧的 1 表示宏展开的层次，如果在宏的内部又调用了其他的宏，那么该值将会增加。下面的循环显示了字母表中前 20 个字母：

```text
  mov al,'A'
  mov ecx,20
L1:
  mPutchar al      ;宏调用
  inc al
  loop L1
```

该循环由预处理程序在下面的代码中展开（源列表文件中可见），其中，宏调用在其展开的前面：

```text
  mov al,'A'
  mov ecx,20
L1:
  mPutchar al  ;调用宏
  1 push eax
  1 mov al,al
  1 call WriteChar
  1 pop eax
  inc al
  loop L1
```

> 提示：与过程相比，宏执行起来更快，其原因是过程的 CALL 和 RET 指令需要额外的开销。但是，使用宏也有缺点：重复使用大型宏会增加程序的大小，因为，每次调用宏都会在程序中插入宏代码的一个新副本。

#### 调试宏

调试使用了宏的程序相当具有挑战性。程序汇编之后，检查其列表文件（扩展名为 .LST\) 以确保每个宏都按照程序员的要求展开。然后，在Visual Studio 调试器中启动该程序，在调试窗口点击右键，从弹出菜单中选择Go to Disassemblyo每个宏调用的后面都紧 跟其生成代码。示例如下：

```text
mWriteAt 15,10,"Hi there"
    push edx
    mov dh, 0Ah
    mov dl, 0Fh
    call _Gotoxy@0 (401551h)
    pop edx
    push edx
    mov edx,offset ??0000 (405004h)
    call _WriteString@0 (401D64h)
pop edx
```

由于 Irvine32 链接库使用的是 STDCALL 调用规范，因此函数名用下划线 \(\_\) 开始。

## 10.10 宏的特性

通过《[宏简述](http://c.biancheng.net/view/3713.html)》一节的学习，我们已经对宏有了一定的了解，下面来介绍一下宏的一些特性。

### 1\) 规定形参

利用 REQ 限定符，可以指定必需的宏形参。如果被调用的宏没有实参与规定形参相匹配，那么汇编器将显示出错消息。如果一个宏有多个规定形参，则每个形参都要使用 REQ 限定符。

下面是宏 mPutChar，形参 char 是必需的：

```text
mPutchar MACRO char:REQ
    push eax
    mov al,char
    call WriteChar
    pop eax
ENDM
```

### 2\) 宏注释

宏定义中的注释行一般都出现在每次宏展开的时候。如果希望忽略宏展开时的注释，就在它们的前面添加双分号 \(;;\)。示例如下：

```text
mPutchar MACRO char:REQ
    push eax             ;; 提示：char 必须包含 8 个比特
    mov al, char
    call WriteChar
    pop eax
ENDM
```

### 3\) ECHO 伪指令

在程序汇编时，ECHO 伪指令写一个字符串到标准输出。下面的 mPutChar 在汇编时会显示消息“Expanding the mPutChar macro” :

```text
mPutchar MACRO char:REQ
    ECHO Expanding the mPutchar macro
    push eax
    mov al,char
    call WriteChar
    pop eax
ENDM
```

Visual Studio 2012 的控制台窗口不会捕捉 ECHO 伪指令的输出，除非在编写程序时将其设置为生成详细输出。设置方法如下：从 Tool 菜单选择 Options，选择 Projects and Solutions，选择 Build and Run，再从 MSBuild project build output verbosity 下拉列表中选择 Detailed。或者打开一个命令提示符并汇编程序。

首先，执行如下命令，调整 Visual Studio 当前版本的路径：

```text
"C:\Program Files\Microsoft Visual Studio 11.0\VC\bin\vcvars32"
```

然后，键入如下指令，其中 filename.asm 是程序的源代码文件名：

```text
ml.exe /c /I "c:\Irvine" filename.asm
```

### 4\) LOCAL 伪指令

宏定义中常常包含了标号，并会在其代码中对这些标号进行自引用。例如，下面的宏 makeString 声明了一个变量 string，且将其初始化为字符数组：

```text
makestring MACRO text
    .data
    string BYTE text,0
ENDM
```

假设两次调用宏：

```text
makeString "Hello"
makeString "Goodbye"
```

由于汇编器不允许两个标号有相同的名字，因此结果出现错误：

```text
makeString "Hello"
.data
string BYTE "Hello",0
makeString "Goodbye"
.data
string BYTE "Goodbye",0   ;错误！
```

#### 使用 LOCAL

为了避免标号重命名带来的问题，可以对一个宏定义内的标号使用 LOCAL 伪指令。若标号被标记为 LOCAL，那么每次进行宏展开时，预处理程序就把标号名转换为唯一的标识符。下面是使用了 LOCAL 的宏 makeString：

```text
makeString MACRO text
    LOCAL string
    .data
    string BYTE text,0
ENDM
```

假设和前面一样，也是两次调用宏，预处理程序生成的代码会将每个string替换成唯一 的标识符：

```text
makeString "Hello"
.data
??0000 BYTE "Hello",0
 makeString "Goodbye"
.data
??0001 BYTE "Goodbye",0
```

汇编器生成的标号名使用了 ??nnnn 的形式，其中 nnnn 是具有唯一性的整数。local 伪指令还可以用于宏内的代码标号。

### 5\) 包含代码和数据的宏

宏通常既包含代码又包含数据。例如，下面的宏mWrite在控制台显示文本字符串：

```text
mWrite MACRO text
    LOCAL string            ;;local号
    .data                   ;;定义字符串
    string BYTE text,0
    .code
    push edx
    mov edx, OFFSET string
    call WriteString
    pop edx
ENDM
```

下面的语句两次调用宏，并向其传递不同的字符串文本：

```text
mWrite "Please enter your first name"
mWrite "Please enter your last name"
```

汇编器对这两条语句进行展开时，每个字符串都被赋予了唯一的标号，且MOV指令也 作了相应的调整：

```text
mWrite "Please enter your first name"
.data
??0000 BYTE "Please enter your first name",0
.code
push edx
mov edx, OFFSET ??0000
call WriteString
pop edx
 mWrite "Please enter your last name"
.data
??0001 BYTE "Please enter your last name", 0
.code
push edx
mov edx, OFFSET ??0001
call Writestring
pop edx
```

### 6\) 宏嵌套

被其他宏调用的宏称为被嵌套的宏 \(nested macro\)。当汇编器的预处理程序遇到对被嵌套宏的调用时，它会就地展开该宏。传递给主调宏的形参也将直接传递给它的被嵌套宏。

提示：使用模块方法创建宏。保持它们的简短性，以便将它们组合到更复杂的宏内。这样有助于减少程序中的复制代码量。

【示例】下面的宏 mWritein 写一个字符串文本到控制台，并添加换行符。它调用宏 mWrite 和 Crlf 过程：

```text
mWriteln MACRO text    mWrite text    call CrlfENDM
```

形参 text 被直接传递给 mWrite。假设用下述语句调用 mWriteln：

```text
mWriteln "My Sample Macro Program"
```

在结果代码展开，语句旁边的嵌套级数（2）表示被调用的是一个嵌套宏：

```text
mWriteln "My Sample Macro Program"
.data
??0002 BYTE "My Sample Macro Program",0
.code
push edx
mov  edx,OFFSET ??0002
call WriteString
pop  edx
call Crlf
```

## 10.11 Macro宏库详解

本教程提供的示例程序包含了一个小而实用的 32 位链接库，只需要在程序的 INCLUDE 后面添加如下代码行就可以使用该链接库：

```text
INCLUDE Macros.inc
```

有些宏封装在了 Irvine32 链接库的过程中，这样传递参数就更加容易。其他宏则提供新的功能。下表详细介绍了每个宏。

| 宏名 | 形式参数 | 说明 |
| :--- | :--- | :--- |
| mDump | varName, useLabel | 用变量名和默认属性显示一个变量 |
| mDumpMem | abbress, itemCount, componentsize | 显示内存区域 |
| mGotoxy | X,Y | 将光标位置设置在控制台窗口缓冲区 |
| mReadString | varName | 从键盘读取一个字符串 |
| mShow | itsName, format | 用各种格式显示一个变量或寄存器 |
| mShowRegister | itsName, regValue | 显示32位寄存器名，并用十六进制显示其内容 |
| mWrite | text | 向控制台窗口输出一个字符串文本 |
| mWriteSpace | count | 向控制台窗口输出一个或多个空格 |
| mWriteString | buffer | 向控制台窗口输岀一个字符串变量的内容 |

### 1\) mDumpMem

宏 mDumpMem 在控制台窗口显示一个内存区域。向其传递的第一个实参为包含待显示内存偏移量的常数、寄存器或者变量，第二个实参应为待显示内存中存储对象的数量，第三个实参为每个存储对象的大小。

宏在调用mDumpMem库过程时，分别将这三个实参分配给 ESI、ECX 和 EBX。现假设有一数据定义如下：

```text
.data
array DWORD 1000h, 2000h, 3000h, 4000h
```

下面的语句按照默认属性显示数组:

```text
mDumpMem OFFSET array, LENGTHOF array, TYPE array
```

输出为：

```text
Dump of offset 00405004
------------------------------
00001000 00002000 00003000 00004000
```

下面的语句则将同一个数组显示为字节序列：

```text
mDumpMem OFFSET array, SIZEOF array, TYPE BYTE
```

输出为：

```text
Dump of offset 00405004
------------------------------
00 10 00 00 00 20 00 00 00 30 00 00 00 40 00 00
```

下面的代码把三个数值压入堆栈，并设置好 EBX、ECX 和 ESI，然后调用 mDumpMem 显示堆栈：

```text
mov eax,0AAAAAAAAh
push eax
mov eax,0BBBBBBBBh
push eax
mov eax,OCCCCCCCCh
push eax
mov ebx,1
mov ecx,2
mov esi,3
mDumpMem esp, 8, TYPE DWORD
```

显示出来的结果堆栈区域表明，宏已经先把 EBX、ECX 和 ESI 压入了堆栈。这些数值之后是在调用 mDumpMem 之前入栈的 3 个整数：

```text
Dump of offset 0012FFAC
------------------------------
00000003 00000002 00000001 CCCCCCCC BBBBBBBB AAAAAAAA 7C816D4F
0000001A
```

实现宏代码清单如下：

```text
mDumpMem MACRO address:REQ, itemCount:REQ, componentsize:REQ
;用 DumpMem 过程显示一个内存区域。
;接收：内存偏移量、显示对象的数量，以及每个存储对象的大小。
;避免用 EBX、ECX 和 ESI 传递实参。
    push ebx
    push ecx
    push esi
    mov esi, address
    mov ecx, itemCount
    mov ebx, componentSize
    call DumpMem
    pop esi
    pop ecx
    pop ebx
ENDM
```

### 2\) mDump

宏 mDump 用十六进制显示一个变量的地址和内容。传递给它的参数有：变量名和（可选的）一个字符以表明在该变量之后应显示的标号。显示格式自动与变量的大小属性（BYTE、WORD 或 DWORD）匹配。

下面的例子展示了对 mDump 的两次调用:：

```text
.data
diskSize DWORD 12345h
.code
mDump diskSize      ; no label
mDump diskSize,Y    ; show label
```

代码执行后，产生的输出如下所示：

```text
Dump of cffset 00405000
------------------------
00012345
Variable name: diskSize
Dump of offset 00405000
------------------------
00012345
```

下面是宏 mDump 的代码清单，它反过来又调用了 mDumpMem。代码用一个新的伪指令 IFNE （若不为空）来发现主调者是否向第二个形参传递了实参：

```text
mDumpMem MACRO address:REQ, itemCount:REQ, componentsize:REQ
;用 DumpMem 过程显示一个内存区域。
;接收：内存偏移量、显示对象的数量，以及每个存储对象的大小。
;避免用 EBX、ECX 和 ESI 传递实参。
    push ebx
    push ecx
    push esi
    mov esi, address
    mov ecx, itemCount
    mov ebx, componentSize
    call DumpMem
    pop esi
    pop ecx
    pop ebx
ENDM
```

&varName 中的符号 & 是替换操作符，它允许将 varName 形参的值插入到字符串文本中。

### 3\) mGotoxy

宏 mGotoxy 把光标定位在控制台窗口缓冲区内指定的行列上。可以向其传递 8 位立即数、内存操作数和寄存器值：

```text
mGotoxy  10,20         ;立即数
mGotoxy  row, col       ;内存操作数
mGotoxy  ch,cl         ;寄存器值
```

实现 下面是宏的源代码清单：

```text
;-----------------------------
mGotoxy MACRO X:REQ, Y:REQ
;设置光标在控制台窗口的位置。
;接收：X和Y坐标（类型为BYTE）。避免用DH和DL传递实参。
;-----------------------------
    push edx
    mov dh,Y
    mov dl,X
    call Gotoxy
    pop edx
ENDM
```

若宏的实参是寄存器，它们有时可能会与宏内使用的寄存器发生冲突。比如，调用 mGotoxy 时用了 DH 和 DL，那么就不会生成正确的代码。为了说明原因，现在来查看上述参数被替换后展开的代码：

```text
push edx
mov dhr dl   ;;行
mov dl,dh   ;;列
call Gotoxy
pop edx
```

假设 DL 传递的是 Y 值，DH 传递的是 X 值，代码行 2 会在代码行 3 有机会把列值复制 到DL之前就替换了 DH的原值。

> 提示：只要有可能，宏定义应该用注释说明哪些寄存器不能用作实参。

### 4\) mReadString

宏 mReadSrting 从键盘读取一个字符串，并将其存储在缓冲区。在这个宏的内部封装了一个对 ReadString 库过程的调用。需向其传递缓冲区名：

```text
.data
firstName BYTE 30 DUP(?)
.code
mReadString firstName
```

下面是宏的源代码：

```text
;-----------------------------------------   
mReadString MACRO varName:REQ
;从标准输入读到缓冲区。
;接收:缓冲区名。避免用 ECX 和 EDX 传递实参。
;-----------------------------------------   
    push ecx
    push edx
    mov edx,OFFSET varName
    mov ecx,SIZEOF varName
    call Readstring
    pop edx
    pop ecx
ENDM
```

### 5\) mShow

宏 mShow 按照主调者选择的格式显示任何寄存器或变量的名字和内容。传递给它的是寄存器名，其后可选择性地加上一个字母序列，以表明期望的格式。字母选择如下：H = 十六进制，D = 无符号十进制，I 二有符号十进制，B 二二进制，N = 换行。

可以组合多种输出格式，还可以指定多个换行。默认格式为“HIN”。mShow 是一种有用的辅助调试工具，经常被 DumpRegs 库过程使用。可以把mShow当作调试工具，显示重要寄存器或变量的值。

【示例】下面的语句将 AX 寄存器的值显示为十六进制、有符号十进制、无符号十进制和二进制：

```text
mov ax, 4096
mShow AX    ;默认选项：HTN
mShow AX,DBN  ;无符号十进制，二进制，换行
```

输出如下：

```text
AX = 1000h +4096d
AX = 4096d 0001 0000 0000 0000b
```

【示例】下面的语句在同一行上，用无符号十进制格式显示 AX, BX, CX 和 DX：

```text
;插入测试数值，显示4个寄存器：
mov ax, 1
mov bx, 2
mov cx, 3
mov dxz 4
mShow AX, D
mShow BX, D
mShow CX,D
mShow DX, DN
```

相应输出如下：

```c
AX = Id BX = 2d CX = 3d DX = 4d
```

【示例】下面的代码调用 mShow，用无符号十进制格式显示 mydword 的内容，并换行：

```text
.data
mydword. DWORD ?
.code
mS how mydword,DN
```

实现 mShow的实现代码太长不便在这里给岀，不过可以在本书安装文件夹（C ： \Irvine）内的Macros.inc文件中找到完整代码。在编写mShow时，需要注意在寄存器被宏 自身的内部语句修改之前显示其当前值。

### 6\) mShowRegister

宏 mShowRegister 显示单个 32 位寄存器的名称，并用十六进制格式显示其内容。传递给它的是希望被显示的寄存器名，其后紧跟寄存器本身。下面的宏调用指定了被显示的名称为 EBX：

```text
mShowRegister EBX, ebx
```

产生的输出如下：

```text
EBX=7FFD9000
```

下面的调用使用尖括号把标号括起来，其原因是标号内有一个空格：

```text
mShowRegister <Stack Pointer>, esp
```

产生输出如下：

```text
Stack Pointer=0012FFC0
```

实现宏的源代码如下：

```text
;------------------------------------
mShowRegister MACRO regName, regValue
LOCAL tempStr
;显示寄存器名和内容。
;接收：寄存器名，寄存器值
;------------------------------------
    .data
    tempStr BYTE " &regName=",0
    .code
    push eax
    ;显示寄存器名
    push edx
    mov edx,OFFSET tempStr
    call WriteString
    pop edx
    ;显示寄存器内容
    mov eax,regValue
    call WriteHex
    pop eax
ENDM
```

### 7\) mWriteSpace

宏 mWriteSpace 向控制台窗口输出一个或多个空格。可以选择性地向其传递一个整数形参，以指定空格数 \( 默认为一个 \)。例如，下面的语句写了 5 个空格：

```text
mWriteSpace 5
```

实现mWriteSpace的源代码如下：

```text
;-------------------------------------------
mWriteSpace MACRO count:=<1>
;向控制台窗口输出一个或多个空格。
;接收：一个整数以指定空格数。
;默认个数为l。
;-------------------------------------------
LOCAL spaces
.data
spaces BYTE count DUP('    '),0
.code
    push edx
    mov edx,OFFSET spaces
    call WriteString
    pop edx
ENDM
```

### 8\) mWriteString

宏 mWriteSrting 向控制台窗口输出一个字符串变量的内容。从宏的内部来看，它通过在同一语句行上传递字符串变量名简化了对 WriteString的调用。例如：

```text
.data
str1 BYTE "Please enter your name: ", 0
.code
mWriteString str1
```

mWriteString 的实现如下，它将 EDX 保存到堆栈，然后把字符串偏移量赋给 EDX，在过程调用后，再从堆栈恢复 EDX 的值：

```text
;------------------------------
mWriteString MACRO buffer:REQ
;向标准输出写一个字符串变量。
;接收：字符串变量名。
;------------------------------
    push edx
    mov    edx,OFFSET buffer
    call WriteString
    pop edx
ENDM
```

## 10.12 实例：封装器

现在创建一个简短的程序 Wraps.asm 来展示之前已介绍的作为过程封装器的宏。由于每个宏都隐含了大量繁琐的参数传递，因此程序出奇得紧凑。假设这里所有的宏当前都在 Macros.inc 文件内：

```text
; 过程封装器宏        (Wraps.asm)
; 本程序演示宏作为库过程的封装器。
; 内容: mGotoxy, mWrite, mWriteString, mReadString, 和 mDumpMem.
INCLUDE Irvine32.inc
INCLUDE Macros.inc            ; 宏定义

.data
array DWORD 1,2,3,4,5,6,7,8
firstName BYTE 31 DUP(?)
lastName  BYTE 31 DUP(?)

.code
main PROC
    mGotoxy 0,0
    mWrite <"Sample Macro Program",0dh,0ah>

; 输入用户名
    mGotoxy 0,5
    mWrite "Please enter your first name: "
    mReadString firstName
    call Crlf

    mWrite "Please enter your last name: "
    mReadString lastName
    call Crlf

; 显示用户名
    mWrite "Your name is "
    mWriteString firstName
    mWriteSpace
    mWriteString lastName
    call Crlf

; 显示整数数组
    mDumpMem OFFSET array,LENGTHOF array, TYPE array
    exit
main ENDP
END main
```

程序输出 程序输出的示例如下：

![img](http://c.biancheng.net/uploads/allimg/190521/4-1Z521152442152.gif)

## 10.13 条件汇编伪指令简述

很多不同的条件汇编伪指令都可以和宏一起使用，这使得宏更加灵活。条件汇编伪指令常用语法如下所示：

```text
IF condition
  statements
[ELSE
  statements]
ENDIF
```

下表列出了更多常用的条件汇编伪指令。若说明为该伪指令允许汇编，就意味着所有的后续语句都将被汇编，直到遇到下一个 ELSE 或 ENDIF 伪指令。必须强调的是，表中列出的伪指令是在汇编时而不是运行时计算。

| 伪指令 | 说明 |
| :--- | :--- |
| IF expression | 若 expression 为真（非零）则允许汇编。可能的关系运算符为 LT、GT、EQ、NE、LE 和 GE |
| IFB | 若 argument 为空则允许汇编。实参名必须用尖括号（＜＞）括起来 |
| IFNB | 若 argument 为非空则允许汇编。实参名必须用尖括号（＜＞）括起来 |
| IFIDN, | 若两个实参相等（相同）则允许汇编。采用区分大小写的比较 |
| IFIDNI, | 若两个实参相等（相同）则允许汇编。采用不区分大小写的比较 |
| IFDIF, | 若两个实参不相等则允许汇编。采用区分大小写的比较 |
| IFDIFI, | 若两个实参不相等则允许汇编。采用不区分大小写的比较 |
| IFDIF name | 若 name 已定义则允许汇编 |
| IFNDEF name | 若 name 还未定义则允许汇编 |
| ENDIF | 结束用一个条件汇编伪指令开始的代码块 |
| ELSE | 若条件为真，则终止汇编之前的语句。若条件为假，ELSE 汇编语句直到遇到下一个 ENDIF |
| ELSEIF expression | 若之前条件伪指令指定的条件为假，而当前表达式为真，则汇编全部语句直到出现 ENDIF |
| EXITM | 立即退出宏，阻止所有后续宏语句的展开 |

## 10.14 IFB和IFNB伪指令：检查缺失的参数

宏能够检查其参数是否为空。通常，宏若接收到空参数，则预处理程序在进行宏展开时会导致出现无效指令。例如，如果调用宏 mWriteString 却又不传递实参，那么宏展开在把字符串偏移量传递给 EDX 时，就会出现无效指令。

汇编器生成的如下语句检测出缺失的操作数，并产生了一个错误消息：

```text
mWriteString
push edx
mov edx,OFFSET
Macro2.asm(18) : error A2081: missing operand after unary operator
call WriteString
pop edx
```

为了防止由于操作数缺失而导致的错误，可以使用 IFB \(if blank\) 伪指令，若宏实参为空，则该伪指令返回值为真。

还可以使用 IFNB \(if not blank\) 运算符，若宏实参不为空，则其返回值为真。现在编写 mWriteString 的另一个版本，使其可以在汇编时显示错误消息：

```text
mWriteString MACRO string
    IFB <string>
        ECHO -------------------------------------------
        ECHO * Error: parameter missing in mWriteString
        ECHO *  (no code generated)
        ECHO -------------------------------------------
    EXITM
    ENDIF
    push edx
    mov edx,OFFSET string
    call WriteString
    pop edx
ENDM
```

程序汇编时，ECHO 伪指令向控制台写一个消息。EXITM 伪指令告诉预处理程序退岀宏，不再展开更多宏语句。汇编的程序有缺失参数时，其屏幕输出如下所示：

![img](http://c.biancheng.net/uploads/allimg/190521/4-1Z521161326492.gif)

## 10.15 宏默认值设定及布尔表达式简述

宏可以有默认参数初始值。如果调用宏出现了宏参数缺失，那么就可以使用默认参数。其语法如下：

```text
paramname := < argument >
```

运算符前后的空格是可选的。比如，宏 mWriteln 提供含有一个空格的字符串作为其默认参数。如果对其进行无参数调用，它仍然会打印一个空格并换行：

```text
mWriteln MACRO text:=<" ">
  mWrite text
  call Crlf
ENDM
```

若把空字符串 \(" "\) 作为默认参数，那么汇编器会产生错误，因此必须在引号之间至少插入一个空格。

### 布尔表达式

汇编器允许在包含 IF 和其他条件伪指令的常量布尔表达式中使用下列关系运算符：

| LT | 小于 |
| :--- | :--- |
| GT | 大于 |
| EQ | 等于 |
| NE | 不等于 |
| LE | 小于等于 |
| GE | 大于等于 |

## 10.16 IF、ELSE和DENDIF伪指令

IF 伪指令的后面必须跟一个常量布尔表达式。该表达式可以包含整数常量、符号常量或者常量宏实参，但不能包含寄存器或变量名。仅适用于 IF 和 ENDIF 的语法格式如下：

```text
IF expression
  statement-list
ENDIF
```

另一种格式则适用于 IF、ELSE 和 ENDIF：

```text
IF expression
  statement-list
ELSE
  statement-list
ENDIF
```

【示例】宏 mGotoxyConst 利用 LT 和 GT 运算符对传递给宏的参数进行范围检查。实参 X 和 Y 必须为常数。还有一个常数符号 ERRS 对发现的错误进行计数。根据 X 的值，可以将 ERRS 设置为 1。根据 Y 的值，可以将 ERRS 加 1。最后，如果 ERRS 大于零，EXITM 伪指令退岀宏：

```text
;-------------------------------------
mGotoxyConst MACRO X:REQ, Y:REQ
;
;将光标位置设置在 X 列 Y 行。
;要求 X 和 Y 的坐标为常量表达式
;其范围为 0 ≤ X < 80, 0 ≤ Y < 25。
;-------------------------------------
    LOCAL ERRS               ;;门局部常量
    ERRS = 0
    IF (X LT 0) OR (X GT 79)
        ECHO Warning: First argument to mGotoxy (X) is out of range.
        ECHO ******************************************************
        ERRS = 1
    ENDIF
    IF (Y LT 0) OR (Y GT 24)
        ECHO Warning: Second argument to mGotoxy (Y) is out of range.
        ECHO ******************************************************
        ERRS = ERRS + 1
    ENDIF
    IF ERRS GT 0                 ;;若发现错误，
        EXITM                    ;;退出宏
    ENCIF
    push edx
    mov dh,Y
    mov dl,X
    call Gotoxy
    pop edx
ENDM
```

## 10.17 IFIDN和IFIDNI伪指令：对两个参数进行比较

IFIDNI 伪指令在两个符号（包括宏参数名）之间进行不区分大小写的比较，如果它们相等，则返回真。IFIDN 伪指令执行的是区分大小写的比较。

如果想要确认宏主调者使用的寄存器参数不会与宏内使用的寄存器发生冲突，那么可以使用这两个伪指令中的前者。IFIDNI 的语法如下：

```text
IFIDNI <symbol>, <symbol>
  statements
ENDIF
```

IFIDN 的语法与之相同。例如下面的宏 mReadBuf，其第二个参数不能用 EDX，因为当 buffer 的偏移量被送入 EDX 时，原来的值就会被覆盖。

在如下修改过的宏代码中，如果这个条件不满足，就会显示一条警告消息：

```text
;-------------------------------------
mReadBuf MACRO bufferPtr, maxChars
;将键盘输入读到缓冲区。
;接收：缓冲区偏移量，最多可输入字符的数量。第二个参数不能用 edx 或EDX。
;-------------------------------------
    IFIDNI <maxChars>,<EDX>
        ECHO Warning: Second argument to mReadBuf cannot be EDX
        ECHO **************************************************
        EXITM
    ENDIF
    push ecx
    push edx
    mov edx,bufferPtr
    mov ecx,maxChars
    call Readstring
    pop edx
    pop ecx
ENDM
```

下面的语句将会导致宏产生警告消息，因为 EDX 是其第二个参数：

```text
mReadBuf OFFSET buffer,edx
```

## 10.18 实例：矩阵行求和

前面《[二维数组简介](http://c.biancheng.net/view/3692.html)》一节中已经展示了如何计算字节矩阵中单个行的总和。尽管这个解决方案有些冗长，现在还是要看看能否用宏来简化任务。首先，我们来回顾一下 calc\_row\_sum 过程：

```text
;------------------------------------------------------------
; calc_row_sum
; 计算字节矩阵中一行的和数
; 接收: EBX = 表偏移量, EAX = 行索引
;       ECX = 按字节计的行大小
; 返回:  EAX 为和数
;------------------------------------------------------------
calc_row_sum PROC uses ebx ecx edx esi

    mul     ecx                             ; 行索引 * 行大小
    add     ebx,eax                         ; 行偏移量
    mov     eax,0                           ; 累加器
    mov     esi,0                           ; 列索引

L1:    movzx edx,BYTE PTR[ebx + esi]        ; 取一个字节
    add     eax,edx                         ; 与累加器相加
    inc     esi                             ; 行中的下一个字节
    loop L1

    ret
calc_row_sum ENDP
```

从把 PROC 改为 MACRO 开始，删除 RET 指令，把 ENDP 改为 ENDM。由于没有宏与 USES 伪指令功能相当，因此插入 PUSH 和 POP 指令：

```text
mCalc_row_sum MACRO
    push ebx              ;保存被修改的寄存器
    push ecx
    push esi
    mul ecx               ;行索引x行大小
    add ebx,eax           ;行偏移量
    mov eax,0             ;累加器
    mov esi,0             ;列索引
L1: movzx edx,BYTE PTR[ebx + esi]
    add eax, edx          ;取一个字节
    inc esi               ;与累加器相加
    loop L1               ;行内的下一个字节
    pop esi               ;恢复被修改的寄存器
    pop ecx
    pop ebx
ENDM
```

接着，用宏参数代替寄存器参数，并对宏内寄存器进行初始化：

```text
mCalc_row_sum MACRO index, arrayoffset, rowSize
    push ebx                       ;保存被修改的寄存器
    push ecx
    push esi
;设置需要的寄存器
    mov eax,index
    mov ebx,arrayOffset
    mov ecx,rowSize
    mul ecx                        ;行索引x行大小
    add ebx,eax                    ;行偏移量
    mov eax,0                      ;累加器
    mov esi,0                      ;列索引
L1: movzx edx, BYTE PTR[ebx + esi] ;取一个字节
    add eax, edx                   ;与累力口器相力口
    inc esi                        ;行内的下一个字节
    loop L1
    pop esi                        ;恢复被修改的寄存器
    pop ecx
    pop ebx
ENDM
```

然后，添加一个参数 eltType 指定数组类型 \(BYTE、WORD 或 DWORD\)：

```text
mCalc_row_sum MACRO index, arrayOffset, rowSize, eltType
```

复制到 ECX 的参数 rowSize 现在表示的是每行的字节数。如果要用其作为循环计数器，那么它就必须转换为每行的元素 \(element\) 个数。

因此，若为 16 位数组，就将 ECX 除以 2 ;若为双字数组，就将 ECX 除以 4。实现上述操作的快捷方式为：eltType 除以 2，把商作为移位计数器，再将 ECX 右移：

```text
shr ecx,(TYPE eltType/2)     ; byte=0, word=1, dword=2
```

TYPE eltType 就成为 MOVZX 指令中基址-变址操作数的比例因子：

```text
movzx edx,eltType PTR[ebx + esi*(TYPE eltType)]
```

若 MOVZX 右操作数为双字，那么指令不会汇编。所以，当 eltType 为 DWORD 时，需要用 IFIDNI 运算符另外编写一条 MOV 指令：

```text
IFIDNI <eltType>,<DWORD>
  mov edx,eltType PTR[ebx + esi*(TYPE eltType)]
ELSE
  movzx edx, eltType PTR[ebx + esi*(TYPE eltType)]
ENDIF
```

最后必须结束宏，记住要把标号 L1 指定为 LOCAL：

```text
;-----------------------------------------------------
mCa1c_row_sum MACRO index, arrayOffset, rowSize, eltType
;计算二维数组中一行的和数。
;接收：行索引、数组偏移量、每行的字节数、数组类型 (BYTE、WORD、或 DWORD)。
;返回：EAX= 和数。
;-----------------------------------------------------
LOCAL L1
    push ebx                    ;保存被修改的寄存器
    push ecx
    push esi
;设置需要的寄存器
    mov eax, index
    mov ebx, arrayOffset
    mov ecx, rowSize
;计算行偏移量
    mul ecx                      ;行索引x行大小
    add ebx, eax                 ;行偏移量
;初始化循环计数器
    shr ecx,(TYPE eltType/2)     ;byte=0, word=1, dword=2
;初始化累加器和列索引
    mov eax,0                    ;累加器
    mov esi,0                    ;列索引
L1:
    IFIDNI <eltType>, <DWORD>
        mov edx,eltType PTR[ebx + esi*(TYPE eltType)]
    ELSE
        movzx edx,eltType PTR[ebx + esi*(TYPE eltType)]
    ENDIF
    add eax,edx                  ;与累加器相加
    inc esi
    loop L1
    pop esi                      ;恢复被修改的寄存器
    pop ecx
    pop ebx
ENDM
```

下面用字节数组、字数组和双字数组对宏进行示例调用：

```text
.data
tableB BYTE 10h, 20h, 30h, 40h, 50h
RowSizeB = ($ - tableB)
    BYTE 60h, 70h, 80h, 90h, 0A0h
    BYTE 0B0h, 0C0h, 0D0h, 0E0h, 0F0h

tableW WORD 10h, 20h, 30h, 40h, 50h
RowSizeW = ($ - tableW)
    WORD 60h, 70h, 80h, 90h, 0A0h
    WORD 0B0h, 0C0h, 0D0h, 0E0h, 0F0h

tableD DWORD 10h, 20h, 30h, 40h, 50h
RowSizeD = ($ - tableD)
    DWORD 60h, 70h, 80h, 90h, 0A0h
    DWORD 0B0h, 0C0h, 0D0h, 0E0h, 0F0h

index DWORD ?
.code
mCalc_row_sum index, OFFSET tableB, RowSizeB, BYTE
mCalc_row_sum index, OFFSET tableW, RowSizeW, WORD
mCalc_row_sum index, OFFSET tableD, RowSizeD, DWORD
```

## 10.19 替换（&）、文本（&lt;&gt;）、字符（!）、展开（%）运算符简述

下述四个汇编运算符使得宏更加灵活：

| & | 替换运算符 |
| :--- | :--- |
| &lt;&gt; | 文字文本运算符 |
| ! | 文字字符运算符 |
| % | 展开运算符 |

### 替换运算符（&）

替换运算符（&）解析对宏参数名的有歧义的引用。宏 mShowRegister 显示了一个 32 位寄存器的名称和十六进制的内容。示例调用如下：

```text
.code
mShowRegister ECX
```

下面是调用 mShowRegister 产生的示例输出：

```text
ECX=00000101
```

在宏内可以定义包含寄存器名的字符串变量：

```text
mShowRegister MACRO regName
.data
tempStr BYTE "regName=",0
```

但是预处理程序会认为 regName 是字符串文本的一部分，因此，不会将其替换为传递给宏的实参值。相反，如果添加了 & 运算符，它就会强制预处理程序在字符串文本中插入宏实参 \( 如 ECX\)。下面展示的是如何定义 tempStr：

```text
mShowRegister MACRO regName
.data
tempStr BYTE "&regName=",0
```

### 展开运算符（%）

展开运算符（%）展开文本宏并将常量表达式转换为文本形式。有几种方法实现该功能。若使用的是 TEXTEQU，% 运算符就计算常量表达式，再把结果转换为整数。

在下面的例子中，% 运算符计算表达式 \(5+count\)，并返回整数 15 \( 以文本形式 \)：

```text
count = 10
sumVal TEXTEQU %(5 + count)      ;="15"
```

如果宏请求的实参是整数常量，% 运算符就能使程序具有传递一个整数表达式的灵活性。计算这个表达式得到结果值，然后将这个值传递给宏。例如，调用 mGotoxyConst 时，计算表达式的结果分别为 50 和 7：

```text
mGotoxyConst %(5 * 10), %(3 + 4)
```

预处理程序将产生如下语句：

```text
push edx
mov dh,7
mov dl,50
call Gotoxy
pop edx
```

#### % 在一行的首位

当展开运算符 \(%\) 是一行源代码的第一个字符时，它指示预处理程序展开该行上的所有文本宏和宏函数。比如，假设想在汇编时将数组大小显示在屏幕上。下面的尝试不会产生期望的结果：

```text
.data
array DWORD 1,2,3,4,5,6,7,8
.code
ECHO The array contains (SIZEOF array) bytes
ECHO The array contains %(SIZEOF array) bytes
```

屏幕输出没什么用：

```text
The array contains (SIZEOF array) bytes
The array contains %(SIZEOF array) bytes
```

反之，如果用 TEXTEQU 编写包含 \(SIZEOF array\) 的文本宏，那么该宏就可以展开为之后的代码行：

```text
TempStr TEXTEQU %(SIZEOF array)
%  ECHO The array contains TempStr bytes
```

产生的输出如下所示：

```text
The array contains 32 bytes
```

#### 显示行号

下面的宏 Mul32 将它前两个实参相乘，乘积由第三个实参返回。其形参可以是寄存器、内存操作数和立即数 \( 乘积除外 \)：

```text
Mul32 MACRO op1, op2, product
    IFIDNI <op2>,<EAX>%
        LINENUM TEXTEQU %(@LINE)
        ECHO ----------------------------------------------
%       ECHO * Error on line LINENUM: EAX cannot be the second
        ECHO * argument when invoking the MUL32 macro.
        ECHO ----------------------------------------------
    EXITM
    ENDIF
    push eax
    mov eax,op1
    mul op2
    mov product,eax
    pop eax
ENDM
```

Mul32 要检查的一个重要要求是：EAX 不能作为第二个实参。这个宏有趣的地方是，它显示的是其调用者的行号，这样更加易于追踪并解决问题。首先定义文本宏 LINENUM，它引用的 @LINE 是一个预先定义的汇编运算符，其功能为返回当前源代码行的编号：

```text
LINENUM TEXTEQU % ((@LINE)
```

接着，在含有 ECHO 语句的代码行第一列上的展开运算符 \(%\) 使得 LINENUM 被展开：

```text
%  ECHO * Error on line LINENUM: EAX cannot be the second
```

假设如下宏调用发生在程序的 40 行：

```text
MUL32 val1, eax,val3
```

那么，汇编时将显示如下信息：

![img](http://c.biancheng.net/uploads/allimg/190522/4-1Z522110605B7.gif)

### 文字文本运算符（&lt;&gt;）

文字文本（literal-text）运算符（&lt;&gt;）把一个或多个字符和符号组合成一个文字文本，以防止预处理程序把列表中的成员解释为独立的参数。

在字符串含有特殊字符时该运算符非常有用，比如逗号、百分号（%）、和号（&）以及分号（;），这些符号既可以被解释为分隔符，又可以被解释为其他的运算符。例如，之前给岀的宏 mWrite 接收一个字符串文本作为其唯一的实参。如果传递的字符串如下所示，预处理程序就会将其解释为3个独立的实参：

```text
mWrite "Line three", 0dh, 0ah
```

第一个逗号后面的文本会被丢弃，因为宏只需要一个实参。然而，如果用文字文本运算 符将字符串括起来，那么预处理程序就会把尖括号内所有的文本当作一个宏实参：

```text
mWrite ＜"Line three", 0dh, 0ah＞
```

### 文字字符运算符（!）

构造文字字符（literal-character）运算符（!）的目的与文字文本运算符的几乎完全一样：强制预处理程序把预先定义的运算符当作普通的字符。在下面的 TEXTEQU 定义中，运算符 ! 可以防止符号 &gt; 被当作文本分隔符：

```text
BadYValue TEXTEQU <Warning: Y-coordinate is !> 24>
```

#### 警告信息示例

下面的例子有助于说明运算符 %、& 和 ! 是如何工作的。假设已经定义了符号 BadYValue。现在创建一个宏 ShowWarning，接收一个用引号括起来的文本实参，并将其传递给宏 mWrite。注意替换（&）运算符的用法：

```text
ShowWarning MACRO message
  mWrite "&message"
ENDM
```

然后调用 ShowWarning，把表达式 %BadYValue 传递给它。% 运算符计算（解析） BadYValue，并生成与之等价的字符串：

```text
.code
ShowWarning %BadYValue
```

正如所期望的，程序运行并显示警告信息：

```text
Warning: Y-coordinate is > 24
```

## 10.20 宏函数

宏函数与宏过程有相似的地方，它也为[汇编语言](http://c.biancheng.net/asm/)语句列表分配一个名称。不同的地方在于，宏函数通过 EXITM 伪指令总是返回一个常量（整数或字符串）。如下例所示，如果给定符号已定义则宏 IsDefined 返回真（-1）；否则返回假（0）：

```text
IsDefined MACRO symbol
    IFDEF symbol
        EXITM <-l>     ;; 真
    ELSE
        EXITM <0>      ;; 假
    ENDIF
ENDM
```

EXITM （退出宏）伪指令终止了所有后续的宏展开。

### 调用宏函数

调用宏函数时，它的实参列表必须用括号括起来。比如，调用宏 IsDefined 并传递 RealMode （一个可能已定义也可能还未定义的符号名）：

```text
IF IsDefined(RealMode)
  mov ax, @data
  mov ds, ax
ENDIF
```

如果在汇编过程中，汇编器在此之前已经遇到过对 RealMode 的定义，那么它就会汇编这两条指令：

```text
mov ax,@data
mov ds,ax
```

同样的 IF 伪指令可以被放在名为 Startup 的宏内：

```text
Startup MACRO
    IF IsDefined(RealMode)
        mov ax,@data
        mov ds,ax
    ENDIF
ENDM
```

像 IsDefined 这样的宏可以用于设计多种内存模式的程序。比如，可以用它来决定使用哪种头文件：

```text
IF IsDefined(RealMode)
    INCLUDE Irvine16.inc
ELSE
    INCLUDE Irvine32.inc
ENDIF
```

### 定义 RealMode 符号

剩下的任务就只是找到定义 RealMode 符号的方法。方法之一是把下面的代码行放在程序开始的位置：

```text
RealMode = 1
```

或者，汇编器命令行也有选项来定义符号，即，使用 -D。下面的 ML 命令行定义了 RealMode 符号并为其赋值 1：

```text
ML -c -DRealMode=l myProg.asm
```

而保护模式程序中相应的 ML 命令就没有定义 RealMode 符号：

```text
ML -c myProg.asm
```

### HelioNew 程序

下面的程序 \(HelloNew.asm\) 使用刚才介绍的宏，在屏幕上显示了一条消息：

```text
; 宏函数    (HelloNew.asm)

INCLUDE Macros.inc
IF IsDefined( RealMode )
    INCLUDE Irvine16.inc
ELSE
    INCLUDE Irvine32.inc
ENDIF

.code
main PROC
    Startup

    mWrite <"This program can be assembled to run ",0dh,0ah>
    mWrite <"in both Real mode and Protected mode.",0dh,0ah>

    exit
main ENDP
END main
```

16 位实模式程序运行于模拟的 MS-DOS 环境中，使用的是 Irvine16.inc 头文件和 Irvine16 链接库。

## 10.21 定使用WHILE、REPEAT、FOR 和 FORC伪指令定义重复语句块

MASM 有许多循环伪指令用于生成重复的语句块：WHILE、REPEAT、FOR 和 FORC。与 LOOP 指令不同，这些伪指令只在汇编时起作用，并使用常量值作为循环条件和计数器：

* WHILE 伪指令根据一个布尔表达式来重复语句块。
* REPEAT 伪指令根据计数器的值来重复语句块。
* FOR 伪指令通过遍历符号列表来重复语句块。
* FORC 伪指令通过遍历字符串来重复语句块。

### WHILE 伪指令

WHILE 伪指令重复一个语句块，直到特定的常量表达式为真。其语法如下：

```text
WHILE constExpression
  statements
ENDM
```

下面的代码展示了如何在 1 到 F000 0000h 之间生成斐波那契 \(Fibonacci\) 数，作为汇编时常数序列：

```text
WEEKS_PER_YEAR = 52
    WeatherReadings STRUCT
    location BYTE 50 DUP(0)
    REPEAT WEEKS_PER_YEAR
        LOCAL rainfall, humidity
        rainfall DWORD ?
        humidity DWORD ?
    ENDM
WeatherReadings ENDS
```

### REEPEAT 伪指令

在汇编时，REPEAT 伪指令将一个语句块重复固定次数。其语法如下：

```text
REPEAT constExpression
  statements
ENDM
```

constExpression 是一个无符号整数常量表达式，用于确定重复次数。

在创建数组时，REPEAT 的用法与 DUP 类似。在下面的例子中，WeatherReadings 结构含有一个地点字符串和一个包含了降雨量与湿度读数的数组：

```text
WEEKS_PER_YEAR = 52
    WeatherReadings STRUCT
    location BYTE 50 DUP(0)
    REPEAT WEEKS_PER_YEAR
        LOCAL rainfall, humidity
        rainfall DWORD ?
        humidity DWORD ?
    ENDM
WeatherReadings ENDS
```

由于汇编时循环会对降雨量和湿度重定义，使用 LOCAL 伪指令可以避免因其导致的错误。

### FOR 伪指令

FOR 伪指令通过迭代用逗号分隔的符号列表来重复一个语句块。列表中的每个符号都会引发循环的一次迭代过程。其语法如下：

```text
FOR parameter,<arg1,arg2,arg3,...>
  statements
ENDM
```

第一次循环迭代时，parameter 取 arg1 的值，第二次循环迭代时，parameter 取 arg2 的值; 以此类推，直到列表的最后一个实参。

【示例】现在创建一个学生注册的场景，其中，COURSE 结构含有课程编号和学分值；SEMESTER 结构包含一个有 6 门课程的数组和一个计数器 NumCourses：

```text
COURSE STRUCT
    Number BYTE 9 DUP(?)
    Credits BYTE ?
COURSE ENDS
;semester 含有一个课程数组。
SEMESTER STRUCT
    Courses COURSE 6 DUP(<>)
    NumCourses WORD ?
SEMESTER ENDS
```

使用 FOR 循环可以定义 4 个 SEMESTER 对象，每一个对象都从由尖括号括起的符号列表中选择一个不同的名称：

```text
.data
  FOR semName,<Fall2013, [Spring](http://c.biancheng.net/spring/)2014, Summer2014, Fall2014>
  semName SEMESTER <>
ENDM
```

如果查看列表文件就会发现如下变量：

```text
.data
Fall2013 SEMESTER <>
Spring2014 SEMESTER <>
Summer2014 SEMESTER <>
Fall2014 SEMESTER <>
```

### FORC 伪指令

FORC 伪指令通过迭代字符串来重复一个语句块。字符串中的每个字符都会引发循环的一次迭代过程。其语法如下：

```text
FORC parameter, <string>
  statements
ENDM
```

第一次循环迭代时，parameter 等于字符串的第一个字符，第二次循环迭代时，parameter 等于字符串的第二个字符；以此类推，直到最后一个字符。

下面的例子创建了一个字符查找表，其中包含了一些非字母字符。注意，&lt; 和 &gt; 的前面必须有文字字符（!）运算符，以防它们违反FORC伪指令的语法：

```text
Delimiters LABEL BYTE
FORC code, <@#$%^&*!<!>>
  BYTE "&code"
ENDM
```

生成的数据表如下所示，可以在列表文件中查看：

```text
00000000 40 1 BYTE "@"
00000001 23 1 BYTE "#"
00000002 24 1 BYTE "$"
00000003 25 1 BYTE "%"
00000004 5E 1 BYTE "^"
00000005 26 1 BYTE "&"
00000006 2A 1 BYTE "*"
00000007 3C 1 BYTE "<"
00000008 3E 1 BYTE ">"
```

#### 示例：链表

结合结构声明与 REPEAT 伪指令以指示汇编器创建一个链表的[数据结构](http://c.biancheng.net/data_structure/)是相当简单的。链表中的每个节点都含有一个数据域和一个链接域：

![img](http://c.biancheng.net/uploads/allimg/190522/4-1Z522150045911.gif)

在数据域中，一个或多个变量可以保存每个节点所特有的数据。在链接域中，一个指针包含了链表下一个节点的地址。最后一个节点的链接域通常是一个空指针。现在编写程序创建并显示一个简单链表。首先，程序定义一个节点，其中含有一个整数\(数据\)和一个指向下一个节点的指针：

```text
ListNode STRUCT
  NodeData DWORD ?  ;节点的数据
  NextPtr DWORD ?    ;指向下一个节点的指针
ListNode ENDS
```

接着 REPEAT 伪指令创建了 ListNode 对象的多个实例。为了便于测试，NodeData 域含有一个整数常量，其范围为 1〜15，在循环内部，计数器加 1 并将值插入到 ListNode 域：

```text
TotalNodeCount = 15
NULL = 0
Counter = 0

.data
LinkedList LABEL PTR ListNode
REPEAT TotalNodeCount
    Counter = Counter + 1
    ListNode <Counter, ($ + Counter * SIZEOF ListNode)>
ENDM
```

表达式 \($+Counter\*SIZEOF ListNode\) 告诉汇编器把计数值与 ListNode 的大小相乘，并将乘积与当前地址计数器相加。结果值插入结构内的 NextPtr 域。［注意一个有趣的现象：位置计数器的值 \($\) 固定在表的第一节点上。］该表用尾节点 \(tail node\) 来标记末尾，其 NextPtr 域为空 \(0\)：

```text
ListNode <0,0>
```

当程序遍历该表时，它用下面的语句检索 NextPtr 域，并将其与 NULL 比较，以检查是否为表的末尾：

```text
mov eax,(ListNode PTR [esi]).NextPtr
cmp eax,NULL
```

### 程序清单

完整的程序清单如下所示。在 main 中，一个循环遍历链表并显示全部的节点值。与使用固定计数值控制循环相比，程序检查是否为尾节点的空指针，若是则停止循环：

```text
; 创建一个链表            (List.asm)
INCLUDE Irvine32.inc

ListNode STRUCT
  NodeData DWORD ?
  NextPtr  DWORD ?
ListNode ENDS

TotalNodeCount = 15
NULL = 0
Counter = 0

.data
LinkedList LABEL PTR ListNode
REPT TotalNodeCount
    Counter = Counter + 1
    ListNode <Counter, ($ + Counter * SIZEOF ListNode)>
ENDM
ListNode <0,0>    ; tail node

.code
main PROC
    mov  esi,OFFSET LinkedList

; 显示 NodeData 域的值
NextNode:
    ; 检查是否为尾节点
    mov  eax,(ListNode PTR [esi]).NextPtr
    cmp  eax,NULL
    je   quit

    ; 显示节点数据
    mov  eax,(ListNode PTR [esi]).NodeData
    call WriteDec
    call Crlf

    ; 获取下一个节点的指针
    mov  esi,(ListNode PTR [esi]).NextPtr
    jmp  NextNode

quit:
    exit
main ENDP
END main
```

