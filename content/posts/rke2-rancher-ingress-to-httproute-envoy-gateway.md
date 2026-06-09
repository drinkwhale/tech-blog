---
title: "RKE2 + Rancher 환경에서 Ingress를 HTTPRoute로 마이그레이션하기 (Envoy Gateway)"
date: 2026-06-01T17:01:15+09:00
draft: false
tags: ["Kubernetes", "Gateway API", "HTTPRoute", "Envoy Gateway", "Rancher", "RKE2", "Ingress", "마이그레이션"]
categories: ["DevOps", "Kubernetes"]
description: "RKE2 + Rancher 클러스터에서 기존 Ingress 리소스를 Kubernetes Gateway API의 HTTPRoute로 전환하는 과정과 삽질 기록"
---

## TL;DR

> - RKE2 + Rancher 환경에서 Envoy Gateway를 설치하고 GatewayClass / Gateway / HTTPRoute를 구성했다.
> - 기존 Ingress 리소스를 HTTPRoute로 전환하는 과정에서 `parentRefs` 설정 누락, TLS Secret 참조 방식 차이로 삽질했다.
> - 마이그레이션 후 라우팅 규칙을 코드(YAML)로 명확히 표현할 수 있어 운영 복잡도가 줄었다.

---

## 배경 / 문제 상황

팀에서 운영 중인 RKE2 기반 클러스터에는 수십 개의 Ingress 리소스가 흩어져 있었다.
Ingress는 `nginx.ingress.kubernetes.io/` 어노테이션을 통해 라우팅 규칙을 설정하는데,
어노테이션이 늘어날수록 의도를 파악하기 어렵고 컨트롤러마다 동작이 달랐다.

Kubernetes Gateway API(HTTPRoute)는 이런 문제를 해결하기 위해 설계된 표준 스펙이다.
Envoy Gateway는 CNCF 프로젝트로, 이 스펙의 구현체 중 하나다.

**전환 목표:**
- Ingress 어노테이션 의존 제거
- 경로별 라우팅 규칙을 YAML로 명시적으로 표현
- 향후 트래픽 분산(Canary), 헤더 기반 라우팅 확장을 위한 기반 마련

**환경 스펙:**

| 항목 | 버전 |
|------|------|
| Rancher | 2.8.x |
| RKE2 | v1.29.x |
| Envoy Gateway | 1.0.x |
| Kubernetes Gateway API CRD | v1.1.x |

---

## 시도한 것들 (삽질 과정)

### 1차 시도 — CRD 없이 Envoy Gateway 설치

Envoy Gateway를 Helm으로 설치했는데 Pod가 계속 `CrashLoopBackOff` 상태였다.

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.0.0 \
  -n envoy-gateway-system --create-namespace
```

로그를 보니 `GatewayClass` CRD가 없다는 에러였다.
Gateway API CRD는 Envoy Gateway Helm 차트에 포함되지 않아 **별도로 설치**해야 한다.

```bash
# 실패 로그
failed to get API group resources: ...
no matches for kind "GatewayClass" in version "gateway.networking.k8s.io/v1"
```

### 2차 시도 — HTTPRoute parentRefs 설정 누락

CRD를 먼저 설치하고 다시 시도했는데, HTTPRoute를 적용해도 트래픽이 전혀 라우팅되지 않았다.
원인은 HTTPRoute의 `parentRefs`에 올바른 Gateway 이름을 지정하지 않았기 때문이었다.

기존 Ingress는 `ingressClassName`으로 컨트롤러를 지정하지만,
HTTPRoute는 **`parentRefs`로 특정 Gateway 오브젝트를 명시적으로 가리켜야** 한다.

```yaml
# 잘못된 설정 — parentRefs 없음
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
spec:
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: my-app-svc
          port: 80
```

### 3차 시도 — TLS Secret 네임스페이스 참조 문제

TLS를 설정하려고 기존 Ingress에서 쓰던 Secret을 그대로 참조했는데
Gateway가 `ResolvedRefs: False` 상태가 됐다.

Gateway API는 보안 정책상 **다른 네임스페이스의 Secret을 참조하려면 `ReferenceGrant`** 가 필요하다.
Ingress는 이 제약이 없어서 차이를 몰랐다.

---

## 해결 과정

### Step 1. Gateway API CRD 설치

```bash
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
```

### Step 2. Envoy Gateway 설치 (Helm)

```bash
helm install eg oci://docker.io/envoyproxy/gateway-helm \
  --version v1.0.0 \
  -n envoy-gateway-system --create-namespace

# 설치 확인
kubectl get pods -n envoy-gateway-system
```

### Step 3. GatewayClass 생성

```yaml
# gatewayclass.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

```bash
kubectl apply -f gatewayclass.yaml
kubectl get gatewayclass
# envoy-gateway   gateway.envoyproxy.io/...   Accepted   True
```

### Step 4. Gateway 생성

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: default
spec:
  gatewayClassName: envoy-gateway
  listeners:
    - name: http
      protocol: HTTP
      port: 80
    - name: https
      protocol: HTTPS
      port: 443
      tls:
        mode: Terminate
        certificateRefs:
          - name: my-tls-secret
            namespace: default
```

### Step 5. HTTPRoute 작성 (기존 Ingress 대응)

**기존 Ingress:**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: my-app-svc
                port:
                  number: 80
```

**HTTPRoute로 전환:**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  namespace: default
spec:
  parentRefs:                        # 반드시 Gateway를 명시
    - name: main-gateway
      namespace: default
  hostnames:
    - "app.example.com"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /api
      filters:
        - type: URLRewrite
          urlRewrite:
            path:
              type: ReplacePrefixMatch
              replacePrefixMatch: /   # rewrite-target: / 대응
      backendRefs:
        - name: my-app-svc
          port: 80
```

### Step 6. TLS Secret 크로스 네임스페이스 허용 (필요한 경우)

Secret이 다른 네임스페이스에 있다면 `ReferenceGrant`를 추가해야 한다.

```yaml
# referencegrant.yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway-tls
  namespace: cert-store          # Secret이 있는 네임스페이스
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: Gateway
      namespace: default         # Gateway가 있는 네임스페이스
  to:
    - group: ""
      kind: Secret
```

### Step 7. 상태 확인

```bash
# Gateway 상태
kubectl get gateway main-gateway -o yaml | grep -A 10 status

# HTTPRoute 상태 — Parents[].conditions 확인
kubectl get httproute my-app -o yaml | grep -A 20 status

# Envoy Gateway가 생성한 Envoy Proxy Pod
kubectl get pods -n envoy-gateway-system
```

---

## 결과

| 지표 | Before (Ingress) | After (HTTPRoute) |
|------|-----------------|-------------------|
| 라우팅 규칙 표현 방식 | 어노테이션 (암묵적) | YAML 스펙 (명시적) |
| 컨트롤러 종속 어노테이션 수 | 평균 6-8개 | 0개 |
| 트래픽 분산(Canary) 지원 | 별도 CRD 필요 | HTTPRoute weight로 내장 |
| TLS 설정 가시성 | Secret 이름만 | listener + ReferenceGrant로 명확 |

---

## 배운 점 / 회고

- **Gateway API는 Ingress보다 명시적이다.** 처음엔 설정이 많아 복잡해 보이지만, 의도가 YAML에 그대로 드러나서 리뷰가 편해진다.
- **`parentRefs`는 필수다.** Ingress의 `ingressClassName`과 역할이 같지만, 특정 *오브젝트*를 가리키는 방식이라 처음엔 낯설다.
- **ReferenceGrant는 처음부터 설계에 포함시켜야 한다.** Secret 네임스페이스 전략을 미리 정하지 않으면 마이그레이션 중간에 막힌다.
- Rancher UI에서는 아직 HTTPRoute를 직접 편집하는 뷰가 없어, `kubectl`로 관리해야 한다.

---

## 참고 자료

- [Kubernetes Gateway API 공식 문서](https://gateway-api.sigs.k8s.io/)
- [Envoy Gateway 공식 문서](https://gateway.envoyproxy.io/docs/)
- [Envoy Gateway Helm 설치 가이드](https://gateway.envoyproxy.io/docs/install/install-helm/)
- [ReferenceGrant 스펙](https://gateway-api.sigs.k8s.io/api-types/referencegrant/)
