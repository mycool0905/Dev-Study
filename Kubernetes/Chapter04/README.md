# Chapter04


## 레플리케이션


### 레플리케이션에 관해서

`파드(Pod)`는 배포 가능한 기본 단위이다. 그렇지만 수동으로 안정적인 상태로 유지되게 관리하는 것은 쉽지 않다. 그래서 __쿠버네티스에서는 파드를 직접 생성하는 일 없이 여러가지 리소스를 이용해 파드를 관리할 수 있도록 한다__. `레플리케이션컨트롤러` 또는 `디플로이먼트`와 같은 유형의 리소스가 그 역할을 한다.


## 파드를 안정적으로 유지하기

일단 레플리케이션을 알아보기 전에 `라이브니스 프로브`에 대해서 알아본다.

### 라이브니스 프로브(liveness probe)

쿠버네티스는 `라이브니스 프로브(liveness probe)`를 통해 컨테이너가 살아 있는지 확인할 수 있다. 파드의 스펙에 각 컨테이너의 라이브니스 프로브를 지정할 수 있다. 쿠버네티스는 주기적으로 프로브를 실행하고 프로브가 실패할 경우 컨테이너를 다시 시작한다.

- 쿠버네티스의 세 가지 컨테이너 프로브 실행 메커니즘
  + `HTTP GET 프로브`: 지정한 IP 주소, 포트, 경로에 HTTP GET 요청을 수행한다. 프로브가 응답을 수신하고 응답 코드가 오류(HTTP STATUS: 4xx, 5xx)를 나타내지 않는 경우에 프로브가 성공했다고 간주한다. 서버가 오류 응답 코드를 반환하거나 전혀 응답하지 않으면 프로브가 실패한 것으로 간주돼 컨테이너를 다시 시작한다.
  + `TCP 소켓 프로브`: 컨테이너의 지정된 포트에 TCP 연결을 시도한다. 연결에 성공하면 프로브가 성공한 것이고, 그렇지 않으면 컨테이너가 다시 시작된다.
  + `Exec 프로브`: 컨테이너 내의 임의의 명령을 실행하고 명령의 종료 상태 코드를 확인한다. 상태 코드가 0이면 프로브가 성공한 것이다. 모든 다른 코드는 실패로 간주된다.
  

기존 kubia 이미지에서 억지로 실패하게 만드는 이미지인 `luksa/kubia-unhealthy`를 이용해 실습 가능하다. (5번의 Request를 받으면 애플리케이션은 HTTP STATUS 500을 반환하기 시작한다.)

#### kubia-liveness-probe.yaml
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:
  - image: luksa/kubia-unhealthy     # 약간 문제가 있는 애플리케이션
    name: kubia
    livenessProbe:
      httpGet:                       # HTTP GET을 수행하는 라이브니스 프로브
        path: /                      # HTTP 요청 경로
        port: 8080                   # 프로브가 연결해야 하는 네트워크 포트
```

이 파드 디스크립터는 쿠버네티스가 주기적으로 "/" 경로와 8080포트에 HTTP GET을 요청해서 컨테이너가 정상 동작하는지 확인하도록 httpGet 라이브니스 프로브를 정의한다. 이런 요청은 컨테이너가 실행되는 즉시 시작된다.

- ```$ kubectl logs <pod name> --previous```
  + 이전 컨테이너가 종료된 이유를 파악하기 위해 현재 컨테이너의 로그 대신 이전 컨테이너의 로그를 보기
- ```$ kubectl describe po <pod name>```
  + 여기서 보면 다시 시작된 횟수와 컨테이너가 재생성된 이유를 볼 수 있다.

![image](https://user-images.githubusercontent.com/43199318/112426455-5450b000-8d7b-11eb-9b61-d5368ede6001.png)

![image](https://user-images.githubusercontent.com/43199318/112426507-63cff900-8d7b-11eb-85cc-74bbc5946c92.png)

위에 적혀 있듯이, 컨테이너가 종료되면 완전히 새로운 컨테이너가 생성된다. 동일한 컨테이너가 다시 시작되는 것이 아니다.

#### 라이브니스 프로브의 추가 속성

![image](https://user-images.githubusercontent.com/43199318/112426702-bc06fb00-8d7b-11eb-88df-17bef64abcd2.png)

- 지연(delay): delay=0s, 컨테이너가 시작된 후 바로 프로브가 시작된다.
- 제한 시간(timeout): timeout=1s, 컨테이너가 1초 안에 응답해야 한다.
- 기간(period): period=10s, 컨테이너는 10초마다 프로브를 수행한다.
- 실패 횟수(failure): #failure=3, 프로브가 3번 연속 실패하면 컨테이너가 다시 시작된다.


#### 초기 지연을 추가한 라이브니스 프로브: kubia-liveness-probe-initial-delay.yaml
``` yaml
livenessProbe:
  httpGet:
    path: /
    port: 8080
  initialDelaySeconds: 15       # 첫 번째 프로브 실행까지 15초를 대기한다.
```

초기 지연을 설정하지 않으면 프로브는 컨테이너가 시작되자마자 프로브를 시작한다. 이 경우 대부분 애플리케이션이 요청을 받을 준비가 돼 있지 않기 때문에 프로브가 실패한다. 실패 횟수가 실패 임곗값을 초과하면 요청을 올바르게 응답하기 전에 컨테이너가 다시 시작된다. __애플리케이션 시작 시간을 고려해 초기 지연 설정을 꼭 해야한다는 점을 명심해야한다__.

#### 효과적인 라이브니스 프로브
1. 특정한 라이브니스 프로브를 위한 API를 설계하여 특정 URL 경로에 요청하도록 할 수 있다.(예:/health)
2. 애플리케이션 내부만 체크하도록 한다.(예: 프론트엔드 웹서버의 라이브니스 프로브는 웹서버가 백엔드 데이터베이스에 연결할 수 없을 때 실패를 반환하면 안된다. 근본적인 원인이 데이터베이스 자체에 있을 수 있다.)
3. 라이브니스 프로브는 너무 많은 연산 리소스를 사용해서는 안되고, 완료하는 데 너무 오래 걸리지 않아야 한다. 기본적으로 프로브는 비교적 자주 실행되며 1초 내에 완료돼야 한다. (너무 많은 일을 하는 프로브는 컨테이너의 속도를 상당히 느려지게 만든다.)
4. 프로브에 재시도 루프를 구현하지 말고, 프로브 실패 임곗값을 설정해라.(#failure)


### 레플리케이션컨트롤러

__`레플리케이션컨트롤러`는 쿠버네티스 리소스로서 파드가 항상 실행되도록 보장한다.__ 어떤 이유에서든 파드가 사라지면, 쉽게 말해 클러스터에서 노드가 사라지거나 노드에서 파드가 제거된 경우, 레플리케이션컨트롤러는 사라진 파드를 감지해 교체 파드를 생성한다.

![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/04fig01_alt.jpg)

위 그림은 파드가 두 개 있는 노드가 다운될 때 어떤 일이 일어나는지 보여준다. 파드 A는 직접해 관리되지 않는 파드인 반면, 파드 B는 레플리케이션컨트롤러에 의해 관리된다. 노드에 장애가 발생한 후 레플리케이션컨트롤러는 사라진 파드 B를 교체하기 위해 새 파드(Pod B2)를 생성하지만 파드 A는 완전히 유실된다.


### 레플리케이션컨트롤러의 동작 이해

레플리케이션컨트롤러는 실행 중인 파드 목록을 지속적으로 모니터링하고, 특정 유형(type)의 실제 파드 수가 의도하는 수와 일치하는지 항상 확인한다. 이런 파드가 너무 적게 실행 중인 경우 파드 템플릿에서 새 복제본을 만든다. 너무 많은 파드가 실행 중이면 초과 복제본이 제거된다.


#### 의도하는 수의 복제본보다 많은 복제본이 생기는 경우
 - 누군가 같은 유형의 파드를 수동으로 만든다.
 - 누군가 기존 파드의 유형(type)을 변경한다.
 - 누군가 의도하는 파드 수를 줄인다.

여기서 유형(type)은 레이블 셀렉터(label selector)를 뜻한다.


#### 컨트롤러 조정 루프
레플리케이션컨트롤러의 역할은 정확한 수의 파드가 항상 레이블 셀렉터와 일치하는지 확인하는 것이다. 그렇지 않은 경우 레플리케이션컨트롤러는 의도하는 파드 수와 실제 파드 수를 일치시키기 위한 적절한 조치를 취한다.

![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/04fig02_alt.jpg)


#### 레플리케이션컨트롤러의 세 가지 요소
 - 레이블 셀릭터(label selector): 레플리케이션컨트롤러의 범위에 있는 파드를 결정한다.
 - 레플리카 수(replica count): 원하는 파드의 수를 지정한다.
 - 파드 템플릿(pod template): 새로운 파드 레플리카를 만들 때 사용한다.

 ![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/04fig03.jpg)


 이 때, 레플리카 수는 변경을 하면 기존 파드에 영향을 미친다(레플리카 수가 부족하면 파드를 더 만들고, 초과하면 파드를 줄인다). 그러나, 레이블 셀렉터와 파드 템플릿은 변경해도 기존 파드에 영향을 미치지 않는다. 레이블 셀렉터를 중간에 변경하면 기존에 존재하던 파드들은 레플리케이션컨트롤러의 범위를 벗어나므로 관리 대상에서 제외된다. 그리고는 변경한 레이블 셀렉터에 대한 파드들이 생성된다. 파드 템플릿은 레플리케이션컨트롤러가 새로운 파드를 생성할 때만 영향을 미친다(새로 생성할 파드의 스펙 느낌).


#### 레플리케이션컨트롤러의 이점
- 기존 파드가 사라지면 새 파드를 시작해 파드(또는 여러 파드의 복제본)가 항상 실행되도록 한다.
- 클러스터 노드에 장애가 발생하면 장애가 발생한 노드에서 실행 중인 모든 파드(레플리케이션컨트롤러의 제어하에 있는 파드)에 관한 교체 복제본이 생성된다.
- 수동 또는 자동으로 파드를 쉽게 수평으로 확장할 수 있게 한다.


### 레플리케이션컨트롤러 생성
#### kubia-rc.yaml
``` yaml
apiVersion: v1
kind: ReplicationController         # 레플리케이션컨트롤러(RC)의 매니페스트 정의
metadata:
  name: kubia                       # 레플리케이션컨트롤러 이름
spec:
  replicas: 3                       # 복제 파드 수
  selector:                         # 파드 셀렉터 RC의 관리 파드 선택
    app: kubia
  template:                         # 새로 생성되는 파드의 템플릿
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: mycool0905/kubia
        ports:
        - containerPort: 8080
```

- ```$ kubectl create -f kubia-rc.yaml```
  + 해당 명령어로 ReplicationController 생성
  + 아래의 사진과 같이 파드들이 생성된다.
![image](https://user-images.githubusercontent.com/43199318/113114477-58327580-9246-11eb-801c-374b84754a66.png)

- ```$ kubectl delete pod kubia-2f8ph```
  + 해당 명령어로 ReplicationController가 관리하는 파드 하나 삭제하기
  + 아래의 사진과 같이 원래 있던 파드 하나가 사라지고 새로운 파드가 생성된다.
![image](https://user-images.githubusercontent.com/43199318/113114959-dee75280-9246-11eb-989c-d83a5947029c.png)


- ```$ kubectl get rc```
  + 레플리케이션 컨트롤러의 정보 알아보기
  + 아래의 그림과 같이 의도하는 파드 수, 실제 파드 수, 준비된 파드 수를 표시한다.
  ![image](https://user-images.githubusercontent.com/43199318/113115162-19e98600-9247-11eb-816b-3e5b7639e8f0.png)


- ```$ kubectl describe rc kubia```
  + 레플리케이션컨트롤러의 상세한 정보 살펴보기

#### 컨트롤러가 새로운 파드를 생성한 원인

![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/04fig04_alt.jpg)

삭제하는 행동 그 자체에 대한 대응이 아니라 결과적인 상태(부족한 파드 수)에 대응하는 것이다. 레플리케이션컨트롤러는 삭제되는 파드에 대해 즉시 통지를 받지만(API 서버는 클라이언트가 리소스 및 리소스 목록의 변경을 감시할 수 있도록 허용한다), 이 통지 자체가 대체 파드를 생성하게 하는 것은 아니다. 이 통지는 컨트롤러가 실제 파드 수를 확인하고, 적절한 조치를 취하도록 하는 트리거 역할을 한다.

멀티 노드에서 실행했을 때, 어느 노드가 장애가 발생하면 해당 노드에 있는 파드들의 상태는 `NotReady` > `Unknown` 상태가 되며, RC는 이 때 새로운 파드를 생성한다. 그리고 장애가 발생했던 노드가 다시 살아나면 `Unknown` 상태의 파드들을 삭제한다.


### 레플리케이션컨트롤러에서 레이블

위에서 나왔지만, 레플리케이션컨트롤러는 레이블 셀렉터와 일치하는 파드만을 관리한다. 파드의 레이블만 조작하면 RC의 관리 대상에서 제외시키고 또 다른 RC의 관리 대상으로 보낼 수도 있다.

- ```$ kubectl label pod kubia-47vjv type=special```
  + 해당 파드에 `type=special` 레이블 추가
- ```$ kubectl get po --show-labels```
  + 파드들의 레이블 보기
  ![image](https://user-images.githubusercontent.com/43199318/113117720-b3b23280-9249-11eb-87f7-588d4a8e02e4.png)

- ```$ kubectl label pod kubia-47vjv app=foo --overwrite```
  + 해당 파드의 레이블을 `app=foo`로 변경하기. --overwrite 인수가 있어야 기존의 레이블 값을 변경할 수 있다.

- ```$ kubectl get po -L app```
  + app 레이블을 포함하여 파드들 표시
  ![image](https://user-images.githubusercontent.com/43199318/113118060-0c81cb00-924a-11eb-80f6-c6e48a79fe7e.png)

여기서 보면 기존의 `kubia-47vjv`는 이제 관리 대상이 아니어서 제외되고 새로운 파드가 생성되는 중이다. 아래의 그림과 같다.

![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/04fig05_alt.jpg)

- ```$ kubectl edit rc kubia```
  + RC kubia의 매니페스트를 수정하여 재정의 할 수 있다.
  + 레이블 셀렉터를 변경하면 새로 변경된 레이블 셀렉터를 바라보는 파드들이 생성된다.
  + 파드 템플릿을 변경하면 새로 생성되는 파드들에만 영향을 준다(아래 그림 참고).

![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/04fig06_alt.jpg)

- ```$ kubectl scale rc kubia --replicas=10```
  + 레플리케이션 리소스의 `replicas` 필드 값을 변경하며 수평적으로 파드 스케일링을 진행한다.
  + 당연히 현존하는 값보다 적어지면 줄이고, 커지면 늘린다.
  + ```$ kubectl edit rc kubia``` 이 명령어로도 변경 가능하다.

- ```$ kubectel delete rc kubia```
  + RC를 삭제하고 해당 RC가 관리하던 파드들도 삭제한다.
  + `--cascade=false` 옵션을 추가하면 파드들은 삭제되지 않는다(아래 그림 참고).

![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/04fig07_alt.jpg)