# Chapter03


## 파드 정리


### 파드에 관해서

`파드(Pod)`는 함께 배치된 컨테이너 그룹이며 쿠버네티스의 기본 빌딩 블록이다. 컨테이너를 개별적으로 배포하기보다는 컨테이너를 가진 파드를 배포하고 운영한다. 일반적으로 파드는 하나의 컨테이너만 포함한다. 파드의 핵심 사항은 파드가 여러 컨테이너를 가지고 있을 경우에, 모든 컨테이너는 항상 하나의 워커 노드에서 실행되며 여러 워커 노드를 걸쳐 실행되지 않는다.

![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/03fig01_alt.jpg)


### 파드가 왜 필요한가?

- 단일 컨테이너 내에서 여러 프로세스를 실행하는 것은 가능하나, 컨테이너는 단일 프로세스를 실행하는 것을 목적으로 설계했다(프로세스가 자기 자신의 자식 프로세스를 생성하는 것 제외).
  + 여러 프로세스는 동일한 표준 출력으로 로그를 기록하기 때문에 힘든 로그 관리
  + 개별 프로세스가 실패하는 경우 자동으로 재시작하는 메커니즘
- 여러 프로세스를 단일 컨테이너로 묶지 않기 때문에, 컨테이너를 함께 묶고 하나의 단위로 관리할 수 있는 또 다른 상위 구조인 `파드(Pod)`가 필요하다.
- 컨테이너 모음을 사용해 밀접하게 연관된 프로세스를 함께 실행하고 단일 컨테이너 안에서 모두 함께 실행되는 것처럼 (거의) 동일한 환경을 제공할 수 있으면서도 이들을 격리된 상태로 유지할 수 있다.


### 파드의 적절한 구성

`파드(Pod)`는 스케일링의 기본 단위이다.
컨테이너를 파드로 묶어 그룹으로 만들 때에는, 즉 두 개 이상의 컨테이너를 단일 파드로 넣을지 아니면 두 개의 별도의 파드에 넣을지를 결정하기 위해서는 다음과 같은 질문을 할 필요가 있다.

1. 컨테이너를 함께 실행해야 하는가? 혹은 서로 다른 호스트에서 실행할 수 있는가?
2. 여러 컨테이너가 모여 하나의 구성 요소를 나타내는가? 혹은 개별적인 구성 요소인가?
3. 컨테이너가 함께, 혹은 개별적으로 스케일링돼야 하는가?

여러 설명이 있지만 그림 하나로 이해 된다. 

![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/03fig04_alt.jpg)

컨테이너는 여러 프로세스를 실행하지 말아야 하고, 파드는 동일한 머신에서 실행할 필요가 없다면 여러 컨테이너를 포함하지 말아야 한다.


## 파드 실습


### YAML 디스크립터로 리소스 생성

파드를 포함한 다른 쿠버네티스 리소스는 일반적으로 쿠버네티스 REST API 엔드포인트에 `JSON` 혹은 `YAML` 매니페스트를 전송해 생성한다. `YAML` 파일에 모든 쿠버네티스 오브젝트를 정의하면 버전 관리 시스템에 넣는 것이 가능해져, 그에 따른 이점을 누릴 수 있다.

참고: https://kubernetes.io/docs/reference/

- ```$ kubectl get po <pod name> -o yaml```
  + 파드의 전체 YAML 정의 보기

여기서 YAML을 보다보면 거의 모든 쿠버네티스 리소스가 갖고 있는 세 가지 주요 부분이 있다.
- `Metadata`: 이름, 네임스페이스, 레이블 및 파드에 관한 기타 정보를 포함한다.
- `Spec`: 파드 컨테이너, 볼륨, 기타 데이터 등 파드 자체에 관한 실제 명세를 가진다.
- `Status`: 파드 상태, 각 컨테이너 설명과 상태, 파드 내부 IP, 기타 기본 정보 등 현재 실행 중인 파드에 관한 현재 정보를 포함한다. (새로운 리소스를 생성할 때 status 부분은 작성 안한다.)

#### kubia-manual.yaml
```yaml
apiVersion: v1                  # 디스크립터는 쿠버네티스 API 버전 v1을 준수한다.
kind: Pod                       # 오브젝트 종류는 파드
metadata:                       
  name: kubia-manual            # 파드 이름
spec:
  containers:
  - image: mycool0905/kubia     # 컨테이너를 만드는 컨테이너 이미지
    name: kubia                 # 컨테이너 이름
    ports:
    - containerPort: 8080       # 애플리케이션이 수신하는 포트
      protocol: TCP
```

여기서 포트 정의 안에서 포트를 지정해둔 것은 단지 정보에 불과하다. 이를 생략해도 다른 클라이언트에서 포트를 통해 파드에 연결할 수 있는 여부에 영향을 미치지 않는다. 컨테이너가 0.0.0.0 주소에 열어 둔 포트를 통해 접속을 허용할 경우 파드 스펙에 포트를 명시적으로 나열하지 않아도 다른 파드에서 항상 해당 파드에 접속할 수 있다. 그러나 포트를 명시적으로 정의한다면, 클러스터를 사용하는 모든 사람이 파드에서 노출한 포트를 빠르게 볼 수 있다. 또한 포트를 명시적으로 정의하면 포트에 이름일 지정해 편리하게 사용할 수 있다.


### kubectl explain
- ```$ kubectl explain <object>```
  + 해당 오브젝트에 관한 설명과 오브젝트가 가질 수 있는 속성을 출력한다.
- ```$ kubectl explain <object>.spec``` 
  + 해당 오브젝트의 spec 속성을 살펴볼 수 있다.



### kubectl create
- ```$ kubectl create -f kubia-manual.yaml```
  + `kubia-manual.yaml`로 해당 리소스 생성

 ![image](https://user-images.githubusercontent.com/43199318/112274624-0c218700-8cc2-11eb-9bc0-bf6e7f65a731.png)
- ```$ kubectl get po <pod name> -o yaml```
  + 위에서 적었듯이 해당 파드의 전체 정의를 볼 수 있다.
- ```$ kubectl logs <pod name>```
  + 해당 파드의 로그 확인

![image](https://user-images.githubusercontent.com/43199318/112274843-4b4fd800-8cc2-11eb-8176-47b4e374ef7d.png)

- ```$ kubectl port-forward kubia-manual 8888:8080```
  + 포트 포워딩. 로컬 포트 8888을 kubia-manual 파드의 8080포트로 향하게 한다.

![image](https://user-images.githubusercontent.com/43199318/112275247-c0231200-8cc2-11eb-89cc-778f3c3917e9.png)

다른 터미널로 ```$ curl localhost:8888``` 요청을 하면 `localhost:8888`에서 실행되고 있는 `kubectl port-forward` 프록시를 통해 HTTP 요청을 해당 파드에 보낼 수 있다.

![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/03fig05_alt.jpg)


### 레이블을 이용한 파드 구성

`레이블`: 파드를 분류하기 위한 수단, 이를 통해 파드와 기타 다른 쿠버네티스 오브젝트의 조직화가 이뤄진다.
레이블 키가 해당 리소스 내에서 고유하다면, 하나 이상 원하는 만큼 레이블을 가질 수 있다. 일반적으로 리소스를 생성할 때 레이블을 붙이지만, 나중에 레이블을 추가하거나 기존 레이블 값을 수정할 수도 있다.

#### 레이블 적용 전
![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/03fig06_alt.jpg)

#### 레이블 적용 후
![iamge](https://drek4537l1klr.cloudfront.net/luksa/Figures/03fig07_alt.jpg)

레이블 예시
- `app`: 파드가 속한 애플리케이션, 구성 요소 혹은 마이크로서비스 지정한다.
- `rel`: 파드에서 실행 중인 애플리케이션이 안정, 베타 혹은 카나리 릴리스인지 보여준다.


#### kubia-manual-with-labels.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-manual-v2
  labels:
    creation_method: manual         # 레이블 두 개를 파드에 붙였다.
    env: prod
spec:
  containers:
  - image: mycool0905/kubia
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
```


- ```$ kubectl get po --show-labels```
  + `--show-labels`를 추가함으로써 레이블을 볼 수 있다.

![image](https://user-images.githubusercontent.com/43199318/112277014-ab477e00-8cc4-11eb-972d-356ce48c3333.png)


- ```$ kubectl label po <pod name> <label name>=<label value>```
  + 해당 파드에 레이블을 추가한다.
- ```$ kubectl label po <pod name> <label name>=<label value> --overwrite```
  + 기존 레이블을 변경할 때는 `--overwrite` 옵션이 필요하다
- ```$ kubectl get po -L <label name>[,<label name>,...]```
  + 모든 파드들의 해당 레이블 명을 포함해서 표시하도록 한다.


### 레이블 셀렉터를 이용한 파드 부분 집합

`레이블`이 중요한 이유는 `레이블 셀렉터`와 함께 사용된다는 점이다. `레이블 셀렉터`는 특정 레이블로 태그된 파드의 부분 집합을 선택해 원하는 작업을 수행한다. `레이블 셀렉터`는 특정 값과 레이블을 갖는지 여부에 따라 리소스를 필터링하는 기준이 된다.
- 특정한 키를 포함하거나 포함하지 않는 레이블
- 특정한 키와 값을 가진 레이블
- 특정한 키를 갖고 있지만, 다른 값을 가진 레이블

#### 레이블 셀렉터를 사용해 파드 나열
- ```$ kubectl get po -l <label name>=<label value>```
  + 해당 레이블의 키와 값이 일치하는 파드들 표시
- ```$ kubectl get po -l <label name>```
  + 해당 레이블을 가지고 있지만, 값이 무엇이든 상관없는 파드들 표시
- ```$ kubectl get po -l '!<label name>'```
  + 해당 레이블을 가지고 있지 않은 파드들 표시
- ```$ kubectl get po -l <label name>!=<label value>```
  + 해당 레이블을 가지고 있는 파드 중에서 해당하는 값이 아닌 파드들 표시
- ```$ kubectl get po -l <label name> in (a,b,c)```
  + 해당 레이블 값이 a 또는 b 또는 c로 설정돼 있는 파드들 표시
- ```$ kubectl get po -l <label name> notin (a,b,c)```
  + 해당 레이블 값이 a,b,c가 아닌 파드들 표시

당연히 모두 `,`를 붙여서 여러 조건을 붙일 수 있다.


### 레이블과 셀렉터를 이용해 파드 스케줄링 제한

특수한 자원이 필요한 파드는 해당 자원을 가지고 있는 노드에 스케줄링 하고 싶다. 이럴 경우, 해당 노드에도 레이블을 붙여서 스케줄링을 제한할 수 있다.

#### 예: gpu를 가지고 있는 노드에 gpu=true를 붙인 뒤, 해당 노드에 파드 스케줄링하기
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  nodeSelector:                 # nodeSelector는 쿠버네티스에
    gpu: "true"                 # gpu=true 레이블을 포함한 노드에
  containers:                   # 이 파드를 배포하도록 지시한다.
  - image: mycool0905/kubia
    name: kubia
```


### 파드 내부의 어노테이션


`어노테이션`은 키-값 쌍으로 레이블과 거의 비슷하지만 식별 정보를 갖지 않는다. 그냥 많은 정보를 보유할 수 있다. 그냥 주석같은 느낌으로, 오브젝트에 관한 정보를 적어두는 것이다.

- ```$ kubectl annotate po <pod name> <annotation>="contents"```
  + 해당 파드에 어노테이션 추가

`kubectl describe` 명령으로 확인할 수 있다.


### 네임스페이스


`네임스페이스`는 리눅스 네임스페이스가 아니라 `쿠버네티스 네임스페이스`를 뜻한다. 그냥 IDE에서 프로젝트 같은 느낌으로 보면 될 것 같다.

- ```$ kubectl get ns```
  + 클러스터에 존재하는 모든 네임스페이스 확인
- ```$ kubectl get po --namespace(-n) <namespace>```
  + 해당 네임스페이스에 존재하는 파드들 확인

#### 네임스페이스 정의 YAML, custom-namespace.yaml
``` yaml
apiVersion: v1
kind: Namespace
metadata:
  name: custom-namespace
```
- ```$ kubectl create -f custom-namespace.yaml```
  + YAML로 네임스페이스 생성
- ```$ kubectl create namespace <namespace>```
  + 간단하게 네임스페이스 생성
- ```$ kubectl create -f kubia-manual.yaml -n custom-namespace```
  + `kubia-manual` 파드를 `custom-namespace` 에 생성

`kubectl`은 현재 `kubectl context`에 구성돼 있는 기본 네임스페이스에서 작업을 수행한다. 현재 컨텍스트의 네임스페이스와 현재 컨텍스트 자체는 `kubectl config` 명령으로 변경할 수 있다.

서로 다른 네임스페이스는 오브젝트를 별도 그릅으로 분리해 특정한 네임스페이스 안에 속한 리소스를 대상으로 작업할 수 있게 해주지만, 실행 중인 오브젝트에 대한 격리는 제공하지 않는다.

#### 별칭 설정
`alias kcd='kubectl config set-context $(kubectl config current-context) --namespace '`
- ```$ kcd other-namespace```
  + `other-namespace`로 네임스페이스 컨텍스트 전환


## 리소스 삭제
- ```$ kubectl delete po <pod name>```
  + 해당 파드 삭제
- ```$ kubectl delete po -l <label name>=<label value>```
  + 레이블 셀렉터를 이용해 삭제
- ```$ kubectl delete ns <namespace>```
  + 해당 네임스페이스 삭제(내부의 파드들도 함께 자동으로 삭제)
- ```$ kubectl delete po --all```
  + 현재 네임스페이스에 있는 모든 파드들 삭제
- ```$ kubectl delete all --all```
  + 현재 네임스페이스에 있는 모든 리소스 삭제



## 마무리
아직 기초라서 쉬운 것 같다. 일단 기본기를 제대로 익히자.