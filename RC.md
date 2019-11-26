# 포드 배포 방법
3장에서는 포드를 수동으로 생성, 감동, 관리하는 방법을 배웠다.(관리되지 않은 포드) 그렇지만 실제로는 직접 포드를 만드는 일은 없으며 4장에서 배우는 레플리카로 주로 포드를 만들어서 관리를 한다.(관리되고 있는 포드)
- 컨테이너가 살아있는지 확인하고 그렇지 않은 경우 다시 시작하는 방법
- 관리된 포드 실행하는 방법

1. 라이브 프로브
2. 레플리카
   1. 레플리케이션 컨트롤러
   2. 레플리카셋
3. 데몬셋
4. Job
   1. Job : 1번 실행
   2. CronJob : 주기적 1번 실행

4장에 주로 설명하는 레플리케이션컨트롤러와 레플리카셋은 일정의 pod를 유지시켜주는 객체이다. 컨테이너가 살아있는지 확인하여 포드유지에 사용하는 것이 라이브니스 프로브이다.
- 그래서 레플리케이션컨트롤러와 레플라카셋에는 라이브니스 프로브가 설정되어있다.
> ㅇㅇ? 그런내용이 없는데? 꼭 라이브니스 프로브 대체제가 rc와 레플리카셋같다?
>> 레플리카에서 라이브니스 프로브 설정하는 예제가 없다. 
>>> 스터디에서는 레플리카내에 포함되었다고 이야기 했는데... 한번더 확인이 필요하다.

> 파일명이 yml이든 yaml이든 상관없나보다.

### 라이브 프로브 실습
- 살아있니?

kubia-liveness-probe.yml
```yml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:
  - image: luksa/kubia-unhealthy
    name: kubia
    livenessProbe:
      httpGet:
        path: /
        port: 8080
```
```bash
$kubectl create -f kubia-liveness-probe.yml
$kubectl get po kubia-liveness
$kubectl describe po kubia-liveness #오류 확인
```
```yml
apiVersion: v1
kind: Pod
metadata:
  name: kubia-liveness
spec:
  containers:
  - image: luksa/kubia-unhealthy
    name: kubia
    livenessProbe:
      httpGet:
        path: /
        port: 8080
      initialDelaySeconds: 15 #딜레이를 안넣으면 필히 오류가 발생한다.  
```
```bash
$kubectl delete po kubia-liveness
$kubectl create -f kubia-liveness-probe.yml
```

> 컨테이너에서 자바 애플리케이션을 실행하는 경우에는 Exec 프로브 대신 HTTP GET 라이브니스 프로브를 사용해야 한다.       


### rc 실습

kubia-rc.yaml
```yaml
apiVersion: v1
kind: ReplicationController
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:
      containers:
        - name: kubia
          image: luksa/kubia
          ports:
          - containerPort: 8080
```

```bash
$kubectl delete rc kubia
$kubectl create -f kubia-rc.yaml
$kubectl get pods
$kubectl delete po kubia-v8znk
$kubectl get pods
$kubectl get rc
$kubectl describe rc kubia #오류 확인
$kubectl scale rc kubia --replicas=10 # 스케일업
$kubectl edit rc kubia #실행중인 rc 변경, 설정파일변경
$kubectl delete rc kubia --cascade=false #pod를 삭제하지 않고 rc지우기
```
* 일반적으로 rc를 지우면 그에 포함된 pod도 삭제된다.
  - 근데 의미가 있나? 관리가 안되자나?
    - 라벨 셀렉터로 다시 관리할수 있다고 한다.

### 레플리카셋 실습 - 진짜는 9장에서 진행
일반적으로 레플리카셋은 직접 생성하지 않는다. 그 대신 상위 수준의 디플로이먼트 리소스를 만들때 자동으로 생성한다. 
여기서는 그래도 레플리카셋이 어떤지 알아두기 위해서 억지 세트를 만들어서 진행해보겠다.

* 레플리카셋이 나아진점은 라벨 셀렉터의 표현식이 더 좋아졌다는 것이다.

> 실습을 하고 싶은데... v1의 버전을 몰로해야할지 몰라서 계속 오류 발생한다. 우선써놓기는 하자.

kubia-replicaset.yaml
```yaml
apiVersion: apps/v1beta2
kind: ReplicaSet #책에는 대문자로 안되어 있다.
metadata:
  name: kubia
spec:
  replicas: 3
  selector:
    matchLabels:
      app: kubia
  template:
    metadata:
      labels:
        app: kubia
    spec:      # 책에는 metadata안으로 되어있다.
      containers:
      - name: kubia
        image: luksa/kubia
```

```bash
$kubectl get rs
$kubectl describe rs
```

### 데몬셋 실습

ssd-monitor-daemonset.yaml
```yaml
# VI에서 자꾸 띄워쓰기가 멀찍이 되어서 정리안하고 해보았다.
apiVersion: apps/v1beta2
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
                        nodeSelector:
                                disk: ssd
                        containers:
                        - name: main
                          image: luksa/ssd-monitor
```

```bash
$kubectl get ds
$kubectl get node
# 결과 시작
NAME                                                STATUS   ROLES    AGE   VERSION
gke-standard-cluster-1-default-pool-3cc53fe8-5zp7   Ready    <none>   25h   v1.13.11-gke.14
gke-standard-cluster-1-default-pool-3cc53fe8-jr0f   Ready    <none>   25h   v1.13.11-gke.14
gke-standard-cluster-1-default-pool-3cc53fe8-pqtb   Ready    <none>   25h   v1.13.11-gke.14
# 결과 완료, 노드에 라벨 설정하해야한다.
$kubectl label node gke-standard-cluster-1-default-pool-3cc53fe8-5zp7 disk=ssd
```

### 배치(Job) 실습

export.yaml
```yaml
apiVersion: batch/v1
kind: Job
metadata:
        name: batch-job
spec:
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

``` bash
$kubectl create -f export.yaml
$kubectl get jobs
$kubectl get po
$kubectl get po -a
$kubectl logs batch-job-2sr49
$kubectl scale job [잡명칭] --replicas 3 #병렬처리로 변경(한번에 처리하는 수 정의)
```

### CronJob 실습

cronjob.yaml
```yaml
apiVersion: batch/v1beta1
kind: CronJob
metadata:
        name: batch-job-every-fifteen-minutes
spec:
        schedule: "0,15,30,25 ***" #매이 작업은 매일 매시간 0,15,305분에 일한다는
        jobTemplate:
                spec:
                        template:
                                metadata:
                                        labels:
                                                app: periodic-batch-job
                                spec:
                                        restartPolicy: OnFailure
                                        containers:
                                                -name: main
                                                -image: luksa/batch-bjo
```


서비스 알아보기

kubia-svc.xml
``` yaml
apiVersion: v1
kind: Service
metadata:
        name: kubia
spec:
        ports:
                - port: 80
                  targetPort: 8080
        selector:
                app: kubia
```                

질문
--- 

1. 스케줄은 무엇을 말하는가?