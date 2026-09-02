---
name: design-to-code
description: Convert an already-built Claude Design screen into real React source code inside the target project — not a copy of the Claude Design markup, but a fresh implementation that reuses the project's existing components/patterns and follows convention.md. Works whether or not the project has its own design system. Use when the user hands over a Claude Design screen link and wants it turned into working source, or asks to "소스화" a Claude Design screen. Requires design-env-setup and a project convention.md.
metadata:
  author: 김대섭
  version: "1.1.0"
  status: "local-only draft — not yet validated end-to-end"
---

# design-to-code

가장 중요한 것은 **"Design을 보고 코드를 생성하는 AI"가 아니라 "Design과 Repository 사이의 차이를 분석하고, 기존 프로젝트 규칙 안에서 구현 방법을 결정하는 AI"**를 만드는 것이다. 코드 생성(Step 8)보다 그 앞 단계들(이해 → 요구사항 추출 → 레이아웃 계획 → 컴포넌트 매핑 → 갭 분석 → 구현 계획)의 정확성이 우선한다. 순서를 건너뛰지 않는다.

## Architecture principle — 책임 분리

```
Claude Design  = 무엇을 만들 것인가
StyleGallery   = 어떤 Layout 구조로 만들 것인가
디자인 시스템    = 어떤 UI Component를 쓸 것인가 (있는 경우에만 — 없으면 이 층은 그냥 없는 것)
design-to-code = 기존 프로젝트에서 어떻게 구현할 것인가
Claude Code    = 실제 Source Code 생성 및 검증
```

StyleGallery가 컴포넌트를 고르거나 디자인 시스템이 레이아웃을 결정하는 구조를 만들지 않는다. 각 층은 자기 책임만 진다. **디자인 시스템 층은 선택 사항이다** — 프로젝트에 등록된 디자인 시스템이 없으면 이 스킬은 프로젝트가 이미 갖고 있는 컴포넌트/프리미티브(또는 순수 HTML+스타일링)로 구현한다.

## Step 0 — Prerequisites

### 입력 모드 판단 (`{input_mode}`) — 가장 먼저 결정

이후 Step 1/Step 10을 포함한 모든 단계가 이 값에 따라 갈라진다.

- **Case 1 — `mcp`**: Claude Design 화면 링크(`open_url`) 또는 `project_id`로 시작 — 이 세션에 라이브 `claude-design` MCP 접근이 있다.
- **Case 2 — `local`**: `design-export` 스킬로 이미 뽑아온 로컬 `.dc.html` 파일(들)로 시작 — Claude Design 계정/MCP 접근이 전혀 없어도 된다.

**스킬을 시작하면 가장 먼저 사용자에게 Case 1 / Case 2 중 어떤 방식으로 진행할지 직접 묻는다** (예: AskUserQuestion). 사용자의 최초 요청 안에 이미 방식이 명확히 드러나 있으면(예: URL/`project_id`를 직접 주며 "Case 1로" 라고 명시, 또는 로컬 `.dc.html` 경로를 주며 "MCP 접근 없이" 라고 명시) 그 답을 그대로 확정하고 다시 묻지 않는다. 그 외에는 URL/파일 경로만 보고 조용히 넘겨짚지 않고 반드시 질문해서 `{input_mode}`를 확정한다 — 애매한 채로 다음 단계로 진행하지 않는다.

순서대로 확인하고, 하나라도 실패하면 **거기서 멈춘다** — 다음 단계로 조용히 진행하지 않는다.

1. **(Case 1에서만) `claude-design` MCP 툴 호출 가능 여부** — `design-env-setup`의 최종 체크리스트가 만족됐는지 확인 (별도로 재검증하지 말고 `ToolSearch query="select:mcp__claude-design__list_design_systems"`로 spot-check). **Case 2는 이 항목을 건너뛴다** — MCP 접근이 없는 게 정상 전제다.
2. **타겟 프로젝트에 디자인 시스템 패키지가 있다면, 그게 실제 npm 의존성으로 설치돼 있는지** — `package.json`을 직접 읽어 확인한다. Claude Design 안에서 쓰던 `window.<Bundle>.*` 전역 참조는 소스화 결과물에 쓰지 않으므로, 진짜 `import`가 가능해야 한다. 없으면 설치부터 하고(패키지 매니저로 add + 필요하면 스타일시트를 entry CSS에 import) 다음 단계로. **디자인 시스템이 애초에 없는 프로젝트라면 이 항목은 그냥 통과** — Step 4/6에서 항상 `CREATE_LOCAL`/`COMPOSE`로 흐르게 된다. (두 케이스 공통)
3. **`convention.md`가 프로젝트에 존재하는지** — 없으면 멈추고 사용자에게 먼저 만들어달라고 요청한다. CLAUDE.md 기본값으로 조용히 대체하지 않는다 (정보 없으면 추측하지 않는다는 이 스킬 전체의 원칙과 동일). (두 케이스 공통)

## Step 1 — Design Source Validation

`{input_mode}`(Step 0에서 정한 값)로 시작한다 — Case 1은 사용자가 준 Claude Design 화면 링크(`open_url`, 여러 개 가능), Case 2는 로컬 `.dc.html` 파일(들)로, 경로를 이미 받았으면 그걸로 진행하고 아니면 먼저 로컬 파일을 찾아 사용자에게 어떤 파일로 작업할지(또는 새 파일 추가할지) 확인한다. 코드 생성으로 바로 넘어가지 않고 먼저 검증한다 — 케이스별 파일 선택/검증 방법과 출력 형식은 `references/design-artifact.md`.

```
✓ Design URL accessible
✓ Screen found
✓ Design specification (.dc.html) readable
✓ Assets accessible

Status: READY
```

실패하면 `Status: BLOCKED` + 이유를 명시하고 멈춘다.

## Step 2 — Requirement Extraction

Design Artifact에서 Purpose/UI/Behavior/Data/State를 추출한다. **Behavior와 State는 정적 아티팩트에서 뽑아낼 수 없는 정보다** — Claude Design 프레임은 화면당 하나의 정적 상태만 보여주므로, 초안을 만든 뒤 반드시 사용자에게 확인받는다. Design에 명시되지 않은 business behavior를 임의로 만들지 않는다. 방법과 출력 형식은 `references/requirement-extraction.md`.

## Step 3 — Layout Planning

화면 구조와 UI 컴포넌트를 분리해서 본다. `StyleGallery`는 Layout Pattern만 담당 — 이미 `spec-to-design`의 `../spec-to-design/references/stylegallery-recipe.md`에 정리된 `material-search`/`material-get` 사용법을 그대로 재사용한다 (같은 내용을 여기서 다시 쓰지 않음). 적합한 패턴이 없으면 억지로 끼워 맞추지 않고 명시한다.

## Step 4 — Component Mapping

Design의 UI 요소를 실제 Repository 컴포넌트에 매핑한다. 우선순위(디자인 시스템이 있다면 그것부터)와 매핑 시 반드시 확인해야 할 것들은 `references/component-mapping.md`.

## Step 5 — Repository Analysis (per-run)

Component Mapping/Gap Analysis 직전에, 그 시점 타겟 프로젝트에 실제로 존재하는 컴포넌트·상태관리·라우팅·폴더 구조를 다시 확인한다 (캐시된 지식에 의존하지 않음 — 이전 실행 이후 프로젝트가 바뀌었을 수 있음). **아직 아무 패턴도 없으면("cold start") "기존 패턴 없음, 이 화면이 baseline이 됨"이라고 명시**하고 넘어간다 — 없는 걸 억지로 찾지 않는다.

## Step 6 — Component Gap Analysis

Step 4/5 결과를 `REUSE / COMPOSE / EXTEND / CREATE_LOCAL / CREATE_SHARED / BLOCKED`로 분류한다. 분류 기준과 `CREATE_LOCAL` vs `CREATE_SHARED` 판단 기준은 `references/gap-analysis.md`. 결과는 `templates/gap-report.md` 형식으로 정리해 필요하면 사용자에게 보여준다. **AI가 임의로 공통(shared) 컴포넌트를 만들지 않는다** — 승격은 근거가 있을 때만. 디자인 시스템이 없는 프로젝트에서는 대부분의 UI 요소가 `CREATE_LOCAL` 또는 `COMPOSE`로 분류되는 게 정상이다 — 그 자체가 문제가 아니다.

## Step 7 — Implementation Plan

코드 생성 전에 반드시 `templates/implementation-plan.md` 형식으로 계획을 작성하고, **BLOCKED/Unknown 항목이 있으면 명확히 표시한 뒤 사용자 확인을 받는다.** 확인 없이 다음 단계로 넘어가지 않는다.

## Step 8 — Source Code Generation

Implementation Plan이 확정된 뒤에만 실행한다. 반드시 지킬 것 / 금지 사항 / 기존 컴포넌트가 Design을 완전히 만족 못 할 때의 해결 순서(props/config → 조합 → 확장 → 그래도 안 되면 신규) / `vercel-react-best-practices`가 설치돼 있을 때 성능 컨벤션 적용 방법은 `references/implementation-rules.md`. `convention.md`가 폴더 구조·네이밍·export 정책의 최종 권한을 가진다 — 이 스킬은 그걸 따를 뿐 재정의하지 않는다.

## Step 9 — Build / Typecheck / Lint

```bash
pnpm typecheck && pnpm lint && pnpm build
```

(프로젝트가 다른 패키지 매니저를 쓰면 그에 맞는 동일한 스크립트로 대체한다.) 실패하면 다음 단계(QA)로 넘어가지 않고 고친다.

## Step 10 — Visual / Functional QA

`pnpm dev`로 실제 렌더 후 브라우저 스크린샷을 비교 대상과 비교한다 — Case 1은 Claude Design의 `open_url` 스크린샷, Case 2는 사용자가 제공한 참고 이미지(없으면 정적 마크업 구조만으로 QA). `{input_mode}`별 처리와 체크 항목·결과 표기 형식은 `references/qa-rules.md`. 화면이 여러 개면 화면별로 이 체크포인트를 거친 뒤 다음 화면으로 넘어간다 (`spec-to-design`과 동일한 패턴).

## Step 11 — Implementation Report

`templates/implementation-report.md` 형식으로 최종 정리: 구현된 것, 재사용/신규 컴포넌트, Design Gap, 검증/QA 결과, 남은 이슈.

## Reference docs

| Doc | Load when |
|---|---|
| `references/design-artifact.md` | Step 1 — 검증 항목, Design Artifact 스키마, "정보 없으면 추측 금지" 규칙 |
| `references/requirement-extraction.md` | Step 2 |
| `references/layout-planning.md` | Step 3 — 실제 사용법은 spec-to-design의 stylegallery-recipe.md로 연결 |
| `references/component-mapping.md` | Step 4 — 디자인 시스템 매핑 우선순위 (있는 경우) + 없을 때의 기본 흐름 |
| `references/gap-analysis.md` | Step 6 |
| `references/implementation-rules.md` | Step 8 — includes when/how to apply `vercel-react-best-practices`, if installed |
| `references/qa-rules.md` | Step 10 |
