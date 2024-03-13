

# Calling convention
## 函数调用时，参数是怎么传递的，是怎么进去函数的，函数是怎么样返回的
首先，在callees和callers之间约定了calling convention，以保证在不同编译器便以后的函数在遵守同种convention的前提下可以互相调用。  
cs61的资料中主要讲了AMD64 ABI（followed on Solaris, Linux, FreeBSD, macOS）
### Argument passing and stack frames
在x86-64 Linux中前六个参数（integer or pointer arguments）会分别通过六个指定寄存器（%rdi, %rsi, %rdx, %rcx, %r8, and %r9）传递，第六个和随后的参数则会通过堆栈传递，而函数的返回值则通过RAX传递。（参数小于等于8bytes的情况下）  
当参数长度超过单个寄存器容量时，就会用多个存储单元（这里是8bytes）传递参数，超过2个单元则会用堆栈进行传递  
表格补充：
| argument type | registers |
| :----: | :----: |
| Integer/Pointer Argument 1-6 | %rdi, %rsi, %rdx, %rcx, %r8, and %r9 |
| floating point argument 1-8 | XMM0 - XMM7 |
| excess argument | stack |
| static chain pointer | %r10 |
| Integer return values(<8bytes) | %rax |
| Integer return values(>8bytes <16bytes) | %rax and %rdx |
| Floating-point(similar as integer) | XMM0 and XMM1 |

示例1：
```c
struct small {
    char a1,a2;
};

int f(struct small s) {
    return s.a1 + 2 * s.a2;
}
```
```x86asm
f:                                      # @f
        push    rbp
        ;栈指针 rsp栈顶指针
        mov     rbp, rsp
        ;copy argument to ax(di is the first register)
        mov     ax, di
        mov     word ptr [rbp - 2], ax
        ;eax lowest byte of argument (s.a1)
        movsx   eax, byte ptr [rbp - 2]
        ;ecx 2nd byte of argument (s.a2)
        movsx   ecx, byte ptr [rbp - 1]
        ;2*s.a2
        shl     ecx
        ;eax(return value)
        add     eax, ecx
        pop     rbp
        ret
```
示例2：
```c
struct medium {
    int a1,a2,a3,a4; // or long a1,a2
};

long f(struct small s) { 
    return s.a1 + 2 * s.a2 + s.a3 + s.a4;
}
```
```x86asm
f:                                      # @f
        push    rbp
        mov     rbp, rsp
        ;still use rdi
        mov     qword ptr [rbp - 16], rdi
        ;use rsi for extra a3 and a4
        mov     qword ptr [rbp - 8], rsi
        mov     eax, dword ptr [rbp - 16]
        mov     ecx, dword ptr [rbp - 12]
        shl     ecx, 1
        add     eax, ecx
        add     eax, dword ptr [rbp - 8]
        add     eax, dword ptr [rbp - 4]
        ;符号拓展
        ;Convert Double word to Quad word for Extension
        ;Convert Double word to Extened Quad word
        cdqe
        pop     rbp
        ret
```
对比
```c
struct small {
    int a1,a2,a3,a4,a5;// or long a1,a2,a3
};

long f(struct small s) {
    return s.a1 + 2 * s.a2 + s.a3 + s.a4 + s.a5;
}
```
```x86asm
f:                                      # @f
        push    rbp
        mov     rbp, rsp
        ;传递参数超过16bytes直接用堆栈进行传递参数
        lea     rcx, [rbp + 16]
        mov     eax, dword ptr [rcx]
        mov     edx, dword ptr [rcx + 4]
        shl     edx, 1
        add     eax, edx
        add     eax, dword ptr [rcx + 8]
        add     eax, dword ptr [rcx + 12]
        add     eax, dword ptr [rcx + 16]
        cdqe
        pop     rbp
        ret
```
示例3：
```c
struct small {
    long a1,a2;
};

struct small f(struct small s) {
    return s;
}
```
```x86asm
f:                                      # @f
        push    rbp
        mov     rbp, rsp
        ;参数传递没超过16bytes所以使用寄存器
        mov     qword ptr [rbp - 32], rdi
        mov     qword ptr [rbp - 24], rsi
        ;SSE字段
        movups  xmm0, xmmword ptr [rbp - 32]
        movaps  xmmword ptr [rbp - 16], xmm0
        ;返回值通过rax和rdx传递
        mov     rax, qword ptr [rbp - 16]
        mov     rdx, qword ptr [rbp - 8]
        pop     rbp
        ret
```
对比：
```c
struct small {
    long a1,a2,a3,a4;
};

struct small f(struct small s) {
    return s;
}
```
```x86asm
f:                                      # @f
        push    rbp
        mov     rbp, rsp
        ;oversized struct return 
        ;another pointer to a caller-provided space is prepended as the first argument
        mov     rax, rdi
        ;shifting all other arguments to the right by one place, and the value of this pointer is returned in RAX
        lea     rcx, [rbp + 16]
        mov     rdx, qword ptr [rcx]
        mov     qword ptr [rdi], rdx
        mov     rdx, qword ptr [rcx + 8]
        mov     qword ptr [rdi + 8], rdx
        mov     rdx, qword ptr [rcx + 16]
        mov     qword ptr [rdi + 16], rdx
        mov     rcx, qword ptr [rcx + 24]
        mov     qword ptr [rdi + 24], rcx
        pop     rbp
        ret
```
### callee 是如何返回caller 的，callee 怎么知道 caller 是谁
首先，在x86-64中c语言中的函数调用是通过%rsp寄存器(保存当前栈顶指针的位置)控制的，关于栈的工作原理在数据结构中已经学习过，这里不过多赘述，需要补充的是在push的过程中，栈顶指针的地址是在变小的，而pop则相反。  
c语言中，函数在自己的函数入口中会首先push %rbp 储存caller's %rbp地址，用来保证在callee函数运行结束后可以定位到caller  
例子1：
```c
int add(int n, int m)
{

}

```
```x86asm
add:                                    # @add
        ; push rbp 将caller的rbp压入栈中
        push    rbp
        ;save current stack pointer in %rbp
        mov     rbp, rsp
        mov     dword ptr [rbp - 8], edi
        mov     dword ptr [rbp - 12], esi
        mov     eax, dword ptr [rbp - 4]
        ;(%rbp)+8是函数最终返回的地址
        pop     rbp
        ret
```
例子2：
```c
int add(int n, int m)
{
    return n + m;
}

int sub(int n, int m)
{
    add(n,m);
    return n - m;
}


```
```x86asm
sub:                                    # @sub
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        ;后面的操作都是在rbp的基础上进行
        mov     dword ptr [rbp - 4], edi
        mov     dword ptr [rbp - 8], esi
        mov     edi, dword ptr [rbp - 4]
        mov     esi, dword ptr [rbp - 8]
        call    add
        mov     eax, dword ptr [rbp - 4]
        sub     eax, dword ptr [rbp - 8]
        add     rsp, 16
        ; 将rbp的内容出栈并跳转至(%rbp)+8
        pop     rbp
        ret

```
### call and ret
CALL pushes the return address onto the stack and transfers control to a procedure.
RET pops the return address off the stack and returns control to that location.  
call and ret use the stack and the IP register. Remember the IP register always hold the address of the next instruction to be executed. 

```c
int sub(int n, int m)
{
    add(n,m);
    return n - m;
}
```
```x86asm
sub:                                    # @sub
        push    rbp
        mov     rbp, rsp
        sub     rsp, 16
        mov     dword ptr [rbp - 4], edi
        mov     dword ptr [rbp - 8], esi
        mov     edi, dword ptr [rbp - 4]
        mov     esi, dword ptr [rbp - 8]
        call    add ;push IP register contents (address 
                    ;of next instruction) 
                    ;onto the stack, and pass control 
                    ;to the address of add
        mov     eax, dword ptr [rbp - 4]
        sub     eax, dword ptr [rbp - 8]
        add     rsp, 16
        pop     rbp
        ret         ;return result in eax(32bits) 
                    ;or rax(64bits)
```
 the contents of the last item pushed on the stack is pst item pushed on the stack is put into the IP register and the SP register is adjusted to point to the next newest value. Since normally, what is copied into the IP register is the address of the instruction after the call that we were just discussing, the program continues from that point.

## c/c++头文件的目的是做什么的

1. 为c/c++提供模块化服务结构并允许接口和实现分离(将多个程序中的共性抽象出来，减少代码量)，用户可以不必考虑调用函数的具体实现，增加代码的可维护性
2. 有头文件作为函数的标识后，经过预处理后可以有效的避免相同函数或变量的重复定义。  
比如以下代码，如果一个叫DOTHINGS_H的符号被定义了，他就不会再次定义
```c
#ifndef DOTHINGS_H
#define DOTHINGS_H

typedef struct 
{
    char buff[64];
} dothings;

dothings *create_thing(void);

#endif
```
3. c是采用局部编译的，linker需要这些c文件之间遵循一些协议，而头文件则作为中间的文件将这些信息包含其中。 
hello.c
```c
#include <stdio.h>

#include "dothings.h"

int main(int argc, char const *argv[])
{
    printf("hello world!\n");
    dothings *d = creat_thing();
    return 0;
}

```
dothings.c
```c
#include <stdio.h>
#include <stdlib.h>

#include "dothings.h"

dothings *create_thing() {
    return malloc(sizeof(dothings));
}
```
dothings.h
```c
#ifndef DOTHINGS_H
#define DOTHINGS_H

typedef struct 
{
    char buff[64];
} dothings;

dothings *create_thing(void);

#endif
```
```
gcc -c -o main.o hello.c 
gcc -c -o dothings.o dothings.c
gcc -o dothings main.o dothings.o
./dothings 

hello world!
```
编译器处理 C 文件时，首先读到的是 include 头文件包含语句，然后，将那个头文件里的内容原封不动地复制到当前 C 文件。各种预处理器指令会根据条件判断处理必要的头文件。  
有了头文件中的各种预处理判断，c语言在预处理阶段就可以根据用户设置或者不同硬件平台或编译器生成适合的文件，以确保c语言可以在不同的平台和编译器上一致的编写和编译。  

### 如果没有头文件的话
头文件主要是三部份，macro 宏，各类数据结构的声明，函数声明。  
没有头文件，宏的话，就需要每次都在c文件里定义导致重复；不知道数据结构的话，不知道变量的size，无法在栈和堆上分配对象；不知道函数声明（入参和返回值类型），编译器就不知道如何传参和处理返回值（以x64平台为例，例如32位参数用 eax，而64位是 rax）


