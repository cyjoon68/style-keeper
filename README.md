<p align="center">
  <img src="https://img.shields.io/badge/Version-1.0.0-brightgreen.svg" alt="Version">
  <img src="https://img.shields.io/badge/License-MIT-blue.svg" alt="License">
  <img src="https://img.shields.io/badge/Platform-OpenCode-purple.svg" alt="Platform">
</p>

# Style Keeper — 코드 스타일 통일화 AI 스킬

> **"코드 스타일 정리해줘" — 한 마디로 코드베이스 전체의 패턴 불일치를 잡아낸다.**

ESLint와 Prettier가 잡지 못하는 **고수준 코드 패턴 불일치**를 찾아내고, 사용자에게 하나씩 물어본 후 일괄 적용하는 AI 스킬.

## 문제

- 같은 프로젝트인데 파일마다 함수 선언 방식이 다름
- `interface`와 `type` alias가 혼용됨
- 컴포넌트 선언 패턴이 제각각
- import/export 스타일이 통일되지 않음
- ESLint/Prettier로는 잡을 수 없는 "취향/컨벤션" 수준의 차이

## 해결 방식

```
Phase 0: 린트/프리티어 선행 질문
    │
Phase 1: 4개 탐색기 병렬 실행 ── 함수, 타입, 컴포넌트, 에러 핸들링
    │
Phase 2: 결과 분류 + 통계 + 우선순위 산정
    │
Phase 3: 사용자에게 하나씩 질문 ── "이거랑 이거 중에 뭐로 통일?"
    │
Phase 4: AST-grep 기반 일괄 적용 + LSP 검증
    │
Phase 5: 완료 보고서
```

## 설치

```bash
# OpenCode
git clone --single-branch --depth 1 https://github.com/cyjoon68/style-keeper.git ~/.config/opencode/skills/style-keeper
```

다른 AI 코딩 에이전트(Claude Code, Codex CLI, Cursor 등)도 유사한 방식으로 설치 가능.

## 사용법

AI 에이전트(Sisyphus, Claude Code 등)에게 말한다:

```
코드 스타일 정리해줘
패턴 통일해줘
코드베이스 일관화해줘
스타일 분석해줘
```

그러면 자동으로:
1. 린트/프리티어 실행 여부 확인
2. 4개 탐색기 동시에 돌려서 모든 스타일 변종 발견
3. 분류 후 하나씩 "이거랑 이거 중에 뭐로 통일할까?" 질문
4. 사용자가 선택한 스타일로 AST-grep 일괄 적용 + 타입 검증

## 동작 예시

```
> "코드 스타일 정리해줘"

[Phase 0] 린트+프리티어 먼저 실행할까요? (y/n) → y
[Phase 1] 4개 탐색기 병렬 실행 중...
[Phase 2] 6개 카테고리 발견 (4개 질문 대상)

[Phase 3] (1/4) 함수 선언 방식
  현재: 화살표 78%, named 22%
  (A) 화살표 함수로 통일 (25개 변경)
  (B) named function으로 통일
  (C) 건너뛰기
→ A

[Phase 3] (2/4) 타입 정의 방식
  현재: interface 65%, type 35%
  (A) interface로 통일 (35개 변경)
  (B) type alias로 통일
  (C) 건너뛰기
→ A

[Phase 4] 2개 카테고리 일괄 적용 중...
✅ LSP 검증 통과 (신규 에러 0건)
✅ 변경 완료: 60개 파일 수정

=== Style Keeper 실행 완료 ===
📊 변경된 파일: 60개
✅ 신규 LSP 에러: 0건
📁 보고서: _workspace/phase-4-applied/summary.md
```

## 라이선스

MIT
