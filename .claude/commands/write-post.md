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

1. `hugo new content posts/<슬러그>.md` 실행하여 파일 생성 (archetypes 템플릿 자동 적용)
2. 생성된 파일에 아래 STAR 구조로 내용 작성:
   - **TL;DR**: 3줄 이내 핵심 요약 (배울 것 / 환경 / 결론)
   - **배경 / 문제 상황**: 환경 스펙 테이블 포함, 왜 이 문제가 중요한지
   - **시도한 것들 (삽질 과정)**: 1차·2차·3차 시도, 각 실패 이유
   - **해결 과정**: Step별 코드·설정 블록 포함
   - **결과**: Before/After 수치 비교 테이블
   - **배운 점 / 회고**: 다음에 같은 상황이면 어떻게 할지
   - **참고 자료**: 관련 공식 문서 링크
3. front matter에 `tags`, `categories`, `description` 채우기
4. `draft: true` 유지 (사용자가 검토 후 `false`로 변경)

## 규칙

- 교과서식 설명 금지 — 실제 삽질 과정과 에러 메시지 중심으로 작성
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
