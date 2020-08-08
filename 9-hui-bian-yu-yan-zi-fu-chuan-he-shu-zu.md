---
description: >-
  学会有效地处理字符串和数组，就能够掌握代码优化中最常见的情况。本章以编写高效代码为目的，阐释字符串和数组处理技术。本章将首先介绍字符串基本指令，它们针对数据块的传送、比较、加载和保存进行过优化。然后是
  Irvine32 和 Irvine64 链接库的几个字符串处理过程，它们的实现与标准 C
  字符串库中的实现非常相似。最后将展示如何利用高级间接寻址方式——基址变址和相对基址变址一一操作二维数组。
---

# 9 汇编语言字符串和数组

## 9.1 字符串基本指令简介

x86 指令集有五组指令用于处理字节、字和双字数组。虽然它们被称为字符串原语 \(string primitives\)，但它们并不局限于字符数组。32 位模式中，下表中的每条指令都隐含使用 ESI、EDI，或是同时使用这两个寄存器来寻址内存。

| 指令 | 说明 |
| :--- | :--- |
| MOVSB、MOVSW、MOVSD | 传送字符串数据：将 ESI 寻址的内存数据复制到 EDI 寻址的内存位置 |
| CMPSB、CMPSW、CMPSD | 比较字符串：比较分别由 ESI 和 EDI 寻址的内存数据 |
| SCASB、SCASW、SCASD | 扫描字符串：比较累加器 \(AL、AX 或 EAX\) 与 EDI 寻址的内存数据 |
| STOSB、STOSW、STOSD | 保存字符串数据：将累加器内容保存到 EDI 寻址的内存位置 |
| LODSB、LODSW、LODSD | 从字符串加载到累加器：将 ESI 寻址的内存数据加载到累加器 |

根据指令数据大小，对累加器的引用隐含使用 AL、AX 或 EAX。字符串原语能高效执行，因为它们会自动重复并增加数组索引。

### 使用重复前缀

就其自身而言，字符串基本指令只能处理一个或一对内存数值。如果加上重复前缀，指令就可以用 ECX 作计数器重复执行。重复前缀使得单条指令能够处理整个数组。下面为可用的重复前缀：

| REP | ECX &gt; 0 时重复 |
| :--- | :--- |
| REPZ、REPE | 零标志位置 1 且 ECX &gt; 0 时重复 |
| REPNZ、REPNE | 零标志位清零且 ECX &gt; 0 时重复 |

【示例】复制字符串：下面的例子中，MOVSB 从 string1 传送 10 个字节到 string2。重复前缀在执行 MOVSB 指令之前，首先测试 ECX 是否大于 0。若 ECX=0，MOVSB 指令被忽略，控制传递到程序的下一行代码；若 ECX&gt;0，则 ECX 减 1 并重复执行 MOVSB 指令：

```text
cld                       ;清除方向标志位
mov esi, OFFSET string1      ; ESI 指向源串
mov edi, OFFSET string2      ; EDI 执行目的串
mov ecx, 10                ;计数器赋值为10
rep movsb                 ;传送io个字节
```

重复 MOVSB 指令时，ESI 和 EDI 自动增加，这个操作由 CPU 的方向标志位控制。

### 方向标志位

根据方向标志位的状态，字符串基本青令增加或减少 ESI 和 EDI 如下表所示。可以用 CLD 和 STD 指令显式修改方向标志位：

```text
CLD ;方向标志位清零（正向）
STD ;方向标志位置 1（反向）
```

| 方向标志位的值 | 对ESI和EDI的影响 | 地址顺序 |
| :--- | :--- | :--- |
| 0 | 增加 | 低到高 |
| 1 | 减少 | 高到低 |

在执行字符串基本指令之前，若忘记设置方向标志位会产生大麻烦，因为 ESI 和 EDI 寄存器可能无法按预期增加或减少。

## 9.2 MOVSB、MOVSW和MOVSD指令：将数据到EDI指向的内存

MOVSB、MOVSW 和 MOVSD 指令将数据从 ESI 指向的内存位置复制到 EDI 指向的内存位置。（根据方向标志位的值）这两个寄存器自动地增加或减少：

| MOVSB | 传送（复制）字节 |
| :--- | :--- |
| MOVSW | 传送（复制）字 |
| MOVSD | 传送（复制）双字 |

MOVSB、MOVSW 和 MOVSD 可以使用重复前缀。方向标志位决定 ESI 和 EDI 是否增加或减少。增加 / 减少的量如下表所示：

| 指令 | ESI 和 EDI 增加或减少的数值 |
| :--- | :--- |
| MOVSB | 1 |
| MOVSW | 2 |
| MOVSD | 4 |

【示例】复制双字数组，假设现在想从 source 复制 20 个双字整数到 target。数组复制完成后，ESI 和 EDI 将分别指向两个数组范围之外的一个位置（即超出 4 字节）：

```text
.data
source DWORD 20 DUP(OFFFFFFFFh)
target DWORD 20 DUP(?)
.code
cld                     ;方向为正向
mov ecx,LENGTHOF source ;设置 REP 计数器
mov esi,OFFSET source   ;ES工指向 source
mov edi,OFFSET target   ;ED工指向 target
rep novsd               ;复制双字
```

## 9.3 CMPSB、CMPSW和CMPSD指令：比较两个操作数

CMPSB、CMPSW 和 CMPSD 指令比较 ESI 指向的内存操作数与 EDI 指向的内存操作数：

| CMPSB | 比较字节 |
| :--- | :--- |
| CMPSW | 比较字 |
| CMPSD | 比较双字 |

CMPSB、CMPSW 和 CMPSD 可以使用重复前缀。方向标志位决定 ESI 和 EDI 的增加或减少。

【示例】比较双字，假设现在想用 CMPSD 比较两个双字。下例中，source 的值小于 target，因此 JA 指令不会跳转到标号 L1。

```text
.data
source DWORD 1234h
target DWORD 5678h
.code
mov esi,OFFSET source
mov edi,OFFSET target
cmpsd                 ;比较双字
ja L1                 ;若 source > target 则跳转
```

比较多个双字时，清除方向标志位（正向），ECX 初始化为计数器，并给 CMPSD 添加重复前缀：

```text
mov esi,OFFSET source
mov edi,OFFSET target
cld                       ;方向为正向
mov ecx,LENGTHOF source   ;设置重复计数器
repe cmpsd                ;相等则重复
```

REPE 前缀重复比较操作，并自动增加 ESI 和 EDI，直到 ECX 等于 0，或者发现了一对不相等的双字。

## 9.4 SCASB、SCASW和SCASD指令：在字符串或数组中寻找一个值

SCASB、SCASW 和 SCASD 指令分别将 AL/AX/EAX 中的值与 EDI 寻址的一个字节 / 字 / 双字进行比较。这些指令可用于在字符串或数组中寻找一个数值。结合 REPE（或 REPZ）前缀，当 ECX &gt; 0 且 AL/AX/EAX 的值等于内存中每个连续的值时，不断扫描字符串或数组。

REPNE 前缀也能实现扫描，直到 AL/AX/EAX 与某个内存数值相等或者 ECX = 0。

扫描是否有匹配字符下面的例子扫描字符串 alpha，在其中寻找字符 F。如果发现该字符，则 EDI 指向匹配字符后面的一个位置。如果未发现匹配字符，则 JNZ 执行退出：

```text
.data
alpha BYTE "ABCDEFGH",0
.code
mov edi,OFFSET alpha        ;ED工指向字符串
mov al, 'F'                 ;检索字符F
mov ecx,LENGTHOF alpha      ;设置检索计数器
cld                         ;方向为正向
repne seasb                 ;不相等则重复
jnz quit                    ;若未发现字符则退出
dec edi                     ;发现字符：EDI 减 1
```

循环之后添加了 JNZ 以测试由于 ECX=0 且没有找到 AL 中的字符而结束循环的可能性。

## 9.5 STOSB、STOSW和STOSD指令：把AL/AX/EAX的内容存储到EDI指向的内存单元中

STOSB、STOSW 和 STOSD 指令分别将 AL/AX/EAX 的内容存入由 EDI 中偏移量指向的内存位置。EDI 根据方向标志位的状态递增或递减。

与 REP 前缀组合使用时，这些指令实现用同一个值填充字符串或数组的全部元素。例如，下面的代码就把 string1 中的每一个字节都初始化为 OFFh：

```text
.data
Count = 100
string1 BYTE Count DUP(?)
.code
mov al, OFFh              ;要保存的数值
mov edi,OFFSET string1       ;ED：［指向目标字符串
mov ecx,Count              ;字符计数器
cld                          ;方向为正向
rep stosb                  ;用 AL 的内容实现填充
```

## 9.6 LODSB、LODSW和LODSD指令：加载一个字节或字

LODSB、LODSW 和 LODSD 指令分别从 ESI 指向的内存地址加载一个字节或一个字到 AL/AX/EAX。ESI 按照方向标志位的状态递增或递减。

LODS 很少与 REP 前缀一起使用，原因是，加载到累加器的新值会覆盖其原来的内容。相对而言，LODS常常被用于加载单个 数值。在后面的例子中，LODSB代替了如下两条指令（假设方向标志位清零）：

```text
mov al, ［esi］  ;将字节送入AL
inc esi  ;指向下一个字节
```

【示例】数组乘法，下面的程序把一个双字数组中的每个元素都乘以同一个常数。程序同时 使用了 LODSD 和 STOSD：

```text
;数组乘法    （Mult.asm）
;本程序将一个32位整数数组中的每个元素都乘以一个常数。
INCLUDE Irvine32.inc
.data
array DWORD 1,2,3,4,5,6,7,8,9,10    ;测试数据
mug" 0W0RD -10
.code

main PROC
    cld                             ;方向为正向
    mov esi,OFFSET array            ;源数组索引
    itqv edi,esi                    ;目标数组索引
    mov ecx,LENGTHOF array          ;循环计数器
L1: lodsd                           ;将 [ESI] 加载到 EAX
mul multiplier                      ;与常数相乘
stosd                               ;将 EAX 保存到［EDI］
loop L1
exit
main ENDP
END main
```

## 9.7 Irvine32字符串过程详解\[附带实例\]

本节将演示用 Irvine32 链接库中的几个过程来处理空字节结束的字符串。这些过程与标准 C 库中的函数有着明显的相似性：

```text
;将源串复制到目的串。
Str_copy PROTO,
  source:PTR BYTE,
  target:PTR BYTE
;用 EAX 返回串长度（包括零字节）。
Str_length PROTO,
  pString:PTR BYTE
;比较字符串 1 和字符串 2。
;并用与 CMP 指令相同的方法设置零标志位和进位标志位。
Str_compare PROTO,
  string1:PTR BYTE,
  string2:PTR BYTE
;从字符串尾部去掉特定的字符。
;第二个参数为要去除的字符。
Str_trim PROTO,
  pString:PTR BYTE,
  char:BYTE
;将字符串转换为大写。
Str_ucase PROTO,
  pString:PTR BYTE
```

### Str\_compare 过程

Str\_compare 过程比较两个字符串，其调用格式如下：

```text
INVOKE Str_compare, ADDR string1, ADDR string2
```

它从第一个字节开始按正序比较字符串。这种比较是区分大小写的，因为字母的大写和小写 ASCII 码不相同。该过程没有返回值，若参数为 string1 和 string2，则进位标志位和零标志位的含义如下表所示。

| 关系 | 进位标志位 | 零标志位 | 为真则分支（指令） |
| :--- | :--- | :--- | :--- |
| string1 &lt; string2 | 1 | 0 | JB |
| string1 = string2 | 0 | 1 | JE |
| string1 &gt; string2 | 0 | 0 | JA |

回顾《[CMP指令](http://c.biancheng.net/view/3561.html)》一节中 CMP 指令如何设置进位标志位和零标志位。下面给出了 Str\_compare 过程的代码清单。

```text
;--------------------------------------
Str_compare PROC USES eax edx esi edi,
    string1:PTR BYTE,
    string2:PTR BYTE
;比较两个字符串。
;无返回值，但是零标志位和进位标志位受到的影响与 CMP 指令相同。
;--------------------------------------
    mov esi, string1
    mov edi, string2
L1: mov al, [esi]
    mov dl, [edi]
    cmp al, 0            ; string1 结束？
    jne L2               ; 否
    cmp dl, 0            ; 是：string2 结束？
    jne L2               ; 否
    jmp L3               ; 是，退出且ZF=1
L2: inc esi              ; 指向下一个字符
    inc edi              ; 字符相等？
    cmp al,dl            ; 是：继续循环
    je L1
L3: ret                  ; 否：退出并设置标志位
Str_compare ENDP
```

实现 Str\_compare 时也可以使用 CMPSB 指令，但是这条指令要求知道较长字符串的长度，这样就需要调用 Str\_length 程两次。

本例中，在同一个循环内检测两个字符串的零结束符显得更加容易。CMPSB 在处理长度已知的大型字符串或数组时最有效。

### Str\_length 过程

Str\_length 过程用 EAX 返回一个字符串的长度。调用该过程时，要传递字符串的偏移地址。例如：

```text
INVOKE Str_length, ADDR myString
```

过程实现如下：

```text
Str_length PROC USES edi,
    pString:PTR BYTE       ;指向字符串
    mov edi, pString       ;字符计数器
    mov eax, 0             ;字符结束？
L1: cmp BYTE PTR[edi],0
    je L2                  ;是：退出
    inc edi                ;否：指向下一个字符
    inc eax                ;计数器加1
    jmp L1
L2: ret
Str_length ENDP
```

### Str\_copy 过程

Str\_copy 过程把一个空字节结束的字符串从源地址复制到目的地址。调用该过程之前，要确保目标操作数能够容纳被复制的字符串。Str\_copy 的调用语法如下：

```text
INVOKE Str_copy, ADDR source, ADDR target
```

过程无返回值。下面是其实现：

```text
;--------------------------------------
Str_copy PROC USES eax ecx esi edi,
    source:PTR BYTE,       ; source string
    target:PTR BYTE        ; target string
;将字符串从源串复制到目的串。
;要求：目标串必须有足够空间容纳从源复制来的串。
;--------------------------------------
    INVOKE Str_length, source      ;EAX = 源串长度
    mov ecx, eax                   ;重复计数器
    inc    ecx                     ;由于有零字节，计数器加 1
    mov esi, source
    mov edi, target
    cld                            ;方向为正向
    rep    movsb                   ;复制字符串
    ret
Str_copy ENDP
```

### Str\_trim 过程

Str\_trim 程从空字节结束字符串中移除所有与选定的尾部字符匹配的字符。其调用语法如下：

```text
INVOKE Str_trim, ADDR string, char_to_trim
```

这个过程的逻辑很有意思，因为程序需要检查多种可能的情况（以下用 \# 作为尾部字符）：

1\) 字符串为空。

2\) 字符串一个或多个尾部字符的前面有其他字符，如“Hello\#”。

3\) 字符串只含有一个字符，且为尾部字符，如“\#”。

4\) 字符串不含尾部字符，如“Hello”或“H”。

5\) 字符串在一个或多个尾部字符后面跟随有一个或多个非尾部字符，如“\#H”或“\#\#Hello”

使用 Str\_trim 过程可以删除字符串尾部的全部空格（或者任何重复的字符）。从字符串中去掉字符的最简单的方法是，在想要移除的字符前面插入一个空字节。空字节后面的任何字符都会变得无意义。

下表列出了一些有用的测试例子。在所有例子中都假设从字符串中删除的是 \# 字符，表中给出了期望的输出。

| 输入字符串 | 预期修改后的字符串 |
| :--- | :--- |
| “Hello\#\#” | “Hello” |
| “\#” | “”（空字符串） |
| “Hello” | "Hello” |
| “H” | “H” |
| “\#H” | “\#H” |

现在来看看测试 Str\_trim 程的代码。INVOKE 语句向 Str\_trim 传递字符串地址：

```text
.datastring_1 BYTE "Hello##",0.codeINVOKE Str_trim,ADDR string_1,'#'INVOKE ShowString,ADDR string_1
```

ShowString 过程用方括号显示了被裁剪后的字符串，这里未给出其代码。过程输出示例如下：

```text
[Hello]
```

下面给出了 Str\_trim 的实现，它在想要保留的最后一个字符后面插入了一个空字节。空字节后面的任何字符一般都会被字符串处理函数所忽略。

```text
;-------------------------------------
;Str_trim
;从字符串末尾删除所有与给定分隔符匹配的字符。
;返回：无
;-------------------------------------
Str_trim PROC USES eax ecx edi,
    pString:PTR BYTE,                ;指向字符串
    char: BYTE                       ;要移必的字符
    mov edi,pString                  ;准备调用 Str_length
    INVOKE Str_length,edi            ;用 EAX 返回鬆
    cmp eax,0                        ;长度是否为零？
    je L3                            ;是：立刻退出
    mov ecx, eax                     ;否：ECX = 字符串长度
    dec eax
    add edi,eax                      ;指向最后一个字符
L1: mov al, [edi]                    ;取一个字符
    cmp al,char                      ;是否为分隔符？
    jne L2                           ;否：插入空字节
    cec edi                          ;是：继续后退一个字符
    loop L1                          ;直到字符串的第一个字符
L2: mov BYTE PTR [edi+1 ],0          ;插入一个空字节
L3:    ret
Str_trim ENDP
```

#### 详细说明

现在仔细研究一下 Str\_trim。该算法从字符串最后一个字符开始，反向进行串扫描，以寻找第一个非分隔符字符。当找到这样的字符后，就在该字符后面的位置上插入一个空字节：

```c
ecx = length(str)
if length (str) > 0 then
    edi = length - 1
    do while ecx > 0
        if str[edi] ≠ delimiter then
            str[edi+1] = null
            break
        else
            edi = edi - 1
        end if
        ecx = ecx - 1
    end do
```

下面逐行查看代码实现。首先，pString 为待裁剪字符串的地址。程序需要知道该字符串的长度，Str\_length 过程用 EDI 寄存器接收其输入参数：

```text
mov edi,pString       ;准备调用 Str_length
INVOKE Str_length,edi  ;过程返回值在 Eax 中
```

Str\_length 过程用 EAX 寄存器返回字符串长度，所以，后面的代码行将它与零进行比较，如果字符串为空，则跳过后续代码：

```text
cmp eax,0  ;字符串长度等于零吗？
je L3       ;是：立刻退出
```

在继续后面的程序之前，先假设该字符串不为空。ECX 为循环计数器，因此要将字符串长度赋给它。由于希望 EDI 指向字符串最后一个字符，因此把 EAX（包含字符串长度）减 1 后再加到 EDI 上：

```text
mov ecx,eax     ;否：ECX =字符串长度
dec eax
add edi,eax      ;指向最后一个字符
```

现在 EDI 指向的是最后一个字符，将该字符复制到 AL 寄存器，并与分隔符比较：

```text
L1: mov al, [edi]  ;取字符
  cmp al, char   ;是分隔符吗？
```

如果该字符不是分隔符，则退出循环，并用标号为 L2 的语句插入一个空字节：

```text
jne L2          ;否：插入空字节
```

否则，如果发现了分隔符，则继续循环，逆向搜索字符串。实现的方法为：将 EDI 后退一个字符，再重复循环：

```text
dec edi         ;是：继续后退
loop L1        ;直到字符串的第一个字符
```

如果整个字符串都由分隔符组成，则循环计数器将减到零，并继续执行 loop 指令下面的代码行，即标号为 L2 的代码，在字符串中插入一个空字节：

```text
L2: mov BYTE PTR [edi+1], 0   ;插入空字节
```

假如程序控制到达这里的原因是循环计数减为零，那么，EDI 就会指向字符串第一个字符之前的位置。因此需要用表达式 \[edi+1\] 来指向第一个字符。

在两种情况下，程序会执行标号 L2：

* 其一，在字符串中发现了非分隔符字符；
* 其二， 循环计数减为零。

标号 L2 后面是标号为 L3 的 RET 指令，用来结束整个过程：

```text
L3:  ret
Str_trim ENDP
```

### Str\_ucase 过程

Str\_ucase 过程把一个字符串全部转换为大写字母，无返回值。调用过程时，要向其传 递字符串的偏移量：

```text
INVOKE Str_ucase, ADDR myString
```

过程实现如下：

```text
;---------------------------------
;Str_ucase
;将空字节结束的字符串转换为大写字母。
;返回：无
;---------------------------------
Str_ucase PROC USES eax esi,
pString:PTR BYTE
    mov esi,pString
L1:
    mov al, [esi]           ;取字符
    cmp al, 0               ;字符串是否结束？
    je L3                   ;是：退出
    cnp al, 'a'             ;小于"a" ？
    jb L2
    cnp al, 'z'             ;大于"z" ？
    ja L2
    and BYTE PTR [esi], 11011111b ;转换字符
L2: inc esi                 ;下一个字符
    jmp L1
L3: ret
Str_ucase ENDP
```

### 字符串演示程序

下面的 32 位程序演示了对 Irivne32 链接库中 Str\_trim、Str\_ucase、Str\_compare 和 Str\_length 过程的调用：

```text
; String Library Demo    (StringDemo.asm)
; 该程序演示了链接库中字符串处理过程
INCLUDE Irvine32.inc

.data
string_1 BYTE "abcde////",0
string_2 BYTE "ABCDE",0
msg0     BYTE "string_1 in upper case: ",0
msg1     BYTE "string1 and string2 are equal",0
msg2     BYTE "string_1 is less than string_2",0
msg3     BYTE "string_2 is less than string_1",0
msg4     BYTE "Length of string_2 is ",0
msg5     BYTE "string_1 after trimming: ",0

.code
main PROC

    call trim_string
    call upper_case
    call compare_strings
    call print_length

    exit
main ENDP

trim_string PROC
; 从 string_1 删除尾部字符
    INVOKE Str_trim, ADDR string_1,'/'
    mov        edx,OFFSET msg5
    call    WriteString
    mov        edx,OFFSET string_1
    call    WriteString
    call    Crlf

    ret
trim_string ENDP

upper_case PROC
; 将 string_1 转换为大写字母
    mov        edx,OFFSET msg0
    call    WriteString
    INVOKE  Str_ucase, ADDR string_1
    mov        edx,OFFSET string_1
    call    WriteString
    call    Crlf

    ret
upper_case ENDP

compare_strings PROC
; 比较 string_1 和 string_2.
    INVOKE Str_compare, ADDR string_1, ADDR string_2
    .IF ZERO?
    mov    edx,OFFSET msg1
    .ELSEIF CARRY?
    mov    edx,OFFSET msg2     ; string 1 小于...
    .ELSE
    mov    edx,OFFSET msg3     ; string 2 小于...
    .ENDIF
    call    WriteString
    call    Crlf

    ret
compare_strings  ENDP

print_length PROC
; 显示 string_2 的长度
    mov        edx,OFFSET msg4
    call    WriteString
    INVOKE  Str_length, ADDR string_2
    call    WriteDec
    call    Crlf

    ret
print_length ENDP
END main
```

调用 Str\_trim 过程从 string\_1 删除尾部字符，调用 Str\_ucase 过程将字符串转换为大写字母。

String Library Demo 程序的输出如下所示：

![img](http://c.biancheng.net/uploads/allimg/190517/4-1Z51GA1311a.gif)

## 9.8 Irivne64字符串过程详解\[附带实例\]

下面将一些比较重要的字符串处理过程从 Irvine32 链接库转换为 64 位模式。变化非常简单，删除堆栈参数，并将所有的 32 位寄存器都替换为 64 位寄存器。

下表列出了这些字符串过程、过程说明及其输入输出。

| Str\_compare | 比较两个字符串 输入参数：RSI 为源串指针，RDI 为目的串指针 返回值：若源串 &lt; 目的串，则进位标志位 CF=1；若源串 = 目的串，则零标志位 ZF=1；若源串 &gt; 目的串，则 CF=0 且 ZF=0 |
| :--- | :--- |
| Str\_copy | 将源串复制到目的指针指向的位置 输入参数：RSI 为源串指针，RDI 指向被复制串将要存储的位置 |
| Str\_length | 返回空字节结束字符串的长度 输入参数：RCX 为字符串指针 返回值：RAX 为该字符串的长度 |

Str\_compare 过程中，RSI 和 RDI 是输入参数的合理选择，因为字符串比较循环会用到它们。使用这两个寄存器参数能在过程开始时避免将输入参数复制到 RSI 和 RDI 寄存器中：

```text
;------------------------------------
;Str_compare
;比较两个字符串
;接收：RSI 为源串指针
;     RDT 为目的串指针
;返回：若字符串相等，ZF 置 1
;      若源串 < 目的串，CF 置 1
;------------------------------------
Str_compare PROC USES rax rdx rsi rdi
L1: mov al,[rsi]
    mov dl,[rdi]
    cmp al, 0            ; string1 结束？
    jne L2               ; 否
    cmp dl, 0            ;是：string2 结束？
    jne L2               ;否
    jmp L3               ;是：退出且 ZF=1
L2: inc rsi              ;指向下一个字符
    inc rdi
    cmp al,dl            ;字符相等？
    je L1                ;是：继续循环
                         ;否：退出并设置标志位
L3: ret
Str_compare ENDP
```

注意，PROC 伪指令用 USES 关键字列出了所有需要在过程开始时入栈、在过程时返回出栈的寄存器。

Str\_copy 过程用 RSI 和 RDI 接收字符串指针：

```text
;-------------------------------------
;Str_copy
;复制字符串
;接收：RSI 为源串指针
;     RDI 为目的串指针
;返回：无
;-------------------------------------
Str_copy PROC USES rax rex rsi rdi
    mov rex,rsi               ;获得源串长度
    call Str_length           ;RAX 返回长度
    mov rex,rax               ;循环计数器
    inc rex                   ;有空字节，加 1
    cld                       ;方向为正向
    rep movsb                 ;复制字符串
    ret
Str_copy ENDP
```

Str\_length 过程用 RCX 接收字符串指针，然后循环扫描该字符串直到发现空字节。字符串长度用 RAX 返回：

```text
;-------------------------------------
;Str_length
;计算辜符串长度
;接收：RCX 指向字符串
;返回：RAX 为字符串长度
;-------------------------------------
Str_length PROC USES rdi
    mov rdi,rex           ;获得指针
    mov eax,0             ;字符计数
L1：
    cmp BYTE PTR [rdi],0  ;字符串结束？
    je L2                 ;是：退出
    inc rdi               ;否：指向下一个字符
    inc rax               ;计数器加 1
    jmp L1
L2: ret                   ;RAX 返回计数值
Str_length ENDP
```

#### 一个简单的测试程序

下面的测试程序调用了 64 位的 Str\_length、Str\_copy 和Str\_compare 过程。虽然程序中没有显示字符串的语句，但是建议在 Visual Studio 凋试器中运行，这样就可以查看内存窗口、寄存器和标志位。

```text
; 测试 Irvine64 字符串程序
Str_compare        proto
Str_length        proto
Str_copy        proto
ExitProcess     proto

.data
source byte "AABCDEFGAABCDFG",0      ; 大小为 15
target byte 20 dup(0)
.code
main proc
    mov   rax,offset source
    call  Str_length                ; 用 RAX 返回长度

    mov   rsi,offset source
    mov   rdi,offset target
    call  str_copy

; 由于刚刚才复制了字符串，因此它们应该相等

    call  str_compare                ; ZF = 1, 字符串相等

; 修改目的串的第一个字符，再比较两个字符串
; compare them again:

    mov   target,'B'
    call  str_compare                ; CF = 1, 源串 < 目的串

    mov   ecx,0
    call  ExitProcess
main ENDP
```

## 9.9 二维数组简介

在[汇编语言](http://c.biancheng.net/asm/)程序员看来，二维数组是一位数组的高级抽象。高级语言有两种方法在内存中存放数组的行和列：行主序和列主序，如下图所示。

![&#x884C;&#x4E3B;&#x5E8F;&#x548C;&#x5217;&#x4E3B;&#x5E8F;](http://c.biancheng.net/uploads/allimg/190520/4-1Z52009554X11.gif)

使用行主序（最常用）时，第一行存放在内存块开始的位置，第一行最后一个元素后面紧跟的是第二行的第一个元素。使用列主序时，第一列的元素存放在内存块开始的位置，第一列最后一个元素后面紧跟的是第二列的第一个元素。

用汇编语言实现二维数组时，可以选择其中的任意一种顺序。这里使用的是行主序。如果是为高级语言编写汇编子程序，那么应该使用高级语言文档中指定的顺序。

x86 指令集有两种操作数类型：基址-变址和基址-变址-位移量，这两种类型都适用于数组。下面将对它们进行研究并通过例子来说明如 何有效地使用它们。

### 基址-变址操作数

基址-变址操作数将两个寄存器（称为基址和变址）相加，生成一个偏移地址：

```text
[base + index]
```

其中的方括号是必需的。32 位模式下，任一 32 位通用寄存器都可以用作基址和变址寄存器。（通常情况下避免使用 EBP，除非进行堆栈寻址。）下面的例子是 32 位模式中基址和变址操作数的各种组合：

```text
.data
array WORD 1000h,2000h,3000h
.code
mov ebx,OFFSET array
mov esi, 2
mov ax,[ebx+esi]    ; AX = 2000h
mov edi, OFFSET array
mov ecx,4
mov ax,[edi+ecx]    ; AX = 3000h
mov ebp,OFFSET array
mov esi, 0
mov ax,[ebp+esi]    ; AX = l000h
```

#### 二维数组

按行访问一个二维数组时，行偏移量放在基址寄存器中，列偏移量放在变址寄存器中。例如，下表给出的数组为 3 行 5 列：

```text
tableB BYTE 10h, 20h, 30h, 40h, 50h
Rowsize = （$ - tableB）
    BYTE 60h, 70h, 80h, 90h, 0A0h
    BYTE 0B0h, 0C0h, 0D0h, 0E0h, 0F0h
```

该表为行主序，汇编器计算的常数 Rowsize 是表中每行的字节数。如果想用行列坐标定位表中的某个表项，则假设坐标基点为 0，那么，位于行 1 列 2 的表项为 80h。

将 EBX 设置为该表的偏移量，加上（Rowsizerow\_index），计算出行偏移量，将 ESI 设置为列索引：

```text
row_index = 1
column_index = 2
mov ebx, OFFSET tableB            ; 表偏移量
add ebx, RowSize * row_index      ; 行偏移量
mov esi, column_index
mov al,[ebx + esi]                ; AL = 80h
```

假设该数组位置的偏移量为 0150h，则其有效地址表示为 EBX+ESI，计算得 0157h。下图展示了如何通过 EBX 加上 ESI 生成 tableB\[1, 2\] 字节的偏移量。如果有效地址指向该程序数据区之外，那么就会产生一个运行时错误。

![&#x7528;&#x57FA;&#x5740;-&#x53D8;&#x5740;&#x64CD;&#x4F5C;&#x6570;&#x5BFB;&#x5740;&#x6570;&#x7EC4;](http://c.biancheng.net/uploads/allimg/190520/4-1Z520095K1b5.gif)

#### 1\) 计算数组行之和

基于变址的寻址简化了二维数组的很多操作。比如，用户可能想要计算一个整数矩阵中一行的和。下面的 32 位 calc\_row\_sum 程序就计算了一个 8 位整数矩阵中被选中行的和数：

```text
;------------------------------------------------------------
; calc_row_sum
; 计算字节矩阵中一行的和数
; 接收: EBX = 表偏移量, EAX = 行索引
;       ECX = 按字节计的行大小
; 返回:  EAX 为和数
;------------------------------------------------------------
calc_row_sum PROC uses ebx ecx edx esi

    mul     ecx              ; 行索引 * 行大小
    add     ebx,eax          ; 行偏移量
    mov     eax,0            ; 累加器
    mov     esi,0            ; 列索引

L1:    movzx edx,BYTE PTR[ebx + esi]        ; 取一个字节
    add     eax,edx                         ; 与累加器相加
    inc     esi                             ; 行中的下一个字节
    loop L1

    ret
calc_row_sum ENDP
```

BYTE PTR 是必需的，用于声明 MOVZX 指令中操作数的类型。

#### 2\) 比例因子

如果是为字数组编写代码，则需要将变址操作数乘以比例因子 2。下面的例子定位行 1 列 2 的元素值：

```text
tablew WORD 10h, 20h, 30h, 40h, 50h
RowsizeW = ($ - tableW)
    WORD 60h, 70h, 80h, 90h, 0A0h
    WORD 0B0h, 0C0h, 0D0h, 0E0h, 0F0h
.code
row_index = 1
column_index = 2
mov ebx,OFFSET tableW             ;表偏移量
add ebx,RowSizeW * row_index      ;行偏移量
mov esi, column_index
mov ax,[ebx + esi*TYPE tableW]    ;AX = 0080h
```

本例的比例因子 \(TYPE tableW\) 等于 2。同样，如果数组类型为双字，则比例因子为 4：

```text
tableD DWORD 10h, 20h, . . .etc.
.code
mov eax,[ebx + esi*TYPE tableD]
```

### 基址-变址-偏移量操作数

基址-变址-偏移量操作数用一个偏移量、一个基址寄存器、一个变址寄存器和一个可选的比例因子来生成有效地址。格式如下：

```text
[base + index + displacement]
displacement[base + index]
```

Displacement \( 偏移量 \) 可以是变量名或常量表达式。32 位模式下，任一 32 位通用寄存器都可以用作基址和变址寄存器。基址-变址-偏移量操作数非常适于处理二维数组。偏移量可以作为数组名，基址操作数为行偏移量，变址操作数为列偏移量。

双字数组示例 下面的二维数组包含了 3 行 5 列的双字：

```text
tableD DWORD 10h, 20h, 30h, 40h, 50h
Rowsize = ($ - tableD)
DWORD 60h, 70h, 80h, 90h, 0A0h
DWORD 0B0h, 0C0h, 0D0h, 0E0h, 0F0h
```

Rowsize 等于 20 \(14h\) 。假设坐标基点为 0，那么位于行 1 列 2 的表项为 80h。为了访问到这个表项，将 EBX 设置为行索引，ESI 设置为列索引：

```text
mov ebx, Rowsize       ;行索弓|
mov esi, 2             ;列索引
mov eax, tableD[ebx + esi*TYPE tableD]
```

设 tableD 开始于偏移量 0150h 处，下图展示了 EBX 和 ESI 相对于该数组的位置。偏移量为十六进制。

![&#x57FA;&#x5740;-&#x53D8;&#x5740;-&#x504F;&#x79FB;&#x91CF;&#x64CD;&#x4F5C;&#x6570;&#x793A;&#x4F8B;](http://c.biancheng.net/uploads/allimg/190520/4-1Z5201000263E.gif)

### 64 位模式下的基址-变址操作数

64 位模式中，若用寄存器索引操作数则必须为 64 位寄存器。基址-变址操作数和基址-变址-偏移量操作数都可以使用。

下面是一段小程序，它用 get\_tableVal 过程在 64 位整数的二维数组中定位一个数值。如果将其与前面的 32 位代码进行比较，会发现 ESI 被替换为 RSI，EAX 和 EBX 也成了 RAX 和 RBX。

```text
;64 位模式下的二维数组 (TwoDimArrays.asm)
Crlf        proto
WriteInt64  proto
ExitProcess proto

.data
table QWORD 1,2,3,4,5
RowSize = ($ - table)
      QWORD 6,7,8,9,10
      QWORD 11,12,13,14,15

.code
main proc
; 基址-变址-偏移量操作数

    mov    rax,1                    ; 行索引基点为0
    mov    rsi,4                    ; 列索引基点为0
    call    get_tableVal            ; RAX中为返回值
    call    WriteInt64              ; 显示返回值
    call    Crlf

    mov   ecx,0           
    call  ExitProcess               ; 程序结束
main endp

;------------------------------------------------------
; get_tableVal
; 返回四字二维数组中给定行列值的元素
; 接收: RAX = 行数, RSI = 列数
; 返回: RAX中的数值
;------------------------------------------------------
get_tableVal proc uses rbx
    mov    rbx,RowSize
    mul    rbx                ; 乘积（低） = RAX
    mov    rax,table[rax + rsi*TYPE table]

    ret
get_tableVal endp
end
```

## 9.10 冒泡排序简述

冒泡排序从位置 0 和 1 开始，对比数组的两个数值。如果比较结果为逆序，就交换这两个数。下图展示了对一个整数数组进行一次遍历的过程。

![img](http://c.biancheng.net/uploads/allimg/190520/4-1Z520111602556.gif)

一次冒泡过程之后，数组仍没有按序排列，但此时最高索引位置上是最大数。外层循环则开始对该数组再一次遍历。经过 1 次遍历后，数组就会按序排列。

冒泡排序对小型数组效果很好，但对较大的数组而言，它的效率就十分低下。计算机科学家在衡量算法的相对效率时，常常使用一种被称为 “时间复杂度”（big-O）的概念来描述随着处理对象数量的增加，平均运行时间是如何增加的。

冒泡排序是 O \(n²\) 算法，这就意味着，它的平均运行时间随着数组元素 \(n\) 个数的平方增加。比如，假设 1000 个元素排序需要 0.1 秒。当元素个数增加 10 倍时，该数组排序所需要的时间就会增加 10² \(100\) 倍。

下表列出了不同数组大小需要的排序时间，假设 1000 个数组元素排序花费 0.1 秒：

| 数组大小 | 时间\(秒\) | 数组大小 | 时间\(秒\) |
| :--- | :--- | :--- | :--- |
| 1 000 | 0.1 | 100 000 | 1000 |
| 10 000 | 10.0 | 1 000 000 | 100 000 （27.78 小时） |

对于一百万个整数来说，冒泡排序谈不上有效率，因为它完成任务的时间太长了！但是对于几百个整数，它的效率是足够的。

用类似于[汇编语言](http://c.biancheng.net/asm/)的伪代码为冒泡排序编写的简化代码是有用的。代码用 N 表示数组大小，cx1 表示外循环计数器，cx2 表示内循环计数器：

```text
cx1 = N - 1
while( cxl > 0 )
{
    esi = addr (array)
    cx2 = cx1
    while ( cx2 > 0 )
    {
        if(array[esi] > array[esi+4])
            exchange(array[esi], array[esi+4])
        add esi,4
        dec cx2
    }
    dec cxl
}
```

如保存和恢复外循环计数器等的机械问题被刻意忽略了。注意内循环计数 \(cx2\) 是基于外循环计数 \(cx1\) 当前值的，每次遍历数组时它都依次递减。

根据伪代码能够很容易生成与之对应的汇编程序，并将它表示为带参数和局部变量的过程：

```text
;----------------------------------------
;BubbleSort
;使用冒泡算法，将一个 32 位有符号整数数组按升序进行排列。
;接收：数组指针，数组大小
;返回：无
;----------------------------------------
BubbleSort PROC USES eax ecx esi,
    pArray:PTR DWORD,           ;数组指针
    Count: DWORD                ;数组大小
    mov ecx,Count
    dec ecx                     ;计数值减1
L1: push ecx                    ;保存外循环计数值
    mov esi,pArray              ;指向第一个数值
L2: mov eax, [esi]              ;取数组元素值
    cmp [esi+4],eax             ;比较两个数值
    jg L3                       ;如果[ESI]<=[ESI + 4]，不交换
    xchg eax, [esi+4]           ;交换两数
    mov [esi],eax
L3: add esi,4                   ;两个指针都向前移动一个元素
    loop L2                     ;内循环
    pop    ecx                  ;恢复外循环计数值
    loop L1                     ;若计数值不等于0,则继续外循环
L4: ret
BubbleSort ENDP
```

## 9.11 对半查找（二分查找）简述

数组查找是日常编程中最常见的一类操作。对小型数组 \(1000 个元素或更少 \) 而言，顺序查找\(sequential search\) 是很容易的，从数组开始的位置顺序检查每一个元素，直到发现匹配的元素为止。对任意 n 个元素的数组，顺序查找平均需要比较 n/2 次。如果查找的是小型数组，则执行时间也很少。但是，如果查找的数组包含一百万个元素就需要相当多的处理时间了。

对半查找 \(binary search\) 算法用于从大型数组中查找一个数值是非常有效的。但是它有一个重要的前提：数组必须是按升序或降序排列。下面的算法假设数组元素是升序：

开始查找前，请求用户输入一个整数，将其命名为 searchVal

1\) 被查找数组的范围用下标行 first 和 last 来表示。如果 first &gt; last，则退出查找，也就是说没有找到匹配项。

2\) 计算位于数组 first 和 last 下标之间的中点。

3\) 将 searchVal 与数组中点进行比较：

* 如果数值相等，将中点送入 EAX，并从过程返回。该返回值表示在数组中发现了匹配值。
* 否则，如果 searchVal 大于中点值，则将 first 重新设置为中点后一位元素的位置。
* 或者，如果 searchVal 小于中点值，则将 last 重新设置为中点前一位元素的位置。

4\) 返回步骤 1

对半查找效率高的原因是它采用了分而治之的策略。每次循环迭代中，数值范围都被对半分为成两部分。通常它被描述为 O \(log n\) 算法，即，当数组元素增加 n 倍时，平均查找时间仅增加 log₂n 倍。

为了帮助了解对半查找效率有多高，下表列出了数组大小相同时，顺序查找和对半查找需要执行的最大比较次数。表中的数据代表的是最坏的情况一一在实际应用 中，经过更少次的比较就可能找到匹配数值。

| 数组大小 | 顺序查找 | 对半查找 |
| :--- | :--- | :--- |
| 64 | 64 | 6 |
| 1 024 | 1 024 | 10 |
| 65 536 | 65 536 | 17 |
| 1 048 576 | 1 048 576 | 21 |
| 4 294 967 296 | 4 294 967 296 | 33 |

下面是用 [C++](http://c.biancheng.net/cplus/) 语言实现的对半查找功能，用于有符号整数数组：

```text
int BinSearch( int values[], const int searchVal, int count )
{
    int first = 0;
    int last = count - 1;
    while( first <= last )
    {
        int mid = (last + first) / 2;
        if( values[mid] < searchVal )
            first = mid + 1;
        else if( values[mid] > searchVal )
            last = mid - 1;
        else
            return mid;    // 成功
    }
    return -1;    // 未找至U
}
```

该 C++ 代码示例的[汇编语言](http://c.biancheng.net/asm/)程序清单如下所示：

```text
;-------------------------------------------------------------
; Binary Search procedure
INCLUDE Irvine32.inc
.code
BinarySearch PROC USES ebx edx esi edi,
    pArray:PTR DWORD,          ; 数组指针
    Count:DWORD,               ; 数组大学
    searchVal:DWORD            ; 给定查找数值
LOCAL first:DWORD,             ; first 的位置
    last:DWORD,                ; last 的位置
    mid:DWORD                  ; 中点

; 接收: 数组指针、数组大小、给定查找数值
; 返回: 若发现匹配项则EAX=该匹配元素在数组中的位置 否则 EAX = -1
;-------------------------------------------------------------
    mov     first,0              ; first = 0
    mov     eax,Count            ; last = (count - 1)
    dec     eax
    mov     last,eax
    mov     edi,searchVal        ; EDI = searchVal
    mov     ebx,pArray           ; EBX 为数组指针

L1: ; while first <= last
    mov     eax,first
    cmp     eax,last
    jg     L5                    ; 退出查找
; mid = (last + first) / 2
    mov     eax,last
    add     eax,first
    shr     eax,1
    mov     mid,eax

; EDX = values[mid]
    mov     esi,mid
    shl     esi,2                ; 将 mid 值乘 4
    mov     edx,[ebx+esi]        ; EDX = values[mid]

; if ( EDX < searchval(EDI) )
;    first = mid + 1;
    cmp     edx,edi
    jge     L2
    mov     eax,mid                ; first = mid + 1
    inc     eax
    mov     first,eax
    jmp     L4

; else if( EDX > searchVal(EDI) )
;    last = mid - 1;
L2:    cmp     edx,edi
    jle     L3
    mov     eax,mid                ; last = mid - 1
    dec     eax
    mov     last,eax
    jmp     L4

; else return mid
L3:    mov     eax,mid                  ; 发现数值
    jmp     L9                          ; 返回 (mid)

L4:    jmp     L1                       ; 继续循环

L5:    mov     eax,-1                   ; 查找失败
L9:    ret
BinarySearch ENDP
END
```

## 9.12 Java如何字符串处理及常用方法

在《[Java虚拟机](http://c.biancheng.net/view/3677.html)》一节中介绍了 [Java](http://c.biancheng.net/java/) 字节码，并说明了怎样将 java.class 文件反汇编为可读的字节码格式。本节将展示 Java 如何处理字符串，以及处理字符串的方法。

【示例】：寻址子串，下面的 Java 代码定义了一个字符串变量，其中包含了一个雇员 ID 和该雇员的姓氏。然后，调用 substring 方法将账号送入第二个字符串变量：

```c
String empInfo = "10034Smith";
String id = empInfo.substring(0,5);
```

对该 Java 代码反汇编，其字节码显示如下：

```text
ldc #32; //字符串 10034Smithastore_0aload_0iconst_0iconst_5invokevirtual #34; // Method java/lang/String.substringastore_1
```

现在分步研究这段代码，并加上自己的注释。ldc 指令把一个对字符串文本的引用从常量池加载到操作数栈。接着，astore\_0 指令从运行时堆栈弹出该字符串引用，并把它保存到局部变量 empInfo 中，其在局部变量区域中的索引为 0：

```text
ldc #32; //加载文本字符串：10034Smith
astore_0 //保存到 empInfo （索引 0）
```

接下来，aload\_0 指令把对 empInfo 的引用压入操作数栈：

```text
aload_0 //加载 empInfo 到堆栈
```

然后，在调用 substring 方法之前，它的两个参数（0 和 5）必须压入操作数栈。该操作由指令 iconst\_0 和 iconst\_5 完成：

```text
iconst_0
iconst_5
```

invokevirtual 指令调用 substring 方法，它的引用 ID 号为 34：

```text
invokevirtual #34; // Method java/lang/String.substring
```

substring 方法将参数弹出堆栈，创建新字符串，并将该字符串的引用压入操作数栈。其后的 astore\_1 指令把这个字符串保存到局部变量区域内索引 1 的位置，也就是变量 id 所在的位置：

```text
astore_1
```

