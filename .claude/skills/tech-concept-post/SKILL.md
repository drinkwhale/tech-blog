---
name: tech-concept-post
description: 기술 개념 소개 블로그 글을 "무엇인가 → 왜 필요한가 → 어떻게 동작하는가" 3단 구조로 작성한다. Mermaid 다이어그램 2개 이상과 비교 표를 포함한다. 새로운 기술·도구·표준을 처음 설명하는 글에 사용한다.
---

# Tech Concept Post

개념 설명 중심의 기술 블로그 글 구조. 독자가 "무엇인지 → 왜 쓰는지 → 어떻게 쓰는지"를 순서대로 이해하도록 설계한다.

## When to Activate

- 새로운 기술·표준·도구를 처음 소개하는 글 (예: Gateway API, eBPF, ArgoCD)
- 개념 비교 글 (예: Ingress vs Gateway API, Helm vs Kustomize)
- 입문~중급 독자를 대상으로 한 개념 정리

**이 스킬을 쓰지 않는 경우:**

- 트러블슈팅·사고기록 → 기존 STAR 템플릿 (`archetypes/default.md`) 사용
- 핸즈온 튜토리얼 (설치부터 실습까지) → 별도 tutorial 구조 권장

## Post Structure (필수)

```markdown
---
title: "{블로그 친화적 제목 — 조사보고서·정리·분석 등 문서 유형명 금지}"
date: { date }
draft: true
tags: [{ 기술명 }, { 카테고리 }]
categories: [{ DevOps | Kubernetes | Networking | ... }]
---

## TL;DR

> (3줄 이내 — 독자가 이 글에서 얻는 핵심 1가지)

---

## 무엇인가 (What)

<!-- 정의 + 기존 것과의 차이 비교 표 -->

### 핵심 개념 요약

| 개념 | 설명 |
| ---- | ---- |
| ...  | ...  |

---

## 왜 필요한가 (Why)

<!-- 기존 방식의 한계 → 이 기술이 해결하는 문제 -->

### [기존 방식]의 한계

<!-- Mermaid 다이어그램 1: 기존 아키텍처 또는 문제 상황 -->

\`\`\`mermaid
graph LR
...
\`\`\`

---

## 어떻게 동작하는가 (How)

<!-- 동작 원리 + 흐름 다이어그램 + 코드/YAML 예시 -->

<!-- Mermaid 다이어그램 2: 트래픽/데이터 흐름 -->

\`\`\`mermaid
flowchart TD
...
\`\`\`

### 예시 코드

\`\`\`yaml

# 핵심 설정 예시

\`\`\`

---

## 선택 가이드 (선택 섹션)

<!-- 구현체·버전·옵션이 여러 개일 때만 포함 -->

| 옵션 | 특징 | 추천 상황 |
| ---- | ---- | --------- |

---

## 참고 자료

- [제목](URL)
```

## Rules

### 제목 규칙

- ❌ 금지: "~조사보고서", "~정리", "~분석", "~개요", "~소개"
- ✅ 권장: 질문형 ("Gateway API, Ingress를 대체할 수 있을까?"), 선언형 ("Ingress는 이제 그만 — Gateway API로 넘어갈 때다"), 가치 제시형 ("Gateway API: 트래픽 라우팅의 새 표준")

### Mermaid 다이어그램 규칙

- **최소 2개** 필수: ① 구조/아키텍처 ② 흐름/시퀀스
- 다이어그램 전에 1줄 설명 작성 (예: "아래는 Gateway API의 3계층 구조입니다.")
- 노드 레이블은 한국어 또는 실제 리소스명 사용
- 지나치게 복잡한 다이어그램은 2개로 분리

### Mermaid 렌더링 확인

`layouts/partials/extend_head.html`이 렌더링된 본문에 `class="mermaid"`가 있는지 자동 감지해서
mermaid.js를 로드한다 (`layouts/_markup/render-codeblock-mermaid.html` render hook이 ` ```mermaid `
코드펜스를 `<pre class="mermaid">`로 출력). 별도 front matter나 `hugo.toml` 설정 없이,
글에 ` ```mermaid ` 코드블록만 있으면 자동으로 렌더링된다. `hugo server -D`로 브라우저에서
실제 다이어그램이 그려지는지 최종 확인만 하면 된다.

### 길이 가이드

- 입문~~중급 대상: 2,000~~3,500자 (한국어 기준)
- 섹션당 핵심 내용만, 코드는 최소 단위 예시로 제한
- 각 섹션 끝에 독자가 기억할 1가지 포인트 명시 가능

### 금지 패턴 (article-writing 스킬에서 상속)

- "최근 급변하는 환경에서"
- "혁신적인", "강력한", "게임체인저"
- 출처 없는 수치·통계
- 섹션을 끝내는 "~에 대해 알아보겠습니다"

## Writing Process

1. PRD에서 섹션 구조 확인 (`.claude/prds/`)
2. `hugo new content posts/<slug>.md` 로 파일 생성
3. **무엇인가** 섹션: 정의 + 비교 표 작성
4. **왜 필요한가** 섹션: 문제 3가지 나열 → Mermaid 다이어그램 1
5. **어떻게 동작하는가** 섹션: 흐름 다이어그램 2 → YAML/코드 예시
6. 선택 가이드 표 작성 (구현체/옵션이 복수일 때)
7. 참고 자료 링크 6개 이상
8. `draft: true` 상태로 저장, `hugo server -D` 로 미리보기 확인
9. 확인 후 `draft: false` 로 변경

## Quality Gate

발행 전 체크:

- [ ] 제목에 "조사보고서" 등 문서 유형명 없음
- [ ] TL;DR 3줄 이내
- [ ] Mermaid 다이어그램 2개 이상
- [ ] 비교 표 1개 이상
- [ ] 참고 자료 링크 6개 이상
- [ ] `hugo server -D` 빌드 오류 없음
- [ ] Mermaid 다이어그램 브라우저에서 정상 렌더링 확인
