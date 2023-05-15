## Kubernetes Engine이란?

- Kubernetes Engine은 컨테이너를 위한 강력한 클러스터 관리자 및 조정 시스템인 `Kubernetes`의 Google Cloud 호스팅 버전
    - `Kubernetes`는 노트북에서 **고가용성 다중 노드 클러스터, 가상 머신에서 베어 메탈까지 다양한 환경에서 실행**할 수 있는 오픈소스 프로젝트
- Kubernetes 앱은 `컨테이너`로 빌드되었으며, 컨테이너는 실행을 위해 필요한 모든 종속 항목 및 라이브러리와 함께 번들로 제공되는 경량 애플리케이션이다.
    - 기본 구조 덕분에 Kubernetes 애플리케이션은 **안전**하고 **가용성이 높으며** **빠른 배포**가 가능하여 클라우드 개발자에게 이상적인 프레임워크이다.
    

## Jenkins란?

![](https://velog.velcdn.com/images/bjo6300/post/9a360b15-e14d-426a-80f9-9deffb91d2a2/image.png)


- 빌드, 테스트, 배포 파이프라인을 유연하게 조정할 수 있는 `오픈소스 자동화 서버`
    - 개발자는 **지속적 배포**로 인해 발생할 수 있는 **오버헤드** 문제에 대한 걱정 없이 **프로젝트를 신속하게 변경 및 개선**할 수 있다.

## 지속적 배포란?

- 지속적 배포(CD) 파이프라인을 설정해야 하는 경우 `Jenkins를 Kubernetes Engine으로 배포`하면 표준 VM 기반 배포 대비 상당한 이점을 얻을 수 있다.
- 빌드 프로세스에서 컨테이너를 사용하는 경우 하나의 가상 호스트로 여러 운영체제에서 작업이 가능하다.
    - Kubernetes Engine에서는 `일시적 빌드 실행자(ephemeral build executors)`를 제공하는데, 이 기능은 **빌드가 활발하게 실행될 때만 사용**되므로 일괄 처리 작업과 같은 다른 클러스터 작업에 사용할 여유 리소스를 확보할 수 있다.
    - 시작하는데 몇 초밖에 걸리지 않는 빠른 속도가 장점이다.

## 작업 1. 소스 코드 다운로드하기

1. 영역을 `us-east1-b`(으)로 설정

```jsx
gcloud config set compute/zone us-east1-b
```

2. 실습 샘플 코드 설정

```jsx
gsutil cp gs://spls/gsp051/continuous-deployment-on-kubernetes.zip .
```

```jsx
unzip continuous-deployment-on-kubernetes.zip
```

3. 디렉토리 이동

```jsx
cd continuous-deployment-on-kubernetes
```

## 작업 2. Jenkins 프로비저닝하기

### Kubernetes 클러스터 만들기

1. Kubernetes 클러스터를 프로비저닝 (수 분 소요)

**프로비저닝** : **IT 인프라를 생성하고 설정하는 프로세스**로서, 다양한 리소스에 대한 사용자 및 시스템 액세스를 관리하는 데 필요한 단계를 포함한다.

```jsx
gcloud container clusters create jenkins-cd \
--num-nodes 2 \
--machine-type n1-standard-2 \
--scopes "https://www.googleapis.com/auth/source.read_write,cloud-platform"
```

scopes : Jenkins가 Cloud Source Repositories 및 Google Container Registry에 액세스할 수 있도록 한다.

2. 클러스터가 실행 중인지 확인

```jsx
gcloud container clusters list
```

3. 클러스터의 사용자 인증 정보를 가져오기

```jsx
gcloud container clusters get-credentials jenkins-cd
```

4. Kubernetes Engine은 이 사용자 인증 정보를 사용하여 새로 프로비저닝된 클러스터에 액세스한다. 이 명령어를 실행해서 연결할 수 있는지 확인하자.

```jsx
kubectl cluster-info
```

## 작업 3. Helm 설정하기

**Helm** : Kubernetes 애플리케이션을 쉽게 구성하고 배포할 수 있게 해주는 `패키지 관리자`

Helm을 사용하여 차트 저장소에서 Jenkins를 설치한다.

1. Helm의 안정적인 차트 저장소 추가

```jsx
helm repo add jenkins https://charts.jenkins.io
```

2. 저장소가 최신 상태인지 확인

```jsx
helm repo update
```

## 작업 4. Jenkins 구성 및 설치하기

- Jenkins 설치 시 `values` 파일을 템플릿으로 사용하여 설정에 필요한 값을 제공할 수 있다.
- 커스텀 `values` 파일을 사용하면 Kubernetes Cloud를 자동으로 구성하고 다음 필수 플러그인을 추가할 수 있다.

```jsx
Kubernetes:latest
Workflow-multibranch:latest
Git:latest
Configuration-as-code:latest
Google-oauth-plugin:latest
Google-source-plugin:latest
Google-storage-plugin:latest
```

1. Helm CLI를 사용하여 해당 구성 설정으로 차트를 배포 (수 분 소요)

```jsx
helm install cd jenkins/jenkins -f jenkins/values.yaml --wait
```

2. 명령어가 완료되면 Jenkins 포드가 `Running` 상태이고 컨테이너가 READY 상태인지 확인

```jsx
kubectl get pods
```

3. 클러스터에 배포할 수 있도록 Jenkins 서비스 계정을 구성

```jsx
kubectl create clusterrolebinding jenkins-deploy --clusterrole=cluster-admin --serviceaccount=default:cd-jenkins
```

4. 다음 명령어를 실행하여 Cloud Shell에서 Jenkins UI로의 포트 전달을 설정

```jsx
export POD_NAME=$(kubectl get pods --namespace default -l "app.kubernetes.io/component=jenkins-master" -l "app.kubernetes.io/instance=cd" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
```

5. Jenkins 서비스가 제대로 만들어졌는지 확인

```jsx
kubectl get svc
```

Jenkins 마스터가 요청할 경우 필요에 따라 빌더 노드가 자동으로 시작되도록 [Kubernetes 플러그인](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin)을 사용하고 있다. 관련 작업이 완료되면 자동으로 해제되고 리소스가 클러스터 리소스 풀에 다시 추가된다.

이 서비스에서는 `selector`와 일치하는 모든 포드가 액세스할 수 있도록 포트 `8080` 및 `50000`을 노출한다. 이를 통해 Kubernetes 클러스터 내의 Jenkins 웹 UI 및 빌더/에이전트 등록 포트가 노출된다. 또한 `jenkins-ui` 서비스는 클러스터 외부에서 액세스할 수 없도록 ClusterIP를 통해 노출된다.

## 작업 5. Jenkins에 연결하기

1. Jenkins 차트에서는 자동으로 관리자 비밀번호를 만들어 줍니다. 이를 확인하려면 다음을 실행한다.

```jsx
printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
```

2. Jenkins 사용자 인터페이스로 이동하려면 Cloud Shell에서 **웹 미리보기** 버튼을 클릭한 다음, **포트 8080에서 미리보기**를 클릭

![](https://velog.velcdn.com/images/bjo6300/post/5ad8c9ea-c9b3-4995-8d8d-8f3bc66867d8/image.png)


3. 메시지가 표시되면 사용자 이름 `admin`과 자동 생성된 비밀번호로 로그인

## 작업 6. 애플리케이션 이해하기

지속적 배포 파이프라인에 샘플 애플리케이션 `gceme`를 배포한다. 이 애플리케이션은 Go 언어로 작성되었으며 저장소의 sample-app 디렉터리에 있다. Compute Engine 인스턴스에서 gceme 바이너리를 실행하면, 앱이 정보 카드에 인스턴스의 메타데이터를 표시한다.

![](https://velog.velcdn.com/images/bjo6300/post/60f6ca45-0248-4cdf-a3e6-d9125d61ab28/image.png)


이 애플리케이션은 마이크로서비스를 모방하여 두 가지 작동 모드를 지원한다.

- **백엔드 모드**에서 gceme는 포트 8080을 수신 대기하고 Compute Engine 인스턴스 메타데이터를 JSON 형식으로 반환한다.
- **프런트엔드 모드**에서 gceme는 백엔드 gceme 서비스를 쿼리하고 결과 JSON을 사용자 인터페이스에서 렌더링한다.

![](https://velog.velcdn.com/images/bjo6300/post/da8af54f-305d-47c9-9eb2-a0e4f908b9a3/image.png)


## 작업 7. 애플리케이션 배포하기

애플리케이션을 2개의 다른 환경에 배포한다.

- **프로덕션**: 사용자가 액세스하는 라이브 사이트이다.
- **카나리아**: 사용자 트래픽 중 **일부만 수용**하는 소규모 사이트다. 이 환경을 사용하여 실제 트래픽으로 소프트웨어의 **이상 유무를 확인**한 후 모든 사용자에게 배포한다.

1. Google Cloud Shell에서 샘플 애플리케이션 디렉터리로 이동

```jsx
cd sample-app
```

2. Kubernetes 네임스페이스를 만들어 배포를 논리적으로 격리

```jsx
kubectl create ns production
```

3. 프로덕션 및 카나리아 배포를 생성하고 `kubectl apply` 명령어를 사용하여 서비스를 생성

```jsx
kubectl apply -f k8s/production -n production
```

```jsx
kubectl apply -f k8s/canary -n production
```

```jsx
kubectl apply -f k8s/services -n production
```

4. 다음 명령어를 실행하여 프로덕션 환경 프런트엔드를 확장

```jsx
kubectl scale deployment gceme-frontend-production -n production --replicas 4
```

`kubectl scale` 명령어를 사용하여 최소 4개의 복제본이 항상 실행되도록 한다.

5. 이제 프런트엔드에 5개의 포드, 프로덕션 트래픽에 4개의 포드, 카나리아 릴리스에 1개의 포드가 실행 중인지 확인한다(카나리아 릴리스에 변경이 생기면 사용자 5명 중 1명(20%)에만 영향을 준다).

```jsx
kubectl get pods -n production -l app=gceme -l role=frontend
```

6. 백엔드에 2개의 포드, 프로덕션에 1개의 포드, 카나리아에 1개의 포드가 있는지 확인한다.

```jsx
kubectl get pods -n production -l app=gceme -l role=backend
```

7. 프로덕션 서비스의 외부 IP를 검색

```jsx
kubectl get service gceme-frontend -n production
```

8. 나중에 사용할 수 있도록 *프런트엔드 서비스* 부하 분산기 IP를 환경 변수에 저장

```jsx
export FRONTEND_SERVICE_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)
```

9. 브라우저에서 프런트엔드 외부 IP 주소를 열어 두 서비스가 정상 작동하는지 확인
10. 서비스의 버전을 확인합니다(1.0.0으로 표시되어야 함).

```jsx
curl http://$FRONTEND_SERVICE_IP/version
```

## 작업 8. Jenkins 파이프라인 만들기

### 샘플 앱 소스 코드를 호스팅하는 저장소 만들기

1. 다음과 같이 `gceme` 샘플 앱의 복사본을 만들고 [Cloud Source Repository](https://cloud.google.com/source-repositories/docs/)로 푸시

```jsx
gcloud source repos create default
```

저장소 만들기

```jsx
git init
```

2. 자체 Git 저장소로 sample-app 디렉터리를 초기화

```jsx
git config credential.helper gcloud.sh
```

3. 다음 명령어 실행한다.

```jsx
git remote add origin https://source.developers.google.com/p/$DEVSHELL_PROJECT_ID/r/default
```

4. Git 커밋을 위한 사용자 이름 및 이메일 주소를 설정한다. 아래에서 `[EMAIL_ADDRESS]`를 Git 이메일 주소로, `[USERNAME]`을 Git 사용자 이름으로 바꾸어 입력한다.

```jsx
git config --global user.email "[EMAIL_ADDRESS]"
```

```jsx
git config --global user.name "[USERNAME]"
```

5. 파일 추가, 커밋, 푸시

```jsx
git add .
```

```jsx
git commit -m "Initial commit"
```

```jsx
git push origin master
```

### 서비스 계정 사용자 인증 정보 추가

- 사용자 인증 정보를 구성하여 Jenkins에서 코드 저장소에 액세스할 수 있도록 허용
- Jenkins는 Cloud Source Repositories에서 코드를 다운로드하기 위해 클러스터의 서비스 계정 사용자 인증 정보를 사용

1. Jenkins 사용자 인터페이스의 왼쪽 탐색 메뉴에서 **Manage Jenkins(Jenkins 관리)**를 클릭하고 **Manage Credentials(사용자 인증 정보 관리)**를 클릭
2. **System(시스템)**을 클릭

![](https://velog.velcdn.com/images/bjo6300/post/aa5e2134-31aa-4e56-9776-46b8d908bec0/image.png)


3. **Global credentials (unrestricted)(전역 사용자 인증 정보(무제한))**를 클릭
4. 오른쪽 상단에서 **Add Credentials(사용자 인증 정보 추가)**를 클릭
5. **Kind(종류)** 드롭다운에서 **Google Service Account from metadata(메타데이터의 Google 서비스 계정)**를 선택하고 **Create(만들기)**를 클릭

![](https://velog.velcdn.com/images/bjo6300/post/30bfcbe0-1c8d-4828-b704-6047ad934f61/image.png)


### Kubernetes용 Jenkins Cloud 구성

1. Jenkins 사용자 인터페이스에서 **Manage Jenkins(Jenkins 관리)** > **Manage nodes and cloud(노드 및 클라우드 관리)**를 선택
2. 왼쪽 탐색창에서 **Configure Clouds(클라우드 구성)**를 클릭
3. **Add a new cloud(새 클라우드 추가)**를 클릭하고 **Kubernetes**를 선택
4. **Kubernetes Cloud Details(Kubernetes 클라우드 세부정보)**를 클릭
5. **Jenkins URL** 필드에 `http://cd-jenkins:8080`을 입력
6. **Jenkins tunnel(Jenkins 터널)** 필드에 `cd-jenkins-agent:50000`을 입력
7. **Save(저장)**를 클릭

### Jenkins 작업 만들기

Jenkins 사용자 인터페이스로 이동하고 다음 단계에 따라 파이프라인 작업을 구성합니다.

1. 왼쪽 패널에서 **Dashboard(대시보드)** > **New Item(새 항목)**을 클릭
2. 프로젝트 이름을 **sample-app**으로 지정하고 **Multibranch Pipeline(다중 브랜치 파이프라인)** 옵션을 선택하고 **OK(확인)**를 클릭
3. 다음 페이지의 **Branch Sources(브랜치 소스)** 섹션에 있는 **Add Source(소스 추가)** 드롭다운에서 **Git**을 선택
4. Cloud Source Repositories에 있는 sample-app 저장소의 **HTTPS clone URL(HTTPS 클론 URL)**을 **Project Repository(프로젝트 저장소)** 필드에 붙여넣는다. `[PROJECT_ID]`를 **프로젝트 ID**로 변경한다.

```jsx
https://source.developers.google.com/p/[PROJECT_ID]/r/default
```

5. 이전 단계에서 서비스 계정을 추가할 때 만든 사용자 인증 정보의 이름을 **사용자 인증 정보** 드롭다운에서 선택
6. **Scan Multibranch Pipeline Triggers(다중 브랜치 파이프라인 트리거 검색)** 섹션에서 **Periodically if not otherwise run(별도로 실행하지 않는 경우 주기적으로 실행)** 상자를 선택하고 **Interval(간격)** 값을 1분으로 설정
7. 작업 구성이 다음과 같이 표시

![](https://velog.velcdn.com/images/bjo6300/post/bd084130-d7fb-4cb9-a4de-d7d3069e1db3/image.png)


8. 다른 모든 옵션은 기본값으로 두고 **Save(저장)**를 클릭

## 작업 9. 개발 환경 만들기

### 개발 브랜치 만들기

기능 브랜치로부터 개발 환경을 만들려면 브랜치를 Git 서버에 푸시하고 Jenkins를 통해 환경을 배포한다.

개발 브랜치를 만들고 Git 서버에 푸시

```jsx
git checkout -b new-feature
```

### 파이프라인 정의 수정하기

해당 파이프라인을 정의하는 `Jenkinsfile`은 [Jenkins 파이프라인 Groovy 구문](https://jenkins.io/doc/book/pipeline/syntax/)을 사용하여 작성된다. `Jenkinsfile`을 사용하면 전체 빌드 파이프라인을 소스 코드가 포함된 단일 파일로 표현할 수 있다. 파이프라인에서는 동시 로드와 같은 강력한 기능을 지원하며 사용자의 수동 승인이 필요하다.

파이프라인이 정상적으로 작동하도록 하려면 `Jenkinsfile`을 수정하여 프로젝트 ID를 설정한다.

1. 터미널 편집기(예: `vi`)에서 Jenkinsfile 열기

```jsx
vi Jenkinsfile
```

2. 편집기 시작

```jsx
i
```

3. `REPLACE_WITH_YOUR_PROJECT_ID` 값에 `PROJECT_ID`를 추가한다. `PROJECT_ID`는 프로젝트 ID로 실습의 `CONNECTION DETAILS` 섹션에 있다. `gcloud config get-value project`를 실행해서 찾을 수도 있다.

4. `CLUSTER_ZONE`의 값을 `<filled in at lab start>`(으)로 변경한다.
5. `Jenkinsfile` 파일을 저장한다. Esc 누르고 다음 명령어 입력한다.

```jsx
:wq
```

### 사이트 수정하기

애플리케이션이 변경되었음을 보여주기 위해 gceme 카드를 **파란색**에서 **주황색**으로 변경한다.

1. html.go 열기

```jsx
vi html.go
```

2. i를 누르고 `<div class="card blue">` 코드를 수정한다.

```jsx
<div class="card orange">
```

3. `html.go` 파일을 저장

```jsx
:wq
```

4. main.go 열기

```jsx
vi main.go
```

5. i를 누르고 코드 수정

1.0.0에서 2.0.0으로 변경한다.

```jsx
const version string = "2.0.0"
```

6. main.go 파일을 한 번 더 저장한다.

```jsx
:wq
```

## 작업 10. 배포 시작하기

1. 변경사항을 커밋하고 푸시

```jsx
git add Jenkinsfile html.go main.go
```

```jsx
git commit -m "Version 2.0.0"
```

```jsx
git push origin new-feature
```

이렇게 하면 개발 환경 빌드가 시작된다.

2. 빌드가 실행되면 왼쪽 탐색 메뉴에서 **build(빌드)** 옆의 아래쪽 화살표를 클릭하고 **Console output(콘솔 출력)**을 선택한다.

![](https://velog.velcdn.com/images/bjo6300/post/2df8805a-3113-42b3-9bcc-35134c635195/image.png)


3. 몇 분 동안 빌드 출력을 추적하면서 `kubectl --namespace=new-feature apply...` 메시지가 시작되는지 확인한다. 이제 new-feature 브랜치가 클러스터에 배포된다.
4. 모두 처리되었다면 다음 명령어를 실행하여 백그라운드에서 프록시를 시작한다.

```jsx
kubectl proxy &
```

5. 정지되는 경우 **Ctrl + C**를 눌러 종료합니다. `localhost`에 요청을 보내고 `kubectl` 프록시에서 이를 서비스에 전달하도록 하여 애플리케이션에 액세스할 수 있는지 확인한다.

```jsx
curl \
http://localhost:8001/api/v1/namespaces/new-feature/services/gceme-frontend:80/proxy/version
```

2.0.0으로 응답이 와야한다. 개발 환경을 설정했다. 이제 이전 모듈에서 학습한 내용을 바탕으로 카나리아 릴리스를 배포하여 새로운 기능을 테스트할 예정이다.

## 작업 11. 카나리아 릴리스 배포하기

1. 카나리아 브랜치를 만들고 Git 서버에 푸시

```jsx
git checkout -b canary
```

```jsx
git push origin canary
```

2. Jenkins에서 **카나리아** 파이프라인이 시작되었음을 알 수 있다. 작업이 완료되면 서비스 URL을 확인하여 일부 트래픽에 새 버전을 제공하고 있는지 파악할 수 있다. 5개의 요청마다 약 1개의 요청(정해진 순서 없음)이 버전 `2.0.0`을 반환해야 한다.

```jsx
export FRONTEND_SERVICE_IP=$(kubectl get -o \
jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)
```

```jsx
while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1; done
```

3. 계속 1.0.0이 표시되면 위의 명령어를 다시 실행해보자. 위의 동작이 확인되면 **Ctrl + C** 키를 사용하여 명령어를 종료한다.

## 작업 12. 프로덕션에 배포하기

1. 카나리아 브랜치를 만들고 Git 서버에 푸시

```jsx
git checkout master
```

```jsx
git merge canary
```

```jsx
git push origin master
```

2. *완료되면*(몇 분 소요될 수 있음) 서비스 URL을 확인하여 모든 트래픽에 새 버전인 2.0.0이 제공되고 있는지 파악할 수 있다.

```jsx
export FRONTEND_SERVICE_IP=$(kubectl get -o \
jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)
```

```jsx
while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1; done
```

3. 역시 `1.0.0` 버전의 인스턴스가 표시되면 위의 명령어를 다시 실행해보자. **Ctrl + C** 키를 눌러 이 명령어를 중단할 수 있다.

![](https://velog.velcdn.com/images/bjo6300/post/22b52779-0474-4182-8e16-4b278046b1a5/image.png)


![](https://velog.velcdn.com/images/bjo6300/post/890e2696-87e7-4a49-8457-effbb7bcb9e9/image.png)

> Jenkins, Kubernetes를 직접 사용해보고 프로덕션, 카나리아 배포를 실습해보니 어떤건지 이해했다. 실제 배포환경에 대해 조금씩 다가가는거 같아서 뿌듯했다.
