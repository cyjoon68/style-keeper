---
name: style-bulk-applier
description: >
  사용자가 결정한 코드 스타일 변경사항을 전체 코드베이스에 일괄 적용.
  AST-aware 리팩토링(ast-grep)을 우선 사용하고, 복잡한 변환은 에이전트에 위임.
  카테고리별 순차 적용 + LSP 검증 + 롤백 안전장치 포함.
  "스타일 일괄 적용", "코드 컨벤션 통일 실행", "패턴 변경 적용"이 필요할 때 반드시 이 스킬을 사용할 것.
---

# Style Bulk Applier — 스타일 일괄 적용

사용자의 결정에 따라 전체 코드베이스의 코드 스타일을 안전하게 일괄 변경하는 스킬.

## 실행 조건
- `_workspace/phase-3-decisions/decisions.json`에 최소 1개 이상의 결정이 있을 때

## 워크플로우

### Step 1: 변경 계획 수립
각 결정을 AST 변환 규칙으로 매핑

### Step 2: 카테고리별 순차 적용
1. Dry-run → 영향 범위 확인
2. 실제 적용
3. LSP 검증
4. 실패 시 롤백

### Step 3: 복잡한 패턴 처리
AST-grep 불가능한 패턴은 에이전트 위임

### Step 4: 변경 보고서 생성
`_workspace/phase-4-applied/summary.md`

## 원칙
- 변경 전 dry-run 필수
- 한 카테고리씩 순차 적용
- LSP 에러 발생 시 즉시 롤백
- 의미 동일한 변환만 수행
