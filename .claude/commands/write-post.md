---
name: write-post
description: "Hugo 블로그 글을 STAR 구조로 자동 작성한다. 주제·환경·문제·해결 정보를 받아 archetypes/default.md 기반 초안을 생성한다."
---

# /write-post — Hugo 블로그 포스트 자동 생성

## 사용법

```
/write-post
주제: <글 주제>
슬러그: <파일명 (영어 kebab-case)>
환경: <버전, 플랫폼 등>
문제: <겪은 문제>
원인: <원인>
해결: <해결 방법>
```

## 동작 순서

0. (선택) **사실 검증이 필요하면** `deep-research` skill 실행 — 버전별 동작, 공식 문서 링크, 벤치마크 수치를 확인해야 할 때만. 이미 직접 겪은 내용이면 생략
1. `hugo new content posts/<슬러그>.md` 실행하여 파일 생성 (archetypes 템플릿 자동 적용)
2. `article-writing` skill을 적용해 아래 STAR 구조로 본문 작성:
   - **TL;DR**: 3줄 이내 핵심 요약 (배울 것 / 환경 / 결론)
   - **배경 / 문제 상황**: 환경 스펙 테이블 포함, 왜 이 문제가 중요한지
   - **시도한 것들 (삽질 과정)**: 1차·2차·3차 시도, 각 실패 이유
   - **해결 과정**: Step별 코드·설정 블록 포함
   - **결과**: Before/After 수치 비교 테이블
   - **배운 점 / 회고**: 다음에 같은 상황이면 어떻게 할지
   - **참고 자료**: 관련 공식 문서 링크
3. `@seo-specialist` agent 호출 → front matter의 `title`·`description`·`tags`·`categories` 최적화 및 H2/H3 구조 검토
4. `draft` 값 확인 — archetype이 `draft: false`로 생성하므로, 검토 전 공개를 막으려면 이 단계에서 `true`로 바꾼 뒤 검토 후 되돌린다
5. (선택) 글 구조·표현에서 재사용할 패턴이 나왔다면 `/learn`으로 캡처

## 규칙

- 교과서식 설명 금지 — 실제 삽질 과정과 에러 메시지 중심으로 작성 (`article-writing`의 banned patterns 준수)
- 코드 블록에는 실제 명령어·설정값 사용 (플레이스홀더 최소화)
- 환경 스펙은 테이블 형식으로 버전 명시
- 삽질 과정은 최소 2개 이상 작성 (1개면 독자 신뢰도 낮음)
- Before/After 수치가 없으면 정성적 비교라도 테이블로 표현

## 예시 호출

```
/write-post
주제: Kubernetes Pod OOM 트러블슈팅
슬러그: k8s-pod-oom-troubleshooting
환경: RKE2 v1.29, 워커 노드 8GB RAM
문제: 특정 시간대 Pod OOMKilled 반복
원인: resource limits 미설정 컨테이너 메모리 누수
해결: VPA 도입 + limits 정책 강제화
```

## 다른 글 유형이면 이 커맨드를 쓰지 말 것

| 글 유형 | 사용할 것 |
| --- | --- |
| 기술 개념 소개 (What/Why/How + Mermaid) | `tech-concept-post` skill |
| 제품·기능 운영 지식 정리 (이모지 섹션·설정값 나열) | `creed-blog-style` skill |
| 장문 에세이·voice 중심 | `article-writing` skill 단독 |
