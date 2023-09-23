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
