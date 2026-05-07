---
name: style-keeper
description: >
  코드베이스 전반의 ESLint/Prettier가 잡지 못하는 코드 스타일/패턴 불일치를
  병렬 탐색하고, 사용자에게 하나씩 물어서 일괄 적용하는 스킬.
  "코드 스타일 정리", "패턴 통일", "코드베이스 일관화", "스타일 분석",
  "컨벤션 맞춰줘", "전체 코드 스타일 통일해줘" 요청 시 반드시 이 스킬을 사용할 것.
  "재실행", "다시 실행", "업데이트" 후속 요청에도 사용.
  함수 선언 방식, 타입 정의 방법, 컴포넌트 패턴, import/export 컨벤션,
  에러 핸들링 패턴 등 고수준 코드 패턴의 불일치를 찾아 통일한다.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Glob
  - Grep
  - ast_grep_search
  - ast_grep_replace
  - lsp_diagnostics
  - task
triggers:
  - 코드 스타일
  - 패턴 통일
  - 일관화
  - 스타일 분석
  - 컨벤션
  - 코드 정리
  - 스타일 통일
---

# Style Keeper — 코드 스타일 통일화 오케스트레이터

코드베이스 전반에서 ESLint/Prettier가 잡지 못하는 고수준 코드 패턴 불일치를 병렬 탐색하고,
사용자에게 스타일을 하나씩 질문한 후 일괄 적용한다.

## 이 스킬이 해결하는 문제

- 같은 프로젝트인데 파일마다 함수 선언 방식이 다름 (`function` vs `const = () =>`)
- `interface`와 `type alias`가 혼용됨
- 컴포넌트 선언 패턴이 제각각 (`export default function` vs `export const`)
- import/export 스타일이 통일되지 않음
- ESLint/Prettier로는 잡을 수 없는 "취향/컨벤션" 수준의 차이
- 타입 정의 방식, 에러 핸들링 패턴, 조건문 스타일 등 프레임워크 레벨의 불일치

---

## 워크플로우 개요

```
Phase 0: 린트/프리티어 선행 질문
    │
Phase 1: 4개 Miner 병렬 팬아웃
    ├─ Miner A: 함수/메서드 스타일
    ├─ Miner B: 타입/인터페이스/import 스타일
    ├─ Miner C: 컴포넌트/모듈/export 패턴
    └─ Miner D: 에러 핸들링/조건문/nullish 처리
    │
Phase 2: 변종 분류 + 통계 + 우선순위 (팬인)
    │
Phase 3: 카테고리별 사용자 질문 (순차)
    │
Phase 4: AST-grep 기반 일괄 적용 + LSP 검증 (순차)
    │
Phase 5: 최종 정리 + 포맷팅
```

---

## Phase 0: 린트/프리티어 선행 질문

워크플로우 시작 전, 사용자에게 질문한다:

> "분석 전에 린트 + 프리티어 포맷팅을 먼저 실행할까요?
> (y = 실행 후 시작, n = 바로 분석 시작)"

- **y** → 프로젝트에 맞는 린트/포맷 명령어 실행
  - Node.js: `pnpm lint && pnpm format` 또는 `npx eslint . --fix && npx prettier --write .`
  - Python: `ruff check . --fix && ruff format .`
  - Go: `gofmt -w .`
  - 기타: 프로젝트 설정에 따라 적절한 명령어 실행
- **n** → 바로 Phase 1로 진행

> 이 단계는 코드베이스를 1차 정규화하여 이후 스타일 분석을 더 정확하게 하기 위함.
> ESLint/Prettier가 다루는 포맷팅 수준(세미콜론, 들여쓰기, 따옴표 등) 자체가 이 스킬의 대상은 아니다.

---

## Phase 1: 병렬 스타일 마이닝 (팬아웃)

**실행 모드:** Sub-agents (병렬 팬아웃)

최소 4개 이상의 탐색기를 동시에 실행하여 코드베이스 전역의 스타일 변종을 발견한다.

### 탐색 카테고리 표

| Miner | 담당 카테고리 | 주요 탐색 패턴 | 도구 |
|-------|-------------|--------------|------|
| **Miner A** | 함수/메서드 선언 | `function foo()` vs `const foo = () => {}` | ast-grep |
| **Miner B** | 타입/import | `interface` vs `type`, `import type` 방식 | ast-grep + grep |
| **Miner C** | 컴포넌트/export | `export default function` vs `export const`, named vs default | ast-grep + explore |
| **Miner D** | 에러/조건문/nullish | try-catch 스타일, early return, `?.` vs `&&`, Promise 패턴 | grep + explore |

### 병렬 실행 코드

```typescript
// 4개 이상의 탐색기를 동시에 run_in_background=true로 실행
const minerA = task(
  subagent_type="explore",
  load_skills=[],
  description="Mine function declarations",
  prompt="[GOAL] Find function declaration style variations. [SEARCH] 1) `function name() {}` vs `const name = () => {}` vs `const name = function() {}`. Use ast-grep for function patterns and grep for arrow function assignments. [OUTPUT] JSON with count, file paths (with line numbers), and sample snippet per variant. Exclude node_modules, dist, build, .git.",
  run_in_background=true
)

const minerB = task(
  subagent_type="explore",
  load_skills=[],
  description="Mine type definitions",
  prompt="[GOAL] Find type definition style variations. [SEARCH] 1) `interface X {}` vs `type X = {}` vs `type X = {...}&{...}`, 2) `import type { X }` vs `import { X }`, 3) inline type vs extracted type. [OUTPUT] JSON with count, file paths, and samples. Exclude node_modules, dist.",
  run_in_background=true
)

const minerC = task(
  subagent_type="explore",
  load_skills=[],
  description="Mine component/export styles",
  prompt="[GOAL] Find component declaration and export pattern variations. [SEARCH] 1) `export default function Comp()` vs `export const Comp = () =>` vs `function Comp(){} export default Comp`, 2) named exports vs default exports, 3) destructuring patterns. [OUTPUT] JSON with counts, file paths, and samples. Exclude node_modules, dist.",
  run_in_background=true
)

const minerD = task(
  subagent_type="explore",
  load_skills=[],
  description="Mine error handling patterns",
  prompt="[GOAL] Find error handling and control flow pattern variations. [SEARCH] 1) try-catch logging styles, 2) early return vs nested if-else, 3) optional chaining `?.` vs `&&` vs null checks `if (x != null)`, 4) Promise.catch() vs async/await try-catch. [OUTPUT] JSON with counts, file paths, and samples. Exclude node_modules, dist.",
  run_in_background=true
)

// 필요시 추가 Miner (프레임워크별 특화 패턴)
// const minerE = task(subagent_type="explore", ...) // React hooks patterns
// const minerF = task(subagent_type="explore", ...) // NestJS controller patterns

// 모든 Miner 결과를 background_output()으로 수집 (시스템 알림 후)
```

### 수집 결과 저장

모든 Miner의 결과를 다음 구조로 수집:
```
_workspace/phase-1-mining/
├── miner-a-functions.json
├── miner-b-types.json
├── miner-c-components.json
├── miner-d-error-handling.json
└── ... (추가 Miner 결과)
```

각 결과 파일 형식:
```json
{
  "category": "function-declaration",
  "patterns": [
    {
      "name": "arrow-vs-named",
      "description": "화살표 함수 vs named function",
      "variants": [
        {
          "style": "arrow-const",
          "count": 89,
          "files": ["src/components/Header.tsx:15", "src/utils/format.ts:42"],
          "sample": "const foo = () => { ... }"
        },
        {
          "style": "named-function",
          "count": 25,
          "files": ["src/utils/helpers.ts:3", "src/services/api.ts:88"],
          "sample": "function foo() { ... }"
        }
      ]
    }
  ]
}
```

**원칙:**
- ESLint/Prettier 영역(세미콜론, 들여쓰기, 따옴표, trailing comma 등)은 절대 탐색하지 않는다
- 과도한 탐색 금지 — 카테고리당 20개 이상 패턴 발견 시 상위 10개만 기록
- 모든 결과에 파일 경로 + 라인 번호 포함
- AST-grep 우선 사용 (단순 텍스트 검색보다 정확)

---

## Phase 2: 변종 분류 및 통계 (팬인)

**실행 모드:** Sub-agents (단일 분류기)

Phase 1의 모든 결과를 수집하여 통합 분류한다.

### Step 2-1: 결과 수집
모든 Miner의 `background_output()`을 수집하여 하나의 데이터셋으로 통합.

### Step 2-2: 분류 및 병합

```typescript
task(
  category="deep",
  load_skills=[],
  description="Classify style variants",
  prompt="[GOAL] Classify, merge, and prioritize style mining results. [INPUT] Results from 4 parallel miners. [TASKS] 1) Merge similar patterns across miners (e.g., 'function declaration' and 'component declaration' may overlap). 2) Remove duplicates. 3) Calculate percentages. 4) Assign priority 1-10: balanced splits (40-60%) = high priority, extreme ratios (>95%) = low. 5) Set ask_user=true for meaningful variations. [OUTPUT] Write to _workspace/phase-2-classified/patterns.json.",
  run_in_background=false
)
```

### Step 2-3: 우선순위 산정 기준

| 비율 | 우선순위 | ask_user | 설명 |
|------|---------|----------|------|
| 50:50 ~ 70:30 | 8-10 | true | 실제 논의 필요 |
| 70:30 ~ 90:10 | 4-7 | true | 통일해도 좋음 |
| 90:10 ~ 98:2 | 1-3 | true | 거의 통일됨, 그래도 물어봄 |
| 98:2 이상 | 0 | false | 사실상 이미 통일됨 |
| 변종 1개뿐 | 0 | false | 완전 일관됨 |

### Step 2-4: 조기 종료 체크
모든 카테고리의 `ask_user`가 `false`면:
> "코드베이스가 이미 일관된 상태입니다. 변경할 스타일이 없습니다. 🎉"
→ 워크플로우 종료

### 분류 결과 형식

```json
{
  "categories": [
    {
      "id": "func-declaration",
      "title": "함수 선언 방식",
      "description": "화살표 함수와 named function이 혼용됨",
      "priority": 9,
      "variants": [
        {
          "id": "A",
          "label": "Arrow function (`const foo = () => {}`)",
          "count": 89,
          "percentage": 78.1,
          "sample_path": "src/components/Header.tsx:15"
        },
        {
          "id": "B",
          "label": "Named function (`function foo() {}`)",
          "count": 25,
          "percentage": 21.9,
          "sample_path": "src/utils/helpers.ts:3"
        }
      ],
      "recommendation": "A",
      "ask_user": true
    }
  ]
}
```

**원칙:**
- 극단적 불균형(98% vs 2%)은 "거의 통일됨" → 우선순위 낮게
- 균형 비슷한 것(55% vs 45%)일수록 우선순위 상향
- 각 변종마다 대표 샘플 1개씩만 포함
- 사용자가 이해하기 쉬운 용어 사용 (실제 코드 예시 위주)

---

## Phase 3: 사용자 스타일 질문 (순차)

**실행 모드:** 순차 (사용자 상호작용)

Phase 2의 결과에서 `ask_user: true`인 카테고리를 **하나씩** 사용자에게 제시한다.

### Step 3-1: 카테고리 로드
`patterns.json`에서 우선순위 내림차순 정렬하여 순차 질문 준비.

### Step 3-2: 카테고리별 질문

각 카테고리를 다음 형식으로 제시:

```
## ({current}/{total}) {title} (우선순위: {priority}/10)

{description}

**현재 분포:**
- (A) {label_A} — {count_A}개 ({percentage_A}%)
  샘플: {sample_path_A}
- (B) {label_B} — {count_B}개 ({percentage_B}%)
  샘플: {sample_path_B}

**변경 영향:** A 선택 시 {count_B}개 파일 변경, B 선택 시 {count_A}개 파일 변경

어느 스타일로 통일할까요?
[A] {label_A}로 통일 (권장)
[B] {label_B}로 통일
[C] 이 카테고리는 건너뛰기
```

### Step 3-3: 결정 수집

```typescript
const decision = await question({
  header: `스타일 결정 (${current}/${total})`,
  question: "...",
  options: [
    { label: "A 선택", description: "다수 스타일로 통일" },
    { label: "B 선택", description: "소수 스타일로 통일" },
    { label: "건너뛰기", description: "이 카테고리는 나중에" }
  ]
});
```

### Step 3-4: 최종 승인
모든 카테고리 완료 후 변경 예정 요약 표시:

```
=== 변경 예정 요약 ===
[적용 예정]
  - 함수 선언 방식: 화살표 함수로 통일 (25개 파일 변경)
  - 타입 정의: interface로 통일 (12개 파일 변경)

[건너뛰기]
  - import 스타일: 사용자 보류

일괄 적용을 진행할까요? (y/n)
```

### 결정 저장
```json
{
  "decisions": [
    {
      "category_id": "func-declaration",
      "title": "함수 선언 방식",
      "chosen_style": "arrow-const",
      "chosen_label": "Arrow function",
      "affects_files_count": 25
    }
  ],
  "skipped": ["import-style-category-id"]
}
```

**원칙:**
- **한 번에 하나의 질문만** — 절대 여러 카테고리를 동시에 질문하지 않음
- 코드 샘플을 항상 제시 — 사용자가 실제 차이를 볼 수 있어야 함
- 영향 범위 투명하게 공개 — "이 선택 시 N개 파일 변경"
- "건너뛰기" 옵션 항상 제공
- 중립적인 어조 유지 — 사용자의 선택을 평가하지 않음

---

## Phase 4: 일괄 적용 및 검증 (순차)

**실행 모드:** 순차 적용 + 검증

사용자의 결정에 따라 각 카테고리를 순차적으로 적용하고 검증한다.

### Step 4-1: 변환 규칙 매핑

각 결정을 AST 변환 규칙으로 매핑:

| 카테고리 | 선택 | 변환 규칙 (ast-grep) |
|---------|------|--------------------|
| func-declaration: named→arrow | arrow-const | `function $NAME($$$) { $$$ }` → `const $NAME = ($$$) => { $$$ }` |
| type-definition: type→interface | interface | `type $NAME = { $$$ }` → `interface $NAME { $$$ }` |
| type-import: type→inline | inline-import | `import type { $NAME } from $SOURCE` → `import { $NAME } from $SOURCE` |
| component: named→arrow | arrow-comp | `export function $NAME($$$) { $$$ }` → `export const $NAME = ($$$) => { $$$ }` |

### Step 4-2: 카테고리별 순차 적용

각 카테고리를 다음 순서로 처리:

1. **Dry-run**: `ast_grep_replace(pattern="...", rewrite="...", dryRun=true)`로 영향 범위 확인
2. **사용자 확인**: "N개 파일 변경, 진행할까요?"
3. **실제 적용**: `ast_grep_replace(pattern="...", rewrite="...", dryRun=false)`
4. **LSP 검증**: `lsp_diagnostics(filePath=project_root)` — 신규 에러 0 확인
5. **에러 발생 시 롤백**: `Bash(git checkout -- .)` 또는 `Bash(git restore .)`

### Step 4-3: 복잡한 패턴 처리

AST-grep으로 1:1 변환이 불가능한 패턴:
```typescript
task(
  category="deep",
  load_skills=["nestjs-best-practices", "vercel-react-native-skills"],
  description="Apply {category} style transformation",
  prompt="[GOAL] Transform specific files to follow decided style. [FILES] ... [RULE] ... [CONSTRAINTS] Only change specified pattern. No other code changes. Verify with LSP after changes.",
  run_in_background=true
)
```

### Step 4-4: 변경 보고서

`_workspace/phase-4-applied/summary.md` 생성:

```markdown
# 스타일 변경 적용 보고서

## 변경 완료
| 카테고리 | 변경 전 → 변경 후 | 변경 파일 수 | 상태 |
|---------|-----------------|------------|------|
| 함수 선언 | named → arrow | 25 | ✅ |
| 타입 정의 | type → interface | 12 | ✅ |

## 건너뛰기/실패
| 카테고리 | 사유 |
|---------|------|
| import 스타일 | 사용자 보류 |

## LSP Diagnostics
- 변경 전: 3 warnings
- 변경 후: 3 warnings (기존과 동일, 신규 에러 없음)

## Diff 샘플
### 함수 선언
```diff
- function getTotal(items: Item[]): number {
+ const getTotal = (items: Item[]): number => {
```
```

**원칙:**
- 변경 전 dry-run으로 영향 범위 반드시 확인
- 한 카테고리씩 순차 적용 + LSP 검증
- LSP 에러 발생 시 즉시 롤백 후 다른 방식 재시도
- AST-grep을 우선 사용하고, 불가능한 경우만 에이전트 위임
- 의미가 동일한 변환만 수행 (동작 변경 금지)

---

## Phase 5: 최종 정리

1. **ESLint/Prettier 재포맷팅 (선택)**
   > "스타일 변경 후 포맷팅 확인을 위해 린트+프리티어를 다시 실행할까요? (y/n)"
2. **변경 이력 기록** — 작업한 내용 사용자에게 최종 요약
3. **보고서 경로 안내** — `_workspace/phase-4-applied/summary.md`

```
=== Style Keeper 실행 완료 ===

📊 처리 결과:
- 탐색한 패턴 카테고리: 6개
- 사용자 질문: 5개
- 적용된 변경: 3개 카테고리
- 건너뛰기: 1개
- 변경된 파일: 총 42개
- 신규 LSP 에러: 0건 ✅

📁 보고서: _workspace/phase-4-applied/summary.md
```

---

## 데이터 전달 프로토콜

| Phase | 입력 | 출력 |
|-------|------|------|
| Phase 0 | 사용자 응답 | - (린트 실행 결정) |
| Phase 1 | 프로젝트 경로 | `_workspace/phase-1-mining/*.json` |
| Phase 2 | `_workspace/phase-1-mining/*.json` | `_workspace/phase-2-classified/patterns.json` |
| Phase 3 | `_workspace/phase-2-classified/patterns.json` | `_workspace/phase-3-decisions/decisions.json` |
| Phase 4 | `_workspace/phase-3-decisions/decisions.json` | `_workspace/phase-4-applied/summary.md` |
| Phase 5 | Phase 4 결과 | 최종 요약 |

---

## 에러 핸들링

| 에러 유형 | 처리 |
|----------|------|
| **탐색 결과 없음** | "변경할 스타일 불일치가 없습니다" → 조기 종료 |
| **사용자 결정 없음** (모두 건너뛰기) | "적용할 변경사항이 없습니다" → 종료 |
| **AST-grep 변환 실패** | 에이전트 위임 폴백, 실패 시 건너뛰고 보고서 기록 |
| **LSP 에러 발생** | 즉시 롤백(`git restore .`) → 다른 방식 재시도 → 실패 시 건너뛰기 |
| **중간 중단** | 현재 상태 `_workspace/`에 저장 → 재실행 시 Phase 0에서 재개 여부 질문 |
| **대규모 변경 (50+ 파일)** | 사용자에게 추가 승인 요청 후 진행 |

---

## 테스트 시나리오

### 정상 흐름
1. Phase 0: y → 린트/프리티어 실행
2. Phase 1: 4개 Miner 병렬 실행 → 6개 카테고리 발견
3. Phase 2: 4개 카테고리 ask_user=true, 2개 false
4. Phase 3: 4개 카테고리 질문 → 3개 선택, 1개 건너뛰기
5. Phase 4: 3개 카테고리 순차 적용 → LSP 통과
6. Phase 5: 보고서 생성 → 완료

### 에러 흐름 (탐색 결과 없음)
1. Phase 0: n → 바로 분석
2. Phase 1: 4개 Miner 실행 → 모든 변종 1개뿐
3. Phase 2: 모든 카테고리 ask_user=false
4. → "코드베이스가 이미 일관됨" 🎉 → 조기 종료

### 에러 흐름 (사용자 전부 건너뛰기)
1. Phase 3: 모든 카테고리를 사용자가 건너뜀
2. decisions.json이 비어있음
3. → "적용할 변경사항이 없습니다" → Phase 4/5 생략

---

## 설치

```bash
# OpenCode
git clone --single-branch --depth 1 https://github.com/cyjoon68/style-keeper.git ~/.config/opencode/skills/style-keeper
```

### 수동 설치
레포를 클론한 후 `~/.config/opencode/skills/style-keeper/`에 심링크 또는 복사:

```bash
git clone https://github.com/cyjoon68/style-keeper.git
ln -s "$(pwd)/style-keeper" ~/.config/opencode/skills/style-keeper
```

### 사용법
AI 에이전트에게 말한다:
> "코드 스타일 정리해줘"
> "패턴 통일해줘"
> "코드베이스 일관화해줘"
> "스타일 분석해줘"

---

## 참고

### AST-grep 유용한 패턴 모음

```typescript
// 함수 선언 → 화살표 함수
ast_grep_replace(
  pattern: "function $NAME($$$) { $$$ }",
  rewrite: "const $NAME = ($$$) => { $$$ }",
  lang: "typescript"
)

// type → interface (객체 타입만)
ast_grep_replace(
  pattern: "type $NAME = { $$$ }",
  rewrite: "interface $NAME { $$$ }",
  lang: "typescript"
)

// export function → export const arrow
ast_grep_replace(
  pattern: "export function $NAME($$$) { $$$ }",
  rewrite: "export const $NAME = ($$$) => { $$$ }",
  lang: "tsx"
)
```

### 프로젝트별 린트/포맷 명령어 참고

| 프로젝트 타입 | 린트 명령어 | 포맷 명령어 |
|-------------|------------|------------|
| Node.js (pnpm) | `pnpm lint` | `pnpm format` |
| Node.js (npm) | `npm run lint` | `npx prettier --write .` |
| Python (ruff) | `ruff check . --fix` | `ruff format .` |
| Go | `golint ./...` | `gofmt -w .` |
| Rust | `cargo clippy` | `cargo fmt` |
