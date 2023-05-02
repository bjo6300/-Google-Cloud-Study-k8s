## 간단한 Kubernetes 데모

nginx 컨테이너의 단일 인스턴스를 실행

- Kubernetes가 배포를 생성
    - 배포 덕분에 pod가 작동하고 있으며, pod가 실행하는 노드에 오류가 발생해도 계속해서 작동

```jsx
kubectl create deployment nginx --image=nginx:1.10.0
```

---

실행 중인 nginx 컨테이너를 확인

```jsx
kubectl get pods
```

Kubernetes 외부로 노출

- 이 공개 IP 주소를 조회하는 모든 클라이언트는 서비스 백그라운드에 있는 포드로 라우팅

```jsx
kubectl expose deployment nginx --port 80 --type LoadBalancer
```

`kubectl get services` 를 통해 ExternalIP를 가져올 수 있고 `http://<External IP>:80` 으로 컨테이너를 조회할 수 있다.

## 포드

- 포드는 1개 이상의 컨테이너가 포함된 모음을 나타낸다.
- 일반적으로 상호 의존성이 높은 컨테이너가 여러 개 있으면 이를 하나의 포드에 패키징한다.
- 포드에는 [볼륨](http://kubernetes.io/docs/user-guide/volumes/) 또한 포함한다.
    - 볼륨 : 포드가 존재하는 한 계속해서 존재하는 데이터 디스크
- 이 예시의 포드 안에 있는 2개의 컨테이너는 서로 통신할 수 있으며 첨부된 볼륨도 공유

![](https://velog.velcdn.com/images/bjo6300/post/438bb48e-e981-42cf-b56c-cd55fe44a2ca/image.png)


## 포드 만들기

```jsx
# 모놀리식 포드 구성 예시, 이 파일을 생성하고 모놀리식 포드 만드는 코드를 실행한다.
apiVersion: v1
kind: Pod
metadata:
  name: monolith
  labels:
    app: monolith
spec:
  containers:
    - name: monolith
      image: kelseyhightower/monolith:1.0.0
      args:
        - "-http=0.0.0.0:80"
        - "-health=0.0.0.0:81"
        - "-secret=secret"
      ports:
        - name: http
          containerPort: 80
        - name: health
          containerPort: 81
      resources:
        limits:
          cpu: 0.2
          memory: "10Mi"
```

모놀리식 포드 만드는 코드

```jsx
kubectl create -f pods/monolith.yaml
```

기본 네임스페이스에서 실행 중인 모든 포드 나열 (몇 초 소요)

- Docker Hub에서 모놀리식 컨테이너 이미지를 가져온다.

```jsx
kubectl get pods
```

## 포드와 상호작용하기

포드에는 기본적으로 비공개 IP 주소가 부여되며 클러스터 밖에서는 접근할 수 없다. `kubectl port-forward` 명령어를 사용하여 로컬 포트를 모놀리식 포드 안의 포트로 매핑한다.

포드 전달 설정 (새로운 터미널)

```jsx
kubectl port-forward monolith 10080:80
```

pod와 통신

```jsx
curl http://127.0.0.1:10080
```

실시간 로그 스트림 가져오기

```jsx
kubectl logs monolith
```

모놀리식 포드의 대화형 셸 실행(컨테이너 내부 접속)

```jsx
kubectl exec monolith --stdin --tty -c monolith /bin/sh
```

셸 나가기

```jsx
exit
```

## 서비스

![](https://velog.velcdn.com/images/bjo6300/post/75b85e62-03da-45fb-9d2b-ec321c4ba169/image.png)


- 포드를 위해 안정적인 엔드포인트를 제공
- 포드는 영구적으로 지속되지 않기 때문에 사용
    - 다시 시작 시 IP 변경 가능
- 라벨을 사용하여 어떤 포드에서 작동할지 결정
- 포드에 라벨이 정확히 지정되어 있다면 서비스가 이를 자동으로 감지하고 노출

서비스가 제공하는 포드 집합에 대한 액세스 수준 유형

- `ClusterIP`(내부) - 기본 유형이며 이 서비스는 클러스터 안에서만 볼 수 있다.
- `NodePort` 클러스터의 각 노드에 외부에서 액세스 가능한 IP 주소를 제공
- `LoadBalancer`는 클라우드 제공업체로부터 부하 분산기를 추가하며 서비스에서 유입되는 트래픽을 내부에 있는 노드로 전달

## 서비스 만들기

보안이 설정된 모놀리식 포드와 구성 데이터를 만들기

```jsx
kubectl create secret generic tls-certs --from-file tls/
kubectl create configmap nginx-proxy-conf --from-file nginx/proxy.conf
kubectl create -f pods/secure-monolith.yaml
```

monolith.yaml

```
kind: Service
apiVersion: v1
metadata:
  name: "monolith"
spec:
  selector:
    app: "monolith".    #enabled 라벨이 지정된 포드를 자동으로 찾고 노출시키는 선택기
    secure: "enabled"
  ports:
    - protocol: "TCP"
      port: 443
      targetPort: 443.  #외부 트래픽을 포트 31000에서 포트 443의 nginx로 전달하기 위해 NodePort를 노출
      nodePort: 31000
  type: NodePort
```

모놀리식 서비스 만들기

```jsx
kubectl create -f services/monolith.yaml
```

트래픽을 노출된 NodePort의 모놀리식 서비스로 보내기

```jsx
gcloud compute firewall-rules create allow-monolith-nodeport \
  --allow=tcp:31000
```

노드 1개의 외부 IP 주소를 가져오기

```jsx
gcloud compute instances list
```

curl을 이용해 조회하면 시간 초과가 뜬다. 다음 섹션에서 해결하겠다.

```jsx
kubectl get services monolith

kubectl describe services monolith

질문:

모놀리식 서비스가 응답하지 않은 이유는 무엇인가요?
모놀리식 서비스는 몇 개의 엔드포인트를 가지고 있나요?
모놀리식 서비스가 포드를 감지하게 하려면 포드에 어떤 라벨이 지정되어 있어야 하나요?
```

## 포드에 라벨 추가하기

모놀리식 라벨이 지정되어 실행되는 포드 몇 개가 있다는 사실을 확인

```jsx
kubectl get pods -l "app=monolith"
```

보안이 설정된 모놀리식 포드에 누락된 `secure=enabled` 라벨을 추가

```jsx
kubectl label pods secure-monolith 'secure=enabled'
kubectl get pods secure-monolith --show-labels
```

모놀리식 서비스의 엔드포인트 목록 확인

```jsx
kubectl describe services monolith | grep Endpoints
```

노드 중 하나를 조회하여 엔드포인트 테스트

```jsx
gcloud compute instances list
curl -k https://<EXTERNAL_IP>:31000
```

## Kubernetes로 애플리케이션 배포하기

디플로이먼트(deployments)

- 실행중인 포드의 개수가 사용자가 명시한 포드 개수와 동일하게 만드는 선언적 방식
- 어떤 이유로든 포드가 중지되면 재시작을 담당하여 처리

![](https://velog.velcdn.com/images/bjo6300/post/0e3eb31b-1471-4658-ba01-e73f5b4ad202/image.png)


예시)

![](https://velog.velcdn.com/images/bjo6300/post/20b1b8b2-410b-43a1-943d-786d9aa626d6/image.png)


포드는 생성 기반 노드의 전체 기간과 연결되어 있다. 위 예시에서 Node3이 중단되면서 포드도 중단되었다. 직접 새로운 포드를 만들고 이를 위한 노드를 찾는 대신, 디플로이먼트가 새로운 포드를 만들고 Node2에서 실행했다.

## 디플로이먼트 만들기

모놀리식 앱을 3가지 부분으로 나눈다.

```jsx
auth - 인증된 사용자를 위한 JWT 토큰을 생성합니다.
hello - 인증된 사용자를 안내합니다.
frontend - 트래픽을 auth 및 hello 서비스로 전달합니다.
```

각 서비스용 디플로이먼트를 만들 준비가 됐다. 그런 다음 auth 및 hello 디플로이먼트용 내부 서비스와 frontend 디플로이먼트용 외부 서비스를 정의하겠다. 이렇게 하면 모놀리식과 같은 방식으로 마이크로서비스와 상호작용할 수 있으며, 각 서비스를 독립적으로 확장하고 배포할 수 있다.

deployments/auth.yaml

```jsx
apiVersion: apps/v1
kind: Deployment
metadata:
  name: auth
spec:
  selector:
    matchlabels:
      app: auth
  replicas: 1
  template:
    metadata:
      labels:
        app: auth
        track: stable
    spec:
      containers:
        - name: auth
          image: "kelseyhightower/auth:2.0.0"
          ports:
            - name: http
              containerPort: 80
            - name: health
              containerPort: 81
...
```

디플로이먼트 개체 만들기

```jsx
kubectl create -f deployments/auth.yaml
```

디플로이먼트용 서비스 만들기

```jsx
kubectl create -f services/auth.yaml
```

hello 디플로이먼트 만들기와 노출

```jsx
kubectl create -f deployments/hello.yaml
kubectl create -f services/hello.yaml
```

frontend 디플로이먼트 만들기와 노출

```jsx
kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
kubectl create -f deployments/frontend.yaml
kubectl create -f services/frontend.yaml
```

외부 IP 주소를 확보하고 frontend와 상호작용

```jsx
kubectl get services frontend # EXTERNAL-IP 만드는데 수 분 소요
```

```jsx
curl -k https://<EXTERNAL-IP>
```
