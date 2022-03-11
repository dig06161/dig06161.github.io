---
published: true
image: /img
layout: post
title: Dreamhack rev-basic-4 문제풀이
tags: [Dreamhack, reversing, ctf, writeup]
math: true
date: 2022-03-11 21:00
---
이번 문제는 리버싱문제 rev-basic-4이다.

우선 문제파일을 다운받으면 chall4.exe라는 프로그램이 나온다.
이 프로그램을 실행 시키면 iput : 이라는 문구와 함께 입력창이 활성화 된다. 이후 임의 문자열을 입력하면 Wrong이라는 문자열을 표시하고 종료된다.

이제 이 프로그램을 x64디버거로 열어보자.
<br>
<center><img src="/img/dreamhack-reb-basic-4/x64_main.png" width="80%" height="80%"></center>

이 프로그램의 main부분이다 input 이후에 
```C++
call chall4.7FF68AC31000
```
부분에서 문자열을 비교해 je 명령어로 Correct와 Wrong을 나눠준다.

이부분에 bp를 걸고 쭉 진행해 임의의 문자열을 입력하고 내용을 보자.

<center><img src="/img/dreamhack-reb-basic-4/func_in.png" width="80%" height="80%"></center>
위와같은 함수의 어셈블리가 보여진다.

간단히 돌려보며 해석을 해보자. 일단 임의값을 "AAAAA"로 입력했다.

```C++
jae chall4.7FF68AC31065
```
위 명령어를 통해 반복문을 진행하며 한글자씩 불러 검사하는 로직이다.

간단한 해석을 붙이면 입력받은 문자열을 AAAAA라고 했을때, 입력받은 문자열의 첫번째 글자를 오른쪽으로 4만큼 시프트 연산한 후 eax에 넣는다. 이후 입력받은 문자열의 첫번째 글자 A를 왼쪽으로 4만큼 시프트 연산 이후 F0와 AND연산 후 ecx에 넣는다. 그다음 eax와 ecx를 OR연산하여 eax에 넣고 7FF68AC33000에 위치한 문자열의 첫번째 글자를 불러 ecx에 넣는다. 이후 cmp명령어를 통해 서로 일치할 경우 jmp를 통해 두번째 문자열의 비교를 시작한다.
<br><br>
그러면 문자열 A를 직접 계산기를 이용해 계산해보자. A의 hex값은 41이며 이진수로 표현하면 0100 0001이다. 41를 위 계산을 통해나온 결과를 보면 0001 0100이라는 값을 가지고 있다.

다른 숫자로 몇번 더 계산을 해보면 hex또는 2진수의 좌우를 바꿔주는것을 볼수 있다.
<br><br>
그러면 7FF68AC33000에 위치한 문자열들의 hex값의 좌우를 바꿔주면 쉽게 flag를 구할 수 있다.

```C++

00007FF68AC33000  24 27 13 C6 C6 13 16 E6 47 F5 26 96 47 F5 46 27  $'.ÆÆ..æGõ&.GõF'  
00007FF68AC33010  13 26 26 C6 56 F5 C3 C3 F5 E3 E3 00 00 00 00 00  .&&ÆVõÃÃõãã.....  

```
위의 hex값을 파이썬을 이용해 좌우 자릿수를 바꿔보자.

```python
a = [0x24,0x27,0x13,0xC6,0xC6,0x13,0x16,0xE6,0x47,0xF5,0x26,0x96,0x47,0xF5,0x46,0x27,0x13,0x26,0x26,0xC6,0x56,0xF5,0xC3,0xC3,0xF5,0xE3,0xE3]

for i in a:
        b = i >> 4
        c = i % 0x10
        result = (c*0x10)+b
        print (chr(result), end='')
print("\n")
```
위 코드는 16진수 배열을 하나씩 불러와 자리수를 서로 바꿔준 후 아스키코드 문자열로 출력해주는 코드이다. 예를 들면 16진수 24를 42로 바꾸어 42에 해당하는 아스키코드인 "B"를 출력한다.

위 코드를 돌리면 플레그 값을 얻을 수 있다.