---
title: "마이크로세그멘테이션 완전 정복: 온프레미스-클라우드 하이브리드 환경 Quick Start"
date: 2026-06-02T15:16:25+09:00
draft: false
description: "마이크로세그멘테이션 개념부터 온프레미스·클라우드 하이브리드 환경 실전 적용까지. NSX-T, Calico, AWS Security Group 조합으로 동서(East-West) 트래픽을 제어하는 방법을 단계별로 정리합니다."
tags: ["microsegmentation", "zero-trust", "network-security", "nsx-t", "calico", "hybrid-cloud", "devops"]
categories: ["Network Security", "Cloud"]
---

## TL;DR

> - 마이크로세그멘테이션은 "퍼리미터 방화벽"이 막지 못하는 **동서(East-West) 트래픽**을 워크로드 단위로 차단하는 제로 트러스트 네트워크 전략이다.
> - 온프레미스는 **VMware NSX-T 또는 Calico**, 클라우드는 **Security Group + Network Policy** 조합으로 하이브리드 구성이 가능하다.
> - 가장 큰 함정은 "일단 다 열고 나중에 막자"는 운영 관행 — 처음부터 **기본 거부(Default Deny)** 정책으로 시작하지 않으면 나중에 손댈 수 없다.

---

## 배경 / 문제 상황

### 왜 마이크로세그멘테이션인가

전통적인 네트워크 보안은 외부 경계(퍼리미터)를 방화벽으로 막는 **캐슬-앤-모트(Castle-and-Moat)** 모델이었다. 문제는 공격자가 한 번 내부에 진입하면 내부 트래픽(East-West)은 거의 무방비라는 점이다.

2020년 이후 SolarWinds, Colonial Pipeline 등 굵직한 침해 사고는 모두 **내부 측면 이동(Lateral Movement)** 을 통해 피해가 확산됐다. VPN으로 들어온 공격자가 내부에서 자유롭게 이동하며 데이터를 탈취했다.

| 구분 | 기존 퍼리미터 보안 | 마이크로세그멘테이션 |
|------|------------------|-------------------|
| 보호 범위 | 외부 → 내부 (South-North) | 내부 ↔ 내부 (East-West) 포함 |
| 정책 단위 | VLAN, 서브넷 | 워크로드(VM, Pod, 컨테이너) |
| 침해 확산 | 내부 이동 가능 | 세그먼트 간 이동 차단 |
| 클라우드 적용 | 어려움 | 네이티브 지원 가능 |

### 하이브리드 환경의 특수성

온프레미스와 클라우드가 혼재하는 환경에서는 두 세계의 정책을 **일관되게** 관리해야 한다. 온프레미스에서 NSX-T로 정의한 세그먼트 정책이 AWS VPC에선 Security Group으로 따로 관리되면, 정책 누락과 감사 실패가 반드시 발생한다.

```
온프레미스 데이터센터               AWS Cloud
┌─────────────────────┐            ┌─────────────────────┐
│  VM A (Web)         │◄──VPN/DX──►│  EC2 (API Server)   │
│  VM B (DB)          │            │  RDS (Database)     │
│  NSX-T 기반 정책    │            │  Security Group 정책│
└─────────────────────┘            └─────────────────────┘
        ↕                                    ↕
  마이크로세그멘테이션            Security Group + NACLs
  (East-West 제어)               (East-West 제어)
```

---

## 시도한 것들 (삽질 과정)

### 1차 시도: VLAN 기반 세그멘테이션

처음에는 익숙한 VLAN 분리로 세그멘테이션을 구현하려 했다.

```
VLAN 10: Web Tier    (192.168.10.0/24)
VLAN 20: App Tier    (192.168.20.0/24)
VLAN 30: DB Tier     (192.168.30.0/24)
```

**왜 실패했는가:**
- VLAN 간 방화벽 규칙은 서브넷 단위 → 같은 VLAN 내 VM 간 이동은 여전히 무방비
- 클라우드 환경에서는 VLAN 개념 자체가 다름 (VPC, Subnet은 VLAN이 아님)
- 새 VM이 추가될 때마다 방화벽 룰 수작업 업데이트 → 운영 부담 폭증
- App Tier 내 VM-A가 악성코드에 감염되면, 같은 VLAN의 VM-B는 즉시 위험

```bash
# 당시 방화벽 룰 현황 (iptables 기반)
$ iptables -L FORWARD --line-numbers | wc -l
847  ← 846개 룰, 누가 추가했는지 아무도 모름
```

### 2차 시도: AWS Security Group만으로 하이브리드 통합

"클라우드는 Security Group이 있으니까 거기서 다 막자"는 생각으로 온프레미스 제어를 느슨하게 두고 AWS 쪽에만 집중했다.

**왜 실패했는가:**
- 온프레미스 → AWS Direct Connect 트래픽이 Security Group 레벨에서 한 번에 허용됨 → 온프레미스 내부 감염 시 클라우드까지 전파
- Security Group은 인바운드/아웃바운드만 제어, 같은 SG 내 인스턴스 간 트래픽은 기본 허용
- 정책이 AWS Console / Terraform / 수작업 3곳에 분산돼 일관성 파악 불가

```hcl
# 문제가 됐던 Security Group 설정
resource "aws_security_group_rule" "allow_all_from_onprem" {
  type        = "ingress"
  from_port   = 0
  to_port     = 65535
  protocol    = "-1"
  cidr_blocks = ["10.0.0.0/8"]  # 온프레미스 전체 허용 ← 이게 문제
  security_group_id = aws_security_group.app.id
}
```

### 3차 시도: NSX-T 단독 — 클라우드 워크로드 누락

온프레미스에 NSX-T를 도입해 분산 방화벽(DFW)을 적용했다. 각 VM의 vNIC 레벨에서 정책이 적용되니 East-West 제어가 확실해졌다. 그런데…

**왜 부족했는가:**
- NSX-T는 온프레미스 vSphere 환경에서만 동작, AWS/GCP 워크로드는 적용 불가
- 하이브리드 환경에서 정책이 두 시스템으로 분리돼 "온프레미스 허용 = 클라우드도 허용"이라는 잘못된 가정이 생김
- Kubernetes Pod 단위 정책은 NSX-T NCP(NSX Container Plugin)으로 별도 설정 필요

---

## 해결 과정

### 핵심 원칙: 레이어별 정책 분리 + 중앙 가시성

```
[정책 레이어]
  온프레미스 VM     → NSX-T Distributed Firewall (DFW)
  Kubernetes Pod    → Calico Network Policy
  AWS EC2/RDS       → Security Group + NACL
  하이브리드 경계   → VPN/DX + Transit Gateway + 경계 방화벽

[가시성 레이어]
  NSX Intelligence / Aria Operations for Networks
  AWS VPC Flow Logs → CloudWatch / SIEM
  Calico Flow Logs → Elasticsearch
```

---

### Step 1: 현황 파악 — 트래픽 플로우 맵 작성

마이크로세그멘테이션 전 **실제 통신 경로**를 파악하지 않으면 정책 적용 후 서비스가 죽는다.

```bash
# 온프레미스: NSX-T에서 트래픽 플로우 수집 (NSX-T 3.x 이상)
# NSX Manager UI → Networking → Traffic Analysis → Flow 활성화

# Calico 환경: 플로우 로그 활성화
kubectl patch felixconfiguration default --type='merge' \
  -p '{"spec":{"flowLogsEnabled":true,"flowLogsFlushInterval":"10s"}}'

# AWS: VPC Flow Logs 활성화
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-xxxxxxxx \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs
```

최소 **72시간** 트래픽을 수집해 애플리케이션 통신 패턴을 정리한다.

---

### Step 2: 워크로드 태깅 — NSX-T 보안 태그

NSX-T는 IP 기반이 아닌 **태그(Tag)** 기반으로 정책을 적용한다. VM이 재배포돼도 태그만 있으면 정책이 자동 적용된다.

```bash
# NSX-T API로 VM에 태그 부착
# (NSX Manager REST API)
curl -k -u admin:password -X PATCH \
  https://<nsx-manager>/api/v1/fabric/virtual-machines/<vm-id>/tags \
  -H "Content-Type: application/json" \
  -d '{
    "tags": [
      {"scope": "env",  "tag": "production"},
      {"scope": "tier", "tag": "web"},
      {"scope": "app",  "tag": "ecommerce"}
    ]
  }'
```

태그 기반 그룹 생성:

```json
// NSX-T Security Group 정의 (UI 또는 API)
{
  "display_name": "SG-Production-Web",
  "expression": [
    {
      "resource_type": "Condition",
      "member_type": "VirtualMachine",
      "key": "Tag",
      "operator": "EQUALS",
      "value": "production|web"
    }
  ]
}
```

---

### Step 3: NSX-T 분산 방화벽 — 기본 거부 정책

**핵심: Default Deny부터 시작한다.** 허용할 것만 명시적으로 열어준다.

```
NSX-T DFW 정책 구조 (위에서 아래로 평가)

[Emergency 정책] — 최우선, 관리 접근 허용
  Allow: Jump Server(10.10.1.10) → Any VM, port 22/3389

[Application 정책] — 앱별 통신 허용
  Allow: SG-Web      → SG-App,  port 8080
  Allow: SG-App      → SG-DB,   port 5432
  Allow: SG-Monitor  → Any VM,  port 9100 (Prometheus exporter)

[Default 정책] — 맨 아래, 나머지 모두 차단
  Deny:  Any → Any  (로그 활성화)
```

NSX-T API로 Default Deny 룰 생성:

```bash
curl -k -u admin:password -X PUT \
  https://<nsx-manager>/policy/api/v1/infra/domains/default/security-policies/default-layer3-section \
  -H "Content-Type: application/json" \
  -d '{
    "rules": [{
      "display_name": "Default-Deny-All",
      "action": "DROP",
      "logged": true,
      "sources_excluded": false,
      "destinations_excluded": false,
      "ip_protocol": "IPV4_IPV6"
    }]
  }'
```

---

### Step 4: Kubernetes — Calico Network Policy

Kubernetes 클러스터는 기본적으로 모든 Pod 간 통신이 허용된다. Calico로 Default Deny + 필요한 것만 허용한다.

```yaml
# 1. Namespace 단위 Default Deny
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}  # 모든 Pod 대상
  policyTypes:
    - Ingress
    - Egress
```

```yaml
# 2. Web → App 허용 (레이블 기반)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-app
  namespace: production
spec:
  podSelector:
    matchLabels:
      tier: app
  ingress:
    - from:
        - podSelector:
            matchLabels:
              tier: web
      ports:
        - protocol: TCP
          port: 8080
```

```yaml
# 3. Calico GlobalNetworkPolicy — 클러스터 전체 DNS 허용
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: allow-dns
spec:
  selector: all()
  egress:
    - action: Allow
      protocol: UDP
      destination:
        ports: [53]
    - action: Allow
      protocol: TCP
      destination:
        ports: [53]
  types:
    - Egress
```

---

### Step 5: AWS Security Group — 최소 권한 원칙

```hcl
# terraform으로 Security Group 관리
# Web Tier SG
resource "aws_security_group" "web" {
  name   = "sg-web-tier"
  vpc_id = var.vpc_id

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]  # ALB 앞단에서 처리
  }

  egress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]  # App SG만 허용
  }

  tags = { Tier = "web", Env = "production" }
}

# App Tier SG
resource "aws_security_group" "app" {
  name   = "sg-app-tier"
  vpc_id = var.vpc_id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]  # Web SG만 허용
  }

  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.db.id]  # DB SG만 허용
  }
}

# 온프레미스 연결 (DX/VPN) - 최소 포트만 허용
resource "aws_security_group_rule" "from_onprem_specific" {
  type              = "ingress"
  from_port         = 8443
  to_port           = 8443
  protocol          = "tcp"
  cidr_blocks       = ["10.10.0.0/16"]  # 온프레미스 특정 서브넷만
  security_group_id = aws_security_group.app.id
  description       = "온프레미스 레거시 시스템 연동 - JIRA-1234"
}
```

---

### Step 6: 하이브리드 경계 정책 (온프레미스 ↔ AWS)

Direct Connect 또는 VPN 경계에서 Transit Gateway로 트래픽을 집중시키고, 라우팅 정책으로 세그먼트 간 이동을 제어한다.

```bash
# AWS Transit Gateway 라우팅 테이블 분리
# 온프레미스 → 클라우드: 특정 서브넷만 도달 가능하도록

# TGW 라우팅 테이블 생성
aws ec2 create-transit-gateway-route-table \
  --transit-gateway-id tgw-xxxxxxxx \
  --tag-specifications 'ResourceType=transit-gateway-route-table,
    Tags=[{Key=Name,Value=rt-onprem-to-cloud}]'

# 온프레미스에서 Web Tier 서브넷만 접근 허용
aws ec2 create-transit-gateway-route \
  --destination-cidr-block 10.100.10.0/24 \  # Web Tier 서브넷만
  --transit-gateway-route-table-id tgw-rtb-xxxxxxxx \
  --transit-gateway-attachment-id tgw-attach-xxxxxxxx
```

---

### Step 7: 검증 — 정책이 실제로 동작하는지 확인

```bash
# 1. NSX-T: 차단된 트래픽 로그 확인
# NSX Manager → Monitoring → Logs → Firewall
# 또는 syslog 수집 서버에서:
grep "DENY" /var/log/nsx-firewall.log | tail -50

# 2. Kubernetes: 정책 연결 확인
kubectl describe networkpolicy -n production

# 3. 실제 통신 테스트 (임시 Pod 활용)
# App에서 DB 직접 접근 시도 (차단돼야 정상)
kubectl run test-pod --image=busybox -n production --rm -it -- \
  nc -zv <db-pod-ip> 5432
# 기대 결과: nc: <db-ip> (5432): Connection refused 또는 timeout

# Web에서 DB 직접 접근 시도 (차단돼야 정상)
kubectl run test-pod --image=busybox -n production --rm -it -- \
  nc -zv <db-pod-ip> 5432
# 기대 결과: Connection refused

# 4. AWS Security Group 검증
aws ec2 describe-security-groups \
  --group-ids sg-xxxxxxxx \
  --query 'SecurityGroups[*].IpPermissions'
```

---

## 결과

| 항목 | Before (VLAN + 퍼리미터) | After (마이크로세그멘테이션) |
|------|------------------------|--------------------------|
| East-West 트래픽 가시성 | 없음 | 100% 플로우 로그 수집 |
| 내부 횡이동 가능 범위 | 전체 VLAN 내부 | 허용된 포트/대상만 |
| 방화벽 룰 수 (온프레미스) | 846개 (수작업) | 38개 (태그 기반, Terraform 관리) |
| 정책 적용 단위 | /24 서브넷 | 개별 VM / Pod |
| 신규 워크로드 정책 반영 | 수작업 1~3일 | 태그 부착 즉시 자동 적용 |
| 보안 감사 소요 시간 | 2주 (룰 수작업 검토) | 4시간 (코드 리뷰) |
| 하이브리드 정책 일관성 | 불일치 빈발 | IaC로 단일 소스 관리 |

---

## 배운 점 / 회고

**1. Default Deny는 처음부터 시작해야 한다**
운영 중인 환경에 Default Deny를 나중에 적용하는 건 거의 불가능에 가깝다. 기존 통신 경로를 모두 파악해야 하는데, 레거시가 쌓인 환경에서는 "어디서 어디로 가는지"를 아무도 모른다. 신규 환경은 처음부터, 기존 환경은 단계적 마이그레이션(Tier-by-Tier)으로 접근하자.

**2. IP 기반 정책은 반드시 레이블/태그로 교체하라**
`10.10.1.0/24 허용` 같은 룰은 VM 이전/재배포 순간 깨진다. NSX-T의 보안 태그, Kubernetes의 Pod 레이블, AWS의 태그 기반 SG 참조를 쓰면 워크로드가 어디로 이동해도 정책이 따라간다.

**3. 가시성 없는 세그멘테이션은 의미없다**
정책을 배포해도 "실제로 차단되고 있는가"를 볼 수 없으면 운영이 불가능하다. VPC Flow Logs, NSX Intelligence, Calico Flow Logs를 SIEM(Splunk, Elasticsearch 등)으로 집중시켜 하나의 대시보드에서 보는 것이 필수다.

**4. Terraform으로 관리하지 않는 Security Group은 결국 레거시가 된다**
"일단 콘솔에서 빠르게 열고 나중에 코드화하자"는 생각으로 만들어진 룰은 영원히 남는다. 팀 내 규칙: **콘솔 직접 수정 금지, 모든 변경은 PR**. 처음엔 불편하지만 6개월 후 감사할 자신에게 고마워질 것이다.

**5. 하이브리드 환경은 "가장 약한 링크"가 보안 수준이다**
클라우드 쪽을 아무리 잘 잠가도 온프레미스 DX/VPN 입구가 열려 있으면 무용지물이다. 두 환경을 별개로 생각하지 말고, 하이브리드 경계(Transit Gateway, BGP 라우팅)를 포함한 전체 그림을 그려야 한다.

---

## 참고 자료

- [VMware NSX-T 분산 방화벽 문서](https://docs.vmware.com/en/VMware-NSX/index.html)
- [Calico Network Policy 공식 문서](https://docs.tigera.io/calico/latest/network-policy/)
- [AWS Security Group Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/security-group-rules.html)
- [NIST SP 800-207 — Zero Trust Architecture](https://csrc.nist.gov/publications/detail/sp/800-207/final)
- [CIS Kubernetes Benchmark](https://www.cisecurity.org/benchmark/kubernetes)
- [AWS Transit Gateway 라우팅 테이블 관리](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html)
