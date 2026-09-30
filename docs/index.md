# CKA 실습 가이드

스킬잇 CKA 온라인 강의의 **따라 하기 실습**입니다.
강의 영상 하나를 보고 나서 이 페이지를 열어 명령을 하나씩 쳐 보면, 영상에서 본 개념이 손에 남습니다.

---

## 이 가이드와 채점 실습은 다릅니다

강의마다 실습이 두 가지 있습니다. **이 가이드를 먼저 하고, 채점 실습으로 확인하세요.**

| | 이 가이드 | 채점 실습 (cka-lab) |
|---|---|---|
| 하는 일 | 따라 하며 **이해** | 시험처럼 풀고 **채점** |
| 형태 | 웹 문서 — 단계별 설명과 명령 | `bash exam.sh start` + 웹 패널 |
| 정답 | 명령과 결과를 그대로 보여 줍니다 | 알려 주지 않습니다 (힌트만) |
| 목적 | 개념 채우기 · 복습 | 손에 익히기 · 시간 재기 |

!!! tip "리소스 이름이 서로 다릅니다"
    이 가이드는 채점 실습과 **다른 이름과 네임스페이스**를 씁니다.
    가이드를 따라 한 결과가 채점 문제의 답이 되지 않고, 두 실습이 서로의 리소스를 지우지도 않습니다.

---

## 강의별 실습

**페이지 번호 = 강의 번호 = 채점 세트 폴더 번호**입니다. 6강을 봤다면 6강 페이지를 열고, 끝나면 `06-workloads` 를 풉니다.
번호가 비는 강의(1·2·5·12·18강)는 따라 하기 실습이 없고, 20강(모의고사)은 `mock-1` ~ `mock-6` 을 풉니다.

| 강 | 주제 | 핵심 키워드 | 이어서 풀 채점 세트 |
|------|------|------------|------------------|
| [3강](labs/03-architecture.md) | 아키텍처 | 컨트롤플레인, etcd, kubelet, 선언적 모델 | `03-architecture` |
| [4강](labs/04-cluster-build.md) | 클러스터 구축 (VirtualBox) | 3노드 구축, containerd, kubeadm init/join, Cilium | — (이후 모든 실습의 전제) |
| [6강](labs/06-workloads.md) | 워크로드 | Deployment, 롤백, ConfigMap, CronJob, DaemonSet, 네이티브 사이드카 | `06-workloads` |
| [7강](labs/07-scheduling.md) | 스케줄링 | Requests/Limits, Affinity, Taint, HPA, PriorityClass | `07-scheduling` |
| [8강](labs/08-storage.md) | 스토리지 | PV/PVC, StorageClass, StatefulSet, emptyDir | `08-storage` |
| [9강](labs/09-networking.md) | 서비스와 내부 통신 | ClusterIP, NodePort, DNS | `09-networking` |
| [10강](labs/10-networkpolicy.md) | NetworkPolicy | 기본 거부, 선택 허용, Egress | `10-networkpolicy` |
| [11강](labs/11-ingress.md) | Ingress | 경로·호스트 라우팅, TLS | `11-ingress` |
| [13강](labs/13-cluster-setup.md) | 클러스터 구축 | kubeadm join, crictl, CNI | `13-cluster-setup` |
| [14강](labs/14-rbac.md) | RBAC | ServiceAccount, Role, ClusterRole | `14-rbac` |
| [15강](labs/15-maintenance.md) | 유지보수 | drain, etcd 백업, 업그레이드, 인증서 | `15-maintenance` |
| [16강](labs/16-node-ops.md) | 노드 운영 | drain/uncordon, etcd, static pod | `16-node-ops` |
| [17강](labs/17-helm.md) | Helm과 Kustomize | install/upgrade/rollback, template, overlay | `17-helm` |
| [19강](labs/19-troubleshooting.md) | 트러블슈팅 | CrashLoopBackOff, ImagePull, Endpoints, 노드 | `19-troubleshooting` |

처음이라면 [4강 · 클러스터 구축](labs/04-cluster-build.md)으로 클러스터를 만들고 [시작하기](start.md)를 보세요.
