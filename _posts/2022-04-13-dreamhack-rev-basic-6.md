---
published: ture
image: /img
layout: post
title: `[`Dreamhack`]` rev-basic-6 문제풀이
tags: [Dreamhack, reversing, ctf, writeup]
math: true
date: 2022-04-13 18:30
---
이번에는 드림핵 리버싱 베이직 6번 문제를 풀어보자.

이전에 올렸던 rev-basic-5 문제와 동일하게 바이너리를 실행 시키면

```shell
input : 
```
이라는 문자열과 함께 문자열을 입력 받고 정답이면 Correct, 아니면 Wrong을 출력한다.

우선 동일하게 x64 디버거를 이용해 어셈블리를 분석해보자.

해당 프로그램 main의 어셈블리는 다음과 같다.

```c++
00007FF782681120 | 40:57                    | push rdi                                | rdi:&"ALLUSERSPROFILE=C:\\ProgramData"
00007FF782681122 | 48:81EC 30010000         | sub rsp,130                             |
00007FF782681129 | 48:8B05 F81F0000         | mov rax,qword ptr ds:[7FF782683128]     |
00007FF782681130 | 48:33C4                  | xor rax,rsp                             |
00007FF782681133 | 48:898424 20010000       | mov qword ptr ss:[rsp+120],rax          |
00007FF78268113B | 48:8D4424 20             | lea rax,qword ptr ss:[rsp+20]           |
00007FF782681140 | 48:8BF8                  | mov rdi,rax                             | rdi:&"ALLUSERSPROFILE=C:\\ProgramData"
00007FF782681143 | 33C0                     | xor eax,eax                             |
00007FF782681145 | B9 00010000              | mov ecx,100                             |
00007FF78268114A | F3:AA                    | rep stosb                               |
00007FF78268114C | 48:8D0D BD100000         | lea rcx,qword ptr ds:[7FF782682210]     | 00007FF782682210:"Input : "
00007FF782681153 | E8 58000000              | call chall6.7FF7826811B0                |
00007FF782681158 | 48:8D5424 20             | lea rdx,qword ptr ss:[rsp+20]           |
00007FF78268115D | 48:8D0D B8100000         | lea rcx,qword ptr ds:[7FF78268221C]     | 00007FF78268221C:"%256s"
00007FF782681164 | E8 A7000000              | call chall6.7FF782681210                |
00007FF782681169 | 48:8D4C24 20             | lea rcx,qword ptr ss:[rsp+20]           |
00007FF78268116E | E8 8DFEFFFF              | call chall6.7FF782681000                |
00007FF782681173 | 85C0                     | test eax,eax                            |
00007FF782681175 | 74 0F                    | je chall6.7FF782681186                  |
00007FF782681177 | 48:8D0D AA100000         | lea rcx,qword ptr ds:[7FF782682228]     | 00007FF782682228:"Correct"
00007FF78268117E | FF15 04100000            | call qword ptr ds:[<&puts>]             |
00007FF782681184 | EB 0D                    | jmp chall6.7FF782681193                 |
00007FF782681186 | 48:8D0D A3100000         | lea rcx,qword ptr ds:[7FF782682230]     | 00007FF782682230:"Wrong"
00007FF78268118D | FF15 F50F0000            | call qword ptr ds:[<&puts>]             |
00007FF782681193 | 33C0                     | xor eax,eax                             |
00007FF782681195 | 48:8B8C24 20010000       | mov rcx,qword ptr ss:[rsp+120]          |
00007FF78268119D | 48:33CC                  | xor rcx,rsp                             |
00007FF7826811A0 | E8 5B010000              | call chall6.7FF782681300                |
00007FF7826811A5 | 48:81C4 30010000         | add rsp,130                             |
00007FF7826811AC | 5F                       | pop rdi                                 | rdi:&"ALLUSERSPROFILE=C:\\ProgramData"
00007FF7826811AD | C3                       | ret                                     |
```

여기서 정답의 로직을 분석하는 부분은 00007FF78268116E이다.

테스트로 AAAAA를 입력한 뒤, 위 주소부분의 어셈블리를 살펴보자.

```c++
00007FF782681000 | 48:894C24 08             | mov qword ptr ss:[rsp+8],rcx            | [rsp+8]:"%256s"
00007FF782681005 | 48:83EC 18               | sub rsp,18                              |
00007FF782681009 | C70424 00000000          | mov dword ptr ss:[rsp],0                |
00007FF782681010 | EB 08                    | jmp chall6.7FF78268101A                 |
00007FF782681012 | 8B0424                   | mov eax,dword ptr ss:[rsp]              |
00007FF782681015 | FFC0                     | inc eax                                 |
00007FF782681017 | 890424                   | mov dword ptr ss:[rsp],eax              |
00007FF78268101A | 48:630424                | movsxd rax,dword ptr ss:[rsp]           |
00007FF78268101E | 48:83F8 12               | cmp rax,12                              |
00007FF782681022 | 73 31                    | jae chall6.7FF782681055                 |
00007FF782681024 | 48:630424                | movsxd rax,dword ptr ss:[rsp]           |
00007FF782681028 | 48:8B4C24 20             | mov rcx,qword ptr ss:[rsp+20]           |
00007FF78268102D | 0FB60401                 | movzx eax,byte ptr ds:[rcx+rax]         | rcx+rax*1:"AAAA"
00007FF782681031 | 48:8D0D E81F0000         | lea rcx,qword ptr ds:[7FF782683020]     | rcx:"AAAAA"
00007FF782681038 | 0FB60401                 | movzx eax,byte ptr ds:[rcx+rax]         | rcx+rax*1:"AAAA"
00007FF78268103C | 48:630C24                | movsxd rcx,dword ptr ss:[rsp]           |
00007FF782681040 | 48:8D15 B91F0000         | lea rdx,qword ptr ds:[7FF782683000]     |
00007FF782681047 | 0FB60C0A                 | movzx ecx,byte ptr ds:[rdx+rcx]         |
00007FF78268104B | 3BC1                     | cmp eax,ecx                             |
00007FF78268104D | 74 04                    | je chall6.7FF782681053                  |
00007FF78268104F | 33C0                     | xor eax,eax                             |
00007FF782681051 | EB 07                    | jmp chall6.7FF78268105A                 |
00007FF782681053 | EB BD                    | jmp chall6.7FF782681012                 |
00007FF782681055 | B8 01000000              | mov eax,1                               |
00007FF78268105A | 48:83C4 18               | add rsp,18                              |
00007FF78268105E | C3                       | ret                                     |
```

중요한 부분은 00007FF782681031 부터 00007FF78268104B를 보면 될것 같다.

입력받은 문자열을 순서대로 비교하는 로직이다. 위 주소를 살펴보면 7FF782683020 + 입력받은 문자열의 hex값 을 계산해 7FF782683000와 비교한다. 이를 역산하면 7FF782683000에 있는 hex값이 7FF782683020로부터 얼마만큼 떨어져 있는지 확인하면 쉽게 답을 찾을 수 있다. 이런 로직을 보고 파이썬 코드를 작성했다.

```python
a = [0x00, 0x4D, 0x51, 0x50, 0xEF, 0xFB, 0xC3, 0xCF, 0x92, 0x45, 0x4D, 0xCF, 0xF5, 0x04, 0x40, 0x50, 0x43, 0x63]
b =[0x63, 0x7C, 0x77, 0x7B, 0xF2, 0x6B, 0x6F, 0xC5, 0x30, 0x1, 0x67, 0x2B, 0xFE, 0xD7, 0xAB, 0x76
,0xCA, 0x82, 0xC9, 0x7D, 0xFA, 0x59, 0x47, 0xF0, 0xAD, 0xD4, 0xA2, 0xAF, 0x9C, 0xA4, 0x72, 0xC0
,0xB7, 0xFD, 0x93, 0x26, 0x36, 0x3F, 0xF7, 0xCC, 0x34, 0xA5, 0xE5, 0xF1, 0x71, 0xD8, 0x31, 0x15
,0x04, 0xC7, 0x23, 0xC3, 0x18, 0x96, 0x05, 0x9A, 0x07, 0x12, 0x80, 0xE2, 0xEB, 0x27, 0xB2, 0x75
,0x09, 0x83, 0x2C, 0x1A, 0x1B, 0x6E, 0x5A, 0xA0, 0x52, 0x3B, 0xD6, 0xB3, 0x29, 0xE3, 0x2F, 0x84
,0x53, 0xD1, 0x00, 0xED, 0x20, 0xFC, 0xB1, 0x5B, 0x6A, 0xCB, 0xBE, 0x39, 0x4A, 0x4C, 0x58, 0xCF
,0xD0, 0xEF, 0xAA, 0xFB, 0x43, 0x4D, 0x33, 0x85, 0x45, 0xF9, 0x02, 0x7F, 0x50, 0x3C, 0x9F, 0xA8
,0x51, 0xA3, 0x40, 0x8F, 0x92, 0x9D, 0x38, 0xF5, 0xBC, 0xB6, 0xDA, 0x21, 0x10, 0xFF, 0xF3, 0xD2
,0xCD, 0x0C, 0x13, 0xEC, 0x5F, 0x97, 0x44, 0x17, 0xC4, 0xA7, 0x7E, 0x3D, 0x64, 0x5D, 0x19, 0x73
,0x60, 0x81, 0x4F, 0xDC, 0x22, 0x2A, 0x90, 0x88, 0x46, 0xEE, 0xB8, 0x14, 0xDE, 0x5E, 0x0B, 0xDB
,0xE0, 0x32, 0x3A, 0x0A, 0x49, 0x06, 0x24, 0x5C, 0xC2, 0xD3, 0xAC, 0x62, 0x91, 0x95, 0xE4, 0x79
,0xE7, 0xC8, 0x37, 0x6D, 0x8D, 0xD5, 0x4E, 0xA9, 0x6C, 0x56, 0xF4, 0xEA, 0x65, 0x7A, 0xAE, 0x8
,0xBA, 0x78, 0x25, 0x2E, 0x1C, 0xA6, 0xB4, 0xC6, 0xE8, 0xDD, 0x74, 0x1F, 0x4B, 0xBD, 0x8B, 0x8A
,0x70, 0x3E, 0xB5, 0x66, 0x48, 0x03, 0xF6, 0x0E, 0x61, 0x35, 0x57, 0xB9, 0x86, 0xC1, 0x1D, 0x9E
,0xE1, 0xF8, 0x98, 0x11, 0x69, 0xD9, 0x8E, 0x94, 0x9B, 0x1E, 0x87, 0xE9, 0xCE, 0x55, 0x28, 0xDF
,0x8C, 0xA1, 0x89, 0x0D, 0xBF, 0xE6, 0x42, 0x68, 0x41, 0x99, 0x2D, 0x0F, 0xB0, 0x54, 0xBB, 0x16
,0x42, 0xCD, 0xB7, 0x32, 0x13, 0x59, 0xFF, 0xFF, 0xBD, 0x32, 0x48, 0xCD, 0xEC, 0xA6]

for i in a:
        b_count = 0x00
        for j in b:
                if(i == j):
                        break
                else:
                        b_count=b_count+1
        print(chr(b_count), end='')

```

위 코드에서 변수 a 는 7FF782683000에 있는 hex값이고, 변수 b 는 7FF782683020에 있는 hex값을 의미하며 반복문을 통해 순서대로 돌면서 떨어진 거리를 계산한다.

위 코드를 돌리면 플레그 값을 확인할 수 있다.
