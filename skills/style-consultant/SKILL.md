---
name: style-consultant
description: >
  분류된 코드 스타일 변종들을 사용자에게 하나씩 제시하고 결정을 수집.
  각 카테고리별로 다수/소수 스타일 옵션과 건너뛰기 옵션을 제공.
  사용자의 선택을 decisions.json에 기록하여 일괄 적용 단계에 전달.
  "스타일 결정", "코드 컨벤션 질문", "패턴 통일 의사결정"이 필요할 때 반드시 이 스킬을 사용할 것.
---

# Style Consultant — 사용자 스타일 컨설팅

분류된 코드 스타일 패턴을 사용자에게 명확히 제시하고 결정을 수집하는 스킬.

## 실행 조건
- `_workspace/phase-2-classified/patterns.json`에 `ask_user: true`인 카테고리가 있을 때

## 워크플로우

### Step 1: patterns.json 로드
`ask_user: true`인 카테고리만 추출. 우선순위 내림차순 정렬.

### Step 2: 카테고리별 질문 (순차)
각 카테고리를 하나씩 사용자에게 제시

### Step 3: 결정 저장
`_workspace/phase-3-decisions/decisions.json`에 저장

### Step 4: 결정 요약 제시
모든 카테고리 완료 후 변경 예정 사항 요약

## 원칙
- 한 번에 하나의 질문만
- 코드 샘플 항상 제시
- 영향 범위 투명하게 공개
- 건너뛰기 옵션 항상 제공
