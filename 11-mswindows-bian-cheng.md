---
description: >-
  本章将为大家展示如何用 32 位 Microsoft Windows API 进行控制台窗口编程。应用编程接口 ( API : Application
  Programming Interface)
  是类型、常数和函数的集合体，它提供了一种用计算机代码操作对象的方式。通过本章还将学习到如何用事件处理循环来创建图形化窗口应用程序。虽然不建议用汇编语言进行扩展图形应用程序编程，但是本章将通过实例的形式
---

# 11 MS-Windows编程

## 11.1 MS-Windows编程简述

一个 Windows 应用程序开始的时候，要么创建一个控制台窗口，要么创建一个图形化窗口。本教程的项目文件一直把如下选项与 LINK 命令一起使用。它告诉链接器创建一个基于控制台的应用程序：

```text
/SUBSYSTEM:CONSOLE
```

控制台程序的外观和操作就像 MS-DOS 窗口，它的一些改进部分将在后面进行介绍。控制台有一个输入缓冲区以及一个或多个屏幕缓冲区：

#### 1\) 输入缓冲区（input buffer）：

包含一组输入记录（input records），其中的每个记录都是一个输入事件的数据。输入事件的例子包括键盘输入、鼠标点击，以及用户调整控制台窗口大小。

#### 2\) 屏幕缓冲区（screen buffer）：

是字符与颜色数据的二维数组，它会影响控制台窗口文本的外观。

### Win32 API 参考信息

#### 函数

在接下来的讲解中将介绍 Win32 API 函数的子集并给岀一些简单的例子。由于篇幅的限制，将不会涉及很多细节。如果想了解更多信息，请访问 Microsoft MSDN 网站（地址为：[www.msdn.microsoft.com](http://www.msdn.microsoft.com/)）。在搜索函数或标识符时，把 Filtered by 参数设置为 Platform SDK。

#### 常数

阅读 Win32 API 函数的文档时，常常会见到常量名，如 TIME\_ZONE\_ID\_UNKNOWN。少数情况下，这些常量已经在 SmallWin.inc 中定义过。例如，头文件 WinNT.h 就定义了 TIME\_ZONE\_ID\_UNKNOWN 及相关常量：

```text
#define TIME_ZONE_ID_UNKNOWN 0
#define TIME_ZONE_ID_STANDARD 1
#define TIME_ZONE_ID_DAYLIGHT 2
```

利用这个信息，可以将下述语句添加到 SmallWin.h 或者用户自己的头文件中:

```text
TIME_ZONE_ID_UNKNOWN = 0
TIME_ZONE_ID_STANDARD = 1
TIME_ZONE_ID_DAYLIGHT = 2
```

### 字符集和 Windows API 函数

调用 Win32 API 函数时会使用两类字符集：8 位的 ASCII/ANSI 字符集和 16 位的 Unicode 字符集（所有近期的 Windows 版本中都有）。

Win32 函数可以处理的文本通常有两种版本：一种以字母 A 结尾（8 位 ANSI 字符），另一种以 W 结尾（宽字符集，包括了 Unicode）。WriteConsole 即为其中之一：

```text
WriteConsoleA
WriteConsoleW
```

Windows95 和 98 不支持以 W 结尾的函数名。另一方面，在所有近期的 Windows 版本中，Unicode 都是原生字符集。例如调用名为 WriteConsoleA 的函数，则操作系统将把字符从 ANSI 转换为 Unicode，再调用 WriteConsoleW。

在 Microsoft MSDN 链接库的函数文件中，如 WriteConsole，尾字符 A 和 W 都被省略了。

```text
WriteConsole EQU <WriteConsoleA>
```

这个定义使得程序能按 WriteConsole 的通用名对其进行调用。

### 高级别和低级别访问

控制台访问有两个级别，这就能够在简单控制和完全控制之间进行权衡：

* 高级别控制台函数从控制台输入缓冲区读取字符流，并将字符数据写入控制台的屏幕缓冲区。输入和输出都可以重定向到文本文件。
* 低级别控制台函数检索键盘和鼠标事件，以及用户与控制台窗口交互 \( 拖曳、调整大小等 \) 的详细信息。这些函数还允许对窗口大小、位置以及文本颜色进行详细控制。

### Windows 数据类型

Win32 函数使用 C/[C++](http://c.biancheng.net/cplus/) 程序员的函数声明进行记录。在这些声明中，所有函数参数类型要么基于标准 C 类型，要么基于 MS-Windows 预定义类型 \(下表中列出了部分类型 \) 之一。

| MS-Windows 类型 | MASM类型 | 说明 |
| :--- | :--- | :--- |
| BOOL, BOOLEAN | DWORD | 布尔值 \(TRUE 或 FALSE\) |
| BYTE | BYTE | 8 位无符号整数 |
| CHAR | BYTE | 8 位 Windows ANSI 字符 |
| COLORREF | DWORD | 作为颜色值的 32 位数值 |
| DWORD | DWORD | 32 位无符号整数 |
| HANDLE | DWORD | 对象句柄 |
| HFILE | DWORD | 用 OpenFile 打开的文件的句柄 |
| INT | SDWORD | 32 位有符号整数 |
| LONG | SDWORD | 32 位有符号整数 |
| LPARAM | DWORD | 消息参数，由窗口过程和回调函数使用 |
| LPCSTR | PTR BYTE | 32 位指针，指向由 8 位 Windows \(ANSI\)字符组成的空字节结束的字符串常量 |
| LPCVOID | DWORD | 指向任何类型的常量 |
| LPSTR | PTR BYTE | 32 位指针，指向由 8 位 Windows \(ANSI\) 字符组成的空字节结束的字符串 |
| LPCTSTR | PTR WORD | 32 位指针，指向对 Unicode 和双字节字符集可移植的字符串常量 |
| LPTSTR | PTR WORD | 32 位指针，指向对 Unicode 和双字节字符集可移植的字符串 |
| LPVOID | DWORD | 32 位指针，指向未指定类 |
| LRESULT | DWORD | 窗口过程和回调函数返回的 32 位数值 |
| SIZE\_T | DWORD | 一个指针可以指向的最大字节数 |
| UNIT | DWORD | 32 位无符号整数 |
| WNDPROC | DWORD | 32 位指针，指向窗口过程 |
| WORD | WORD | 16 位无符号整数 |
| WPARAM | DWORD | 作为参数传递给窗口过程或回调函数的 32 位数值 |

区分数据值和指向值的指针是很重要的。以字母 LP 开头的类型名是长指针 \(long pointer\)，指向其他对象。

### SmallWin.inc 头文件

SmallWin.inc 是一个头文件，其中包含了 Win32 API 编程的常量定义、等价文本以及函数原型。通过本教程一直使用的 Irvine32.inc，SmallWin.inc 被自动包含在程序中。

大多数常量都可以在用于 C 和 C++ 编程的头文件 Windows.h 中找到。与它的名字不同，SmallWin.inc 文件相当大， 因此这里只展示其突出部分：

```text
DO_NOT_SHARE = 0
NULL = 0
TRUE = 1
FALSE = 0
;Win32 控制台句柄
STD_INPUT_HANDLE EQU -10
STD_OUTPUT_HANDLE EQU -11
STD_ERROR_HANDLE EQU -12
```

类型 HANDLE 是 DWORD 的代名词,能帮助函数原型与 Microsoft Win32 文档更加一致：

```text
HANDLE TEXTEQU <DWORD>
```

SmallWin.inc 也包括用于 Win32 调用的结构定义。下面给出了两个结构定义：

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

### 控制台句柄

几乎所有的 Win32 控制台函数都要求向其传递一个句柄作为第一个实参。句柄 \(handle\) 是 一个 32 位无符号整数，用于唯一标识一个对象，例如一个位图、画笔或任何输入/输岀设备：

```text
STD_INPUT_HANDLE standard input
STD_OUTPUT_HANDLE standard output
STD_ERROR_HANDLE standard error output
```

上述句柄中的后两个用于写控制台活跃屏幕缓冲区。

GetStdHandle 函数返回一个控制台流的句柄：输入、输出或错误输出。基于控制台的程序中所有的输入/输出操作都需要句柄。函数原型如下：

```text
GetStdHandle PROTO,
  nStdHandle:HANDLE    ;句柄类型
```

nStdHandle 可以是 STD_INPUT\_HANDLE、STD\_OUTPUT\_HANDLE 或者 STD\_ERROR_ HANDLE。函数用 EAX 返回句柄，且应将它复制给变量保存。下面是一个调用示例：

```text
.data
inputHandle HANDLE ?
.code
INVOKE GetStdHandle, STD_INPUT_HANDLE
mov inputHandle；eax
```

## 11.2 Win32控制台函数简述

下表为所有 Win32 控制台函数的一览表。在 www.msdn.microsoft.com 上可以找到 MSDN 库中每个函数的完整描述。

> 提示：Win32 API 函数不保存 EAX、EBX、ECX 和 EDX，因此程序员需自己完成这些寄存器的入栈和出栈操作。

| 函数 | 描述 |
| :--- | :--- |
| AllocConsole | 为调用进程分配一个新控制台 |
| CreateConsoleScreenBuffer | 创建控制台屏幕缓冲区 |
| ExitProcess | 结束进程及其所有线程 |
| FillConsoleOutputAttribute | 为指定数量的字符单元格设置文本和背景颜色属性 |
| FillConsoleOutputCharacter | 按指定次数将一个字符写入屏幕缓冲区 |
| FlushConsoleInputBuffer | 刷新控制台输入缓冲区 |
| FreeConsole | 将主调进程与其控制台分离 |
| GenerateConsoleCtrlEvent | 向控制台进程组发送指定信号，这些进程组共享与主调进程关联的控制台 |
| GetConsoleCP | 获取与主调进程关联的控制台使用的输入代码页 |
| GetConsoleCursorInfo | 获取指定控制台屏幕缓冲区光标大小和可见性的信息 |
| GetConsoleMode | 获取控制台输入缓冲区的当前输入模式或控制台屏幕缓冲区的当前输出模式 |
| GetConsoleOutputCP | 获取与主调进程关联的控制台使用的输出代码页 |
| GetConsoleScreenBufferInfo | 获取指定控制台屏幕缓冲区信息 |
| GetConsoleTitle | 获取当前控制台窗口的标题栏字符串 |
| GetConsoleWindow | 获取与主调进程关联的控制台使用的窗口句柄 |
| GetLargestConsoleWindowSize | 获取控制台窗口最大可能的大小 |
| GetNumberOfConsoleInputEvents | 获取控制台输入缓冲区中未读输入记录的个数 |
| GetNumberOfConsoleMouseButtons | 获取当前控制台使用的鼠标按钮数 |
| GetStdHandle | 获取标准输入、标准输出或标准错误设备的句柄 |
| HandlerRoutine | 与 SetConsoleCtrlHandler 函数一起使用的应用程序定义的函数 |
| PeekConsoleInput | 从指定控制台输入缓冲区读取数据，且不从缓冲区删除该数据 |
| ReadConsole | 从控制台输入缓冲区读取并删除输入字符 |
| ReadConsoleInput | 从控制台输入缓冲区读取并删除该数据 |
| ReadConsoleOutput | 从控制台屏幕缓冲区的矩形字符单元格区域读取字符和颜色属性数据 |
| ReadConsoleOutputAttribute | 从控制台屏幕缓冲区的连续单元格复制指定数量的前景和背景颜色属性 |
| ReadConsoleOutputCharacter | 从控制台屏幕缓冲区的连续单元格复制若干字符 |
| ScrollConsoleScreenBuffer | 移动屏幕缓冲区内的一个数据块 |
| SetConsoleActiveScreenBuffer | 设置指定屏幕缓冲区为当前显示的控制台屏幕缓冲区 |
| SetConsoleCP | 设置主调过程的控制台输入代码页 |
| SetConsoleCtrlHandler | 为主调过程从处理函数列表中添加或删除应用程序定义的 HandlerRoutine |
| SetConsoleCursorInfo | 设置指定控制台屏幕缓冲区光标的大小和可见度 |
| SetConsoleCursorPosition | 设置光标在指定控制台屏幕缓冲区中的位置 |
| SetConsoleMode | 设置控制台输入缓冲区的输入模式或者控制台屏幕缓冲区的输出模式 |
| SetConsoleOntputCP | 设置主调过程的控制台输出代码页 |
| SetConsoleScreenBufferSize | 修改指定控制台屏幕缓冲区的大小 |
| SetConsoleTextAttribute | 设置写入屏幕缓冲区的字符的前景（文本）和背景颜色属性 |
| SetConsoleTitle | 为当前控制台窗口设置标题栏字符串 |
| SetConsoleWindowInfo | 设置控制台屏幕缓冲区窗口当前的大小和位置 |
| SetStdHandle | 设置标准输入、输出和标准错误设备的句柄. |
| WriteConsole | 向由当前光标位置标识开始的控制台屏幕缓冲区写一个字符串 |
| WriteConsoleInput | 直接向控制台输入缓冲区写数据 |
| WriteConsoleOutput | 向控制台屏幕缓冲区内指定字符单元格的矩形块写字符和颜色属性数据 |
| WriteConsoleOutputAttribute | 向控制台屏幕缓冲区的连续单元格复制一组前景和背景颜色属性 |
| WriteConsoleOutputCharacter | 向控制台屏幕缓冲区的连续单元格复制一组字符 |

## 11.3 MessageBoxA函数：显示消息框

Win32 应用程序生成输岀的一个最简单的方法就是调用 MessageBoxA 函数：

```text
MessageBoxA PROTO,
  hWnd:DWORD,          ;窗口句柄（可以为空）
  lpText:PTR BYTE,         ;字符串，对话框内
  lpCaption:PTR BYTE,      ;字符串，对话框标题
  uType:DWORD          ;内容和行为
```

基于控制台的应用程序可以将 hWnd 设置为空，表示该消息框没有相关的包含窗口或父窗口。lpText 参数是指向空字节结束字符串的指针，该字符串将出现在消息框内。lpCaption 参数指向作为对话框标题的空字节结束字符串。uType 参数指定对话框的内容和行为。

#### 内容和行为

uType 参数包含的位图整数组合了三种选项：显示按钮、图标和默认按钮选择。几种可能的按钮组合如下：

* MB\_OK
* MB\_OKCANCEL
* MB\_YESNO
* MB\_YESNOCANCEL
* MB\_RETRYCANCEL
* MB\_ABORTRETRYIGNORE
* MB\_CANCELTRYCONTINUE

#### 默认按钮

可以选择按钮作为用户点击 Enter 键时的自动选项。选项包括 MB\_DEFBUTTON1（默认）、MB\_DEFBUTTON2、MB\_DEFBUTTON3 和 MB\_DEFBUTTON4。按钮从左到右，从 1 开始编号。

#### 图标

有四个图标可用。有时多个常数会产生相同的图标：

* 停止符：MB\_ICONSTOP. MB\_ICONHAND 或 MB\_ICONERROR
* 问号（?）：MB\_ICONQUESTION
* 信息符（i）：MB\_ICONINFORMATION、MB\_ICONASTERISK
* 感叹号（!）：MB\_ICONEXCLAMATION、MB\_ICONWARNING

#### 返回值

如果 MessageBoxA 失败，则返回零；否则，它将返回一个整数以表示用户在关闭对话框时点击的按钮。选项包括 IDABORT、IDCANCEL、IDCONTINUE、IDIGNORE、IDNO、IDOK、IDRETRY、IDTRYAGAIN，以及 IDYES。

Smallwin.inc 将 MessageBoxA 重定义为 MessageBox，这个名字看上去具有更强的用户友好性。

如果想要消息框窗口浮动于桌面所有其他窗口之上，就在传递的最后一个参数（uType 参数）值上添加 MB\_SYSTEMMODAL 选项。

#### 1\) 演示程序

下面将通过一个小程序来演示函数 MessageBoxA 的一些功能。第一个函数调用显示一条警告信息：

![img](http://c.biancheng.net/uploads/allimg/190522/4-1Z5221I135K3.gif)

第二个函数调用显示一个问号图标以及 Yes/No 按钮。如果用户选择 Yes 按钮，则程序利用返回值选择一个操作：

![img](http://c.biancheng.net/uploads/allimg/190522/4-1Z5221I2491M.gif)

第三个函数调用显示一个信息图标以及三个按钮：

![img](http://c.biancheng.net/uploads/allimg/190522/4-1Z5221I303609.gif)

第四个函数调用显示一个停止图标和一个 OK 按钮：

![img](http://c.biancheng.net/uploads/allimg/190522/4-1Z5221I31S05.gif)

#### 2\) 程序清单

MessageBoxA 演示程序的完整清单如下所示。函数 MessageBoxA 重命名为函数 MessageBox，这样就可以使用更加简单的函数名：

```text
; 演示 MessageBoxA
INCLUDE Irvine32.inc

.data
captionW        BYTE "Warning",0
warningMsg    BYTE "The current operation may take years "
                BYTE "to complete.",0

captionQ        BYTE "Question",0
questionMsg    BYTE "A matching user account was not found."
                BYTE 0dh,0ah,"Do you wish to continue?",0   

captionC        BYTE "Information",0
infoMsg        BYTE "Select Yes to save a backup file "
                BYTE "before continuing,",0dh,0ah
                BYTE "or click Cancel to stop the operation",0

captionH        BYTE "Cannot View User List",0
haltMsg        BYTE "This operation not supported by your "
                BYTE "user account.",0               

.code
main PROC

; 显示感叹号图标和 OK 按钮
    INVOKE MessageBox, NULL, ADDR warningMsg,
        ADDR captionW,
        MB_OK + MB_ICONEXCLAMATION

; 显示问号图标和 Yes/No 按钮
    INVOKE MessageBox, NULL, ADDR questionMsg,
        ADDR captionQ, MB_YESNO + MB_ICONQUESTION

    ; 解释用户点击的按钮  
    cmp    eax,IDYES        ; YES button clicked?

; 显示信息图标和 Yes/No/Cancel 按钮
    INVOKE MessageBox, NULL, ADDR infoMsg,
      ADDR captionC, MB_YESNOCANCEL + MB_ICONINFORMATION \
          + MB_DEFBUTTON2

; 显示停止图标和 OK 按钮
    INVOKE MessageBox, NULL, ADDR haltMsg,
        ADDR captionH,
        MB_OK + MB_ICONSTOP

    exit
main ENDP
END main
```

## 11.4 ReadConsole函数：读取文本输入并将其送入缓冲区

函数 ReadConsole 为读取文本输入并将其送入缓冲区提供了便捷的方法。其原型如下所示：

```text
ReadConsole PROTO,
  hConsoleInput: HANDLE z            ;输入句柄
  lpBuffer:PTR BYTE,                  ;缓冲区指针
  nNumberOfCharsToRead:DWORD,     ;读取的字符数
  lpNumberOfCharsRead:PTR DWORD,   ;指向读取字节数的指针
  lpReserved:DWORD                 ;未使用
```

hConsoleInput 是函数 GetStdHandle 返回的可用控制台输入句柄。参数 lpBuffer 是字符数组的偏移量。nNumberOfCharsToRead 是一个 32 位整数，指明读取的最大字符数。lpNumberOfCharsRead 是一个允许函数填充的双字指针，当函数返回时，字符数的计数值将被放入缓冲区。最后一个参数未使用，因此传递的值为 0。

在调用 ReadConsole 时，输入缓冲区还要包含两个额外的字节用来保存行结束字符。如果希望输入缓冲区里是空字节结束字符串，则用空字节来代替内容为 ODh 的字节。Irvine32.lib 的过程 ReadString 就是这样操作的。

> 注意：Win32 API 函数不会保存 EAX、EBX、ECX 和 EDX 寄存器。

【示例】要读取用户输入的字符，就调用 GetStdHandle 来获得控制台标准输入句柄，再使用该句柄调用 ReadConsoleo 下面的 ReadConsole 程序演示了这个方法。

> 提示：Win32 API 调用与 Irvine32 链接库兼容，因此在调用 Win32 函数的同时还可以调用 DumpRegs

```text
; 从控制台读取    (ReadConsole.asm)

INCLUDE Irvine32.inc
BufSize = 80

.data
buffer BYTE BufSize DUP(?),0,0
stdInHandle HANDLE ?
bytesRead   DWORD ?

.code
main PROC
    ; 获取标准输入句柄
    INVOKE GetStdHandle, STD_INPUT_HANDLE
    mov    stdInHandle,eax

    ; 等待用户输入
    INVOKE ReadConsole, stdInHandle, ADDR buffer,
      BufSize, ADDR bytesRead, 0

    ; 显示缓冲区
    mov    esi,OFFSET buffer
    mov    ecx,bytesRead
    mov    ebx,TYPE buffer
    call    DumpMem

    exit
main ENDP
END main
```

如果用户输入 “abcdefg”，程序将生成如下输出。缓冲区会插入 9 个字节：“abcdefg” 再加上 ODh 和 OAh，即用户按下 Enter 键时产生的行结束字符。bytesRead 等于 9：

![img](http://c.biancheng.net/uploads/allimg/190522/4-1Z5221KU6107.gif)

## 11.5 GetLastError和FormatMessage函数：获取错误信息

若 Windows API 函数返回了错误值 \( 如 NULL\)，则可以调用 API 函数 GetLastError 来获取该错误的更多信息。该函数用 EAX 返回 32 位整数错误码：

```text
.data
messageId DWORD ?
.code
call GetLastError
mov messageId,eax
```

MS-Windows 有大量的错误码，因此，程序员可能希望得到一个消息字符串来对错误进行说明。要想达到这个目的，就调用函数 FormatMessage：

```text
FormatMessage PROTO,     ;格式化消息
  dwFlags:DWORD,        ;格式化选项
  lpSource:DWORD,        ;消息定义的位置
  dwMsgID:DWORD,       ;消息标识符
  dwLanguageID:DWORD,   ;语言标识符
  lpBuffer:PTR BYTE,        ;缓冲区接收字符串指针
  nSize:DWORD,           ;缓冲区大小
  va_list: DWORD          ;参数列表指针
```

该函数的参数有点复杂，程序员需要阅读 SDK 文档来了解全部信息。下面简要列出了最常用的参数值。除了 lpBuffer 是输出参数外，其他都是输入参数：

#### 1\) dwFlags

保存格式化选项的双字整数，包括如何解释参数 lpSource。它规定怎样处理换行，以及格式化输出行的最大宽度。建议值为 FORMAT\_MESSAGE\_ALLOCATE\_BUFFER 和 FORMAT\_MESSAGE\_FROM\_SYSTEM。

#### 2\) lpSource

消息定义位置的指针。若按照建议值设置 dwFlags，则 lpSource 设置为 NULL\(0\)。

#### 3\) dwMsgID

调用 GetLastError 后返回的双字整数。

#### 4\) dwLanguageID

语言标识符。若将其设置为 0，则消息为语言无关，否则将对应于用户的默认语言环境。

#### 5\) lpBuffer\( 输出参数 \)

接收空字节结束消息字符串的缓冲区指针。如果使用了 FORMAT\_MESSAGE\_ALLOCATE\_BUFFER 选项，则会自动分配缓冲区。

#### 6\) nSize

用于指定一个缓冲区来保存消息字符串。如果 dwFlags 使用了上述建议选项，则该参数可以设置为 0。

#### 7\) va\_list

数组指针，该数组包含了可以插入到格式化消息的值。由于没有格式化错误消息，这个参数可以为 NULL\(0\)。

FormatMessage 的示例调用如下：

```text
.data
messageId DWORD ?
pErrorMsg DWORD ?            ;指向错误消息
.code
call GetLastError
mov messageId,eax
INVOKE FormatMessage, FORMAT_MESSAGE_ALLOCATE_BUFFER + \
    FORMAT_MESSAGE_FROM_SYSTEM, NULL, messagelD, 0,
    ADDR pErrorMsg, 0, NULL
```

调用 FormatMessage 后，再调用 LocalFree 来释放由 FormatMessage 分配的存储空间：

```text
INVOKE LocalFree, pErrorMsg
```

### WriteWindowsMsg

Irvine32 链接库有如下 WriteWindowsMsg 程，它封装了消息处理的细节：

```text
;----------------------------------------------------
WriteWindowsMsg PROC USES eax edx
;
; 显示包含 MS-Windows 最新生成的错误字符串
; 接收: 无
; 返回: 无
;----------------------------------------------------
.data
WriteWindowsMsg_1 BYTE "Error ",0
WriteWindowsMsg_2 BYTE ": ",0
pErrorMsg DWORD ?              ; 指向错误消息
messageId DWORD ?
.code
    call    GetLastError
    mov    messageId,eax

; 显示错误号
    mov    edx,OFFSET WriteWindowsMsg_1
    call    WriteString
    call    WriteDec    ; show error number
    mov    edx,OFFSET WriteWindowsMsg_2
    call    WriteString

; 获取相应的消息字符串
    INVOKE FormatMessage, FORMAT_MESSAGE_ALLOCATE_BUFFER + \
      FORMAT_MESSAGE_FROM_SYSTEM, NULL, messageID, NULL,
      ADDR pErrorMsg, NULL, NULL

; 显示 MS-Windows 生成的错误消息
    mov    edx,pErrorMsg
    call    WriteString

; 释放错误消息字符串的空间
    INVOKE LocalFree, pErrorMsg

    ret
WriteWindowsMsg ENDP
```

## 11.6 单字符输入简述

控制台模式下的单字符输入有些复杂。MS-Windows 为当前安装的键盘提供了驱动器。当一个键被按下时，一个 8 位的扫描码 \(scan code\) 就被传递到计算机的键盘端口。当这个键被释放时，就会传递第二个扫描码。

MS-Windows 利用设备驱动程序将扫描码转换为 16 位的虚拟键码 \(virtual-key code\)，即 MS-Windows 定义的用于标识按键用途的与设备无关数值。MS-Windows 生成含有扫描码、虚拟键码和其他信息的消息。这个消息放在 MS-Windows 消息队列中，并最终进入当前执行程序线程（由控制台输入句柄标识）。

如果想要进一步了解键盘输入过程，请参阅 Platform SDK 文档中的 About Keyboard Input 主题。虚拟键常数列表位于本教程 \Examples\chll 目录下的 VirtualKeys.inc 文件中。

Irvine32 键盘过程 Irvine32 链接库由两个相关过程：

* ReadChar：等待键盘输入一个 ASCII 字符，并用 AL 返回该字符。
* ReadKey：过程执行无等待键盘检查。如果控制台输入缓冲区中没有等待的按键，则零标志位置 1。如果发现有按键，则零标志位清零且 AL 等于零或 ASCII 码。EAX 和 EDX 的高 16 位被覆盖。

如果 ReadKey 过程中的 AL 等于 0，那么用户可能按下了特殊键（功能键、光标箭头等）。AH 寄存器为键盘扫描码。DX 为虚拟键码，EBX 为键盘控制键状态信息。

下表为控制键值列表。调用 ReadKey 之后，可以用 TEST 指令检查各种键值。

| 值 | 含义 | 值 | 含义 |
| :--- | :--- | :--- | :--- |
| CAPSLOCK\_ON | CAPSLOCK 指示灯亮 | RIGHT\_ALT\_PRESSED | 右 ALT 键被按下 |
| ENHANCED\_KEY | 被按下增强的 | RIGHT\_CTRL\_PRESSED | 右 CTRL 键被按下 |
| LEFT\_ALT\_PRESSED | 该键是左 ALT 键 | SCROLLLOCL\_ON | SCROLLLOCK 指示灯亮 |
| LEFT\_CTRL\_PRESSED | 左 CTRL 键被按下 | SHIFT\_PRESSED | SHIFT 键被按下 |
| NUMLOCK\_ON | NUMLOCK 指示灯亮 |  |  |

### ReadKey 测试程序

下面是 ReadKey 测试程序：等待一个按键，然后报告按下的是否为 CapsLock 键。程序应考虑延迟因素，以便在调用 ReadKey 时留出时间让 MS-Windows 处理其消息循环：

```text
; 测试 ReadKey    ( TestReadkey. asm)

INCLUDE Irvine32.inc
INCLUDE Macros.inc

.code
main PROC

L1: mov    eax,10             ; 消息处理带来的延迟
    call    Delay
    call    ReadKey           ; 等待按键
    jz    L1

    test    ebx,CAPSLOCK_ON   
    jz    L2
    mWrite <"CapsLock is ON",0dh,0ah>
    jmp    L3

L2:    mWrite <"CapsLock is OFF",0dh,0ah>

L3:    exit
main ENDP
END main
```

## 11.7 GetKeyState函数：获得键盘状态

通过测试单个键盘按键可以发现当前按下的是哪个键。方法是调用 API 函数 GetKeyState。

```text
GetKeyState PROTO, nVirtKey:DWORD
```

向该函数传递如下表所示的虚拟键值。测试程序必须按照同一个表来测试 EAX 里面的返回值。

| 按键 | 虚拟键符号 | EAX 中被测试的位 |
| :--- | :--- | :--- |
| NumLock | VK\_NUMLOCK | 0 |
| Scroll Lock | VK\_SCROLL | 0 |
| Left Shift | VK\_LSHIFT | 15 |
| Right Shift | VK\_tRSHIFT | 15 |
| Left Ctrl | VK\_LCONTROL | 15 |
| Right Ctrl | VK\_RCONTROL | 15 |
| Left Menu | VK\_LMENU | 15 |
| Right Menu | VK\_RMENU | 15 |

下面的测试程序通过检查 NumLock 键和左 Shift 键的状态来演示 GetKeyState 函数：

```text
; 键盘切换键    （Keybd.asm）
INCLUDE Irvine32.inc
INCLUDE Macros.inc
; 如果当前触发了切换键 (CapsLock, NumLock, ScrollLock)，
; 则 GetKeyState 将 EAX 的位 0 置 1
; 如果当前按下了特殊键，则将 EAX 的最高位置 1

.code
main PROC

    INVOKE GetKeyState, VK_NUMLOCK
    test al,1
    .IF !Zero?
      mWrite <"The NumLock key is ON",0dh,0ah>
    .ENDIF

    INVOKE GetKeyState, VK_LSHIFT
    call DumpRegs
    test eax,80000000h
    .IF !Zero?
      mWrite <"The Left Shift key is currently DOWN",0dh,0ah>
    .ENDIF

    exit
main ENDP
END main
```

## 11.8 WriteConsole和WriteConsoleOutputCharacter函数：控制台输出

本节将为大家讲解如何直接调用 Win32 函数在控制台输出，如 WriteConsole 和 WriteConsoleOutputCharacter。直接调用要求了解更多细节，但是它也提供了比 Irvine32 链接库过程更大的灵活性。

## [数据结构](http://c.biancheng.net/data_structure/)

有些 Win32 控制台函数使用的是预定义的数据结构，包括 COORD 和 SMALL\_RECT。COORD 结构包含的是控制台屏幕缓冲区内字符单元格的坐标。坐标原点（0, 0）位于左上角单元格：

```text
COORD STRUCT
  X WORD ?
  Y WORD ?
COORD ENDS
```

SMALL\_RECT 结构包含的是矩形的左上角和右下角，它指定控制台窗口中的屏幕缓冲区字符单元格区域：

```text
SMALL_RECT STRUCT
  Left WORD ?
  Top WORD ?
  Right WORD ?
  Bottom WORD ?
SMALL_RECT ENDS
```

### WriteConsole 函数

函数 WriteConsole 在控制台窗口的当前光标所在位置写一个字符串，并将光标留着字符串末尾右边的字符位置上。它按照标准 ASCII 控制字符操作，比如制表符、回车和换行。

字符串不一定以空字节结束。函数原型如下：

```text
WriteConsole PROTO,
  hConsoleOutput:HANDLE,
  lpBuffer:PTR BYTE,
  nNumberOfCharsToWrite:DWORD,
  lpNumberOfCharsWritten:PTR DWORD,
  lpReserved:DWORD
```

hConsoleOutput 是控制台输出流句柄；lpBuffer 是输出字符数组的指针；nNumberOfCharsToWrite 是数组长度；lpNumberOfCharsWritten 是函数返回时实际输出字符数量的整数指针。最后一个参数未使用，因此将其设置为 0。

### 示例程序：Console1

下面的程序通过向控制台窗口写字符串演示了函数 GetStdHandle、ExitProcess 和 WriteConsole：

```text
; Win32 控制台示例 #1    (Consolel.asm)
; 本程序调用如下 Win32 控制台函数:
; GetStdHandle, ExitProcess, WriteConsole
INCLUDE Irvine32.inc

.data
endl EQU <0dh,0ah>            ; 行结尾

message LABEL BYTE
    BYTE "This program is a simple demonstration of "
    BYTE "console mode output, using the GetStdHandle "
    BYTE "and WriteConsole functions.", endl
messageSize DWORD ($-message)

consoleHandle HANDLE 0     ; 标准输出设备句柄
bytesWritten  DWORD ?      ; 输出字节数

.code
main PROC
  ; 获得控制台输出句柄
    INVOKE GetStdHandle, STD_OUTPUT_HANDLE
    mov consoleHandle,eax

  ; 向控制台写一个字符串
    INVOKE WriteConsole,
      consoleHandle,          ; 控制台输出句柄
      ADDR message,           ; 字符串指针
      messageSize,            ; 字符长度
      ADDR bytesWritten,      ; 返回输出字节数
      0                       ; 未使用

    INVOKE ExitProcess,0
main ENDP
END main
```

程序生成输出如下所示：

![img](http://c.biancheng.net/uploads/allimg/190523/4-1Z523130322348.gif)

### WriteConsoleOutputCharacter 函数

函数 WriteConsoleOutputCharacter 从指定位置开始，向控制台屏幕缓冲区的连续单元格内复制一组字符。原型如下：

```text
WriteConsoleOutputCharacter PROTO,
  hConsoleOutput:HANDLE,             ;控制台输出句柄
  lpCharacter :PTR BYTE,                ;缓冲区指针
  nLength: DWORD,                   ;缓冲区大小
  dwWriteCoord: COORD,               ;第一个单元格的坐标
  lpNumberOfCharsWritten: PTR DWORD  ;输出计数器
```

如果文本长度超过了一行，字符就会输岀到下一行。屏幕缓冲区的属性值不会改变。如果函数不能写字符，则返回零。ASCII 码，如制表符、回车和换行，会被忽略。

## 11.9 CreateFile函数：创建新文件或者打开已有文件

函数 CreateFile 可以创建一个新文件或者打开一个已有文件。如果调用成功，函数返回打开文件的句柄；否则，返回特殊常数 INVALID\_HANDLE\_VALUEO 原型如下：

```text
CreateFile PROTO,                ;创建新文件
  lpFilename:PTR BYTE,            ;文件名指针
  dwDesiredAccess:DWORD,       ;访问模式
  dwShareMode:DWORD,         ;共享模式
  lpSecurityAttributes:DWORD,     ;安全属性指针
  dwCreationDisposition:DWORD,   ;文件创建选项
  dwFlagsAndAttributes:DWORD,   ;文件属性
  hTemplateFile:DWORD          ;文件模板句柄
```

下表对参数进行了说明。如果函数调用失败则返回值为零。

| 参数 | 说明 |
| :--- | :--- |
| lpFileName | 指向一个空字节结束字符串，该串为部分或全部合格的文件名（drive:\path\filename） |
| dwDesiredAccess | 指定文件访问方式（读或写） |
| dwShareMode | 控制多个程序对打开文件的访问能力 |
| lpSecurityAttributes | 指向安全结构，该结构控制安全权限 |
| dwCreationDisposition | 指定文件存在或不存在时的操作 |
| dwFlagsAndAttributes | 包含位标志指定文件属性，如存档、加密、隐藏、普通、系统和临时 |
| hTemplateFile | 包含一个可选的文件模板句柄，该文件为已创建的文件提供文件属性和扩展属性；如果不使用该参数，就将其设置为 0 |

#### dwDesiredAccess

参数 dwDesiredAccess 允许指定对文件进行读访问、写访问、读/写访问，或者设备查询访问。可以从下表列出的值中选择，也可以从表中未列出的更多特定标志值选择。

| 值 | 含义 |
| :--- | :--- |
| 0 | 为对象指定设备查询访问。应用程序可以查询设备属性而无需访问设备，也可以检查文件是否存在 |
| GENERIC\_READ | 为对象指定读访问。可以从文件中读取数据，文件指针可以移动。与 GENERIC\_WRITE 一起使用为读/写访问 |
| GENERIC\_WRITE | 对对象指定写访问。可以向文件中写入数据，文件指针可以移动。与 GENERIC\_READ 一起使用为读/写访问 |

#### dwCreationDisposition

参数 dwCreationDisposition 指定当文件存在或不存在时应采取怎样的操作。可从下表中选择一个值。

| 值 | 含义 |
| :--- | :--- |
| CREATE\_NEW | 创建一个新文件。要求将参数 dwDesiredAccess 设置为 GENERIC\_WRITE。如果文件已经存在，则函数调用失败 |
| CREATE\_ALWAYS | 创建一个新文件。如果文件已存在，则函数会覆盖原文件，清除现有属性，并合并文件 属性与预定义的常数 FILE\_ATTRIBUTES\_ARCHIVE 中属性参数指定的标志。要求将参数 dwDesiredAccess 设置为 GENERIC WRITE |
| OPEN\_EXISTING | 打开文件。如果文件不存在，则函数调用失败。可用于读取和/或写入文件 |
| OPEN\_ALWAYS | 如果文件存在，则打开文件。如果不存在，则函数创建文件，就好像CreateDisposition 的值为 CREATE NEW |
| TRUNCATE\_EXISTING | 打开文件。一旦打开，文件将被截断，使其大小为零。要求将参数 dwDesiredAccess 设置为 GENERIC\_WRITE。如果文件不存在，则函数调用失败 |

下表列出了参数 dwFlagsAndAttributes 比较常用的值。（完整列表请在 Microsoft 在线文档中搜索CreateFiko）允许任意属性组合，除了 FILE\_ATTRIBUTE\_NORMAL 会被其他 所有属性覆盖。这些值能映射为 2 的幂，因此可以用汇编时 OR 运算符或 + 运算符将它们组 合为一个参数：

```text
FILE_ATTRIBUTE_HIDDEN OR FILE_ATTRIBUTE_READONLY
FILE_ATTRIBUTE_HIDDEN + FILE_ATTRIBUTE_READONLY
```

| 属性 | 含义 |
| :--- | :--- |
| FILE\_ATTRIBUTE\_ARCHIVE | 文件存档。应用程序使用这个属性标记文件以便备份或移动 |
| FILE\_ATTRIBUTE\_HIDDEN | 文件隐藏。不包含在普通目录列表中 |
| FILE\_ATTRIBUTE\_NORMAL | 文件没有其他属性设置。该属性只在单独使用时有效 |
| FILE\_ATTRIBUTE\_READONLY | 文件只读。应用程序可以读文件但不能写或删除文件 |
| FILE\_ATTRIBUTE\_TEMPORARY | 文件被用于临时存储 |

【示例】下面的例子仅具说明性，展示了如何创建和打开文件。请参阅在线从 Microsoft文 档，了解 CreateFile 更多可用选项：

打开并读取（输入）已存在文件：

```text
INVOKE CreateFile,
    ADDR filename,            ;文件名指针
    GENERIC_READ,             ;读文件
    DO_NOT_SHARE,             ;共享模式
    NULL,                     ;安全属性指针
    OPEN_EXISTING,            ;打开已存在文件
    FILE_ATTRIBUTE_NORMALA    ;普通文件属性
    0                         ;未使用
```

打开并写入（输出）已存在文件。文件打开后，可以通过写入覆盖当前数据，或者将文件指针移到末尾，向文件添加新数据（参见11.1.6节的SetFilePointer）：

```text
INVOKE CreateFile,
    ADDR filename,
    GENERIC_WRITEZ,      ;写文件
    DO_NOT_SHARE,
    NULL,
    OPEN_EXISTIN,       ;文件必须存在
    FILE_ATTRIBUTE_NORMAL,
    0
```

创建有普通属性的新文件，并删除所有已存在的同名文件：

```text
INVOKE CreateFile,
    ADDR filename,
    GENERIC_WRITE,       ;写文件
    DO _NOT_SHARE,
    NULL,
    CREATE_ALWAYS,       ;覆盖已存在的文件
    FILE_ATTRIBUTE_NORMAL,
    0
```

若文件不存在，则创建文件；否则打开并输出现有文件：

```text
INVOKE CreateFile,
    ADDR filename,
    GENERIC_WRITE,         ;写文件
    DO_NOT_SHARE,
    NULL,
    CREATE_NEW,            ;不删除已存在文件
    FILE_ATTRIBUTE_NORMAL,
    0
```

## 11.10 CloseHandle函数：关闭一个打开的对象句柄

函数 CloseHandle 关闭一个打开的对象句柄。其原型如下：

```text
CloseHandle PROTO,
  hObject: HANDLE ;对象句柄
```

可以用 CloseHandle 关闭当前打开的文件句柄。如果函数调用失败，则返回值为零。

## 11.11 ReadFile函数：从输入文件中读取文本

函数 ReadFile 从输入文件中读取文本。其原型如下：

```text
ReadFile PROTO,
  hFile:HANDLE,                      ;输入句柄
  lpBuffer:PTR BYTE,                   ;缓冲区指针
  nNumberOfBytesToRead:DWORD,      ;读取的字节数
  lpNumberOfBytesRead:PTR DWORD,    ;实际读出的 字节数
  lpOverlapped:PTR DWORD            ;异步信息指针
```

其中：

* hFile 是由 CreateFile 返回的打开文件的句柄；
* lpBuffer 指向的缓冲区接收从该文件读取的数据；
* nNumberOfBytesToRead 定义从该文件读取的最大字节数；
* lpNumberOfBytesRead 指向的整数为函数返回时实际读取的字节数；
* lpOverlapped 应被设置为 NULL\(0\)。若函数调用失败，则返回值为零。

如果对同一个打开文件的句柄进行多次调用，那么 ReadFile 就会记住最后一次读取的位置，并从这个位置开始读。换句话说，函数有一个内部指针指向文件内的当前位置。

ReadFile 还可以运行在异步模式下，这意味着调用程序不用等到读操作完成。

## 11.12 WriteFile函数：向文件写入数据

函数 WriteFile 用输出句柄向文件写入数据。句柄可以是屏幕缓冲区句柄，也可以是分配给文本文件的句柄。函数从文件内部位置指针所指向的位置开始写数据。

写操作完成后，文件位置指针按照实际写入的字节数进行调整。函数原型如下：

```text
WriteFile PROTO,
  hFile:HANDLE,                      ;输出句柄
  lpBuffer:PTR BYTE,                   ;缓冲区指针
  nNumberOfBytesToWrite:DWORD,      ;缓冲区大小
  lpNumberOfBytesWritten:PTR DWORD,  ;写入字节数
  lpOverlapped:PTR DWORD            ;异步信息指针
```

其中：

* hFile 是已打开文件的句柄；
* lpBuffer 指向的缓冲区包含了写入到文件的数据；
* nNumberOfBytesToWrite 指定向文件写入多少字节；
* lpNumberOfBytesWritten 指向的整数为函数执行后实际写入的字节数；
* 若为同步操作，则 lpOverlapped 应被设置为 NULL。若函数调用失败，则返回值为零。

## 11.13 SetFilePointer函数：移动打开文件的位置指针

函数 SetFilePointer 移动打开文件的位置指针。该函数可以用于向文件添加数据，或是执行随机访问记录处理：

```text
SetFilePointer PROTO,
  hFile:HANDLE,                     ;文件句柄
  lpDistanceToMove:SDWORD,         ;指针移动 字节数
  lpDistanceToMoveHigh:PTR SDWORD,  ;指针移动字节数，高双字
  dwMoveMethod:DWORD            ;起点
```

若函数调用失败，则返回值为零。dwMoveMode 指定文件指针移动的起点，选择项为 3 个预定义符号：FILE\_BEGIN、FILE\_CURRENT 和 FILE\_END。

移动距离本身为 64 位有符号整数值，分为两个部分：

* lpDistanceToMove：低 32 位
* lpDistanceToMoveHigh：含有高 32 位的变量指针

如果 lpDistanceToMoveHigh 为空，则只用 lpDistanceToMove 的值来移动文件指针。例如，下面的代码准备添加到一个文件末尾：

```text
INVOKE SetFilePointer,
  fileHandle,    ;文件句柄
  0,           ;距离低32位
  0,           ;距离高32位
  FILE_END     ;移动模式
```

## 11.14 Irvine32链接库文件I/O（输入/输出）

Irvine32 库中包含了一些简化的文件 I/O 过程。这些过程已经封装到本章描述的 Win32 API 函数中。

下面的源代码就给岀了 CreateOutputFile、OpenFile、WriteToFile、ReadFromFile 和 CloseFile：

```text
;------------------------------------------------------
CreateOutputFile PROC
;
; 创建一个新文件并以输出模式打开
; 接收: EDX 指向文件名
; 返回: 如果文件创建成功, EAX 包含一个有效的文件句柄。 
; 否则，EAX 等于 INVALID_HANDLE_VALUE
;------------------------------------------------------
    INVOKE CreateFile,
      edx, GENERIC_WRITE, DO_NOT_SHARE, NULL,
      CREATE_ALWAYS, FILE_ATTRIBUTE_NORMAL, 0
    ret
CreateOutputFile ENDP

;-------------------------------------------------------
OpenFile PROC
;打开一个新的文本文件进行输入。
;接收：EDX 指向文件名。
;返回：如果文件打开成功，EAX 包含一个有效的文件
;句柄。否则，EAX 等于 INVALID_HANDLE_VALUE。
;-------------------------------------------------------
    INVOKE CreateFilez
        edx, GENERIC_READ, DO_NOT_SHARE, NULL,
        OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, 0
    ret
OpenFile ENDP

;--------------------------------------------------------
WriteToFile PROC
;
; 将缓冲区内容写入一个输出文件
; 接收: EAX = 文件句柄, EDX = 缓冲区偏移量,
;    ECX = 写入字节数
; 返回: EAX = 实际写入文件的字节数
; 如果 EAX 返回的值小于 ECX 中的参数， 则可能发生错误
;--------------------------------------------------------
.data
WriteToFile_1 DWORD ?        ; 已写入字节数
.code
    INVOKE WriteFile,        ; 向文件写缓冲区
        eax,                 ; 文件句柄
        edx,                 ; 缓冲区指针
        ecx,                 ; 写入字节数
        ADDR WriteToFile_1,  ; 已写入字节数
        0                    ; 覆盖执行标志
    mov    eax,WriteToFile_1 ; 返回值
    ret
WriteToFile ENDP

;--------------------------------------------------------
ReadFromFile PROC
; 将一个输入文件读入缓冲区
; 接收: EAX = 文件句柄, EDX = 缓冲区偏移量,
;    ECX = 读字节数
; 返回: 如果 CF=0，EAX = 已读字节数
; 如果 CF=1，则EAX包含Win32 API 函数 GetLastError 返回的系统错误码
;--------------------------------------------------------
.data
ReadFromFile_1 DWORD ?            ; 已读字节数
.code
    INVOKE ReadFile,
        eax,                      ; 文件句柄
        edx,                      ; 缓冲区指针
        ecx,                      ; 读取的最大字节数
        ADDR ReadFromFile_1,      ; 已读字节数
        0                         ; 覆盖执行标志
    mov    eax,ReadFromFile_1
    ret
ReadFromFile ENDP

;--------------------------------------------------------
CloseFile PROC
; 使用句柄为标识符关闭一个文件
; 接收: EAX = 文件句柄
; 返回: EAX = 非 0，如果文件被成功关闭
;--------------------------------------------------------
    INVOKE CloseHandle, eax
    ret
CloseFile ENDP
```

## 11.15 实例：文件I/O（输入/输出）过程

下面通过两个实例程序来演示文件I/O（输入/输出）的过程。

#### 1\) CreatFile 程序示例

下面的程序用输岀模式创建一个文件，要求用户输入一些文本，将这些文本写到输出文件，并报告已写入的字节数，然后关闭文件。在试图创建文件后，程序要进行错误检查：

```text
; 创建一个文件    (CreateFile.asm)

INCLUDE Irvine32.inc 

BUFFER_SIZE = 501
.data
buffer BYTE BUFFER_SIZE DUP(?)
filename     BYTE "output.txt",0
fileHandle   HANDLE ?
stringLength DWORD ?
bytesWritten DWORD ?
str1 BYTE "Cannot create file",0dh,0ah,0
str2 BYTE "Bytes written to file [output.txt]: ",0
str3 BYTE "Enter up to 500 characters and press "
     BYTE "[Enter]: ",0dh,0ah,0

.code
main PROC
; 创建一个新文本文件
    mov    edx,OFFSET filename
    call    CreateOutputFile
    mov    fileHandle,eax

; 错误检查
    cmp    eax, INVALID_HANDLE_VALUE      ; 发现错误?
    jne    file_ok                        ; 否: 跳过
    mov    edx,OFFSET str1                ; 显示错误
    call    WriteString
    jmp    quit
file_ok:

; 提示用户输入字符串
    mov    edx,OFFSET str3                 ; "Enter up to ...."
    call    WriteString
    mov    ecx,BUFFER_SIZE                 ; 输入字符串
    mov    edx,OFFSET buffer
    call    ReadString
    mov    stringLength,eax                ; 计算输入字符数

; 将缓冲区写入输出文件
    mov    eax,fileHandle
    mov    edx,OFFSET buffer
    mov    ecx,stringLength
    call    WriteToFile
    mov    bytesWritten,eax                ; 保存返回值
    call    CloseFile

; 显示返回值
    mov    edx,OFFSET str2                 ; "Bytes written"
    call    WriteString
    mov    eax,bytesWritten
    call    WriteDec
    call    Crlf

quit:
    exit
main ENDP
END main
```

#### 2\) ReacIFile 程序示例

下面的程序打开一个文件进行输入，将文件内容读入缓冲区，并显示该缓冲区。所有过程都从 Irvine32 链接库调用：

```text
; 读文件      (ReadFile.asm)
; 使用 Irvine32.lib 的过程打开，读取并显示一个文本文件

INCLUDE Irvine32.inc
INCLUDE macros.inc
BUFFER_SIZE = 5000

.data
buffer BYTE BUFFER_SIZE DUP(?)
filename    BYTE 80 DUP(0)
fileHandle  HANDLE ?

.code
main PROC

; 用户输入文件名
    mWrite "Enter an input filename: "
    mov    edx,OFFSET filename
    mov    ecx,SIZEOF filename
    call    ReadString

; 打开文件进行输入
    mov    edx,OFFSET filename
    call    OpenInputFile
    mov    fileHandle,eax

; 错误检查
    cmp    eax,INVALID_HANDLE_VALUE           ; 错误打开文件?
    jne    file_ok                            ; 否: 跳过
    mWrite <"Cannot open file",0dh,0ah>
    jmp    quit                               ; 退出
file_ok:

; 将文件读入缓冲区
    mov    edx,OFFSET buffer
    mov    ecx,BUFFER_SIZE
    call    ReadFromFile
    jnc    check_buffer_size                ; 错误读取?
    mWrite "Error reading file. "           ; 是: 显示错误消息
    call    WriteWindowsMsg
    jmp    close_file

check_buffer_size:
    cmp    eax,BUFFER_SIZE                    ; 缓冲区足够大?
    jb    buf_size_ok                         ; 是
    mWrite <"Error: Buffer too small for the file",0dh,0ah>
    jmp    quit                               ; 退出

buf_size_ok:   
    mov    buffer[eax],0                    ; 插入空结束符
    mWrite "File size: "
    call    WriteDec                        ; 显示文件大小
    call    Crlf

; 显示缓冲区
    mWrite <"Buffer:",0dh,0ah,0dh,0ah>
    mov    edx,OFFSET buffer                ; 显示缓冲区
    call    WriteString
    call    Crlf

close_file:
    mov    eax,fileHandle
    call    CloseFile

quit:
    exit
main ENDP
END main
```

如果文件不能打开，则程序报告错误：

![img](http://c.biancheng.net/uploads/allimg/190523/4-1Z5231AR5I6.gif)

如果程序不能从文件读取，则报告错误。比如，假设有一个错误为在读文件时使用了不正确的文件句柄：

![img](http://c.biancheng.net/uploads/allimg/190523/4-1Z5231AT6251.gif)

缓冲区可能太小，无法容纳文件：

![img](http://c.biancheng.net/uploads/allimg/190523/4-1Z5231AZ4122.gif)

## 11.16 控制台窗口操作

Win32 API 提供了对控制台窗口及其缓冲区相当大的控制权。下图显示了屏幕缓冲区可以大于控制台窗口当前显示的行数。控制台窗口就像是一个“视窗”，显示部分缓冲区。

![&#x5C4F;&#x5E55;&#x7F13;&#x51B2;&#x533A;&#x548C;&#x63A7;&#x5236;&#x53F0;&#x7A97;&#x53E3;](http://c.biancheng.net/uploads/allimg/190523/4-1Z5231KF0352.gif)

下列函数影响的是控制台窗口及其相对于屏幕缓冲区的位置：

* SetConsoleWindowInfo：设置控制台窗口相对于屏幕缓冲区的大小和位置。
* GetConsoleScreenBufferInfo：返回（还包括其他一些信息）控制台窗口相对于屏幕缓冲区的矩形坐标。
* SetConsoleCursorPosition：将光标设置在屏幕缓冲区内的任何位置；如果区域不可见，则移动控制台窗口直到光标可见。
* ScrollConsoleScreenBuffer：移动屏幕缓冲区中的一些或全部文本，本函数会影响控制台窗口显示的文本。

#### 1\) SetConsoleTitle

函数 SetConsoleTitle 可以改变控制台窗口的标题。示例如下：

```text
.data
.titleStr BYTE "Console title", 0
.code
INVOKE SetConsoleTitle, ADDR titleStr
```

#### 2\) GetConsoleScreenBufferInfo

函数 GetConsoleScreenBufferInfo 返回控制台窗口的当前状态信息。它有两个参数：控制台屏幕的句柄和指向该函数填充的结构的指针：

```text
GetConsoleScreenBufferInfo PROTO,
  hConsoleOutput:HANDLE,
  lpConsoleScreenBufferInfo:PTR CONSOLE_SCREEN_BUFFER_INFO
```

CONSOLE\_SCREEN\_BUFFER\_INFO 结构如下：

```text
CONSOLE_SCREEN_BUFFER_INFO STRUCT
    dwSize COORD <>
    dwCursorPosition COORD <>
    wAttributes WORD ?
    srWindow SMALL_RECT <>
    dwMaximumWindowSize COORD <>
CONSOLE_SCREEN_BUFFER_INFO ENDS
```

dwSize 按字符行列数返回屏幕缓冲区大小。dwCursorPosition 返回光标的位置。这两个字段都是 COORD 结构。

wAttributes 返回字符的前景色和背景色，字符由诸如 WriteConsole 和 WriteFile 等函数写到控制台。srWindow 返回控制台窗口相对于屏幕缓冲区的坐标。

dwMaximumWindowSize 以当前屏幕缓冲区的大小、字体和视频显示大小为基础，返回控制台窗口的最大尺寸。函数示例调用如下所示：

```text
.data
consoleInfo CONSOLE_SCREEN_BUFFER_INFO <>
outHandle HANDLE ?
.code
INVOKE GetConsoleScreenBufferInfo, outHandle,
  ADDR consoleInfo
```

#### 3\) SetConsoleWindowInfo 函数

函数 SetConsoleWindowInfo 可以设置控制台窗口相对于其屏幕缓冲区的大小和位置。函数原型如下：

```text
SetConsoleWindowInfo PROTO,
  hConsoleOutput:HANDLE,         ;屏幕缓冲区句柄
  bAbsolute:DWORD,               ;坐标类型
  lpConsoleWindow:PTR SMALL_RECT ;矩形窗口指针
```

bAbsolute 说明如何使用结构中由 lpConsoleWindow 指出的坐标。如果 bAbsolute 为真，则坐标定义控制台窗口新的左上角和右下角。如果 bAbsolute 为假，则坐标与当前窗口坐标相加。

下面的程序向屏幕缓冲区写 50 行文本。然后重定义控制台窗口的大小和位置，有效地向后滚动文本。该程序使用了函数 SetConsoleWindowInfo：

```text
; 滚动控制台窗口    (Scroll.asm)

INCLUDE Irvine32.inc

.data
message BYTE ":  This line of text was written "
        BYTE "to the screen buffer",0dh,0ah
messageSize DWORD ($-message)

outHandle     HANDLE 0                     ; 标准输出句柄
bytesWritten  DWORD ?                      ; 已写入字节数
lineNum DWORD 0
windowRect    SMALL_RECT <0,0,60,11>       ; 上，下，左，右

.code
main PROC
    INVOKE GetStdHandle, STD_OUTPUT_HANDLE
    mov outHandle,eax

.REPEAT
      mov    eax,lineNum
      call    WriteDec                     ; 显示每行编号
    INVOKE WriteConsole,
      outHandle,                           ; 控制台输出句柄
      ADDR message,                        ; 字符串指针
      messageSize,                         ; 字符串长度
      ADDR bytesWritten,                   ; 返回已写字节数
      0                                    ; 未使用
      inc  lineNum                         ; 下一行编号
.UNTIL lineNum > 50

; 调整控制台窗口相对于屏幕缓冲区的大小和位置
    INVOKE SetConsoleWindowInfo,
      outHandle,
      TRUE,
      ADDR windowRect

    call    Readchar                      ; 等待按键
    call    Clrscr                        ; 清除屏幕缓冲区
    call    Readchar                      ; 等待第二次按键

    INVOKE ExitProcess,0
main ENDP
END main
```

最好能直接从 MS-Windows Exlporer 中，或者直接以命令行形式运行程序，而不使用集成的编辑环境。否则，编辑器可能会影响控制台窗口的行为和外观。在程序结束时需要两次按键：第一次清除屏幕缓冲区，第二次结束程序。

#### 4\) SetConsoleScreenBufferSize 函数

函数 SetConsoleScreenBufferSize 可以将屏幕缓冲区设置为 X 列 \* Y 行。其原型如下：

```text
SetConsoleScreenBufferSize PROTO,
  hConsoleOutput:HANDLE,         ;屏幕缓冲区句柄
  dwSize:COORD                  ;新屏幕缓冲区大小
```

## 11.17 控制台光标设置函数简述

Win32 API 提供了函数用于设置控制台应用光标的大小、可见度和屏幕位置。与这些函数相关的重要[数据结构](http://c.biancheng.net/data_structure/)是 CONSOLE\_CURSOR\_INFO，其中包含了控制台光标的大小和可见度信息：

```text
CONSOLE_CURSOR_INFO STRUCT
  dwSize DWORD ?
  bVisible DWORD ?
CONSOLE_CURSOR_INFO ENDS
```

dwSize 为光标填充的字符单元格的百分比（从 1 到 100）。如果光标可见，则 bVisible 等于 TRUE\(1\)。

#### 1\) GetConsoleCursorInfo 函数

函数 GetConsoleCursorInfo 返回控制台光标的大小和可见度。需向其传递指向结构 CONSOLE\_CURSOR\_INFO 的指针：

```text
GetConsoleCursorInfo PROTO,
  hConsoleOutput:HANDLE,
  lpConsoleCursorInfo:PTR CONSOLE_CURSOR_INFO
```

默认情况下，光标大小为 25，这表示光标占据了 25% 的字符单元格。

#### 2\) SetConsoleCursorInfo 函数

函数 SetConsoleCursorInfo 设置光标的大小和可见度。需向其传递指向结构 CONSOLE\_CURSOR\_INFO 的指针：

```text
SetConsoleCursorInfo PROTO,
  hConsoleOutput:HANDLE,
  lpConsoleCursorInfo:PTR CONSOLE_CURSOR_INFO
```

#### 3\) SetConsoleCursorPosition

函数 SetConsoleCursorPosition 设置光标的 X、Y 位置。向其传递一个 COORD 结构和控制台输岀句柄：

```text
SetConsoleCursorPosition PROTO,
  hConsoleOutput:DWORD,  ;输入模式句柄
  dwCursorPosition:COORD  ;屏幕 X、Y 坐标
```

## 11.18 SetConsoleTextAttribute和WriteConsoleOutputAttribute函数：控制文本颜色

控制台窗口中的文本颜色有两种控制方法。

* 通过调用 SetConsoleTextAttribute 来改变当前文本颜色，这种方法会影响控制台中所有后续输出文本。
* 调用 WriteConsoleOutputAttribute 来设置指定单元格的属性。函数 GetConsoleScreenBufferlnfo 返回当前屏幕的颜色以及其他控制台信息。

#### 1\) SetConsoleTextAttribute 函数

函数 SetConsoleTextAttribute 可以设置控制台窗口所有后续输出文本的前景色和背景色。原型如下：

```text
SetConsoleTextAttribute PROTO,
  hConsoleOutput:HANDLE,      ;控制台输出句柄
  wAttributes : WORD           ;颜色属性
```

颜色值保存在 wAttributes 参数的低字节中。

#### 2\) WriteConsoleOutputAttribute 函数

函数 WriteConsoleOutputAttribute 从指定位置开始，向控制台屏幕缓冲区的连续单元格复制一组属性值。原型如下：

```text
WriteConsoleOutputAttribute PROTO,
  hConsoleOutput:DWORD,                ;输出句柄
  lpAttribute:PTR WORD,                  ;写属性
  nLength:DWORD,                      ;单元格数
  dwWriteCoord :COORD,                 ;第一个单元格坐标
  lpNumberOfAttrsWritten:PTR DWORD      ;输出计数
```

其中：

* lpAttribute 指向属性数组，其中每个字节的低字节都包含了颜色值；
* nLength 为数组长度；
* dwWriteCoord 为接收属性的开始屏幕单元格；
* lpNumberOfAttrsWritten 指向一个变量，其中保存的是已写单元格的数量。

#### 3\) 示例：写文本颜色

为了演示颜色和属性的用法，程序 WriteColors.asm 创建了一个字符数组和一个属性数组， 属性数组中的每个元素都对应一个字符。程序调用 WriteConsoleOutputAttribute 将属性复制到屏幕缓冲区，调用 WriteConsoleOutputCharacter 将字符复制到相同的屏幕缓冲区单元格：

```text
; 写文本颜色      (WriteColors.asm)

INCLUDE Irvine32.inc
.data
outHandle    HANDLE ?
cellsWritten DWORD ?
xyPos COORD <10,2>

; 字符编号数组
buffer BYTE 1,2,3,4,5,6,7,8,9,10,11,12,13,14,15
       BYTE 16,17,18,19.20
BufSize DWORD ($ - buffer)
; 属性数组
attributes WORD 0Fh,0Eh,0Dh,0Ch,0Bh,0Ah,9,8,7,6
           WORD 5,4,3,2,1,0F0h,0E0h,0D0h,0C0h,0B0h

.code
main PROC

; 获取控制台标准输出句柄
    INVOKE GetStdHandle,STD_OUTPUT_HANDLE
    mov outHandle,eax

; 设置相邻单元格颜色
INVOKE WriteConsoleOutputAttribute,
      outHandle, ADDR attributes,
      BufSize, xyPos,
      ADDR cellsWritten

; 写 1 到 20 号字符
    INVOKE WriteConsoleOutputCharacter,
      outHandle, ADDR buffer, BufSize,
      xyPos, ADDR cellsWritten

    INVOKE ExitProcess,0
main ENDP
END main
```

![WriteColors&#x7A0B;&#x5E8F;&#x7684;&#x8F93;&#x51FA;](http://c.biancheng.net/uploads/allimg/190524/4-1Z524101945H3.gif)

## 11.19 Win32时间与日期函数

Win32 API 有相当多的时间和日期函数可供选择。最常见的是，用户想要用这些函数来获得和设置当前日期与时间。这里只能讨论这些函数的一小部分，不过在 Platform SDK 文档中可以查阅到下表列出的 Win32 函数。

| 函数 | 说明 |
| :--- | :--- |
| CompareFileTime | 比较两个 64 位的文件时间 |
| DosDateTimeToFileTime | 把 MS-DOS 日期和时间值转换为一个 64 位的文件时间 |
| FileTimeToDosDateTime | 把 64 位文件时间转换为 MS-DOS 日期和时间值 |
| FileTimeToLocalFileTime | 把 UTC（通用协调时间）文件时间转换为本地文件时间 |
| FileTimeToSystemTime | 把 64 位文件时间转换为系统时间格式 |
| GetFileTime | 检索文件创建、最后访问和最后修改的日期与时间 |
| GetLocalTime | 检索当前本地日期和时间 |
| GetSystemTime | 以 UTC 格式检索当前系统日期和时间 |
| GetSystemTimeAdjustment | 决定系统是否对其日历钟进行周期性时间调整 |
| GetSystemTimeAsFileTime | 以 UTC 格式检索当前系统日期和时间 |
| GetTickCount | 检索自系统启动后经过的毫秒数 |
| GetTimeZoneInformation | 检索当前时区参数 |
| LocalFileTimeToFileTime | 把本地文件时间转换为基于 UTC 的文件时间 |
| SetFileTime | 设置文件创建、最后访问和最后修改的日期与时间 |
| SetLocalTime | 设置当前本地时间与日期 |
| SetSystemTime | 设置当前系统时间与日期 |
| SetSystemTimeAdjustment | 启用或禁用对系统日历钟进行周期性时间调整 |
| SetTimeZoneInformation | 设置当前时区参数 |
| SystemTimeToFileTime | 把系统时间转换为文件时间 |
| SystemTimeToTzSpecificLocalTime | 把 UTC 时间转换为指定时区对应的本地时间 |

#### SYSTEMTIME 结构

SYSTEMTIME 结构由 Windows API 的日期和时间函数使用：

```text
SYSTEMTIME STRUCT
  wYear WORD ?          ;年（4 个数子）
  wMonth WORD ?        ;月（1 ~ 12）
  wDayOfWeek WORD ?   ;星期（0 ~ 6）
  wDay WORD ?          ;日（1 ~ 31）
  wHour WORD ?         ;小时（0 ~ 23）
  wMinute WORD ?       ;分钟（0 ~ 59）
  wSecond WORD ?       ;秒（0 ~ 59）
  wMilliseconds WORD ?   ;毫秒（0 ~ 999）
SYSTEMTIME ENDS
```

字段 wDayOfWeek 的值依序为星期天 = 0，星期一 = 1，以此类推。wMilliseconds 中的值不确定，因为系统可以与时钟源同步周期性地刷新时间。

### GetLocalTime 和 SetLocalTime

函数 GetLocalTime 根据系统时钟返回日期和当前时间。时间要调整为本地时区。调用该函数时，需向其传递一个指针指向 SYSTEMTIME 结构：

```text
GetLocalTime PROTO,
  lpSystemTime:PTR SYSTEMTIME
```

函数 GetLocalTime 调用示例如下：

```text
.data
sysTime SYSTEMTIME <>
.code
INVOKE GetLocalTime, ADDR sysTime
```

函数 SetLocalTime 设置系统的本地日期和时间。调用时，需向其传递一个指针指向包含了期望日期和时间的 SYSTEMTIME 结构：

```text
SetLocalTime PROTO,
  lpSystemTime:PTR SYSTEMTIME
```

如果函数执行成功，则返回非零整数；如果失败，则返回零。

### GetTickCount 函数

函数 GetTickCount 返回从系统启动起经过的毫秒数：

```text
GetTickCount PROTO        ; EAX 为返回值
```

由于返回值为一个双字，因此当系统连续运行 49.7 天后，时间将会回绕归零。可以使用这个函数监视循环经过的时间，并在达到时间限制时终止循环。

下面的程序 Timer.asm 计算两次调用 GetTickCount 之间的时间间隔。程序尝试确定计时器没有回绕（超过 49.7 天）。相似的代码可以用于各种程序：

```text
;计算经过的时间    （Timer.asm）
;用Win32函数GetTickCount演示一个简单的秒表计时器。
INCLUDE Irvine32.inc
INCLUDE macros.inc

.data
startTime DWORD ?

.code
main PROC
    INVOKE GetTickCount         ; 获取开始时间计数
    mov    startTime,eax        ; 保存开始时间计数

; Create a useless calculation loop.
    mov    ecx,10000100h
L1:    imul    ebx
    imul    ebx
    imul    ebx
    loop    L1

    INVOKE GetTickCount         ; 获得新的时间计数
    cmp    eax,startTime        ; 比开始时间计数小
    jb    error                 ; 时间回绕

    sub    eax,startTime        ; 计算时间间隔
    call    WriteDec            ; 显示时间间隔
    mWrite <" milliseconds have elapsed",0dh,0ah>
    jmp    quit

error:
    mWrite "Error: GetTickCount invalid--system has "
    mWrite <"been active for more than 49.7 days",0dh,0ah>
quit:
    exit
main ENDP

END main
```

### Sleep 函数

有些时候程序需要暂停或延迟一小段时间。虽然可以通过构造一个计算循环或忙循环来保持处理器工作，但是不同的处理器会使得执行时间不同。另外，忙循环还不必要地占用了处理器，这会降低在同一时间执行程序的速度。

Win32 函数 Sleep 按照指定毫秒数暂停当前执行的线程：

```text
Sleep PROTO,
  dwMilliseconds:DWORD
```

由于本教程中[汇编语言](http://c.biancheng.net/asm/)程序是单线程的，因此假设一个线程就等同于一个程序。当线程休眠时，它不会消耗处理器时间。

### GetDateTime 过程

Irvine32 链接库中的过程 GetDateTime 以 100 纳秒为间隔，返回从 1601 年 1 月 1 日起经过的时间间隔数。这看起来有点奇怪，因为那个时候计算机还是未知的。对任何事件，Microsoft 都用这个值来跟踪文件日期和时间。

如果想要为日期计算准备系统日期/时间值，Win32 SDK 建议采用如下步骤：

* 调用函数，如 GetLocalTime，填充 SYSTEMTIME 结构。
* 调用函数 SystemTimeToFileTime，将 SYSTEMTIME 结构转换为 FILETIME 结构。
* 将得到的 FILETIME 结构复制到 64 位的四字。

FILETIME 结构把 64 位四字分割为两个双字：

```text
FILETIME STRUCT
  loDateTime DWORD ?
  hiDateTime DWORD ?
FILETIME ENDS
```

下面的 GetDateTime 过程接收一个指针，指向 64 位四字变量。它用 Win32 FILETIME 格式将当前日期和时间保存到变量中：

```text
;--------------------------------------------------
GetDateTime PROC,
    pDateTime:PTR QWORD
    LOCAL sysTime:SYSTEMTIME, flTime:FILETIME
;
; 以64位整数形式 ( 按 Win32 FILETIME 格式 ) 获得并保存当前本地时间/日期
;--------------------------------------------------
; 获得系统本地时间
    INVOKE GetLocalTime,
      ADDR sysTime

; SYSTEMTIME 转换为 FILETIME.
    INVOKE SystemTimeToFileTime,
      ADDR sysTime,
      ADDR flTime

; 把 FILETIME 复制到一个64位整数
    mov esi,pDateTime
    mov eax,flTime.loDateTime
    mov DWORD PTR [esi],eax
    mov eax,flTime.hiDateTime
    mov DWORD PTR [esi+4],eax
    ret
GetDateTime ENDP
```

## 11.20 64位Windows API使用简述

任何对 Windows API 的 32 位调用都可以重新编写为 64 位调用。只需要记住几个关键 点就可以：

1\) 输入与输出句柄是 64 位的。

2\) 调用系统函数前，主调程序必须保留至少 32 字节的影子空间，其方法是将堆栈指针（RSP）寄存器减去 32。这使得系统函数能利用这个空间保存 RCX、RDX、R8 和 R9 寄存器的临时副本。

3\) 调用系统函数时，RSP 需对齐 16 字节地址边界（基本上，任何十六进制地址的最低位数字都是 0）。幸运的是，Win64 API 似乎没有强制执行这条规则，而且在应用程序中对堆栈对齐进行精确控制往往是比较困难的。

4\) 系统调用返回后，主调方必须回复 RSP 的初始值，方法是加上在函数调用前减去的数值。如果是在子程序中调用 Win64 API，那么这一点非常重要，因为在执行 RET 指令时，ESP 最终须指向子程序的返回地址。

5\) 整数参数利用 64 位寄存器传递。

6\) 不允许使用 INVOKE。取而代之，前 4 个参数要按照从左到右的顺序，依次放入这 4 个寄存器：RCX、RDX、R8 和 R9。其他参数则压入运行时堆栈。

7\) 系统函数用 RAX 存放返回的 64 位整数值。

下面的代码行演示了如何从 Irvine64 链接库中调用 64 位 GetStdHandle 函数：

```text
.data
STD_OUTPUT_HANDLE EQU -11
consoleOutHandle QWORD ?
.code
sub rsp, 40                  ;预留影子空间 & 对齐 RSP
mov rex,STD_OUTPUT_HANDLE
call GetstdHandle  -
mov consoleOutHandle,rax
add rsp,40
```

一旦控制台输出句柄被初始化，可以用后面的代码来演示如何调用 64 位 WriteConsoleA 函数。

这里有 5 个参数：RCX（控制台句柄）、RDX（字符串指针）、R8（字符串长度）、 R9（byteWritten 变量指针），以及最后一个虚拟零参数，它位于 RSP 上面的第 5 个堆栈位置。

```text
WriteString proc uses rex rdx r8 r9
    sub rsp, (5*8)            ;为 5 个参数预留空间
    movr cx,rdx
    call Str_length           ;用 EAX 返回字符串长度
    mov rcx,consoleOutHandle
    mov rdx, rdx              ;字符串指针
    mov r8, rax               ;字符串长度
    lea r9,bytesWritten
    mov qword ptr [rsp + 4 * SIZEOF QWORD], 0 ; 总是 0
    call WriteConsoleA
    add rsp,(5*8)             ;恢复 RSP
    ret
WriteString ENDP
```

## 11.21 如何编写图形化的Windows应用程序

本节将展示如何为 32 位 Microsoft Windows 编写简单的图形化应用程序。该程序创建并显示一个主窗口，显示消息框，并响应鼠标事件。本节内容为简介性质，如果希望了解更多信息，请参阅 Platform SDK 文档

下表列出了编写该程序时需要的各种链接库和头文件。

| 文件名 | 说明 |
| :--- | :--- |
| WinApp.asm | 程序源代码 |
| GraphWin.asm | 头文件，包含程序要使用的结构、常量和函数原型 |
| kernel32.lib | 本章前面使用的 MS-Windows API 链接库 |
| user32.lib | 其他 MS-Windows API 函数 |

/SUBSYSTEM:WINDOWS 代替了之前章节中使用的 /SUBSYSTEM:CONSOLE。程序从 kernel32.lib 和 user32.lib 这两个标准 MS-Windows 链接库中调用函数。

#### 主窗口

本程序显示一个全屏主窗口。为了让窗口适合本书页面，这里缩小了它的尺寸

![img](http://c.biancheng.net/uploads/allimg/190524/4-1Z5241421542B.gif)

#### 必要的结构

结构 POINT 以像素为单位，定义屏幕上一个点的 X 坐标和 Y 坐标。它可以用于定位图形对象、窗口和鼠标点击：

```text
POINT STRUCT
  ptX DWORD ?
  ptY DWORD ?
POINT ENDS
```

结构 RECT 定义矩形边界。成员 left 为矩形左边的 X 坐标，成员 top为矩形上边的 Y 坐标。成员 right 和 bottom 保存矩形类似的值：

```text
RECT STRUCT
  left DWORD ?
  top DWORD ?
  right DWORD ?
  bottom DWORD ?
RECT ENDS
```

结构 MSGStruct 定义 MS-Windows 需要的数据：

```text
MSGStruct STRUCT
  msgWnd DWORD ?
  msgMessage DWORD ?
  msgWparam DWORD ?
  msgLparam DWORD ?
  msgTime  DWORD ?
  msgPt POINT <>
MSGStruct ENDS
```

结构 WNDCLASS 定义窗口类。程序中的每个窗口都必须属于一个类，并且每个程序都必须为其主窗口定义一个窗口类。在主窗口可以显示之前，这个类必须要注册到操作系统：

```text
WNDCLASS STRUC
  style DWORD ?                ;窗口样式选项
  lpfnWndProc DWORD ?         ; winProc 函数指针
  cbClsExtra DWORD ?           ;共享内存
  cbWndExtra DWORD ?          ;附加字节数
  hlnstance DWORD ?            ;当前程序句柄
  hlcon DWORD ?               ;图标句柄
  hCursor DWORD ?             ;光标句柄
  hbrBackground DWORD ?       ;背景刷句柄
  IpszMenuName DWORD ?       ;菜单名指针
  IpszClassName DWORD ?       ; WinCZLass 名指针
WNDCLASS ENDS
```

下面对上述参数进行简单小结：

* style 是不同样式选项的集合，比如 WS\_CAPTION 和 WS\_BORDER，用于控制窗口外观和行为。
* lpfnWndProc 是指向函数的指针，该函数接收并处理由用户触发的事件消息。
* cbClsExtra 指向一个类中所有窗口使用的共享内存。可以为空。
* cbWndExtra 指定分配给后面窗口实例的附加字节数。
* hInstance 为当前程序实例的句柄。
* hIcon 和 hCursor 分别为当前程序中图标资源和光标资源的句柄。
* hbrBackground 为背景（颜色）刷的句柄。
* lpszMenuName 指向一个菜单名。
* lpszClassName 指向一个空字节结束的字符串，该字符串中包含了窗口的类名称。

## 11.22 MessageBox函数：显示一个简单的消息框

对程序而言，显示文本最简单的方法是将文本放入弹出消息框中，并等待用户点击按钮。Win32 API 链接库的 MessageBox 函数能显示一个简单的消息框。其函数原型如下：

```text
MessageBox PROTO,
hWnd:DWORD,
lpText:PTR BYTE,
lpCaption:PTR BYTE,
uType:DWORD
```

其中：

* hWnd 是当前窗口的句柄。
* lpText 指向一个空字节结束的字符串，该字符串将在消息框中显示。
* lpCaption 指向一个空字节结束的字符串，该字符串将在消息框的标题栏中显示。
* style 是一个整数，用于描述对话框的图标（可选）和按钮（必选）。

按钮由常数标识，如 MB\_OK 和 MB\_YESNO。图标也由常数标识，如 MB\_ICONQUESTION。

显示消息框时, 可以同时添加图标常数和按钮常数：

```text
INVOKE MessageBox, hWnd, ADDR QuestionText,
  ADDR QuestionTitle, MB_OK + MB_ICONQUESTION
```

## 11.23 WinMain过程：应用程序的启动过程

每个 Windows 应用程序都需要一个启动过程，通常将其命名为 WinMain，该过程负责下述任务：

* 得到当前程序的句柄。
* 加载程序的图标和光标。
* 注册程序的主窗口类，并标识处理该窗口事件消息的过程。
* 创建主窗口。
* 显示并更新主窗口。
* 开始接收和发送消息的循环，直到用户关闭应用程序窗口。

WinMain 包含一个名为 GetMessage 的消息处理循环，从程序的消息队列中取出下一条可用消息。如果 GetMessage 取出的消息是 WM\_QUIT，则返回零，即通知 WinMain 暂停程序。

对于其他消息，WinMain 将它们传递给 DispatchMessage 函数，该函数再将消息传递给程序的 WinProc 过程。若想进一步了解消息，请查阅 Platform SDK 文档的 Windows Messages。

## 11.24 WinProc过程：接收并处理所有与窗口有关的事件消息

WinProc 程接收并处理所有与窗口有关的事件消息。这些事件绝大多数是由用户通过点击和拖动鼠标、按下键盘按键等操作发起的。这个过程的工作就是解码每个消息，如果消息得以识别，则在应用程序中执行与该消息相关的任务。

过程声明如下：

```text
WinProc PROC,
hWnd: DWORD,     ;窗口句柄
localMsg: DWORD,  ;消息 ID
wParam:DWORD,    ;参数 1 （可变）
lParam: DWORD     ;参数 2 （可变）
```

根据具体的消息 ID，第三个和第四个参数的内容可变。比如，若点击鼠标，那么 lParam 就为点击位置的 X 坐标和 Y 坐标。在后面的示例程序中，WinProc 过程处理了三种特定的消息：

* WM\_LBUTTONDOWN，用户按下鼠标左键时产生该消息
* WM\_CREATE，表示刚刚创建主窗口
* WM\_CLOSE，表示将要关闭应用程序主窗口

比如，下面的代码行通过调用 MessageBox 向用户显示一个弹出消息框来处理 WM\_LBUTTONDOWN:

```text
.IF eax == WM_LBUTTONDOWN
INVOKE MessageBox, hWnd, ADDR PopupText,
    ADDR PopupTitle, MB_OK
jmp WinProcExit
```

用户所见的结果消息如下图所示。其他不希望被处理的消息都会被传递给 DefWindow-Proc（MS-Windows 默认的消息处理程序）。

![img](http://c.biancheng.net/uploads/allimg/190524/4-1Z524155415321.gif)

## 11.25 ErrorHandler过程：获取错误信息

过程 ErrorHandler 是可选的，如果在注册和创建程序主窗口的过程中系统报错，则调用该过程。

比如，如果成功注册程序主窗口，则函数 RegisterClass 返回非零值。但是，如果该函数返回值为零，那么就调用 ErrorHandler\( 显示一条消息 \) 并退出程序：

```text
INVOKE RegisterClass, ADDR MainWin
.IF eax == 0
  call ErrorHandler
  jmp Exit_Program
.ENDIF
```

过程 ErrorHandler 需要执行几个重要任务：

* 调用 GetLastError 取得系统错误号。
* 调用 FormatMessage 取得合适的系统格式化的错误消息字符串。
* 调用 MessageBox 显示包含错误消息字符串的弹出消息框。
* 调用 LocalFree 释放错误消息字符串使用的内存空间。

## 11.26 实例：Windows图形化程序

下面通过实例来演示一下如何通过[汇编语言](http://c.biancheng.net/asm/)来创建 Windows 图形化程序。不要担心这个程序的长度，其中大部分的代码在任何 MS-Windows 应用程序中都是一样的：

```text
; Windows 应用程序           (WinApp.asm)
; 本程序显示一个可调大小的应用程序窗口和几个弹出消息框
.386
INCLUDE Irvine32.inc
INCLUDE GraphWin.inc

;==================== DATA =======================
.data

AppLoadMsgTitle BYTE "Application Loaded",0
AppLoadMsgText  BYTE "This window displays when the WM_CREATE "
                BYTE "message is received",0

PopupTitle BYTE "Popup Window",0
PopupText  BYTE "This window was activated by a "
           BYTE "WM_LBUTTONDOWN message",0

GreetTitle BYTE "Main Window Active",0
GreetText  BYTE "This window is shown immediately after "
           BYTE "CreateWindow and UpdateWindow are called.",0

CloseMsg   BYTE "WM_CLOSE message received",0

ErrorTitle  BYTE "Error",0
WindowName  BYTE "ASM Windows App",0
className   BYTE "ASMWin",0

; 定义应用程序的窗口类结构
MainWin WNDCLASS <NULL,WinProc,NULL,NULL,NULL,NULL,NULL, \
    COLOR_WINDOW,NULL,className>

msg          MSGStruct <>
winRect   RECT <>
hMainWnd  DWORD ?
hInstance DWORD ?

;=================== CODE =========================
.code
WinMain PROC
; 获得当前过程的句柄
    INVOKE GetModuleHandle, NULL
    mov hInstance, eax
    mov MainWin.hInstance, eax

; 加载程序的图标和光标
    INVOKE LoadIcon, NULL, IDI_APPLICATION
    mov MainWin.hIcon, eax
    INVOKE LoadCursor, NULL, IDC_ARROW
    mov MainWin.hCursor, eax

; 注册窗口类
    INVOKE RegisterClass, ADDR MainWin
    .IF eax == 0
      call ErrorHandler
      jmp Exit_Program
    .ENDIF

; 创建应用程序的主窗口
    INVOKE CreateWindowEx, 0, ADDR className,
      ADDR WindowName,MAIN_WINDOW_STYLE,
      CW_USEDEFAULT,CW_USEDEFAULT,CW_USEDEFAULT,
      CW_USEDEFAULT,NULL,NULL,hInstance,NULL
    mov hMainWnd,eax

; 若 CreateWindowEx 失败，则显示消息并退出
    .IF eax == 0
      call ErrorHandler
      jmp  Exit_Program
    .ENDIF

; 保存窗口句柄，显示并绘制窗口
    INVOKE ShowWindow, hMainWnd, SW_SHOW
    INVOKE UpdateWindow, hMainWnd

; 显示欢迎消息
    INVOKE MessageBox, hMainWnd, ADDR GreetText,
      ADDR GreetTitle, MB_OK

; 启动程序的连续消息处理循环
Message_Loop:
    ; 从队列中取出下一条消息
    INVOKE GetMessage, ADDR msg, NULL,NULL,NULL

    ; 若没有其他消息则退出
    .IF eax == 0
      jmp Exit_Program
    .ENDIF

    ; 将消息传递给程序的 WinProc
    INVOKE DispatchMessage, ADDR msg
    jmp Message_Loop

Exit_Program:
      INVOKE ExitProcess,0
WinMain ENDP

;-----------------------------------------------------
WinProc PROC,
    hWnd:DWORD, localMsg:DWORD, wParam:DWORD, lParam:DWORD
; 应用程序的消息处理过程，处理应用程序特定的消息。
; 其他所有消息则传递给默认的 windows 消息处理过程
;-----------------------------------------------------
    mov eax, localMsg

    .IF eax == WM_LBUTTONDOWN          ; 鼠标按钮?
      INVOKE MessageBox, hWnd, ADDR PopupText,
        ADDR PopupTitle, MB_OK
      jmp WinProcExit
    .ELSEIF eax == WM_CREATE           ; 创建窗口?
      INVOKE MessageBox, hWnd, ADDR AppLoadMsgText,
        ADDR AppLoadMsgTitle, MB_OK
      jmp WinProcExit
    .ELSEIF eax == WM_CLOSE            ; 关闭窗口?
      INVOKE MessageBox, hWnd, ADDR CloseMsg,
        ADDR WindowName, MB_OK
      INVOKE PostQuitMessage,0
      jmp WinProcExit
    .ELSE                              ; 其他消息?
      INVOKE DefWindowProc, hWnd, localMsg, wParam, lParam
      jmp WinProcExit
    .ENDIF

WinProcExit:
    ret
WinProc ENDP

;---------------------------------------------------
ErrorHandler PROC
; 显示合适的系统错误消息
;---------------------------------------------------
.data
pErrorMsg  DWORD ?         ; 错误消息指针
messageID  DWORD ?
.code
    INVOKE GetLastError    ; 用EAX返回消息ID
    mov messageID,eax

    ; 获取相应的消息字符串
    INVOKE FormatMessage, FORMAT_MESSAGE_ALLOCATE_BUFFER + \
      FORMAT_MESSAGE_FROM_SYSTEM,NULL,messageID,NULL,
      ADDR pErrorMsg,NULL,NULL

    ; 显示错误消息
    INVOKE MessageBox,NULL, pErrorMsg, ADDR ErrorTitle,
      MB_ICONERROR+MB_OK

    ; 释放错误消息字符串
    INVOKE LocalFree, pErrorMsg
    ret
ErrorHandler ENDP

END WinMain
```

#### 运行程序

第一次加载程序时，显示如下消息框：

![img](http://c.biancheng.net/uploads/allimg/190524/4-1Z5241A11V47.gif)

当用户点击 OK 来关闭 Application Loaded 消息框时，则显示另一个消息框：

![img](http://c.biancheng.net/uploads/allimg/190524/4-1Z5241A145Y6.gif)

当用户关闭 Main Window Active 消息框时，就会显示程序的主窗口 ：

![img](http://c.biancheng.net/uploads/allimg/190524/4-1Z5241A201941.gif)

当用户在主窗口的任何位置点击鼠标时，显示如下消息框：

![img](http://c.biancheng.net/uploads/allimg/190524/4-1Z5241A215643.gif)

当用户关闭该消息框，并点击主窗口右上角上的 X 时，那么在窗口关闭之前将显示如下消息框：

![img](http://c.biancheng.net/uploads/allimg/190524/4-1Z5241A230395.gif)

当用户关闭了这个消息框后，则程序结束。

## 11.27 动态内存分配（堆分配）

动态内存分配 \(dynamic memory allocation\)，又被称为堆分配 \(heap allocation\)，是编程语言使用的一种技术，用于在创建对象、数组和其他结构时预留内存。比如在 [Java](http://c.biancheng.net/java/) 语言中，下面的语句就会为 String 对象保留内存：

```c
String str = new String("abcde");
```

同样的，在 [C++](http://c.biancheng.net/cplus/) 中，对变量使用大小属性就可以为一个整数数组分配空间：

```c
int size;
cin >> size;   //用户输入大小
int array[] = new int[size];
```

C、C++ 和 Java 都有内置运行时堆管理器来处理程序请求的存储分配和释放。程序启动时，堆管理器常常从操作系统中分配一大块内存，并为存储块指针创建空闲列表 \(free list\)。

当接收到一个分配请求时，堆管理器就把适当大小的内存块标识为已预留，并返回指向该块的指针。之后，当接收到对同一个块的删除请求时，堆就释放该内存块，并将其返回到空闲列表。每次接收到新的分配请求，堆管理器就会扫描空闲列表，寻找第一个可用的、且容量足够大的内存块来响应请求。

[汇编语言](http://c.biancheng.net/asm/)程序有两种方法进行动态分配：

* 方法一：通过系统调用从操作系统获得内存块。
* 方法二：实现自己的堆管理器来服务更小的对象提出的请求。

利用下表中的几个 Win32 API 函数就可以从 Windows 中请求多个不同大小的内存块。表中所有的函数都会覆盖通用寄存器，因此程序可能想要创建封装过程来实现重要寄存器的入栈和出栈操作。

| 函数 | 描述 |
| :--- | :--- |
| GetProcessHeap | 用 EAX 返回程序现存堆区域的 32 位整数句柄。如果函数成功，则 EAX 中的返回值为堆句柄。 如果函数失败，则 EAX 中的返回值为 NULL |
| HeapAlloc | 从堆中分配内存块。如果成功，EAX 中的返回值就为内存块的地址。如果失败，则 EAX 中的返 回值为 NULL |
| HeapCreate | 创建新堆，并使其对调用程序可用。如果函数成功，则 EAX 中的返回值为新创建堆的句柄。如果失败，则 EAX 的返回值为 NULL |
| HeapDestroy | 销毁指定堆对象，并使其句柄无效。如果函数成功，则 EAX 中的返回值为非零 |
| HeapFree | 释放之前从堆中分配的内存块，该堆由其地址和堆句柄进行标识。如果内存块释放成功，则返回值为非零 |
| HeapReAlloc | 对堆中内存块进行再分配和调整大小。如果函数成功，则返回值为指向再分配内存块的指针。如果函数失败，且没有指定 HEAP GENERATE EXCEPTIONS，则返回值为 NULL |
| HeapSize | 返回之前通过调用 HeapAlloc 或 HeapReAlloc 分配的内存块的大小。如果函数成功，则 EAX 包含被分配内存块的字节数。如果函数失败，则返回值为 SIZE\_T-1 \( SIZE\_T 等于指针能指向的最大字节数 \) |

#### GetProcessHeap

如果使用的是当前程序的默认堆，那么 GetProcessHeap 就足够了。这个函数没有参数，EAX 中的返回值就是堆句柄：

```text
GetProcessHeap PROTO
```

示例调用：

```text
.data
hHeap HANDLE ?
.code
INVOKE GetProcessHeap
.IF eax == NULL           ;不能获取句柄
    jmp quit
.ELSE
    mov hHeap,eax         ;句柄 ok
.ENDIF
```

#### HeapCreate

HeapCreate 能为当前程序创建一个新的私有堆：

```text
HeapCreate PROTO,
  flOptions:DWORD,          ;堆分配选项
  dwInitialSize:DWORD,        ;按字节初始化堆大小
  dwMaximumSize:DWORD    ;最大堆字节数
```

flOptions 设置为 NULL。dwInitialSize 设置为初始堆字节数，其值的上限为下一页的边界。如果 HeapAlloc 的调用超过了初始堆大小，那么堆最大可以扩展到 dwMaximumSize 参数中指定的大小（上限为下一页的边界）。调用后，EAX 中的返回值为空就表示堆未创建成 功。HeapCreate 的调用示例如下：

```text
HEAP_START = 2000000 ; 2 MB
HEAP_MAX = 400000000 ; 400 MB
.data
hHeap HANDLE ?       ; 堆句柄
.code
INVOKE HeapCreate, 0, HEAP_START, HEAP_MAX
.IF eax == NULL      ; 堆未创建
    call WriteWindowsMsg ; 显示错误消息
    jmp quit
.ELSE
    mov hHeap,eax    ; 句柄 OK
.ENDIF
```

#### HeapDestroy

HeapDeatroy 销毁一个已存在的私有堆（由 HeapCreate 创建）。需向其传递堆句柄：

```text
HeapDestroy PROTO,
  hHeap:DWORD     ;堆句柄
```

如果堆销毁失败，则 EAX 等于 NULL。下面为示例调用，其中使用了 WriteWindowsMsg 过程：

```text
.data
hHeap HANDLE ?                ;堆句柄
.code
INVOKE HeapDestroy, hHeap
.IF eax == NULL
    call WriteWindowsMsg      ;显示错误消息
.ENDIF
```

#### HeapAlloc

HeapAlloc 从已存在堆中分配一个内存块：

```text
HeapAlloc PROTO,
  hHeap:HANDLE,    ;现有堆内存块的句柄
  dwFlags :DWORD,  ;堆分配控制标志
  dwBytes:DWORD   ;分配的字节数
```

需传递下述参数：

* hHeap：32 位堆句柄，该堆由 GetProcessHeap 或 HeapCreate 初始化。
* dwFlags：一个双字，包含了一个或多个标志值。可以选择将其设置为 HEAP\_ZERO\_MEMORY，即设置内存块为全零。
* dwBytes：一个双字，表示堆分配的字节数。

如果 HeapAlloc 成功，则 EAX 包含指向新存储区的指针；如果失败，则 EAX 中的返回值为 NULL。下面的代码用 hHeap 标识一个堆，从该堆中分配了一个 1000 字节的数组，并将数组初始化为全零：

```text
.data
hHeap HANDLE ?    ;堆句柄
pArray DWORD ?    ;数组指针
.code
INVOKE HeapAlloc, hHeap, HEAP_ZERO_MEMORY, 1000
.IF eax == NULL
    mWrite "HeapAlloc failed"
    jmp quit
.ELSE
    mov pArray,eax
.ENDIF
```

#### HeapFree

函数 HeapFree 释放之前从堆中分配的一个内存块，该堆由其地址和堆句柄标识：

```text
HeapFree PROTO,
  hHeap:HANDLE,
  dwFlags:DWORD,
  lpMem:DWORD
```

第一个参数是包含该内存块的堆的句柄。第二个参数通常为零，第三个参数是指向将被释放内存块的指针。如果内存块释放成功，则返回值非零。如果该块不能被释放，则函数返回零。

示例调用如下：

```text
INVOKE HeapFree, hHeap, 0, pArray
```

#### Error Handling

若在调用 HeapCreate、HeapDestroy 或 GetProcessHeap 时遇到错误，可以通过调用 API 函数 GetLastError 来获得详细信息。还可以调用 Irvine32 链接库的函数 WriteWindowsMsg。

HeapCreate 调用示例如下：

```text
INVOKE HeapCreate, 0, HEAP_START, HEAP_MAX
.IF eax == NULL                    ;失败？
    call WriteWindowsMsg           ;显示错误信息
.ELSE
    mov    hHeap,eax               ;成功
.ENDIF
```

反之，函数 HeapAlloc 在失败时不会设置系统错误码，因此也就无法调用 GetLastError 或 WriteWindowsMsg。

## 11.28 实例：动态内存分配

下面的示例程序使用动态内存分配创建并填充了一个 1000 字节的数组：

```text
; 堆测试 #1        (Heaptest1.asm)

INCLUDE Irvine32.inc
; 使用动态内存分配，本程序分配并填充一个字节数据
.data
ARRAY_SIZE = 1000
FILL_VAL EQU 0FFh

hHeap   DWORD ?        ; 程序堆句柄
pArray  DWORD ?        ; 内存块指针
newHeap DWORD ?        ; 新堆句柄
str1 BYTE "Heap size is: ",0

.code
main PROC
    INVOKE GetProcessHeap          ; 获取程序堆句柄
    .IF eax == NULL                ; 如果失败，显示消息
    call    WriteWindowsMsg
    jmp    quit
    .ELSE
    mov    hHeap,eax                ; 成功
    .ENDIF

    call    allocate_array
    jnc    arrayOk                  ; 失败 (CF = 1)?
    call    WriteWindowsMsg
    call    Crlf
    jmp    quit

arrayOk:                            ; 成功填充数组
    call    fill_array
    call    display_array
    call    Crlf

    ; 释放数组
    INVOKE HeapFree, hHeap, 0, pArray

quit:
    exit
main ENDP

;--------------------------------------------------------
allocate_array PROC USES eax
;
; 动态分配数组空间
; 接收: EAX = 程序堆句柄
; 返回: 如果内存分配成功，则 CF = 0
;--------------------------------------------------------
    INVOKE HeapAlloc, hHeap, HEAP_ZERO_MEMORY, ARRAY_SIZE

    .IF eax == NULL
       stc                    ; 返回 CF = 1
    .ELSE
       mov  pArray,eax        ; 保存指针
       clc                    ; 返回 CF = 0
    .ENDIF

    ret
allocate_array ENDP

;--------------------------------------------------------
fill_array PROC USES ecx edx esi
;
; 用一个字符填充整个数组
; 接收: 无
; 返回: 无
;--------------------------------------------------------
    mov    ecx,ARRAY_SIZE             ; 循环计数器
    mov    esi,pArray                 ; 指向数组

L1:    mov    BYTE PTR [esi],FILL_VAL ; 填充每个字节
    inc    esi                        ; 下一个位置
    loop    L1

    ret
fill_array ENDP

;--------------------------------------------------------
display_array PROC USES eax ebx ecx esi
;
; 显示数组
; 接收: 无
; 返回: 无
;--------------------------------------------------------
    mov    ecx,ARRAY_SIZE     ; 循环计数器
    mov    esi,pArray         ; 指向数组

L1:    mov    al,[esi]        ; 取出一个字节
    mov    ebx,TYPE BYTE
    call    WriteHexB         ; 显示该字节
    inc    esi                ; 下一个位置
    loop    L1

    ret
display_array ENDP

END main
```

下面的示例采用动态内存分配重复分配大块内存，直到超过堆大小。

```text
; 堆测试 #2      (Heaptest2.asm)

INCLUDE Irvine32.inc
.data
HEAP_START =   2000000    ;   2 MB
HEAP_MAX  =  400000000    ; 400 MB
BLOCK_SIZE =    500000    ;  0.5 MB

hHeap DWORD ?             ; 堆句柄
pData DWORD ?             ; 块指针

str1 BYTE 0dh,0ah,"Memory allocation failed",0dh,0ah,0

.code
main PROC
    INVOKE HeapCreate, 0,HEAP_START, HEAP_MAX

    .IF eax == NULL          ; 失败?
    call    WriteWindowsMsg
    call    Crlf
    jmp    quit
    .ELSE
    mov    hHeap,eax          ; 成功
    .ENDIF

    mov    ecx,2000           ; 循环计数器

L1:    call allocate_block    ; 分配一个块
    .IF Carry?                ; 失败?
    mov    edx,OFFSET str1    ; 显示消息
    call    WriteString
    jmp    quit
    .ELSE                     ; 否: 打印一个点来显示进度
    mov    al,'.'
    call    WriteChar
    .ENDIF

    ;call free_block          ; 允许/禁止本行
    loop    L1

quit:
    INVOKE HeapDestroy, hHeap      ; 销毁堆
    .IF eax == NULL                ; 失败?
    call    WriteWindowsMsg        ; 是: 错误消息
    call    Crlf
    .ENDIF

    exit
main ENDP

allocate_block PROC USES ecx

    INVOKE HeapAlloc, hHeap, HEAP_ZERO_MEMORY, BLOCK_SIZE

    .IF eax == NULL
       stc                        ; 返回 CF = 1
    .ELSE
       mov  pData,eax             ; 保存指针
       clc                        ; 返回 CF = 0
    .ENDIF

    ret
allocate_block ENDP

free_block PROC USES ecx

    INVOKE HeapFree, hHeap, 0, pData
    ret
free_block ENDP

END main
```

## 11.29 x86存储管理简述

本节将对 Windows 32 位存储管理进行简要说明，展示它是如何使用 x86 处理器直接内置功能的。重点关注的是存储管理的两个主要方面：

* 将逻辑地址转换为线性地址
* 将线性地址转换为物理地址 \( 分页 \)

下面先简单回顾一下第2章《[x86处理器架构](http://c.biancheng.net/asm/20/)》介绍过的一些 x86 存储管理术语：

* 多任务处理 \(multitasking\) 允许多个程序（或任务）同时运行。处理器在所有运行程序中划分其时间。
* 段 \(segments\) 是可变大小的内存区，用于让程序存放代码或数据。
* 分段 \(segmentation\) 提供了分隔内存段的方法。它允许多个程序同时运行又不会相互干扰。
* 段描述符 \(segment descriptor\) 是一个 64 位的值，用于标识和描述一个内存段。它包含的信息有段基址、访问权限、段限长、类型和用法。

现在再增加两个新术语：

* 段选择符 \(segment selector\) 是保存在段寄存器 \(CS、DS、SS、ES、FS 或 GS\) 中的一个 16 位数值。
* 逻辑地址 \(logical address\) 就是段选择符加上一个 32 位的偏移量。

本教程一直都忽略了段寄存器，因为用户程序从来不会直接修改这些寄存器，所以只关注了 32 位数据偏移量。但是，从系统程序员的角度来看，段寄存器是很重要的，因为它们包含了对内存段的直接引用。

## 11.30 线性地址简述

在上一节《[x86存储管理](http://c.biancheng.net/view/3811.html)》中提到了线性地址，接下来为大家简单介绍一下线性地址。

### 逻辑地址转换为线性地址

多任务操作系统允许几个程序（任务）同时在内存中运行。每个程序都有自己唯一的数据区。假设现有 3 个程序，每个程序都有一个变量的偏移地址为 200h，那么，怎样区分这 3 个变量而不进行共享？

x86 解决这个问题的方法是，用一步或两步处理过程将每个变量的偏移量转换为唯一的内存地址。

第一步，将段值加上变量偏移量形成线性地址 \(linear address\)。这个线性地址可能就是该变量的物理地址。但是像 MS-Windows 和 Linux 这样的操作系统采用了分页 \(paging\) 功能，它使得程序能使用比可用物理空间更大的线性空间。这种情况下，就必需采用第二步页转换 \(page translation\)，将线性地址转换为物理地址。

首先了解一下处理器如何用段和选择符来确定变量的线性地址。每个段选择符都指向一个段描述符（位于描述符表中），其中包含了该内存段的基地址。如下图所示，逻辑地址中的 32 位偏移量加上段基址就形成了 32 位的线性地址。

![&#x903B;&#x8F91;&#x5730;&#x5740;&#x8F6C;&#x5316;&#x4F4D;&#x7EBF;&#x6027;&#x5730;&#x5740;](http://c.biancheng.net/uploads/allimg/190527/4-1Z52G24U3G2.gif)

线性地址是一个 32 位整数，其范围为 0FFFFFFFFh，它表示一个内存位置。如果禁止分页功能，那么线性地址也就是目标数据的物 理地址。

### 分页

分页是 x86 处理器的一个重要功能，它使得计算机能运行在其他情况下无法装入内存的一组程序。处理器初始只将部分程序加载到内存，而程序的其他部分仍然留在硬盘上。

程序使用的内存被分割成若干小区域，称为页 \(page\)，通常一页大小为 4KB。当每个程序运行时，处理器会选择内存中不活跃的页面替换出去，而将立即会被请求的页加载到内存。

操作系统通过维护一个页目录 \(page directory\) 和一组页表 \(page table\) 来持续跟踪当前内存中所有程序使用的页面。当程序试图访问线性地址空间内的一个地址时，处理器会自动将线性地址转换为物理地址。这个过程被称为页转换 \(page translation\)。

如果被请求页当前不在内存中，则处理器中断程序并产生一个页故障 \(page fault\)。操作系统将被请求页从硬盘复制到内存，然后程序继续执行。从应用程序的角度看，页故障和页转换都是自动发生的。

使用 Microsoft Windows 工具任务管理器（task manager）就可以查看物理内存和虚拟内存的区别。如下图所示计算机的物理内存为 256MB。任务管理器的 Commit Charge 框内为当前可用的虚拟内存总量。虚拟内存的限制为 633MB，大大高于计算机的物理内存。

![windows&#x4EFB;&#x52A1;&#x7BA1;&#x7406;&#x5668;&#x793A;&#x4F8B;](http://c.biancheng.net/uploads/allimg/190527/4-1Z52G24942649.gif)

### 描述符表

段描述符可以在两种表内找到：全局描述符表（global description table）和局部描述符表（local description table）。

全局描述符表（GDT）开机过程中，当操作系统将处理器切换到保护模式时，会创建唯——张 GDT，其基址保存在 GDTR（全局描述符表寄存器）中。表中的表项（称为段描述符）指向段。操作系统可以选择将所有程序使用的段保存在 GDT 中。

局部描述符表（LDT）在多任务操作系统中，每个任务或程序通常都分配有自己的段描述符表，称为 LDT。LDTR 寄存器保存的是程序 LDT 的地址。每个段描述符都包含了段在线性地址空间内的基地址。

一般，段与段之间是相互区分的。如下图所示，图中有三个不同的逻辑地址，这些地址选择了 LDT 中三个不同的表项。这里，假设禁止分页，因此， 线性地址空间也是物理地址空间。

![&#x7D22;&#x5F15;&#x5C40;&#x90E8;&#x63CF;&#x8FF0;&#x7B26;&#x8868;](http://c.biancheng.net/uploads/allimg/190527/4-1Z52G25024159.gif)

### 段描述符详细信息

除了段基址，段描述符还包含了位映射字段来说明段限长和段类型。只读类型段的一个例子就是代码段。如果程序试图修改只读段，则会产生处理器故障。

段描述符可以包含保护等级，以便保护操作系统数据不被应用程序访问。下面是对每个描述符字段的说明：

#### 1\) 基址

一个 32 位整数，定义段在 4GB 线性地址空间中的起始地址。

#### 2\) 特权级

每个段都可以分配一个特权级，特权级范围从 0 到 3，其中 0 级为最高级，一般用于操作系统核心代码。如果特权级数值高的程序试图访问特权级数值低的段，则发生处理器故障。

#### 3\) 段类型

说明段的类型并指定段的访问类型以及段生长的方向（向上或向下）。数据（包括堆栈）段可以是可读类型或读/写类型，其生长方向可以是向上的也可以是向下的。代码段可以是只执行类型或执行/只读类型。

#### 4\) 段存在标志

这一位说明该段当前是否在物理内存中。

#### 5\) 粒度标志

确定对段限长字段的解释。如果该位清零，则段限长以字节为单位。如果该 位置 1，则段限长的解释单位为 4096 字节。

#### 6\) 段限长：

这个 20 位的整数指定段大小。按照粒度标志，这个字段有两种解释：

* 该段有多少字节，范围为 1〜1MB。
* 该段包含多少个 4096 字节，允许段大小的范围为 4KB〜4GB。

## 11.31 页转换：线性地址转换位物理地址

若允许分页，则处理器必须将 32 位线性地址转换为 32 位物理地址。这个过程会用到 3 种结构：

* 页目录：一个数组，最多可包含 1024 个 32 位页目录项。
* 页表：一个数组，最多可包含 1024 个 32 位页表项。
* 页：4KB 或 4MB 的地址空间。

为了简化下面的叙述，假设页面大小为 4KB：

线性地址分为三个字段：页目录表项指针、页表项指针和页内偏移量。控制寄存器（CR3）保存了页目录的起始地址。如下图所示，处理器在进行线性地址到物理地址的转换时，采用如下步骤：

1\) 线性地址引用线性地址空间中的一个位置。

2\) 线性地址中 10 位的目录字段是页目录项的索引。页目录项包含了页表的基址。

3\) 线性地址中 10 位的页表字段是页表的索引，该页表由页目录项指定。索引到的页表项包含了物理内存中页面的基址。

4\) 线性地址中 12 位的偏移量字段与页面基址相加，生成的恰好是操作数的物理地址。

![&#x7EBF;&#x6027;&#x5730;&#x5740;&#x8F6C;&#x6362;&#x4E3A;&#x7269;&#x7406;&#x5730;&#x5740;](http://c.biancheng.net/uploads/allimg/190527/4-1Z52G325224I.gif)

操作系统可以选择让所有的运行程序和任务使用一个页目录，或者选择让每个任务使用一个页目录，还可以选择为两者的组合。

#### Windows 虚拟机管理器

现在对 IA-32 如何管理内存已经有了总体了解，那么看看 Windows 如何处理内存管理可能也会令人感兴趣。

虚拟机管理器（VMM）是 Windows 内核中的 32 位保护模式操作系统。它创建、运行、监视和终止虚拟机。它管理内存、进程、中断和异常。它与虚拟设备（virtual device）一起工作，使得它们能拦截中断和故障，以此来控制对硬件和已安装软件的访问。

VMM 和虚拟设备运行在特权级为 0 的单一 32 位平坦模式地址空间中。系统创建两个全局描述符表项（段描述符），一个是代码段的，一个是数据段的。段固定在线性地址 0。VMM 提供多线程和抢先多任务处理。通过共享运行应用程序的虚拟机之间的 CPU 时间，它可以同时运行多个应用程序。

在上面的文字中，可以将虚拟机解释为 Intel 中的过程或任务。它包含了程序代码、支撑软件、内存和寄存器。每个虚拟机都被分配了自己的地址空间、I/O 端口空间、中断向量表和局部描述符表。运行于虚拟 8086 模式的应用程序特权级为 3。Windows 中保护模式程序的特权级为 0 和 3。

