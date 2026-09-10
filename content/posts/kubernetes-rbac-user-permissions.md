---
title: "Kubernetes RBAC: User 생성부터 권한 바인딩까지"
date: 2026-07-31T08:43:16+09:00
draft: false
tags: ["kubernetes", "rbac", "security", "access-control"]
categories: ["Kubernetes", "DevOps"]
description: "Kubernetes RBAC를 활용하여 특정 사용자에게 선택적 리소스 접근 권한을 부여하는 전체 절차를 단계별로 정리했습니다."
---

## TL;DR

Kubernetes RBAC로 사용자(john)에게 Gateway API 리소스만 관리할 수 있도록 권한을 제한했습니다. 환경은 Kubernetes 1.20+이며, User 인증서 생성 → Custom ClusterRole 정의 → ClusterRoleBinding 연결 단계를 거쳤습니다.

---

## 배경 / 문제 상황

클러스터 관리자로서 팀원들이 접근할 때마다 `admin` 권한으로 kubeconfig를 배포하는 식의 관리는 보안 위험이 높았습니다. 특히 Gateway API 팀원(john)에게는 **Gateway API 리소스만** 관리하게 하고, Pod 삭제나 네트워크 정책 변경 같은 위험한 작업은 막아야 했습니다.

### 환경 스펙

| 항목            | 사양                                               |
| --------------- | -------------------------------------------------- |
| Kubernetes 버전 | 1.20+                                              |
| 클러스터 타입   | 자체 관리형 (on-prem 또는 클라우드)                |
| 인증 방식       | Certificate-based (openssl)                        |
| 목표 사용자     | john (개발팀)                                      |
| 제한할 리소스   | Gateway API 전용 (gateways, httproutes, tcproutes) |
| 접근 범위       | Cluster-wide (모든 namespace)                      |

---

## 시도한 것들 (삽질 과정)

### 1차 시도: Role/RoleBinding으로 namespace 수준 제한

처음에는 특정 namespace(예: `production`)에서만 Gateway API를 관리하도록 `Role`과 `RoleBinding`을 사용했습니다.

```yaml
# 이 방식은 한 namespace에만 적용됨
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: gateway-api-manager
rules:
  - apiGroups: [gateway.networking.k8s.io]
    resources: [gateways, httproutes]
    verbs: [get, list, create, update, delete]
```

**문제점:**

- john이 `production` 이외의 namespace에서 Gateway API를 관리할 수 없었습니다
- 여러 namespace에서 같은 권한을 주려면 매번 RoleBinding을 복제해야 했습니다
- Namespace 수가 증가하면 관리 오버헤드가 심했습니다

### 2차 시도: ClusterRole 사용했으나 aggregation 라벨 누락

그 다음엔 ClusterRole을 시도했지만, 라벨 설정을 놓쳤습니다.

```yaml
# aggregation 라벨 없음
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gateway-api-manager
  # ❌ 이 라벨이 없었음!
rules:
  - apiGroups: [gateway.networking.k8s.io]
    resources: [gateways, httproutes]
    verbs: [get, list, create, update, delete]
```

**문제점:**

- ClusterRole 자체는 생성되었지만, admin 역할에 자동으로 통합되지 않았습니다
- `kubectl auth can-i create httproutes --as=john`을 실행하면 "yes"가 나왔지만, 실제 작업 시 "Forbidden" 오류가 발생했습니다
- aggregation 라벨(`rbac.authorization.k8s.io/aggregate-to-admin: "true"`)의 중요성을 깨달았습니다

---

## 해결 과정

### Step 1: User 인증서 생성

john 사용자의 private key와 인증서를 생성합니다.

```bash
# john을 위한 private key 생성
openssl genrsa -out john.key 2048

# Certificate Signing Request (CSR) 생성
openssl req -new -key john.key \
  -out john.csr \
  -subj "/CN=john/O=developers"

# Kubernetes CA로 인증서 서명 (365일 유효)
sudo openssl x509 -req -in john.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out john.crt \
  -days 365
```

### Step 2: john 사용자의 kubeconfig 생성

```bash
# 클러스터 정보 설정
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://YOUR_CLUSTER_IP:6443 \
  --kubeconfig=john-kubeconfig

# 사용자 자격증명 설정
kubectl config set-credentials john \
  --client-certificate=john.crt \
  --client-key=john.key \
  --embed-certs=true \
  --kubeconfig=john-kubeconfig

# context 설정
kubectl config set-context john@kubernetes \
  --cluster=kubernetes \
  --user=john \
  --kubeconfig=john-kubeconfig
```

### Step 3: Custom ClusterRole 생성 (aggregation 라벨 포함)

핵심은 `rbac.authorization.k8s.io/aggregate-to-admin: "true"` 라벨을 추가하는 것입니다.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gateway-api-manager
  labels:
    # ✅ 이 라벨이 중요! admin 역할에 자동 통합됨
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
rules:
# Gateway API 리소스 관리
- apiGroups:
  - gateway.networking.k8s.io
  resources:
  - gateways
  - httproutes
  - tcproutes
  verbs:
  - get
  - list
  - watch
  - create
  - update
  - patch
  - delete
  - deletecollection

# 상태 조회
- apiGroups:
  - gateway.networking.k8s.io
  resources:
  - gateways/status
  - httproutes/status
  verbs:
  - get
EOF
```

### Step 4: ClusterRoleBinding으로 john 사용자에게 권한 부여

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: john-gateway-api-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: gateway-api-manager
subjects:
# john 사용자를 대상으로 지정
- kind: User
  name: john
  apiGroup: rbac.authorization.k8s.io
EOF
```

### Step 5: 권한 확인 및 테스트

```bash
# john이 HTTPRoute를 생성할 수 있는가?
kubectl auth can-i create httproutes --as=john
# 결과: yes ✅

# john이 Pod를 삭제할 수 있는가? (없는 권한)
kubectl auth can-i delete pods --as=john
# 결과: no ❌

# john의 kubeconfig로 실제 작동 확인
kubectl --kubeconfig=john-kubeconfig get httproutes -A
# NAME         NAMESPACE   AGE
# my-route     default     5d

# Gateway 생성 시도 (권한 있음)
kubectl --kubeconfig=john-kubeconfig create gateway my-gateway \
  --class=istio \
  --listener-name=http \
  --port=80
# gateway.gateway.networking.k8s.io/my-gateway created ✅
```

---

## 결과

| 항목                    | Before                         | After                                    |
| ----------------------- | ------------------------------ | ---------------------------------------- |
| john의 권한 범위        | admin (모든 리소스)            | Gateway API만 관리 가능                  |
| Pod 삭제 가능 여부      | 가능 (위험!)                   | 불가능 ✅                                |
| 여러 namespace에서 적용 | RoleBinding 복제 필요          | ClusterRole 자동 적용                    |
| 관리 방식               | 수동으로 매번 RoleBinding 생성 | 한 번의 ClusterRole + ClusterRoleBinding |

---

## 배운 점 / 회고

### 1. Aggregation Label의 중요성

`rbac.authorization.k8s.io/aggregate-to-admin: "true"` 라벨은 단순한 메타데이터가 아니라, 해당 ClusterRole의 규칙을 `system:admin` 역할에 자동으로 포함시키는 메커니즘입니다. 이를 놓치면 권한이 있어도 동작하지 않을 수 있습니다.

### 2. ClusterRole vs Role

- **Role**: 특정 namespace 내에서만 적용
- **ClusterRole**: 전체 클러스터에 적용

여러 namespace에서 동일한 권한이 필요하면 ClusterRole을 사용해야 관리 오버헤드를 줄일 수 있습니다.

### 3. 권한 테스트는 필수

`kubectl auth can-i` 명령어로 사전에 권한을 검증한 후 배포하면, 런타임 오류를 예방할 수 있습니다.

### 4. Group을 활용한 권한 관리

여러 사용자에게 같은 권한을 주려면, Group을 사용하는 것이 확장성이 높습니다:

```yaml
subjects:
  - kind: Group
    name: gateway-team
    apiGroup: rbac.authorization.k8s.io
```

### 5. 다음 번에는 이렇게 할 것

- ClusterRole 생성 시 aggregation 라벨 체크리스트 추가
- RBAC 정책을 Helm Chart나 Kustomize로 IaC 화
- 정기적으로 `kubectl describe clusterrolebinding` 으로 바인딩 상태 감시

---

## 참고 자료

- [Kubernetes RBAC 공식 문서](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [Gateway API 리소스 정의](https://gateway-api.sigs.k8s.io/)
- [kubectl auth can-i](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#auth)
- [API Server 인증 (Certificate)](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- [ClusterRole Aggregation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles)
