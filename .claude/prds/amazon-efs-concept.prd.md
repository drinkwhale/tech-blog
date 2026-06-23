# Amazon EFS 개념 정리 블로그 글

## Problem

SAA 시험을 준비하는 DevOps 엔지니어가 Amazon EFS·EBS·S3의 차이와 EFS를 써야 하는 시점을 명확하게 정리한 학습 자료가 없어, 공식 문서를 읽더라도 개념이 흩어진 채로 남는다. 이 글은 그 학습 내용을 구조적으로 기록·정리하기 위한 개인 아카이브다.

## Evidence

- Assumption — 학습 목적 개인 기록 (needs validation via SAA 기출 문제 오답률)

## Users

- **Primary**: SAA 시험을 준비 중인 현업 DevOps 엔지니어 (작성자 본인 포함), AWS 스토리지 선택 기준이 혼재되어 있는 상태
- **Not for**: AWS 완전 입문자, 실습 중심 핸즈온 가이드를 찾는 독자

## Hypothesis

We believe **EFS 핵심 개념(What/Why/How) + EFS·EBS·S3 비교표 + SAA 출제 포인트 정리**가 **스토리지 선택 기준을 내면화**하도록 도울 것이다 **SAA 준비 DevOps 엔지니어에게**.
We'll know we're right when **글을 읽은 후 세 서비스의 차이와 EFS 적용 시나리오를 즉시 설명할 수 있다**.

## Success Metrics

| Metric        | Target                                   | How measured            |
| ------------- | ---------------------------------------- | ----------------------- |
| 개념 커버리지 | EFS·EBS·S3 비교 + EFS 사용 시점 포함     | 작성 후 자가 체크리스트 |
| SAA 연관성    | SAA 출제 포인트 최소 3개 이상 반영       | SAA 기출 문제 대조      |
| 가독성        | Mermaid 다이어그램 2개 + 비교표 1개 이상 | 작성 후 육안 검토       |

## Scope

**MVP** — 아래 세 가지만 있으면 충분

1. EFS란 무엇인가 (What) — 정의, 핵심 특징
2. 왜 EFS를 쓰는가 (Why) — EBS·S3와의 차이, 선택 기준
3. 어떻게 동작하는가 (How) — 아키텍처 개요, 마운트 방식, 성능 모드

**Out of scope**

- Terraform / CDK를 이용한 EFS 프로비저닝 실습 — 별도 핸즈온 글로 분리
- 비용 최적화 심화(Intelligent Tiering 세부 설정) — SAA 범위 초과
- EFS Access Point 상세 설정 — 운영 심화 주제로 분리
- 멀티 리전 복제 구성 — 범위 초과

## Delivery Milestones

| #   | Milestone              | Outcome                                                    | Status  | Plan |
| --- | ---------------------- | ---------------------------------------------------------- | ------- | ---- |
| 1   | 글 파일 생성           | `hugo new content posts/amazon-efs-concept.md` 실행 완료   | pending | —    |
| 2   | 초안 작성              | tech-concept-post skill 기반 What/Why/How 구조 초안        | pending | —    |
| 3   | 비교표·다이어그램 삽입 | EFS·EBS·S3 비교표 + Mermaid 2개 포함                       | pending | —    |
| 4   | SAA 포인트 검토        | SAA 출제 관련 내용 확인 및 보완                            | pending | —    |
| 5   | 발행                   | `draft: false` 전환 후 `post/amazon-efs-concept` 브랜치 PR | pending | —    |

## Open Questions

- [ ] SAA-C03 기준 EFS 출제 빈도와 핵심 키워드 확인 필요
- [ ] EFS One Zone vs Standard 차이를 MVP에 포함할지 여부

## Risks

| Risk                              | Likelihood | Impact | Mitigation                             |
| --------------------------------- | ---------- | ------ | -------------------------------------- |
| 공식 문서와 실제 서비스 동작 차이 | 낮음       | 중간   | AWS 공식 문서 링크를 본문에 직접 인용  |
| SAA 출제 범위 변경                | 낮음       | 낮음   | SAA-C03 최신 시험 가이드 기준으로 작성 |

---

_Status: DRAFT — requirements only. Implementation planning pending via /plan._
