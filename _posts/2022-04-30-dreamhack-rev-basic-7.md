---
published: true
image: /img
layout: post
title: Dreamhack rev-basic-7 문제풀이
tags: [Dreamhack, reversing, wargame, writeup]
math: true
date: 2022-05-01 06:30
---

이번 문제는 드림핵 rev-basic-7 리버싱 문제를 풀어보자.

우선 basic-6과 동일하게 문자열을 입력 받고 정답이면 Correct, 아니면 Wrong을 출력한다. 바이너리 실행결과는 다음과 같다.

![binary_start](/img/dreamhack-rev-basic-7/binary_start.png)


우선 가장 먼저 해야할 일은 main함수를 찾는것이다. 윈도우 바이너리인 PE파일의 헤더 구조를 보면 매우 많은 정보들이 들어있다. 그것들 중 리버싱을 할떄 중점으로 봐야할 부분은 .text영역이다. 실질적으로 코드가 컴파일되어 저장되는 영역으로 대부분의 main함수는 이 영역 시작 지점과 인접하게 존재한다. 컴파일러의 보안 미티게이션의 추가로 메모리 주소 랜덤화가 자동으로 걸려 0x401000주소에 main이 들어가는 경우는 이젠 없을 것이다.

바이너리는 input : 과 Wrong 이라는 문자열을 출력했다. 따라서 문자열 검사를 통해 해당 문자열이 사용되는 지점을 찾아 BP를 걸어준다.

![main_asm](/img/dreamhack-rev-basic-7/main_asm.png)
```c++
00007FF7CEE21090 | 40:57                    | push rdi                                |
00007FF7CEE21092 | 48:81EC 30010000         | sub rsp,130                             |
00007FF7CEE21099 | 48:8B05 881F0000         | mov rax,qword ptr ds:[7FF7CEE23028]     |
00007FF7CEE210A0 | 48:33C4                  | xor rax,rsp                             |
00007FF7CEE210A3 | 48:898424 20010000       | mov qword ptr ss:[rsp+120],rax          |
00007FF7CEE210AB | 48:8D4424 20             | lea rax,qword ptr ss:[rsp+20]           |
00007FF7CEE210B0 | 48:8BF8                  | mov rdi,rax                             |
00007FF7CEE210B3 | 33C0                     | xor eax,eax                             |
00007FF7CEE210B5 | B9 00010000              | mov ecx,100                             |
00007FF7CEE210BA | F3:AA                    | rep stosb                               |
00007FF7CEE210BC | 48:8D0D 4D110000         | lea rcx,qword ptr ds:[7FF7CEE22210]     | rcx:"asdf", 00007FF7CEE22210:"Input : "
00007FF7CEE210C3 | E8 58000000              | call chall7.7FF7CEE21120                |
00007FF7CEE210C8 | 48:8D5424 20             | lea rdx,qword ptr ss:[rsp+20]           |
00007FF7CEE210CD | 48:8D0D 48110000         | lea rcx,qword ptr ds:[7FF7CEE2221C]     | rcx:"asdf", 00007FF7CEE2221C:"%256s"
00007FF7CEE210D4 | E8 D7000000              | call chall7.7FF7CEE211B0                |
00007FF7CEE210D9 | 48:8D4C24 20             | lea rcx,qword ptr ss:[rsp+20]           |
00007FF7CEE210DE | E8 1DFFFFFF              | call chall7.7FF7CEE21000                |
00007FF7CEE210E3 | 85C0                     | test eax,eax                            |
00007FF7CEE210E5 | 74 0F                    | je chall7.7FF7CEE210F6                  |
00007FF7CEE210E7 | 48:8D0D 3A110000         | lea rcx,qword ptr ds:[7FF7CEE22228]     | rcx:"asdf", 00007FF7CEE22228:"Correct"
00007FF7CEE210EE | FF15 94100000            | call qword ptr ds:[<&puts>]             |
00007FF7CEE210F4 | EB 0D                    | jmp chall7.7FF7CEE21103                 |
00007FF7CEE210F6 | 48:8D0D 33110000         | lea rcx,qword ptr ds:[7FF7CEE22230]     | rcx:"asdf", 00007FF7CEE22230:"Wrong"
00007FF7CEE210FD | FF15 85100000            | call qword ptr ds:[<&puts>]             |
00007FF7CEE21103 | 33C0                     | xor eax,eax                             |
00007FF7CEE21105 | 48:8B8C24 20010000       | mov rcx,qword ptr ss:[rsp+120]          |
00007FF7CEE2110D | 48:33CC                  | xor rcx,rsp                             |
00007FF7CEE21110 | E8 AB010000              | call chall7.7FF7CEE212C0                |
00007FF7CEE21115 | 48:81C4 30010000         | add rsp,130                             |
00007FF7CEE2111C | 5F                       | pop rdi                                 |
00007FF7CEE2111D | C3                       | ret                                     |
```

해당 문제는 올바른 값을 입력하면 Corrent라는 문구가 출력되며 입력한 값이 flag가 되는 문제이다. 따라서 바이너리 역분석을 통해 적절한 입력값을 찾아야 한다.

우선 테스트 값으로 `aaaaaaaaaaa`라는 문자열을 입력하고 이를 검사하는 부분을 찾아야 한다. 어셈블리를 살펴보면
```c++
00007FF7CEE210D9 | 48:8D4C24 20             | lea rcx,qword ptr ss:[rsp+20]           |
00007FF7CEE210DE | E8 1DFFFFFF              | call chall7.7FF7CEE21000                |
00007FF7CEE210E3 | 85C0                     | test eax,eax                            |
00007FF7CEE210E5 | 74 0F                    | je chall7.7FF7CEE210F6                  |
```
위와 같은 부분이 존재한다. 함수를 Call하고 test를 통한 eax 초기화 후 je를 통해 분기하는 것을 알 수 있다. 따라서 Call 하는 chall7.7FF7CEE21000 부분이 문자열을 검사하는 곳이라고 추측할 수 있다.

해당 부분의 어셈블리는 다음과 같다.
![input_cmp](/img/dreamhack-rev-basic-7/input_cmp.png)
```c++
00007FF7CEE21000 | 48:894C24 08             | mov qword ptr ss:[rsp+8],rcx            |
00007FF7CEE21005 | 48:83EC 18               | sub rsp,18                              |
00007FF7CEE21009 | C70424 00000000          | mov dword ptr ss:[rsp],0                |
00007FF7CEE21010 | EB 08                    | jmp chall7.7FF7CEE2101A                 |
00007FF7CEE21012 | 8B0424                   | mov eax,dword ptr ss:[rsp]              |
00007FF7CEE21015 | FFC0                     | inc eax                                 |
00007FF7CEE21017 | 890424                   | mov dword ptr ss:[rsp],eax              |
00007FF7CEE2101A | 48:630424                | movsxd rax,dword ptr ss:[rsp]           |
00007FF7CEE2101E | 48:83F8 1F               | cmp rax,1F                              |
00007FF7CEE21022 | 73 41                    | jae chall7.7FF7CEE21065                 |
00007FF7CEE21024 | 8B0424                   | mov eax,dword ptr ss:[rsp]              |
00007FF7CEE21027 | 83E0 07                  | and eax,7                               |
00007FF7CEE2102A | 48:630C24                | movsxd rcx,dword ptr ss:[rsp]           |
00007FF7CEE2102E | 48:894C24 08             | mov qword ptr ss:[rsp+8],rcx            |
00007FF7CEE21033 | 48:8B5424 20             | mov rdx,qword ptr ss:[rsp+20]           | [rsp+20]:"aaaaaaaaaaa"
00007FF7CEE21038 | 0FB6C8                   | movzx ecx,al                            |
00007FF7CEE2103B | 48:8B4424 08             | mov rax,qword ptr ss:[rsp+8]            |
00007FF7CEE21040 | 0FB60402                 | movzx eax,byte ptr ds:[rdx+rax]         |
00007FF7CEE21044 | D2C0                     | rol al,cl                               |
00007FF7CEE21046 | 0FB6C0                   | movzx eax,al                            |
00007FF7CEE21049 | 330424                   | xor eax,dword ptr ss:[rsp]              |
00007FF7CEE2104C | 48:630C24                | movsxd rcx,dword ptr ss:[rsp]           |
00007FF7CEE21050 | 48:8D15 A91F0000         | lea rdx,qword ptr ds:[7FF7CEE23000]     |
00007FF7CEE21057 | 0FB60C0A                 | movzx ecx,byte ptr ds:[rdx+rcx]         |
00007FF7CEE2105B | 3BC1                     | cmp eax,ecx                             |
00007FF7CEE2105D | 74 04                    | je chall7.7FF7CEE21063                  |
00007FF7CEE2105F | 33C0                     | xor eax,eax                             |
00007FF7CEE21061 | EB 07                    | jmp chall7.7FF7CEE2106A                 |
00007FF7CEE21063 | EB AD                    | jmp chall7.7FF7CEE21012                 |
00007FF7CEE21065 | B8 01000000              | mov eax,1                               |
00007FF7CEE2106A | 48:83C4 18               | add rsp,18                              |
00007FF7CEE2106E | C3                       | ret                                     |
```
어셈블리를 살펴보면 처음보는 명령어가 있다. 바로 rol이라는 연산인데, 쉬프트 연산의 일종이다. al 레지스터의 값을 cl레지스터 값 만큼 rol연산 해 al 레지스터에 저장하는 연산이다. 일반적인 쉬프트 연산에서 자리수를 넘어가는 값이 나오면 그 수는 그냥 버려지는것이 일반적이지만 ROL(왼쪽)과 ROR(오른쪽)연산의 경우 마지막 자리수에서 쉬프트 연산을 통해 자리올림이 발생했을 때, 반대쪽 자리로 옮겨 올라간 자리를 표시한다.

예를 들면 이렇다. al, cl은 각각 8비트의 길이를 가지고 있기 때문에 8비트로 설명을 해보겠다. 0000 0001을 ROL연산을 하면 0000 0010이 된다. 다만 0000 0001을 ROR연산을 하면 1000 0000이 된다. 이것이 일반적인 쉬프트 연산과 다른점이다.

해당 로직을 실행시켜 레지스터 값 변화와 같이 분석해보면 다음과 같다. 입력한 문자열이 저장된 스택 시작주소에서 반복문 횟수 만큼 값을 더해 결과적으로 각 자리 문자를 가져와 al 레지스터에 저장하는 역할을 한다. 이후 반복분의 횟수는 cl레지스터에 저장되며 al 레지스터에 저장된 값을 cl 레지스터 값 만큼 rol연산한 후 반복분 횟수와 XOR연산해 EAX에 저장한다. 이후 스텍에 저장된 정답 문자열 주소에 반복분 횟수 만큼 더한 자리의 값을 가져와 ECX에 저장한다. EAX와 ECX를 비교 후 같지 않으면 0을 리턴하고 같으면 jmp를 통해 다음 자리의 문자를 검사한다.

```
(각 자리수의 hex값 ROL 자리수(몇번째 자리인지)) XOR 자리수(몇번째 자리인지) == 비교대상 정답
위 값이 참일경우 반복문 동작, 거짓일 경우 0을 리턴하고 main함수에서 Wrong을 출력

# 입력된 문자열만큼 반복
# 자리수는 0부터 시작후 1씩 증가
# 결과 값을 7FF7CEE23000에 위치한 hex값과 비교해 정답 유무 확인
```
위 수식을 입력 받은 글자 수 만큼 반복하면서 비교한다. XOR의 경우 A ⊕ B = C와 A ⊕ C = B가 성립하므로 위 수식의 역을 구하면 다음과 같다.

```
(비교대상 정답 XOR 자리수) ROR 자리수 = 각 자리수의 hex값
```

위 수식을 검산하면서 간과했던 점이 al, cl 레지스터는 8비트 크기를 가지지만 윈도우 계산기의 기본 설정은 QWORD로 64비트의 자리수를 가지고 있다. 따라서 계산기로 검증하면서 byte로 설정을 바꾸어 계산을 진행했다.

이제 파이썬 코드를 작성해보자

```python
def ROL(data, shift, size=8):
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

a의 리스트 값은 입력값과 비교하는 데이터로 7FF7CEE23000 위치에서 가져왔다.

위 코드를 돌리면 플레그를 얻을 수 있다.