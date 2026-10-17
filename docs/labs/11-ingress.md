# 8강 · Ingress → Gateway API

> 영상: **8강 밖에서 이름으로 들어오기 — Ingress → Gateway API** · 이어서 풀 채점 세트: `08-ingress`

## 실습 목표

- Ingress 오브젝트는 **규칙**일 뿐이고, 실제로 트래픽을 받는 것은 **컨트롤러 파드**라는 것을 확인한다.
- 경로 기반 라우팅에서 **Prefix 와 Exact** 가 어떤 요청을 잡고 놓치는지 표로 확인한다.
- 호스트 기반 라우팅을 **DNS 없이** `-H 'Host: ...'` 와 `--resolve` 로 시험한다.
- openssl 로 만든 자체 서명 인증서를 **TLS Secret → Ingress** 로 붙이고, 어떤 인증서가 나오는지 확인한다.
- 404 · 503 · 504 가 **각각 어디의 문제인지** 직접 만들어 보고 구분한다.
- NetworkPolicy 로 잠근 네임스페이스에 **Ingress 컨트롤러만 들여보내는** 조합을 만든다.
- 기존 Ingress(호스트 + TLS + 경로)를 **Gateway + HTTPRoute 로 옮기고** Ingress 를 지운다. 필드가 어디로 가는지 표로 정리한다.

---

## 0) 준비 — 컨트롤러 확인과 접속 주소

```bash
kubectl create namespace guide-ingress
```

### 컨트롤러가 있는가

이 과정의 클러스터는 **ingress-nginx** 컨트롤러를 씁니다. 먼저 떠 있는지, 어떤 IngressClass 가 있는지 봅니다.

```bash
kubectl get pods -n ingress-nginx
kubectl get ingressclass
kubectl get svc -n ingress-nginx
```

```text
NAME                                        READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-8xk2p        0/1     Completed   0          3d
ingress-nginx-admission-patch-q7m4t         0/1     Completed   0          3d
ingress-nginx-controller-6b8d9c7f5d-2jx9w   1/1     Running     0          3d

NAME    CONTROLLER             PARAMETERS   AGE
nginx   k8s.io/ingress-nginx   <none>       3d

NAME                                 TYPE        CLUSTER-IP     PORT(S)                      AGE
ingress-nginx-controller             NodePort    10.99.215.40   80:31692/TCP,443:32507/TCP   3d
ingress-nginx-controller-admission   ClusterIP   10.108.33.12   443/TCP                      3d
```

관찰 포인트: 컨트롤러 자체도 **NodePort Service** 뒤에 있습니다. 베어메탈이라 LoadBalancer 가 없으니, 밖에서는 `<노드IP>:<NodePort>` 로 들어옵니다([5강 의 NodePort](09-networking.md#4-nodeport)).
포트 번호는 설치할 때 정해지므로 **내 클러스터의 값**을 씁니다.

??? note "컨트롤러가 없다면 — 설치 (베어메탈용 매니페스트)"
    공식 저장소의 베어메탈용 매니페스트는 컨트롤러를 NodePort Service 로 노출합니다. 버전은 강사 안내에 맞추세요.
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/baremetal/deploy.yaml
    kubectl wait -n ingress-nginx --for=condition=Ready pod -l app.kubernetes.io/component=controller --timeout=180s
    ```
    시험 클러스터에는 대개 컨트롤러가 미리 설치돼 있습니다. 시험에서는 **설치하지 말고 `get ingressclass` 로 이름만 확인**합니다.

### 접속 주소를 변수로

이 페이지의 curl 은 모두 **컨트롤플레인 노드에서** 아래 변수로 합니다.

```bash
NODE_IP=192.168.56.11
HTTP_PORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.spec.ports[?(@.name=="http")].nodePort}')
HTTPS_PORT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.spec.ports[?(@.name=="https")].nodePort}')
echo "$NODE_IP  http:$HTTP_PORT  https:$HTTPS_PORT"
curl -s -o /dev/null -w '%{http_code}\n' http://$NODE_IP:$HTTP_PORT/
```

```text
192.168.56.11  http:31692  https:32507
404
```

관찰 포인트: 규칙이 하나도 없는데 **404 가 옵니다.** 컨트롤러가 살아서 "맞는 규칙이 없다" 고 답한 것입니다. 연결 자체가 안 되면(`000`) 컨트롤러나 NodePort 부터 봅니다.

!!! note "컨트롤러가 hostNetwork 로 80/443 을 직접 여는 클러스터라면"
    `kubectl get svc -n ingress-nginx` 에 NodePort 가 없고 컨트롤러 파드가 노드의 80 을 쓰는 구성도 있습니다.
    그때는 `HTTP_PORT=80`, `HTTPS_PORT=443` 으로 두고, `NODE_IP` 는 컨트롤러 파드가 떠 있는 노드 IP 로 합니다.

반복해서 쓸 확인 함수를 하나 만듭니다. **상태 코드와 응답 첫 줄**을 한 줄로 보여 줍니다.

```bash
hit() {   # hit <경로> [curl 추가 옵션...]
  local p=$1; shift
  local code=$(curl -s -o /tmp/hit.out -w '%{http_code}' --max-time 20 "$@" "http://$NODE_IP:$HTTP_PORT$p")
  printf '%-18s %s  %s\n' "$p" "$code" "$(head -1 /tmp/hit.out | cut -c1-72)"
}
```

---

## 1) 백엔드 두 개 — 받은 요청을 그대로 말해 주는 nginx

라우팅이 어디로 갔는지 눈에 보이도록, **어느 파드가 어떤 경로를 받았는지** 응답하는 nginx 를 띄웁니다.

`backends.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: echo-conf
  namespace: guide-ingress
data:
  default.conf: |
    server {
      listen 80;
      default_type text/plain;
      location / {
        # $hostname = 파드 이름, $request_uri = 백엔드가 실제로 받은 경로
        return 200 "served-by=$hostname uri=$request_uri host=$host\n";
      }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: catalog
  namespace: guide-ingress
spec:
  replicas: 2
  selector:
    matchLabels: { app: catalog }
  template:
    metadata:
      labels: { app: catalog }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports: [{ containerPort: 80 }]
          volumeMounts:
            - name: conf
              mountPath: /etc/nginx/conf.d     # ← 기본 default.conf 를 덮어쓴다
      volumes:
        - name: conf
          configMap: { name: echo-conf }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: board
  namespace: guide-ingress
spec:
  replicas: 1
  selector:
    matchLabels: { app: board }
  template:
    metadata:
      labels: { app: board }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports: [{ containerPort: 80 }]
          volumeMounts:
            - name: conf
              mountPath: /etc/nginx/conf.d
      volumes:
        - name: conf
          configMap: { name: echo-conf }
```

```bash
kubectl apply -f backends.yaml
kubectl expose deployment catalog -n guide-ingress --name=catalog-svc --port=80
kubectl expose deployment board   -n guide-ingress --name=board-svc   --port=80
kubectl rollout status deployment/catalog -n guide-ingress
kubectl get svc,endpoints -n guide-ingress
```

```text
NAME                  TYPE        CLUSTER-IP      PORT(S)
service/board-svc     ClusterIP   10.105.19.230   80/TCP
service/catalog-svc   ClusterIP   10.97.142.8     80/TCP

NAME                    ENDPOINTS
endpoints/board-svc     10.0.2.44:80
endpoints/catalog-svc   10.0.1.31:80,10.0.2.18:80
```

Ingress 없이 클러스터 안에서 먼저 확인합니다. **Ingress 문제의 절반은 사실 Service 문제**이므로, 백엔드가 맞는지부터 확정해 둡니다.

```bash
kubectl run tmp -n guide-ingress --rm -i --restart=Never --image=busybox:1.36 -- wget -qO- --timeout=3 http://catalog-svc/hello
```

```text
served-by=catalog-5f7b9c6d4-k2x8p uri=/hello host=catalog-svc
pod "tmp" deleted
```

---

## 2) 규칙과 집행 — IngressClass

Ingress 의 필드 구조는 깊습니다. 헷갈리면 `kubectl explain` 으로 한 단계씩 내려갑니다.

```bash
kubectl explain ingress.spec
kubectl explain ingress.spec.rules.http.paths.backend.service
```

```text
FIELDS:
  defaultBackend      <IngressBackend>
  ingressClassName    <string>
  rules               <[]IngressRule>
  tls                 <[]IngressTLS>
...
FIELDS:
  name  <string> -required-
  port  <ServiceBackendPort>
```

IngressClass 는 "이 Ingress 를 **어느 컨트롤러가** 처리할지" 를 정합니다.

```bash
kubectl get ingressclass nginx -o yaml | grep -E 'controller:|is-default-class'
```

```text
  controller: k8s.io/ingress-nginx
```

`is-default-class` 어노테이션이 **없으면** 기본 클래스가 아닙니다. 이때 `ingressClassName` 을 빼먹은 Ingress 는 어떤 컨트롤러도 처리하지 않습니다.

### 실험 — 클래스 이름이 틀리면

`probe-ing.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: probe-ing
  namespace: guide-ingress
spec:
  ingressClassName: nginx-typo     # ← 존재하지 않는 클래스
  rules:
    - http:
        paths:
          - path: /probe
            pathType: Prefix
            backend:
              service:
                name: catalog-svc
                port:
                  number: 80
```

```bash
kubectl apply -f probe-ing.yaml
sleep 30; kubectl get ingress probe-ing -n guide-ingress
hit /probe
```

```text
ingress.networking.k8s.io/probe-ing created                     ← 에러 없이 만들어진다
NAME        CLASS        HOSTS   ADDRESS   PORTS   AGE
probe-ing   nginx-typo   *                 80      30s          ← ADDRESS 가 비어 있다
/probe             404  <html>                                  ← 컨트롤러가 이 규칙을 모른다
```

클래스를 고쳐 봅니다.

```bash
kubectl patch ingress probe-ing -n guide-ingress --type=merge -p '{"spec":{"ingressClassName":"nginx"}}'
sleep 30; kubectl get ingress probe-ing -n guide-ingress
hit /probe
```

```text
NAME        CLASS   HOSTS   ADDRESS         PORTS   AGE
probe-ing   nginx   *       192.168.56.11   80      70s     ← 컨트롤러가 가져가면 ADDRESS 가 붙는다
/probe             200  served-by=catalog-5f7b9c6d4-k2x8p uri=/probe host=192.168.56.11
```

관찰 포인트: Ingress 는 **에러 없이 만들어지고 조용히 무시됩니다.** "Ingress 를 만들었는데 안 된다" 면 `ADDRESS` 칸과 `CLASS` 칸부터 봅니다.
(`ADDRESS` 는 컨트롤러 파드가 떠 있는 노드의 IP 로, 붙기까지 수십 초 걸릴 수 있습니다.)

```bash
kubectl delete -f probe-ing.yaml
```

---

## 3) 경로 기반 — Prefix 와 Exact

`shop-paths.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-paths
  namespace: guide-ingress
spec:
  ingressClassName: nginx
  rules:
    - http:                         # ← host 가 없다 = 모든 호스트(*)
        paths:
          - path: /catalog
            pathType: Prefix        # ← /catalog, /catalog/..., 경로 '요소' 단위로 앞부분 일치
            backend:
              service:
                name: catalog-svc
                port:
                  number: 80
          - path: /board
            pathType: Exact         # ← 정확히 /board 만
            backend:
              service:
                name: board-svc
                port:
                  number: 80
```

```bash
kubectl apply -f shop-paths.yaml
for p in /catalog /catalog/ /catalog/items /catalogue /board /board/ /board/list /; do hit $p; done
```

```text
/catalog           200  served-by=catalog-5f7b9c6d4-k2x8p uri=/catalog host=192.168.56.11
/catalog/          200  served-by=catalog-5f7b9c6d4-m9q4t uri=/catalog/ host=192.168.56.11
/catalog/items     200  served-by=catalog-5f7b9c6d4-k2x8p uri=/catalog/items host=192.168.56.11
/catalogue         404  <html>          ← 글자는 겹치지만 경로 '요소'가 다르다
/board             200  served-by=board-7c8d5b9f6-x4w2n uri=/board host=192.168.56.11
/board/            404  <html>          ← Exact 는 끝의 / 하나도 다르게 본다
/board/list        404  <html>
/                  404  <html>
```

관찰 포인트: 백엔드는 **경로를 그대로 받습니다**(`uri=/catalog/items`). Ingress 는 경로를 골라 보낼 뿐, 떼어 내지 않습니다(떼어 내기는 6) 에서).
`served-by` 가 번갈아 바뀌는 것은 catalog 파드 2개에 나눠 보내기 때문입니다.

| pathType | `/catalog` 규칙이 잡는 것 | 놓치는 것 | 언제 쓰나 |
|----------|---------------------------|-----------|-----------|
| `Prefix` | `/catalog` · `/catalog/` · `/catalog/a/b` | `/catalogue` | 대부분 (기본 선택) |
| `Exact` | `/catalog` 만 | `/catalog/` · `/catalog/a` | 특정 엔드포인트 하나 |
| `ImplementationSpecific` | 컨트롤러 마음대로 (ingress-nginx 는 정규식 가능) | — | 6) 의 rewrite 처럼 특수한 경우 |

!!! warning "pathType 은 필수입니다"
    `pathType` 을 빼면 API 서버가 생성을 거부합니다. `kubectl create ingress` 를 쓰면 알아서 들어갑니다.

### 같은 것을 명령 한 줄로

```bash
kubectl create ingress shop-paths -n guide-ingress --class=nginx \
  --rule="/catalog*=catalog-svc:80" --rule="/board=board-svc:80" $do | grep -E 'path:|pathType'
```

```text
      - path: /catalog
        pathType: Prefix       ← 경로 끝의 * 가 Prefix
      - path: /board
        pathType: Exact        ← * 가 없으면 Exact
```

!!! tip "시험 팁"
    `--rule="<host>/<path>*=<svc>:<port>"` 에서 **`*` 하나가 Prefix/Exact 를 가릅니다.** 문제가 Prefix 를 요구하면 `*` 를 꼭 붙이고,
    만든 뒤 `kubectl get ingress <이름> -o yaml | grep pathType` 으로 확인합니다.

---

## 4) 호스트 기반 — 그리고 규칙에 안 맞는 요청

`shop-hosts.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-hosts
  namespace: guide-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: catalog.guide.local     # ← Host 헤더가 이 값일 때만
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: catalog-svc, port: { number: 80 } }
    - host: board.guide.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: board-svc, port: { number: 80 } }
```

```bash
kubectl apply -f shop-hosts.yaml
kubectl get ingress -n guide-ingress
```

```text
NAME         CLASS   HOSTS                                   ADDRESS         PORTS   AGE
shop-hosts   nginx   catalog.guide.local,board.guide.local   192.168.56.11   80      20s
shop-paths   nginx   *                                       192.168.56.11   80      5m
```

DNS 가 없으니 **Host 헤더를 직접** 넣어 보냅니다.

```bash
hit /menu -H 'Host: catalog.guide.local'
hit /menu -H 'Host: board.guide.local'
hit /menu -H 'Host: unknown.guide.local'
hit /menu
```

```text
/menu              200  served-by=catalog-5f7b9c6d4-m9q4t uri=/menu host=catalog.guide.local
/menu              200  served-by=board-7c8d5b9f6-x4w2n uri=/menu host=board.guide.local
/menu              404  <html>          ← 모르는 호스트
/menu              404  <html>          ← Host 가 IP 면 host 없는 규칙(shop-paths)만 본다
```

`--resolve` 를 쓰면 **URL 에 도메인을 그대로** 적을 수 있습니다(`/etc/hosts` 를 고치지 않아도 됩니다).

```bash
curl -s --resolve catalog.guide.local:$HTTP_PORT:$NODE_IP http://catalog.guide.local:$HTTP_PORT/menu
```

```text
served-by=catalog-5f7b9c6d4-k2x8p uri=/menu host=catalog.guide.local
```

| 방법 | 명령 | 특징 |
|------|------|------|
| Host 헤더 | `curl -H 'Host: a.example' http://<IP>:<port>/` | 가장 빠르다. HTTPS 에서는 인증서 이름이 안 맞는다 |
| `--resolve` | `curl --resolve a.example:<port>:<IP> http://a.example:<port>/` | 도메인으로 접속한 것과 같다. **HTTPS(SNI)까지 정확** |
| `/etc/hosts` | `<IP> a.example` 추가 | 브라우저에서도 된다. 끝나면 지워야 한다 |

### 기본 백엔드 — 규칙에 안 맞는 요청을 받아 줄 곳

지금은 규칙에 안 맞으면 컨트롤러가 404 를 줍니다. **규칙 없이 `defaultBackend` 만 있는 Ingress** 를 만들면 그 요청을 받아 줍니다.

`shop-fallback.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-fallback
  namespace: guide-ingress
spec:
  ingressClassName: nginx
  defaultBackend:                   # ← rules 가 없다. 어디에도 안 맞는 요청이 여기로
    service:
      name: board-svc
      port:
        number: 80
```

```bash
kubectl apply -f shop-fallback.yaml
sleep 5
hit /menu -H 'Host: unknown.guide.local'
hit /anything
```

```text
/menu              200  served-by=board-7c8d5b9f6-x4w2n uri=/menu host=unknown.guide.local
/anything          200  served-by=board-7c8d5b9f6-x4w2n uri=/anything host=192.168.56.11
```

관찰 포인트: 이제 404 대신 board 가 받습니다. 뒤의 실습이 헷갈리지 않도록 지워 둡니다.

```bash
kubectl delete -f shop-fallback.yaml
```

---

## 5) TLS — 자체 서명 인증서를 붙인다

### ① 인증서 만들기

```bash
openssl req -x509 -nodes -days 30 -newkey rsa:2048 \
  -keyout guide.key -out guide.crt \
  -subj "/CN=secure.guide.local" \
  -addext "subjectAltName=DNS:secure.guide.local"
openssl x509 -in guide.crt -noout -subject -enddate -ext subjectAltName
```

```text
subject=CN = secure.guide.local
notAfter=Oct 29 06:20:11 2026 GMT
X509v3 Subject Alternative Name:
    DNS:secure.guide.local
```

| 옵션 | 뜻 |
|------|----|
| `-x509` | 서명 요청(CSR)이 아니라 **자체 서명 인증서**를 바로 만든다 |
| `-nodes` | 키에 암호를 걸지 않는다 (컨트롤러가 읽어야 하므로) |
| `-days` | 유효 기간 |
| `-newkey rsa:2048` | 새 키를 함께 만든다 |
| `-subj "/CN=..."` | 대화형 질문 없이 주체 이름을 준다 |
| `-addext subjectAltName` | 요즘 클라이언트는 CN 이 아니라 **SAN** 을 본다 (없어도 Ingress 에는 붙는다) |

### ② TLS Secret 만들기

```bash
kubectl create secret tls guide-tls -n guide-ingress --cert=guide.crt --key=guide.key
kubectl get secret guide-tls -n guide-ingress
kubectl get secret guide-tls -n guide-ingress -o jsonpath='{.data}' | grep -o '"tls\.[a-z]*"'
```

```text
NAME        TYPE                DATA   AGE
guide-tls   kubernetes.io/tls   2      3s
"tls.crt"
"tls.key"
```

!!! warning "타입이 `kubernetes.io/tls` 여야 합니다"
    `kubectl create secret generic` 으로 만들면 타입이 `Opaque` 가 됩니다. Ingress TLS 에는 `create secret tls` 를 씁니다.
    그리고 Secret 은 **Ingress 와 같은 네임스페이스**에 있어야 합니다.

### ③ Ingress 에 붙이기

`shop-tls.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: shop-tls
  namespace: guide-ingress
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - secure.guide.local        # ← 이 호스트로 온 HTTPS 요청에
      secretName: guide-tls         # ← 이 인증서를 쓴다
  rules:
    - host: secure.guide.local      # ← tls.hosts 와 같은 값이어야 한다
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: catalog-svc, port: { number: 80 } }
```

```bash
kubectl apply -f shop-tls.yaml
sleep 5
curl -s --cacert guide.crt --resolve secure.guide.local:$HTTPS_PORT:$NODE_IP https://secure.guide.local:$HTTPS_PORT/pay
```

```text
served-by=catalog-5f7b9c6d4-k2x8p uri=/pay host=secure.guide.local
```

관찰 포인트: `-k` 없이 `--cacert guide.crt` 로 **검증까지 통과**했습니다. 백엔드는 평문 HTTP 로 받습니다. HTTPS 는 컨트롤러에서 끝납니다(TLS 종료).

### 어떤 인증서가 나오는지 보기

```bash
for name in secure.guide.local other.guide.local; do
  echo "== SNI $name"
  openssl s_client -connect $NODE_IP:$HTTPS_PORT -servername $name </dev/null 2>/dev/null | openssl x509 -noout -subject
done
```

```text
== SNI secure.guide.local
subject=CN = secure.guide.local
== SNI other.guide.local
subject=O = Acme Co, CN = Kubernetes Ingress Controller Fake Certificate
```

관찰 포인트: `tls.hosts` 에 없는 이름으로 들어오면 컨트롤러의 **가짜 기본 인증서**가 나옵니다. "HTTPS 는 되는데 인증서가 이상하다" 면 `tls.hosts` · `rules.host` · Secret 이름을 대조합니다.

### HTTP 로 들어오면

```bash
curl -s -o /dev/null -w '%{http_code} -> %{redirect_url}\n' -H 'Host: secure.guide.local' http://$NODE_IP:$HTTP_PORT/pay
```

```text
308 -> https://secure.guide.local/pay
```

ingress-nginx 는 TLS 가 설정된 호스트의 HTTP 요청을 **HTTPS 로 돌려보냅니다**(끄려면 `nginx.ingress.kubernetes.io/ssl-redirect: "false"`).

!!! tip "시험 팁 — create ingress 로 TLS 까지 한 줄"
    ```bash
    kubectl create ingress <이름> -n <ns> --class=nginx --rule="<host>/*=<svc>:<port>,tls=<secret>"
    ```
    만든 뒤 `kubectl get ingress <이름> -o jsonpath='{.spec.tls}'` 로 hosts 와 secretName 을 확인합니다.

---

## 6) rewrite 어노테이션 — 경로를 떼어 내기

3) 에서 백엔드는 `/catalog/items` 를 그대로 받았습니다. 실제 앱이 `/items` 만 안다면 **앞부분을 떼어서** 보내야 합니다. 표준 필드에는 이 기능이 없어서 컨트롤러별 **어노테이션**을 씁니다.

다른 Ingress 에 영향을 주지 않도록 **별도 호스트**(`api.guide.local`)로 실험합니다.

=== "① 흔한 실수 — rewrite-target: /"
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: shop-rewrite
      namespace: guide-ingress
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /      # ← 무엇이 오든 '/' 로 바꾼다
    spec:
      ingressClassName: nginx
      rules:
        - host: api.guide.local
          http:
            paths:
              - path: /v1
                pathType: Prefix
                backend:
                  service: { name: catalog-svc, port: { number: 80 } }
    ```
=== "② 올바른 모양 — 캡처 그룹"
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: shop-rewrite
      namespace: guide-ingress
      annotations:
        nginx.ingress.kubernetes.io/rewrite-target: /$2    # ← 두 번째 괄호 부분만 남긴다
    spec:
      ingressClassName: nginx
      rules:
        - host: api.guide.local
          http:
            paths:
              - path: /v1(/|$)(.*)                         # ← 정규식
                pathType: ImplementationSpecific           # ← 정규식은 Prefix/Exact 에 못 쓴다
                backend:
                  service: { name: catalog-svc, port: { number: 80 } }
    ```

①을 `shop-rewrite.yaml` 로 저장해 적용하고 확인한 뒤, ②로 바꿔 다시 적용합니다.

```bash
kubectl apply -f shop-rewrite.yaml         # ①
for p in /v1 /v1/items /v1/items/7; do hit $p -H 'Host: api.guide.local'; done
```

```text
/v1                200  served-by=catalog-... uri=/ host=api.guide.local
/v1/items          200  served-by=catalog-... uri=/ host=api.guide.local        ← /items 가 사라졌다
/v1/items/7        200  served-by=catalog-... uri=/ host=api.guide.local        ← 전부 / 로 간다
```

```bash
kubectl apply -f shop-rewrite.yaml         # ②로 고친 뒤
for p in /v1 /v1/items /v1/items/7; do hit $p -H 'Host: api.guide.local'; done
```

```text
/v1                200  served-by=catalog-... uri=/ host=api.guide.local
/v1/items          200  served-by=catalog-... uri=/items host=api.guide.local
/v1/items/7        200  served-by=catalog-... uri=/items/7 host=api.guide.local
```

관찰 포인트: `rewrite-target: /` 는 **나머지 경로까지 버립니다.** 앞부분만 떼려면 정규식 캡처 그룹(`/$2`)을 씁니다.

!!! warning "rewrite 는 조심해서 씁니다"
    - 어노테이션은 **ingress-nginx 전용**입니다. 다른 컨트롤러(Traefik, Gateway API 등)로 옮기면 다시 써야 합니다.
    - ingress-nginx 는 한 호스트에 rewrite(정규식)가 하나라도 있으면 **그 호스트의 모든 경로**를 정규식으로 해석합니다. 그래서 여기서는 호스트를 따로 뒀습니다.
    - 최근 버전은 Prefix/Exact 경로에 정규식 문자가 있으면 거부합니다. 정규식 경로는 `ImplementationSpecific` 으로 둡니다.
    - 시험에서 rewrite 를 요구하지 않으면 **붙이지 않습니다.** 채점은 보통 path · pathType · backend 만 봅니다.

---

## 7) NetworkPolicy 와 함께 — 컨트롤러만 들여보내기

"밖은 Ingress 로 열고, 안은 NetworkPolicy 로 잠근다" 는 조합입니다. 원리는 [14강 · NetworkPolicy](10-networkpolicy.md) 에서 다루니 여기서는 결과만 확인합니다.

`deny-in-all.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-in-all
  namespace: guide-ingress
spec:
  podSelector: {}           # ← 이 네임스페이스의 모든 파드
  policyTypes:
    - Ingress               # ← 규칙이 없다 = 들어오는 것 전부 거부
```

```bash
kubectl apply -f deny-in-all.yaml
time hit /menu -H 'Host: catalog.guide.local'
```

```text
/menu              504  <html>
real    0m15.1s
```

관찰 포인트: **504 Gateway Timeout** 입니다. 컨트롤러는 규칙을 찾았지만(404 아님), 백엔드 파드로 가는 연결이 **조용히 버려져** 기다리다 포기했습니다.

컨트롤러가 있는 네임스페이스에서 오는 것만 허용합니다.

`from-ingress-nginx.yaml`:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: from-ingress-nginx
  namespace: guide-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx   # ← 컨트롤러 파드가 있는 ns
      ports:
        - protocol: TCP
          port: 80                                         # ← 백엔드 파드 포트
```

```bash
kubectl apply -f from-ingress-nginx.yaml
hit /menu -H 'Host: catalog.guide.local'
kubectl run tmp -n guide-ingress --rm -i --restart=Never --image=busybox:1.36 -- wget -qO- --timeout=3 http://catalog-svc/menu
```

```text
/menu              200  served-by=catalog-5f7b9c6d4-k2x8p uri=/menu host=catalog.guide.local
wget: download timed out                  ← 같은 ns 의 다른 파드는 여전히 막힌다
```

!!! tip "시험 팁 — 출처를 가리지 않고 포트만 열 때"
    `from` 을 빼고 `ports` 만 두면 **어디서 오든 그 포트만** 허용합니다.
    ```yaml
      ingress:
        - ports:
            - protocol: TCP
              port: <파드포트>     # from 이 없다 = 출처 무관
    ```
    기본 거부 정책은 그대로 두고 이런 허용 정책을 **따로 추가**합니다. 두 정책의 허용 목록이 합쳐집니다.

!!! warning "컨트롤러가 hostNetwork 로 떠 있다면"
    컨트롤러 파드가 노드 네트워크를 쓰면 패킷의 출발지가 '파드' 가 아니라 '노드' 로 보여서 `namespaceSelector` 에 걸리지 않습니다.
    그때는 CNI 별 방법(노드 대역 허용 등)이 필요합니다. 이 과정의 NodePort 구성에서는 위 정책으로 충분합니다.

---

## 8) 안 될 때 — 증상으로 가른다

### describe 로 규칙과 백엔드를 한 번에

```bash
kubectl describe ingress shop-paths -n guide-ingress
```

```text
Name:             shop-paths
Namespace:        guide-ingress
Address:          192.168.56.11
Ingress Class:    nginx
Default backend:  <default>
Rules:
  Host        Path  Backends
  ----        ----  --------
  *
              /catalog   catalog-svc:80 (10.0.1.31:80,10.0.2.18:80)   ← 파드 IP 가 보여야 정상
              /board     board-svc:80 (10.0.2.44:80)
Events:
  Type    Reason  Age   From                      Message
  ----    ------  ----  ----                      -------
  Normal  Sync    9m    nginx-ingress-controller  Scheduled for sync
```

### 실험 — 백엔드 파드가 0 개면

```bash
kubectl scale deployment board -n guide-ingress --replicas=0
kubectl describe ingress shop-paths -n guide-ingress | grep board
hit /board
```

```text
              /board     board-svc:80 (<none>)          ← 백엔드 목록이 비었다
/board             503  <html>
```

### 컨트롤러 로그 — 요청이 어디로 갔는지

```bash
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=3
```

```text
192.168.56.10 - - [29/Sep/2026:06:31:02 +0000] "GET /board HTTP/1.1" 503 190 "-" "curl/7.81.0" 83 0.000 [guide-ingress-board-svc-80] [] - - - - 5c1f...
192.168.56.10 - - [29/Sep/2026:06:30:41 +0000] "GET /menu HTTP/1.1" 200 71 "-" "curl/7.81.0" 95 0.002 [guide-ingress-catalog-svc-80] [] 10.0.1.31:80 71 0.002 200 9a7e...
...
```

관찰 포인트: `[네임스페이스-서비스-포트]` 가 **어느 규칙에 걸렸는지**, 그 뒤의 `10.0.1.31:80` 이 **실제로 간 파드**입니다. 503 줄에는 파드 주소가 `-` 로 비어 있습니다.

```bash
kubectl scale deployment board -n guide-ingress --replicas=1
```

| 증상 | 어디 문제인가 | 확인 |
|------|---------------|------|
| `ADDRESS` 가 비어 있다 | 컨트롤러가 없거나 클래스 불일치 | `get pods -n ingress-nginx` · `get ingressclass` · `spec.ingressClassName` |
| 접속 자체가 안 된다 (`000`) | 컨트롤러 Service / NodePort | `get svc -n ingress-nginx`, 포트 번호 |
| **404** | 규칙 (path · pathType · host) | `describe ingress` 의 Rules, `-H 'Host: ...'` 를 줬는가 |
| **503** | 백엔드 Service 에 Endpoints 없음 | `describe ingress` 의 Backends, `get endpoints` |
| **504** | 백엔드로 가는 연결이 막힘 | NetworkPolicy, 파드 포트 |
| 이상한 인증서 (Fake Certificate) | tls.hosts · Secret 이름/타입/네임스페이스 | `get secret`, `openssl s_client -servername` |
| 백엔드가 이상한 경로를 받는다 | rewrite 어노테이션 | 컨트롤러 로그, 앱이 받은 URI |

---

## 9) Ingress 를 Gateway API 로 옮기기

Gateway API 는 Ingress 의 뒤를 잇는 표준입니다(구조 전체는 8강 후반의 Gateway API 부분에서 다룹니다). 시험에는 **이미 있는 Ingress 를 읽고, 같은 동작을 하는 Gateway + HTTPRoute 를 만든 뒤, Ingress 를 지우는** 문제가 나옵니다. **HTTPS 리스너**까지 옮겨야 합니다.

여기서는 5) 에서 만든 `shop-tls`(호스트 `secure.guide.local` · Secret `guide-tls` · `catalog-svc:80`)를 옮깁니다.

### 준비 — GatewayClass 확인, 7) 의 정책 걷어내기

```bash
kubectl get gatewayclass
kubectl api-resources --api-group=gateway.networking.k8s.io
```

```text
NAME    CONTROLLER                                   ACCEPTED   AGE
nginx   gateway.nginx.org/nginx-gateway-controller   True       3d

NAME              SHORTNAMES   APIVERSION                           NAMESPACED   KIND
gatewayclasses    gc           gateway.networking.k8s.io/v1         false        GatewayClass
gateways          gtw          gateway.networking.k8s.io/v1         true         Gateway
grpcroutes                     gateway.networking.k8s.io/v1         true         GRPCRoute
httproutes                     gateway.networking.k8s.io/v1         true         HTTPRoute
referencegrants   refgrant     gateway.networking.k8s.io/v1beta1    true         ReferenceGrant
```

관찰 포인트: Ingress 에서 `get ingressclass` 로 클래스 이름을 확인했듯, 여기서는 **`get gatewayclass` 의 NAME** 이 Gateway 의 `gatewayClassName` 에 들어갑니다. `ACCEPTED` 가 `True` 여야 컨트롤러가 살아 있는 것입니다.

??? note "GatewayClass 가 없다면 — 설치 (NGINX Gateway Fabric)"
    Gateway API 는 **CRD + 구현(컨트롤러)** 을 따로 설치합니다. 채점 세트 `08-ingress` 의 `start` 가 아래와 같은 버전으로 자동 설치하므로, 채점 세트를 한 번 시작했다면 이미 있습니다.
    ```bash
    NGF_VERSION=v2.7.0
    kubectl kustomize "https://github.com/nginx/nginx-gateway-fabric/config/crd/gateway-api/standard?ref=${NGF_VERSION}" | kubectl apply -f -
    kubectl apply --server-side -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/${NGF_VERSION}/deploy/crds.yaml
    kubectl apply -f https://raw.githubusercontent.com/nginx/nginx-gateway-fabric/${NGF_VERSION}/deploy/nodeport/deploy.yaml
    kubectl wait --for=condition=Available deployment --all -n nginx-gateway --timeout=180s
    ```
    시험 클러스터에는 GatewayClass 가 미리 있습니다. 시험에서는 **설치하지 말고 이름만 확인**합니다.

NGINX Gateway Fabric 은 Gateway 를 하나 만들 때마다 **그 Gateway 와 같은 네임스페이스에** 데이터 플레인 파드를 띄웁니다. 7) 의 기본 거부 정책이 남아 있으면 그 파드로 들어오는 요청도, 그 파드가 백엔드로 가는 요청도(출발지가 `ingress-nginx` 네임스페이스가 아니므로) 막힙니다. 먼저 걷어냅니다.

```bash
kubectl delete -f from-ingress-nginx.yaml -f deny-in-all.yaml
```

### ① 옮길 Ingress 를 읽는다

```bash
kubectl get ingress shop-tls -n guide-ingress -o yaml | sed -n '/^spec:/,/^status:/p'
```

```text
spec:
  ingressClassName: nginx
  rules:
  - host: secure.guide.local
    http:
      paths:
      - backend:
          service:
            name: catalog-svc
            port:
              number: 80
        path: /
        pathType: Prefix
  tls:
  - hosts:
    - secure.guide.local
    secretName: guide-tls
status:
```

이 값들을 한 칸씩 옮겨 적습니다.

| Ingress 필드 | 옮겨 갈 곳 | 이 실습의 값 |
|--------------|-----------|-------------|
| `spec.ingressClassName` | Gateway `spec.gatewayClassName` | `nginx` |
| (컨트롤러 Service 의 443 포트) | Gateway `spec.listeners[]` — `port: 443`, `protocol: HTTPS` | 리스너 이름 `https` |
| `spec.tls[].hosts[]` | 리스너의 `hostname` | `secure.guide.local` |
| `spec.tls[].secretName` | 리스너의 `tls.certificateRefs[].name` (`kind: Secret`) | `guide-tls` |
| (HTTPS 는 컨트롤러에서 끝난다) | 리스너의 `tls.mode: Terminate` (생략하면 기본값) | `Terminate` |
| `spec.rules[].host` | HTTPRoute `spec.hostnames[]` | `secure.guide.local` |
| `paths[].pathType: Prefix` + `path` | `rules[].matches[].path` — `type: PathPrefix` + `value` | `/` |
| `paths[].pathType: Exact` | `type: Exact` | — |
| `backend.service.name` / `port.number` | `rules[].backendRefs[].name` / `port` | `catalog-svc` / `80` |
| — (Ingress 에는 없던 개념) | HTTPRoute `spec.parentRefs[]` — 어느 Gateway 에 붙나 | `guide-gw` |

Ingress 한 장이 **두 장으로 나뉩니다.** 입구(포트·프로토콜·호스트·인증서)는 Gateway 로, 라우팅(호스트·경로·백엔드)은 HTTPRoute 로 갑니다. 호스트는 양쪽에 다 적습니다.

### ② Gateway — HTTPS 리스너

필드 이름이 헷갈리면 Ingress 때처럼 `explain` 으로 내려갑니다.

```bash
kubectl explain gateway.spec.listeners.tls
```

```text
FIELDS:
  certificateRefs     <[]Object>
  mode                <string>
  options             <map[string]string>
```

`guide-gw.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: guide-gw
  namespace: guide-ingress
spec:
  gatewayClassName: nginx               # ← ingressClassName
  listeners:
    - name: https
      port: 443
      protocol: HTTPS
      hostname: secure.guide.local      # ← tls[].hosts
      tls:
        mode: Terminate                 # ← 생략해도 Terminate
        certificateRefs:
          - kind: Secret
            name: guide-tls             # ← tls[].secretName (같은 네임스페이스)
```

```bash
kubectl apply -f guide-gw.yaml
kubectl get gateway guide-gw -n guide-ingress
kubectl get gateway guide-gw -n guide-ingress -o jsonpath='{range .status.listeners[*]}{.name}{"  attachedRoutes="}{.attachedRoutes}{"\n"}{range .conditions[*]}{"  "}{.type}={.status}{"  "}{.reason}{"\n"}{end}{end}'
```

```text
NAME       CLASS   ADDRESS   PROGRAMMED   AGE
guide-gw   nginx             True         15s

https  attachedRoutes=0
  Accepted=True  Accepted
  ResolvedRefs=True  ResolvedRefs        ← 인증서 Secret 을 찾았다
  Programmed=True  Programmed
  Conflicted=False  NoConflicts
```

관찰 포인트: `attachedRoutes=0` — 입구는 열렸지만 **아직 붙은 라우트가 없습니다.** 이 상태로 접속하면 404 입니다. Ingress 에서는 규칙이 입구와 한 몸이었지만, Gateway API 에서는 HTTPRoute 가 와서 붙어야 합니다.

### 실험 — Secret 이름이 틀리면

```bash
kubectl patch gateway guide-gw -n guide-ingress --type=json \
  -p='[{"op":"replace","path":"/spec/listeners/0/tls/certificateRefs/0/name","value":"guide-tls-typo"}]'
sleep 3
kubectl get gateway guide-gw -n guide-ingress -o jsonpath='{.status.listeners[0].conditions[?(@.type=="ResolvedRefs")]}{"\n"}'
```

```text
{"lastTransitionTime":"2026-10-04T06:41:12Z","message":"Secret guide-ingress/guide-tls-typo does not exist","observedGeneration":2,"reason":"InvalidCertificateRef","status":"False","type":"ResolvedRefs"}
```

관찰 포인트: Ingress 는 Secret 이 틀려도 조용히 **가짜 인증서**를 내줬지만(5)), Gateway 는 리스너 상태에 **`InvalidCertificateRef`** 로 이유를 적어 줍니다. 되돌려 놓습니다.

```bash
kubectl apply -f guide-gw.yaml
```

!!! warning "인증서 Secret 은 Gateway 와 같은 네임스페이스에"
    `certificateRefs` 에 다른 네임스페이스의 Secret 을 적으려면 그쪽 네임스페이스에 **ReferenceGrant** 가 있어야 합니다. 없으면 위와 같은 `ResolvedRefs=False` 가 됩니다.
    Ingress 를 옮길 때는 Secret 이 이미 Ingress 와 같은 네임스페이스에 있으므로, **Gateway 도 같은 네임스페이스에** 만드는 것이 가장 단순합니다.

### ③ HTTPRoute — 호스트·경로·백엔드

`guide-route.yaml`:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: guide-route
  namespace: guide-ingress
spec:
  parentRefs:
    - name: guide-gw                    # ← 어느 Gateway 에 붙나
  hostnames:
    - secure.guide.local                # ← rules[].host
  rules:
    - matches:
        - path:
            type: PathPrefix            # ← pathType: Prefix
            value: /
      backendRefs:
        - name: catalog-svc             # ← backend.service.name
          port: 80                      # ← backend.service.port.number
```

```bash
kubectl apply -f guide-route.yaml
kubectl get httproute guide-route -n guide-ingress
kubectl get httproute guide-route -n guide-ingress -o jsonpath='{range .status.parents[*]}{.parentRef.name}{": "}{range .conditions[*]}{.type}={.status} {end}{"\n"}{end}'
kubectl get gateway guide-gw -n guide-ingress -o jsonpath='{.status.listeners[0].attachedRoutes}{"\n"}'
```

```text
NAME          HOSTNAMES                AGE
guide-route   ["secure.guide.local"]   10s

guide-gw: Accepted=True ResolvedRefs=True
1
```

관찰 포인트: HTTPRoute 쪽에도 상태가 있습니다. **`Accepted`** 는 "Gateway 가 이 라우트를 받아들였다"(리스너 hostname 과 라우트 hostnames 가 맞아야 함), **`ResolvedRefs`** 는 "backendRefs 의 Service·포트를 찾았다" 입니다. 그리고 Gateway 의 `attachedRoutes` 가 1 로 올라갔습니다.

| 증상 | 어디를 보나 |
|------|------------|
| Gateway `PROGRAMMED` 가 비거나 False | `gatewayClassName` 오타, GatewayClass `ACCEPTED` |
| 리스너 `ResolvedRefs=False` (`InvalidCertificateRef`) | `certificateRefs` 의 Secret 이름 · 네임스페이스 · 타입 |
| 라우트 `Accepted=False` (`NoMatchingListenerHostname`) | 리스너 `hostname` 과 라우트 `hostnames` 불일치 |
| 라우트 `ResolvedRefs=False` (`BackendNotFound`) | `backendRefs` 의 Service 이름 · 포트 |
| `attachedRoutes=0` | `parentRefs` 의 Gateway 이름 · 네임스페이스 |

### ④ HTTPS 로 확인 — 입구가 두 개인 상태

Gateway 를 만들면 컨트롤러가 그 앞에 Service 를 하나 만듭니다. 그 Service 의 **443 에 붙은 NodePort** 로 들어갑니다.

```bash
kubectl get svc -n guide-ingress -l gateway.networking.k8s.io/gateway-name=guide-gw
GW_HTTPS_PORT=$(kubectl get svc -n guide-ingress -l gateway.networking.k8s.io/gateway-name=guide-gw \
  -o jsonpath='{.items[0].spec.ports[?(@.port==443)].nodePort}')
echo "gateway https:$GW_HTTPS_PORT   ingress https:$HTTPS_PORT"

echo "== Gateway";  curl -s --cacert guide.crt --resolve secure.guide.local:$GW_HTTPS_PORT:$NODE_IP https://secure.guide.local:$GW_HTTPS_PORT/pay
echo "== Ingress";  curl -s --cacert guide.crt --resolve secure.guide.local:$HTTPS_PORT:$NODE_IP    https://secure.guide.local:$HTTPS_PORT/pay
```

```text
NAME             TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
guide-gw-nginx   LoadBalancer   10.104.77.9   <pending>     443:30943/TCP   2m

gateway https:30943   ingress https:32507
== Gateway
served-by=catalog-5f7b9c6d4-m9q4t uri=/pay host=secure.guide.local
== Ingress
served-by=catalog-5f7b9c6d4-k2x8p uri=/pay host=secure.guide.local
```

관찰 포인트: 같은 인증서(`--cacert guide.crt` 로 검증 통과), 같은 백엔드입니다. 지금은 **Ingress 와 Gateway 두 입구가 같은 앱을 가리키는** 이전 중 상태입니다. 실제 운영에서도 새 입구를 확인한 다음에 옛 입구를 닫습니다.
(Service 이름과 TYPE 은 구현·설치 방식에 따라 다를 수 있습니다. LoadBalancer 가 Pending 이어도 NodePort 는 열려 있습니다.)

!!! note "Gateway 쪽만 응답이 없다면 — externalTrafficPolicy"
    NGINX Gateway Fabric 이 만드는 Service 는 `externalTrafficPolicy: Local` 이라, **데이터 플레인 파드가 떠 있는 노드로 들어온 요청만** 받습니다.
    ```bash
    kubectl get pods -n guide-ingress -l gateway.networking.k8s.io/gateway-name=guide-gw -o wide
    ```
    `NODE` 칸의 노드 IP 로 `--resolve` 를 바꿔서 다시 해 봅니다.

### ⑤ Ingress 를 지운다

새 입구가 동작하는 것을 확인했으니 옛 입구를 닫습니다. **시험에서도 이 단계가 채점 항목**입니다.

```bash
kubectl delete ingress shop-tls -n guide-ingress
echo "== Gateway";  curl -s --cacert guide.crt --resolve secure.guide.local:$GW_HTTPS_PORT:$NODE_IP https://secure.guide.local:$GW_HTTPS_PORT/pay
echo "== Ingress";  curl -sk -o /dev/null -w '%{http_code}\n' --resolve secure.guide.local:$HTTPS_PORT:$NODE_IP https://secure.guide.local:$HTTPS_PORT/pay
```

```text
ingress.networking.k8s.io "shop-tls" deleted
== Gateway
served-by=catalog-5f7b9c6d4-k2x8p uri=/pay host=secure.guide.local
== Ingress
404
```

관찰 포인트: Ingress 쪽은 이제 규칙이 없어 **404**(인증서도 가짜 기본 인증서라 `-k` 가 필요)입니다. 남은 입구는 Gateway 하나입니다. Secret `guide-tls` 와 `catalog-svc` 는 Gateway 가 계속 쓰므로 **지우면 안 됩니다.**

!!! tip "시험 팁 — 옮기기 문제의 순서"
    1. `kubectl get ingress <이름> -n <ns> -o yaml` 로 **class · host · secretName · path · pathType · backend** 를 메모한다.
    2. `kubectl get gatewayclass` 로 클래스 이름을 확인한다.
    3. Gateway(리스너 `https` · 443 · HTTPS · hostname · certificateRefs) → HTTPRoute(parentRefs · hostnames · PathPrefix · backendRefs) 순으로 만든다. `kubectl create` 로 만드는 명령이 없으므로 YAML 로 씁니다. 필드는 `kubectl explain gateway.spec.listeners` · `kubectl explain httproute.spec.rules` 로 확인합니다.
    4. `kubectl describe gateway` · `describe httproute` 로 Accepted · Programmed · ResolvedRefs 를 확인한다.
    5. **마지막에 Ingress 를 지운다.** 문제에 적힌 대로 Service · Secret 은 남긴다.

??? note "ingress-nginx 가 해 주던 HTTP → HTTPS 리다이렉트는?"
    5) 에서 본 308 리다이렉트는 ingress-nginx 가 **알아서** 하던 일입니다. Gateway API 에서는 직접 만들어야 합니다. HTTP 리스너를 하나 더 열고, 그 리스너에만 붙는 리다이렉트 라우트를 둡니다.
    ```yaml
    # guide-gw.yaml 의 listeners 에 추가
        - name: http
          port: 80
          protocol: HTTP
          hostname: secure.guide.local
    ---
    apiVersion: gateway.networking.k8s.io/v1
    kind: HTTPRoute
    metadata:
      name: guide-redirect
      namespace: guide-ingress
    spec:
      parentRefs:
        - name: guide-gw
          sectionName: http           # ← http 리스너에만 붙인다
      hostnames: [secure.guide.local]
      rules:
        - filters:
            - type: RequestRedirect
              requestRedirect: { scheme: https, statusCode: 301 }
    ```
    이때 `guide-route` 의 `parentRefs` 에도 `sectionName: https` 를 넣어야 합니다. 안 넣으면 `guide-route` 가 두 리스너 모두에 붙어서 HTTP 로도 그냥 응답합니다.
    rewrite 어노테이션(6))도 같은 사정으로, Gateway API 에서는 HTTPRoute 의 `URLRewrite` 필터로 다시 씁니다.

---

## 시험에서 빠르게

```bash
kubectl get ingressclass                                     # 클래스 이름 확인
kubectl create ingress <n> -n <ns> --class=nginx --rule="/<path>*=<svc>:<port>"             # Prefix
kubectl create ingress <n> -n <ns> --class=nginx --rule="<host>/*=<svc>:<port>"             # 호스트
kubectl create ingress <n> -n <ns> --class=nginx --rule="<host>/*=<svc>:<port>,tls=<secret>" # TLS
kubectl create ingress <n> -n <ns> --class=nginx --default-backend=<svc>:<port>              # 기본 백엔드

openssl req -x509 -nodes -days <일> -newkey rsa:2048 -keyout <k> -out <c> -subj "/CN=<host>"
kubectl create secret tls <secret> -n <ns> --cert=<c> --key=<k>

kubectl get ingress <n> -n <ns>                               # CLASS · ADDRESS
kubectl describe ingress <n> -n <ns>                          # Rules · Backends (파드 IP)
curl -H 'Host: <host>' http://<노드IP>:<NodePort>/<path>
curl -k --resolve <host>:<https포트>:<노드IP> https://<host>:<https포트>/

# Ingress → Gateway API
kubectl get gatewayclass                                      # gatewayClassName
kubectl get ingress <n> -n <ns> -o yaml                       # class · host · secretName · path · backend 메모
kubectl explain gateway.spec.listeners.tls                    # certificateRefs · mode
kubectl describe gateway <gw> -n <ns>                         # Accepted · Programmed · 리스너 ResolvedRefs
kubectl describe httproute <rt> -n <ns>                       # Accepted · ResolvedRefs
kubectl delete ingress <n> -n <ns>                            # 마지막에 옛 입구를 닫는다
```

---

## 정리

```bash
kubectl delete namespace guide-ingress
rm -f backends.yaml probe-ing.yaml shop-paths.yaml shop-hosts.yaml shop-fallback.yaml shop-tls.yaml \
      shop-rewrite.yaml deny-in-all.yaml from-ingress-nginx.yaml guide-gw.yaml guide-route.yaml \
      guide.key guide.crt /tmp/hit.out
unset -f hit
```

---

## 확인 질문

??? question "1. Ingress 를 만들었는데 ADDRESS 가 계속 비어 있습니다. 무엇을 보나요?"
    컨트롤러 파드가 떠 있는지(`get pods -n ingress-nginx`), `get ingressclass` 의 이름과 Ingress 의 `ingressClassName` 이 같은지 봅니다.
    기본 클래스가 없는데 `ingressClassName` 을 빼먹은 경우도 같은 증상입니다.

??? question "2. `path: /shop`, `pathType: Prefix` 규칙은 `/shopping` 을 잡나요?"
    잡지 않습니다. Prefix 는 `/` 로 나뉜 **경로 요소** 단위로 비교합니다. `/shop`, `/shop/`, `/shop/cart` 는 잡습니다.

??? question "3. `kubectl create ingress --rule` 에서 Prefix 와 Exact 는 어떻게 구분하나요?"
    경로 끝에 `*` 를 붙이면 Prefix, 붙이지 않으면 Exact 입니다. 예: `--rule="/shop*=shop-svc:80"`.

??? question "4. DNS 없이 호스트 기반 Ingress 를 확인하는 방법 두 가지는?"
    `curl -H 'Host: <host>' http://<노드IP>:<포트>/` 와 `curl --resolve <host>:<포트>:<노드IP> http://<host>:<포트>/`.
    HTTPS 는 SNI 까지 맞추는 `--resolve` 가 정확합니다.

??? question "5. TLS Ingress 에 접속했더니 'Kubernetes Ingress Controller Fake Certificate' 가 나옵니다. 원인 후보는?"
    요청한 호스트가 `tls.hosts` 에 없거나, `secretName` 이 틀렸거나, Secret 이 다른 네임스페이스에 있거나, 타입이 `kubernetes.io/tls` 가 아닌 경우입니다.

??? question "6. 404 · 503 · 504 는 각각 어디를 보라는 신호인가요?"
    404 는 규칙(path · pathType · host), 503 은 백엔드 Service 의 Endpoints, 504 는 백엔드로 가는 연결(NetworkPolicy · 파드 포트)입니다.

??? question "7. `rewrite-target: /` 만 붙이면 어떤 문제가 생기나요?"
    `/v1/items` 처럼 뒤에 붙은 경로까지 모두 `/` 로 바뀝니다. 앞부분만 떼려면 정규식 경로(`/v1(/|$)(.*)`)와 `rewrite-target: /$2` 를 함께 씁니다.

??? question "8. 네임스페이스에 Ingress 기본 거부를 걸었더니 Ingress 로도 접속이 안 됩니다. 어떻게 여나요?"
    컨트롤러가 있는 네임스페이스(`kubernetes.io/metadata.name: ingress-nginx`)에서 오는 트래픽을 백엔드 파드 포트로 허용하는 정책을 추가합니다.

??? question "9. Ingress 의 `tls[].hosts` 와 `tls[].secretName` 은 Gateway API 에서 어디로 가나요?"
    Gateway 리스너(`port: 443`, `protocol: HTTPS`)의 `hostname` 과 `tls.certificateRefs[].name`(`kind: Secret`)으로 갑니다. `rules[].host` 는 HTTPRoute 의 `hostnames` 로 갑니다.

??? question "10. Gateway 와 HTTPRoute 를 만들었는데 Gateway 리스너의 `attachedRoutes` 가 0 입니다. 무엇을 보나요?"
    HTTPRoute 의 `parentRefs` 에 적은 Gateway 이름·네임스페이스, 그리고 라우트 `hostnames` 가 리스너 `hostname` 과 맞는지(라우트 `Accepted` 조건) 봅니다.

??? question "11. Ingress 를 Gateway API 로 옮긴 뒤 Secret 이나 Service 도 지워도 되나요?"
    안 됩니다. Gateway 의 `certificateRefs` 가 같은 Secret 을, HTTPRoute 의 `backendRefs` 가 같은 Service 를 계속 씁니다. 지우는 것은 Ingress 하나입니다.

---

## 다음 단계 — 채점 실습

개념이 잡혔으면 시험 형식으로 풀어 봅니다. 이번에는 **답이 없습니다.** 컨트롤러가 설치돼 있어야 실제 접속까지 확인할 수 있습니다. Q5 에 필요한 Gateway API(NGINX Gateway Fabric)는 클러스터에 없으면 `start` 가 자동으로 설치합니다.

```bash
cd ~/cka-lab/08-ingress
bash exam.sh start
```

| 문제 | 이 페이지에서 연습한 것 |
|------|------------------------|
| Q1 경로 기반 Ingress | 1) 2) 3) — Service 확인, ingressClassName, Prefix |
| Q2 호스트 기반 Ingress | 4) — rules.host, `-H 'Host: ...'` 로 확인 |
| Q3 자체 서명 인증서로 TLS Ingress | 5) — openssl → `create secret tls` → `spec.tls` |
| Q4 NetworkPolicy 로 네임스페이스 격리 | 7) (원리는 [14강](10-networkpolicy.md) 의 2) · 3)) |
| Q5 Ingress 를 Gateway API 로 옮기기 (HTTPS) | 9) — 필드 대응표, HTTPS 리스너 · certificateRefs, HTTPRoute, Ingress 삭제 |
