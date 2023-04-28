### Docker란?

- 애플리케이션을 개발, 출시, 실행하는 데 사용하는 개방형 플랫폼
- 코드를 더욱 빠르게 출시, 테스트, 배포하고 코드 작성과 실행 주기를 단축하는 데 도움
- Kubernetes에서 직접 사용할 수 있으므로 Kubernetes Engine에서 쉽게 실행

받은 GCP Username, Password, GCP Project ID를 이용해 Google Console 열기

**계정 이름 목록을 표시**

```jsx
gcloud auth list
```

**프로젝트 ID 목록을 표시**

```jsx
gcloud config list project
```

### 작업 1. Hello World

1. hello world 컨테이너를 실행

```jsx
docker run hello-world
```

Docker 데몬이 hello-world 이미지를 검색했으나 로컬에서 이미지를 찾지 못했고, Docker Hub라는 공개 레지스트리에서 이미지를 가져오고, 가져온 이미지에서 컨테이너를 생성하고, 컨테이너를 실행

2. Docker Hub에서 가져온 컨테이너 이미지를 확인

```jsx
docker images
```

3. 실행 중인 컨테이너 확인

```jsx
docker ps
```

실행 종료했기 때문에 hello-world는 결과창에 나오지 않는다.

4. 모든 컨테이너 확인

```jsx
docker ps -a
```

### 작업 2. 빌드

1. `test`라는 이름의 폴더를 만들고 이 폴더로 전환

```jsx
mkdir test && cd test
```

2.  `Dockerfile` 만들기

```jsx
cat > Dockerfile <<EOF
# 상위 이미지 지정
FROM node:lts
# 컨테이너의 현재 작업 디렉터리 설정
WORKDIR /app
# 현재 디렉터리의 콘텐츠("."로 표시)를 컨테이너에 추가
ADD . /app
# 컨테이너의 포트를 공개하여 해당 포트에서의 연결을 허용
EXPOSE 80
# Run app.js using node when the container launches
CMD ["node", "app.js"]
EOF
```

3. 노드 애플리케이션을 생성

```jsx
cat > app.js <<EOF
const http = require('http');
const hostname = '0.0.0.0';
const port = 80;
const server = http.createServer((req, res) => {
    res.statusCode = 200;
    res.setHeader('Content-Type', 'text/plain');
    res.end('Hello World\n');
});
server.listen(port, hostname, () => {
    console.log('Server running at http://%s:%s/', hostname, port);
});
process.on('SIGINT', function() {
    console.log('Caught interrupt signal and will exit');
    process.exit();
});
EOF
```

4. 이미지 빌드

```jsx
docker build -t node-app:0.1 .
```

- . 은 현재 디렉터리 의미
- `-t`는 `name:tag` 문법을 사용하여 이미지의 이름과 태그를 지정

5. 빌드한 이미지 확인

```jsx
docker images
```

### 작업 3. 실행

1. 빌드한 이미지를 기반으로 하는 컨테이너를 실행

```jsx
docker run -p 4000:80 --name my-app node-app:0.1
```

2. 다른 터미널을 열고(Cloud Shell에서 `+` 아이콘을 클릭) 서버를 테스트

```jsx
curl http://localhost:4000
```

3. 초기 터미널을 닫은 후 다음 명령어를 실행하여 컨테이너를 중지하고 삭제

```jsx
docker stop my-app && docker rm my-app
```

4. 백그라운드에서 컨테이너를 시작

```jsx
docker run -p 4000:80 --name my-app -d node-app:0.1
docker ps
```

-d : 백그라운드에서 시작, demon을 의미

5. `docker ps`의 출력된 결과에서 컨테이너가 실행 중임을 확인할 수 있다. `docker logs [container_id]`를 실행하면 로그를 볼 수 있다.

```jsx
docker logs [container_id]
```

**애플리케이션 수정**

1. 앞서 실습에서 만든 테스트 디렉터리를 열기

```jsx
cd test
```

2. 원하는 텍스트 편집기(예: nano 또는 vim)로 `app.js`를 편집하고 'Hello World'를 다른 문자열로 변경

```jsx
....
const server = http.createServer((req, res) => {
    res.statusCode = 200;
    res.setHeader('Content-Type', 'text/plain');
    res.end('Welcome to Cloud\n');
});
....
```

3. 새 이미지를 빌드하고 `0.2`로 태그를 지정

```jsx
docker build -t node-app:0.2 .
```

명령어 결과

```jsx
Step 1/5 : FROM node:lts
 ---> 67ed1f028e71
Step 2/5 : WORKDIR /app
 ---> Using cache
 ---> a39c2d73c807
Step 3/5 : ADD . /app
 ---> a7087887091f
Removing intermediate container 99bc0526ebb0
Step 4/5 : EXPOSE 80
 ---> Running in 7882a1e84596
 ---> 80f5220880d9
Removing intermediate container 7882a1e84596
Step 5/5 : CMD node app.js
 ---> Running in f2646b475210
 ---> 5c3edbac6421
Removing intermediate container f2646b475210
Successfully built 5c3edbac6421
Successfully tagged node-app:0.2
```

**2단계**에서 기존 캐시 레이어를 사용하고 있음을 확인 가능

**3단계** 이후부터는 `app.js`를 변경했기 때문에 레이어가 수정

4. 새 이미지 버전으로 다른 컨테이너를 실행한다. 이때 호스트 포트를 80 대신 8080으로 매핑하는 방법을 확인한다. (포트 4000은 이전 이미지가 사용중)

```jsx
docker run -p 8080:80 --name my-app-2 -d node-app:0.2
docker ps
```

5. 컨테이너 테스트

```jsx
curl http://localhost:8080
```

`Welcome to Cloud`

6. 처음 작성한 컨테이너 테스트

```jsx
curl http://localhost:4000
```

`Hello World`

### 작업 4. 디버그

1. 컨테이너의 로그 확인

```jsx
docker logs -f [container_id]
```

컨테이너가 실행중이면 -f 옵션 사용

2. 다른 터미널을 열고(Cloud Shell에서 + 아이콘을 클릭) 다음 명령어를 입력

```jsx
docker exec -it [container_id] bash
```

-it : pseudo-tty를 할당하고 stdin을 열린 상태로 유지하여 컨테이너와 상호작용

`root@xxxxxxxxxxxx:/app#`

3. Bash 세션을 종료

```jsx
exit
```

4. Docker에서 컨테이너의 메타데이터를 검토

```jsx
docker inspect [container_id]
```

5. `--format`
을 사용하여 반환된 JSON의 특정 필드를 검사합니다. 예시이다.

```jsx
docker inspect --format='{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' [container_id]
```

### 작업 5. 게시

이미지를 [Google Artifact Registry](https://cloud.google.com/artifact-registry)로 푸시한다. 그런 다음 모든 컨테이너와 이미지를 삭제하여 새로운 환경을 시뮬레이션하고 컨테이너를 가져와서 실행한다. 이를 통해 Docker 컨테이너의 이식성을 확인한다.

```jsx
탐색 메뉴의 CI/CD에서 Artifact Registry > 저장소로 이동합니다.

저장소 만들기를 클릭합니다.

저장소 이름으로 my-repository를 지정합니다.

형식으로 Docker를 선택합니다.

위치 유형에서 리전을 선택한 후 us-central1 (Iowa) 위치를 선택합니다.

만들기를 클릭합니다.
```

이미지를 푸시하거나 가져오려면 먼저 Docker가 Artifact Registry에 대한 요청을 인증하는 데 Google Cloud CLI를 사용하도록 구성

```jsx
gcloud auth configure-docker us-central1-docker.pkg.dev
```

Y 입력

**컨테이너를** **Artifact Registry로 푸시하기**

1. 다음 명령어를 실행하여 프로젝트 ID를 설정하고 Dockerfile이 포함된 디렉터리로 변경

```jsx
export PROJECT_ID=$(gcloud config get-value project)
cd ~/test
```

2. 명령어를 실행하여 `node-app:0.2`에 태그를 지정

```jsx
docker build -t us-central1-docker.pkg.dev/$PROJECT_ID/my-repository/node-app:0.2 .
```

3. 빌드된 Docker 이미지를 확인

```jsx
docker images
```

4. 이미지를 Artifact Registry로 푸시

```jsx
docker push us-central1-docker.pkg.dev/$PROJECT_ID/my-repository/node-app:0.2
```

5. Artifact Registry에서 자신이 만든 repository 확인

**이미지 테스트하기**

1. 테스트를 위해 모든 컨테이너 중지 및 삭제

```jsx
docker stop $(docker ps -q)
docker rm $(docker ps -aq)
```

2. 모든 docker 이미지 삭제

```jsx
docker rmi us-central1-docker.pkg.dev/$PROJECT_ID/my-repository/node-app:0.2
docker rmi node:lts
docker rmi -f $(docker images -aq) # remove remaining images
docker images
```

3. 이미지를 가져와서 실행

```jsx
docker pull us-central1-docker.pkg.dev/$PROJECT_ID/my-repository/node-app:0.2
docker run -p 4000:80 -d us-central1-docker.pkg.dev/$PROJECT_ID/my-repository/node-app:0.2
curl http://localhost:4000
```
