---
name: style-classifier
description: >
  코드 스타일 마이닝 결과를 입력받아 카테고리별로 체계적으로 분류하고 통계를 산출.
  유사 패턴 병합, 중복 제거, 우선순위 정렬 수행. 사용자가 의사결정하기 쉬운 형태로 가공.
  "스타일 분류", "패턴 통계", "변종 분석 결과 정리"가 필요할 때 반드시 이 스킬을 사용할 것.
---

# Style Classifier — 스타일 분류 및 통계

Style Miner의 원시 발견 데이터를 가공하여 사용자 의사결정에 최적화된 형태로 변환하는 스킬.

## 실행 조건
- `_workspace/phase-1-mining/`에 최소 1개 이상의 JSON 결과 파일이 존재할 때
- `code-style-orchestrator`의 Phase 2로 실행될 때

## 워크플로우

### Step 1: 원시 데이터 로드
`_workspace/phase-1-mining/`의 모든 JSON 파일을 읽는다.

### Step 2: 패턴 병합 및 정규화
유사 패턴을 통합하고 의미 없는 차이는 제거

### Step 3: 우선순위 산정
각 카테고리에 1-10 우선순위 점수 부여

### Step 4: 의사결정 자료 생성
`_workspace/phase-2-classified/patterns.json` 생성

## 원칙
- 극단적 불균형은 "거의 통일됨"으로 표시
- 균형 비슷할수록 우선순위 상향
- 각 변종마다 대표 샘플 1개씩만
