---
published: ture
image: /img
layout: post
title: "Ghidra remote debugger"
excerpt: "Ghidra 원격 디버깅 사용하기"
tags: [PWN, Ghidra, remote_debug]
math: true
date: 2025-08-10 18:30
---

업무나 워게임 풀이에 있어서 Ghidra를 적극 사용중이다. 이런 Ghidra를 단순 디컴파일러로만 사용하기엔 아까워서 remote debug를 사용해봤다. IDA Pro는 정보가 많이 나오는데 Ghidra를 통한 remote debug 기능을 활용한 글을 거의 볼 수 없어서 가이드 느낌으로 적어본다. 개인적으로는 gdbserver나 qemu debug에도 활용 가능해 매우 유용할 것으로 판단했다.

필자의 환경은 분석용 데스크톱 ubuntu 24.04, 업무 또는 개인용 노트북으로 Windows 11을 사용한다.

Ghidra는 ubuntu에서 사용할 예정이고 RDP를 사용했다. 나중에는 Ghidra 서버를 통해 동적 remote 분석 가능 여부에 대해서도 테스트 할 예정이다.

Ghidra는 공식 github에서 11.4.1 빌드를 사용했고 snap 설치 환경에서는 권한 분리로 인해 gdb 등 일부 기능이 사용 불가능하다. 공식 github에서 다운 받아 실행하는 것을 추천한다.

gdb는 필수로 설치되어야 하고 Ghidra를 실행하는 계정에 대해 실행 권한을 가져야 한다. 

war game 문제 하나를 도커로 올려 gdb server static 빌드 한 바이너리와 같이 넣어준다.

![image.png](/img/Ghidra-remote-debugger/image.png)

prob는 대상 문제파일로 다음 gdbserver 명령을 통해 구동한다. 당연히 포트 expose나 라우팅은 도커 설정으로 잡아줘야 한다. 물론 방화벽도 마찬가지

```
pwn@dd809d5e20f8:~$ ./gdbserver :5555 prob
gdbserver: Error disabling address space randomization: Operation not permitted
Process prob created; pid = 40
Listening on port 5555
```

위 로그를 보면 gdbserver가 ASLR를 비활성화 해 동작하려 했지만 권한 문제로 실패했다고 뜬다. 필자는 실 문제 환경에서 분석하길 원해 따로 해결하진 않았다. 

gdbserver 가 5555를 디버그 포트로 하여금 접속 대기중이다. 이제 Ghidra의 debugger 기능을 활용해보자. 프로젝트를 생성하고 분석대상 바이너리를 추가해 Debugger 옵션으로 열어준다.

![image.png](/img/Ghidra-remote-debugger/image%201.png)

![image.png](/img/Ghidra-remote-debugger/image%202.png)

필자는 미리 auto Analysis를 통해 디컴파일을 진행했다. Debugger → Configure and Launch 대상 바이너리 이름 using… → gdb remote 를 선택해 연결 환경을 구성한다.

![image.png](/img/Ghidra-remote-debugger/image%203.png)

gdbserver에서 사용했던 5555번 포트로 설졍해줬다. 이 설정들은 분석 환경에 따라 자유롭게 설정하면 된다. 만약 x64가 아닌 mips나 aarch64 환경이라면 gdb command를 수정해 gdb-multiarch 를 사용하고 원하는 옵션을 추가할 수 있다.

![image.png](/img/Ghidra-remote-debugger/image%204.png)

이후 Launch 버튼을 누르면 다음과 같은 화면을 볼 수 있다.

![image.png](/img/Ghidra-remote-debugger/image%205.png)

gdbserver에 연결되었다. 아래 Terminal 창을 통해 실제 gdb 처럼 상호작용 가능하지만 0x101268 지점에 파란 불 부분을 더블 클릭하면 자동으로 bp를 걸어주는 등 자동화된 기능들이 많다. 

예를 들어 특정 값을 입력 후 메모리 맵에서 스택을 확인하고 싶은 경우, bp가 걸린 상태에서 F5를 눌러 컨티뉴 명령을 수행하고 값을 입력해 bp 지점까지 이동한다. 이후 Memory View에서 Track Stack Pointer를 선택해 스텍 포인터를 자동으로 따라다니며 확인 가능하고 수정 또한 가능하다.

![image.png](/img/Ghidra-remote-debugger/image%206.png)

당연하게도 bp 리스트 메모리 맵 확인 어셈 코드 수정 등 많은 기능들을 지원한다. 개인적으로 IDA Pro 보단 안전성이 떨어지지만 충분히 감안하고 사용할 만 하다. ni si 명령 등을 통해 움직이는 RIP 레지스터를 실시간으로 따라가며 디컴파일된 코드에서도 동일하게 추적한다.

이 분석 환경은 IDA PRO를 사용하지 못하는 환경에서는 대신 사용할 만 하다. 다만 아직은 불안정한 부분이 보이는데. 메모리쪽 문제로 오류가 간혹 뜨는 문제가 있는데 이 부분은 차차 업데이트로 나아질 것이라 기대한다.

매우 간단하게 Ghidra의 remote debugger 기능에 대해서 살펴봤다.

원격 환경은 네트워크를 많이 타기 때문에 속도나 PING이 좋지 않은 환경에서는 생각보다 불편하다. 나중에는 Ghidra 서버를 통해 원격 디버깅이 가능한지, WSL GUI 환경 Ghidra 등 여러 테스트를 진행하고 정리해볼 예정이다.

Windows에서 MSYS2 gdb를 활용하면 가능할 것으로 보이는데 이 부분은 아직 테스트하지 않았다. 굳이 윈도우에서 테스트할 필요성을 못느꼈다.

Ghidra remote debugger 기능을 더 많은 사람들이 사용해서 더욱 좋은 도구로 발전했으면 한다.