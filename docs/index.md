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

!!! note "2026-10 온라인 강의 번호로 바꿨습니다 — 페이지 주소(파일 이름)는 예전 번호 그대로"
    페이지 순서와 제목의 번호는 **온라인 강의(24강) 순서**를 따릅니다. 즐겨찾기가 깨지지 않도록 페이지 주소는 예전 번호를 그대로 둡니다.
    예: 4·6·10강 워크로드 페이지의 주소는 `labs/06-workloads/` 입니다.

**페이지의 강 번호 = 온라인 강의 번호 = 채점 세트 폴더 번호**입니다. 4강을 봤다면 4강이 붙은 페이지를 열고, 끝나면 `04-pods` 를 풉니다. 여러 강의에 걸친 페이지는 강의마다 이어서 풀 세트가 따로 있습니다.
따라 하기 페이지가 없는 강의(1·2·9강)는 영상과 채점 세트로 공부하고, 24강(모의고사)은 `mock-1` ~ `mock-6` 과 `final-test` 를 풉니다.

| 강 | 주제 | 핵심 키워드 | 이어서 풀 채점 세트 |
|------|------|------------|------------------|
| [3강](labs/04-cluster-build.md) | 내 클러스터 만들기 (VirtualBox) | 3노드 구축, containerd, kubeadm init/join, Cilium | — (이후 모든 실습의 전제) |
| [3·19강](labs/13-cluster-setup.md) | 노드 조인 · CNI · CRI | kubeadm join, crictl, CNI, kubelet, cri-dockerd | `03-cluster-setup` |
| [4·6·10강](labs/06-workloads.md) | 워크로드 | Deployment, 롤백, ConfigMap, Secret, CronJob, DaemonSet, 네이티브 사이드카 | `04-pods` · `06-config` · `10-workloads` |
| [5강](labs/09-networking.md) | 서비스와 내부 통신 | ClusterIP, NodePort, DNS | `05-networking` |
| [7·11·12강](labs/07-scheduling.md) | 자원 · 스케줄링 · HPA | Requests/Limits, QoS, LimitRange, Quota, Affinity, Taint, HPA, PriorityClass | `07-resources` · `11-scheduling` |
| [8강](labs/11-ingress.md) | Ingress → Gateway API | 경로·호스트 라우팅, TLS, Gateway 이전 | `08-ingress` |
| [13강](labs/08-storage.md) | 스토리지 | PV/PVC, StorageClass, StatefulSet, emptyDir | `13-storage` |
| [14강](labs/10-networkpolicy.md) | NetworkPolicy | 기본 거부, 선택 허용, Egress | `14-networkpolicy` |
| [15강](labs/14-rbac.md) | RBAC | ServiceAccount, Role, ClusterRole | `15-rbac` |
| [16·17강](labs/17-helm.md) | Helm · Kustomize · CRD | install/upgrade/rollback, template, overlay, CRD | `16-helm` |
| [18강](labs/03-architecture.md) | 아키텍처 (컨트롤플레인) | 컨트롤플레인, etcd, kubelet, 선언적 모델 | `18-architecture` |
| [19강](labs/16-node-ops.md) | 노드 운영 | static pod, kubelet NotReady, drain 걸림돌, etcdctl | `19-node-ops` |
| [20·21강](labs/15-maintenance.md) | 유지보수 | drain, PDB, etcd 백업·복원, 업그레이드, 인증서 | `20-maintenance` |
| [22·23강](labs/19-troubleshooting.md) | 트러블슈팅 | CrashLoopBackOff, ImagePull, Endpoints, 노드, 컨트롤플레인 | `22-troubleshooting` |

처음이라면 [3강 · 내 클러스터 만들기](labs/04-cluster-build.md)로 클러스터를 만들고 [시작하기](start.md)를 보세요.
