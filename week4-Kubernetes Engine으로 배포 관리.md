## 실습에서 사용할 샘플코드 가져오기

1. 컨테이너와 배포 만들기

```jsx
gsutil -m cp -r gs://spls/gsp053/orchestrate-with-kubernetes .
cd orchestrate-with-kubernetes/kubernetes
```

2. 노드 3개로 클러스터 만들기 (수 분 소요)

```jsx
gcloud container clusters create bootcamp \
  --machine-type e2-small \
  --num-nodes 3 \
  --scopes "https://www.googleapis.com/auth/projecthosting,storage-rw"
```

## 작업 1. 배포 객체에 관해 알아보기

1. 배포 객체에 대해 알기

```jsx
kubectl explain deployment
```

2. 배포 객체의 모든 필드 보기

```jsx
kubectl explain deployment --recursive
```

## 작업 2. 배포 만들기

1. `deployments/auth.yaml` 구성 파일을 업데이트

```jsx
vi deployments/auth.yaml
```

2. 편집기 시작해서 `i` 로 `image: kelseyhightower/auth:1.0.0` 수정
3. 저장 (:wq)
4. cat 명령어로 확인

```jsx
cat deployments/auth.yaml
```

- `kubectl create` 명령어를 실행하여 인증 배포를 만들면 배포 매니페스트의 데이터에 따라 하나의 포드가 생성
- `replicas` 필드에 지정된 숫자를 변경하여 **포드의 수**를 조정할 수 있다.

5. 배포 객체 생성

```jsx
kubectl create -f deployments/auth.yaml
```

6. 배포 생성 여부 확인

```jsx
kubectl get deployments
```

7. 배포에 관한 ReplicaSet를 생성 여부 확인

```jsx
kubectl get replicasets
```

8. 배포의 일부로 생성된 포드 확인

```jsx
kubectl get pods
```

9. 인증을 배포하기 위한 서비스 생성

```jsx
kubectl create -f services/auth.yaml
```

10. 같은 방법으로 hello 배포를 만들고 노출

```jsx
kubectl create -f deployments/hello.yaml
kubectl create -f services/hello.yaml
```

11. frontend 배포를 만들고 노출

```jsx
kubectl create secret generic tls-certs --from-file tls/
kubectl create configmap nginx-frontend-conf --from-file=nginx/frontend.conf
kubectl create -f deployments/frontend.yaml
kubectl create -f services/frontend.yaml
```

12. 외부 IP를 가져와서 프런트엔드와 연결함으로써 프런트엔드와 상호작용

```jsx
kubectl get services frontend
```

```jsx
curl -ks https://<EXTERNAL-IP>
```

EXTERNAAL-IP 생성되고 나서 응답 확인

curl의 한 줄 명령어

```jsx
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`
```

### 배포 확장

`spec.replicas` 필드를 업데이트한다.

1. 필드에 관한 설명 확인

```jsx
kubectl explain deployment.spec.replicas
```

2. replicas 필드를 업데이트

```jsx
kubectl scale deployment hello --replicas=5
```

배포가 업데이트된 후, Kubernetes는 연결된 ReplicaSet를 자동으로 업데이트하고 새로운 pod를 시작하여 pod의 총 개수를 5로 만든다.

3. `hello` 포드가 5개 실행되고 있는지 확인

```jsx
kubectl get pods | grep hello- | wc -l
```

4. 애플리케이션 다시 축소

```jsx
kubectl scale deployment hello --replicas=3
```

레플리카 3개로 축소

5. 포드 개수 확인

```jsx
kubectl get pods | grep hello- | wc -l
```

## 작업 3. 순차적 업데이트

![](https://velog.velcdn.com/images/bjo6300/post/cb327e15-663a-471f-834c-99db4d55febb/image.png)


배포는 순차적 업데이트 메커니즘을 통해 이미지를 새 버전으로 업데이트하도록 지원한다. 배포가 새 버전으로 업데이트되면 새 ReplicaSet가 만들어지고, 이전 ReplicaSet의 복제본이 감소하면서 새 ReplicaSet의 복제본 수가 천천히 증가한다.

### 순차적 업데이트 트리거하기

1. 배포 업데이트

```jsx
kubectl edit deployment hello
```

2. 배포의 containers 섹션에 있는 `image`를 다음과 같이 변경

```jsx
...
containers:
  image: kelseyhightower/hello:2.0.0
...
```

3. 저장 후 종료
4. Kubernetes에서 생성한 새로운 ReplicaSet를 확인

```jsx
kubectl get replicaset
```

5. 출시 기록에 새로운 항목이 표시될 수도 있다.

```jsx
kubectl rollout history deployment/hello
```

### 순차적 업데이트 일시중지하기

실행 중인 출시에 문제가 발생하면 일시중지하여 업데이트를 중지한다.

1. 중지하기

```jsx
kubectl rollout pause deployment/hello
```

2. 현재 출시 상태 확인

```jsx
kubectl rollout status deployment/hello
```

3. 포드에서 확인

```jsx
kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

### 순차적 업데이트 재개

출시가 일시중지되었으므로 일부 포드는 새 버전이고 일부 포드는 이전 버전이다.

1. 출시를 계속 진행

```jsx
kubectl rollout resume deployment/hello
```

2. 출시가 완료되면 `status` 명령어를 실행할 때 다음이 표시된다.

```jsx
kubectl rollout status deployment/hello
```

### 업데이트 롤백하기

새 버전에서 버그가 발견되었다고 가정해 보겠다. 새 버전에 문제가 있는 것으로 간주되므로 새 포드에 연결된 모든 사용자가 문제를 경험하게 된다.

이전 버전으로 롤백하여 문제를 조사한 다음 제대로 수정된 버전을 출시할 수 있다.

1. 이전 버전으로 롤백

```jsx
kubectl rollout undo deployment/hello
```

2. 기록에서 롤백 확인

```jsx
kubectl rollout history deployment/hello
```

3. 모든 포드가 이전 버전으로 롤백되었는지 확인

```jsx
kubectl get pods -o jsonpath --template='{range .items[*]}{.metadata.name}{"\t"}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
```

## 작업 4. Canary 배포

![](https://velog.velcdn.com/images/bjo6300/post/457f4132-d484-43b1-954b-a9f5aaf4760b/image.png)


- 카나리아 새처럼 위험을 빠르게 감지할 수 있는 배포 기법이다.
- 구 버전의 서버와 새 버전의 서버들을 구성하고 일부 트래픽을 새 버전으로 분산하여 오류 여부를 판단한다.
- 작은 규모의 일부 사용자에게만 변경 사항을 릴리스하여 새로운 릴리스와 관련된 위험을 완화할 수 있다.

1. 새 버전의 새로운 Canary 배포를 만든다.

```jsx
cat deployments/hello-canary.yaml
```

2. Canary 배포 만들기

```jsx
kubectl create -f deployments/hello-canary.yaml
```

3. Canary 배포를 만들면 `hello` 및 `hello-canary`의 두 가지 배포가 생긴다. 다음 `kubectl` 명령어로 확인한다.

```jsx
kubectl get deployments
```

### Canary 배포 확인하기

1. hello 버전 확인

```jsx
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
```

2. 이 명령어를 여러 번 실행하면, 일부 요청은 hello 1.0.0에서 제공하고 소규모 하위 집합(1/4=25%)은 2.0.0에서 제공함을 알 수 있다.

## 작업 5. Blue/Green 배포

![](https://velog.velcdn.com/images/bjo6300/post/04341bfa-77ef-4642-8644-6ce57d7b7391/image.png)


- 구 버전에서 새 버전으로 일제히 전환하는 전략
- 구 버전의 서버와 새 버전의 서버들을 동시에 나란히 구성하고 배포 시점이 되면 트래픽을 일제히 전환시킨다. 하나의 버전만 프로덕션 되므로 버전 관리 문제를 방지할 수 있고, 또한 빠른 롤백이 가능하다.
- 단, 시스템 자원이 두 배로 필요하고, 전체 플랫폼에 대한 테스트가 진행 되어야 한다.

### 서비스

서비스 업데이트

```jsx
kubectl apply -f services/hello-blue.yaml
```

Blue : 이전 버전, Green : 새 버전으로 가정한다.

### Blue/Green 배포를 사용하여 업데이트하기

1. Green 배포 만들기

```jsx
kubectl create -f deployments/hello-green.yaml
```

2. Green 배포가 있고 제대로 시작된 경우 현재 1.0.0 버전이 아직 사용되고 있는지 확인

```jsx
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
```

3. 서비스가 새 버전을 가리키도록 업데이트

```jsx
kubectl apply -f services/hello-green.yaml
```

4. 서비스가 업데이트되면 'green' 배포가 즉시 사용된다. 이제 항상 새 버전이 사용되고 있는지 확인한다.

```jsx
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
```

### Blue/Green 롤백

1. 'blue' 배포가 아직 실행 중일 때 서비스를 이전 버전으로 다시 업데이트

```jsx
kubectl apply -f services/hello-blue.yaml
```

2. 서비스를 업데이트하면 롤백이 성공적으로 완료된다. 사용 중인 버전이 정확한지 다시 확인한다.

```jsx
curl -ks https://`kubectl get svc frontend -o=jsonpath="{.status.loadBalancer.ingress[0].ip}"`/version
```
