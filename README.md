# homelab-gitops

Argo CD로 자기소개 웹 앱, Traefik Ingress Controller, Cloudflare Tunnel Connector와 Calico 네트워크 설정을 관리하는 Kubernetes GitOps 저장소입니다.

```text
argocd/
├── self-Intruduction.yaml       # 웹 앱 Argo CD Application
├── traefik.yaml                  # Traefik Helm chart Argo CD Application
├── cloudflared.yaml             # Tunnel Connector Argo CD Application
└── calico.yaml                  # Calico 주소 탐지 Argo CD Application (자동 동기화)
apps/self-Intruduction/
├── deployment.yaml               # 웹 앱 Deployment
├── service.yaml                  # 내부 ClusterIP Service
└── ingress.yaml                  # intro.woonapp.com → 웹 앱
infrastructure/
├── cloudflared/
│   ├── kustomization.yaml
│   └── deployment.yaml            # 원격 관리 Tunnel Connector
├── traefik/
│   └── values.yaml                # Traefik Helm chart 설정
└── calico/
    ├── kustomization.yaml
    └── wired-network.yaml         # 기존 Installation/default의 유선 주소 탐지
```

## 트래픽 흐름

```text
브라우저 HTTPS
  → Cloudflare DNS / Tunnel
  → cloudflared (Kubernetes)
  → Traefik Service (HTTP, 클러스터 내부)
  → Traefik Ingress: intro.woonapp.com
  → self-introduction Service:8080
  → Spring Boot Pod
```

앱 Service는 `ClusterIP`로만 노출됩니다. Tunnel Connector 토큰은 Git에 저장하지 않고, `cloudflare` 네임스페이스의 `cloudflared-token` Secret으로 별도 생성합니다. Cloudflare Dashboard의 Tunnel 및 Published Application 설정도 이 저장소에서 관리하지 않습니다.

## 최초 연결 순서

### 1. Argo CD 설치

Argo CD는 홈서버 Kubernetes 클러스터에 설치되어 있습니다. 아래는 새 클러스터를 구성할 때 최초 1회 실행하는 설치 절차이며, 비고가용성(single replica) 공식 매니페스트를 버전 `v3.6.1`로 고정합니다.

```bash
kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
kubectl apply --server-side --force-conflicts -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.6.1/manifests/install.yaml
kubectl rollout status deployment/argocd-server -n argocd --timeout=5m
```

Argo CD 서비스는 기본 ClusterIP로 유지합니다. 저장소가 비공개라면 Argo CD에 GitHub read-only 저장소 인증을 별도로 등록해야 합니다.

### 2. Cloudflare Tunnel 사용

Cloudflare Dashboard의 `Networking → Tunnels`에 원격 관리 Tunnel `homeServer-tunnel`을 생성했습니다. Connector 두 개가 Kubernetes에서 실행 중이며 Cloudflare에 연결된 상태입니다. 토큰은 `cloudflare` 네임스페이스의 `cloudflared-token` Secret에만 저장하고 Git에는 넣지 않습니다. Docker 안내 명령은 로컬에서 실행하지 않습니다.

### 3. Traefik 등록/업데이트 및 실제 내부 주소 확인

Argo CD Application은 공식 Helm chart `42.3.1`을 사용하고, 설정값은 이 저장소의 `infrastructure/traefik/values.yaml`에서 읽습니다. chart 리소스 자체는 Helm chart에서 렌더링되며 이 저장소에는 chart 설정 override만 둡니다. Service는 `ClusterIP`로 유지합니다.

기존 Traefik Application을 inline `helm.values`에서 Git의 values 파일을 읽는 multi-source 방식으로 바꾸려면, Git 변경을 `main`에 반영한 뒤 Application 정의를 한 번 적용합니다. 이 저장소에는 Application 정의를 자동 동기화하는 루트 Application이 없으므로 이 단계가 필요합니다. 그 다음부터 `values.yaml` 변경은 Argo CD가 감지해 동기화합니다.

```bash
kubectl apply -f argocd/traefik.yaml
kubectl rollout status deployment/traefik -n traefik --timeout=5m
kubectl get ingressclass
kubectl get svc -n traefik
```

### 4. 홈서버의 실제 Traefik 주소 확인

홈서버에 연결된 VS Code 터미널에서 확인합니다.

```bash
kubectl get nodes
kubectl get ingressclass
kubectl get svc -A | grep -i traefik
```

IngressClass가 `traefik`인지 확인합니다. Tunnel Published Application의 Service URL에는 조회 결과에 맞춰 다음 형식의 Traefik 내부 주소를 사용합니다.

```text
http://<service-name>.<namespace>.svc.cluster.local:<service-port>
```

예를 들어 Service가 `traefik` 네임스페이스의 `traefik`이고 포트가 80이면 `http://traefik.traefik.svc.cluster.local:80`입니다. 이 예시를 실제 조회 없이 그대로 사용하지 않습니다.

### 5. 토큰을 Kubernetes Secret으로 저장

토큰을 교체해야 하면 홈서버 터미널에서 아래 명령을 실행하고, 프롬프트에 Tunnel 토큰을 붙여 넣습니다. 입력값은 화면에 표시되지 않고 Git에도 기록되지 않습니다. Secret을 갱신한 뒤 Connector Pod를 재시작합니다.

```bash
kubectl create namespace cloudflare --dry-run=client -o yaml | kubectl apply -f -
read -rsp "Cloudflare Tunnel token: " TUNNEL_TOKEN
echo
printf '%s' "$TUNNEL_TOKEN" | kubectl create secret generic cloudflared-token \
  --namespace cloudflare \
  --from-file=token=/dev/stdin \
  --dry-run=client -o yaml | kubectl apply -f -
unset TUNNEL_TOKEN
kubectl rollout restart deployment/cloudflared -n cloudflare
```

Kubernetes Secret은 기본적으로 etcd에서 암호화되지 않을 수 있으므로 클러스터의 etcd 암호화와 RBAC 접근 제한을 확인합니다.

### 6. 앱을 Argo CD에 등록

저장소 변경 사항을 `main`에 반영한 다음 홈서버 터미널에서 새 Connector Application을 등록합니다. 웹 앱 Application이 아직 등록되지 않았다면 함께 적용합니다.

```bash
kubectl apply -f argocd/cloudflared.yaml
kubectl apply -f argocd/self-Intruduction.yaml
```

세 Application(Traefik, Connector, 웹 앱)은 자동 동기화됩니다. Git source를 organization 저장소로 바꾼 Application 정의는 이 저장소에서 자동 관리되지 않으므로, 변경을 `main`에 반영한 뒤 `kubectl apply -f argocd/traefik.yaml -f argocd/cloudflared.yaml -f argocd/self-Intruduction.yaml`로 기존 Application을 갱신하고 Argo CD의 새 저장소 접근 권한을 확인해야 합니다. Connector Deployment는 `cloudflare` 네임스페이스의 Secret을 읽고 `cloudflare/cloudflared:2026.10.7`을 2개 고정 replica로 실행합니다. `/ready` probe가 Cloudflare 연결 상태를 확인하며, 자동 스케일링은 사용하지 않습니다.

### 7. 공개 호스트 연결

Cloudflare Dashboard에서 `Networking → Tunnels → homeServer-tunnel → Routes → Add route → Published application`을 선택합니다. 현재 `intro.woonapp.com` 경로를 생성했고, Cloudflare가 Tunnel 대상 CNAME DNS 레코드를 자동으로 만들었습니다. Cloudflare Dashboard의 Tunnel과 Published Application 설정은 이 저장소가 관리하지 않습니다.

| 항목 | 값 |
| --- | --- |
| Subdomain | `intro` |
| Domain | `woonapp.com` |
| Service type | `HTTP` |
| Service URL | 위에서 확인한 Traefik 내부 Service 주소 |

저장 후 `intro.woonapp.com` DNS 레코드가 만들어졌는지 확인합니다. 자동 생성되지 않았다면 Tunnel UUID를 대상으로 한 Proxied CNAME을 추가합니다. 첫 연결은 `intro.woonapp.com`만 설정하고, 와일드카드 호스트는 별도 검증 후 추가합니다.

## Calico 유선망 주소 탐지 (자동 동기화)

Tigera Operator와 Calico CRD의 최초 설치는 클러스터 bootstrap 단계에서 유지합니다. `infrastructure/calico/wired-network.yaml`은 이미 존재하는 `Installation/default`에서 기존 `firstFound`를 끄고 **노드 IPv4 주소 탐지 방식만** `192.168.10.0/24`로 선언합니다. Argo CD는 server-side apply로 해당 필드를 관리하며 기존 IP pool과 VXLAN 설정은 유지합니다. `metadata.name: default`는 Kubernetes 리소스 이름이고, 파일명은 Git 내 이름입니다.

이 설정은 home-server2 전용이 아니라 **모든 Calico 노드에 적용**됩니다. 현재 home-server의 유선 주소도 선택되며, Kubernetes Node InternalIP와 API Server의 Wi-Fi 주소는 별개입니다. 향후 유선망 주소가 없는 클라우드 Worker를 추가하기 전에 주소 탐지·노드 간 터널 경로를 다시 설계해야 합니다.

변경을 `main`에 반영하고 실제 노드의 유선 주소·연결·복구 경로를 확인한 뒤에만 Application을 등록합니다. 기존 클러스터는 Tigera Operator가 `firstFound: true`를 소유하므로 새 CIDR만 server-side apply하면 `firstFound`가 남고, `false`를 선언해도 필드 소유권 충돌이 발생합니다. **처음 이전할 때 한 번** 아래 명령으로 두 값을 함께 변경한 다음 상태를 확인하고 `kubectl apply -f argocd/calico.yaml`로 Application을 등록합니다. 패치하는 순간 네트워크가 바뀌므로 복구 경로를 확보해야 합니다.

```bash
kubectl patch installation default --type=merge -p \
  '{"spec":{"calicoNetwork":{"nodeAddressAutodetectionV4":{"firstFound":false,"cidrs":["192.168.10.0/24"]}}}}'
kubectl get installation default -o jsonpath='{.spec.calicoNetwork.nodeAddressAutodetectionV4}'
kubectl apply -f argocd/calico.yaml
```

Application 등록 후 `main`의 설정 변경은 자동 반영되고 클러스터에서 수동 수정한 값은 self-heal로 되돌아갑니다. CNI 삭제를 방지하기 위해 자동 prune은 끄고 `Installation/default`에도 prune 방지 옵션을 둡니다. 등록 전에 이미 적용된 값을 Argo CD가 채택했는지 확인하고, Calico 노드 주소·`calico-node` 상태 및 노드 간 Pod 통신을 검증합니다.

문제가 생기면 Wi-Fi 관리 경로로 접속해 먼저 Argo CD 자동 동기화를 중지한 다음 기존 탐지 방식으로 복원합니다. Application 등록 이전이라면 첫 번째 명령은 생략합니다.

```bash
kubectl -n argocd patch application calico-network --type=merge -p \
  '{"spec":{"syncPolicy":{"automated":null}}}'
kubectl patch installation default --type=merge -p \
  '{"spec":{"calicoNetwork":{"nodeAddressAutodetectionV4":{"firstFound":true,"cidrs":null}}}}'
```

## 상태 확인

```bash
kubectl get pods -n cloudflare
kubectl logs -n cloudflare deployment/cloudflared
kubectl get svc,ingress -n self-introduction
curl -I https://intro.woonapp.com
```

Cloudflare Tunnel을 통한 외부 요청은 Cloudflare에서 HTTPS로 받고, Tunnel 내부에서 Traefik까지는 HTTP로 전달합니다. 앱 경로는 `apps/self-Intruduction/ingress.yaml`에서 선언적으로 관리합니다.
