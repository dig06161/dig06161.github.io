---
published: ture
image: /img
layout: post
title: Dreamhack rev-basic-5 문제풀이
tags: [Dreamhack, reversing, wargame, writeup]
math: true
date: 2022-04-09 18:30
---
이번 문제는 드림핵 리버싱문제 rev-basic-5이다. 확실히 베이직 문제이다 보니 약간의 분석과정만 거치면 풀이법이 보여 쉬운편에 속했다.

우선 exe 파일을 다운 받으면 chall5.exe라는 바이너리가 다운로드 된다. 이후 이 바이너리를 실행 시키면 

```shell
input : 
```
구문으로 문자열을 입력 받고 맞으면 Correct 틀리면 Wrong이라는 문자열을 출력한다.

<br><br>

이제 이 바이너리를 x64 디버거로 분석 해보자.

```c++
00007FF6A16C1130 | 40:57                    | push rdi                                | rdi:L"샰櫚Ʊ"
00007FF6A16C1132 | 48:81EC 30010000         | sub rsp,130                             |
00007FF6A16C1139 | 48:8B05 E81E0000         | mov rax,qword ptr ds:[7FF6A16C3028]     |
00007FF6A16C1140 | 48:33C4                  | xor rax,rsp                             |
00007FF6A16C1143 | 48:898424 20010000       | mov qword ptr ss:[rsp+120],rax          |
00007FF6A16C114B | 48:8D4424 20             | lea rax,qword ptr ss:[rsp+20]           |
00007FF6A16C1150 | 48:8BF8                  | mov rdi,rax                             | rdi:L"샰櫚Ʊ"
00007FF6A16C1153 | 33C0                     | xor eax,eax                             |
00007FF6A16C1155 | B9 00010000              | mov ecx,100                             |
00007FF6A16C115A | F3:AA                    | rep stosb                               |
00007FF6A16C115C | 48:8D0D AD100000         | lea rcx,qword ptr ds:[7FF6A16C2210]     | 00007FF6A16C2210:"Input : "
00007FF6A16C1163 | E8 58000000              | call chall5.7FF6A16C11C0                |
00007FF6A16C1168 | 48:8D5424 20             | lea rdx,qword ptr ss:[rsp+20]           |
00007FF6A16C116D | 48:8D0D A8100000         | lea rcx,qword ptr ds:[7FF6A16C221C]     | 00007FF6A16C221C:"%256s"
00007FF6A16C1174 | E8 A7000000              | call chall5.7FF6A16C1220                |
00007FF6A16C1179 | 48:8D4C24 20             | lea rcx,qword ptr ss:[rsp+20]           |
00007FF6A16C117E | E8 7DFEFFFF              | call chall5.7FF6A16C1000                |
00007FF6A16C1183 | 85C0                     | test eax,eax                            |
00007FF6A16C1185 | 74 0F                    | je chall5.7FF6A16C1196                  |
00007FF6A16C1187 | 48:8D0D 9A100000         | lea rcx,qword ptr ds:[7FF6A16C2228]     | 00007FF6A16C2228:"Correct"
00007FF6A16C118E | FF15 F40F0000            | call qword ptr ds:[<&puts>]             |
00007FF6A16C1194 | EB 0D                    | jmp chall5.7FF6A16C11A3                 |
00007FF6A16C1196 | 48:8D0D 93100000         | lea rcx,qword ptr ds:[7FF6A16C2230]     | 00007FF6A16C2230:"Wrong"
00007FF6A16C119D | FF15 E50F0000            | call qword ptr ds:[<&puts>]             |
00007FF6A16C11A3 | 33C0                     | xor eax,eax                             |
00007FF6A16C11A5 | 48:8B8C24 20010000       | mov rcx,qword ptr ss:[rsp+120]          |
00007FF6A16C11AD | 48:33CC                  | xor rcx,rsp                             |
00007FF6A16C11B0 | E8 5B010000              | call chall5.7FF6A16C1310                |
00007FF6A16C11B5 | 48:81C4 30010000         | add rsp,130                             |
00007FF6A16C11BC | 5F                       | pop rdi                                 | rdi:L"샰櫚Ʊ"
00007FF6A16C11BD | C3                       | ret                                     |
```

다음과 같은 형태의 바이너리이다. 여기서 문자열을 입력 받고 정답임을 검사하는 함수의 위치는 00007FF6A16C117E 이다. 임의 값을 넣고 함수에 bp를 걸어 동작을 확인해보자.

AAAAA 라는 문자열을 입력 하였고 정답을 검증하는 함수의 어셈블리는 다음과 같다.

```c++
00007FF6A16C1000 | 48:894C24 08             | mov qword ptr ss:[rsp+8],rcx            | [rsp+8]:"%256s"
00007FF6A16C1005 | 48:83EC 18               | sub rsp,18                              |
00007FF6A16C1009 | C70424 00000000          | mov dword ptr ss:[rsp],0                |
00007FF6A16C1010 | EB 08                    | jmp chall5.7FF6A16C101A                 |
00007FF6A16C1012 | 8B0424                   | mov eax,dword ptr ss:[rsp]              |
00007FF6A16C1015 | FFC0                     | inc eax                                 |
00007FF6A16C1017 | 890424                   | mov dword ptr ss:[rsp],eax              |
00007FF6A16C101A | 48:630424                | movsxd rax,dword ptr ss:[rsp]           |
00007FF6A16C101E | 48:83F8 18               | cmp rax,18                              |
00007FF6A16C1022 | 73 39                    | jae chall5.7FF6A16C105D                 |
00007FF6A16C1024 | 48:630424                | movsxd rax,dword ptr ss:[rsp]           |
00007FF6A16C1028 | 48:8B4C24 20             | mov rcx,qword ptr ss:[rsp+20]           |
00007FF6A16C102D | 0FB60401                 | movzx eax,byte ptr ds:[rcx+rax]         | rcx+rax*1:"AAAA"
00007FF6A16C1031 | 8B0C24                   | mov ecx,dword ptr ss:[rsp]              |
00007FF6A16C1034 | FFC1                     | inc ecx                                 |
00007FF6A16C1036 | 48:63C9                  | movsxd rcx,ecx                          | rcx:"AAAAA"
00007FF6A16C1039 | 48:8B5424 20             | mov rdx,qword ptr ss:[rsp+20]           |
00007FF6A16C103E | 0FB60C0A                 | movzx ecx,byte ptr ds:[rdx+rcx]         |
00007FF6A16C1042 | 03C1                     | add eax,ecx                             |
00007FF6A16C1044 | 48:630C24                | movsxd rcx,dword ptr ss:[rsp]           |
00007FF6A16C1048 | 48:8D15 B11F0000         | lea rdx,qword ptr ds:[7FF6A16C3000]     |
00007FF6A16C104F | 0FB60C0A                 | movzx ecx,byte ptr ds:[rdx+rcx]         |
00007FF6A16C1053 | 3BC1                     | cmp eax,ecx                             |
00007FF6A16C1055 | 74 04                    | je chall5.7FF6A16C105B                  |
00007FF6A16C1057 | 33C0                     | xor eax,eax                             |
00007FF6A16C1059 | EB 07                    | jmp chall5.7FF6A16C1062                 |
00007FF6A16C105B | EB B5                    | jmp chall5.7FF6A16C1012                 |
00007FF6A16C105D | B8 01000000              | mov eax,1                               |
00007FF6A16C1062 | 48:83C4 18               | add rsp,18                              |
00007FF6A16C1066 | C3                       | ret                                     |
```

00007FF6A16C1039 부터 00007FF6A16C1053 부분이 주요 부분이고 이 부분을 분석해보면 첫번째 사이클에서 입력받은 첫번째 문자열의 아스키코드 값과 두번째 아스키코드 값을 서로 더하여 7FF6A16C3000에 위치하는 hex값과 비교하는 절차를 가지고 있다.

이걸 분석 해보면 다양한 경우의 수가 나올것 같다. 우선 수기로 검증을 해보고 파이선 코드를 작성해 A부터 Z까지 넣었을 경우의 경우의 수를 전부 출력했다.

코드는 다음과 같다.

```python
a = [0xAD, 0xD8, 0xCB, 0xCB, 0x9D, 0x97, 0xCB, 0xC4, 0x92, 0xA1, 0xD2, 0xD7, 0xD2, 0xD6, 0xA8, 0xA5, 0xDC, 0xC7, 0xAD, 0xA3, 0xA1, 0x98, 0x4C]
start = 0x00
temp = 0x00

b = 0x41
while(b<0x5b):
        start = b;
        print("start : "+chr(b))
        for i in a:
                temp = i - start
                print (chr(start), end='')
                start = temp
        print("\n\n")

        b = b+1
```

위 코드는 [목표값 = 첫번째 값 + 두번째 값] 와 [두번째 값 = 목표값 - 첫번째 값] 이 동일하다는 간단한 식으로 작성하였다. 변수 a는 7FF6A16C3000에 들어있는 목표값 들이고 시작 아스키 코드를 A에 해당하는 hex 0x41로 주어 반복문을 돌렸다.

이후 결과는 다음과 같다.

<center><img src="/img/dreamhack-reb-basic-5/result.png" width="80%" height="80%"></center>

딱 보면 정답같아 보이는 부분이 있다.
A로 시작하는 부분이 플레그 값이다.