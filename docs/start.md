# 시작하기

## 실습 환경

이 가이드는 **4강(VirtualBox 클러스터 구축)** 으로 만든 3노드 클러스터를 전제로 합니다. 아직 없다면 [4강 · 클러스터 구축](labs/04-cluster-build.md)을 먼저 따라 하세요.

| 항목 | 값 |
|------|----|
| 버전 | Kubernetes v1.32 · Ubuntu Server 24.04 · containerd · Cilium (기본 모드, kube-proxy 있음) |
| VM 사용자 | `vboxuser` (설치 때 정한 이름) |
| 파드 IP | `10.0.x.x` (Cilium IPAM) · Service IP `10.96.0.0/12` |

| 노드 | 역할 | IP (예시) |
|------|------|-----------|
| `control-plane` | 컨트롤플레인 | 192.168.56.10 |
| `worker-1` | 워커 | 192.168.56.11 |
| `worker-2` | 워커 | 192.168.56.12 |

노드 이름은 환경마다 다를 수 있습니다(`worker1`, `worker-1`, `node01` …). 가이드에 노드 이름이 나오면 **내 클러스터의 이름으로 바꿔서** 입력하세요.

모든 명령은 컨트롤플레인 노드에서 실행합니다. 호스트 PC 에서 `ssh vboxuser@192.168.56.10` 으로 접속하고, 노드끼리는 hosts 파일 덕분에 `ssh worker-1` 처럼 이름으로 접속할 수 있습니다. 시작 전에 노드가 모두 `Ready` 인지 확인하세요.

```bash
kubectl get nodes
```

```text
NAME            STATUS   ROLES           AGE   VERSION
control-plane   Ready    control-plane   3d    v1.32.x
worker-1        Ready    <none>          3d    v1.32.x
worker-2        Ready    <none>          3d    v1.32.x
```

---

## 시험장과 같은 셸 만들기

CKA 시험장에는 `k` 별칭과 자동완성이 준비돼 있습니다. 실습 환경도 똑같이 맞춰 두면 시험 때 손이 덜 헷갈립니다.

```bash
cat <<'EOF' >> ~/.bashrc
source <(kubectl completion bash)
alias k=kubectl
complete -o default -F __start_kubectl k
export do="--dry-run=client -o yaml"
EOF
source ~/.bashrc
```

`$do` 는 이 가이드에서도 자주 씁니다. 리소스를 만들지 않고 YAML 만 뽑아 주는 옵션입니다.

```bash
kubectl create deployment demo --image=nginx $do
```

!!! note "채점 실습 저장소를 이미 받았다면"
    `cka-lab` 의 `bash setup-shell.sh` 가 위 설정을 대신 해 줍니다.

---

## 페이지 구성

각 페이지는 같은 순서로 되어 있습니다.

1. **실습 목표** — 이 페이지에서 확인할 것
2. **준비** — 실습용 네임스페이스 만들기
3. **단계별 실습** — 명령 → 결과 → 왜 그렇게 되는지
4. **정리** — 만든 리소스 지우기
5. **확인 질문** — 답이 바로 나오면 이해한 것입니다
6. **다음 단계** — 이어서 풀 채점 세트

!!! warning "페이지마다 네임스페이스를 따로 씁니다"
    각 페이지는 `guide-` 로 시작하는 전용 네임스페이스에서 진행합니다.
    끝나면 **정리** 단계로 네임스페이스째 지우세요. 다른 실습에 영향을 주지 않습니다.
