# 9강 · 서비스와 내부 통신

> 영상: **09강-서비스와내부통신** · 이어서 풀 채점 세트: `09-networking`

## 실습 목표

- Service 가 **셀렉터 → EndpointSlice → 파드 IP** 순서로 대상을 찾는다는 것을 직접 확인한다.
- ClusterIP · NodePort · LoadBalancer · Headless 를 모두 띄워 보고 **무엇이 다른지** 비교한다.
- `port` · `targetPort` · `nodePort` 세 포트를 **일부러 다르게** 줘 보고, 틀렸을 때의 증상을 구분한다.
- 파드 안에서 **DNS 이름 규칙**(짧은 이름 · 네임스페이스 · FQDN · Headless)을 확인한다.
- 실제 전달을 맡는 **kube-proxy / Cilium** 의 흔적을 노드에서 찾아본다.
- NetworkPolicy 를 **맛보기로** 한 번 걸어 보고, 10강으로 넘어갈 준비를 한다.

---

## 0) 준비

이 페이지의 모든 리소스는 `guide-net` 네임스페이스에 만듭니다.

```bash
kubectl create namespace guide-net
```

통신 확인은 **임시 파드**로 합니다. 이 페이지에서는 아래 한 줄 모양을 계속 씁니다.

```bash
kubectl run tmp -n guide-net --rm -i --restart=Never --image=busybox:1.36 -- wget -qO- --timeout=3 http://<대상>
```

| 옵션 | 뜻 |
|------|----|
| `--rm` | 명령이 끝나면 파드를 지운다 |
| `-i` | 출력을 내 터미널로 받는다 (`-t` 는 빼도 된다. 붙이면 경고가 섞일 수 있다) |
| `--restart=Never` | Deployment 가 아니라 **파드 하나**만 만든다 |
| `--timeout=3` | 막혔을 때 오래 기다리지 않는다 |

!!! note "busybox 의 wget · nslookup 은 '작은 버전'입니다"
    - `wget` 은 HTTPS 인증서 검증·헤더 옵션이 제한적입니다. HTTP 확인용으로만 씁니다.
    - `nslookup` 은 검색 도메인을 하나씩 붙여 보다가 `NXDOMAIN` 줄을 섞어 출력하기도 합니다. **`Name:` / `Address:` 줄만 보면 됩니다.**
    - 더 자세히 보고 싶으면 `nicolaka/netshoot`(dig · curl · nc 포함)이나 `curlimages/curl` 을 씁니다.

---

## 1) 파드 IP 는 믿을 수 없다

Deployment 를 만들고 파드 IP 를 적어 둡니다.

```bash
kubectl create deployment menu -n guide-net --image=nginx:1.27 --replicas=3
kubectl rollout status deployment/menu -n guide-net
kubectl get pods -n guide-net -l app=menu -o wide
```

```text
NAME                    READY   STATUS    IP          NODE
menu-7c9d6b8f5d-4kq2p   1/1     Running   10.0.1.23   worker-1
menu-7c9d6b8f5d-h8x7m   1/1     Running   10.0.2.41   worker-2
menu-7c9d6b8f5d-wz5tn   1/1     Running   10.0.2.57   worker-2
```

파드 하나를 지우고 다시 봅니다.

```bash
kubectl delete pod -n guide-net menu-7c9d6b8f5d-4kq2p     # 내 파드 이름으로 바꾸세요
kubectl get pods -n guide-net -l app=menu -o wide
```

```text
NAME                    READY   STATUS    IP          NODE
menu-7c9d6b8f5d-h8x7m   1/1     Running   10.0.2.41   worker-2
menu-7c9d6b8f5d-r6j9c   1/1     Running   10.0.1.88   worker-1   ← 새 이름, 새 IP
menu-7c9d6b8f5d-wz5tn   1/1     Running   10.0.2.57   worker-2
```

관찰 포인트: 파드는 다시 생기지만 **IP 는 바뀝니다**. 호출하는 쪽이 파드 IP 를 적어 두는 방식은 성립하지 않습니다.
그래서 **바뀌지 않는 주소(Service)** 가 필요합니다. 파드 IP 대역(`10.0.x.x`)은 Cilium 이 노드마다 나눠 준 것입니다.

---

## 2) ClusterIP Service 와 EndpointSlice

### expose 로 만들기 — port 와 targetPort 를 일부러 다르게

```bash
kubectl expose deployment menu -n guide-net --name=menu-svc --port=8080 --target-port=80
kubectl get svc menu-svc -n guide-net
```

```text
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
menu-svc   ClusterIP   10.104.62.18   <none>        8080/TCP   3s
```

- `--type` 을 안 주면 **ClusterIP** 입니다.
- `CLUSTER-IP` 는 Service CIDR(`10.96.0.0/12`) 안에서 하나 받습니다. **삭제할 때까지 바뀌지 않습니다.**
- `expose` 는 Deployment 의 셀렉터(`app=menu`)를 그대로 복사합니다. 그래서 셀렉터 오타가 날 일이 없습니다.

### 대상 목록은 어디에 있나

Service 는 트래픽을 보낼 **파드 IP 목록**을 따로 가지고 있습니다. 예전 형식이 Endpoints, 지금 형식이 EndpointSlice 입니다.

```bash
kubectl get endpoints menu-svc -n guide-net
kubectl get endpointslices -n guide-net -l kubernetes.io/service-name=menu-svc
```

```text
NAME       ENDPOINTS                                AGE
menu-svc   10.0.1.88:80,10.0.2.41:80,10.0.2.57:80   20s

NAME             ADDRESSTYPE   PORTS   ENDPOINTS                        AGE
menu-svc-8gk2x   IPv4          80      10.0.1.88,10.0.2.41,10.0.2.57    20s
```

관찰 포인트: 목록에 찍힌 포트는 **80(targetPort)** 입니다. Service 의 8080 은 "Service 가 받는 포트", 80 은 "파드로 넘기는 포트"입니다.

EndpointSlice 를 YAML 로 보면 파드마다 **Ready 여부**와 **어느 노드인지**까지 들어 있습니다.

```bash
kubectl get endpointslices -n guide-net -l kubernetes.io/service-name=menu-svc -o yaml | grep -E 'addresses|ready|nodeName' -A1 | head -12
```

```text
  - addresses:
    - 10.0.1.88
    conditions:
      ready: true          ← readinessProbe 를 통과하지 못하면 false 가 되고 트래픽에서 빠진다
    ...
    nodeName: worker-1
```

### 실제로 접속하기

```bash
kubectl run tmp -n guide-net --rm -i --restart=Never --image=busybox:1.36 -- wget -qO- --timeout=3 http://menu-svc:8080
```

```text
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
pod "tmp" deleted
```

### 실험 ① — 파드를 셀렉터 밖으로 빼 보기

파드 하나의 레이블을 바꿉니다.

```bash
POD=$(kubectl get pods -n guide-net -l app=menu -o jsonpath='{.items[0].metadata.name}')
kubectl label pod $POD -n guide-net app=menu-debug --overwrite
kubectl get pods -n guide-net -L app
kubectl get endpoints menu-svc -n guide-net
```

```text
NAME                    READY   STATUS    APP
menu-7c9d6b8f5d-h8x7m   1/1     Running   menu-debug   ← 떨어져 나간 파드 (아직 살아 있다)
menu-7c9d6b8f5d-r6j9c   1/1     Running   menu
menu-7c9d6b8f5d-wz5tn   1/1     Running   menu
menu-7c9d6b8f5d-xk3v8   1/1     Running   menu         ← ReplicaSet 이 새로 만든 파드

NAME       ENDPOINTS                                AGE
menu-svc   10.0.1.88:80,10.0.2.57:80,10.0.1.94:80   2m
```

관찰 포인트: Service 와 ReplicaSet 은 **둘 다 레이블만 봅니다**. 레이블이 바뀐 파드는 Endpoints 에서 빠지고, ReplicaSet 은 수가 모자라니 새로 만듭니다.
장애 파드를 트래픽에서만 빼고 살려 둔 채 조사할 때 실제로 쓰는 방법입니다.

```bash
kubectl delete pod $POD -n guide-net      # 확인했으면 지운다
```

### 실험 ② — 셀렉터 오타

`menu-typo.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: menu-typo
  namespace: guide-net
spec:
  selector:
    app: menus            # ← 's' 하나가 더 붙었다. 이 레이블의 파드는 없다
  ports:
    - port: 8080
      targetPort: 80
```

```bash
kubectl apply -f menu-typo.yaml
kubectl get svc,endpoints menu-typo -n guide-net
```

```text
NAME                TYPE        CLUSTER-IP      PORT(S)
service/menu-typo   ClusterIP   10.109.14.201   8080/TCP     ← Service 는 멀쩡히 만들어진다

NAME                  ENDPOINTS   AGE
endpoints/menu-typo   <none>      5s                         ← 대상이 없다
```

```bash
kubectl run tmp -n guide-net --rm -i --restart=Never --image=busybox:1.36 -- wget -qO- --timeout=3 http://menu-typo:8080
```

```text
wget: can't connect to remote host (10.109.14.201): Connection refused
```

관찰 포인트: **에러 없이 만들어지고, 접속만 안 됩니다.** Service 진단은 항상 `get endpoints` 가 비었는지부터 봅니다.

| Endpoints 가 비었다 | 원인 | 확인 |
|---------------------|------|------|
| 셀렉터 ≠ 파드 레이블 | 오타 · 다른 키 | `kubectl get pods --show-labels` 와 `describe svc` 의 Selector 대조 |
| 파드가 Ready 가 아니다 | readinessProbe 실패, CrashLoop | `kubectl get pods` 의 READY 칸 |
| 다른 네임스페이스 | Service 와 파드가 다른 ns | Service 는 **같은 네임스페이스의 파드만** 고른다 |

```bash
kubectl delete -f menu-typo.yaml
```

---

## 3) port · targetPort · nodePort — 세 포트 구분

필드 이름이 헷갈리면 `kubectl explain` 으로 바로 확인합니다. 시험장에서도 됩니다.

```bash
kubectl explain service.spec.ports
```

```text
FIELDS:
  appProtocol   <string>
  name          <string>
  nodePort      <integer>
    The port on each node on which this service is exposed when type is
    NodePort or LoadBalancer. ...
  port          <integer> -required-
    The port that will be exposed by this service.
  protocol      <string>
  targetPort    <IntOrString>
    Number or name of the port to access on the pods targeted by the service.
    ... If this is not specified, the value of the 'port' field is used ...
```

| 필드 | 누가 여는 포트인가 | 누가 쓰나 | 이 페이지 값 |
|------|-------------------|-----------|-------------|
| `port` | Service (ClusterIP) | 클러스터 안의 클라이언트 → `menu-svc:8080` | 8080 |
| `targetPort` | 파드 컨테이너 | Service → 파드 (Endpoints 에 찍히는 포트) | 80 |
| `nodePort` | **모든 노드** | 클러스터 밖 → `<노드IP>:<nodePort>` (30000–32767) | 4) 에서 31234 |

!!! warning "`targetPort` 를 안 적으면 `port` 와 같은 값이 됩니다"
    `port: 8080` 만 적고 `targetPort` 를 빼먹으면 파드의 8080 으로 보냅니다. nginx 는 80 을 들으니 접속이 안 됩니다.

### 실험 — targetPort 를 틀리면

```bash
kubectl expose deployment menu -n guide-net --name=menu-wrongport --port=8080     # targetPort 생략 → 8080
kubectl get endpoints menu-wrongport -n guide-net
kubectl run tmp -n guide-net --rm -i --restart=Never --image=busybox:1.36 -- wget -qO- --timeout=3 http://menu-wrongport:8080
```

```text
NAME             ENDPOINTS                                      AGE
menu-wrongport   10.0.1.88:8080,10.0.2.57:8080,10.0.1.94:8080   3s   ← 차 있다!

wget: can't connect to remote host (10.97.203.5): Connection refused
```

관찰 포인트: 셀렉터 오타와 증상은 같지만(**Connection refused**) **Endpoints 는 차 있습니다.** 여기서 원인이 갈립니다.

| Endpoints | 접속 | 의심할 곳 |
|-----------|------|-----------|
| 비어 있다 | 실패 | 셀렉터 · Ready · 네임스페이스 |
| 차 있다 | refused | **targetPort** (파드가 그 포트를 안 듣는다) |
| 차 있다 | timeout | NetworkPolicy · CNI (8) 과 10 에서) |

```bash
kubectl delete svc menu-wrongport -n guide-net
```

---

## 4) NodePort — 모든 노드의 같은 포트로 연다

`kubectl expose` 는 `nodePort` **번호를 정할 수 없습니다.** 번호를 고정하려면 YAML 로 뽑아서 고칩니다.

```bash
kubectl expose deployment menu -n guide-net --name=menu-np --type=NodePort --port=80 --target-port=80 $do > menu-np.yaml
```

`menu-np.yaml` 을 열어 `nodePort` 한 줄을 넣습니다.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: menu
  name: menu-np
  namespace: guide-net
spec:
  type: NodePort              # ← 타입
  selector:
    app: menu
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
      nodePort: 31234         # ← 직접 추가. 30000–32767 범위, 클러스터 안에서 겹치면 거부된다
```

```bash
kubectl apply -f menu-np.yaml
kubectl get svc menu-np -n guide-net
```

```text
NAME      TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
menu-np   NodePort   10.110.7.66    <none>        80:31234/TCP   4s
```

`80:31234` 는 "Service 포트 80, 노드 포트 31234" 라는 뜻입니다. **NodePort 도 ClusterIP 를 함께 받습니다.**

### 세 노드 모두로 접속해 보기

컨트롤플레인 노드에서 실행합니다.

```bash
for ip in 192.168.56.10 192.168.56.11 192.168.56.12; do
  printf "%s -> " $ip; curl -s -o /dev/null -w '%{http_code}\n' --max-time 3 http://$ip:31234
done
```

```text
192.168.56.10 -> 200
192.168.56.11 -> 200
192.168.56.12 -> 200
```

관찰 포인트: menu 파드가 **하나도 없는 control-plane 의 IP 로도 200** 이 옵니다. 노드에 들어온 패킷을 커널 규칙이 파드가 있는 노드로 넘겨 주기 때문입니다.

### 실험 — 이미 쓰고 있는 nodePort

```bash
kubectl create service nodeport dup -n guide-net --tcp=80:80 --node-port=31234
```

```text
The Service "dup" is invalid: spec.ports[0].nodePort: Invalid value: 31234: provided port is already allocated
```

!!! tip "시험 팁 — `kubectl create service nodeport` 에는 `--node-port` 가 있습니다"
    `expose` 대신 `kubectl create service nodeport <이름> --tcp=<port>:<targetPort> --node-port=<번호>` 로 한 줄에 만들 수 있습니다.
    다만 **셀렉터가 `app=<Service 이름>` 으로 자동 설정**됩니다. 파드 레이블과 Service 이름이 다르면 Endpoints 가 비므로,
    이름이 다를 때는 `$do` 로 뽑아 셀렉터를 고치거나 위처럼 `expose` + YAML 을 씁니다.

---

## 5) LoadBalancer 는 Pending, Headless 는 IP 가 없다

### LoadBalancer — 베어메탈에서는 Pending 이 정상

```bash
kubectl expose deployment menu -n guide-net --name=menu-lb --type=LoadBalancer --port=80 --target-port=80
kubectl get svc menu-lb -n guide-net
```

```text
NAME      TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
menu-lb   LoadBalancer   10.98.144.73    <pending>     80:31877/TCP   5s
```

관찰 포인트: `EXTERNAL-IP` 가 `<pending>` 에서 멈춥니다. 외부 IP 를 붙여 줄 **클라우드 로드밸런서가 없기 때문**입니다(고장이 아닙니다).
그래도 `80:31877` 처럼 **NodePort 는 이미 열려 있습니다.** LoadBalancer = NodePort + 외부 LB 라서입니다.

```bash
curl -s -o /dev/null -w '%{http_code}\n' http://192.168.56.11:31877      # 내 포트 번호로
```

### Headless — ClusterIP 가 None

```bash
kubectl expose deployment menu -n guide-net --name=menu-headless --port=80 --cluster-ip=None
kubectl get svc menu-headless -n guide-net
```

```text
NAME            TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
menu-headless   ClusterIP   None         <none>        80/TCP    3s
```

부하 분산용 가상 IP 가 없고, DNS 가 **파드 IP 들을 직접** 돌려줍니다(6) 에서 확인). StatefulSet 이 파드마다 고정 이름을 받을 때 씁니다.

| 타입 | ClusterIP | 밖에서 접속 | 이 클러스터에서 |
|------|-----------|-------------|----------------|
| ClusterIP | 있음 | 안 됨 | 파드끼리 부르는 기본 |
| NodePort | 있음 | `<노드IP>:30000–32767` | 실습·시험에서 외부 노출 |
| LoadBalancer | 있음 (+NodePort) | 외부 LB IP | `EXTERNAL-IP <pending>` |
| Headless (`clusterIP: None`) | **없음** | 안 됨 | DNS 가 파드 IP 목록을 준다 |
| ExternalName | 없음 | — | 셀렉터 없이 DNS CNAME 만 만든다 |

```bash
kubectl delete svc menu-lb -n guide-net
```

---

## 6) DNS — 이름으로 부른다

### 파드의 resolv.conf

DNS 확인용 파드를 하나 띄워 둡니다.

```bash
kubectl run dnsbox -n guide-net --image=busybox:1.36 --command -- sleep 3600
kubectl exec dnsbox -n guide-net -- cat /etc/resolv.conf
kubectl get svc kube-dns -n kube-system
```

```text
search guide-net.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.96.0.10
options ndots:5

NAME       TYPE        CLUSTER-IP   PORT(S)                  AGE
kube-dns   ClusterIP   10.96.0.10   53/UDP,53/TCP,9153/TCP   3d
```

관찰 포인트: `nameserver` 는 **CoreDNS 의 Service IP**(이름은 역사적 이유로 `kube-dns`)이고,
`search` 첫 줄이 **내 네임스페이스**입니다. 그래서 같은 네임스페이스의 Service 는 짧은 이름으로 부를 수 있습니다.

### 이름 네 가지로 조회하기

```bash
kubectl exec dnsbox -n guide-net -- nslookup menu-svc
kubectl exec dnsbox -n guide-net -- nslookup menu-svc.guide-net.svc.cluster.local
```

```text
Server:		10.96.0.10
Address:	10.96.0.10:53

Name:	menu-svc.guide-net.svc.cluster.local
Address: 10.104.62.18                ← menu-svc 의 CLUSTER-IP 와 같다
```

다른 네임스페이스의 Service 를 짧은 이름으로 불러 봅니다.

```bash
kubectl exec dnsbox -n guide-net -- nslookup kube-dns                 # 실패
kubectl exec dnsbox -n guide-net -- nslookup kube-dns.kube-system     # 성공
```

```text
** server can't find kube-dns.guide-net.svc.cluster.local: NXDOMAIN
...
Name:	kube-dns.kube-system.svc.cluster.local
Address: 10.96.0.10
```

| 부르는 이름 | 같은 ns 에서 | 다른 ns 에서 |
|-------------|-------------|-------------|
| `menu-svc` | ✅ | ❌ (내 ns 에서 찾는다) |
| `menu-svc.guide-net` | ✅ | ✅ |
| `menu-svc.guide-net.svc` | ✅ | ✅ |
| `menu-svc.guide-net.svc.cluster.local` (FQDN) | ✅ | ✅ |

### Headless 는 파드 IP 들을 돌려준다

```bash
kubectl exec dnsbox -n guide-net -- nslookup menu-headless
```

```text
Name:	menu-headless.guide-net.svc.cluster.local
Address: 10.0.1.88
Address: 10.0.2.57
Address: 10.0.1.94
```

관찰 포인트: `menu-svc` 는 IP **하나**(ClusterIP), `menu-headless` 는 **파드 수만큼**입니다. `kubectl get endpoints menu-headless -n guide-net` 의 IP 와 같습니다.

### 실험 — CoreDNS 가 없으면

CoreDNS 를 잠깐 0 개로 줄여 봅니다. **실습 클러스터에서만** 하세요.

```bash
kubectl scale deployment coredns -n kube-system --replicas=0
kubectl exec dnsbox -n guide-net -- nslookup menu-svc
```

```text
;; connection timed out; no servers could be reached
```

```bash
kubectl exec dnsbox -n guide-net -- wget -qO- --timeout=3 http://10.104.62.18:8080 | head -4   # IP 로는 된다
kubectl scale deployment coredns -n kube-system --replicas=2                                    # 반드시 되돌린다
kubectl rollout status deployment/coredns -n kube-system
```

관찰 포인트: **이름 해석만 죽고 IP 통신은 살아 있습니다.** "이름으로는 안 되는데 IP 로는 된다" 면 DNS(CoreDNS · Egress 53) 를 봅니다.

!!! tip "시험 팁"
    DNS 문제는 대개 `nslookup <svc>` 결과를 파일로 남기라고 합니다. `kubectl exec <pod> -- nslookup <svc> > /경로/파일` 처럼 **리다이렉트는 exec 바깥**에서 합니다.

---

## 7) 누가 실제로 넘기는가 — kube-proxy / Cilium, 그리고 port-forward

Service 는 **가상 IP 와 목록**일 뿐입니다. 패킷을 실제 파드로 바꿔 보내는 것은 각 노드의 규칙입니다.

```bash
kubectl get ds -n kube-system
kubectl get cm kube-proxy -n kube-system -o jsonpath='{.data.config\.conf}' 2>/dev/null | grep -E '^mode'
```

```text
NAME           DESIRED   CURRENT   READY   ...
cilium         3         3         3
kube-proxy     3         3         3       ← 이 과정의 클러스터(Cilium 기본 모드)에는 있다
mode: ""                                   ← 빈 값이면 iptables 모드
```

=== "kube-proxy 가 있을 때 (iptables)"
    ```bash
    sudo iptables -t nat -L KUBE-SERVICES -n | grep menu-svc
    ```
    ```text
    KUBE-SVC-ABCD...  tcp  --  0.0.0.0/0  10.104.62.18  /* guide-net/menu-svc cluster IP */ tcp dpt:8080
    ```
=== "Cilium 이 대신할 때 (eBPF)"
    ```bash
    kubectl -n kube-system exec ds/cilium -c cilium-agent -- cilium-dbg service list | grep -A2 10.104.62.18
    ```
    ```text
    27   10.104.62.18:8080   ClusterIP   1 => 10.0.1.88:80 (active)
                                         2 => 10.0.2.57:80 (active)
                                         3 => 10.0.1.94:80 (active)
    ```
    Cilium 버전이 낮으면 명령 이름이 `cilium-dbg` 가 아니라 `cilium` 입니다.

관찰 포인트: 어느 쪽이든 **ClusterIP:8080 → 파드IP:80 목록**이 노드에 심어져 있습니다. 이 목록의 원본이 2) 의 EndpointSlice 입니다.
ClusterIP 는 포트 규칙만 있는 가상 IP 라서 `ping` 에는 보통 응답하지 않습니다. 통신 확인은 `wget`/`curl` 로 합니다.

| 부품 | 하는 일 |
|------|---------|
| EndpointSlice 컨트롤러 | 셀렉터에 맞고 Ready 인 파드 IP 목록을 만든다 |
| kube-proxy (또는 Cilium) | 그 목록을 노드의 iptables/IPVS/eBPF 규칙으로 심는다 |
| 커널 | 규칙대로 목적지를 바꿔 전달한다 |
| CoreDNS | 이름 → ClusterIP (Headless 면 파드 IP) |
| CNI (Cilium) | 파드 IP 할당 · 노드 간 통신 · NetworkPolicy 집행 |

### port-forward — 내 터미널에서 잠깐 붙기

```bash
kubectl port-forward -n guide-net svc/menu-svc 18080:8080 &
sleep 1; curl -s http://127.0.0.1:18080 | grep title
kill %1
```

```text
Forwarding from 127.0.0.1:18080 -> 80
<title>Welcome to nginx!</title>
```

관찰 포인트: `-> 80` 입니다. port-forward 는 Service 의 셀렉터로 **파드 하나를 골라 그 파드의 targetPort 로 바로** 붙습니다. 부하 분산이 없고 디버깅용입니다.

---

## 8) NetworkPolicy 미리 보기 — 한 번만 막아 본다

지금까지는 **모든 파드가 menu 에 닿습니다.** 두 클라이언트를 만들고, 한쪽만 허용해 봅니다. 원리와 함정은 [10강 · NetworkPolicy](10-networkpolicy.md) 에서 자세히 다룹니다.

```bash
kubectl run cashier -n guide-net --image=busybox:1.36 --labels=role=cashier --command -- sleep 3600
kubectl run guest   -n guide-net --image=busybox:1.36 --labels=role=guest   --command -- sleep 3600
```

`menu-from-cashier.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: menu-from-cashier
  namespace: guide-net
spec:
  podSelector:
    matchLabels:
      app: menu               # ← 이 정책이 '지키는' 파드
  policyTypes:
    - Ingress                 # ← 들어오는 방향만 제한
  ingress:
    - from:
        - podSelector:
            matchLabels:
              role: cashier   # ← 이 파드에서 오는 것만
      ports:
        - protocol: TCP
          port: 80            # ← 파드 포트(targetPort). Service 포트 8080 이 아니다
```

```bash
kubectl apply -f menu-from-cashier.yaml
kubectl exec cashier -n guide-net -- wget -qO- --timeout=3 http://menu-svc:8080 | grep title
kubectl exec guest   -n guide-net -- wget -qO- --timeout=3 http://menu-svc:8080
```

```text
<title>Welcome to nginx!</title>
wget: download timed out                ← refused 가 아니라 timeout: 패킷이 조용히 버려진다
```

### 실험 — 정책에 Service 포트를 적으면

허용 포트를 Service 포트(8080)로 바꿔 봅니다.

```bash
kubectl patch networkpolicy menu-from-cashier -n guide-net --type=json \
  -p '[{"op":"replace","path":"/spec/ingress/0/ports/0/port","value":8080}]'
kubectl describe networkpolicy menu-from-cashier -n guide-net | grep 'To Port'
kubectl exec cashier -n guide-net -- wget -qO- --timeout=3 http://menu-svc:8080
```

```text
    To Port: 8080/TCP
wget: download timed out                ← 허용된 cashier 까지 막혔다
```

관찰 포인트: 정책은 Service 를 모릅니다. 패킷이 파드에 도착할 때의 포트(**targetPort 80**)로 판단합니다. 되돌려 둡니다.

```bash
kubectl apply -f menu-from-cashier.yaml
```

---

## 시험에서 빠르게

```bash
# Service
kubectl expose deploy <d> --name=<svc> --port=<p> --target-port=<tp>                 # ClusterIP
kubectl expose deploy <d> --name=<svc> --port=<p> --type=NodePort $do > s.yaml       # nodePort 는 YAML 에 추가
kubectl create service nodeport <이름=앱레이블> --tcp=<p>:<tp> --node-port=<np>       # 셀렉터 app=<이름>
kubectl expose deploy <d> --name=<svc> --port=<p> --cluster-ip=None                  # Headless

# 진단 순서
kubectl get svc,endpoints <svc> -n <ns>          # ① 비었나 차 있나
kubectl get pods -n <ns> --show-labels           # ② 셀렉터와 맞나 · READY 인가
kubectl describe svc <svc> -n <ns>               # ③ Selector · TargetPort
kubectl run tmp -n <ns> --rm -i --restart=Never --image=busybox:1.36 -- wget -qO- --timeout=3 http://<svc>:<p>

# DNS
kubectl exec <pod> -- nslookup <svc>.<ns>.svc.cluster.local
kubectl exec <pod> -- cat /etc/resolv.conf
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

---

## 정리

```bash
kubectl delete namespace guide-net
rm -f menu-typo.yaml menu-np.yaml menu-from-cashier.yaml
kubectl get deployment coredns -n kube-system      # READY 가 원래 수(보통 2/2)로 돌아와 있는지 확인
```

---

## 확인 질문

??? question "1. Service 는 만들어졌는데 `get endpoints` 가 `<none>` 입니다. 원인 후보 셋은?"
    ① 셀렉터와 파드 레이블 불일치 ② 파드가 Ready 가 아님 ③ Service 와 파드가 다른 네임스페이스.
    `kubectl get pods --show-labels` 와 `describe svc` 의 Selector 를 대조하는 것이 가장 빠릅니다.

??? question "2. Endpoints 는 차 있는데 `Connection refused` 가 납니다. 무엇을 의심하나요?"
    `targetPort` 입니다. Endpoints 에 찍힌 포트를 파드 컨테이너가 실제로 듣고 있는지 확인합니다.
    refused 가 아니라 **timeout** 이라면 NetworkPolicy 나 CNI 쪽입니다.

??? question "3. `kubectl expose` 로 nodePort 번호를 지정할 수 있나요?"
    없습니다. `$do` 로 YAML 을 뽑아 `spec.ports[].nodePort` 를 추가하거나,
    `kubectl create service nodeport <이름> --tcp=... --node-port=...` 을 씁니다(이때 셀렉터는 `app=<이름>`).

??? question "4. NodePort Service 에 파드가 없는 노드의 IP 로 접속하면?"
    됩니다. 모든 노드에 같은 포트가 열리고, 노드의 규칙(kube-proxy/Cilium)이 파드가 있는 노드로 넘겨 줍니다.

??? question "5. 베어메탈 클러스터에서 LoadBalancer Service 의 EXTERNAL-IP 가 `<pending>` 입니다. 고장인가요?"
    아닙니다. 외부 IP 를 붙여 줄 로드밸런서 구현이 없을 뿐입니다. NodePort 는 이미 열려 있어 `<노드IP>:<nodePort>` 로 접속됩니다.

??? question "6. `other` 네임스페이스의 파드가 `guide-net` 의 `menu-svc` 를 부르려면 어떤 이름을 써야 하나요?"
    `menu-svc.guide-net` (또는 FQDN `menu-svc.guide-net.svc.cluster.local`). 짧은 이름 `menu-svc` 는 `other` 네임스페이스에서 찾으므로 실패합니다.

??? question "7. 일반 Service 와 Headless Service 를 nslookup 하면 결과가 어떻게 다른가요?"
    일반 Service 는 ClusterIP 하나, Headless 는 Ready 인 파드 IP 들이 모두 나옵니다.

??? question "8. NetworkPolicy 의 `ports.port` 에는 Service 의 port 와 targetPort 중 무엇을 적나요?"
    targetPort(파드가 듣는 포트)입니다. 정책은 파드에 도착하는 패킷을 기준으로 판단합니다.

---

## 다음 단계 — 채점 실습

개념이 잡혔으면 시험 형식으로 풀어 봅니다. 이번에는 **답이 없습니다.** 채점은 실제로 임시 파드와 노드 IP 로 접속까지 확인합니다.

```bash
cd ~/cka-lab/09-networking
bash exam.sh start
```

| 문제 | 이 페이지에서 연습한 것 |
|------|------------------------|
| Q1 Deployment + ClusterIP Service | 1) 2) — expose, Endpoints 확인, 임시 파드로 접속 |
| Q2 NodePort Service (포트 고정) | 3) 4) — `$do` 로 뽑아 nodePort 추가 |
| Q3 NetworkPolicy — frontend 에서만 backend 로 | 8) (자세한 원리는 [10강](10-networkpolicy.md)) |
| Q4 파드 안에서 클러스터 DNS 확인 | 6) — resolv.conf, 짧은 이름과 FQDN |
