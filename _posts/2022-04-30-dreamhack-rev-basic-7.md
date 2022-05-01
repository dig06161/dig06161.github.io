---
published: true
image: /img
layout: post
title: Dreamhack rev-basic-7 문제풀이
tags: [Dreamhack, reversing, ctf, writeup]
math: true
date: 2022-05-01 06:30
---

이번 문제는 드림핵 rev-basic-7 리버싱 문제를 풀어보자.

여담으로 드림핵에 단계별로 올라와 있는 문제들은 시간이 얼마나 걸리든 한번씩 다 풀어볼 생각이다.

우선 basic-6과 동일하게 문자열을 입력 받고 정답이면 Correct, 아니면 Wrong을 출력한다.

이 바이너리를 x64 디버거를 통해 main함수 부분을 확인해 보자.

```c++
00007FF6ADDC1090 | 40:57                    | push rdi                                | rdi:&"ALLUSERSPROFILE=C:\\ProgramData"
00007FF6ADDC1092 | 48:81EC 30010000         | sub rsp,130                             |
00007FF6ADDC1099 | 48:8B05 881F0000         | mov rax,qword ptr ds:[7FF6ADDC3028]     |
00007FF6ADDC10A0 | 48:33C4                  | xor rax,rsp                             |
00007FF6ADDC10A3 | 48:898424 20010000       | mov qword ptr ss:[rsp+120],rax          |
00007FF6ADDC10AB | 48:8D4424 20             | lea rax,qword ptr ss:[rsp+20]           |
00007FF6ADDC10B0 | 48:8BF8                  | mov rdi,rax                             | rdi:&"ALLUSERSPROFILE=C:\\ProgramData"
00007FF6ADDC10B3 | 33C0                     | xor eax,eax                             |
00007FF6ADDC10B5 | B9 00010000              | mov ecx,100                             |
00007FF6ADDC10BA | F3:AA                    | rep stosb                               |
00007FF6ADDC10BC | 48:8D0D 4D110000         | lea rcx,qword ptr ds:[7FF6ADDC2210]     | 00007FF6ADDC2210:"Input : "
00007FF6ADDC10C3 | E8 58000000              | call chall7.7FF6ADDC1120                |
00007FF6ADDC10C8 | 48:8D5424 20             | lea rdx,qword ptr ss:[rsp+20]           |
00007FF6ADDC10CD | 48:8D0D 48110000         | lea rcx,qword ptr ds:[7FF6ADDC221C]     | 00007FF6ADDC221C:"%256s"
00007FF6ADDC10D4 | E8 D7000000              | call chall7.7FF6ADDC11B0                |
00007FF6ADDC10D9 | 48:8D4C24 20             | lea rcx,qword ptr ss:[rsp+20]           |
00007FF6ADDC10DE | E8 1DFFFFFF              | call chall7.7FF6ADDC1000                |
00007FF6ADDC10E3 | 85C0                     | test eax,eax                            |
00007FF6ADDC10E5 | 74 0F                    | je chall7.7FF6ADDC10F6                  |
00007FF6ADDC10E7 | 48:8D0D 3A110000         | lea rcx,qword ptr ds:[7FF6ADDC2228]     | 00007FF6ADDC2228:"Correct"
00007FF6ADDC10EE | FF15 94100000            | call qword ptr ds:[<&puts>]             |
00007FF6ADDC10F4 | EB 0D                    | jmp chall7.7FF6ADDC1103                 |
00007FF6ADDC10F6 | 48:8D0D 33110000         | lea rcx,qword ptr ds:[7FF6ADDC2230]     | 00007FF6ADDC2230:"Wrong"
00007FF6ADDC10FD | FF15 85100000            | call qword ptr ds:[<&puts>]             |
00007FF6ADDC1103 | 33C0                     | xor eax,eax                             |
00007FF6ADDC1105 | 48:8B8C24 20010000       | mov rcx,qword ptr ss:[rsp+120]          |
00007FF6ADDC110D | 48:33CC                  | xor rcx,rsp                             |
00007FF6ADDC1110 | E8 AB010000              | call chall7.7FF6ADDC12C0                |
00007FF6ADDC1115 | 48:81C4 30010000         | add rsp,130                             |
00007FF6ADDC111C | 5F                       | pop rdi                                 | rdi:&"ALLUSERSPROFILE=C:\\ProgramData"
00007FF6ADDC111D | C3                       | ret                                     |
```

main의 코드는 위와 같다. 이부분에서 00007FF6ADDC10DE에 있는 call chall7.7FF6ADDC1000 구문을 통해 문자열을 비교하는 함수로 들어가게 된다.

AAAAAA를 입력하고 함수로 들어가 내부 코드를 살펴보자

```c++
00007FF6ADDC1000 | 48:894C24 08             | mov qword ptr ss:[rsp+8],rcx            | [rsp+8]:"%256s"
00007FF6ADDC1005 | 48:83EC 18               | sub rsp,18                              |
00007FF6ADDC1009 | C70424 00000000          | mov dword ptr ss:[rsp],0                |
00007FF6ADDC1010 | EB 08                    | jmp chall7.7FF6ADDC101A                 |
00007FF6ADDC1012 | 8B0424                   | mov eax,dword ptr ss:[rsp]              |
00007FF6ADDC1015 | FFC0                     | inc eax                                 |
00007FF6ADDC1017 | 890424                   | mov dword ptr ss:[rsp],eax              |
00007FF6ADDC101A | 48:630424                | movsxd rax,dword ptr ss:[rsp]           |
00007FF6ADDC101E | 48:83F8 1F               | cmp rax,1F                              |
00007FF6ADDC1022 | 73 41                    | jae chall7.7FF6ADDC1065                 |
00007FF6ADDC1024 | 8B0424                   | mov eax,dword ptr ss:[rsp]              |
00007FF6ADDC1027 | 83E0 07                  | and eax,7                               |
00007FF6ADDC102A | 48:630C24                | movsxd rcx,dword ptr ss:[rsp]           |
00007FF6ADDC102E | 48:894C24 08             | mov qword ptr ss:[rsp+8],rcx            | [rsp+8]:"%256s"
00007FF6ADDC1033 | 48:8B5424 20             | mov rdx,qword ptr ss:[rsp+20]           | [rsp+20]:"젌/冷\x01"
00007FF6ADDC1038 | 0FB6C8                   | movzx ecx,al                            |
00007FF6ADDC103B | 48:8B4424 08             | mov rax,qword ptr ss:[rsp+8]            | [rsp+8]:"%256s"
00007FF6ADDC1040 | 0FB60402                 | movzx eax,byte ptr ds:[rdx+rax]         |
00007FF6ADDC1044 | D2C0                     | rol al,cl                               |
00007FF6ADDC1046 | 0FB6C0                   | movzx eax,al                            |
00007FF6ADDC1049 | 330424                   | xor eax,dword ptr ss:[rsp]              |
00007FF6ADDC104C | 48:630C24                | movsxd rcx,dword ptr ss:[rsp]           |
00007FF6ADDC1050 | 48:8D15 A91F0000         | lea rdx,qword ptr ds:[7FF6ADDC3000]     |
00007FF6ADDC1057 | 0FB60C0A                 | movzx ecx,byte ptr ds:[rdx+rcx]         |
00007FF6ADDC105B | 3BC1                     | cmp eax,ecx                             |
00007FF6ADDC105D | 74 04                    | je chall7.7FF6ADDC1063                  |
00007FF6ADDC105F | 33C0                     | xor eax,eax                             |
00007FF6ADDC1061 | EB 07                    | jmp chall7.7FF6ADDC106A                 |
00007FF6ADDC1063 | EB AD                    | jmp chall7.7FF6ADDC1012                 |
00007FF6ADDC1065 | B8 01000000              | mov eax,1                               |
00007FF6ADDC106A | 48:83C4 18               | add rsp,18                              |
00007FF6ADDC106E | C3                       | ret                                     |
00007FF6ADDC106F | CC                       | int3                                    |
00007FF6ADDC1070 | 48:8D05 E91F0000         | lea rax,qword ptr ds:[7FF6ADDC3060]     |
00007FF6ADDC1077 | C3                       | ret                                     |
```

문자열을 비교하는 함수의 내부의 어셈블리는 다음과 같다.

간단하게 로직을 확인해보자. 00007FF6ADDC1044의 위치에서 al과 cl을 ROL연산을 한다. 간단히 설명해보겠다. 여기서 봐야할 점은 ROL 연산이 있다. 일반적인 쉬프트 연산에서 자리수를 넘어가는 값이 나오면 그 수는 그냥 버려지는것이 일반적이지만 ROL(왼쪽)과 ROR(오른쪽)연산의 경우 마지막 자리수에서 쉬프트 연산을 통해 자리올림이 발생했을 때, 반대쪽 자리로 옮겨 올라간 자리를 표시한다.

예를 들면 이렇다. al, cl은 각각 8비트의 길이를 가지고 있기 때문에 8비트로 설명을 해보겠다. 0000 0001을 ROL연산을 하면 0000 0010이 된다. 다만 0000 0001을 ROR연산을 하면 1000 0000이 된다. 이것이 일반적인 쉬프트 연산과 다른점이다.

로직을 분석해 간단히 수식화 하면 다음과 같다.

```
(각 자리수의 hex값 ROL 자리수) XOR 자리수 = 비교대상 정답

# 입력된 문자열만큼 반복
# 자리수는 0부터 시작후 1씩 증가
# 결과 값을 7FF6ADDC3000에 위치한 hex값과 비교해 정답 유무 확인
```
위 수식을 입력 받은 글자 수 만큼 반복하면서 비교한다. XOR의 경우 A ⊕ B = C와 A ⊕ C = B가 성립하므로 위 수식의 역을 구하면 다음과 같다.

```
(비교대상 정답 XOR 자리수) ROR 자리수 = 각 자리수의 hex값
```

위 수식을 검산하면서 간과했던 점이 al, cl 레지스터는 8비트 크기를 가지지만 윈도우 계산기의 기본 설정은 QWORD로 64비트의 자리수를 가지고 있다. 따라서 계산기로 검증하면서 byte로 설정을 바꾸어 계산을 진행했다.

이제 파이썬 코드를 작성해보자

```python
def ROL(data, shift, size=32):
        shift %= size
        remains = data >> (size - shift)
        body = (data << shift) - (remains << size )
        return (body + remains)

def ROR(data, shift, size=8):
        shift %= size
        body = data >> shift
        remains = (data << (size - shift)) - (body << size)
        return (body + remains)

a = [0x52, 0xDF, 0xB3, 0x60, 0xF1, 0x8B, 0x1C, 0xB5, 0x57, 0xD1, 0x9F, 0x38, 0x4B, 0x29, 0xD9, 0x26
, 0x7F, 0xC9, 0xA3, 0xE9, 0x53, 0x18, 0x4F, 0xB8, 0x6A, 0xCB, 0x87, 0x58, 0x5B, 0x39, 0x1E]
b = 0
temp = ""

for i in a:
        temp = (i ^ b)
        temp = ROR(temp, b)
        print(chr(temp), end='')
        b = b + 1
""
```

찾아보니 파이썬에는 ROL, ROR 연산 함수가 없어 https://bbolmin.tistory.com/133 블로그에서 코드를 빌려왔다. 감사하게도 ROL, ROR 코드를 작성해 올려주셨다.


위 코드를 보면 ROR 함수에 size 값이 8인것을 볼 수 있다. 이 또한 al의 크기인 8비트를 맞춰주기 위해 코드를 수정했다.

위 코드를 돌리면 플레그를 얻을 수 있다.