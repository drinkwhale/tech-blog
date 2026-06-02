---
title: "Claude Code Agent & Skill로 블로그 글 자동 작성하기"
date: 2026-06-02T10:00:56+09:00
draft: false
tags: ["Claude Code", "AI", "자동화", "블로그", "Agent", "Skill"]
categories: ["AI", "자동화"]
description: "Claude Code의 Agent와 Skill 개념을 이해하고, sc:implement 같은 Skill을 활용해 기술 블로그 글을 자동으로 작성하는 방법을 정리한다."
---

## TL;DR

> - Claude Code에서 **Agent**는 특정 역할에 전문화된 독립 AI 인스턴스이고, **Skill**은 그 Agent를 실행하는 슬래시 커맨드다.
> - Agent는 콜드 스타트이므로 CLAUDE.md에 블로그 컨텍스트를 잘 정의해두는 것이 핵심이다.
> - 주제와 삽질 경험만 던지면 STAR 구조 초안을 자동으로 받을 수 있다.

---

## 배경 / 문제 상황

기술 블로그를 운영하다 보면 글을 쓰는 행위 자체보다 **구조를 잡는 데 시간이 더 걸린다**.

- 경험한 문제는 머릿속에 있는데, STAR 형식으로 정리하려면 별도의 작업이 필요하다.
- 반복적인 섹션(TL;DR, 배경, 결과 테이블)마다 같은 고민을 반복한다.
- 결국 "나중에 써야지"가 쌓여서 블로그가 방치된다.

Claude Code의 **Agent**와 **Skill** 시스템을 활용하면 이 반복 작업을 위임할 수 있다.

---

## Agent와 Skill — 개념 정리

### Agent란?

Claude Code에서 Agent는 **특정 역할에 최적화된 독립 AI 인스턴스**다.

메인 Claude(일반 대화)와 다른 점:

| 구분 | 메인 Claude | Agent |
|------|------------|-------|
| 컨텍스트 | 현재 대화 전체를 앎 | **콜드 스타트** — 별도로 브리핑 필요 |
| 역할 | 범용 | 특정 역할에 전문화 |
| 실행 방식 | 현재 세션 | 독립 서브프로세스로 병렬 실행 가능 |
| 컨텍스트 보호 | — | 메인 컨텍스트 창을 오염시키지 않음 |

내장 Agent 타입 예시:

```
backend-architect   — 데이터 무결성·보안 중심 백엔드 설계
frontend-architect  — UX·성능 중심 UI 구현
security-engineer   — 취약점 탐지 및 보안 검토
technical-writer    — 기술 문서 작성
quality-engineer    — 테스트 전략 및 엣지 케이스 탐지
refactoring-expert  — 코드 품질 개선
Explore             — 읽기 전용 코드 탐색 (파일 검색, grep)
Plan                — 구현 계획 수립
```

### Skill이란?

Skill은 **Agent를 실행하는 슬래시 커맨드**다.

`/sc:implement`, `/sc:document` 같은 Skill을 입력하면 Claude Code가 해당 역할에 맞는
전문화 프롬프트와 함께 작업을 시작한다.

자주 쓰는 Skill 목록:

```
/sc:implement   — 기능 구현 (전문 페르소나 활성화)
/sc:document    — 문서 생성
/sc:analyze     — 코드 분석 (품질·보안·성능·아키텍처)
/sc:improve     — 코드 품질 개선
/sc:test        — 테스트 작성 및 실행
/sc:design      — 아키텍처·API 설계
/sc:explain     — 코드·개념 설명
/sc:troubleshoot — 문제 진단 및 해결
/sc:index       — 프로젝트 문서·지식베이스 생성
```

### Skill 파일 위치 — 전역 vs 프로젝트

Skill 파일은 Markdown으로 작성하며, **두 곳** 중 하나에 둔다.

```
~/.claude/commands/          ← 전역: 모든 프로젝트에서 사용 가능
└── sc/
    ├── implement.md         # /sc:implement
    ├── document.md          # /sc:document
    └── ...

<프로젝트 루트>/.claude/commands/   ← 프로젝트 전용
└── write-post.md            # /write-post (이 블로그에서만 사용)
```

| 위치 | 적용 범위 | 용도 |
|------|----------|------|
| `~/.claude/commands/` | 전역 — 모든 프로젝트 | 범용 워크플로우 Skill |
| `.claude/commands/` | 해당 프로젝트만 | 프로젝트 특화 Skill |

**같은 이름이면 프로젝트 Skill이 우선**된다.
전역 Skill을 프로젝트에서 재정의하고 싶을 때 활용할 수 있다.

#### Skill 파일 구조

```markdown
---
name: write-post
description: "Hugo 블로그 STAR 구조 초안 자동 생성"
---

# /write-post — Hugo 블로그 포스트 자동 생성

## 사용법
/write-post
주제: <글 주제>
슬러그: <파일명 kebab-case>
환경: <버전, 플랫폼>
문제: <겪은 문제>
해결: <해결 방법>

## 동작 순서
1. `hugo new content posts/<슬러그>.md` 실행
2. STAR 구조로 내용 작성
3. front matter tags·categories 채우기
```

`name`과 `description`은 Claude가 Skill을 인식하는 데 쓰이고,
본문은 실제로 Agent가 따르는 지시사항이다.

### Agent와 Skill의 관계

```
사용자 입력: /sc:implement "Redis 캐시 레이어 추가"
         ↓
   Skill 로더가 실행됨
         ↓
   전문화된 Agent 프롬프트 적용
   (backend-architect 역할)
         ↓
   파일 읽기 → 코드 작성 → 결과 반환
```

Skill은 Agent를 **어떻게 초기화할지** 정의하는 래퍼(wrapper)라고 이해하면 된다.

### 병렬 실행이 가능하다

독립적인 작업은 Agent를 동시에 여러 개 띄울 수 있다.
예를 들어 두 편의 글 초안을 동시에 작성하거나, 탐색과 구현을 병렬로 처리한다.

```
Agent 1 (technical-writer): Redis 트러블슈팅 글 초안 작성
Agent 2 (technical-writer): Envoy Gateway 마이그레이션 글 초안 작성
→ 동시 실행 → 메인 컨텍스트에서 결과 리뷰
```

---

## 시도한 것들 (삽질 과정)

### 1차 시도 — 그냥 "글 써줘"

처음엔 Claude에게 단순히 "RKE2 Ingress 마이그레이션 글 써줘"라고 했다.

결과는 나쁘지 않았지만 문제가 있었다:
- 블로그 프로젝트의 archetypes 템플릿을 반영하지 못함
- 내가 실제로 겪은 삽질 과정이 아닌 교과서식 설명이 됨
- `hugo new content`로 파일을 만들고 내용을 채우는 Hugo 워크플로우를 모름

### 2차 시도 — Agent 호출 시 컨텍스트 미전달

Agent는 콜드 스타트이기 때문에 현재 대화의 컨텍스트를 모른다.
CLAUDE.md가 프로젝트 루트에 있어도, Agent를 호출할 때 맥락을 브리핑하지 않으면
빈 템플릿에서 교과서식 글을 써버린다.

```markdown
# 잘못된 호출 — Agent가 프로젝트 맥락을 모름
/sc:implement "블로그 글 작성"

# 올바른 호출 — 주제·환경·삽질 포인트를 직접 전달
/sc:implement
이 프로젝트는 Hugo + PaperMod 블로그입니다.
content/posts/ 아래에 새 파일을 만들고
archetypes/default.md의 STAR 구조로 글을 작성해주세요.

주제: Redis 캐시 트러블슈팅
환경: Kubernetes, Redis 7.x
실제 겪은 문제: maxmemory-policy 기본값(noeviction)으로 OOM 발생
```

---

## 해결 과정

### Step 1. CLAUDE.md에 블로그 컨텍스트 명시

Agent가 프로젝트를 처음 볼 때 읽는 파일이 `CLAUDE.md`다.
블로그 글 작성 규칙을 여기에 정의해두면 매번 브리핑을 반복하지 않아도 된다.

```markdown
## Writing Convention

- 새 글: `hugo new content posts/<slug>.md` 로 생성 (archetypes 자동 적용)
- 구조: TL;DR → 배경 → 삽질 과정 → 해결 → 결과 → 회고
- `draft: false` 로 변경해야 빌드에 포함됨
- 실제 경험 기반으로 작성 (교과서식 설명 금지)
- 코드 블록에는 실제 명령어·설정값 사용
- 결과 섹션에는 Before/After 수치 테이블 포함
```

### Step 2. Skill로 글쓰기 위임

Skill을 호출할 때 다음 네 가지 정보를 전달하면 완성도 높은 초안이 나온다:

```
/sc:implement

다음 조건으로 블로그 초안을 작성해주세요.

[주제] Kubernetes Pod OOM 트러블슈팅
[환경] RKE2 v1.29, 워커 노드 8GB RAM
[문제] 특정 시간대에 Pod가 OOMKilled되는 현상 반복
[원인] resource limits 미설정 컨테이너의 메모리 누수
[해결] VPA(Vertical Pod Autoscaler) 도입 + limits 정책 강제화

파일 생성: hugo new content posts/k8s-pod-oom-troubleshooting.md
구조: archetypes/default.md의 STAR 형식 준수
```

### Step 3. 로컬에서 확인 및 보완

Agent가 초안을 작성하면 로컬에서 렌더링을 먼저 확인한다.

```bash
# 드래프트 포함 로컬 서버 실행
hugo server -D

# 브라우저에서 확인
open http://localhost:1313
```

보완할 포인트:
1. 삽질 과정에 실제 에러 메시지·로그 추가
2. 결과 테이블 Before/After 수치 직접 입력
3. 참고 자료 링크 확인
4. `draft: false` 변경

### Step 4. 배포

```bash
git add content/posts/k8s-pod-oom-troubleshooting.md
git commit -m "feat: Kubernetes Pod OOM 트러블슈팅 포스트 추가"
git push
# GitHub Actions가 자동으로 빌드 → GitHub Pages 배포
```

### Step 5. 반복 가능한 워크플로우

```
메모앱에 주제 기록
  └─ 문제, 환경, 핵심 삽질 포인트 3줄
         ↓
/sc:implement 으로 초안 위임 (5~10분)
         ↓
hugo server -D 로 확인
         ↓
실제 경험 수치·에러 보완 (20~30분)
         ↓
git push → 자동 배포
```

---

## 결과

| 항목 | Before (직접 작성) | After (Agent + Skill) |
|------|------------------|-----------------------|
| 초안 작성 시간 | 1~2시간 | 5~10분 |
| 구조 잡는 시간 | 30분 | 0분 (템플릿 자동 적용) |
| 발행 주기 | 월 0~1편 | 월 3~4편 |
| 작성자 집중 영역 | 전체 | 삽질 과정 구체화, 수치 입력 |

---

## 배운 점 / 회고

**Agent는 콜드 스타트다.**
가장 중요한 교훈이다. Agent를 부를 때마다 "새 동료에게 온보딩하는 것"처럼 브리핑해야 한다.
CLAUDE.md가 그 온보딩 문서 역할을 한다. 잘 쓰인 CLAUDE.md 하나가 매번의 브리핑을 대체한다.

**Skill은 역할의 프리셋이다.**
`/sc:implement`와 `/sc:document`는 같은 Claude지만 전혀 다른 관점으로 접근한다.
글 작성엔 `technical-writer` 관점이, 코드 리뷰엔 `quality-engineer` 관점이 더 적합하다.
Skill을 선택하는 것 자체가 품질에 영향을 준다.

**병렬 실행은 컨텍스트 보호다.**
Agent를 백그라운드로 돌리면 메인 대화창이 긴 파일 탐색 결과로 가득 차지 않는다.
탐색 작업이 많을수록, 혹은 독립적인 초안을 여러 개 동시에 뽑을 때 효과적이다.

**AI의 역할은 대필이 아니라 구조화다.**
실제 경험과 수치는 내가 채워야 한다. 그 부분이 기술 블로그의 핵심 가치다.
Agent는 구조를 잡아주고, 인간은 경험을 채운다. 이 분업이 지속 가능한 블로그 운영을 만든다.

---

## 참고 자료

- [Claude Code 공식 문서 — Agent](https://docs.anthropic.com/en/docs/claude-code/sub-agents)
- [Claude Code 공식 문서 — 슬래시 커맨드](https://docs.anthropic.com/en/docs/claude-code/slash-commands)
- [Hugo Content Management](https://gohugo.io/content-management/)
- [Hugo Archetypes](https://gohugo.io/content-management/archetypes/)
