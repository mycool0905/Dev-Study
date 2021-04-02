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
  + ![image](https://user-images.githubusercontent.com/43199318/113117720-b3b23280-9249-11eb-87f7-588d4a8e02e4.png)

- ```$ kubectl label pod kubia-47vjv app=foo --overwrite```
  + 해당 파드의 레이블을 `app=foo`로 변경하기. --overwrite 인수가 있어야 기존의 레이블 값을 변경할 수 있다.

- ```$ kubectl get po -L app```
  + app 레이블을 포함하여 파드들 표시
  + ![image](https://user-images.githubusercontent.com/43199318/113118060-0c81cb00-924a-11eb-80f6-c6e48a79fe7e.png)

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


### 레플리카셋

초기에는 레플리케이션컨트롤러가 파드를 복제하고 노드 장애가 발생했을 때 재스케줄링하는 유일한 쿠버네티스 구성 요소였다. 후에 `레플리카셋(ReplicaSet)`이라는 유사한 리소스가 도입됐다. 상위 수준의 리소스인 `디플로이먼트(Deployment)`를 생성할 때 자동으로 생성되게 한다.


#### 레플리카셋과 레플리케이션컨트롤러 비교

- 레이블
  + 레플리케이션컨트롤러: 레플리케이션컨트롤러의 레이블 셀렉터는 특정 레이블이 있는 파드만을 매칭시킬 수 있다. 
  + 레플리카셋: 레플리카셋의 셀렉터는 특정 레이블이 없는 파드나 레이블의 값과 상관없이 특정 레이블의 키를 갖는 파드를 매칭시킬 수 있다. 
  + 예시) 레플리케이션컨트롤러는 각 레이블이 `env=production`인 파드와 `env=devel`인 파드를 동시에 매칭시킬 수 없다. 그러나 레플리카셋은 하나의 레플리카셋으로 두 파드 세트를 모두 매칭시켜 하나의 그룹으로 취급할 수 있다.
  + 예시) 레플리케이션컨트롤러는 값에 상관없이 레이블 키의 존재만으로 파드를 매칭시킬 수 없지만, 레플리카셋은 가능하다. 레플리카셋은 실제 값이 무엇이든 `env` 키가 있는 레이블을 갖는 모든 파드를 매칭시킬 수 있다(`env=*`로 생각할 수 있다).

### 레플리카셋 생성
#### kubia-replicaset.yaml
``` yaml
apiVersion: apps/v1beta2          # 레플리카셋은 API 그룹 apps, 버전 v1beta2에 속한다.
kind: ReplicaSet
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:                  # 레플리케이션컨트롤러와 유사한 간단한 matchLabels 셀렉터를 사용한다.
      app: kubia
  template:                       # 템플릿은 레플리케이션컨트롤러와 동일하다.
    metadata:
      labels:
        app: kubia
    spec:
      containers:
      - name: kubia
        image: mycool0905/kubia
```

- ```$ kubectl create -f kubia-replicaset.yaml```
  + 레플리카셋 생성
- ```$ kubectl get rs```
- ```$ kubectl describe rs kubia```
  + 생성된 레플리카셋 검사(레플리케이션컨트롤러와 유사하게 출력된다.)
  + ![image](https://user-images.githubusercontent.com/43199318/113383409-d44fc900-93be-11eb-9c37-fc4ecb2b06a8.png)
- ```$ kubectl delete rs kubia```
  + 레플리카셋 삭제

### 레플리카셋의 표현적인 레이블 셀렉터 사용
#### kubia-resplicaset-matchexpressions.yaml
``` yaml
selector:
  matchExpressions:
    - key: app                      # 이 셀렉터는 파드의 키가 "app"인 레이블을 포함해야 한다.
      operator: In
      values:                       # 레이블의 값은 "kubia"여야 한다.
      - kubia
```

- `In`: 레이블의 값이 지정된 값 중 하나와 일치해야 한다.
- `NotIn`: 레이블의 값이 지정된 값과 일치하지 않아야 한다.
- `Exists`: 파드는 지정된 키를 가진 레이블이 포함돼야 한다(값은 중요하지 않음). 이 연산자를 사용할 때는 `value` 필드를 지정하지 않아야 한다.
- `DoesNotExist`: 파드에 지정된 키를 가진 레이블이 포함돼 있지 않아야 한다. 값 필드를 지정하지 않아야 한다.

여러 표현식을 지정하는 경우 셀렉터가 파드와 매칭되기 위해서는, 모든 표현식이 `true`여야 한다. `matchLabels`와 `matchExpressions`를 모두 지정하면, 셀렉터가 파드를 매칭하기 위해서는, 모든 레이블이 일치하고, 모든 표현식이 `true`로 평가돼야 한다.


### 데몬셋

레플리케이션컨트롤러와 레플리카셋은 쿠버네티스 클러스터 내 어딘가에 지정된 수만큼의 파드를 실행하는 데 사용된다. 그러나 클러스터의 모든 노드에, 노드당 하나의 파드만 실행되길 원하는 경우(아래의 그림과 같이)가 있을 수 있다. 대개 시스템 수준의 작업을 수행하는 인프라 관련 파드가 이런 경우다(모든 노드에서 로그 수집기와 리소스 모니터를 실행하려는 경우, 쿠버네티스의 `kube-proxy` 프로세스 이는 서비스를 작동시키기 위해 모든 노드에서 실행돼야 한다).

![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/04fig08_alt.jpg)

쿠버네티스를 사용하지 않는 환경에서는 일반적으로 노드가 부팅되는 동안에 `시스템 초기화 스크립트(init script)` 또는 `systemd 데몬`을 통해 시작된다. 쿠버네티스 노드에서도 이처럼 시스템 프로세스를 실행할 수도 있지만, 그렇게 하면 쿠버네티스가 제공하는 모든 기능을 최대한 활용할 수가 없다.

#### 데몬셋이란

모든 클러스터 노드마다 파드를 하나만 실행하려면 `데몬셋(DaemonSet)` 오브젝트를 생성해야 한다. 데몬셋에 의해 생성되는 파드는 타깃 노드가 이미 지정돼 있고 쿠버네티스 스케줄러를 건너뛰는 것을 제외하면 이 오브젝트는 레플리케이션컨트롤러 또는 레플리카셋과 매우 유사하다. 대신 클러스터 내에 무작위로 흩어져 배포되는 것이 아니라, 위의 그림처럼 노드 수만큼 파드를 만들고 각 노드에 배포된다.

또한, `복제본 수(replicas)` 라는 개념이 없고, 노드의 수에 따라 생성된다. 새 노드가 클러스터에 추가되면 데몬셋은 즉시 새 파드 인스턴스를 새 노드에 배포한다.

파드가 노드의 일부에서만 실행되도록 지정하지 않으면 데몬셋은 클러스터의 모든 노드에 파드를 배포한다. 그래서 데몬셋을 사용해 특정 노드에서만 파드를 실행하게 하려면 데몬셋 정의의 일부인 파드 템플릿에서 `node-Selector` 속성을 지정하면 된다.

#### ssd-monitor-daemonset.yaml
``` yaml
apiVersion: apps/v1beta2                # 데몬셋은 API 그룹 apps의 v1beta2 버전에 있다.
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:                      # 파드 템플릿은 disk=ssd 레이블이 있는 노드를 선택하는 노드 셀렉터를 갖는다.
        disk: ssd
      containers:
      - name: main
        image: luksa/ssd-monitor
```


- ```$ kubectl create -f ssd-monitor-daemonset.yaml```
  + 데몬셋 생성
- ```$ kubectl get ds```
  + 데몬셋 검색
  + ![image](https://user-images.githubusercontent.com/43199318/113385127-745b2180-93c2-11eb-9ebe-cf5aff3b3762.png)

위에서 0이 나온 이유는 아직 노드에 레이블을 달지 않았기 때문이다. (노드에 레이블 추가하는 실습은 하지 않겠다.)
- ```$ kubectl delete ds ssd-monitor```
  + 데몬셋 삭제


### 잡(Job) 리소스

지금까지는 지속적으로 실행돼야 하는 파드에 관해서만 이야기했다. 그러나 완료 가능한 태스크에 대해서는 프로스세가 종료된 후에 다시 시작되지 않는다. 이를 쿠버네티스 `잡(Job)` 리소스라고 하는데, 다른 리소스와 유사 하지만 `잡(Job)`은 파드의 컨테이너 내부에서 실행 중인 프로세스가 성공적으로 완료되면 컨테이너를 다시 시작하지 않는 파드를 실행할 수 있다. 일단 그렇게 되면 파드는 완료된 것으로 간주된다. 노드에 장애가 발생한 경우 해당 노드에 있던 잡이 관리하는 파드는 레플리카셋 파드와 같은 방식으로 다른 노드로 다시 스케줄링된다. 프로세스 자체에 장애가 발생한 경우, `잡(Job)`에서 컨테이너를 다시 시작할 것인지 설정할 수 있다. 아래의 그림은 초기에 스케줄링된 노드에 장애가 발생했을 때 `잡(Job)`에 의해 생성된 파드를 새 노드로 다시 스케줄링하는 방법을 보여준다. 또한 관리되지 않는 파드(다시 스케줄링되지 않음)와 레플리카셋이 관리하는 파드를 보여준다.

![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/04fig10_alt.jpg)


`잡(Job)`은 작업이 제대로 완료되는 것이 중요한 임시 작업에 유용하다. 관리되지 않은 파드에서 작업을 실행하고 완료될 때까지 기다릴 수 있지만 작업이 수행되는 동안 노드에 장애가 발생하거나 파드가 노드에서 제거되는 경우 수동으로 다시 생성해야 한다. 특히 잡을 완료하는 데 몇 시간이 걸리는 경우 이 작업을 수동으로 수행한다는 것은 말이 안되는 일이다. 예를 들면 배치 작업이 있다.


### 잡(Job) 리소스 생성
#### exporter.yaml
``` yaml
apiVersion: batch/v1                    # 잡은 batch API 그룹, v1버전에 속한다.
kind: Job
metadata:
  name: batch-job
spec:                                   # 파드 셀렉터를 지정하지 않았다.
  template:                             # (파드 템플릿의 레이블을 기반으로 만들어진다.)
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure          # 잡은 기본 재시작 정책(Always)을 사용할 수 없다.
      containers:
      - name: main
        image: luksa/batch-job
```

- ```$ kubectl create -f exporter.yaml```
  + 잡 리소스 생성
- ```$ kubectl get jobs```
  + 잡 리소스 검색
  + ![image](https://user-images.githubusercontent.com/43199318/113386313-f3e9f000-93c4-11eb-9a0c-e705ee4ad9f9.png)
- ```$ kubectl get po```
  + 생성된 잡 리소스 확인
  + ![image](https://user-images.githubusercontent.com/43199318/113386493-55aa5a00-93c5-11eb-9f67-a2327d841b8c.png)
- ```$ kubectl get po```
  + 2분후 완료된 잡 리소스 확인
  + ![image](https://user-images.githubusercontent.com/43199318/113386669-b174e300-93c5-11eb-895d-708aeba0b795.png)
- ```$ kubectl logs batch-job-mtwtt```
  + 해당 잡 리소스의 파드 로그 확인
  + ![image](https://user-images.githubusercontent.com/43199318/113386770-e719cc00-93c5-11eb-9a7b-4fe7a10fc84d.png)
- ```$ kubectl get job```
  + 잡 리소스 검색
  + ![image](https://user-images.githubusercontent.com/43199318/113386813-fef15000-93c5-11eb-9516-55933562c5e9.png)


이 때, 잡에서 여러 파드 인스턴스를 생성해 병렬 또는 순차적으로 실행할 수 있다. 잡 스펙에 `completions`와 `parallelism` 속성을 설정해 수행한다.

#### multi-completion-parallel-batch-job.yaml
``` yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: multi-completion-batch-job
spec:
  completions: 5                        # 이 잡은 다섯 개의 파드를 성공적으로 완료해야 한다.
  parallelism: 2                        # 두 개까지 병렬로 실행할 수 있다.
  template:
    metadata:
      labels:
        app: batch-job
    spec:
      restartPolicy: OnFailure
      containers:
      - name: main
        image: luksa/batch-job
```

- ```$ kubectl create -f multi-completion-batch-job.yaml```
  + 잡 리소스 생성
- ```$ kubectl get po```
  + 파드 검색
  + `parallelism`을 2로 설정해서 2개가 동시에 생성되는 것을 볼 수 있다.
  + ![image](https://user-images.githubusercontent.com/43199318/113387356-21379d80-93c7-11eb-8fd2-0511d0be7ad6.png)
- ```$ kubectl scale job multi-completion-batch-job --replicas 3```
  + 잡이 실행되는 동안 잡의 `parallelism` 속성을 3으로 변경
  + 하나의 파드가 추가되는 것을 볼 수 있다.
  + ![image](https://user-images.githubusercontent.com/43199318/113387477-59d77700-93c7-11eb-82f9-1aa323dd04ba.png)
- ```$ kubectl get jobs```
  + 잡 리소스 검색
  + 5개의 파드가 완료된 것을 확인할 수 있다.
  + ![image](https://user-images.githubusercontent.com/43199318/113387649-b2a70f80-93c7-11eb-8c86-0c27d4d4625a.png)

파드 스펙에 `activeDeadlineSeconds` 속성을 설정하면 파드의 실행 시간을 제한할 수 있다. 파드가 이보다 오래 실행되면 시스템이 종료를 시도하고 잡을 실패한 것으로 표시한다.

잡의 매니페스트에서 `spec.backoffLimit` 필드를 지정해 실패한 것으로 표시되기 전에 잡을 재시도할 수 있는 횟수를 설정할 수 있다. 명시적으로 지정하지 않으면 기본값은 6이다.


### 크론잡(CronJob)

잡 리소스를 생성하면 즉시 해당하는 파드를 실행한다. 그러나 많은 배치 잡이 미래의 특정 시간 또는 지정된 간격으로 반복 실행해야 한다. 이런 작업을 `크론(cron)` 작업이라고 하고 쿠버네티스에서는 이를 `크론잡(CronJob)` 리소스를 만들어서 구성한다.


### 크론잡 리소스 생성
#### cronjob.yaml
``` yaml
apiVersion: batch/v1beta2
kind: CronJob
metadata:
  name: batch-job-every-fifteen-minutes
spec:
  schedule: "0,15,30,45 * * * *"          # 이 잡은 매일, 매시간 0, 15, 30, 45분에 실행해야 한다.
  startingDeadlineSeconds: 15
  jobTemplate:
    spec:
      template:
        metadata:
          labels:
            app: periodic-batch-job
        spec:
          restartPolicy: OnFailure
          containers:
          - name: main
            image: luksa/batch-job
```

- `스케줄(schedule)` 설정하기(크론 스케줄 형식)
  + 5개를 순서대로 적으면 된다.
  + 분
  + 시
  + 일
  + 월
  + 요일 (0: 일, 1: 월, 2: 화, 3: 수, 4: 목, 5: 금, 6: 토)
  + 예) 매월 매일 매시 30분에 실행: 30 * * * *
  + 예) 일요일 오후 3시마다 실행: 0 15 * * 0

- `startingDeadlineSeconds` 설정하기
  + 잡이나 파드가 예정된 시간을 초과해서 시작돼서는 안 된다는 엄격한 요구사항 설정
  + 위에서 15로 설정했으므로, 해당 크론 스케줄 시각부터 15초 이내로 시작하지 않으면 실패로 표시



## 마무리
슬슬 양이 많아지네, 일단 10단원까지는 기본개념이니까 열심히 달려보자.