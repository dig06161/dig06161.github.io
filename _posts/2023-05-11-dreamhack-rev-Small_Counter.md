---
published: true
image: /img
layout: post
title: "[Dreamhack] REV Small Counter"
excerpt: "드림핵 리버싱 Small Counter 문제풀이"
tags: [Dreamhack, reversing, wargame, writeup]
math: true
date: 2023-05-31 09:00
---

이번 문제는 Dreamhack CTF Season 3 Round #4 (🌱Div2)에 출제된 리버싱 문제이다. 오랜만에 풀어보는 리버싱 문제인데, 리눅스 기반 ELF 바이너리다. 일단 실행시켜보자.

```c++
root@dig06161-virtual-machine:/home/dig06161/file/dreamhack/Small_Counter# ./chall
---Counter---
10
9
8
7
6
5
4
3
2
1
---END---
root@dig06161-virtual-machine:/home/dig06161/file/dreamhack/Small_Counter# 
```

10부터 1까지 출력한다. 여기서 flag를 출력하는 부분을 찾아 실행해야 할 것 같다. 우선  ghidra를 통해 바이너리를 열어보자. main함수의 어셈블리와 디컴파일 코드는 다음과 같다.

```c++
0x0000555555555494 <+0>:     endbr64 
   0x0000555555555498 <+4>:     push   rbp
   0x0000555555555499 <+5>:     mov    rbp,rsp
   0x000055555555549c <+8>:     sub    rsp,0xf0
   0x00005555555554a3 <+15>:    mov    DWORD PTR [rbp-0x4],0x0
   0x00005555555554aa <+22>:    lea    rax,[rip+0xb53]        # 0x555555556004
   0x00005555555554b1 <+29>:    mov    rdi,rax
   0x00005555555554b4 <+32>:    call   0x555555555090 <puts@plt>
   0x00005555555554b9 <+37>:    mov    DWORD PTR [rbp-0x4],0xa
   0x00005555555554c0 <+44>:    jmp    0x5555555555a0 <main+268>
   0x00005555555554c5 <+49>:    mov    eax,DWORD PTR [rbp-0x4]
   0x00005555555554c8 <+52>:    mov    esi,eax
   0x00005555555554ca <+54>:    lea    rax,[rip+0xb41]        # 0x555555556012
   0x00005555555554d1 <+61>:    mov    rdi,rax
   0x00005555555554d4 <+64>:    mov    eax,0x0
   0x00005555555554d9 <+69>:    call   0x5555555550b0 <printf@plt>
   0x00005555555554de <+74>:    cmp    DWORD PTR [rbp-0x4],0x3
   0x00005555555554e2 <+78>:    jne    0x55555555559c <main+264>
   0x00005555555554e8 <+84>:    movabs rax,0x38383830357b4d49
   0x00005555555554f2 <+94>:    movabs rdx,0x6a37386a32336a39
   0x00005555555554fc <+104>:   mov    QWORD PTR [rbp-0xf0],rax
   0x0000555555555503 <+111>:   mov    QWORD PTR [rbp-0xe8],rdx
   0x000055555555550a <+118>:   movabs rax,0x3035363435676a39
   0x0000555555555514 <+128>:   movabs rdx,0x6a68383234303438
   0x000055555555551e <+138>:   mov    QWORD PTR [rbp-0xe0],rax
   0x0000555555555525 <+145>:   mov    QWORD PTR [rbp-0xd8],rdx
   0x000055555555552c <+152>:   movabs rax,0x6838306969326968
   0x0000555555555536 <+162>:   movabs rdx,0x3833356a68693437
   0x0000555555555540 <+172>:   mov    QWORD PTR [rbp-0xd0],rax
   0x0000555555555547 <+179>:   mov    QWORD PTR [rbp-0xc8],rdx
   0x000055555555554e <+186>:   movabs rax,0x3667376a33343568
   0x0000555555555558 <+196>:   movabs rdx,0x68696a386b6a356b
   0x0000555555555562 <+206>:   mov    QWORD PTR [rbp-0xc0],rax
   0x0000555555555569 <+213>:   mov    QWORD PTR [rbp-0xb8],rdx
   0x0000555555555570 <+220>:   mov    DWORD PTR [rbp-0xb0],0x7d663232
   0x000055555555557a <+230>:   mov    BYTE PTR [rbp-0xac],0x0
   0x0000555555555581 <+237>:   lea    rcx,[rbp-0xf0]
   0x0000555555555588 <+244>:   lea    rax,[rbp-0x50]
   0x000055555555558c <+248>:   mov    edx,0x45
   0x0000555555555591 <+253>:   mov    rsi,rcx
   0x0000555555555594 <+256>:   mov    rdi,rax
   0x0000555555555597 <+259>:   call   0x5555555550c0 <memcpy@plt>
   0x000055555555559c <+264>:   sub    DWORD PTR [rbp-0x4],0x1
   0x00005555555555a0 <+268>:   cmp    DWORD PTR [rbp-0x4],0x0
   0x00005555555555a4 <+272>:   jg     0x5555555554c5 <main+49>
   0x00005555555555aa <+278>:   cmp    DWORD PTR [rbp-0x4],0x5
   0x00005555555555ae <+282>:   jne    0x5555555555fe <main+362>
   0x00005555555555b0 <+284>:   lea    rax,[rip+0xa5f]        # 0x555555556016
   0x00005555555555b7 <+291>:   mov    rdi,rax
   0x00005555555555ba <+294>:   call   0x555555555090 <puts@plt>
   0x00005555555555bf <+299>:   mov    eax,DWORD PTR [rbp-0x4]
   0x00005555555555c2 <+302>:   mov    DWORD PTR [rbp-0x8],eax
   0x00005555555555c5 <+305>:   mov    edx,DWORD PTR [rbp-0x8]
   0x00005555555555c8 <+308>:   lea    rcx,[rbp-0xa0]
   0x00005555555555cf <+315>:   lea    rax,[rbp-0x50]
   0x00005555555555d3 <+319>:   mov    rsi,rcx
   0x00005555555555d6 <+322>:   mov    rdi,rax
   0x00005555555555d9 <+325>:   call   0x5555555551c9 <flag_gen>
   0x00005555555555de <+330>:   lea    rax,[rbp-0xa0]
   0x00005555555555e5 <+337>:   mov    rsi,rax
   0x00005555555555e8 <+340>:   lea    rax,[rip+0xa2d]        # 0x55555555601c
   0x00005555555555ef <+347>:   mov    rdi,rax
   0x00005555555555f2 <+350>:   mov    eax,0x0
   0x00005555555555f7 <+355>:   call   0x5555555550b0 <printf@plt>
   0x00005555555555fc <+360>:   jmp    0x55555555560d <main+377>
   0x00005555555555fe <+362>:   lea    rax,[rip+0xa1c]        # 0x555555556021
   0x0000555555555605 <+369>:   mov    rdi,rax
   0x0000555555555608 <+372>:   call   0x555555555090 <puts@plt>
   0x000055555555560d <+377>:   mov    eax,0x0
   0x0000555555555612 <+382>:   leave  
   0x0000555555555613 <+383>:   ret 
```

```c++
undefined8 main(void)

{
  undefined8 local_f8;
  undefined8 local_f0;
  undefined8 local_e8;
  undefined8 local_e0;
  undefined8 local_d8;
  undefined8 local_d0;
  undefined8 local_c8;
  undefined8 local_c0;
  undefined4 local_b8;
  undefined local_b4;
  undefined local_a8 [80];
  char local_58 [72];
  uint local_10;
  uint local_c;
  
  local_c = 0;
  puts("---Counter---");
  for (local_c = 10; 0 < (int)local_c; local_c = local_c - 1) {
    printf("%d\n",(ulong)local_c);
    if (local_c == 3) {
      local_f8 = 0x38383830357b4d49;
      local_f0 = 0x6a37386a32336a39;
      local_e8 = 0x3035363435676a39;
      local_e0 = 0x6a68383234303438;
      local_d8 = 0x6838306969326968;
      local_d0 = 0x3833356a68693437;
      local_c8 = 0x3667376a33343568;
      local_c0 = 0x68696a386b6a356b;
      local_b8 = 0x7d663232;
      local_b4 = 0;
      memcpy(local_58,&local_f8,0x45);
    }
  }
  if (local_c == 5) {
    puts("Nice!");
    local_10 = local_c;
    flag_gen(local_58,(long)local_a8,local_c);
    printf("\n%s\n",local_a8);
  }
  else {
    puts("---END---");
  }
  return 0;
}
```

코드를 분석해보면 플레그를 만들어주는 함수는 flag_gen 함수이고 for 가 동작하는 반복문과 별개의 코드로 if가 동작해야지 플래그를 출력한다. if가 동작하기 위해서는 반복문이 끝난 후 local_c가 5를 가지고 있어야 동작한다. 다만 반복문이 종료되면 local_c가 0을 가지게 된다. 따라서 gdb를 통해 bp를 반복분이 끝나는 시점에 걸고 local_c 값을 강제로 5로 바꾸면 풀 수 있을 것이다.

다만 바이너리에 bp를 걸면 좀 이상하게 동작한다. PIE가 걸려있어 바이너리를 실행할 때 마다 주소값이 바뀌는 문제가 있다. gdb에서는 디버깅의 편의를 위해 PIE 보호기법이 걸린 바이너리는 코드영역 주소값을 0x555555555000로 가진다. 따라서 한번 실행한 후 main함수를 disass 하면 0x0000555555555494주소를 가진다. 이 부분에 bp를 걸고 이후 main+278부분에 bp를 걸어주면 중단지점을 설정할 수 있다. 이후 set 기능을 이용해 rbp - 4 부분을 5로 쓰면 플래그를 얻을 수 있다.

```c++
pwndbg> $rbp-4
Undefined command: "$rbp-4".  Try "help".
pwndbg> p $rbp-4
$1 = (void *) 0x7fffffffe32c
pwndbg> x/x 0x7fffffffe32c
0x7fffffffe32c: 0x00000000
pwndbg> set *0x7fffffffe32c=5
pwndbg> x/x 0x7fffffffe32c
0x7fffffffe32c: 0x00000005
pwndbg> c
Continuing.
Nice!

DH{389998e56e90e8eb34238948469ce중략...}
[Inferior 1 (process 48876) exited normally]
pwndbg> 
```