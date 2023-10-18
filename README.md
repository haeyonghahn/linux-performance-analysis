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

### 정리
1. `dmesg` 명령을 이용해서 OOME 에러 혹은 SYN Flooding 공격이 발생하지는 않는지 확인한다.
2. OOME 에러가 발생한다면 더 많은 메모리를 확보하고 SYN Flooding 공격이 발생하면 방화벽을 확인한다. 

## free
`free`라는 명령어는 시스템의 메모리 정보를 출력해주는 명령어이다. `-m` 옵션은 megabyte 단위로 표시하라는 의미이다. 그래서 메모리 전체량은 총 8GB 정도 되는 것을 볼 수 있다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/771eab71-91ca-4ba2-9304-0ad0f027485a)

### free와 available의 차이점
`free`는 어느 누구도 사용하고 있지 않은 메모리이다. `available`은 아무도 사용하지 않는 메모리가 아니고 누군가는 사용하고 있지만 어플리케이션이 필요로 한다면 할당 가능하다는 의미이다. `왜 free가 있고 available이 따로 있을까?`

### buff/cache
`buff`라는 것은 블록 디바이스가 가지고 있는 블록 자체에 대한 캐시이다. 그리고 `cache`라는 영역은 I/O 성능 향상을 위해서 사용하는 페이지 캐시를 의미한다. `페이지 캐시`는 리눅스 서버를 운영하기 위해서 꼭 알아야 하는 여러가지 개념 중 아주 중요한 개념이다.

### 페이지 캐시
어플리케이션 A가 `open`이라는 시스템 코드를 이용해서 블록 디바이스 안에 있는 `test.txt`라는 파일을 읽어서 사용자에게 보여주는 어플리케이션이 있다고 가정을 해보자. 그런데 사실은 블록 디바이스에서 바로 불러오는 것이 아니다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/15d2f7ad-b26b-4e92-9eb6-2d97f6806351)

실제로는 커널이 가운데에서 페이지 캐시라는 것을 이용해서 돌려준다. 블록 디바이스 안에 있는 `test.txt`파일의 내용을 페이지 캐시라고 불리는 곳에 저장을 하고 어플리케이션 A는 실제로는 페이지 캐시로부터 읽는 것이다. 위에서 알 수 있었던 것처럼 페이지 캐시는 buff/cache 영역 중 cache 영역을 의미한다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/5538d071-e262-442e-9791-6b67ee34b8b2)

한가지 애플리케이션이 또 더 만들어졌다고 가정해보자. 애플리케이션 B라는 애플리케이션이 만들어졌고 B도 `test.txt`라는 파일을 읽어드리려고 할 때 페이지 캐시가 없다라고 한다면, 애플리케이션 A가 읽은 것처럼 애플리케이션 B도 블록 디바이스에서 읽어야 한다. 만약에 애플리케이션 A가 수정을 하지 않고 읽기만 한다면 똑같은 내용을 읽어야 하는데, 블록 디바이스에서 똑같은 내용을 두 번 읽게 되면 어떻게 될까? 블록 디바이스는 디스크 드라이브이다. 아무리 빠르다고 해도 메모리보다 빠를 수 없다. 그래서 애플리케이션 B도 페이지 캐시로부터 파일의 내용을 읽게 되어 있다. 그래서 애플리케이션 A가 읽어가면서 저장했던 것이 B가 또 가져가는 것이다. 그래서 A가 맨 처음엔 `test.txt`파일을 읽을 때는 느릴 수 있다. 왜냐하면 페이지 캐시에 아무것도 없다면 블록 디바이스로부터 내용을 꺼내서 페이지 캐시에 저장하고 돌려주는 과정이 필요하가 때문이다. 그렇다면 `페이지 캐시를 통해서 무엇을 얻으려고 하는걸까?`

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/0d475f95-dd0d-4d3f-94cf-7116f9061e5c)

### I/O 성능 향상
한 번 읽었던 파일은 페이지 캐시에 저장함으로써 블록 디바이스가 아닌 메모리에서 파일의 내용을 가져오는 것이다. 메모리에서 파일을 가져오기 때문에 훨씬 속도가 빠르고 성능이 좋아진다.

아래의 그림은 업타임이 오래된 서버이다. 노랗게 표시된 부분이 캐시이다. 전체 32GB 중에 25GB를 페이지 캐시에 써야 할 정도로 I/O가 많이 일어나고 있다. 즉, I/O가 많이 일어나는 서버고 페이지 캐시를 25GB 정도를 채워놨기 때문에 그만큼 성능도 상당히 많이 올라갈 수 있을 것으로 기대를 할 수 있다. 이렇게 운영이 되는 대표적인 애플리케이션이 바로 `엘라스틱 서치`이다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/de5ecc22-75d8-4740-baa4-0a5bfcae0aae)

### available = free + buff/cache
buff/cache는 I/O 성능 향상을 위해 존재한다. 그래서 애플리케이션에서 메모리를 필요로 한다면 buff/cache 영역을 해제하고 애플리케이션이 사용할 수 있는 영역으로 바꾼다. 메모리를 필요로 하는데 메모리가 없다면 `OOM`이 일어난다. 그런데 OOM이 일어나기 전에 I/O 성능 향상을 위해 사용하는 buff/cache 영역이 남아 있으면 해당 영역을 해제하고 애플리케이션이 사용할 수 있는 영역으로 바꿔버린다. 메모리가 부족해서 OOM을 일으켜 프로세스를 죽여서 메모리를 확보하는 것보다는 buff/cache는 I/O 성능 향상을 위해 그냥 존재하는 것이니까 I/O 성능이 조금 떨어지더라도 메모리를 필요로 하는 애플리케이션의 메모리를 맞춰주는 것이 맞다. 그래서 모든 프로세스들이 메모리 걱정 없이 사용할 수 있도록 buff/cache 영역을 해제하고 애플리케이션을 사용할 수 있는 영역으로 바꿔버리는 것. 이것이 중요하다. 그래서 `free` 명령어를 입력했을 때 avaliable과 free를 따로 보여주는 것이다. 즉, `available`이 무엇이냐면 애플리케이션에게 줄 수 있는 메모리 양이다. `free`는 아무것도 사용하지 않은 것이니까 줄 수 있는 것이고 buff/cache는 I/O 성능 향상을 위해 임시로 있는 것이니까 해제해서 줄 수 있는 것이고. 그래서 등호가 완벽하게 들어맞진 않지만 대부분 free와 buff/cache를 합치면 얼추 available 양이 될 수 있다.

### swap
아래의 서버는 EC2 서버이기 때문에 swap이 없도록 기본 세팅이 되어 있다. 그래도 swap이 무엇인지 알아보자. `swap` 영역은 메모리가 부족한 상황에서 사용되는 가상 메모리 공간이다. 주로 `블록 디바이스`의 일부 영역을 사용한다. 그렇다면 swap 영역이 어떻게 사용되는지 알아보자.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/82a635e8-aa04-49d4-aa1a-c6bf94bd8b2e)

프로세스 A와 프로세스 B가 있고 메모리 주소가 있다고 보자. 메모리 주소가 5칸 있고 프로세스 A가 3개, 프로세스 B가 2개를 사용하고 있다고 하자. 프로세스 B가 커널에게 `메모리를 주세요`라고 요청을 하면, 메모리 주소가 꽉 찬 상태에서. buff/cache를 해제할 수도 있겠지만, swap 영역을 사용하게 될 때를 생각해보자.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/58b1d6a7-ee95-464f-817c-6bd7102c298b)

그러면 커널은 메모리 영역 중에 B에게 줄 메모리 영역을 확보해야 한다. 그 중에서 프로세스 A의 0번 메모리 영역을 swap 영역으로 넘겨 버리는 결정을 하게 될 수도 있다. 그래서 이렇게 swap 영역으로 넘기는 것을 swap out 이라고 부른다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/7874f814-e868-42b8-825e-1e4af089b6e1)

이렇게 swap out이 되면 0번 메모리에 있는 내용이 swap 영역으로 옮겨가고 메모리 공간이 비워지고 해당 영역을 B에게 주게 된다. 그렇다면 `프로세스 A가 0번 영역을 원한다면 어떻게 될까?`

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/7b0297c7-5b2a-471b-a44d-34b8c896df2f)   

0번 메모리 참조 요청이 프로세스 A로부터 발생하면 커널에게 요청이 가고 커널은 0번 메모리를 swap 영역으로 넘겼으니까 프로세스 B의 2번 메모리를 swap out 시켜버린다. 그래서 메모리 주소를 비우고 해당 영역을 swap 영역에 있는 프로세스 A의 0번 메모리를 메모리 주소로 가지고 온다. 이것을 `swap in`이라고 한다. 그리고 다시 0번을 채워놓고 0번 메모리 주소를 프로세스 A에게 전달하게 된다. swap in이 되어도 swap 영역에서 0번 메모리는 지워지지 않는다. 이 영역을 `swap cached`라고 부른다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/6a292e61-6d0c-47f2-845f-154f5187f7ed)   
![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/d7b9979d-ff94-4d56-a311-020ce213a12c)   
![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/0daba4ee-1bd8-4991-af68-feffbef8e4e3)

### swap 영역이 사용되면 성능 저하 발생
swap in, swap out이 될 때 메모리를 참조한다. 프로세스 입장에서는 메모리 참조였는데, 사실은 알고보면 I/O가 일어나는 것이다. swap 영역은 메모리가 아니고 블록 디바이스이기 때문에 I/O가 일어난다. 즉, swap 영역에 사용이 된다는 것은 성능 저하가 발생한다는 것이다.

### 성능이 떨어지더라도 OOM Killer에 의해 죽는 걸 막을 것이냐 vs 성능이 떨어질 바에는 OOM Killer에 의해 죽는 게 나을 것이냐
두 가지는 애플리케이션 환경이 어떻게 동작 중이냐가 가장 중요하다. 그런데, 최근 트랜드는 swap 영역 비활성화가 트랜드이다. 위에서 볼 수 있듯이 EC2를 만들면 디폴트는 swap이 없다.

### 정리
- free 명령을 이용해서 시스템의 메모리 사용 현황을 볼 수 있다.
- available은 실질적으로 프로세스에게 할당할 수 있는 메모리 양이다.
- buff/cache가 높다면 I/O가 빈번하게 발생한다는 의미이다.
- swap 영역을 사용한다는 것은 메모리가 부족하다는 신호이다.

## df
디스크 여유 공간 및 아이노드 공간을 확인하는 명령어이다. `-h`라는 옵션을 이용해서 파티션들을 볼 수 있다. 파티션 별로 사용 중엔 Used 영역, 그리고 전체 크기, 현재 더 사용할 수 있는 크기 및 퍼센트 정도 사용했는지, 그리고 각각의 파일 시스템이 어디에 마운트 되어있는지를 확인할 수 있다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/7f4a737c-ca83-41aa-9e64-02a6024b6371)

### 디스크 사용량 모니터링은 기본 중의 기본
### 파일 시스템이 100%가 된다면?
루트 파티션이 꽉 차게 된다면, 보이는 것처럼 자동 완성 조차 동작하지 않는다. `No space left on device` 에러가 발생한다. 해당 에러가 발생하면서 디스크 풀이 되었다고 볼 수 있다. 특히 루트 파티션에 문제가 생겼을 경우 최악의 경우에는 ssh 접속도 불가능해지는 경우가 생긴다. 그래서 디스크 사용량 모니터링은 꼭 해야 되는. 잊으면 안되는 아주 중요한 모니터링이다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/5e20f61b-0430-49b5-8b28-f759d5058c5f)

### 디렉터리 별 사용량 측정
어떤 디렉터리가 사용을 많이 했는지 알아야 한다. 그래서 사용량 측정을 통해 디스크 사용량을 확보하기 위해서 어느 디렉터리에 무엇을 치우면 될 수 있을지 확인할 수 있다. 그럴 때는 
`du -sh ./*`라는 명령어를 사용한다. 루트 디렉터리에서 해당 명령을 입력하게 되면 루트 디렉터리 하단의 모든 디렉터리를 하나하나씩 봐서 보여주도록 되어 있다. 만약에 루트 권한이 아니면 `sudo` 명령어를 이용해서 확인을 해봐야한다. 

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/708e5a41-8466-4a23-ac99-fccd5003d09c)

### 파일을 지웠는데 용량이 안늘어난다?!

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/e4f7c78a-3ffd-4565-a5ed-efad083a03e5)

이러한 문제점을 이해하기 위해선 `파일 핸들`이라는 것을 알아야 한다. 예를 들어, A라는 프로세스를 참조하고 있으며 참조값들을 기록하고 있는 것이다. PID 2917 프로세스가 test 파일에 대한 파일 핸들을 가지고 있기 때문에 지워졌지만 지워지지 않았다. `lsof` 명령어는 파일 핸들을 확인하는 명령어이다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/b0f9950c-164d-4403-933b-2e147cb68380)

그래서 프로세스를 종료하고 나면 실제로 6.2GB가 유지되도록 2GB 정도 확보가 된다. 이렇게 앞에서 이야기했던 것처럼 파일을 지웠는데도 용량이 그대로라면 `lsof` 명령어로 `파일 핸들`을 사용하고 있는지 확인해야 한다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/4342a94d-64fd-4b45-8f34-682eef34d111)

### inode 사용률
inode는 파일 또는 디렉터리에 대한 메타데이터를 저장하는 구조체이다. 이 말은 파일과 디렉터리의 개수이다. 만약에 파일이 10개면 inode도 10개이다. 결국에는 파일과 디렉터리가 몇 개 있느냐가 inode의 개수를 결정하는 것이다. 그래서 inode는 `파일과 디렉터리가 얼마나 많은가`를 알려준다. `df -h` 명령어는 사용량, 용량을 이야기하는 것이고 inode는 갯수를 의미한다. inode는 `df -i` 명령어를 통해서 확인할 수 있다. 그래서 루트 디렉터리는 127,433개의 파일과 디렉터리가 존재한다라고 볼 수 있다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/e3a1a24c-2212-420e-976d-e720c2dbfc39)

### fs.file-max
해당 명령어는 동시에 파일을 몇개까지 열 수 있는가를 정의하는 값이지, fs.filemax는 커널 파라미터이고 동시에 파일의 핸들이 몇 개까지 열 수 있는가를 정희하는 값이지 파일 시스템에 몇 개까지 저장할 수 있느냐 관련된 커널 파라미터는 아니다.

### 정리
- df 명령어를 이용해서 디스크 여유 공간 및 inode 공간을 확인할 수 있다.
- 간혹 파일 핸들이 남아 있어서 파일을 지웠지만 지워지지 않는 경우가 있다. 이 때는 lsof 명령어를 통해 파일 핸들을 확인한다.
- inode는 파일과 디렉터리의 개수로 생각하면 된다. inode에도 최대값이 있으며 그 이상 파일을 만들 수 없다는 의미이다.

## top
top 명령어는 프로세스들의 상태와 cpu 메모리 사용률 등을 확인할 수 있다. free 명령어를 통해서 확인할 수 있는 것도 top 명령어를 통해서 어느정도는 확인할 수 있다. `uptime` 명령어에서 확인했던 로드 에버리지도 확인할 수 있고 테스크 갯수, 프로세스 갯수 정보도 확인할 수 있다. 그리고 cpu가 지금 얼마나 사용하고 있는지 사용자 cpu, 시스템 cpu, Buff, Cache, Pre, Total 이러한 정보도 볼 수 있다. 그리고 어떤 프로세스들이 지금 동작하고 있는지 PID와 유저 정보 그리고 커맨드 정보도 확인할 수 있다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/c039746c-ac18-493b-8776-c53b3b7d16ae)

### hot key 사용법
hot key를 이용하면 자세한 정보를 원활하게 볼 수 있다. 아래의 사진을 보면 cpu를 퉁 쳤을 때 cpu 에버리지를 봤을 때 49.4% 정도 된다. 많이 안 쓰고 있다라고 생각할 수 있지만, cpu를 나눠서 보면 하나만 일을 하고 있고 하나는 놀고 있다. cpu 불균형이 있는 것이다. 그래서 이렇게 `키패드 숫자 1`을 통해서 각각의 CPU로 봐야 되는 것이 필요하다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/edbcb3e1-d408-4ec3-ab6c-195d64f2b848)

`영문자 d`를 누르면 인터벌을 변경할 수 있다. 기본은 3초 단위이다. 우리가 `top` 명령어를 그냥 두면 3초에 한 번씩 화면이 리프레시 된다. 뭔가 성능에 문제가 있다고 봤을 때 3초는 꽤 길다. 그래서 우리가 성능을 분석할 때는 1초 단위로 빠른 템포로 봐야 할 필요가 있다. 그래서 `d`키를 누르면 change delay라는 명령어가 보이고 여기에 숫자 1을 넣어서 엔터를 치면 화면이 1초에 한 번씩 1초마다 리프레시가 된다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/c776f466-d0fc-4b9c-b3df-a771e033677e)

그래서 만약 `개별 CPU의 사용량을 1초 단위로 보고 싶다면` `1 누르고 d 누르고 1 입력 후 엔터`치면 된다.

### top 명령을 통해 알 수 있는 것들
`CPU Usage`이다. `us`, `sy`, `ni` 이러한 것들이 있는데, 제일 눈여겨 봐야 할 것은 `us`와 `wa`이다.   
- `us` : user를 의미하며, 프로세스의 일반적인 CPU 사용량
- `wa` : waiting을 의미하며, I/O 작업을 대기할 때의 cpu 사용량

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/ae2364f4-fd4c-4890-8776-3e56e19b2260)

`us가 높다?` CPU를 많이 쓰고 있구나 해석할 수 있고 더 좋은 CPU를 가진 서버로 의사결정할 수 있다. 더 높은 헤르츠(Hz)를 가지거나 CPU가 더 많거나 이러한 것들을 확인할 수 있다. 반대로 `wa가 높다?` I/O가 많구나 해석할 수 있고 더 좋은 블록 디바이스를 가진 서버로 변경할 수 있다. EC2를 예로 든다면, gp2에서 gp3로 EBS를 옮긴다거나 더 많은, 더 높은 IOOPS를 가진 블록 디바이스로 가서 `wa`를 줄일 수 있도록 한다.

### CPU가 고르게 사용되고 있나요?
애초부터 CPU를 하나만 쓰는. 예를 들어, 싱글 스레드 애플리케이션이라면 정상이겠지만, 만약 그렇지 않다. 자바 프로세스가 CPU 하나만 잡고 돌아가고 있다면 이상한 것이다. 이러한 부분은 분석을 해봐야 한다. 전체 CPU가 작더라도 제대로 CPU를 못 쓰고 있는건 분명히 문제이기 때문에 해결해야 한다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/ae65f283-b78c-4f03-bf18-629ff88c33a2)

### 프로세스 상태
`top` 명령어를 쳤을 때 확인할 수 있는 프로세서들의 상태가 있다. `D`(uninterruptible sleep), `R`(running), `S`(sleeping), `Z`(zombie) 4가지가 있다. D와 R은 앞에서 살펴봤던 uptime 명령어에서 이야기 했던 것처럼 각각 I/O와 CPU 바운드된 작업을 포함하기 때문에 해당 두 개의 프로세스들의 개수는 로드 에버리지에 포함이 되고` D상태`에 있다는 것은 `vmstat` 명령어에서 확인했을 때처럼 B상태의 프로세스를 D로 이해하면 된다. 그리고 `R상태`는 running을 의미하는 것이고 CPU를 많이 사용하는 일반적인 프로세스를 의미한다. `S상태`는 특별하게 어떤 작업을 하지 않고 잠을 자고 있는 것이고 로드 에버리지에 포함되지 않는다. `Z상태`는 해를 끼치지는 않지만 이슈를 일으킬 수 있는 가능성이 있기 때문에 살펴보아야 한다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/65d13855-78cc-454c-999a-b5618038ab8c)

### 부모 프로세스가 죽었는데도 살아있는 자식프로세스
`Z상태` 즉, 좀비 프로세스는 부모 프로세스가 죽었는데도 살아있는 자식 프로세스를 의미한다. `좀비 프로세스는 시스템 리소스를 사용하진 않는다. 하지만 PID 고갈을 일으킬 수 있다.` 왜냐하면 wating 상태가 되서 CPU를 받으러 간다거나 이러한 일은 발생하지 않는다. 하지만 좀비 프로세스도 프로세스이기 때문에 PID 할당을 받는다. 그래서 좀비 프로세스가 너무 많아서 PID. 어떤 프로세스에게 줘야 할 PID가 부족해지면 프로세스를 수행할 수 없는 현상이 발생할 수도 있다. 그래서 커널 파라미터를 이용해서 PID MAX, 32,768개. 해당 시스템은 동시에 존재할 수 있는 프로세스가 32,768개라는 것이다. 그래서 만약 좀비 프로세스가 32,000개 이상이라면 위험해질 수 있다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/40685fb9-5ec7-4818-b7c9-c7f2ce4834be)

### 정리
- `top` 명령을 이용해서 프로세스의 상태, CPU, 메모리 사용량을 알 수 있다.
- CPU 사용량 중 `us`가 높다면 CPU를 많이 사용하는 워크로드를, `wa`가 높다면 I/O를 많이 사용하는 워크로드를 의미한다.
- 멀티코어 환경이라면 모든 CPU를 사용하고 있는지 확인해야 한다.
- 프로세스의 상태에는 `D`, `R`, `S`, `Z` 상태가 있다.

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
그 이유는 `keepalive_timeout`이라는 HTTP 1.1의 스펙 중 하나이며 연결을 유지하는 설정때문이다. nginx의 환경설정 파일을 `cat`을 이용해서 보면 `keepalive_timeout`이 65로 되어 있는 것을 볼 수 있다. 이것은 웹서버마다 다르지만 nginx의 경우 기본값이 65초라는 것을 알 수 있다. 그래서 HTTP 요청에 대해서 커넥션을 새로 맺지 않아도 65초 안에 새로운 요청을 기다리게 되어 있다. 65초가 지나면 더이상 연결을 유지할 필요가 없다고 생각하고 nginx 서버가 연결을 먼저 끊는다. 그래서 `TIME_WAIT` 소켓이 생성된다. `TIME_WAIT` 소켓이 많이 생겼을까에 대한 고민도 생각해볼 필요가 있다. 즉, 연결을 먼저 끊고 있는건 아닐까에 대해 생각해볼 필요가 있다. `TIME_WAIT가 왜 생기는지, 어떻게 하면 줄일 수 있을지 고민하는 것이 효과적인 대응이 될 수 있다.`

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/27f5a78e-bfa2-4c20-8c17-ffa1f63aafbc)

### 곤란한 상태는?
`CLOSE_WAIT`이다. `SCENE-RECEIVED`가 많다면 `SYN Flooding` 공격이다. 서버의 애플리케이션 관점에서 보면 `CLOSE_WAIT`는 애플리케이션 이상 동작을 하고 있다고 봐야 한다. `정상적으로 소켓을 정리하는 등 연결을 끊기 위한 동작을 하지 못한다는 의미이다.`

### 정리
- `netstat` 명령어를 이용하면 네트워크 연결 정보를 확인할 수 있다.
- 커넥션의 상태와 종단 간 IP 정보 등 서버의 네트워크 연결 정보를 확인할 수 있다.
- `LISTEN`, `ESTABLISHED`, `TIME_WAIT`는 흔히 만나게 되는 소켓 상태이다. `CLOSE_WAIT`가 발생한다면 꼭 원인을 확인하고 조치해야 한다.

## tcpdump
`tcpdump`는 네트워크 패킷을 수집하고 이걸 가지고 어떤 문제가 있는지 우리가 분석할 수 있는 도구이다. 말 그대로 네트워크 패킷의 흐름을 전반적으로 볼 수 있다. 예를 들어, 어떤 서버에서 어떤 서버로 통신이 이루어질 때 특정 구간을 `tcpdump`를 통해서 어떤 패킷들이 어떻게 이동하는지 확인할 수 있다. 하지만 `tcpdump`를 아무런 옵션없이 입력하면 해석하기가 힘들다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/adeb06da-b747-41e1-a91c-526e25500b3e)

### -nn 옵션
프로톨콜과 포트 번호를 숫자 그대로 표현해준다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/e14faf71-b011-4fc5-911b-8858f399a437)

### -vvv옵션
출력 결과에 더 많은 정보를 담아준다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/c1b44b94-8b8a-49a1-8d48-1c4c0e16523f)

### -A 옵션
패킷의 내용도 함께 출력한다. 아래의 정보는 22번 포트이기 때문에 암호화가 돼서 아무 의미 없는 글자로 보인다. 하지만 http 라면 body의 내용들이 보이게 된다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/4ae02a2f-d23d-4852-b24b-d5197cf3c3d2)

### 트러블 슈팅은 목적지와 포트가 존재한다.
트러블 슈팅은 목적지와 포트가 존재한다. 그래서 tcpdump로 모든 패킷을 다 보지 않아도 된다. 그리고 그런 옵션으로 포트라는 옵션이 존재한다. `tcpdump -vvv -nn -A port 80` 예를 들어, `port 80 and host 10.1.1.1` 이라고 한다면 10.1.1.1로 나가면서 포트가 80인 패킷들을 잡게 된다. 아래와 같이 80 포트로 잡아보면 패킷을 잡을 수 있는데 실질적으로 `3-way handshake`과정이 tcp 덤프로 보이는 것이다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/0be0f050-23b6-4f2d-870a-6f3bacb1c11a)

같은 방식으로 연결을 끊을 때 쓰는 `4-way handshake` 과정도 볼 수 있다. 

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/63c66cee-3c7e-45b7-a2ac-1f5127ca6335)

### 좀 더 편하게 분석할 수 없을까?
이때 사용되는 도구가 `wireshark(와이어샤크)`라는 도구이다. tcpdump와 wireshark는 아래와 같은 구조를 가지고 있다. tcpdump가 dump파일을 생성한다. 그러면 pcap 파일이 만들어지고 해당 파일을 로컬로 가지고와서 wireshark라는 명령으로 wireshark라는 프로그램을 열어서 분석을 하게 된다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/5e9cf856-be5c-42cf-8f48-d036a7cc5b77)

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/2fa53480-8cea-42df-bdcb-8a00e0c9da5b)

### 정리
- `tcpdump` 명령을 이용해서 네트워크 패킷을 수집하고 분석할 수 있다.
- `-vvv -nn -A` 옵션을 이용해서 tcpdump를 좀 더 효율적으로 사용할 수 있다.
- `host`, `port` 문구를 이용해서 특정 목적지, 특정 포트로 필터링할 수 있다.
- `tcpdump`로 `pcap` 파일을 생성하고 `wireshark`로 분석할 수 있다.

## 실제 성능 분석 및 트러블 슈팅 사례 살펴보기
|사례|설명|
|----|------------------------|
|nginx miss configuration|nginx workers 설정 미숙으로 인한 장애|
|간혈적인 네트워크 응답 지연|Read Timeout으로 인한 장애|
|간혈적인 커넥션 종료 에러|Keepalive Timeout으로 인한 장애|
|간혈적인 타임아웃|JVM Full-GC로 인한 장애|
|EC2 CPU 이상 동작|물리서버의 CPU 불량으로 인한 장애|

## 간혈적인 네트워크 응답 지연
### 장애 현상
간혈적으로 API 호출 시 타임아웃이 발생한다.

### 메트릭 수집
`netstat`을 통해 네트워크 연결은 잘 되어 있는지 확인하고 `tcpdump`를 통해 패킷들의 흐름을 수집 후 분석한다.

### tcpdump 낚시
간혈적인 타임아웃은 긴 호흡으로 패킷을 수집해야 합니다. `-G`라는 옵션은 파일을 갱신하는 옵션이다. 3600초니까 1시간에 한번씩 파일을 새로 만든다.   
`tcpdump -vvv -nn -A -G 3600 -w /var/log/tcpdump/$(hostname)_%Y%m%d-%H%M%S.pcap`   
그래서 타임아웃이 발생한 순간의 pcap을 분석할 수 있도록 한다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/6606d301-6406-43fe-8d4d-309dbcd0f221)

2894번 패킷과 2897번 패킷을 보면, 2894번 패킷은 POST 요청이다. POST 요청을 보내고 3초 후에 FIN 패킷이 날라갔다. 내가 응답을 받지도 않았는데 종료를 시켜버렸다. 그런데 정작 2899번 패킷을 보면 응답패킷이 돌아왔다. 응답패킷이 돌아왔지만 FIN으로 닫아버렸으니 RST 패킷을 보내버리게 된다. 즉, 294번 패킷에서 2987번 패킷 사이 3초를 기다렸는데 3초 동안 안왔으니 FIN 보내버린 것이다. 그런데 POST에 대한 응답이 5초 후에 도착했다. 이것이 `타임아웃` 현상이다.

### 클라이언트의 타임아웃 설정
타임아웃이란 무엇이고 어떤 종류가 있는지 이것을 어떻게 설정해야 되는지 알아보자. `Timeout`이란 현재 상태가 정상이라고 판단할 때까지 얼마나 기다릴 것인가? 기다리는 시간이 지나면 에러를 표현하는 것이다.

### Connection Timeout과 Read Timeout
`Connection Timeout`은 종단 간의 연결을 처음 맺을 때의 시간이다. `Read Timeout`은 종단 간 연결을 맺은 후 데이터를 주고 받을 때의 시간이다.

### Round Trip Time (RTT)
패킷이 종단 간을 이동할 때 걸리는 시간, 즉 물리적 거리에 따른 시간이다. 서울과 부산을 왕복하는 것보다 서울과 오레곤을 왕복할 때 시간이 더 걸린다. Timeout 설정 시 응답에 소요되는 시간과 RTT를 함께 고려해야 한다. 아래의 그림을 보자. A 서비스가 B 서비스를 호출한다고 가정해보자. RTT는 10ms 이다. 죽었다 깨어나도 10ms보다 더 빨리 올 수 없다. 물리적으로 10ms가 필요하기 때문이다. B 서비스에서는 요청을 받은 후에 그 요청에 대한 처리, 응답을 만들기 위한 프로세싱 시간이 있을 것이다. 그것이 평균적으로 200ms라고 한다면, 우리는 최소한 무조건 어떠한 모든 요청도 210ms보다 죽었다 깨어나도 짧아질 수 없다. 그러니까 210ms 정도는 기다려야 지금 정상인지 아닌지 판단할 수 있다는 이야기이다. 그래서 Read Timeout은 210ms보다 커야한다. `그래서 RTT를 꼭 고려를 해야한다.`

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/f672cddf-41ff-4279-bea8-db925bb0aa7b)

### 두 가지 고려해야 할 것
RTT를 모를 때, 패킷이 유실되었을 때이다. `RTT를 모를 때`는 종단 간의 커넥션을 처음 맺을 때이다. 패킷이 한 번도 흘러본 적이 없으니 시간이 얼마나 걸릴지 알 수 없다. 그래서 이 때 쓰는 것이 `InitRTO`라는 개념이다. RTT를 모를 때 사용하는 커널의 패킷 초기 타임아웃 값이다. 예를 들어, A랑 B와 통신할 때 B한테 패킷을 쐈을 때 RTT를 모르는 상태이다. 그러면 최소한 `InitRTO`에 있는 값 정도는 기다려 보자라고 하는 것이 소스코드에 하드코딩되어 있다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/77e0d750-1298-417c-aef2-611f66ebbe12)

### Connection Timeout 설정
3-Way Handshake 과정 중 `최소한 한 번의 패킷 유실 정도는 방어`할 수 있어야 한다. 그래서 `3초 (1초 + RTT 고려)` 정도로 설정하는 것이 좋다.

### Retransmission Timeout (RTO)
이번엔 `패킷이 유실되었을 때`를 알아보자. 패킷에 대한 응답이 RTO 이내에 도착하지 않으면 유실로 간주한다. RTO는 RTT를 기반으로 계산된다. RTT가 아무리 짧아도 RTO의 최소 값은 있다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/2af7358e-dc09-4b4b-b3a6-890a43a206c7)

A와 B가 있다고 가정했을 때 A가 보낸 GET 요청이 만약에 유실되었다고 한다면, 다시 보내야한다. 그때 RTO가 최소 200ms. RTT가 몇 이건 상관없이 200ms 보다 작아도 최소 200ms는 기다렸다가 GET 요청을 다시 보낸다고 할 수 있다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/9db22d18-dc10-43f0-8e39-510393d5e7d0)

B가 A로부터 요청을 받고 나서 그거에 대한 응답을 처리하기까지 300ms라고 가정을 해보자. 그런데 중간에 유실이 되었다. 그래서 B 입장에서 내가 보낸 응답이 유실이 되었네라고 생각해서 200ms를 기다렸다가 응답을 다시 보내준다. 그리고 여기에 대한 ACK를 받아서 통신 잘했다고 끝을 낸다. 그런데 RTT가 10ms라고 가정을 하면 310ms만의 처리가 가능했을 것이다. 그런데 중간에 패킷 유실이 한 번 돼서 200ms가 더 걸렸고 총 510ms가 소요되었다. 해당 이야기를 한 이유는 `Read Timeout 설정을 어떻게 할 것이냐`에 대한 질문 때문이다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/e99d04a7-a204-4777-9820-d8aa10850652)

### Read Timeout 설정
프로세싱 시간을 고려하고 최소한 한 번의 패킷 유실을 방어할 수 있어야 한다. 그래서 `프로세싱 시간 + RTO 고려`한 시간 정도로 설정하는 것이 좋다. 물론 이것은 프로세싱 시간이 있기 때문에 각자의 환경에 맞게 충분히 고려하여 설정해야 한다. 프로세싱 시간이 `프로세싱 시간 + RTO 고려`한 시간이 넘는 경우라면 타임아웃 에러가 빈번하게 발생할 수 있다. 앞에서 살펴봤던 것을 다시 보면,

2894번 패킷에서 2897번 패킷까지 3초가 걸렸다. 결국엔 Read Timeout은 3초였다는 이야기이다. 그런데 이 요청 같은 경우에는 프로세싱 시간에 5초가 소요되었기 때문에 클라이언트 입장에서는 Read Timeout을 6초 정도로 설정을 했어야 했다. 그러면 Timeout이 안나고 제대로 응답을 받았을 것이다. 물론 프로세싱 시간이 너무 오래 걸려서 비정상일 수도 있고 프로세싱 시간을 줄여서 대응을 합시다라고 할 수도 있다. 그런데 그것은 우리가 호출하는 쪽에서 어쨌든 해당 문제를 해결해야 하는 것이고 여기에 시간이 너무 오래 걸린다면 그때까지 우리가 굳이 Read Timeout을 3초로 잡아서 이렇게 프로세싱이 오래 걸릴 때 타임 아웃을 굳이 낼 필요가 없다. 그러니까 클라이언트 입장에서는 즉, 이 서비스를 호출하는 입장에서는 에러가 나지 않도록 최선을 다할 필요가 있고 그런 면에서 보면 5초나 걸리는 프로세싱 시간이 단축되는. 최적화가 되기 전까지는 Read Timeout을 조금 더 늘려서 대응할 필요가 있다. 그리고 서비스에 더 좋은 영향을 줄 수 있다. 결국엔 에러가 나면 서비스를 바라보는 고객 입장에서는 에러 화면을 바라볼 수 밖에 없을 것이고 조치가 되기 전까지는 조금 느리더라도 응답을 받는 게 좋은 선택이 아닐까 생각해볼 수 있다.

![image](https://github.com/haeyonghahn/linux-performance-analysis/assets/31242766/a3546777-1763-481b-bb65-55e245a56894)

### Lesson Learned
환경에 적합한 Timeout 값을 설정하자. 그렇지 않으면 불필요하게 에러가 발생할 수 있다.
