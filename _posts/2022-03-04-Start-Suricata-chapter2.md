---
published: true
image: /img
layout: post
title: Suricata IPS, IDS 시작하기 2편(운용)
tags: [Suritaca, 수리카타, FW, ELK, splunk, IPS, IDS]
math: true
date: 2022-03-04 03:0
---

<h4 style="color:red">제가 공부하면서 적은 내용으로 틀린 내용이 있을수 있으니 지적해주시면 정정하겠습니다. 감사합니다.</h4><hr><br>

우선 시작하기 앞서서 Suricata IPS의 테스트를 위한 플라스크 기반 서버를 만들었다. 이는 실무에서 적용될때 서비스 하고자 하는 대상의 서버가 될것이다.

환경은 VM웨어의 Ubuntu 20.04버전을 기반으로 하였고 이 위에 Docker를 설치해 각 서버나 기능별로 컨테이너를 분리해 사용하려고 한다.

그냥 기본적인 확인을 위한 서버이니 메인에 접속시 Hello, World를 표시해주는 아주 간단한 서버이다. 컨테이너의 이름은 server 이며 

```shell
docker run -itd --name server -p 8080:8080 ubuntu:20.04 bash
```
위 명령어로 실행 한다.


도커에서 suricata IPS모드를 사용하기 위해서는 관리자 권한을 필요로 한다. NFQ를 위한 설정을 해줘야 하는데 이를 iptables로 적용하고 도커에서 iptables를 사용하기 위해서 관지자 권한이 필요하기 때문이다. 따라서 컨테이너를 실행시킬때 privileged 옵션을 주어야 한다.

``` shell
docker run -itd --name ips --privileged --net=container:server ubuntu:20.04 bash
```

위 명령어에서 --privileged 옵션으로 관리자 권한을 줬고, --net 옵션으로 server 컨테이너와 네트워크를 공유하도록 설정해주었다. --net 옵션을 주고 server와 ips 컨테이너에서 ip를 확인하면 똑같은 ip를 할당받은 것을 볼수 있다.

이제 suricata에서 제공하는 문서를 기반으로 설치를 해보자.
각 환경에서 코드를 빌드할 필요 없이, PPA방식으로 편하게 설치할 수 있다.

```shell
apt install iptables
apt install software-properties-common
add-apt-repository ppa:oisf/suricata-stable
apt update
apt install suricata jq
```
ppa저장소를 추가하고 suricata와 로그를 볼때 도움이 되는 jq를 설치한다.
ubuntu를 사용해본 사람이라면 여기까지 오래걸리지 않을 것이다.

```shell
suricata --build-info
```
위 명령어를 이용해 빌드 정보를 확인할수 있으며 나오는 정보는 다음과 같다.

<center><img src="/img/suricata2/suricata-build-info.png" width="80%" height="80%"></center>

사진을 보면 NFQueue support에 yes로 되어있지 않으면 IPS모드를 사용할 수 없다.

```shell
nano /etc/suricata/suricata.yaml
```
위 파일을 수정함으로써 suricata의 기본적인 설정을 진행한다.
필자는 nano에디터를 주로 쓰지만 각자 편한 에디터를 사용하면 된다.
위 설정으로 기본적인 HOME NETWORK와 각종 로그 등을 할수 있다. 기본적인 설정은 Suricata 최신 권장설정으로 되어있으나 환경에 따라서 바꿔야 할 부분을 수정한다.

<center><img src="/img/suricata2/suricata-home-net-config.png" width="80%" height="80%"></center>

위와같이 필자는 도커 네트워크 기반이므로 172.17.0.0/24를 주었고 아이피는 호스트나 환경에 따라 달라질수 있다.

```shell
port-groups:
    #HTTP_PORTS: "80"
    HTTP_PORTS: "8080"
    SHELLCODE_PORTS: "!80"
    ORACLE_PORTS: 1521
    SSH_PORTS: 22
    DNP3_PORTS: 20000
    MODBUS_PORTS: 502
    FILE_DATA_PORTS: "[$HTTP_PORTS,110,143]"
    FTP_PORTS: 21
    GENEVE_PORTS: 6081
    VXLAN_PORTS: 4789
    TEREDO_PORTS: 3544
```

홈 네트워크 설정 바로 아래있는 포트 그룹설정이다. 이 포스트에서 서버의 서비스 포트를 8080으로 설정했으니 변경해준다.
추가로 {% raw %}http-log{% endraw %}가 비활성화 되어있는데 이를 활성화 해준다.

```shell
- http-log:
      enabled: yes
      filename: http.log
      append: yes
      #extended: yes     # enable this for extended logging information
      #custom: yes       # enable the custom logging format (defined by customformat)
      #customformat: "%{%D-%H:%M:%S}t.%z %{X-Forwarded-For}i %H %m %h %u %s %B %a:%p ->
      #filetype: regular # 'regular', 'unix_stream' or 'unix_dgram'
```

수정 내용을 저장하고 탐지 룰 업데이트를 진행한다. 수리카타는

```
suricata-update
```

명령어로 통합 룰 업데이트를 지원한다. 다만 기본 룰들이 전부 alert로 되어있다. 이는 탐지는 하나 패킷드롭은 하지 않는다는 것이다. 옛날에는 오탐율 때문에 필요한 항목을 직접 드롭으로 바꿔주라는 글을 봤던 기억이 있다. 하나하나 확인하는게 좋지만 일단....문자열 치환을 이용해 alert를 전부 드롭으로 바꿔준다.

suricata 룰은 업데이트 되면서 통합되어 /var/lib/suricata/rules 위치에 suricata.rules 로 저장된다. 이를 정규식을 이용해 alert를 전부 drop으로 바꿔주면 ips모드에서 패킷 차단이 가능하다.

```
#정규식
:%s/^alert/drop/
```

치환전 룰인
<center><img src="/img/suricata2/alert-rules.png" width="80%" height="80%"></center>

에서 vim을 이용해 위 정규식을 이용하면
<center><img src="/img/suricata2/drop-rules.png" width="80%" height="80%"></center>

이런식으로 바꿔준 후 저장한다. 통합된 룰들을 확인해 보면 기본적으로 Suricata에서 제공하는 룰과 별도로 emergingthreats.net에서 제공하는 룰들이 같이 병합되어 있다.

이후 
NFQ 설정을 위해 iptables 옵션을 적용해준다.
이 설정을 적용하면 ips컨테이너와 같이 묶여있는 server 둘의 네트워크는 ips가 켜저있지 않으면 망 단절이 일어난다. 이게 인라인 IPS모드이다.

명령어는 아래와 같다

```shell
apt install iptables
iptables -I INPUT -j NFQUEUE
iptables -I OUTPUT -j NFQUEUE
```

이후 Suricata를 IPS모드로 작동시켜 준다.

```shell
suricata -c /etc/suricata/suricata.yaml -q 0
```

그럼 다음과 같은 화면이 뜰것이다.
<center><img src="/img/suricata2/suricata-ips-start.png" width="80%" height="80%"></center>

이후 

```shell
tail -f /var/log/suricata/fast.log
```

를 통해 탐지로그를 살펴보자.

이제 kali에서 hping3를 이용해 SYN Flooding을 시도해본다.
공격을 시도하는 순간 다음과 같은 로그들이 생성된다.

<h3 style="color:red">자주가는 카페의 기본 IP와 충돌이 나 docker 네트워크 브짓지 대역을 192.168.64.0/24로 수정했습니다 </h3><br>
<center><img src="/img/suricata2/flooding-drop.png" width="80%" height="80%"></center>
<br>
<center><img src="/img/suricata2/network-flow.png" width="80%" height="80%"></center>

위 사진을 보면 이상한 것이 있다. 분명히 막히긴 했는데 서버에서 공격지로 나가려다 비정상 트레픽으로 차단된 ACK 로그가 있다. 탐지 룰의 트레숄드 값에 의해 통과된 일부 트래픽에 대한 응답 트레픽이 탐지 된것 같다.

또 다음사진을 보면 Suricata 경고로 flow경고가 나오고 있다. Suricata에 설정된 처리 트래픽 양보다 더 많은 트레픽이 들어와 뜨는 경고다. 따라서 기업이나 트레픽 양이 많은 곳에 적용할 경우 설정을 수정하고 IDS모드로 미리 테스트를 거친 후에 IPS모드로 전환해야 할 것 같다.

hping3에서는 랜덤 ip를 통한 Flooding공격도 가능하다.
이를 테스트 해봤는데 결과는 다음과 같다.

<center><img src="/img/suricata2/flooding-drop-rand.png" width="80%" height="80%"></center>

로그를 보면 ET DROP Spamhaus DROP Listed Traffic Inbound group이라고 되어있다.
Spamhaus DROP이라는 그룹에서 DROP을 권장하는 IP리스트를 제공하고 있는데 이 리스트에 hping3의 임의ip가 포함되어 있는것 같다.

이 IPS를 적용하고서 생기는 문제점들이 있다. 이번에 생긴 문제점은 기본적으로 적용한 DROP룰 중에 APT update 서버로 향하는 트레픽을 막는 룰이 추가되어 있다.

따라서 이 룰을 Alert로 바꾸거나 주석 처리해야지 정상적인 apt update가 가능하다.

이번에 생긴 업데이트 문제 뿐만 아니라 각 룰들이 업데이트 되면서 다른 문제가 생길 가능성도 있다. 이러한 것들을 직접 핸들링 하고 테스트 하면서 수정해 줘야 한다.

