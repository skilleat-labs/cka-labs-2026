# 8강 · 스토리지

> 영상: **08강-스토리지** · 이어서 풀 채점 세트: `08-storage`

## 실습 목표

- 컨테이너 안에 쓴 파일이 **재시작 한 번에 사라지는 것**과, 볼륨이 그걸 막는 범위(emptyDir vs PV)를 비교한다.
- `volumes` + `volumeMounts` 두 블록 구조로 **같은 파드 안 컨테이너끼리** 파일을 주고받는다.
- PV 와 PVC 를 직접 만들어 **Bound** 시키고, 일부러 조건을 어겨 **Pending 의 이유를 Events 로 읽는다**.
- reclaimPolicy `Retain` 에서 PVC 를 지운 뒤 **Released → Available** 로 되살리는 과정을 해 본다.
- StorageClass 의 `volumeBindingMode` 가 **언제, 어느 노드의 PV 에** 바인딩하는지를 바꾼다는 것을 확인한다.
- StatefulSet 의 `volumeClaimTemplates` 가 **파드마다 PVC 를 찍어 내고**, 파드가 다시 떠도 같은 PVC 를 쓰는 것을 확인한다.

---

## 0) 준비

이 페이지의 리소스는 `guide-storage` 네임스페이스에 만듭니다. PV 와 StorageClass 는 **클러스터 범위**라서 네임스페이스를 지워도 남습니다. 마지막 **정리** 단계를 꼭 진행하세요.

```bash
kubectl create namespace guide-storage
kubectl get storageclass
```

```text
No resources found
```

**StorageClass 가 하나도 없습니다.** kubeadm 으로 만든 실습 클러스터에는 동적 프로비저너가 없어서 PVC 를 만들어도 디스크가 저절로 생기지 않습니다.
그래서 이 페이지는 강의와 같이 **PV 를 직접 만드는 정적 방식**으로 진행합니다. 실제 저장 위치는 각 워커 노드의 `/srv/guide-storage/` 아래 디렉터리(hostPath)입니다.

!!! note "클라우드·관리형 클러스터라면"
    `kubectl get sc` 에 `(default)` 표시가 붙은 클래스가 이미 있을 수 있습니다. 그러면 PVC 에 클래스를 안 적어도 그 클래스가 자동으로 들어가서
    이 페이지의 4)~5) 결과가 달라집니다. 이 가이드는 **기본 StorageClass 가 없는** 실습 클러스터를 전제로 합니다.

---

## 1) 컨테이너 안에 쓴 데이터는 사라진다

컨테이너 이미지는 읽기 전용이고, 그 위에 **컨테이너마다 얇은 쓰기 층**이 얹힙니다. 컨테이너가 다시 만들어지면 그 층도 새로 만들어집니다.

### 볼륨 없이

```bash
kubectl run scratchpad -n guide-storage --image=nginx:1.27
kubectl exec scratchpad -n guide-storage -- sh -c 'echo "memo-1" > /tmp/note.txt; cat /tmp/note.txt'
```

```text
memo-1
```

컨테이너의 주 프로세스(nginx)를 멈춰서 **컨테이너만** 재시작시킵니다. 파드는 그대로입니다.

```bash
kubectl exec scratchpad -n guide-storage -- nginx -s stop
kubectl get pod scratchpad -n guide-storage
kubectl exec scratchpad -n guide-storage -- cat /tmp/note.txt
```

```text
NAME         READY   STATUS    RESTARTS      AGE
scratchpad   1/1     Running   1 (4s ago)    40s

cat: /tmp/note.txt: No such file or directory
```

**관찰 포인트** — 파드 이름도, IP 도, 노드도 그대로인데 `RESTARTS` 가 1 이 되면서 파일이 사라졌습니다. 로그든 DB 파일이든 컨테이너 파일시스템에 쓴 것은 **컨테이너 수명**과 같습니다.

### emptyDir 을 붙이면

`scratchpad-vol.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: scratchpad-vol
  namespace: guide-storage
spec:
  volumes:
    - name: work               # ← ① 파드에 볼륨을 붙이고
      emptyDir: {}
  containers:
    - name: web
      image: nginx:1.27
      volumeMounts:
        - name: work           # ← ② 같은 이름으로 컨테이너 경로에 단다
          mountPath: /work
```

```bash
kubectl apply -f scratchpad-vol.yaml
kubectl exec scratchpad-vol -n guide-storage -- sh -c 'echo "memo-2" > /work/note.txt'
kubectl exec scratchpad-vol -n guide-storage -- nginx -s stop
sleep 5
kubectl exec scratchpad-vol -n guide-storage -- cat /work/note.txt
```

```text
memo-2          ← 컨테이너가 재시작했는데 남아 있다
```

이번에는 **파드를 지우고** 같은 YAML 로 다시 만듭니다.

```bash
kubectl delete pod scratchpad-vol -n guide-storage
kubectl apply -f scratchpad-vol.yaml
kubectl exec scratchpad-vol -n guide-storage -- ls /work
```

```text
(아무것도 없다)
```

| 무엇이 없어지면 | 컨테이너 파일시스템 | emptyDir | PV (4) 부터) |
|-----------------|--------------------|----------|--------------|
| 컨테이너 재시작 | 사라진다 | **남는다** | 남는다 |
| 파드 삭제 | 사라진다 | 사라진다 | **남는다** |

```bash
kubectl delete pod scratchpad scratchpad-vol -n guide-storage
```

---

## 2) 볼륨의 기본 구조 — emptyDir 로 컨테이너끼리 공유하기

볼륨은 종류가 무엇이든 **두 블록**입니다. `spec.volumes` 에 **무엇을** 붙일지, 각 컨테이너의 `volumeMounts` 에 **어디에** 달지 적고, 둘을 `name` 으로 잇습니다.
같은 볼륨을 **컨테이너마다 다른 경로**에 달 수도 있습니다.

`relay.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: relay
  namespace: guide-storage
spec:
  volumes:
    - name: outbox                     # ← 볼륨 이름 (둘을 잇는 열쇠)
      emptyDir:
        sizeLimit: 50Mi                # ← 넘으면 kubelet 이 파드를 쫓아낸다
  containers:
    - name: producer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do echo \"<p>$(date)</p>\" > /out/index.html; sleep 3; done"]
      volumeMounts:
        - name: outbox
          mountPath: /out              # ← producer 는 /out 으로 본다
    - name: consumer
      image: nginx:1.27
      volumeMounts:
        - name: outbox
          mountPath: /usr/share/nginx/html   # ← consumer 는 nginx 문서 경로로 본다
          readOnly: true               # ← 읽기만 허용
```

```bash
kubectl apply -f relay.yaml
kubectl get pod relay -n guide-storage
kubectl exec relay -n guide-storage -c consumer -- cat /usr/share/nginx/html/index.html
```

```text
NAME    READY   STATUS    RESTARTS   AGE
relay   2/2     Running   0          10s

<p>Mon Sep 29 07:21:12 UTC 2026</p>
```

nginx 가 실제로 그 파일을 서비스하는지 파드 IP 로 확인합니다.

```bash
POD_IP=$(kubectl get pod relay -n guide-storage -o jsonpath='{.status.podIP}')
kubectl run probe -n guide-storage --rm -it --image=busybox:1.36 --restart=Never -- wget -qO- http://$POD_IP
```

```text
<p>Mon Sep 29 07:21:18 UTC 2026</p>
pod "probe" deleted
```

**관찰 포인트** — producer 는 `/out` 에 쓰고 consumer 는 `/usr/share/nginx/html` 에서 읽는데 **같은 디렉터리**입니다. 경로가 아니라 **볼륨 이름**이 같은 것이 중요합니다.

### 실험 — readOnly 마운트에 쓰기

```bash
kubectl exec relay -n guide-storage -c consumer -- sh -c 'echo hack > /usr/share/nginx/html/index.html'
```

```text
sh: 1: cannot create /usr/share/nginx/html/index.html: Read-only file system
command terminated with exit code 2
```

같은 볼륨이라도 **마운트마다** 읽기 전용을 따로 줄 수 있습니다.

### 실험 — 이름이 어긋나면

파드 이름만 바꾼 복사본을 만들고, 편집기로 **consumer** 의 `volumeMounts` 에 있는 `name: outbox` 를 `outbx` 로 고친 뒤 적용합니다.

```bash
sed 's/name: relay$/name: relay-typo/' relay.yaml > relay-typo.yaml
vi relay-typo.yaml          # consumer 의 volumeMounts → name: outbx
kubectl apply -f relay-typo.yaml
```

```text
The Pod "relay-typo" is invalid: spec.containers[1].volumeMounts[0].name: Not found: "outbx"
```

스케줄러까지 가지도 않고 API 서버가 **생성 자체를 거부**합니다. 파드가 Pending 조차 되지 않습니다.

```bash
kubectl explain pod.spec.volumes.emptyDir
```

```text
FIELDS:
  medium        <string>
    medium represents what type of storage medium should back this directory.
    The default is "" which means to use the node's default medium. Must be an
    empty string (default) or Memory. ...
  sizeLimit     <Quantity>
    sizeLimit is the total amount of local storage required for this EmptyDir
    volume. ...
```

`medium: Memory` 로 두면 노드 디스크 대신 **메모리(tmpfs)** 를 쓰고, 그 용량은 컨테이너 메모리 사용량으로 계산됩니다.

!!! tip "시험 팁 — 멀티 컨테이너 파드"
    `kubectl run x --image=busybox:1.36 $do -- sh -c '...' > x.yaml` 로 첫 컨테이너를 뽑고,
    `containers` 아래 블록을 **복사해서 이름·명령만** 바꾸는 것이 가장 빠릅니다. `volumes` 는 `containers` 와 같은 들여쓰기(`spec` 바로 아래)입니다.

```bash
kubectl delete pod relay -n guide-storage
```

---

## 3) hostPath — 노드의 디렉터리를 그대로 붙인다

hostPath 는 파드가 **뜬 그 노드**의 파일·디렉터리를 붙입니다. 노드 자체를 봐야 하는 DaemonSet(로그 수집기 등)에 쓰고, 이 실습에서는 PV 의 실제 저장 위치로 씁니다.

### 노드의 파일 읽기

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-peek
  namespace: guide-storage
spec:
  nodeSelector:
    kubernetes.io/hostname: worker-1
  volumes:
    - name: host-name
      hostPath:
        path: /etc/hostname      # ← 노드의 파일
        type: File               # ← 파일이 없으면 파드가 뜨지 않는다
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "cat /node/hostname; sleep 3600"]
      volumeMounts:
        - name: host-name
          mountPath: /node/hostname
          readOnly: true
```

`node-peek.yaml` 로 저장하고 적용합니다.

```bash
kubectl apply -f node-peek.yaml
kubectl logs node-peek -n guide-storage
```

```text
worker-1
```

### 실험 — 다른 노드의 파드는 다른 데이터를 본다

같은 hostPath 경로를 쓰는 파드를 worker-1 과 worker-2 에 하나씩 띄웁니다.

`hp-pair.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hp-w1
  namespace: guide-storage
spec:
  nodeSelector: { kubernetes.io/hostname: worker-1 }
  volumes:
    - name: box
      hostPath:
        path: /srv/guide-storage/hp      # ← 두 파드가 같은 경로
        type: DirectoryOrCreate          # ← 없으면 만든다
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "echo written-by-hp-w1 > /box/who.txt; sleep 3600"]
      volumeMounts: [{ name: box, mountPath: /box }]
---
apiVersion: v1
kind: Pod
metadata:
  name: hp-w2
  namespace: guide-storage
spec:
  nodeSelector: { kubernetes.io/hostname: worker-2 }
  volumes:
    - name: box
      hostPath:
        path: /srv/guide-storage/hp
        type: DirectoryOrCreate
  containers:
    - name: app
      image: busybox:1.36
      command: ["sh", "-c", "sleep 3600"]          # ← 이쪽은 쓰지 않는다
      volumeMounts: [{ name: box, mountPath: /box }]
```

```bash
kubectl apply -f hp-pair.yaml
kubectl exec hp-w1 -n guide-storage -- ls /box
kubectl exec hp-w2 -n guide-storage -- ls /box
```

```text
who.txt       ← worker-1 의 /srv/guide-storage/hp
              ← worker-2 의 /srv/guide-storage/hp (비어 있다)
```

**관찰 포인트** — 경로가 같아도 **노드가 다르면 다른 디렉터리**입니다. 파드가 다른 노드로 옮겨 가면 데이터가 "사라진" 것처럼 보입니다. 그래서 일반 앱의 저장소로 hostPath 를 쓰려면 파드를 그 노드에 **묶어야** 합니다(4) 의 PV `nodeAffinity`).

| `hostPath.type` | 동작 |
|-----------------|------|
| (빈 값) | 검사 없이 마운트 |
| `DirectoryOrCreate` | 디렉터리가 없으면 만든다 (권한 0755) |
| `Directory` | 디렉터리가 있어야 한다. 없으면 파드가 `ContainerCreating` 에 멈춘다 |
| `FileOrCreate` / `File` | 파일 버전 |

!!! warning "hostPath 는 보안상 위험합니다"
    노드의 `/`, `/etc`, `/var/run/containerd` 같은 경로를 쓰기 가능하게 붙이면 컨테이너가 노드를 장악할 수 있습니다. 실무에서는 Pod Security 로 막는 경우가 많습니다.

```bash
kubectl delete pod node-peek hp-w1 hp-w2 -n guide-storage
```

---

## 4) PV 와 PVC — 준비하는 쪽과 요청하는 쪽

- **PersistentVolume(PV)** — 관리자가 준비한 실제 저장 공간. **클러스터 범위**(`-n` 없음).
- **PersistentVolumeClaim(PVC)** — 앱이 내는 요청서("이만큼, 이런 방식으로"). **네임스페이스 범위**.
- 파드는 PV 를 직접 모르고 **PVC 이름만** 적습니다. 저장소가 hostPath 인지 NFS 인지 클라우드 디스크인지 앱은 몰라도 됩니다.

### PV 만들기 — `kubectl create` 가 없다

PV 는 명령으로 만들 수 없어서 YAML 을 씁니다. kubernetes.io 의 **"Configure a Pod to Use a PersistentVolume for Storage"** 문서 예제가 이 모양입니다.

`pv-a.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: guide-pv-a               # ← 네임스페이스가 없다 (클러스터 범위)
  labels:
    purpose: notes
spec:
  capacity:
    storage: 2Gi                 # ← 이 PV 가 제공하는 용량
  accessModes:
    - ReadWriteOnce              # ← 한 노드에서 읽기·쓰기
  persistentVolumeReclaimPolicy: Retain   # ← PVC 를 지워도 데이터를 남긴다 (6) 참고)
  hostPath:
    path: /srv/guide-storage/pv-a
    type: DirectoryOrCreate
  nodeAffinity:                  # ← 이 PV 는 worker-1 에만 있다 → 쓰는 파드도 worker-1 로 간다
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values: ["worker-1"]
```

```bash
kubectl apply -f pv-a.yaml
kubectl get pv
```

```text
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   VOLUMEATTRIBUTESCLASS   REASON   AGE
guide-pv-a   2Gi        RWO            Retain           Available                          <unset>                          3s
```

`STORAGECLASS` 칸이 비어 있습니다. 클래스를 적지 않은 PV 는 **클래스가 없는(빈 문자열) PVC** 와만 짝이 됩니다.

### PVC 만들기

`notes-claim.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: notes-claim
  namespace: guide-storage       # ← PVC 는 네임스페이스 안에 산다
spec:
  accessModes:
    - ReadWriteOnce              # ← PV 가 이 모드를 지원해야 한다
  resources:
    requests:
      storage: 2Gi               # ← PV 용량 ≥ 이 값이어야 한다
  # storageClassName 을 적지 않았다 → 기본 StorageClass 가 없으니 "클래스 없음" 으로 남는다
```

```bash
kubectl apply -f notes-claim.yaml
kubectl get pv,pvc -n guide-storage
```

```text
NAME                          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                       STORAGECLASS   ...
persistentvolume/guide-pv-a   2Gi        RWO            Retain           Bound    guide-storage/notes-claim                  ...

NAME                                STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS   ...
persistentvolumeclaim/notes-claim   Bound    guide-pv-a   2Gi        RWO                           ...
```

**관찰 포인트** — PV 의 `CLAIM` 칸에 `네임스페이스/PVC이름` 이, PVC 의 `VOLUME` 칸에 PV 이름이 적혔습니다. 이 짝짓기를 **바인딩**이라 하며, PV 컨트롤러가 자동으로 합니다.

### 파드에서 PVC 쓰기 — 볼륨 종류만 바뀐다

`notes-app.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: notes-app
  namespace: guide-storage
spec:
  volumes:
    - name: notes
      persistentVolumeClaim:
        claimName: notes-claim   # ← PV 이름이 아니라 PVC 이름
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts:
        - name: notes
          mountPath: /data
```

```bash
kubectl apply -f notes-app.yaml
kubectl get pod notes-app -n guide-storage -o wide
kubectl exec notes-app -n guide-storage -- sh -c 'echo "saved at $(date)" > /data/memo.txt'
kubectl exec notes-app -n guide-storage -- df -h /data
```

```text
NAME        READY   STATUS    NODE
notes-app   1/1     Running   worker-1          ← PV 의 nodeAffinity 때문에 worker-1

Filesystem                         Size  Used Avail Use% Mounted on
/dev/mapper/ubuntu--vg-ubuntu--lv   30G  8.2G   20G  30% /data
```

!!! warning "hostPath PV 의 용량은 강제되지 않습니다"
    `df` 에 2Gi 가 아니라 **노드 디스크 전체 크기**가 보입니다. PV 의 `capacity` 는 바인딩할 때 비교하는 **숫자**일 뿐, hostPath 에서는 실제 제한이 없습니다. 클라우드 디스크(CSI)는 그 크기로 만들어집니다.

### 파드를 지우고 다시 만들면

```bash
kubectl delete pod notes-app -n guide-storage
kubectl apply -f notes-app.yaml
kubectl exec notes-app -n guide-storage -- cat /data/memo.txt
```

```text
saved at Mon Sep 29 07:35:02 UTC 2026        ← 파드가 새로 만들어져도 남아 있다
```

**관찰 포인트** — 1) 의 emptyDir 과 달리 **파드가 삭제돼도** 데이터가 남습니다. 데이터의 수명이 파드가 아니라 **PVC(와 PV)** 에 묶였기 때문입니다.

```bash
kubectl explain pv.spec.persistentVolumeReclaimPolicy
```

```text
FIELD: persistentVolumeReclaimPolicy <string>
ENUM:
    Delete
    Recycle
    Retain
DESCRIPTION:
    persistentVolumeReclaimPolicy defines what happens to a persistent volume
    when released from its claim. ...
```

---

## 5) 바인딩 조건 — 일부러 Pending 을 만들어 본다

PVC 가 PV 에 붙으려면 다음이 모두 맞아야 합니다. 하나라도 어긋나면 PVC 는 **Pending** 에 머뭅니다.

| 조건 | 맞아야 하는 것 |
|------|----------------|
| storageClassName | PV 와 PVC 가 **같은 값** (둘 다 비어 있는 것도 "같다") |
| accessModes | PVC 가 요청한 모드를 PV 가 **지원** |
| 용량 | PV 용량 **≥** PVC 요청 |
| 짝 | PV 하나에 PVC 하나 (1:1). 이미 Bound 된 PV 는 못 쓴다 |
| (선택) selector / volumeName | PVC 가 특정 레이블·이름의 PV 를 지정했다면 그것만 |

실험용으로 worker-2 에 3Gi PV 를 하나 더 만듭니다.

`pv-b.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: guide-pv-b
  labels:
    purpose: scratch
spec:
  capacity:
    storage: 3Gi
  accessModes: ["ReadWriteOnce"]
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /srv/guide-storage/pv-b
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - { key: kubernetes.io/hostname, operator: In, values: ["worker-2"] }
```

```bash
kubectl apply -f pv-b.yaml
```

어긋난 PVC 네 개를 한 번에 만듭니다.

`bad-claims.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: want-5g, namespace: guide-storage }
spec:
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 5Gi } }       # ← 3Gi PV 보다 크다
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: want-rwx, namespace: guide-storage }
spec:
  accessModes: ["ReadWriteMany"]                   # ← PV 는 RWO 만 지원
  resources: { requests: { storage: 300Mi } }
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: want-typo, namespace: guide-storage }
spec:
  storageClassName: guide-manul                    # ← 없는 클래스 (오타)
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 300Mi } }
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: want-small, namespace: guide-storage }
spec:
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 300Mi } }     # ← 이것만 조건이 맞는다
```

```bash
kubectl apply -f bad-claims.yaml
kubectl get pvc -n guide-storage
```

```text
NAME          STATUS    VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS
notes-claim   Bound     guide-pv-a   2Gi        RWO
want-5g       Pending
want-rwx      Pending
want-small    Bound     guide-pv-b   3Gi        RWO                         ← 300Mi 를 요청했는데 3Gi
want-typo     Pending                                          guide-manul
```

**관찰 포인트** — `want-small` 은 300Mi 를 요청했지만 **3Gi PV 전체**를 가져갔습니다. 요청 용량은 "최소 요구"이고, PV 는 쪼개지지 않고 1:1 로 붙습니다. 이제 `guide-pv-b` 는 다른 PVC 가 쓸 수 없습니다.

이유는 각 PVC 의 Events 에 있습니다.

```bash
kubectl describe pvc want-5g -n guide-storage | tail -3
kubectl describe pvc want-typo -n guide-storage | tail -3
```

```text
  Type    Reason         Age   From                         Message
  ----    ------         ----  ----                         -------
  Normal  FailedBinding  5s    persistentvolume-controller  no persistent volumes available for this claim and no storage class is set

  Type     Reason              Age   From                         Message
  ----     ------              ----  ----                         -------
  Warning  ProvisioningFailed  5s    persistentvolume-controller  storageclass.storage.k8s.io "guide-manul" not found
```

| Events 메시지 | 뜻 | 확인할 것 |
|---------------|----|-----------|
| `no persistent volumes available for this claim` | 조건 맞는 PV 가 없다 | 용량 · accessModes · 이미 Bound 인지 |
| `storageclass.storage.k8s.io "..." not found` | 그런 클래스가 없다 | `kubectl get sc`, 이름 오타 |
| `waiting for first consumer to be created before binding` | **정상** — 파드를 기다리는 중 | 7) 참고 |

!!! tip "시험 팁 — 특정 PV 에 붙이라는 문제"
    PVC 에 `spec.volumeName: <PV이름>` 을 적으면 그 PV 에만 붙습니다. 레이블로 고르려면 `spec.selector.matchLabels` 를 씁니다 (예: `purpose: notes`).
    그래도 **용량·모드·클래스 조건은 여전히 맞아야** 합니다.

| accessModes | 줄임 | 의미 |
|-------------|------|------|
| ReadWriteOnce | RWO | **한 노드**에서 읽기·쓰기 (같은 노드의 파드 여럿은 함께 쓸 수 있다) |
| ReadOnlyMany | ROX | 여러 노드에서 읽기만 |
| ReadWriteMany | RWX | 여러 노드에서 읽기·쓰기 (NFS 등 일부만 지원) |
| ReadWriteOncePod | RWOP | **파드 하나**만 (CSI 볼륨) |

```bash
kubectl delete -f bad-claims.yaml
kubectl get pv guide-pv-b
```

```text
NAME         CAPACITY   ...   RECLAIM POLICY   STATUS     CLAIM                      ...
guide-pv-b   3Gi        ...   Retain           Released   guide-storage/want-small   ...
```

`want-small` 을 지웠더니 `guide-pv-b` 가 **Released** 가 되었습니다. 이것이 다음 절의 주제입니다.

---

## 6) reclaimPolicy — PVC 를 지우면 PV 는 어떻게 되나

| 정책 | PVC 삭제 후 PV | 데이터 | 주로 쓰는 곳 |
|------|----------------|--------|--------------|
| `Retain` | **Released** 로 남는다 (다른 PVC 가 바로 못 씀) | 남는다 | 정적으로 만든 PV, 중요한 데이터 |
| `Delete` | PV 와 실제 디스크까지 삭제 | 사라진다 | 동적 프로비저닝 PV 의 기본값 |
| `Recycle` | (폐기됨) | — | 쓰지 않는다 |

### Released 를 직접 만들기

4) 의 `notes-claim` 을 지웁니다. 쓰고 있는 파드부터 지워야 합니다.

```bash
kubectl delete pod notes-app -n guide-storage
kubectl delete pvc notes-claim -n guide-storage
kubectl get pv guide-pv-a
```

```text
NAME         CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS     CLAIM                       ...
guide-pv-a   2Gi        RWO            Retain           Released   guide-storage/notes-claim   ...
```

같은 조건의 새 PVC 를 만들어 봅니다.

```bash
sed 's/name: notes-claim/name: notes-claim-2/' notes-claim.yaml | kubectl apply -f -
kubectl get pvc notes-claim-2 -n guide-storage
```

```text
NAME            STATUS    VOLUME   CAPACITY   ACCESS MODES
notes-claim-2   Pending
```

**관찰 포인트** — 용량도 모드도 맞는데 Pending 입니다. Released PV 에는 **옛 PVC 를 가리키는 `claimRef`** 가 남아 있어서, 다른 PVC 가 옛 데이터를 실수로 가져가지 못하게 막습니다.

```bash
kubectl get pv guide-pv-a -o jsonpath='{.spec.claimRef.namespace}/{.spec.claimRef.name}{"\n"}'
```

```text
guide-storage/notes-claim
```

관리자가 "이 데이터는 다시 써도 된다"고 판단하면 `claimRef` 를 지웁니다.

```bash
kubectl patch pv guide-pv-a --type=json -p '[{"op":"remove","path":"/spec/claimRef"}]'
kubectl get pv guide-pv-a
kubectl get pvc notes-claim-2 -n guide-storage
```

```text
NAME         ...   STATUS   CLAIM                         ...
guide-pv-a   ...   Bound    guide-storage/notes-claim-2   ...

NAME            STATUS   VOLUME       CAPACITY
notes-claim-2   Bound    guide-pv-a   2Gi
```

Released → Available 를 거쳐 곧바로 새 PVC 에 붙었습니다. 옛 데이터도 그대로입니다.

```bash
sed 's/claimName: notes-claim /claimName: notes-claim-2 /' notes-app.yaml | kubectl apply -f -
kubectl exec notes-app -n guide-storage -- cat /data/memo.txt
```

```text
saved at Mon Sep 29 07:35:02 UTC 2026
```

!!! warning "정적 hostPath PV 에 `Delete` 를 주면"
    지울 수 있는 플러그인이 없어서 PV 가 **Failed** 상태로 남을 수 있습니다. 정적으로 만든 PV 는 `Retain` 이 기본 선택입니다.
    이미 만든 PV 의 정책은 `kubectl patch pv <이름> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'` 로 바꿀 수 있습니다.

다음 절을 위해 4)~6) 의 리소스를 치웁니다. PV 가 남아 있으면 다음 PVC 가 엉뚱하게 거기에 붙을 수 있습니다.

```bash
kubectl delete pod notes-app -n guide-storage
kubectl delete pvc notes-claim-2 -n guide-storage
kubectl delete pv guide-pv-a guide-pv-b
```

---

## 7) StorageClass 와 volumeBindingMode — 언제, 어느 PV 에 붙일까

StorageClass 는 원래 **동적 프로비저닝**(PVC 가 들어오면 프로비저너가 PV 를 만들어 줌)을 위한 것입니다.
프로비저너가 없는 클러스터에서도 `kubernetes.io/no-provisioner` 로 클래스를 만들어 **바인딩 시점**을 제어하는 데 씁니다. 노드에 묶인 저장소(local·hostPath)에서 특히 중요합니다.

| volumeBindingMode | 바인딩 시점 | 결과 |
|-------------------|-------------|------|
| `Immediate` (기본값) | PVC 를 만들자마자 | PV 가 파드보다 먼저 정해진다 → 파드가 그 PV 의 노드로 **끌려간다** |
| `WaitForFirstConsumer` | 그 PVC 를 쓰는 **파드가 스케줄될 때** | 파드가 갈 노드를 먼저 정하고, **그 노드의 PV** 를 고른다 |

### 두 클래스 만들기

`classes.yaml`:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: guide-wait                              # ← 클러스터 범위
provisioner: kubernetes.io/no-provisioner       # ← PV 를 만들어 주지 않는다 (정적)
volumeBindingMode: WaitForFirstConsumer         # ← 파드가 생길 때까지 기다린다
reclaimPolicy: Retain
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: guide-now
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: Immediate                    # ← 바로 붙인다
reclaimPolicy: Retain
```

```bash
kubectl apply -f classes.yaml
kubectl get storageclass
```

```text
NAME         PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
guide-now    kubernetes.io/no-provisioner   Retain          Immediate              false                  2s
guide-wait   kubernetes.io/no-provisioner   Retain          WaitForFirstConsumer   false                  2s
```

`kubectl create storageclass` 명령은 없습니다. kubernetes.io 의 **"Storage Classes"** 문서(특히 *Local* 항목) 예제를 복사하세요.

### 각 클래스에 PV 준비

`guide-wait` 에는 워커마다 하나씩, `guide-now` 에는 worker-1 에만 PV 를 둡니다.

`class-pvs.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata: { name: guide-wait-w1 }
spec:
  storageClassName: guide-wait          # ← PVC 와 같은 클래스 이름
  capacity: { storage: 2Gi }
  accessModes: ["ReadWriteOnce"]
  hostPath: { path: /srv/guide-storage/wait, type: DirectoryOrCreate }
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions: [{ key: kubernetes.io/hostname, operator: In, values: ["worker-1"] }]
---
apiVersion: v1
kind: PersistentVolume
metadata: { name: guide-wait-w2 }
spec:
  storageClassName: guide-wait
  capacity: { storage: 2Gi }
  accessModes: ["ReadWriteOnce"]
  hostPath: { path: /srv/guide-storage/wait, type: DirectoryOrCreate }
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions: [{ key: kubernetes.io/hostname, operator: In, values: ["worker-2"] }]
---
apiVersion: v1
kind: PersistentVolume
metadata: { name: guide-now-w1 }
spec:
  storageClassName: guide-now
  capacity: { storage: 2Gi }
  accessModes: ["ReadWriteOnce"]
  hostPath: { path: /srv/guide-storage/now, type: DirectoryOrCreate }
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions: [{ key: kubernetes.io/hostname, operator: In, values: ["worker-1"] }]
```

```bash
kubectl apply -f class-pvs.yaml
```

!!! note "PV 의 `persistentVolumeReclaimPolicy` 를 생략하면"
    직접 만든 PV 에서 생략하면 `Retain` 입니다. StorageClass 의 `reclaimPolicy` 는 **프로비저너가 동적으로 만드는 PV** 에만 적용됩니다.

### WaitForFirstConsumer — PVC 만 만들면 Pending 이 정상

```bash
kubectl create -n guide-storage -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: wait-claim }
spec:
  storageClassName: guide-wait
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 400Mi } }
EOF
kubectl get pvc wait-claim -n guide-storage
kubectl describe pvc wait-claim -n guide-storage | tail -2
```

```text
NAME         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
wait-claim   Pending                                      guide-wait

  Normal  WaitForFirstConsumer  3s (x2 over 5s)  persistentvolume-controller  waiting for first consumer to be created before binding
```

이제 이 PVC 를 쓰는 파드를 **worker-2 에 두도록** 만듭니다.

`wait-user.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: wait-user
  namespace: guide-storage
spec:
  nodeSelector: { kubernetes.io/hostname: worker-2 }   # ← 파드는 worker-2 에 가야 한다
  volumes:
    - name: v
      persistentVolumeClaim: { claimName: wait-claim }
  containers:
    - name: app
      image: nginx:1.27
      volumeMounts: [{ name: v, mountPath: /data }]
```

```bash
kubectl apply -f wait-user.yaml
kubectl get pod wait-user -n guide-storage -o wide
kubectl get pvc wait-claim -n guide-storage
```

```text
NAME        READY   STATUS    NODE
wait-user   1/1     Running   worker-2

NAME         STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS
wait-claim   Bound    guide-wait-w2   2Gi        RWO            guide-wait     ← worker-2 의 PV
```

**관찰 포인트** — 후보 PV 가 두 개였는데 **파드가 갈 노드(worker-2)의 PV** 가 골라졌습니다. 스케줄러가 먼저 노드를 정하고, 그 노드에서 쓸 수 있는 PV 를 바인딩한 것입니다.

### 실험 — Immediate 라면

같은 모양으로 `guide-now` 클래스를 씁니다. `guide-now` 의 PV 는 worker-1 에만 있고, 파드는 worker-2 에 두려 합니다.

```bash
kubectl create -n guide-storage -f - <<'EOF'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: now-claim }
spec:
  storageClassName: guide-now
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 400Mi } }
EOF
kubectl get pvc now-claim -n guide-storage
sed -e 's/name: wait-user/name: now-user/' -e 's/claimName: wait-claim/claimName: now-claim/' wait-user.yaml | kubectl apply -f -
kubectl get pod now-user -n guide-storage
kubectl describe pod now-user -n guide-storage | tail -1
```

```text
NAME        STATUS   VOLUME         CAPACITY   ACCESS MODES   STORAGECLASS
now-claim   Bound    guide-now-w1   2Gi        RWO            guide-now      ← 파드도 없는데 이미 Bound

NAME       READY   STATUS    RESTARTS   AGE
now-user   0/1     Pending   0          5s

  Warning  FailedScheduling  5s  default-scheduler  0/3 nodes are available: 1 node(s) didn't match Pod's node affinity/selector, 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 1 node(s) had volume node affinity conflict. preemption: ...
```

**관찰 포인트** — PVC 가 먼저 worker-1 의 PV 에 붙어 버려서, worker-2 에 가야 하는 파드는 **갈 곳이 없습니다**(`volume node affinity conflict`). 노드에 묶인 저장소에 `WaitForFirstConsumer` 를 쓰는 이유가 이것입니다.

### 기본 StorageClass

PVC 에 `storageClassName` 을 **안 적으면** 기본 클래스가 자동으로 들어갑니다. 기본 클래스는 어노테이션 하나로 정합니다.

```bash
kubectl patch storageclass guide-wait \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
kubectl get sc
```

```text
NAME                   PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE
guide-now              kubernetes.io/no-provisioner   Retain          Immediate
guide-wait (default)   kubernetes.io/no-provisioner   Retain          WaitForFirstConsumer
```

확인했으면 **바로 되돌립니다.** 기본 클래스가 남아 있으면 다른 실습의 "클래스 없는 PVC" 가 이 클래스를 받아 결과가 달라집니다.

```bash
kubectl patch storageclass guide-wait \
  -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

!!! warning "`storageClassName` 을 안 적은 것과 `\"\"` 로 적은 것은 다릅니다"
    - 안 적음 → **기본 클래스가 있으면** 그 클래스가 채워진다.
    - `storageClassName: ""` → 항상 "클래스 없음". 클래스가 없는 PV 에만 붙는다.
    시험에서 "no storage class" 를 요구하거나 기본 클래스가 있는 환경이라면 `""` 를 명시하는 것이 안전합니다.

??? note "참고 — 동적 프로비저닝을 직접 보고 싶다면 (선택)"
    실습 클러스터에 `local-path-provisioner` 를 설치하면 PVC 만 만들어도 PV 가 자동으로 생깁니다. 강의 범위 밖이니 보고 나서 지우세요.
    버전은 [릴리스 페이지](https://github.com/rancher/local-path-provisioner/releases)에서 최신 태그를 확인해 바꿔 넣으세요.
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.30/deploy/local-path-storage.yaml
    kubectl get sc local-path
    kubectl create -n guide-storage -f - <<'EOF'
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata: { name: dyn-claim }
    spec:
      storageClassName: local-path
      accessModes: ["ReadWriteOnce"]
      resources: { requests: { storage: 200Mi } }
    EOF
    kubectl run dyn-user -n guide-storage --image=nginx:1.27 \
      --overrides='{"spec":{"volumes":[{"name":"v","persistentVolumeClaim":{"claimName":"dyn-claim"}}],"containers":[{"name":"dyn-user","image":"nginx:1.27","volumeMounts":[{"name":"v","mountPath":"/data"}]}]}}'
    kubectl get pv       # pvc-<UID> 라는 이름의 PV 가 자동으로 생겼다 (RECLAIM POLICY: Delete)
    ```
    지우기:
    ```bash
    kubectl delete pod dyn-user -n guide-storage
    kubectl delete pvc dyn-claim -n guide-storage          # Delete 정책 → PV 도 사라진다
    kubectl delete -f https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.30/deploy/local-path-storage.yaml
    ```

다음 절은 `guide-wait` 클래스를 다시 씁니다. 이 절의 파드·PVC·PV 와 `guide-now` 클래스를 치웁니다.

```bash
kubectl delete pod wait-user now-user -n guide-storage
kubectl delete pvc wait-claim now-claim -n guide-storage
kubectl delete pv guide-wait-w1 guide-wait-w2 guide-now-w1
kubectl delete storageclass guide-now
```

---

## 8) StatefulSet — 파드마다 자기 PVC 를 갖는다

| | Deployment | StatefulSet |
|---|---|---|
| 파드 이름 | `web-7d9f…-x8k2` (무작위) | `kv-0`, `kv-1` (고정) |
| 저장소 | 템플릿의 PVC 를 **모두 공유** | `volumeClaimTemplates` 로 **파드마다 따로** |
| 생성·삭제 | 동시에 | 0 → 1 → 2 순서 (기본 `OrderedReady`) |
| 네트워크 이름 | Service 하나 | headless Service 로 **파드마다 DNS** |

### headless Service + StatefulSet

`kv.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: kv
  namespace: guide-storage
spec:
  clusterIP: None              # ← headless: 로드밸런싱 대신 파드별 DNS 를 만든다
  selector:
    app: kv
  ports:
    - port: 80
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kv
  namespace: guide-storage
spec:
  serviceName: kv              # ← 위 headless Service 이름
  replicas: 2
  selector:
    matchLabels:
      app: kv                  # ← template 레이블과 같아야 한다
  template:
    metadata:
      labels:
        app: kv
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          volumeMounts:
            - name: data       # ← volumeClaimTemplates 의 이름
              mountPath: /data
  volumeClaimTemplates:        # ← PVC 틀. spec.template 이 아니라 spec 바로 아래
    - metadata:
        name: data
      spec:
        storageClassName: guide-wait
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 256Mi
```

### 실험 — PV 가 없는 상태에서 먼저 만든다

```bash
kubectl apply -f kv.yaml
kubectl get sts,pods,pvc -n guide-storage
```

```text
NAME                  READY   AGE
statefulset.apps/kv   0/2     10s

NAME       READY   STATUS    RESTARTS   AGE
pod/kv-0   0/1     Pending   0          10s          ← kv-1 은 아예 없다

NAME                              STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS
persistentvolumeclaim/data-kv-0   Pending                                      guide-wait
```

```bash
kubectl describe pod kv-0 -n guide-storage | tail -1
```

```text
  Warning  FailedScheduling  ...  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 node(s) didn't find available persistent volumes to bind. preemption: ...
```

**관찰 포인트** — ① PVC 이름이 `<템플릿이름>-<파드이름>` = `data-kv-0` 으로 **자동 생성**되었습니다. ② `kv-0` 이 Ready 가 아니라서 `kv-1` 은 **만들어지지도 않았습니다**(순서 보장). ③ 파드의 Pending 이유가 스토리지(`didn't find available persistent volumes`)로 나옵니다.

### PV 를 넣어 주면 이어서 진행된다

`kv-pvs.yaml`:

```yaml
apiVersion: v1
kind: PersistentVolume
metadata: { name: guide-kv-w1 }
spec:
  storageClassName: guide-wait
  capacity: { storage: 512Mi }
  accessModes: ["ReadWriteOnce"]
  hostPath: { path: /srv/guide-storage/kv, type: DirectoryOrCreate }
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions: [{ key: kubernetes.io/hostname, operator: In, values: ["worker-1"] }]
---
apiVersion: v1
kind: PersistentVolume
metadata: { name: guide-kv-w2 }
spec:
  storageClassName: guide-wait
  capacity: { storage: 512Mi }
  accessModes: ["ReadWriteOnce"]
  hostPath: { path: /srv/guide-storage/kv, type: DirectoryOrCreate }
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions: [{ key: kubernetes.io/hostname, operator: In, values: ["worker-2"] }]
```

```bash
kubectl apply -f kv-pvs.yaml
kubectl get pods -n guide-storage -l app=kv -w      # kv-0 → kv-1 순서로 뜨면 Ctrl+C
kubectl get pods,pvc -n guide-storage -o wide
```

```text
NAME       READY   STATUS    NODE
pod/kv-0   1/1     Running   worker-1
pod/kv-1   1/1     Running   worker-2

NAME                              STATUS   VOLUME        CAPACITY   STORAGECLASS
persistentvolumeclaim/data-kv-0   Bound    guide-kv-w1   512Mi      guide-wait
persistentvolumeclaim/data-kv-1   Bound    guide-kv-w2   512Mi      guide-wait
```

(어느 파드가 어느 워커에 가는지는 환경마다 다를 수 있습니다.)

### 파드를 지워도 같은 이름 · 같은 PVC

각 파드의 볼륨에 자기 이름을 적어 둡니다.

```bash
for i in 0 1; do kubectl exec kv-$i -n guide-storage -- sh -c 'cat /etc/hostname > /data/owner.txt'; done
kubectl delete pod kv-0 -n guide-storage
kubectl get pod kv-0 -n guide-storage -o wide
kubectl exec kv-0 -n guide-storage -- cat /data/owner.txt
```

```text
NAME   READY   STATUS    NODE
kv-0   1/1     Running   worker-1

kv-0
```

**관찰 포인트** — 지운 파드가 **같은 이름**으로, **같은 PVC(`data-kv-0`)** 를 달고 돌아왔습니다. Deployment 라면 무작위 이름의 새 파드가 됐을 것입니다.

### 파드별 DNS

```bash
kubectl run dns -n guide-storage --rm -it --image=busybox:1.36 --restart=Never -- \
  nslookup kv-1.kv.guide-storage.svc.cluster.local
```

```text
Name:      kv-1.kv.guide-storage.svc.cluster.local
Address:   10.0.1.87
pod "dns" deleted
```

`<파드이름>.<serviceName>.<네임스페이스>.svc.cluster.local` 형식입니다. DB 복제처럼 **특정 인스턴스**에 붙어야 할 때 씁니다.

### 줄이거나 지워도 PVC 는 남는다

```bash
kubectl scale statefulset kv -n guide-storage --replicas=1
kubectl get pods,pvc -n guide-storage
```

```text
NAME       READY   STATUS    RESTARTS   AGE
pod/kv-0   1/1     Running   0          2m

NAME                              STATUS   VOLUME
persistentvolumeclaim/data-kv-0   Bound    guide-kv-w1
persistentvolumeclaim/data-kv-1   Bound    guide-kv-w2     ← kv-1 은 없어졌는데 PVC 는 남았다
```

다시 2 로 늘리면 `kv-1` 이 **예전 PVC 를 그대로** 받아 `owner.txt` 가 남아 있습니다. 데이터를 보호하려는 설계입니다.
PVC 까지 함께 지우고 싶다면 `spec.persistentVolumeClaimRetentionPolicy`(`whenDeleted`, `whenScaled` 를 `Delete` 로)를 씁니다. v1.32 에서 GA 입니다.

```bash
kubectl explain statefulset.spec.persistentVolumeClaimRetentionPolicy
```

!!! tip "시험 팁 — StatefulSet 은 `create` 가 없다"
    - `kubectl create deployment x --image=... $do > sts.yaml` 로 뽑아 `kind: StatefulSet` 으로 바꾸고, `strategy` 를 지우고, `serviceName` 과 `volumeClaimTemplates` 를 추가합니다.
    - 또는 kubernetes.io 의 **"StatefulSets"** 개념 문서 첫 예제(headless Service + StatefulSet + volumeClaimTemplates)를 통째로 복사해 이름만 바꿉니다.
    - 확인: `kubectl get pvc` 에 `<템플릿>-<sts이름>-<번호>` 가 replicas 개수만큼 있는지. **PV 가 없으면 PVC 는 Pending 이어도 생성은 됩니다.**

!!! warning "자주 하는 실수"
    - `volumeClaimTemplates` 를 `spec.template.spec` 안에 넣음 → 알 수 없는 필드로 거부. **`spec` 바로 아래**입니다.
    - 컨테이너 `volumeMounts.name` 과 템플릿 `metadata.name` 이 다름 → 생성 거부.
    - `selector.matchLabels` 와 `template.metadata.labels` 가 다름 → `selector does not match template labels`.

---

## 시험에서 빠르게

```bash
# 조회 — PV·SC 는 클러스터 범위(-n 없음), PVC 는 네임스페이스 범위
kubectl get pv,sc
kubectl get pvc -n <ns>
kubectl describe pvc <이름> -n <ns> | tail -5     # Pending 이유
kubectl exec <pod> -n <ns> -- df -h /경로         # 실제로 마운트됐나

# 필드 찾기
kubectl explain pv.spec --recursive | less
kubectl explain pvc.spec
kubectl explain pod.spec.volumes.persistentVolumeClaim
kubectl explain sts.spec.volumeClaimTemplates

# 파드 뼈대 뽑아서 volumes / volumeMounts 두 블록 추가
kubectl run p --image=nginx $do > p.yaml

# Released PV 되살리기 · 정책 바꾸기
kubectl patch pv <pv> --type=json -p '[{"op":"remove","path":"/spec/claimRef"}]'
kubectl patch pv <pv> -p '{"spec":{"persistentVolumeReclaimPolicy":"Retain"}}'

# 기본 StorageClass 지정
kubectl patch sc <sc> -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

!!! tip "kubernetes.io 에서 복사할 문서 제목"
    - **Configure a Pod to Use a PersistentVolume for Storage** — hostPath PV · PVC · 파드 3종 세트
    - **Storage Classes** — StorageClass YAML (Local 항목에 no-provisioner + WaitForFirstConsumer)
    - **StatefulSets** — headless Service + volumeClaimTemplates 예제
    - **Communicate Between Containers in the Same Pod Using a Shared Volume** — emptyDir 공유 예제

---

## 정리

```bash
kubectl delete namespace guide-storage

# 클러스터 범위 — 네임스페이스와 함께 지워지지 않는다
kubectl delete pv guide-kv-w1 guide-kv-w2
kubectl delete pv guide-pv-a guide-pv-b guide-wait-w1 guide-wait-w2 guide-now-w1 --ignore-not-found
kubectl delete storageclass guide-wait guide-now --ignore-not-found
kubectl get pv,sc                                   # guide- 로 시작하는 것이 없어야 한다

# 노드에 남은 hostPath 디렉터리 (Retain 이라 데이터가 그대로 있다)
ssh -t worker-1 'sudo rm -rf /srv/guide-storage'
ssh -t worker-2 'sudo rm -rf /srv/guide-storage'

# 로컬 파일
rm -f scratchpad-vol.yaml relay.yaml relay-typo.yaml node-peek.yaml hp-pair.yaml pv-a.yaml pv-b.yaml \
      notes-claim.yaml notes-app.yaml bad-claims.yaml classes.yaml class-pvs.yaml wait-user.yaml kv.yaml kv-pvs.yaml
```

!!! note "순서가 중요합니다"
    네임스페이스를 먼저 지우면 PVC 가 사라지고 PV 는 `Released` 로 남습니다. 그 PV 를 지우는 것이 두 번째 줄입니다.
    기본 StorageClass 어노테이션을 7) 에서 되돌리지 않았다면 StorageClass 를 지우는 것으로 함께 정리됩니다.

---

## 확인 질문

??? question "1. 컨테이너가 재시작되면 emptyDir 의 데이터는 어떻게 되나요? 파드가 삭제되면요?"
    컨테이너 재시작에는 **남습니다**. 파드가 삭제되면 **함께 사라집니다**. 파드 수명을 넘어 남기려면 PVC 를 씁니다.

??? question "2. 같은 파드의 두 컨테이너가 같은 볼륨을 서로 다른 경로에 마운트할 수 있나요?"
    있습니다. 이어 주는 것은 경로가 아니라 `volumes[].name` 과 `volumeMounts[].name` 입니다. 마운트마다 `readOnly` 도 따로 줄 수 있습니다.

??? question "3. PV 는 `kubectl get pv -n <ns>` 로 조회해야 하나요?"
    아닙니다. PV 와 StorageClass 는 **클러스터 범위**라 `-n` 이 의미가 없습니다. PVC 만 네임스페이스 범위입니다.

??? question "4. 300Mi 를 요청한 PVC 가 5Gi PV 에 붙었습니다. PVC 의 CAPACITY 는 얼마로 보이나요?"
    5Gi 입니다. 요청 용량은 최소 요구이고 PV 는 쪼개지지 않고 1:1 로 통째로 붙습니다.

??? question "5. PVC 가 Pending 이고 Events 에 `waiting for first consumer to be created before binding` 이 있습니다. 고장인가요?"
    아닙니다. StorageClass 가 `WaitForFirstConsumer` 라서 **그 PVC 를 쓰는 파드**가 스케줄될 때까지 기다리는 정상 상태입니다. 파드를 만들면 파드가 갈 노드의 PV 로 바인딩됩니다.

??? question "6. Retain 정책의 PV 가 Released 인데 새 PVC 가 붙지 않습니다. 어떻게 다시 쓰나요?"
    PV 에 옛 PVC 를 가리키는 `spec.claimRef` 가 남아 있기 때문입니다. 데이터를 확인한 뒤
    `kubectl patch pv <pv> --type=json -p '[{"op":"remove","path":"/spec/claimRef"}]'` 로 지우면 Available 이 됩니다.

??? question "7. `replicas: 3`, 템플릿 이름 `cache`, StatefulSet 이름 `redis` 이면 PVC 이름은?"
    `cache-redis-0`, `cache-redis-1`, `cache-redis-2` 입니다. 규칙은 `<템플릿이름>-<파드이름>` 이고, 파드 이름은 `<sts이름>-<번호>` 입니다.

??? question "8. StatefulSet 을 지웠는데 PVC 가 남아 있습니다. 정상인가요?"
    정상입니다. 데이터 보호를 위해 기본으로 PVC 를 남깁니다. 함께 지우려면 PVC 를 직접 지우거나 `persistentVolumeClaimRetentionPolicy.whenDeleted: Delete` 를 설정합니다.

---

## 다음 단계 — 채점 실습

개념이 잡혔으면 시험 형식으로 풀어 봅니다. 이번에는 **답이 없습니다.**

```bash
cd ~/cka-lab/08-storage
bash exam.sh start
```

| 문제 | 이 페이지에서 연습한 것 |
|------|------------------------|
| Q1 PersistentVolume + PVC 바인딩 | 4) PV·PVC 작성, 5) 바인딩 조건, 6) reclaimPolicy |
| Q2 StorageClass + 그것을 쓰는 PVC | 7) no-provisioner · WaitForFirstConsumer · Pending 이 정상 |
| Q3 StatefulSet + volumeClaimTemplates | 8) 템플릿 위치 · PVC 이름 규칙 |
| Q4 emptyDir 를 공유하는 컨테이너 2개 | 1) emptyDir 수명, 2) 멀티 컨테이너 공유 |
