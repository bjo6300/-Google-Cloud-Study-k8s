## 작업 1. 기본 컴퓨팅 영역 설정

**컴퓨팅 영역**이란 클러스터와 리소스가 존재하는 리전 내 대략적인 위치를 의미한다. 예를 들어 `us-central1-a`는 `us-central1` 리전에 속한 영역이다.

1. 기본 컴퓨팅 리전 설정

```jsx
gcloud config set compute/region us-central1
```

2. 기본 컴퓨팅 영역 설정

```jsx
gcloud config set compute/zone us-central1-a
```

---

## 작업 2. GKE 클러스터 만들기

`클러스터`는 1개 이상의 **클러스터 마스터** 머신과 **노드**라는 여러 작업자 머신으로 구성된다. `노드`란 클러스터를 구성하기 위해 필요한 Kubernetes 프로세스를 실행하는 Compute Engine 가상 머신(VM) 인스턴스이다.

![](https://velog.velcdn.com/images/bjo6300/post/8dc6b037-91e0-4856-867c-c521528e9a5d/image.png)


1. 클러스터 만들기 (몇 분정도 걸린다.)

```jsx
gcloud container clusters create --machine-type=e2-medium --zone=us-central1-a lab-cluster
```

---

## 작업 3. 클러스터의 사용자 인증 정보 얻기

1. 클러스터에 인증

```jsx
gcloud container clusters get-credentials lab-cluster
```

---

## 작업 4. 클러스터에 애플리케이션 배포

이제 클러스터에 컨테이너화된 애플리케이션을 배포할 수 있다. 이번 실습에서는 `hello-app`을 클러스터에서 실행한다.

GKE는 Kubernetes 객체를 사용하여 클러스터의 리소스를 만들고 관리한다. 웹 서버와 같은 **스테이트리스**(Stateless) 애플리케이션을 배포할 때는 Kubernetes에서 `배포 객체`를 사용한다. `서비스 객체`는 인터넷에서 애플리케이션에 액세스하기 위한 규칙과 부하 분산 방식을 정의한다.

1. `hello-app` 컨테이너 이미지에서 **새 배포** `hello-server`를 생성

```jsx
kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
```

- `--image`는 배포할 컨테이너 이미지를 지정
    - [Container Registry](https://cloud.google.com/container-registry/docs) 버킷에서 예시 이미지를 가져옵니다.

2. 애플리케이션을 외부 트래픽에 노출할 수 있는 Kubernetes 리소스인 **Kubernetes Service를 생성**

```jsx
kubectl expose deployment hello-server --type=LoadBalancer --port 8080
```

- `type="LoadBalancer"`는 컨테이너의 Compute Engine 부하 분산기를 생성

3. `hello-server` 서비스를 **검사** (1분정도 소요)

```jsx
kubectl get service
```

4. 웹브라우저에서 애플리케이션을 보려면 새 탭을 열고 다음 주소를 입력

```jsx
http://[EXTERNAL-IP]:8080
```

EXTERNAL-IP는 `kubectl get service` 입력했을 때 나오는 EXTERNAL-IP를 입력한다.

---

## 작업 5. 클러스터 삭제

1. 클러스터 삭제

```jsx
gcloud container clusters delete lab-cluster
```

2. Y 입력

몇 분 정도 소요된다.
