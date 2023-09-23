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
