# Kubernetes RBAC 전체 시나리오: 처음부터 끝까지

**시나리오:** `john` 사용자가 Gateway API 리소스를 관리할 수 있게 하기

---

## 📋 전체 흐름도

```
1. User 생성 (john)
         ↓
2. Custom ClusterRole 생성 (gateway-api-manage)
         ↓
3. 라벨 붙이기 (aggregate-to-admin)
         ↓
4. ClusterRoleBinding으로 연결
         ↓
5. User가 실제로 권한 사용
```

---

## 🔑 Step 1: User 생성 및 인증 설정

### 1-1) User 인증서 생성 (openssl)

```bash
# john을 위한 private key 생성
openssl genrsa -out john.key 2048

# Certificate Signing Request 생성
openssl req -new -key john.key \
  -out john.csr \
  -subj "/CN=john/O=developers"

# Kubernetes의 CA로 인증서 서명
sudo openssl x509 -req -in john.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out john.crt \
  -days 365

# john의 kubeconfig 생성
kubectl config set-cluster kubernetes \
  --certificate-authority=/etc/kubernetes/pki/ca.crt \
  --embed-certs=true \
  --server=https://YOUR_CLUSTER_IP:6443 \
  --kubeconfig=john-kubeconfig

kubectl config set-credentials john \
  --client-certificate=john.crt \
  --client-key=john.key \
  --embed-certs=true \
  --kubeconfig=john-kubeconfig

kubectl config set-context john@kubernetes \
  --cluster=kubernetes \
  --user=john \
  --kubeconfig=john-kubeconfig
```

**결과:**

- `john` 사용자가 Kubernetes 클러스터에 접근 가능한 상태
- 하지만 아직 권한은 없음

---

## 📦 Step 2: Custom ClusterRole 생성

### 2-1) Gateway API 관리용 ClusterRole

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: gateway-api-manager
  labels:
    # 이 라벨이 핵심!
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

**확인:**

```bash
kubectl get clusterrole gateway-api-manager
# NAME                    CREATED AT
# gateway-api-manager     2026-07-31T10:00:00Z
```

---

## 🔗 Step 3: ClusterRoleBinding으로 User에게 권한 부여

### 3-1) User `john`을 ClusterRole에 연결

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

**확인:**

```bash
kubectl get clusterrolebinding john-gateway-api-manager
# NAME                          ROLE                            AGE
# john-gateway-api-manager      ClusterRole/gateway-api-manager  10s
```

---

## 🎯 Step 4: 권한 확인 (테스트)

### 4-1) john 사용자의 권한 확인

```bash
# HTTPRoute를 생성할 수 있는가?
kubectl auth can-i create httproutes --as=john
# 결과: yes ✅

# Pod를 삭제할 수 있는가? (없는 권한)
kubectl auth can-i delete pods --as=john
# 결과: no ❌

# Gateway를 수정할 수 있는가?
kubectl auth can-i patch gateways --as=john
# 결과: yes ✅
```

---

## 🚀 Step 5: User가 실제로 권한 사용

### 5-1) john 사용자의 kubeconfig 배포

```bash
# john의 kubeconfig 파일을 john 사용자에게 전달
cp john-kubeconfig ~/.kube/config-john
# 또는
export KUBECONFIG=john-kubeconfig
```

### 5-2) john이 자신의 kubeconfig로 접속

```bash
# john 사용자로 실행
kubectl --kubeconfig=john-kubeconfig get httproutes -A
# NAME         NAMESPACE   AGE
# my-route     default     5d

# Gateway 생성 시도 (권한 있음)
kubectl --kubeconfig=john-kubeconfig create gateway my-gateway \
  --class=istio \
  --listener-name=http \
  --port=80
# gateway.gateway.networking.k8s.io/my-gateway created ✅

# Pod 삭제 시도 (권한 없음)
kubectl --kubeconfig=john-kubeconfig delete pod nginx
# Error from server (Forbidden): pods "nginx" is forbidden:
# User "john" cannot delete resource "pods" in API group "" in the namespace "default"
# ❌ 권한 거부됨
```

---

## 📊 전체 구조도

```
┌─────────────────────────────────────────────────────────┐
│ User: john                                               │
│ (인증서: john.crt, john.key)                             │
└────────────────┬────────────────────────────────────────┘
                 │ (kubeconfig로 인증)
                 ↓
┌─────────────────────────────────────────────────────────┐
│ ClusterRoleBinding: john-gateway-api-manager            │
│ (john ← gateway-api-manager 연결)                       │
└────────────────┬────────────────────────────────────────┘
                 │ (참조)
                 ↓
┌─────────────────────────────────────────────────────────┐
│ ClusterRole: gateway-api-manager                        │
│ - gateways: get, list, create, update, delete           │
│ - httproutes: get, list, create, update, delete         │
│ - tcproutes: get, list, create, update, delete          │
└────────────────┬────────────────────────────────────────┘
                 │ (자동 통합)
                 ↓
┌─────────────────────────────────────────────────────────┐
│ ClusterRole: system:admin (aggregated)                  │
│ 이 ClusterRole에 자동으로 규칙이 추가됨                   │
└─────────────────────────────────────────────────────────┘
```

---

## 🔍 Step 6: 권한 확인 - 상세 조회

### 6-1) john에게 정확히 어떤 권한이 있는가?

```bash
# john의 모든 권한 조회
kubectl get clusterrole gateway-api-manager -oyaml
```

**결과:**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    rbac.authorization.k8s.io/aggregate-to-admin: "true"
  name: gateway-api-manager
rules:
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
  - apiGroups:
      - gateway.networking.k8s.io
    resources:
      - gateways/status
      - httproutes/status
    verbs:
      - get
```

### 6-2) john의 바인딩 확인

```bash
kubectl describe clusterrolebinding john-gateway-api-manager
```

**결과:**

```
Name:         john-gateway-api-manager
Labels:       <none>
Annotations:  <none>
Role:
  Kind:  ClusterRole
  Name:  gateway-api-manager
Subjects:
  Kind  Name  Namespace
  ----  ----  ---------
  User  john
```

---

## 📝 실무 체크리스트

```bash
# ✅ 1. User 생성 확인
kubectl config view | grep john

# ✅ 2. ClusterRole 생성 확인
kubectl get clusterrole gateway-api-manager

# ✅ 3. ClusterRoleBinding 생성 확인
kubectl get clusterrolebinding john-gateway-api-manager

# ✅ 4. 권한 테스트
kubectl auth can-i create httproutes --as=john

# ✅ 5. john의 kubeconfig 전달
cat john-kubeconfig

# ✅ 6. john이 실제로 작동하는지 확인
kubectl --kubeconfig=john-kubeconfig get httproutes
```

---

## 🎓 핵심 정리

| 단계      | 내용                    | 명령/파일                    |
| --------- | ----------------------- | ---------------------------- |
| **1단계** | User 인증서 생성        | openssl 명령                 |
| **2단계** | Custom ClusterRole 생성 | gateway-api-manager          |
| **3단계** | 라벨 붙이기             | `aggregate-to-admin: "true"` |
| **4단계** | ClusterRoleBinding 생성 | john ↔ gateway-api-manager   |
| **5단계** | 권한 테스트             | `kubectl auth can-i`         |
| **6단계** | User가 실제 사용        | john의 kubeconfig로 접속     |

---

## ⚠️ 추가 팁

### 여러 User에게 같은 권한 부여하려면?

```bash
# 방법 1: Group으로 관리 (권장)
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: gateway-api-managers
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: gateway-api-manager
subjects:
- kind: Group
  name: gateway-team  # LDAP/OAuth의 그룹
  apiGroup: rbac.authorization.k8s.io
```

### 특정 Namespace에만 권한 주려면?

```bash
# ClusterRole 대신 Role 사용
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production  # 특정 namespace만
  name: gateway-api-manager
rules:
- apiGroups:
  - gateway.networking.k8s.io
  resources:
  - gateways
  verbs:
  - get
  - list
  - create
  - update
```

```bash
# RoleBinding 사용
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: production
  name: john-gateway-api-manager
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: gateway-api-manager
subjects:
- kind: User
  name: john
```

---

## 📌 요약

이 가이드는 다음을 설명합니다:

1. **User 인증** - john 사용자가 클러스터에 접근할 수 있도록 인증서 생성
2. **권한 정의** - Custom ClusterRole로 Gateway API 관리 권한 정의
3. **권한 바인딩** - ClusterRoleBinding으로 john에게 권한 부여
4. **자동 통합** - aggregation 라벨을 사용한 자동 권한 관리
5. **권한 검증** - kubectl auth can-i로 권한 확인
6. **실제 사용** - john이 실제로 Gateway API를 관리

이제 john 사용자가 완벽하게 Gateway API를 관리할 수 있습니다! 🎉
