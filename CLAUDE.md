# CLAUDE.md

## Project
Hugo 기반 DevOps/Cloud 기술 블로그. PaperMod 테마. `main` push → GitHub Pages 자동 배포.
배포 URL: `https://drinkwhale.github.io/tech-blog/` | 테마: `themes/PaperMod` (submodule, 수정 금지)

## Commands
```bash
hugo server -D                     # 로컬 개발 서버 (드래프트 포함)
hugo new content posts/<slug>.md   # 새 글 생성 (STAR 템플릿 자동 적용)
hugo --minify                      # 프로덕션 빌드
```

## Architecture
```
content/posts/               # 블로그 글 (Markdown)
archetypes/                  # 글 템플릿
.github/workflows/deploy.yml # CI/CD
hugo.toml                    # 사이트 설정
.claude/                     # 프로젝트 전용 skills·agents·commands
```

## Writing Convention
STAR 구조: **TL;DR → 배경/문제 → 시도한 것들 → 해결 과정 → 결과(수치) → 배운 점**
완성 후 `draft: false` 로 변경해야 빌드에 포함됨.

## Blog Writing Workflow
1. **리서치** `/deep-research` — 다출처 조사 및 인용 보고서
2. **기획** `/plan-prd` — 글 목적·구조 설계 문서 생성
3. **파일 생성** `hugo new content posts/<slug>.md`
4. **작성** `/write-post` (STAR 초안) 또는 `/article-writing` (장문·voice 중심)
5. **구조 설계 글** `@architect` agent — 아키텍처 리뷰·설계 보조
6. **SEO** `@seo-specialist` — 메타·태그·구조 최적화
7. **패턴 저장** `/learn` — 효과적인 글 구조 즉시 캡처
8. **세션 유지** `/save-session` → 다음 세션 `/resume-session`

## Git Workflow
**main 직접 커밋 금지.** 브랜치 네이밍:
- 글 작성/수정: `post/<slug>` (예: `post/k8s-resource-limit`)
- 기능/설정 변경: `feat/<name>` | 버그 수정: `fix/<name>`

## Deployment
`main` push → GitHub Actions 자동 빌드·배포.
Pages source 설정: Settings → Pages → Source → **GitHub Actions**
