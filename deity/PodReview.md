# pod 전
나에게 이책의 가치는 앞에 내용이 아닐까?
devOps와 쿠버네이트가 왜 필요한지까지의 이야기.

> docker-hub의 설치버전은 무엇이 있는가?

# pod 그후.. 2,3장 다시보기
쿠버네이티스는 실행 최소 단위가 pod이다. docker에서 컨테이너를 실행을 할수 있지만. 쿠버네이티스는 docker을 pod에 넣어서 pod를 실행한다말이다.

>굳이 도커로 돌려도 되는데 쿠버네티스를 쓰는 중 제일 큰 이유는 `자동 관리`이다.
도커는 죽으면 끝이다. 죽은지 모를때도 있다. 운영중에 그렇게 된다면 어떻게 될까? 어? 죽었네? 하고 그냥 살릴까? 그동안 중단된 시간은 어떻게 되는 것일까?죽은것을 다시 살려주는게 쿠버네티스의 주 기능.

쿠버네티스는 온라인도 있지만 오프라인도 있다. `미니큐브`가 그 예이다.
> 더 있을수도 있지 않을까?

목록
- 지속적 실행
  - 레플리케이션(rc)
  - 레플리카셋
- 외부 연결
  - 서비스
- 그룹화 : 그중에서 라벨이 살아남았다. 라벨위주로 찾는다.
  - 라벨
  - 주석
  - 네임스페이스

> 주석은 곧 없어진다고 한다. 1.8에서는 사용하지 않고, 1.9에서는 사라진다고 한다.

## rc와 서비스 간단히 해보기

```bash
$kubectl run kubia --image=luksa/kubia --port=8080 --generator=run/v1
# pod생성 결과
kubia-z9kcl      1/1     Running   0          36s

# 외부통신을 위한 로드밸런서 서비스를 만든다.
$kubectl expose rc kubia --type=LoadBalancer --name bubia-http
$kubectl get services
## 결과, EXTERNAL-IP라고 써있는것이 외부주소이니 그걸쓰면된다.
NAME         TYPE           CLUSTER-IP    EXTERNAL-IP     PORT(S)          AGE
bubia-http   LoadBalancer   10.12.6.180   104.197.53.48   8080:31268/TCP   87s

### 브라우저 확인할수 있는 주소, ip는 조회한것을 사용
http://104.197.53.48:8080/
```

약자로 조회하기
```bash
$kubectl get po  # pods,pod -> po
$kubectl get svc # services, service -> svc
$kubectl get rc  # replicationcontrollers,replicaset -> rc
$kubectl get ds  # daemonSet
$kubectl get ns  # namespaces
$kubectl get node # node
```
> $kubectl get po `-n` kube-system -> kubectl get po `-namespace` kube-system
>> po를 namespace로 검색할때 -n으로 줄여서 쓸수 있다.
>>> kubectl get n은 안된다.

* port-forward도 외부 접속이 가능한데?
> port-forward는 하나의 pod에 관련된것이므로 레플리케이션컨트롤에서는 의미가 없다. 그래서 서비스가 필요한것이다.


## 레플리케이션 해보기

```bash
$kubectl get replicationcontrollers
# 결과
NAME    DESIRED   CURRENT   READY   AGE
kubia   1         1         1       28m

$kubectl scale rc kubia --replicas=3
# 결과
NAME    DESIRED   CURRENT   READY   AGE
kubia   3         3         3       30m #3개로 늘어남

$kubectl get po
NAME             READY   STATUS    RESTARTS   AGE
kubia-gsvc8      1/1     Running   0          3m45s
kubia-p4r4l      1/1     Running   0          3m45s
kubia-z9kcl      1/1     Running   0          33m

# 접속 테스트, ip는 kubectl get svc로 알아보자.
$curl 104.197.53.48:8080
#You've hit kubia-p4r4l
$curl 104.197.53.48:8080
#You've hit kubia-gsvc8
$curl 104.197.53.48:8080
#You've hit kubia-p4r4l
$curl 104.197.53.48:8080
#You've hit kubia-z9kcl

# pod의 실행 node보기
$kubectl get po -o wide

# dash보드 url살펴보기, 하지만 내꺼에는 현재 없다. 
$kubectl cluster-info | grep dashboard #안보인다.
$kubectl cluster-info #dashboard가 없다. 기본이라서 그런가?
```

* 포드 하나하나 실행대신에 레플리케이션컨트롤에서 대신 실행

---

yaml과 json형태로 만들려할때 속성을 알려면 다음을 보면된다.
1. 문서 사이트 : http://kubernetes.io/docs/api
2. 명령어로 문의 : explain
``` bash
$kubectl explain pods
$kubectl explain pod.spec # 하이라키 구조로 넣어야하는 것 같다.
```

* 실제 노드가 어떤 하드웨어에 있는지도 알수있는가?

---
# 라벨로 할수 있는 것들
## 조회할때 라벨로 조회가능
## 라벨을 사용한 워커노드 분류
```bash
# 포드 조회
$kubectl get po -l env
$kubectl get po -l creation_method=manual

# 워커노드 분류
$kubectl get po -o wide # 노드확인
$kubectl label node gke-standard-cluster-1-default-pool-3cc53fe8-pqtb gpu=true
$kubectl get nodes -l gpu=true
```
## 노드를 지정을 위한 포드 스케줄링
### 설정파일에 nodeSelector를 써서 지정한다.
어떤 노드에 넣어라고 꼭 집아서 하지는 못한다. 어떤 라벨이 있는 노드중에 만들어라라고 한다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-gpu
spec:
  nodeSelector:
    gpu: "true" #gpu=true가 있는 노드에 생성하도록 한다.
  containers:
  - image: luksa/kubia
    name: kubia  
```
## 특정한 라벨이 있는 지정된 포드에 스케줄링하기

## 라벨이 안되면 주석을 달아서 할수도 있다.

## 네임스페이스를 통해서 그룹화를 할수 있다.

