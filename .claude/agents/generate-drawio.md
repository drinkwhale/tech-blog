---
name: generate-drawio
description: 자연어 및 마크다운 기반의 아키텍처 설명을 읽고, next-ai-drawio 기반 MCP 서버 도구를 활용하여 Draw.io 파일(XML)을 자동 생성 및 갱신합니다.
---

# Draw.io 다이어그램 생성 (generate-drawio)

이 스킬은 마크다운 문서의 텍스트 혹은 아키텍처 설계를 기반으로, `next-ai-drawio` 기반 MCP 서버 도구를 활용해 네이티브 `.drawio` 다이어그램을 렌더링하고 로컬에 저장하는 데 집중합니다.

## 🎯 실행 지침 (Instructions)

### 1. 설계 내용 파악 및 사전(RAG) 참조

- 지정된 대상 마크다운 파일(예: `architecture/conceptual.md`) 또는 사용자의 요청 텍스트를 읽어냅니다.
- 시스템 구성요소(예: 클러스터, 파드, 로드밸런서 등)와 이들 간의 흐름(네트워크, 관리 흐름)을 파악합니다.
- **[중요]** 다이어그램을 생성하기 전, 반드시 다음의 분리된 RAG 사전 파일들을 참조하여 정확히 일치하는 `style` 값을 맵핑해야 합니다:
  - `references/alibaba_cloud.json`: 알리클라우드 자원 (SLB, RDS, ACR 등)
  - `references/aws4.json`: AWS 자원 및 범용 아이콘 (User, Client 등)
  - `references/kubernetes.json`: 쿠버네티스 자원 (Pod, Node, Deploy 등)
  - `references/third_party.json`: 외부 도구 (Github, Vault, Rancher, ArgoCD 등 퍼블릭 SVG 링크 기반)

### 2. Draw.io MCP 서버 호출

- IDE에 등록된 `drawio` MCP 서버(기반: `@next-ai-drawio/mcp-server`)의 툴셋을 사용하여 다이어그램을 생성합니다.
- 자연어 명령어 형태의 생성 툴(예: `create_diagram`, `generate`, 또는 그에 상응하는 툴)이 있다면 다음 정보를 인자로 넘겨 호출합니다:
  - 아키텍처 구성 요소 (알리클라우드, AWS, K8s 등 특정 테마가 요구된 경우 해당 테마 명시)
  - 노드(Node) 간의 연결 관계와 그룹(Sub-graph/Swimlane) 구성

### 3. 로컬 파일 시스템 저장 (SVG Export 렌더링)

- 다이어그램 구성이 완료되면 반드시 `drawio` MCP 서버의 **`export_diagram`** 툴을 호출하여 깃허브 및 마크다운 미리보기 화면에서 그래픽이 렌더링되면서 동시에 편집도 가능한 **`.drawio.svg`** 확장자로 최종 산출물을 생성합니다.
  - `format`: `"svg"`
  - `path`: `architecture/원하는이름.drawio.svg`

## ⚠️ 제약사항 및 주의점 (Styling & Layout)

- Mermaid 렌더링 방식에서 탈피하여, 가능한 Draw.io의 네이티브 포맷을 직접 다루거나 MCP의 고수준 추상화 툴을 이용해 생성합니다.
- **Conceptual 아키텍처 스타일링**:
  - 딱딱한 다이어그램 대신 개념적인 느낌을 주려면 컨테이너나 연결선에 `sketch=1;` 속성을 부여하여 스케치 효과를 적용하세요.
  - 그룹을 묶는 큰 컨테이너 박스에는 접히는 기능이 있는 `swimlane` 대신, 일반적인 네모 박스 형태인 `shape=rectangle;whiteSpace=wrap;html=1;align=left;verticalAlign=top;` 속성을 사용해야 합니다.
- 타겟 경로에 파일을 저장할 때는 절대 경로를 사용해야 합니다.
