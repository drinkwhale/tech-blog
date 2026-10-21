---
title: "Claude 생태계의 plugin/agent/skill 자동 선택 도구 조사"
date: 2026-09-30
draft: false
tags: ["Claude Code", "AI", "Agent", "Skill", "MCP"]
categories: ["AI"]
---

| 항목 | 내용 |
|------|------|
| 범위 | 외부 기술 트렌드 조사 |
| 대상 | Claude Code / Cowork 환경에서 plugin·agent·skill을 상황에 맞게 자동으로 고르거나 라우팅해주는 도구 |
| 결론 요약 | "이거다" 싶은 표준 솔루션은 아직 없음. Anthropic 자체 빌트인 매칭이 기본값이고, 부족한 부분을 서드파티 MCP 라우터·프레임워크가 메꾸는 중 |
| 조사 시점 | 2026-09 |
| 신뢰도 | 공식 문서는 확인됨, 서드파티 도구는 대부분 사용량·검증 데이터 부족 (표시 필요) |

**TL;DR**: 자동 선택은 지금 세 층위로 나뉩니다. ① Anthropic 공식 — skill의 `description` 필드 의미 매칭 + `pluginSuggestionMarketplaces`로 디렉토리 기반 플러그인 추천. ② 커뮤니티 MCP 라우터 — `skill-router`, Meta/Intelligent Skill Router 같은 지연 로딩·랭킹 도구. ③ 프레임워크형 — SuperClaude Framework처럼 페르소나·모드·명령어를 통째로 오케스트레이션. 크로스벤더 표준인 Agent Plugins는 아직 Claude Code 미지원이라 "확인 필요" 상태입니다.

---

## 📋 왜 이 조사가 필요한가

Claude Code에 설치된 skill·agent·plugin이 늘어날수록, 어떤 요청에 어떤 것을 쓸지 Claude가 스스로 판단해야 하는 부담이 커집니다. 공식 문서는 이 판단이 각 skill의 `description` 필드와 사용자 메시지 사이의 의미적 매칭으로 이뤄진다고 설명합니다. 문제는 이 매칭이 매 턴 전체 목록을 컨텍스트에 올려야 동작하고, description이 애매하면 선택이 흔들린다는 점입니다. 실제로 커뮤니티에서는 "설명대로 자동 활성화 안 된다"는 보고가 나왔고([Scott Spence](https://scottspence.com/posts/claude-code-skills-dont-auto-activate)), 이게 서드파티 라우터 도구들이 등장한 배경입니다.

## 📋 문제: 선택지가 많아지면 정확도가 떨어진다

공식 문서([code.claude.com/docs/en/skills](https://code.claude.com/docs/en/skills))에 따르면 skill 목록은 사용 여부와 무관하게 매 턴 컨텍스트를 소비합니다. description이 "Utility skill"처럼 모호하면 자동 활성화가 안 되고, 반대로 skill이 너무 많아지면 Claude가 올바른 것을 고르기 어려워집니다. Scott Spence는 직접 실험에서 자동 활성화 성공률이 "글로벌 스코프 4/10, 로컬 스코프 5/10" 수준이었다고 보고했습니다. 이 문제는 plugin 단위에서도 반복됩니다 — plugin은 command·agent·skill·hook·MCP 서버를 한 번에 번들링하는데, 설치는 됐어도 상황에 맞게 켜고 끄는 건 별개 문제입니다.

## 📋 해결책 ① Anthropic 공식 빌트인 메커니즘

공식 문서 기준으로 현재 Claude Code/Cowork에 내장된 자동 선택 메커니즘은 다음과 같습니다.

> Claude는 skill의 `description` 필드를 사용자 메시지와 의미적으로 매칭해서 자동으로 로드합니다. `disable-model-invocation: true`를 걸면 수동 호출(`/skill-name`)만 가능해지고, `user-invocable: false`를 걸면 반대로 Claude만 자동 호출할 수 있습니다.

- **Skill 자동 매칭**: description + `when_to_use` 필드 합산 1,536자 제한 안에서 의미 매칭. 부작용이 있는 작업(배포 등)은 `disable-model-invocation: true`로 자동 호출을 막는 게 권장됩니다.
- **Plugin 디렉토리 기반 추천**: 관리자가 [`pluginSuggestionMarketplaces`](https://code.claude.com/docs/en/settings-reference#pluginsuggestionmarketplaces) 설정을 허용 목록에 넣으면, `/plugin` Discover 탭에서 현재 작업 디렉토리와 관련된 플러그인이 "suggested for this directory" 라벨을 달고 상단에 고정됩니다.
- **Code intelligence 자동 제안**: 이미 특정 언어 서버 바이너리가 설치돼 있으면, 프로젝트를 열 때 Claude가 대응하는 LSP 플러그인 설치를 먼저 제안합니다.
- **미사용 skill/plugin 가시화**: `/skill-doctor`, `/plugin` Stats 탭이 각 skill의 컨텍스트 비용과 호출 빈도를 보여주고, 2주/10세션 이상 안 쓴 plugin은 "Not used recently"로 따로 표시됩니다. 자동 선택 자체보다는 "무엇을 꺼야 하는지" 판단을 돕는 도구입니다.
- **Sub-agent 위임**: 서브에이전트도 skill과 마찬가지로 설명 기반 매칭으로 위임 대상이 정해집니다(공식 서브에이전트 문서, [claude.com/blog/subagents-in-claude-code](https://claude.com/blog/subagents-in-claude-code)).

즉 "자동 선택"의 기본 골격은 이미 Claude Code 안에 있지만, 선택의 품질은 description을 얼마나 구체적으로 쓰느냐에 크게 의존합니다.

## 📋 해결책 ② 커뮤니티 MCP 라우터

빌트인 매칭의 부정확성을 보완하려는 서드파티 MCP 서버들이 나와 있습니다. 검증된 사용량 데이터가 없어 품질은 "확인 필요" 상태입니다.

- **skill-router** ([github.com/sorcerai/skill-router](https://github.com/sorcerai/skill-router)): Node.js 단일 파일 MCP 서버. `~/.claude/skills`, `~/.claude/plugins`, `~/.claude/skill-vault` 세 디렉토리를 스캔해서 `skill_search`(토큰 겹침 점수 기반 검색), `skill_load`(선택된 skill만 전체 로드), `skill_reindex` 세 도구를 제공합니다. 매번 전체 skill을 컨텍스트에 올리는 대신 필요한 것만 지연 로딩하는 방식이라, 위에서 지적한 "매 턴 컨텍스트 소비" 문제를 정면으로 겨냥합니다. 의존성이 `@modelcontextprotocol/sdk` 하나뿐이고 약 185개 skill 인덱싱에 25ms 정도 걸린다고 소개돼 있습니다. 심볼릭 링크를 따라가므로 신뢰 안 되는 경로 연결은 피해야 합니다.
- **Meta Skill Router / Intelligent Skill Router / Model Router** (mcpmarket.com 카탈로그): 사용자 의도를 분석해 여러 skill에 우선순위 점수를 매기고, 의존성까지 고려해 실행 계획을 짜준다고 소개된 skill들입니다. "적응형 학습"으로 라우팅이 개선된다는 설명이 있지만, 이 카탈로그 자체가 제3자 마켓플레이스라 실제 채택 사례나 코드 품질은 별도 검증이 필요합니다 (확인 필요).

## 📋 해결책 ③ 프레임워크형 오케스트레이션

개별 라우터보다 큰 단위로, Claude Code 위에 통째로 올라가는 메타 프레임워크도 있습니다.

- **SuperClaude Framework** ([superclaude.org](https://superclaude.org/), [SuperClaude-Org/SuperClaude_Framework](https://github.com/SuperClaude-Org/SuperClaude_Framework)): 30개 슬래시 명령어, 20개 전문 에이전트(페르소나), 7가지 행동 모드(Orchestration, Deep Research, Token-Efficiency 등), Tavily·Context7·Sequential-Thinking·Serena·Playwright 등 8개 MCP 서버를 묶은 구성 프레임워크입니다. `/recommend`, `/select-tool` 같은 명령어로 상황에 맞는 명령/도구를 추천받을 수 있고, "Orchestration" 모드가 여러 MCP 서버·에이전트 조합을 자동 조율한다고 설명합니다. 다만 공식 문서가 자동 조율의 구체적 알고리즘까지는 공개하지 않아, 내부 동작은 블랙박스에 가깝습니다.

## ⚠️ 표준 관점에서 아직 없는 것: Agent Plugins 크로스벤더 스펙

Amazon, Microsoft, OpenAI, Cursor, Vercel이 주도하는 **Agent Plugins**라는 별도 표준이 있습니다. `SKILL.md`와 MCP 서버를 한 폴더로 패키징해서 여러 에이전트 런타임에서 공유하려는 시도인데, 정작 Claude Code의 자체 plugin 시스템과는 무관하고, 2026-09 시점 공식 지원 런타임은 Codex CLI뿐입니다. Claude Code/Cowork에서 이 표준을 통한 자동 선택은 **확인 필요** — 아직 지원되지 않는 것으로 보입니다.

## 📊 비교표

| 도구/메커니즘 | 성격 | 선택 방식 | 검증 상태 |
|---|---|---|---|
| Skill description 매칭 (공식) | Claude Code 내장 | 의미 매칭 + `when_to_use` | 공식 문서로 확인됨 |
| `pluginSuggestionMarketplaces` (공식) | Claude Code 내장 | 작업 디렉토리 기반 추천 | 공식 문서로 확인됨, 관리자 설정 필요 |
| skill-router (sorcerai) | 커뮤니티 MCP 서버 | 토큰 겹침 검색 + 지연 로딩 | GitHub 공개, 채택 규모 확인 필요 |
| Meta/Intelligent/Model Router | 커뮤니티 MCP 서버 | 의도 분석 + 우선순위 점수 | 마켓플레이스 소개 문구뿐, 검증 필요 |
| SuperClaude Framework | 메타 프레임워크 | 명령어/모드 오케스트레이션 | GitHub 다수 포크 존재, 내부 알고리즘 비공개 |
| Agent Plugins 표준 | 크로스벤더 스펙 | (스펙 차원) | Claude Code 미지원, 확인 필요 |

## ✅ 도입 전 확인사항

- [ ] 회사 보안 정책상 서드파티 MCP 서버(예: skill-router) 설치가 허용되는지
- [ ] `pluginSuggestionMarketplaces`를 관리자 설정으로 켤 수 있는 권한이 있는지
- [ ] skill/plugin description을 명확하게 다시 써서 자동 매칭 정확도부터 올릴 여지가 있는지 (도구 도입보다 우선순위가 높을 수 있음)
- [ ] SuperClaude 같은 프레임워크를 쓸 경우 20여 개 에이전트·8개 MCP 서버가 추가하는 컨텍스트 비용을 감당할 수 있는지

## 다음 학습 로드맵

| 레벨 | 주제 |
|------|------|
| 기초 | Claude Code skill의 `description`/`when_to_use` 작성법, `/skill-doctor` 활용 |
| 중급 | `pluginSuggestionMarketplaces` 관리자 설정, plugin 마켓플레이스 운영 |
| 고급 | MCP 기반 커스텀 라우터 직접 구축 (skill-router 구조 참고), Agent Plugins 표준 동향 추적 |

---

## 출처

- [Extend Claude with skills - Claude Code Docs](https://code.claude.com/docs/en/skills)
- [Claude Code Skills Don't Auto-Activate (a workaround) - Scott Spence](https://scottspence.com/posts/claude-code-skills-dont-auto-activate)
- [Discover and install prebuilt plugins through marketplaces - Claude Code Docs](https://code.claude.com/docs/en/discover-plugins)
- [How and when to use subagents in Claude Code | Claude by Anthropic](https://claude.com/blog/subagents-in-claude-code)
- [GitHub - sorcerai/skill-router](https://github.com/sorcerai/skill-router)
- [Meta Skill Router - Claude Code Skill Orchestrator](https://mcpmarket.com/tools/skills/meta-skill-router)
- [Intelligent Skill Router - Claude Code Skill](https://mcpmarket.com/tools/skills/intelligent-skill-router)
- [Model Router - Claude Code Skill for AI Cost Optimization](https://mcpmarket.com/tools/skills/model-router)
- [SuperClaude Framework](https://superclaude.org/)
- [GitHub - SuperClaude-Org/SuperClaude_Framework](https://github.com/SuperClaude-Org/SuperClaude_Framework)
- [Agent Plugins Explained: New Skills + MCP Standard | Get Claude Skills](https://www.getclaudeskills.com/blog/agent-plugins-explained)
