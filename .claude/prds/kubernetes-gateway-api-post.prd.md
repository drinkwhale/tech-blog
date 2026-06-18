# Kubernetes Gateway API: Ingress를 넘어선 차세대 트래픽 라우팅

## Problem

Kubernetes Gateway API는 Ingress의 구조적 한계를 해결하는 공식 차세대 표준이며, Ingress-NGINX Controller의 2026년 3월 EOL로 마이그레이션이 실질적인 과제가 되었다. 그러나 한국어로 개념부터 동작 원리까지 체계적으로 정리된 콘텐츠가 부족해 실무자들이 공식 영문 문서를 분산 탐색해야 한다.

## Evidence

- `/deep-research` 수행 결과: Ingress-NGINX EOL(2026.03.31) 확인 — 신규 보안 패치 없음 (출처: oneuptime.com/blog)
- Gateway API v1.5(2026.02) 출시로 TCPRoute·GRPCRoute 포함 Stable 승격 완료
- CNCF, Kong, Tigera, Nginx 등 주요 업체들이 일제히 Gateway API 전환 가이드 발행 중
- 개인 블로그 독자 피드백 기반 — "Ingress 다음 뭘 써야 하는지 모르겠다"는 질문 유형 증가

## Users

- **Primary**: Kubernetes 실무자 (DevOps/인프라 엔지니어, 입문~중급). 기존 Ingress를 사용 중이며 Gateway API 전환을 검토하고 있음.
- **Not for**: Kubernetes 자체를 처음 접하는 완전 입문자 / 특정 구현체(Istio, Cilium) 심화 튜닝을 원하는 고급 사용자.

## Hypothesis

We believe **"무엇인가 → 왜 필요한가 → 어떻게 동작하는가" 구조에 Mermaid 다이어그램을 결합한 개념 정리 글**이 **Gateway API 전환 결정을 앞둔 DevOps 실무자의 이해를 돕고** for **Kubernetes 실무자 (입문~중급)**.
We'll know we're right when **블로그 체류 시간 3분 이상, GitHub 공유 또는 북마크 피드백 수신**.

## Success Metrics

| Metric                  | Target                                     | How measured              |
| ----------------------- | ------------------------------------------ | ------------------------- |
| 글 구조 완성도          | 무엇·왜·어떻게 3섹션 + 다이어그램 2개 이상 | 최종 파일 리뷰            |
| Mermaid 다이어그램 포함 | 2개 이상 (아키텍처 + 흐름)                 | 파일 내 `mermaid` 블록 수 |
| 참고 출처 인용          | 6개 이상                                   | 참고 자료 섹션 링크 수    |
| draft → 발행            | `draft: false` 전환 후 빌드 성공           | `hugo --minify` 오류 없음 |

## Scope

**MVP** — 단일 Markdown 파일로 아래 구조를 갖춘 블로그 글 1편 + 재사용 가능한 `tech-concept-post` 스킬

```
제목 (블로그 친화적, "조사보고서" 제외)
TL;DR (3줄)
1. 무엇인가  — 정의 + 비교 표
2. 왜 필요한가 — 기존 문제 + Mermaid 아키텍처 다이어그램
3. 어떻게 동작하는가 — 리소스 구조 + Mermaid 흐름 다이어그램 + YAML 예시
4. 구현체 선택 가이드 — 비교 표
5. 참고 자료
```

**Out of scope**

- 특정 구현체(Cilium, Istio) 상세 설치 튜토리얼 — 별도 글로 분리
- 실제 마이그레이션 핸즈온 (kubectl 명령어 단계별) — 후속 글
- 성능 벤치마크 데이터 수집 — 3차 글

## Blog Post Structure Detail

### 섹션 1: 무엇인가 (What)

- Gateway API 한 줄 정의
- Ingress vs Gateway API 비교 표 (지원 프로토콜, 역할 분리, 이식성)
- 주요 리소스 요약 표 (GatewayClass / Gateway / HTTPRoute / GRPCRoute)

### 섹션 2: 왜 필요한가 (Why)

- Ingress 3대 한계: 프로토콜 제한 / Annotation 지옥 / 역할 분리 불가
- Ingress-NGINX EOL(2026.03.31) 타임라인 언급
- **Mermaid 다이어그램 1**: Ingress 아키텍처 vs Gateway API 아키텍처 비교

### 섹션 3: 어떻게 동작하는가 (How)

- 3계층 역할 모델 (인프라 공급자 → 클러스터 운영자 → 앱 개발자)
- **Mermaid 다이어그램 2**: 트래픽 흐름 (Client → GatewayClass → Gateway → HTTPRoute → Service → Pod)
- YAML 예시: GatewayClass, Gateway, HTTPRoute 각 1개
- 크로스 네임스페이스 라우팅 개념

### 섹션 4: 구현체 선택 가이드

- 비교 표 (Cilium / Istio / Envoy GW / NGINX / Kong / Traefik)
- 선택 시나리오별 추천

## Delivery Milestones

| #   | Milestone       | Outcome                                                     | Status  | Plan |
| --- | --------------- | ----------------------------------------------------------- | ------- | ---- |
| 1   | 스킬 생성       | `tech-concept-post` 스킬 파일 완성                          | pending | —    |
| 2   | 글 파일 생성    | `hugo new content posts/k8s-gateway-api.md`                 | pending | —    |
| 3   | 초안 작성       | `/write-post` 또는 `/article-writing` 으로 YAML + 본문 작성 | pending | —    |
| 4   | 다이어그램 삽입 | Mermaid 블록 2개 이상 검수                                  | pending | —    |
| 5   | 발행            | `draft: false`, `hugo --minify` 빌드 성공                   | pending | —    |

## Open Questions

- [ ] 제목 후보 확정 필요: "Kubernetes Gateway API 입문", "Ingress는 이제 그만 — Gateway API 완전 정복", "Gateway API: Ingress를 넘어서" 중 선택
- [ ] Mermaid 다이어그램이 PaperMod 테마에서 렌더링되는지 확인 필요 (`hugo.toml` 설정)
- [ ] 글 길이 목표: 2,000자 이내(압축형) vs 3,500자(상세형) — 독자 체류 시간 목표에 따라 결정

## Risks

| Risk                            | Likelihood | Impact | Mitigation                                         |
| ------------------------------- | ---------- | ------ | -------------------------------------------------- |
| Mermaid가 PaperMod에서 미지원   | Medium     | High   | `hugo.toml`에 Mermaid JS CDN 추가 또는 이미지 대체 |
| Gateway API 버전 정보 빠른 변경 | Low        | Medium | 글 상단에 "v1.5 기준" 명시, 날짜 태그              |
| 글이 너무 길어 독자 이탈        | Medium     | Medium | 섹션별 TL;DR anchor 제공, 목차 자동 생성           |

---

_Status: DRAFT — requirements only. Implementation planning pending via /plan._
_Research source: `/deep-research` session 2026-06-11_
