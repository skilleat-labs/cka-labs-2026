# 7강 · 스케줄링

> 영상: **07강-스케줄링** · 이어서 풀 채점 세트: `07-scheduling`

## 실습 목표

- 파드가 `Pending` 에 머물 때 **Events 메시지 한 줄로 원인을 읽는** 연습을 한다.
- requests 는 **스케줄링 기준**, limits 는 **런타임 상한**이라는 것을 노드 할당량·OOMKilled 로 확인한다.
- QoS 등급, LimitRange, ResourceQuota 가 **자원을 안 적은 파드**를 어떻게 다루는지 비교한다.
- nodeSelector · Node Affinity · Taint/Toleration · Pod (Anti-)Affinity 를 **방향(누가 누구를 고르나)** 으로 구분한다.
- HPA 가 **requests 와 metrics-server** 없이는 동작하지 않는다는 것을 직접 본다.
- PriorityClass 가 붙은 파드가 **자리를 빼앗는(preemption)** 장면을 본다.

---

## 0) 준비

이 페이지의 리소스는 `guide-sched` 네임스페이스에 만듭니다. 노드 레이블·taint·PriorityClass 처럼 **클러스터 전체에 남는 것**도 만들기 때문에 마지막 **정리** 단계를 꼭 진행하세요.

```bash
kubectl create namespace guide-sched
kubectl get nodes
```

### 노드의 "쓸 수 있는 양" 확인

스케줄러는 노드의 **Allocatable** 과 이미 예약된 **requests 합**을 비교합니다. 먼저 워커 노드의 현재 상태를 봐 둡니다.

```bash
kubectl describe node worker-1 | grep -A6 -E '^Allocatable|^Allocated resources'
```

```text
Allocatable:
  cpu:                2
  ephemeral-storage:  ...
  memory:             1912Mi
  pods:               110
...
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests     Limits
  --------           --------     ------
  cpu                350m (17%)   0 (0%)
  memory             140Mi (7%)   340Mi (17%)
```

VM 사양에 따라 숫자는 다릅니다. **내 워커의 CPU 가 몇 코어인지**만 기억해 두세요. 9) 에서 씁니다.

### metrics-server (8) HPA 와 `kubectl top` 에 필요)

kubeadm 클러스터에는 metrics-server 가 기본으로 없습니다. 먼저 있는지 확인합니다.

```bash
kubectl top nodes
```

```text
error: Metrics API not available
```

이렇게 나오면 설치합니다. 실습 VM 의 kubelet 인증서는 자체 서명이라서 `--kubelet-insecure-tls` 인자를 하나 추가해야 합니다.

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

kubectl patch deployment metrics-server -n kube-system --type=json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

kubectl rollout status deployment/metrics-server -n kube-system
```

1분쯤 뒤 다시 확인합니다.

```bash
kubectl top nodes
```

```text
NAME            CPU(cores)   CPU(%)   MEMORY(bytes)   MEMORY(%)
control-plane   182m         9%       1402Mi          73%
worker-1        61m          3%       690Mi           36%
worker-2        58m          2%       655Mi           34%
```

!!! warning "metrics-server 파드가 Running 인데 `top` 이 안 된다면"
    `kubectl logs -n kube-system deploy/metrics-server` 에 `x509: cannot validate certificate` 가 보이면 위의 `--kubelet-insecure-tls` 가 빠진 것입니다.
    시험 환경에는 보통 metrics-server 가 이미 설치돼 있습니다. 이 설치는 **실습 클러스터용**입니다.

---

## 1) 스케줄러는 거르고, 점수를 매기고, 적는다 — Pending 읽기

스케줄러는 ① 조건에 안 맞는 노드를 **거르고(Filter)** ② 남은 노드에 **점수를 매겨(Score)** ③ 파드에 `nodeName` 을 **적습니다(Bind)**.
① 에서 노드가 하나도 안 남으면 파드는 `Pending` 에 머물고, **왜 걸렀는지가 Events 에 그대로 적힙니다.** 이 메시지를 읽는 것이 이 강의 절반입니다.

### 일부러 Pending 만들기

CPU 64 코어를 요청하는 파드를 만듭니다. 실습 노드에는 그런 노드가 없습니다.

`greedy.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: greedy
  namespace: guide-sched
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: "64"          # ← 64 코어. 이런 노드는 없다
```

```bash
kubectl apply -f greedy.yaml
kubectl get pod greedy -n guide-sched -o wide
kubectl describe pod greedy -n guide-sched | tail -5
```

```text
NAME     READY   STATUS    NODE
greedy   0/1     Pending   <none>

Events:
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  5s    default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 Insufficient cpu. preemption: 0/3 nodes are available: 1 Preemption is not helpful for scheduling, 2 No preemption victims found for incoming pod.
```

**관찰 포인트** — 메시지는 항상 "전체 몇 개 중 몇 개가, 각각 왜 걸러졌는지"입니다. 노드 3대가 **이유별로 나뉘어** 적혀 있습니다(taint 1 + cpu 부족 2). 뒤의 `preemption:` 부분은 "다른 파드를 쫓아내도 자리가 안 난다"는 뜻이며 9) 에서 다시 봅니다.

| Events 메시지 | 원인 | 이 페이지 |
|---------------|------|-----------|
| `Insufficient cpu` / `Insufficient memory` | requests 합이 노드 여유를 넘음 | 2) |
| `had untolerated taint {...}` | 노드 taint 를 못 견딤 | 6) |
| `didn't match Pod's node affinity/selector` | nodeSelector·affinity 불일치 | 5) |
| `didn't match pod anti-affinity rules` | 같은 노드에 이미 그 파드가 있음 | 7) |

```bash
kubectl delete pod greedy -n guide-sched
```

!!! tip "시험 팁 — Events 만 빨리 보기"
    `describe` 출력이 길면 끝만 봅니다. 네임스페이스 전체 이벤트를 시간순으로 보는 방법도 있습니다.
    ```bash
    kubectl describe pod <이름> -n <ns> | tail -5
    kubectl get events -n <ns> --sort-by=.lastTimestamp | tail
    kubectl get events -n <ns> --field-selector reason=FailedScheduling
    ```

---

## 2) requests 와 limits — 예약과 상한

- **requests** — "이만큼은 보장해 달라". **스케줄러**가 노드를 고를 때 봅니다.
- **limits** — "이 이상은 쓰지 마라". 노드에서 **kubelet·커널**이 강제합니다.

스케줄러는 limits 를 보지 않습니다. 노드가 실제로는 한가한데 파드가 `Pending` 인 상황은 대부분 **다른 파드들이 requests 로 예약만 해 둔** 경우입니다.

### 필드 이름 찾기 — `kubectl explain`

시험장에서 필드 구조가 기억나지 않으면 문서보다 `explain` 이 빠릅니다.

```bash
kubectl explain pod.spec.containers.resources          # limits · requests · claims 필드와 설명이 나온다
kubectl explain pod.spec.containers.resources.requests
```

### 자원을 적은 파드

뼈대는 명령으로 뽑고 `resources` 만 손으로 넣습니다. (`kubectl run` 에는 자원 옵션이 없습니다.)

```bash
kubectl run sized -n guide-sched --image=nginx:1.27 $do > sized.yaml
```

`sized.yaml` 을 다음처럼 고칩니다.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sized
  namespace: guide-sched
spec:
  containers:
    - name: sized
      image: nginx:1.27
      resources:
        requests:
          cpu: 50m           # ← 0.05 코어. 스케줄러가 이 값으로 노드를 고른다
          memory: 64Mi       # ← Mi = 1024 기반. M(1000 기반)과 다르다
        limits:
          cpu: 300m          # ← 넘으면 느려진다 (throttle)
          memory: 192Mi      # ← 넘으면 죽는다 (OOMKilled)
```

```bash
kubectl apply -f sized.yaml
kubectl get pod sized -n guide-sched -o wide
kubectl get pod sized -n guide-sched -o jsonpath='{.spec.containers[0].resources}{"\n"}'
```

```text
NAME    READY   STATUS    NODE
sized   1/1     Running   worker-2

{"limits":{"cpu":"300m","memory":"192Mi"},"requests":{"cpu":"50m","memory":"64Mi"}}
```

파드가 뜬 노드의 **Allocated resources** 를 0) 에서 본 값과 비교합니다.

```bash
kubectl describe node worker-2 | grep -A5 'Allocated resources'
```

```text
  Resource           Requests     Limits
  --------           --------     ------
  cpu                400m (20%)   300m (15%)     ← requests 가 50m 늘었다
  memory             204Mi (10%)  532Mi (27%)
```

**관찰 포인트** — 파드 하나가 떴을 뿐인데 노드의 Requests 가 정확히 `50m`, `64Mi` 늘었습니다. 파드가 **실제로 쓰는 양**(`kubectl top pod`)과는 관계없이, 스케줄러는 이 "예약 장부"만 봅니다.

```bash
kubectl top pod sized -n guide-sched
```

```text
NAME    CPU(cores)   MEMORY(bytes)
sized   0m           3Mi            ← 실제 사용량은 예약보다 훨씬 작다
```

### 실험 — 메모리 limit 을 넘기면

CPU 는 나눠 쓸 수 있는 자원이라 limit 을 넘으면 **느려질 뿐**입니다. 메모리는 나눠 쓸 수 없어서 넘으면 커널이 프로세스를 **죽입니다**.
공식 문서 예제에 쓰이는 `polinux/stress` 이미지로 150M 를 쓰게 하고 limit 은 64Mi 로 둡니다.

`oom.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: oom
  namespace: guide-sched
spec:
  containers:
    - name: stress
      image: polinux/stress
      command: ["stress"]
      args: ["--vm", "1", "--vm-bytes", "150M", "--vm-hang", "1"]   # ← 150M 를 잡으려 한다
      resources:
        requests:
          memory: 32Mi
        limits:
          memory: 64Mi       # ← 상한은 64Mi
```

```bash
kubectl apply -f oom.yaml
kubectl get pod oom -n guide-sched -w      # Ctrl+C 로 종료
```

```text
NAME   READY   STATUS              RESTARTS
oom    0/1     ContainerCreating   0
oom    0/1     OOMKilled           0
oom    0/1     OOMKilled           1
oom    0/1     CrashLoopBackOff    1
```

```bash
kubectl describe pod oom -n guide-sched | grep -A5 'Last State'
```

```text
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
```

**관찰 포인트** — 로그에는 에러가 없는데 재시작이 반복됩니다. `Last State` 의 `OOMKilled` 와 `Exit Code: 137`(= 128 + SIGKILL 9)이 원인을 알려 줍니다.
"재시작이 반복되는데 로그가 조용하다" → **OOMKilled 부터 의심**하세요.

| | requests | limits |
|---|---|---|
| 뜻 | 이만큼은 보장 (예약) | 이 이상은 금지 (상한) |
| 누가 보나 | 스케줄러 | kubelet · 커널(cgroup) |
| 넘으면 (CPU) | — | throttle, 느려짐 |
| 넘으면 (메모리) | — | OOMKilled, 재시작 |
| 안 적으면 | 0 으로 취급 → 아무 노드에나 몰림 | 무제한 (LimitRange 가 있으면 기본값) |

```bash
kubectl delete pod oom -n guide-sched
```

---

## 3) QoS 등급 — 자원이 모자랄 때 누가 먼저 쫓겨나나

QoS 등급은 **내가 정하는 값이 아니라** requests·limits 를 어떻게 적었는지로 자동 결정됩니다. 노드 메모리가 모자라 kubelet 이 파드를 쫓아낼 때 이 순서를 씁니다.

`qos.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: qos-g
  namespace: guide-sched
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        limits:              # ← limits 만 적으면 requests 가 같은 값으로 채워진다
          cpu: 150m
          memory: 96Mi
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-b
  namespace: guide-sched
spec:
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:            # ← requests 만 있다 (limits 와 다름)
          cpu: 50m
---
apiVersion: v1
kind: Pod
metadata:
  name: qos-be
  namespace: guide-sched
spec:
  containers:
    - name: app
      image: nginx:1.27      # ← resources 가 아예 없다
```

```bash
kubectl apply -f qos.yaml
kubectl get pods -n guide-sched -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass
kubectl get pod qos-g -n guide-sched -o jsonpath='{.spec.containers[0].resources.requests}{"\n"}'
```

```text
NAME     QOS
qos-b    Burstable
qos-be   BestEffort
qos-g    Guaranteed
sized    Burstable

{"cpu":"150m","memory":"96Mi"}     ← 적지 않은 requests 가 limits 와 같게 채워졌다
```

**관찰 포인트** — `qos-g` 에는 limits 만 적었는데 Guaranteed 입니다. requests 가 생략되면 **limits 값으로 채워지기** 때문입니다. 2) 의 `sized` 는 requests ≠ limits 라서 Burstable 입니다.

| 등급 | 조건 | 노드 자원이 모자라면 |
|------|------|---------------------|
| Guaranteed | **모든** 컨테이너에 cpu·memory 의 requests == limits | 가장 마지막에 쫓겨난다 |
| Burstable | 하나라도 requests/limits 가 있지만 Guaranteed 는 아님 | 중간 |
| BestEffort | 어떤 컨테이너에도 requests·limits 가 없음 | 가장 먼저 쫓겨난다 |

!!! warning "컨테이너가 둘 이상이면"
    Guaranteed 는 **모든 컨테이너**가 조건을 만족해야 합니다. 사이드카 하나에 resources 를 빠뜨리면 파드 전체가 Burstable 이 됩니다.

---

## 4) LimitRange 와 ResourceQuota — 네임스페이스 단위의 규칙

- **LimitRange** — 파드(컨테이너) **하나당** 기본값과 한도. 자원을 안 적은 파드에 기본값을 채워 줍니다.
- **ResourceQuota** — 네임스페이스 **전체 합계**의 상한. CPU·메모리뿐 아니라 파드 개수 같은 오브젝트 수도 막습니다.

앞 절의 파드를 먼저 치워서 합계를 0 에서 시작합니다.

```bash
kubectl delete pod --all -n guide-sched
```

### LimitRange

`limitrange.yaml`:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: guide-defaults
  namespace: guide-sched
spec:
  limits:
    - type: Container
      defaultRequest:        # ← requests 를 안 적으면 이 값
        cpu: 50m
        memory: 64Mi
      default:               # ← limits 를 안 적으면 이 값
        cpu: 250m
        memory: 192Mi
      max:                   # ← 이보다 큰 limits 는 생성 거부
        cpu: 500m
        memory: 512Mi
```

```bash
kubectl apply -f limitrange.yaml
kubectl describe limitrange guide-defaults -n guide-sched
```

```text
Type        Resource  Min  Max    Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---  ---    ---------------  -------------  -----------------------
Container   cpu       -    500m   50m              250m           -
Container   memory    -    512Mi  64Mi             192Mi          -
```

자원을 **적지 않은** 파드를 만들어 봅니다.

```bash
kubectl run plain -n guide-sched --image=nginx:1.27
kubectl get pod plain -n guide-sched -o jsonpath='{.spec.containers[0].resources}{"\n"}{.status.qosClass}{"\n"}'
```

```text
{"limits":{"cpu":"250m","memory":"192Mi"},"requests":{"cpu":"50m","memory":"64Mi"}}
Burstable
```

**관찰 포인트** — 아무것도 안 적었는데 resources 가 채워졌고, BestEffort 가 아니라 Burstable 입니다. LimitRange 는 **파드가 생성되는 순간** 적용됩니다. 이미 떠 있던 파드는 바뀌지 않습니다.

### 실험 — max 를 넘는 파드

```bash
kubectl run too-big -n guide-sched --image=nginx:1.27 $do > too-big.yaml
```

`too-big.yaml` 의 컨테이너에 `resources: { limits: { cpu: "1" } }` 를 넣고 적용합니다.

```bash
kubectl apply -f too-big.yaml
```

```text
Error from server (Forbidden): error when creating "too-big.yaml": pods "too-big" is forbidden: maximum cpu usage per Container is 500m, but limit is 1
```

스케줄러까지 가지도 못하고 **API 서버 단계(어드미션)에서 거부**됩니다. 그래서 `Pending` 이 아니라 파드가 아예 안 생깁니다.

### ResourceQuota

`quota.yaml`:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: guide-quota
  namespace: guide-sched
spec:
  hard:
    pods: "3"                # ← 파드는 최대 3개
    requests.cpu: "1"        # ← requests 합계 1 코어
    requests.memory: 1Gi
    limits.cpu: "2"
    limits.memory: 2Gi
```

```bash
kubectl apply -f quota.yaml
kubectl create deployment quota-test -n guide-sched --image=nginx:1.27 --replicas=4
kubectl get deploy quota-test -n guide-sched
kubectl describe quota guide-quota -n guide-sched
```

```text
NAME         READY   UP-TO-DATE   AVAILABLE
quota-test   2/4     2            2

Resource         Used    Hard
--------         ----    ----
limits.cpu       750m    2
limits.memory    576Mi   2Gi
pods             3       3          ← plain 1개 + quota-test 2개
requests.cpu     150m    1
requests.memory  192Mi   1Gi
```

4개를 원했는데 2개만 떴습니다. 파드가 **Pending 도 아니고 아예 없습니다.** 이유는 ReplicaSet 의 이벤트에 있습니다.

```bash
kubectl describe rs -n guide-sched -l app=quota-test | grep FailedCreate | tail -1
```

```text
  Warning  FailedCreate  ...  Error creating: pods "quota-test-6c7d9f-xk2lp" is forbidden: exceeded quota: guide-quota, requested: pods=1, used: pods=3, limited: pods=3
```

**관찰 포인트** — 쿼터 초과는 파드 describe 가 아니라 **ReplicaSet 의 Events** 에 찍힙니다. Deployment 의 READY 가 모자라는데 Pending 파드가 없다면 `describe rs` 를 보세요.

### 실험 — LimitRange 없이 쿼터만 있으면

```bash
kubectl delete limitrange guide-defaults -n guide-sched
kubectl delete deployment quota-test -n guide-sched
kubectl run bare -n guide-sched --image=nginx:1.27
```

```text
Error from server (Forbidden): pods "bare" is forbidden: failed quota: guide-quota: must specify limits.cpu for: bare; limits.memory for: bare; requests.cpu for: bare; requests.memory for: bare
```

쿼터가 `requests.cpu` 등을 세고 있으면 **값을 안 적은 파드는 셀 수가 없어서 거부**됩니다. 그래서 "쿼터로 총량을 막고, LimitRange 로 기본값을 채운다"를 함께 쓰는 것이 정석입니다.

| | LimitRange | ResourceQuota |
|---|---|---|
| 범위 | 컨테이너/파드 **하나당** | 네임스페이스 **합계** |
| 주 용도 | 기본값 채우기 · 1개당 최대/최소 | 총량·개수 상한 |
| 위반 시 | 생성 거부 (Forbidden) | 생성 거부 (exceeded quota) |
| 확인 | `kubectl describe limitrange -n <ns>` | `kubectl describe quota -n <ns>` |

!!! tip "시험 팁 — 쿼터는 명령 한 줄로"
    ```bash
    kubectl create quota my-quota -n <ns> --hard=pods=5,requests.cpu=2,requests.memory=2Gi $do
    ```
    LimitRange 는 `create` 명령이 없습니다. kubernetes.io 에서 **"Configure Default Memory Requests and Limits for a Namespace"** 문서의 YAML 을 복사해 고치세요.

다음 절로 넘어가기 전에 쿼터와 파드를 지웁니다.

```bash
kubectl delete quota guide-quota -n guide-sched
kubectl delete pod --all -n guide-sched
```

---

## 5) nodeSelector 와 Node Affinity — 파드가 노드를 고른다

여기서부터는 **어디에 놓을지**입니다. 먼저 노드에 레이블을 붙입니다.

```bash
kubectl label node worker-1 disk=nvme
kubectl label node worker-2 disk=hdd
kubectl get nodes -L disk
```

```text
NAME            STATUS   ROLES           VERSION   DISK
control-plane   Ready    control-plane   v1.32.x
worker-1        Ready    <none>          v1.32.x   nvme
worker-2        Ready    <none>          v1.32.x   hdd
```

### nodeSelector — 가장 단순한 "딱 일치"

```bash
kubectl run on-nvme -n guide-sched --image=nginx:1.27 \
  --overrides='{"spec":{"nodeSelector":{"disk":"nvme"}}}'
kubectl get pod on-nvme -n guide-sched -o wide
```

```text
NAME      READY   STATUS    NODE
on-nvme   1/1     Running   worker-1
```

!!! note "`--overrides` 는 참고용입니다"
    한 줄로 보여 주려고 썼을 뿐, 시험에서는 `$do` 로 YAML 을 뽑고 `spec.nodeSelector` 를 적는 편이 실수가 적습니다.

### 실험 — 없는 레이블을 고르면

```bash
kubectl run on-optane -n guide-sched --image=nginx:1.27 \
  --overrides='{"spec":{"nodeSelector":{"disk":"optane"}}}'
kubectl describe pod on-optane -n guide-sched | tail -2
```

```text
  Warning  FailedScheduling  3s  default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 node(s) didn't match Pod's node affinity/selector. preemption: ...
```

**관찰 포인트** — nodeSelector 는 대안이 없습니다. 맞는 노드가 없으면 **영원히 Pending** 입니다. 이 파드에 `disk=optane` 레이블을 가진 노드가 생기면 그때 스케줄됩니다.

```bash
kubectl delete pod on-optane -n guide-sched
```

### Node Affinity — required 와 preferred

Node Affinity 는 nodeSelector 의 확장판입니다. **반드시(required)** 와 **가능하면(preferred)** 을 나눠 적을 수 있고 연산자도 여러 개입니다.

`affinity.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: must-nvme
  namespace: guide-sched
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:   # ← 하드 조건. 안 맞으면 Pending
        nodeSelectorTerms:
          - matchExpressions:
              - key: disk
                operator: In           # ← 목록 중 하나와 같으면
                values: ["nvme", "optane"]
  containers:
    - name: app
      image: nginx:1.27
---
apiVersion: v1
kind: Pod
metadata:
  name: prefer-optane
  namespace: guide-sched
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:  # ← 소프트 조건. 없으면 아무 데나
        - weight: 80                   # ← 1~100. 점수(Score) 단계에 더해진다
          preference:
            matchExpressions:
              - key: disk
                operator: In
                values: ["optane"]     # ← 이런 노드는 없다
  containers:
    - name: app
      image: nginx:1.27
```

```bash
kubectl apply -f affinity.yaml
kubectl get pods -n guide-sched -o wide
```

```text
NAME            READY   STATUS    NODE
must-nvme       1/1     Running   worker-1       ← nvme 는 worker-1 뿐
on-nvme         1/1     Running   worker-1
prefer-optane   1/1     Running   worker-2       ← optane 노드가 없어도 떴다 (노드는 아무 곳)
```

**관찰 포인트** — `prefer-optane` 은 조건에 맞는 노드가 하나도 없는데 Running 입니다. preferred 는 **거르기(Filter)가 아니라 점수(Score)** 에만 영향을 주기 때문입니다.

| 연산자 | 의미 | 예 |
|--------|------|----|
| `In` | 값 목록 중 하나와 같다 | `disk In [nvme, optane]` |
| `NotIn` | 값 목록에 없다 (키가 없는 노드도 통과) | `disk NotIn [hdd]` |
| `Exists` | 키가 있기만 하면 된다 (`values` 를 적지 않음) | `disk Exists` |
| `DoesNotExist` | 키가 없어야 한다 | `spot DoesNotExist` |
| `Gt` / `Lt` | 값을 정수로 비교 (노드 affinity 전용) | `cores Gt 8` |

!!! warning "NotIn 은 control-plane 도 통과시킵니다"
    `disk NotIn [hdd]` 는 `disk` 레이블이 **없는** control-plane 도 조건에 맞습니다. 그래도 control-plane 에 안 가는 것은 affinity 때문이 아니라 **taint 때문**입니다 (6) 참고).

### IgnoredDuringExecution — 이미 돌고 있는 파드는 그대로

```bash
kubectl label node worker-1 disk-          # 레이블 제거 (끝에 하이픈)
kubectl get pod must-nvme -n guide-sched -o wide
```

```text
NAME        READY   STATUS    NODE
must-nvme   1/1     Running   worker-1       ← 조건이 깨졌지만 쫓겨나지 않는다
```

이름 끝의 `IgnoredDuringExecution` 이 바로 이 뜻입니다. 조건은 **스케줄링 순간에만** 검사합니다. 레이블을 되돌리고 파드를 정리합니다.

```bash
kubectl label node worker-1 disk=nvme
kubectl delete pod --all -n guide-sched
```

!!! tip "시험 팁 — 긴 필드 이름을 외우지 않는다"
    - `kubectl explain pod.spec.affinity.nodeAffinity --recursive` 로 구조를 봅니다.
    - kubernetes.io 에서 **"Assign Pods to Nodes using Node Affinity"** 문서의 YAML 을 복사하면 들여쓰기 실수가 없습니다.
    - 문제가 "nodeSelector 말고 affinity 로" 라고 하면 **그 방식으로** 풀어야 채점을 통과합니다.

---

## 6) Taint 와 Toleration — 노드가 파드를 거부한다

Affinity 가 "파드가 노드를 고르는 것"이라면 Taint 는 **"노드가 파드를 거르는 것"** 입니다. 방향이 반대입니다.

### NoSchedule taint 걸기

```bash
kubectl taint node worker-2 team=batch:NoSchedule
kubectl describe node worker-2 | grep -A1 Taints
```

```text
Taints:             team=batch:NoSchedule
Unschedulable:      false
```

toleration 이 없는 Deployment 를 띄웁니다.

```bash
kubectl create deployment plain-web -n guide-sched --image=nginx:1.27 --replicas=4
kubectl get pods -n guide-sched -l app=plain-web -o wide
```

```text
NAME                         READY   STATUS    NODE
plain-web-7c9d8b6f5-2kq8x    1/1     Running   worker-1
plain-web-7c9d8b6f5-8mz4t    1/1     Running   worker-1
plain-web-7c9d8b6f5-lp9wn    1/1     Running   worker-1
plain-web-7c9d8b6f5-x7r2c    1/1     Running   worker-1
```

**관찰 포인트** — 4개가 전부 worker-1 입니다. worker-2 는 taint 로, control-plane 은 kubeadm 이 건 기본 taint 로 걸러졌습니다.

### toleration 을 넣으면

`batch-job.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-web
  namespace: guide-sched
spec:
  replicas: 4
  selector:
    matchLabels: { app: batch-web }
  template:
    metadata:
      labels: { app: batch-web }
    spec:
      tolerations:               # ← 파드 템플릿(spec.template.spec) 안이다
        - key: team
          operator: Equal        # ← key 와 value 가 모두 같아야 견딘다
          value: batch
          effect: NoSchedule     # ← taint 의 effect 와 같아야 한다
      containers:
        - name: nginx
          image: nginx:1.27
```

```bash
kubectl apply -f batch-job.yaml
kubectl get pods -n guide-sched -l app=batch-web -o wide
```

```text
NAME                         READY   STATUS    NODE
batch-web-5f8c7d9b4-4hq9z    1/1     Running   worker-2
batch-web-5f8c7d9b4-9wkx2    1/1     Running   worker-1     ← worker-1 에도 간다
batch-web-5f8c7d9b4-m2n8c    1/1     Running   worker-2
batch-web-5f8c7d9b4-t7m4x    1/1     Running   worker-1
```

**관찰 포인트** — toleration 이 있는 파드도 **worker-1 에 갑니다.** toleration 은 "그 노드에 **갈 수 있게**" 해 줄 뿐 "그 노드로 **보내지**" 않습니다. worker-2 전용으로 만들려면 nodeSelector/affinity 를 **함께** 씁니다.

```bash
kubectl explain pod.spec.tolerations.operator
```

```text
FIELD: operator <string>
ENUM:
    Equal
    Exists
DESCRIPTION:
    Operator represents a key's relationship to the value. Valid operators are
    Exists and Equal. Defaults to Equal. Exists is equivalent to wildcard for
    value, so that a pod can tolerate all taints of a particular category.
```

| toleration 모양 | 견디는 것 |
|-----------------|-----------|
| `key: team, operator: Equal, value: batch, effect: NoSchedule` | `team=batch:NoSchedule` 만 |
| `key: team, operator: Exists, effect: NoSchedule` | `team` 키면 value 무관 |
| `key: team, operator: Exists` (effect 생략) | `team` 키의 모든 effect |
| `operator: Exists` (key 도 생략) | **모든 taint** (kube-proxy·CNI DaemonSet 이 이렇게 한다) |

### control-plane 의 taint

```bash
kubectl describe node control-plane | grep -A1 Taints
kubectl get ds -n kube-system -o custom-columns=NAME:.metadata.name,TOL:.spec.template.spec.tolerations
```

```text
Taints:             node-role.kubernetes.io/control-plane:NoSchedule

NAME         TOL
cilium       [map[operator:Exists]]
kube-proxy   [map[operator:Exists]]
```

일반 파드가 worker 에만 몰리는 것은 **고장이 아니라 이 taint 때문**입니다. 시스템 DaemonSet 은 모든 taint 를 견디는 toleration 을 갖고 있어 세 노드에 다 뜹니다.

### 실험 — NoExecute 는 이미 돌고 있는 파드도 쫓아낸다

!!! warning "worker-2 의 다른 파드도 잠시 옮겨집니다"
    NoExecute 는 toleration 이 없는 **모든 파드**를 쫓아냅니다(coredns, metrics-server 포함). 자기 실습 클러스터에서만 하고, 끝나면 바로 taint 를 푸세요.

tolerationSeconds 를 준 파드와 안 준 파드를 worker-2 에 하나씩 띄웁니다.

`noexec.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: stay-40s
  namespace: guide-sched
spec:
  nodeSelector:
    kubernetes.io/hostname: worker-2   # ← 모든 노드에 기본으로 붙는 레이블
  tolerations:
    - { key: team, operator: Equal, value: batch, effect: NoSchedule }
    - key: maint
      operator: Exists
      effect: NoExecute
      tolerationSeconds: 40            # ← NoExecute 를 40초만 견딘다
  containers:
    - { name: app, image: nginx:1.27 }
---
apiVersion: v1
kind: Pod
metadata:
  name: leave-now
  namespace: guide-sched
spec:
  nodeSelector:
    kubernetes.io/hostname: worker-2
  tolerations:
    - { key: team, operator: Equal, value: batch, effect: NoSchedule }
  containers:
    - { name: app, image: nginx:1.27 }
```

```bash
kubectl apply -f noexec.yaml
kubectl get pods -n guide-sched -o wide | grep -E 'stay|leave'
kubectl taint node worker-2 maint=now:NoExecute
kubectl get pods -n guide-sched -w | grep -E 'stay|leave'     # 1분쯤 지켜본 뒤 Ctrl+C
```

```text
leave-now   1/1     Terminating   0     ← taint 를 걸자마자
stay-40s    1/1     Running       0
...
stay-40s    1/1     Terminating   0     ← 약 40초 뒤
```

batch-web 의 worker-2 파드들도 Terminating 되고, Deployment 가 worker-1 에 다시 만듭니다. taint 를 풉니다.

```bash
kubectl taint node worker-2 maint=now:NoExecute-
kubectl taint node worker-2 team=batch:NoSchedule-
kubectl describe node worker-2 | grep Taints
kubectl delete deployment plain-web batch-web -n guide-sched
kubectl delete pod --all -n guide-sched
```

```text
Taints:             <none>
```

| effect | 새 파드 | 이미 돌고 있는 파드 |
|--------|---------|---------------------|
| `NoSchedule` | 배치 안 됨 | 그대로 둔다 |
| `PreferNoSchedule` | 가능하면 피한다 | 그대로 둔다 |
| `NoExecute` | 배치 안 됨 | **쫓아낸다** (tolerationSeconds 만큼 유예) |

!!! note "노드가 NotReady 가 되면"
    쿠버네티스가 자동으로 `node.kubernetes.io/not-ready:NoExecute` taint 를 겁니다. 파드에는 기본으로 `tolerationSeconds: 300` 이 붙어 있어서 **약 5분 뒤** 다른 노드로 옮겨집니다.
    `kubectl get pod <아무 파드> -o jsonpath='{.spec.tolerations}'` 로 확인해 보세요.

---

## 7) Pod Affinity 와 Anti-Affinity — 다른 파드를 기준으로

노드 레이블이 아니라 **이미 돌고 있는 파드의 레이블**을 기준으로 배치합니다. 그래서 `labelSelector` 와, 무엇을 "같은 곳"으로 볼지 정하는 `topologyKey` 가 필요합니다.

### podAffinity — 옆에 붙인다

캐시 역할 파드를 하나 띄우고, 앱 파드들을 **그 파드가 있는 노드**에 붙입니다.

```bash
kubectl run cache -n guide-sched --image=redis:7 --labels=role=cache
kubectl get pod cache -n guide-sched -o wide
```

`near-cache.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: near-cache
  namespace: guide-sched
spec:
  replicas: 3
  selector:
    matchLabels: { app: near-cache }
  template:
    metadata:
      labels: { app: near-cache }
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  role: cache                        # ← 이 레이블의 파드가 있는
              topologyKey: kubernetes.io/hostname    # ← "같은 노드"에 둔다
      containers:
        - name: nginx
          image: nginx:1.27
```

```bash
kubectl apply -f near-cache.yaml
kubectl get pods -n guide-sched -o wide
```

```text
NAME                          READY   STATUS    NODE
cache                         1/1     Running   worker-2
near-cache-6b9f7c8d5-4k2lp    1/1     Running   worker-2
near-cache-6b9f7c8d5-8xq7m    1/1     Running   worker-2
near-cache-6b9f7c8d5-rz5tn    1/1     Running   worker-2
```

### podAntiAffinity — 떨어뜨린다

복제본이 한 노드에 몰리면 그 노드가 죽을 때 전부 죽습니다. 실무에서는 anti-affinity 를 더 많이 씁니다.

`spread.yaml`:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread
  namespace: guide-sched
spec:
  replicas: 3                  # ← 워커는 2대뿐이다
  selector:
    matchLabels: { app: spread }
  template:
    metadata:
      labels: { app: spread }
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchLabels:
                  app: spread                        # ← 자기 자신과 같은 레이블의 파드와
              topologyKey: kubernetes.io/hostname    # ← 같은 노드에 두지 않는다
      containers:
        - name: nginx
          image: nginx:1.27
```

```bash
kubectl apply -f spread.yaml
kubectl get pods -n guide-sched -l app=spread -o wide
kubectl describe pod -n guide-sched -l app=spread | grep FailedScheduling
```

```text
NAME                      READY   STATUS    NODE
spread-7d4c9b8f6-6wm2k    1/1     Running   worker-1
spread-7d4c9b8f6-h5x9q    1/1     Running   worker-2
spread-7d4c9b8f6-v8n3r    0/1     Pending   <none>

  Warning  FailedScheduling  ...  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 node(s) didn't match pod anti-affinity rules. preemption: ...
```

**관찰 포인트** — 워커 2대에 하나씩 놓고 나니 세 번째는 갈 곳이 없습니다. required anti-affinity 는 "노드 수 = 최대 복제본 수"가 됩니다.
`requiredDuring…` 을 `preferredDuring…`(weight + podAffinityTerm 구조)로 바꾸면 세 번째도 어딘가에 뜹니다.

```bash
kubectl explain pod.spec.affinity.podAntiAffinity.preferredDuringSchedulingIgnoredDuringExecution
```

```text
FIELDS:
  podAffinityTerm       <PodAffinityTerm> -required-
  weight        <integer> -required-
```

| topologyKey | "같은 곳"의 단위 |
|-------------|------------------|
| `kubernetes.io/hostname` | 같은 노드 |
| `topology.kubernetes.io/zone` | 같은 가용 영역 (클라우드) |

!!! warning "topologyKey 는 필수입니다"
    required pod (anti-)affinity 에서 `topologyKey` 를 빠뜨리면 API 서버가 생성을 거부합니다. 대부분 `kubernetes.io/hostname` 을 씁니다.

```bash
kubectl delete deployment near-cache spread -n guide-sched
kubectl delete pod cache -n guide-sched
```

### 스케줄러를 거치지 않는 두 가지 — nodeName 과 Static Pod

지금까지는 모두 스케줄러의 거르기 단계 이야기였습니다. **스케줄러를 아예 건너뛰는** 방법도 있습니다.

=== "nodeName"
    `spec.nodeName` 을 직접 적으면 스케줄러가 관여하지 않고 그 노드의 kubelet 이 바로 받습니다.
    ```bash
    kubectl run direct -n guide-sched --image=nginx:1.27 \
      --overrides='{"spec":{"nodeName":"control-plane"}}'
    kubectl get pod direct -n guide-sched -o wide
    ```
    ```text
    NAME     READY   STATUS    NODE
    direct   1/1     Running   control-plane     ← NoSchedule taint 가 있는데도 떴다
    ```
    NoSchedule 은 **스케줄러가 보는** 규칙이라 nodeName 앞에서는 힘이 없습니다. 실무에서는 쓰지 않습니다.
    ```bash
    kubectl delete pod direct -n guide-sched
    ```
=== "Static Pod"
    kubelet 이 감시하는 디렉터리에 파일을 두면 API 서버·스케줄러 없이 kubelet 이 바로 띄웁니다(3강 아키텍처 참고). control-plane 노드에서 합니다.
    ```bash
    sudo grep staticPodPath /var/lib/kubelet/config.yaml
    kubectl run guide-static -n guide-sched --image=nginx:1.27 $do | sudo tee /etc/kubernetes/manifests/guide-static.yaml > /dev/null
    kubectl get pods -n guide-sched -o wide
    ```
    ```text
    staticPodPath: /etc/kubernetes/manifests

    NAME                         READY   STATUS    NODE
    guide-static-control-plane   1/1     Running   control-plane    ← 이름 끝에 노드 이름
    ```
    다른 노드에 만들라는 문제라면 `ssh worker-1` 로 들어가 **그 노드의** staticPodPath 에 파일을 둡니다. 없애려면 파일을 지웁니다.
    ```bash
    sudo rm /etc/kubernetes/manifests/guide-static.yaml
    ```

---

## 8) HPA — 부하에 따라 파드 수를 조절한다

HPA 는 **metrics-server 가 모은 사용량**을 **requests 대비 백분율**로 계산해 목표와 비교하고 Deployment 의 replicas 를 바꿉니다.
그래서 전제가 둘입니다. **metrics-server** 가 있어야 하고(0) 에서 설치), **대상 파드에 requests** 가 있어야 합니다.

### 대상 Deployment

공식 문서 HPA 예제에 쓰는 CPU 소모용 이미지를 씁니다.

```bash
kubectl create deployment php-apache -n guide-sched --image=registry.k8s.io/hpa-example --port=80
kubectl set resources deployment php-apache -n guide-sched --requests=cpu=250m --limits=cpu=500m
kubectl expose deployment php-apache -n guide-sched --port=80
kubectl rollout status deployment/php-apache -n guide-sched
```

!!! tip "시험 팁 — `kubectl set resources`"
    Deployment 에 자원을 넣을 때는 YAML 을 열지 않고 `kubectl set resources` 한 줄로 끝낼 수 있습니다. `-c <컨테이너>` 로 대상 컨테이너를 고를 수 있습니다.
    파드(Pod) 는 resources 를 나중에 바꿀 수 없으니 `$do` 로 YAML 을 뽑아 처음부터 넣으세요.

### HPA 만들기 — 명령 한 줄

```bash
kubectl autoscale deployment php-apache -n guide-sched --min=1 --max=4 --cpu-percent=40
kubectl get hpa -n guide-sched
```

```text
NAME         REFERENCE               TARGETS       MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   cpu: 0%/40%   1         4         1          30s
```

처음 30초쯤은 `cpu: <unknown>/40%` 로 보이다가 숫자로 바뀝니다.

### 부하 걸기

=== "터미널 1 — 지켜보기"
    ```bash
    kubectl get hpa php-apache -n guide-sched -w
    ```
=== "터미널 2 — 부하 발생"
    ```bash
    kubectl run load -n guide-sched --image=busybox:1.36 --restart=Never -- \
      /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
    ```

```text
NAME         REFERENCE               TARGETS         MINPODS   MAXPODS   REPLICAS
php-apache   Deployment/php-apache   cpu: 0%/40%     1         4         1
php-apache   Deployment/php-apache   cpu: 196%/40%   1         4         1
php-apache   Deployment/php-apache   cpu: 196%/40%   1         4         4
php-apache   Deployment/php-apache   cpu: 55%/40%    1         4         4
```

**관찰 포인트** — 사용률이 목표를 넘자 REPLICAS 가 늘어났고, **max(4) 에서 멈췄습니다.** 196% 는 "requests(250m) 의 약 2배를 쓰고 있다"는 뜻입니다. requests 가 계산의 분모입니다.

부하를 멈춥니다.

```bash
kubectl delete pod load -n guide-sched
```

줄어드는 것은 바로 일어나지 않습니다. 기본 **scale-down 안정화 시간이 5분**이라 5분쯤 지나야 1 로 돌아옵니다(출렁임 방지).

### 실험 — requests 가 없으면

```bash
kubectl create deployment no-req -n guide-sched --image=nginx:1.27
kubectl autoscale deployment no-req -n guide-sched --min=1 --max=3 --cpu-percent=40
kubectl get hpa no-req -n guide-sched
kubectl describe hpa no-req -n guide-sched | tail -3
```

```text
NAME     REFERENCE           TARGETS              MINPODS   MAXPODS   REPLICAS
no-req   Deployment/no-req   cpu: <unknown>/40%   1         3         1

  Warning  FailedGetResourceMetric  ...  failed to get cpu utilization: missing request for cpu in container nginx of Pod no-req-5d8f9c7b6-kx2wq
```

**관찰 포인트** — metrics-server 는 멀쩡한데 `<unknown>` 입니다. 분모인 **requests 가 없어서** 백분율을 못 구합니다. `<unknown>` 을 보면 ① metrics-server ② requests 순서로 확인하세요.

### YAML 로 보기 — autoscaling/v2

메모리 기준이나 여러 지표를 쓰려면 YAML 을 씁니다. 구조는 `explain` 으로 확인합니다.

```bash
kubectl explain hpa.spec.metrics.resource.target --api-version=autoscaling/v2
```

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
  namespace: guide-sched
spec:
  scaleTargetRef:              # ← 무엇을 늘리고 줄일지
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization    # ← requests 대비 %
          averageUtilization: 40
```

!!! note "`autoscale ... $do` 의 출력"
    `kubectl autoscale` 에 `$do` 를 붙이면 kubectl 버전에 따라 `autoscaling/v1`(`targetCPUUtilizationPercentage`) 또는 `v2` 형식으로 나옵니다. 어느 쪽이든 CPU 기준 HPA 로는 똑같이 동작합니다.

```bash
kubectl delete hpa php-apache no-req -n guide-sched
kubectl delete deployment php-apache no-req -n guide-sched
kubectl delete service php-apache -n guide-sched
```

---

## 9) PriorityClass — 자리가 없을 때 누가 밀려나나

노드에 자리가 없으면 파드는 보통 `Pending` 으로 기다립니다. 그런데 **우선순위가 더 높은** 파드라면 스케줄러가 낮은 파드를 쫓아내고(preemption) 자리를 만듭니다. `value` 가 클수록 높습니다.

### 기존 값부터 본다

```bash
kubectl get priorityclass
```

```text
NAME                      VALUE        GLOBAL-DEFAULT   AGE
system-cluster-critical   2000000000   false            3d
system-node-critical      2000001000   false            3d
```

"기존 것보다 높게/낮게" 같은 조건이 나오면 **항상 이 목록부터** 봅니다. `system-*` 두 개는 쿠버네티스 예약용이라 직접 쓰지 않습니다.

### PriorityClass 두 개 만들기

```bash
kubectl create priorityclass guide-low  --value=1000   --description="가이드 실습용 낮은 우선순위"
kubectl create priorityclass guide-high --value=100000 --description="가이드 실습용 높은 우선순위"
kubectl get priorityclass guide-low guide-high
```

```text
NAME         VALUE    GLOBAL-DEFAULT   AGE
guide-high   100000   false            3s
guide-low    1000     false            4s
```

### worker-1 을 낮은 우선순위 파드로 채운다

파드 하나가 CPU 500m 를 요청하고 모두 worker-1 에만 가도록 합니다. 워커가 2코어라면 2~3개만 뜨고 나머지는 Pending 이 됩니다.

```bash
kubectl create deployment filler -n guide-sched --image=nginx:1.27 --replicas=8 $do > filler.yaml
```

`filler.yaml` 의 `spec.template.spec` 을 다음처럼 고칩니다.

```yaml
    spec:
      priorityClassName: guide-low         # ← 파드 템플릿 안. spec 바로 아래가 아니다
      nodeSelector:
        kubernetes.io/hostname: worker-1   # ← worker-1 에만
      containers:
        - name: nginx
          image: nginx:1.27
          resources:
            requests:
              cpu: 500m                    # ← 노드를 빨리 채우려고 크게
```

```bash
kubectl apply -f filler.yaml
kubectl get pods -n guide-sched -l app=filler -o wide
```

```text
NAME                      READY   STATUS    NODE
filler-6d9c8b7f5-2xk8p    1/1     Running   worker-1
filler-6d9c8b7f5-7wq4k    1/1     Running   worker-1
filler-6d9c8b7f5-9hz2m    1/1     Running   worker-1
filler-6d9c8b7f5-c4n7t    0/1     Pending   <none>
...                                         ← 나머지 Pending
```

일부가 Pending 이면 준비 완료입니다. (모두 Running 이면 워커 CPU 가 큰 것이니 `--replicas` 를 늘리세요.)
Pending 인 filler 들은 서로 우선순위가 같아서 **아무도 쫓아내지 못하고** 기다립니다.

### 높은 우선순위 파드를 넣는다

`urgent.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: urgent
  namespace: guide-sched
spec:
  priorityClassName: guide-high          # ← 100000 > 1000
  nodeSelector:
    kubernetes.io/hostname: worker-1
  containers:
    - name: app
      image: nginx:1.27
      resources:
        requests:
          cpu: 500m
```

```bash
kubectl apply -f urgent.yaml
kubectl get pods -n guide-sched -o wide -w      # 30초쯤 지켜본 뒤 Ctrl+C
```

```text
NAME                      READY   STATUS        NODE
urgent                    0/1     Pending       <none>
filler-6d9c8b7f5-9hz2m    1/1     Terminating   worker-1     ← 낮은 파드가 밀려난다
urgent                    0/1     ContainerCreating   worker-1
urgent                    1/1     Running       worker-1
filler-6d9c8b7f5-k8v2d    0/1     Pending       <none>       ← ReplicaSet 이 다시 만들지만 자리가 없다
```

```bash
kubectl get pod urgent -n guide-sched -o jsonpath='{.spec.priority}{"\n"}'
kubectl get events -n guide-sched --field-selector reason=Preempted
```

```text
100000
LAST SEEN   TYPE     REASON      OBJECT                        MESSAGE
20s         Normal   Preempted   pod/filler-6d9c8b7f5-9hz2m    Preempted by pod 3f1c9a2e-... on node worker-1
```

**관찰 포인트** — `priorityClassName` 은 파드가 생성될 때 **숫자(`spec.priority`)로 변환**되어 박힙니다. 같은 우선순위끼리는 기다리기만 하고, 더 높은 파드만 낮은 파드를 밀어냅니다.

### 기존 Deployment 에 PriorityClass 붙이기 — 지우지 않고

이미 돌고 있는 Deployment 에 붙일 때는 `patch` 한 줄이면 됩니다. 붙이는 자리는 **`spec.template.spec`** 입니다.

```bash
kubectl create deployment steady -n guide-sched --image=nginx:1.27 --replicas=2
kubectl patch deployment steady -n guide-sched \
  -p '{"spec":{"template":{"spec":{"priorityClassName":"guide-low"}}}}'
kubectl rollout status deployment/steady -n guide-sched
kubectl get deployment steady -n guide-sched \
  -o jsonpath='{.spec.template.spec.priorityClassName} rev={.metadata.annotations.deployment\.kubernetes\.io/revision}{"\n"}'
```

```text
guide-low rev=2
```

**관찰 포인트** — 템플릿이 바뀌었으니 롤링 업데이트가 일어나 revision 이 2 가 되었습니다. 지우고 다시 만들었다면 1 이었을 것입니다.

!!! warning "자주 하는 실수"
    - `spec.priorityClassName` (Deployment 의 spec) 에 넣으면 **알 수 없는 필드**라 무시되거나 거부됩니다. 파드 속성이므로 `spec.template.spec` 입니다.
    - 없는 PriorityClass 이름을 쓰면 ReplicaSet 이벤트에 `no PriorityClass with name ... was found` 가 찍히고 파드가 생성되지 않습니다.

| 필드 | 뜻 |
|------|----|
| `value` | 클수록 높다. 사용자 정의는 10억 이하 |
| `globalDefault: true` | PriorityClass 를 안 적은 파드의 기본값 (클러스터에 하나만) |
| `preemptionPolicy: Never` | 높아도 남을 쫓아내지 않는다 (대기열 앞자리만 차지) |
| `system-cluster-critical` 등 | 20억대, 시스템 예약 |

```bash
kubectl delete deployment filler steady -n guide-sched
kubectl delete pod urgent -n guide-sched
```

---

## 시험에서 빠르게

```bash
# 자원 — 파드는 YAML, Deployment 는 set resources
kubectl run p --image=nginx $do > p.yaml                         # resources 는 손으로
kubectl set resources deploy/d --requests=cpu=150m,memory=64Mi --limits=cpu=400m
kubectl explain pod.spec.containers.resources
kubectl get pod p -o jsonpath='{.status.qosClass}'
kubectl create quota q -n ns --hard=pods=5,requests.cpu=2 $do

# 노드 레이블 · taint (끝에 하이픈 = 제거)
kubectl label node worker-1 key=value        ;  kubectl label node worker-1 key-
kubectl taint node worker-2 key=value:NoSchedule ; kubectl taint node worker-2 key=value:NoSchedule-
kubectl get nodes -L key                     # 레이블을 칸으로
kubectl describe node worker-2 | grep Taints

# affinity · toleration 필드 구조
kubectl explain pod.spec.affinity.nodeAffinity --recursive | less
kubectl explain pod.spec.tolerations

# 확인 — 어느 노드에 떴나, 왜 Pending 인가
kubectl get pod -o wide
kubectl describe pod p | tail -5
kubectl get events --field-selector reason=FailedScheduling

# HPA
kubectl autoscale deploy d --min=1 --max=5 --cpu-percent=70
kubectl get hpa

# PriorityClass
kubectl get pc                               # 기존 값 먼저
kubectl create pc name --value=N --description="..."
kubectl patch deploy d -p '{"spec":{"template":{"spec":{"priorityClassName":"name"}}}}'
```

!!! tip "kubernetes.io 에서 복사할 문서 제목"
    - **Assign Pods to Nodes using Node Affinity** — nodeAffinity YAML
    - **Taints and Tolerations** — tolerations YAML
    - **Assigning Pods to Nodes** — podAffinity / podAntiAffinity 예제
    - **HorizontalPodAutoscaler Walkthrough** — HPA v2 YAML
    - **Pod Priority and Preemption** — PriorityClass YAML

---

## 정리

네임스페이스와 함께 **클러스터 범위에 남긴 것**을 모두 되돌립니다.

```bash
kubectl delete namespace guide-sched

# 노드 레이블 · taint 원복 (이미 지웠다면 "not found" 가 나와도 괜찮다)
kubectl label node worker-1 disk-
kubectl label node worker-2 disk-
kubectl taint node worker-2 team=batch:NoSchedule-
kubectl taint node worker-2 maint=now:NoExecute-

# 클러스터 범위 오브젝트
kubectl delete priorityclass guide-low guide-high

# static pod 파일 (7) 에서 지우지 않았다면)
sudo rm -f /etc/kubernetes/manifests/guide-static.yaml

# 로컬 파일
rm -f greedy.yaml sized.yaml oom.yaml qos.yaml limitrange.yaml too-big.yaml quota.yaml \
      affinity.yaml batch-job.yaml noexec.yaml near-cache.yaml spread.yaml filler.yaml urgent.yaml
```

확인합니다.

```bash
kubectl get nodes -L disk
kubectl describe nodes | grep Taints
```

```text
NAME            STATUS   ROLES           VERSION   DISK
control-plane   Ready    control-plane   v1.32.x
worker-1        Ready    <none>          v1.32.x
worker-2        Ready    <none>          v1.32.x

Taints:             node-role.kubernetes.io/control-plane:NoSchedule
Taints:             <none>
Taints:             <none>
```

metrics-server 는 다른 실습(19강 트러블슈팅의 `kubectl top` 등)에서도 쓰므로 **남겨 둡니다.**

---

## 확인 질문

??? question "1. 노드 CPU 사용률은 10% 인데 파드가 `Insufficient cpu` 로 Pending 입니다. 어떻게 가능한가요?"
    스케줄러는 실제 사용량이 아니라 **requests 합**을 봅니다. 다른 파드들이 requests 로 CPU 를 예약만 해 두고 실제로는 안 쓰는 상태입니다.
    `kubectl describe node <노드>` 의 **Allocated resources** 에서 Requests 합계를 확인합니다.

??? question "2. 컨테이너가 계속 재시작하는데 로그에 에러가 없습니다. 무엇부터 보나요?"
    `kubectl describe pod` 의 **Last State** 입니다. `Reason: OOMKilled`, `Exit Code: 137` 이면 메모리 limit 초과입니다.
    CPU limit 초과는 죽지 않고 느려질 뿐이라 재시작 원인이 되지 않습니다.

??? question "3. limits 만 적은 파드의 QoS 등급은 무엇인가요?"
    Guaranteed 입니다(모든 컨테이너에 cpu·memory limits 를 모두 적은 경우). requests 가 생략되면 limits 값으로 채워져서 requests == limits 가 되기 때문입니다.

??? question "4. ResourceQuota 가 걸린 네임스페이스에서 Deployment 의 READY 가 2/4 인데 Pending 파드가 없습니다. 원인은 어디서 보나요?"
    쿼터 초과로 **파드 생성 자체가 거부**된 것입니다. 파드가 없으니 파드 describe 로는 안 보이고 `kubectl describe rs` 의 `FailedCreate ... exceeded quota` 이벤트에 나옵니다.

??? question "5. toleration 을 넣은 파드가 taint 가 걸린 노드가 아닌 다른 노드에 떴습니다. 잘못한 건가요?"
    아닙니다. toleration 은 그 노드에 **갈 수 있게** 해 줄 뿐 보내지는 않습니다. 특정 노드에 반드시 두려면 nodeSelector 나 nodeAffinity 를 함께 씁니다.

??? question "6. required Node Affinity 로 배치된 파드가 도는 중에 노드 레이블을 지우면 어떻게 되나요?"
    아무 일도 일어나지 않습니다. `IgnoredDuringExecution` 이라서 조건은 스케줄링 시점에만 검사합니다. 파드를 쫓아내고 싶다면 NoExecute taint 를 씁니다.

??? question "7. HPA 의 TARGETS 가 `<unknown>` 입니다. 확인 순서는?"
    ① metrics-server 가 동작하는지 (`kubectl top pods`) ② 대상 파드에 **cpu requests** 가 있는지 (`kubectl describe hpa` 의 `missing request for cpu`). 생성 직후 30초 정도는 정상적으로 `<unknown>` 일 수 있습니다.

??? question "8. 기존 Deployment 에 PriorityClass 를 붙이는 필드 경로는? 붙이면 파드는 어떻게 되나요?"
    `spec.template.spec.priorityClassName` 입니다. 파드 템플릿이 바뀌므로 롤링 업데이트로 파드가 새로 만들어지고 revision 이 하나 올라갑니다.

---

## 다음 단계 — 채점 실습

개념이 잡혔으면 시험 형식으로 풀어 봅니다. 이번에는 **답이 없습니다.** 세트가 노드에 레이블·taint 를 미리 걸어 두므로 문제 설명을 먼저 읽으세요.

```bash
cd ~/cka-lab/07-scheduling
bash exam.sh start
```

| 문제 | 이 페이지에서 연습한 것 |
|------|------------------------|
| Q1 Requests / Limits 가 설정된 파드 | 2) — `$do` 로 뽑고 resources 추가, `explain` |
| Q2 Node Affinity 로 파드 배치 | 5) — required · `In` · `get pod -o wide` 로 확인 |
| Q3 Taint 를 허용하는 Toleration | 6) — Equal toleration, "허용일 뿐" |
| Q4 HPA (자동 스케일) | 8) — `kubectl autoscale`, requests 전제 |
| Q5 PriorityClass 와 우선순위 | 9) — 기존 값 먼저 보기, 템플릿에 `patch` |
