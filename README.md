# linux-performance-analysis
## 성능 분석의 기본 명령어
|명령어|역할|
|----|------------------------|
|uptime|시스템의 가동 시간, Load Average 확인|
|dmesg|커널 메시지 확인 (OOME 발생 여부, SYN Flooding 여부|
|free|메모리 사용 확인|
|df|디스크 여유 공간 및 inode 공간 확인|
|top|프로세스들의 cpu 사용률 확인|
|netstat|네트워크 연결 정보 확인|

## uptime
시스템의 가동 시간과 로그인한 사용자 수, Load Average 확인이 가능하다.    

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/b6384fe1-7986-45b4-8f1f-0f639022c635)

서버가 동작한지 몇 분이 지났는지, 서버에 들어와있는 사용자가 몇 명인지, 로드 에버리지를 확인할 수 있다.

### Load Average
서버가 받고 있는 부하(`얼마나 일을 하고 있는지`) 평균이다. 단위 시간 (1분, 5분, 15분) 동안의 R과 D 상태의 프로세스 개수이다. Load Average는 상대적인 값이다. 두 서버가 똑같이 Load Average가 1 이라고 해도 `CPU가 한 개일 때와 두 개일 때의 의미`가 다르다. `Load Average > CPU 개수`인 경우 현재 처리 가능한 수준에 비해 많은 수의 프로세스가 존재한다는 것을 의미한다. 

### CPU 개수는 어떻게 알 수 있을까?
![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/d7809ccd-3c83-4627-bd55-44929a61842e)

CPU가 1개 이기 때문에 Load Average가 1보다 작으면 괜찮다. 

### 하지만 프로세스 개수보다 CPU 개수가 많으면 항상 괜찮을까?
그럴수도, 아닐수도 있다. 그렇다면 Load Average가 높은 이유는 무엇일까? Load Average는 `R`과 `D` 상태의 프로세스 개수이다. `R`은 `CPU 위주의 작업`이고 `D`는 `I/O 위주의 작업`이다. 그래서 똑같이 프로세스 개수가 1이어도 R상태가 맞냐, D상태가 맞냐에 따라서 그 의미의 대응이 달라진다.    
- R 상태의 프로세스가 많다면 CPU 개수를 늘리거나 스레드 개수를 조절해야 한다. 
- D상태의 프로세스가 많다면 IOPS가 높은 디바이스로 변경하거나 처리량을 줄여야 한다. 예를 들어, EC2를 쓴다면 EBS 붙여서 쓴다. 즉, 더 많은 IO를 처리할 수 있는 환경을 만들어줘야 로드 에버리지가 내려간다.    
그렇다면 R이 많은지 D가 많은지 어떻게 알까?

### vmstat 
![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/a2ffa7b5-f95b-4985-b83c-8f60e757fec0)

해당 명령어를 치면 `PROCS` 컬럼이 있다. 그 중에서도 `R`컬럼과 `B`컬럼이 있는데, `R`컬럼이 R상태, `B`컬럼이 D상태로 이해하면 된다. 위 예시로 든 서버도 로드 에버리지가 1이다. 왜냐하면 R상태가 1이니까 cpu를 많이 필요로 하는 프로세스가 많다는 의미이다. 

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/499cd211-a0c6-47f8-be89-a7df9854bdd6)

위의 예시는 R상태는 거의 없고 B상태가 많다. 위의 서버의 로드 에버리지는 1.5 정도될 것이다. 어쨋든 IO에서 무언가 병목에 걸려있기 때문에 CPU의 개수를 늘려주는 것이 의미가 없다. 더 좋은 IOPS를 가지고 있는 EBS로 교체를 해주는 것이 성능 향상에 도움이 될 것이다. 

### 정리
1. `uptime` 명령을 이용해서 얼마나 많은 부하를 받고 있는지 확인한다.
2. CPU 개수보다 많은 부하를 받고 있다면 어떤 종류의 프로세스 때문인지 확인한다.
3. `vmstat` 명령의 `procs` 컬럼을 확인한다. r이면 `cpu bound`, b이면 `io bound`이기 때문에 각각에 맞는 조치를 취해주자.

## dmesg
커널에서 발생하는 다양한 메시지들을 출력해주는 명령이다. `dmesg -T` 명령을 통해 타임스템프를 편하게 볼 수 있다. 

### 어떤 메시지를 봐야할까?
`OOME(Out-Of-Memory Error)`와 `SYN Flooding` 두 가지가 있다. 두 가지 경우에 D메시지를 통해서 메시지를 확인할 수 있다. 

### OOME
가용한 메모리가 부족해서 더이상 프로세스에게 할당해 줄 메모리가 없는 상황이다. 그래서 OOME 상황이 되면 커널은 내부적으로 `omkiller`라는 걸 동작시킨다. `omkiller`의 역할은 메모리를 굉장히 많이 쓰고 있다면 누군가 가지고 있는 메모리를 회수해야 한다. 결국엔 프로세스를 종료시키는 것이다. 메모리를 많이 사용하고 있는 프로세스를 종료시켜서 이 프로세스가 사용하고 있는 메모리를 다시 커널에게 돌려줘서 메모리를 필요로 하는 다른 프로세스에게 주는 것이다. `그렇다면 어떻게 종료시킬 프로세스를 선택할까?`

### oom_score
종료시킬 프로세스를 잘 골라야 하고 그걸 고르기 위한 기준으로 oom underscore라는 oom 점수를 가지고 있다. OOM Killer가 종료시킬 프로세스를 선택하는 기준이 된다. `스코어가 높은 프로세스`가 더 먼저 종료된다. 그러면 oom 스코어는 어떻게 확인할 수 있을까?

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/814867a0-31f9-4f3f-98e5-ca9033fae058)

`proc`라는 디렉토리가 존재한다. 프로세스들의 메타데이터를 볼 수 있는 디렉토리이다. 여기서 2526번 PID를 가지고 있는 프로세스에 들어와서 `cat oom_score` 스코어롤 보면 666점이다. 해당 프로세스는 배쉬 프로세스이다. 다 죽어야 되는데 OOM이 나서서 누군가를 죽여야 하는데, 그냥 배쉬를 죽였다 그러면 그냥 쉘 하나 잠깐 닫히는 것이다. 시스템적으로 큰 문제는 없을 것이다. 대신에 메모리를 확보할 수 있다면 좋은 것이다. 그렇게 해서 OOM 스코어를 기준으로 어떤 프로세스를 죽일지 결정을 하게 된다. 예를 들어, 

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/d70c5a1b-df1d-4ad7-8f27-2fecb637d511)

Out of Memory Error가 발생하는 소스를 gcc로 컴파일해서 실행을 해보면, omkiller 메세지 보이고 메모리 관련된 정보가 확인된다. OOM 킬러가 동작해서 프로세스를 죽였다. 죽여서 메모리를 확보했다고 확인할 수 있다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/c204b057-dad3-4440-bf3b-995c017f36d9)
![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/d08404e1-cec8-4953-943c-c441fbbaaabb)

그래서 만약에 우리가 D메시지를 가지고 OOM이 발생했나 확인하고 싶을 떄, `dmesg -TL | grep -i oom`해당 옵션과 함께 D메시지를 통해서 확인을 할 수가 있다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/03419ef2-c8fe-4f73-b8f0-873205823c2b)

### SYN Flooding
공격자가 대량의 SYN 패킷만 보내서 소켓을 고갈시키는 공격이다. 씬 플러딩에 대해서 이해를 하려면 먼저 TCP 3-way handshake 를 이해해야 한다. 씬 플러딩 공격을 막기 위해 SYN 패킷의 정보들을 바탕으로 `SYN Cookie`를 만든다. 그리고 그 값을 `SYN + ACK`의 시퀀스 번호로 만들어서 응답한다.     
- SYN Cookie가 동작하면 `SYN Backlog`에 쌓지 않는다. 그래서 자원 고갈 현상이 발생하지 않는다.
- 하지만 `TCP Option 헤더`를 무시하기 때문에 Windows Scaling 등 성능 향상을 위한 옵션이 동작하지 않는다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/e9f67ed8-2c7b-4058-b889-8e7ec133740d)

그런데 요즘에는 AWS 를 사용하면 ALB 도 있고 와프 같은 것들도 있고 AWS 쉴드 같은 것들이 있어서 방어가 잘 되는 편이라서 이런 메시지를 많이 볼 수 없겠지만 혹시라도 다른 환경이라면 가능성이 있기 때문에 D메시지를 이용해서 확인을 해볼 수 있다.

## 정리
1. `dmesg` 명령을 이용해서 OOME 에러 혹은 SYN Flooding 공격이 발생하지는 않는지 확인한다.
2. OOME 에러가 발생한다면 더 많은 메모리를 확보하고 SYN Flooding 공격이 발생하면 방화벽을 확인한다. 

## netstat
네트워크 연결 정보를 확인할 때 사용되는 명령어이다. `-napo` 옵션을 주면 소켓 정보를 확인할 수 있다. 상단이 네트워크 소켓이고 하단이 유닉스 소켓이다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/00bb1e65-f84d-452a-810e-50066662817e)

### 어떻게 해석할 수 있을까?
![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/d3069c43-f1e4-476a-936a-5fd7a8613d9e)

`10.1.10.13 주소의 22번 포트와 1.240.235.98 주소의 51566 포트가 연결되어 있다. 그리고 그 연결은 PID 2528이며 sshd 라는 이름의 프로세스가 사용 중이다`라고 해석할 수 있다.

### 소켓의 상태
`TCP 3-way handshake`는 연결을 맺을 때 모든 네트워크 문제에 시작이 되는 개념이기 때문에 다시 한 번 살펴보자.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/3920c359-0323-40b6-98b5-f4d4158eb77f)

A와 B 두 종단이 있고, 한쪽이 연결읠 먼저 받는 입장이다. B가 연결을 받아서 `LISTEN` 상태가 된다. A에서 B 쪽으로 연결을 맨 처음 맺기 위해서는 `SCENE` 패킷을 보낸다. 그래서 보낸 후 A에서는 `SCENE-SENT` 상태가 된다. 그리고 `LISTEN` 상태에 있던 포트는 `SCENE-RECEIVED`라는 상태가 된다. `SCENE`을 받았다라는 것이다. 이러한 상태가 된 후에 `3-way handshake`처럼 `SCENE` + `ACK`를 다시 보내게되고 최종적으로 서로 `ACK`를 주고 받으면 `ESTABLISHED` 상태가 된다.

다음으로 `TCP 4-way handshake`를 알아보자. (연결을 끊을 때)

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/56e7d60d-2f6c-4c91-855e-3e28faae62a5)

앞서 `TCP 3-way handshake`를 통해 `ESTABLISHED`된 상태이다. 둘 중에 먼저 끊으려는 쪽이 `FIN`을 보낸다. `FIN`을 보내면 받은 쪽은 소켓이 `CLOSE_WAIT`상태로 바뀌고 바로 `ACK`를 보낸다. 그리고 받은 쪽에서는 `FIN_WAIT2` 상태가 된다. 다음으로 `LAST_WAIT` 상태. `CLOSE_WAIT` 상태에서 `LAST_WAIT` 상태로 넘어갈 때는 특별한 패키지가 필요 없다. 시간이 흐르면 자동으로 `LAST_WAIT` 상태가 된다. 그 다음에 정리를 하면서 `FIN`을 한 번 더 보내면 `TIME_WAIT` 상태가 되면서 이제 '정말 연결을 끊겠다'라는 의미로 `ACK`를 보내며 끊어진 상태가 된다.

### 가장 자주 보게 될 상태는?
`LISTEN`, `ESTABLISHED`, `TIME_WAIT` 세 개의 소켓들은 `netstat` 명령어를 이용해서 봤을 때 상당히 자주 볼 수 있는 상태이다.   

### 바로 연결이 끊어지지 않고 깜빡깜빡 기다리다가 서버가 먼저 연결을 끊을까?
그 이유는 `keepalive_timeout`이라는 HTTP 1.1의 스펙 중 하나이며 연결을 유지하는 설정때문이다.
