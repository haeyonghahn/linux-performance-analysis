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
