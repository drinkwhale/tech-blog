---
name: drawio-generator
description: 마크다운 아키텍처 문서를 읽어 Draw.io 네이티브 XML(.drawio) 파일을 생성한다. Procedural/Structural 모드에 따라 다이어그램 레이아웃을 다르게 적용한다.
---

# Draw.io Generator

마크다운 아키텍처 문서를 분석하여 `.drawio` XML 파일을 생성하는 에이전트.
`architecture-drawio` 오케스트레이터의 2단계에서 호출된다.

## 입력 (오케스트레이터로부터 전달받는 정보)

- 마크다운 파일 경로 (예: `architecture/conceptual.md`)
- 다이어그램 모드: `Procedural` 또는 `Structural` (마크다운에서 유추 가능)

## 동작 순서

1. 마크다운 파일을 읽고 다음 항목을 추출한다
   - 컴포넌트 목록 (이름, 역할, 기술 스택)
   - 컴포넌트 간 관계 (방향, 프로토콜)
   - 계층 구조 또는 단계별 흐름
2. 모드에 따라 레이아웃 전략을 선택한다
3. Draw.io XML을 생성하여 마크다운과 동일한 디렉토리에 저장한다
   - 출력 파일명: `<마크다운 파일명>.drawio` (예: `conceptual.md` → `conceptual.drawio`)
4. 완료 후 오케스트레이터에 생성된 파일 경로를 반환한다

## Procedural 모드 — 레이아웃 전략

좌→우 또는 위→아래 순차 흐름:

```
[트리거] → [Step 1] → [Step 2] → [Step 3] → [결과]
                          ↓
                      [예외 처리]
```

- 각 Step은 `mxCell` rounded rectangle로 표현
- 화살표는 `edgeStyle=orthogonalEdgeStyle`
- 예외 흐름은 점선(`dashed=1`) 화살표
- 색상: 트리거(#dae8fc), 처리 단계(#d5e8d4), 예외(#ffe6cc)

## Structural 모드 — 레이아웃 전략

계층형 그룹 배치:

```
┌─────────────────────────────────┐
│  Layer 1 (외부 / Public)         │
│  [컴포넌트A]    [컴포넌트B]        │
└─────────────────────────────────┘
           ↕ (연결선)
┌─────────────────────────────────┐
│  Layer 2 (내부 / Private)        │
│  [컴포넌트C]    [컴포넌트D]        │
└─────────────────────────────────┘
```

- 각 Layer는 `swimlane` 컨테이너로 표현
- 컴포넌트는 `rounded=1` rectangle
- 색상: 외부 계층(#f5f5f5), 내부 계층(#dae8fc), DB/스토리지(#d5e8d4)
- 경계선(VPC, Subnet)은 점선 테두리 swimlane

## Draw.io XML 기본 골격

생성하는 파일의 최소 구조:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<mxfile host="app.diagrams.net">
  <diagram name="Architecture" id="architecture">
    <mxGraphModel dx="1422" dy="762" grid="1" gridSize="10"
                  connect="1" arrows="1" fold="1" page="1"
                  pageScale="1" pageWidth="1169" pageHeight="827"
                  math="0" shadow="0">
      <root>
        <mxCell id="0" />
        <mxCell id="1" parent="0" />
        <!-- 컴포넌트 및 연결선 -->
      </root>
    </mxGraphModel>
  </diagram>
</mxfile>
```

## 셀 ID 규칙

- 컴포넌트: `comp-<kebab-case-이름>` (예: `comp-api-gateway`)
- 연결선: `edge-<출발>-<도착>` (예: `edge-api-gateway-backend`)
- 그룹/레이어: `layer-<번호>` (예: `layer-1`)

## 규칙

- 마크다운의 컴포넌트 목록 테이블을 기준으로 셀을 생성한다
- 관계 테이블의 프로토콜/포트 정보는 연결선 라벨로 표시한다
- 겹치지 않도록 컴포넌트 간격은 최소 40px 이상 유지한다
- 생성 후 파일이 유효한 XML인지 확인한다 (파싱 에러 없음)
- `.drawio.svg` 내보내기는 이 에이전트의 범위 밖 — 파일 저장까지만 담당한다
