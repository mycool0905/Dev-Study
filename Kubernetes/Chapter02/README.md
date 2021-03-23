# Chapter02


## 도커 정리


### 도커 실행시 일어나는 일

```$ docker run busybox echo "Hello world" ```

위 명령어를 실행했을 때, 로컬에 `busybox:latest`가 있는지 체크
존재하지 않는다면 `http://dockerhub.io`의 도커 허브 레지스트리에서 이미지를 다운로드한다.
이미지 다운로드가 완료되면 도커는 이미지로부터 컨테이너를 생성하고 컨테이너 내부에서 명령어를 실행한다.

![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/02fig01_alt.jpg)


### 이미지 실행

태그로 버전까지 설정 가능하다. (최신버전: latest)

```$ docker run <image>[:<tag>]```


 ### node.js 예제

app.js

 ```javascript
const http = require('http');
const os = require('os');

console.log("Kubia server starting...");

var handler = function(request, response) {
    console.log("Received request from " + request.connection.remoteAddress);
    response.writeHead(200);
    response.end("You've hit " + os.hostname() + "\n");
};

var www = http.createServer(handler);
www.listen(8080);
 ```


 ### 이미지를 위한 Dockerfile 생성

 ```Dockerfile
FROM node:7
ADD app.js /app.js
ENTRYPOINT ["node", "app.js"]
 ```

 `FROM`: 시작점(이미지 생성의 기반이 되는 기본 이미지)으로 사용할 컨테이너 이미지를 정의한다.
 `ADD`: 로컬 디렉터리의 app.js 파일을 이미지의 루트 디렉터리에 동일한 이름(app.js)으로 추가한다.
 `ENTRYPOINT`: 이미지를 실행했을 때 수행돼야 할 명령어를 정의한다. ```$ node app.js```


 ### 컨테이너 이미지 생성

```$ docker build -t kubia .```
도커에게 현재 디렉터리의 콘텐츠를 기반으로 kubia라고 부르는 이미지를 빌드하라고 요청했다.
도커는 디렉터리 내 Dockerfile을 살펴보고 파일에 명시된 지시 사항에 근거해 이미지를 빌드한다.

![image](https://drek4537l1klr.cloudfront.net/luksa/Figures/02fig02_alt.jpg)

도커 클라이언트는 빌드된 이미지를 실행하고, 도커 데몬은 빌드 프로세스를 수행한다.
도커 클라이언트와 도커 데몬은 같은 머신에 있을 필요는 없다.
빌드 프로세스 동안 이미지가 사용자 컴퓨터에 저장돼 있지 않다면, 도커는 기본 이미지(node:7)를 퍼블릭 이미지 레포지토리(도커 허브)에서 가져온다.


### 컨테이너 이미지 실행

```$ docker run --name kubia-container -p 8080:8080 -d kubia```
도커가 `kubia` 이미지에서 `kubia-container`라는 이름의 새로운 컨테이너를 실행하도록 한다.
컨테이너는 콘솔에서 분리돼(`-d` 플래그) 백그라운드에서 실행됨을 의미한다. 로컬 머신의 8080포트가 컨테이너 내부의 8080포트와 매핑되므로(`-p 8080:8080` 옵션) http://localhost:8080으로 애플리케이션에 접근할 수 있다.

맥이나 윈도우를 사용하는 경우에는 로컬 머신에서 도커 데몬이 실행 중이 아니다. 그래서 localhost 대신에 데몬이 실행 중인 가상머신의 호스트 이름이나 IP를 사용해야 한다. 이 정보는 `DOCKER_HOST` 환경변수로 확인 가능하다.


### 도커 명령어들

- ```$ docker ps```
  + 컨테이너의 기본 정보 표시
  
- ```$ docker inspect <container name>```
  + 컨테이너의 상세 정보를 JSON 형식으로 출력

- ```$ docker exec -it kubia-container bash```
  + 실행중인 컨테이너 내부에서 셸 실행하기
    * -i: 표준 입력(STDIN)을 오픈 상태로 유지한다. 셸에 명령어를 입력하기 위해 필요하다.
    * -t: 의사(pseudo) 터미널(TTY)을 할당한다.
  + 일반적인 셸을 사용하는 것과 동일하게 셸을 사용하고 싶다면 두 옵션이 필요하다.

- ```$ docker stop <container name>```
  + 실행중인 컨테이너 중지

- ```$ docker rm <container name>```
  + 컨테이너 삭제

- ```$ docker tag kubia mycool0905/kubia```
  + 추가 태그로 이미지 태그 지정, 도커 허브 ID로 지정


