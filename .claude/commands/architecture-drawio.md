---
name: architecture-drawio
description: PRD 또는 아키텍처 초안을 바탕으로 마크다운 문서 생성 및 Draw.io 파일(XML) 생성 파이프라인을 조율하는 오케스트레이터 스킬
---

# /architecture-drawio — Draw.io 아키텍처 오케스트레이터

## 사용법

```
/architecture-drawio
모드: <Procedural | Structural>
대상 파일: <architecture/파일명.md>
PRD 또는 초안: <설계 내용 또는 파일 경로>
```

## 실행 파이프라인

사용자로부터 아키텍처 설계 지시가 들어오면 다음 순서로 서브 에이전트를 호출한다.

### 1단계 — 마크다운 문서 생성

`markdown-architect` 에이전트를 spawn하여 수행:

- **입력**: 사용자 PRD/초안, 다이어그램 모드(Procedural/Structural), 대상 파일 경로
- **출력**: 아키텍처 마크다운 문서 작성 완료 (`architecture/<파일명>.md`)
- **완료 조건**: 문서 하단에 `![아키텍처](파일명.drawio.svg)` 이미지 삽입 문법 포함

1단계가 완전히 완료된 것을 확인한 후 2단계로 진행한다.

### 2단계 — Draw.io 파일 생성

`drawio-generator` 에이전트를 spawn하여 수행:

- **입력**: 1단계에서 작성된 마크다운 파일 경로
- **출력**: 동일 디렉토리에 `.drawio` 네이티브 XML 파일 생성
- **완료 조건**: `.drawio` 파일이 로컬에 존재하고 마크다운의 이미지 경로와 일치

## 모드 분기 기준

| 모드 | 사용 시나리오 | 특징 |
|------|-------------|------|
| **Procedural** | CI/CD 파이프라인, 데이터 흐름도 | 순서·단계 중심, 화살표 흐름 |
| **Structural** | Landing Zone, Network Topology, Multi-Cluster | 컴포넌트 배치·계층 중심 |

## 에이전트 호출 규칙

- 두 에이전트는 **순차 실행** (2단계는 1단계 산출물에 의존)
- 각 에이전트 완료 후 결과 파일 존재 여부를 확인하고 다음 단계 진행
- 에러 발생 시 해당 단계 에이전트를 재호출하며, 오케스트레이터가 직접 수정하지 않음
