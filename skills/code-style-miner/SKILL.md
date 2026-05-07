---
name: code-style-miner
description: >
  코드베이스 전역에서 ESLint/Prettier가 잡지 못하는 코드 스타일/패턴 변종을 병렬 탐색.
  함수 선언 방식, 타입 정의 방법, 컴포넌트 패턴, import/export 컨벤션, 에러 핸들링 패턴 등
  고수준 코드 패턴 불일치를 발굴. 스타일 통일화 워크플로우의 첫 단계.
  "코드 스타일 분석", "패턴 탐색", "코드베이스 스타일 변종 발견"이 필요할 때 반드시 이 스킬을 사용할 것.
---

# Code Style Miner — 병렬 코드 스타일 탐색

코드베이스 전역에서 고수준 코드 스타일/패턴 변종을 병렬로 발굴하는 스킬.
ESLint/Prettier 영역(세미콜론, 들여쓰기, trailing comma 등)은 절대 다루지 않는다.

## 실행 조건
- "코드 스타일 통일", "패턴 일관화" 요청 시
- `code-style-orchestrator`의 Phase 1로 실행될 때

## 워크플로우

### Step 1: 탐색 카테고리 정의
코드베이스 언어/프레임워크에 따라 탐색할 카테고리 결정:

| 카테고리 | 탐색 대상 |
|---------|----------|
| 함수 선언 | `function foo()` vs `const foo = () => {}` |
| 타입 정의 | `interface Foo {}` vs `type Foo = {}` |
| import 스타일 | `import type { X }` vs `import { X }` |
| 컴포넌트 선언 | `function Comp()` vs `const Comp = () =>` |
| 에러 핸들링 | try-catch 스타일, error logging 패턴 |
| 조건문 | early return vs if-else 중첩 |
| nullish 처리 | `?.` vs `&&` vs `if (x != null)` |

### Step 2: 병렬 탐색 실행
각 카테고리별로 AST-grep, Grep, explore 에이전트를 병렬 실행

### Step 3: 결과 수집 및 저장
`_workspace/phase-1-mining/{category}.json`에 저장

## 원칙
- ESLint/Prettier 영역 절대 침범 금지
- 과도한 탐색 금지 (카테고리당 20개 제한)
- 모든 결과에 파일 경로 + 라인 번호 포함
