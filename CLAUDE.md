# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

Hugo 기반 DevOps/Cloud 기술 블로그. PaperMod 테마 사용. GitHub Actions로 `main` 브랜치 push 시 GitHub Pages에 자동 배포.

- 배포 URL: `https://drinkwhale.github.io/tech-blog/`
- 테마: `themes/PaperMod` (git submodule)

## Commands

```bash
# 로컬 개발 서버 (드래프트 포함)
hugo server -D

# 새 글 생성 (STAR 구조 템플릿 자동 적용)
hugo new content posts/제목.md

# 프로덕션 빌드
hugo --minify
```

## Architecture

```
content/posts/    # 블로그 글 (Markdown)
archetypes/       # 새 글 생성 시 사용할 템플릿
themes/PaperMod/  # 테마 (수정 금지 — submodule)
.github/workflows/deploy.yml  # CI/CD
hugo.toml         # 사이트 전체 설정
```

## Writing Convention

`hugo new content posts/<slug>.md` 로 생성하면 `archetypes/default.md` 기반 STAR 구조 템플릿이 적용됨:

1. **TL;DR** — 3줄 요약
2. **배경 / 문제 상황** — 환경, 규모, 왜 중요한지
3. **시도한 것들** — 실패 과정 포함 (핵심)
4. **해결 과정** — 코드/설정
5. **결과** — Before/After 수치
6. **배운 점 / 회고**

글 완성 후 `draft: false` 로 변경해야 빌드에 포함됨.

## Git Workflow

**소스 수정이 있을 때는 반드시 새 브랜치를 만들고 작업할 것. main에 직접 커밋 금지.**

| 변경 유형 | 브랜치 네이밍 | 예시 |
|---|---|---|
| 블로그 글 작성/수정 | `post/<slug>` | `post/k8s-resource-limit` |
| 기능 추가 / 설정 변경 | `feat/<name>` | `feat/dark-mode` |
| 버그 수정 | `fix/<name>` | `fix/broken-image-link` |

```bash
# 새 브랜치 생성 후 작업
git checkout -b post/my-new-post

# 작업 완료 후 main에 merge
git checkout main
git merge post/my-new-post
```

## Deployment

`main` 브랜치에 push하면 GitHub Actions가 자동으로 빌드 → GitHub Pages 배포.

GitHub 저장소 설정에서 Pages source를 **GitHub Actions**로 설정해야 함 (Settings → Pages → Source).
