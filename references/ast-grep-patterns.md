# AST-grep 패턴 모음 — genp 검증 완료

아래 패턴들은 실제 genp 코드베이스에서 검증된 AST-grep 패턴들입니다.

---

## 함수 선언 패턴

### named export function (hooks)
```typescript
// Pattern
export function use$NAME($$$) { $$$ }

// Lang: typescript
// Paths: apps/frontend/src
// Matches: 25개 (useProducts, useProduct, useCreateProduct, ...)

// Rewrite → export const (필요시)
export const use$NAME = ($$$) => { $$$ }

// Rewrite → function (기본 유지)
// (변경 없음 — 이미 일관됨)
```

### async bootstrap function (backend main)
```typescript
// Pattern
async function bootstrap() { $$$ }

// Lang: typescript
// Paths: apps/backend/*/src/main.ts
// Matches: 6개 (api-gateway, auth-service, chat-service, ...)
```

### 일반 named function (test helper)
```typescript
// Pattern
function build$NAME($$$) { $$$ }

// Lang: typescript
// Paths: apps/backend/*/src/__tests__
// Matches: 3개 (buildUser, buildModule, createDbMock)
```

### arrow function (callback)
```typescript
// Pattern
const $NAME = ($$$) => { $$$ }

// Lang: typescript
// Paths: apps/frontend/src
// Matches: 4개 (auth hooks 내부 mutate 등)
```

---

## 타입 정의 패턴

### interface (표준 — 43개)
```typescript
// Pattern
interface $NAME { $$$ }

// Lang: typescript
// Paths: apps/, packages/
// Matches: 43개 (ISignupInput, IAuthResponse, Product, ...)
```

### type 객체 (드묾 — 1개)
```typescript
// Pattern
type $NAME = { $$$ }

// Lang: typescript
// Paths: apps/frontend/src
// Matches: 1개 (AppThemes)

// Rewrite → interface
interface $NAME { $$$ }
// 주의: type alias가 interface로 안전하게 변환 가능한 경우만
// 유니온 타입(type X = A | B)은 변환 불가 → 건너뛰기
```

### type import — 31개 파일
```typescript
// Pattern (탐색 전용)
import type { $$$ } from $SOURCE

// Lang: typescript
// Matches: 31회

// Rewrite → inline import (TypeScript 설정 확인 필요)
import { $$$ } from $SOURCE
// ⚠️ TypeScript verbatimModuleSyntax와 충돌 가능
// ⚠️ 사용자에게 먼저 확인 필수
```

---

## 스타일 정의 패턴

### StyleSheet.create (표준 — 44개)
```typescript
// Pattern
StyleSheet.create({ $$$ })

// Lang: typescript
// Paths: apps/frontend/src
// Matches: 44개

// Rewrite → createStyleSheet (unistyles 마이그레이션)
createStyleSheet((theme) => ({ $$$ }))
// 주의: theme 사용을 위해 스타일 값 변경 필요
// 단순 패턴 변환만으로는 불충분 — 에이전트 위임 권장
```

---

## 에러 핸들링 패턴

### try-catch (4개 발견)
```typescript
// Pattern
try { $$$ } catch ($ERR) { $$$ }

// Lang: typescript
// Paths: apps/
// Matches: 4개
```

### bare catch (변수 없음)
```typescript
// Pattern
try { $$$ } catch { $$$ }

// Lang: typescript
// Matches: 1개 (apps/frontend/src/features/auth/api/hooks.ts)

// Rewrite → catch with variable
try { $$$ } catch (error) { $$$ }
```

### instanceof 에러 분기
```typescript
// Pattern
catch ($ERR) {
  if ($ERR instanceof HTTPError) { $$$ }
  $$$  
}

// Lang: typescript
// Matches: 1개 (apps/frontend/src/lib/apiClient.ts)
```

---

## 사용 팁 (genp 기준)

### 1. dryRun 모드로 먼저 확인

```typescript
// 실제 적용 전에 영향 범위 확인
ast_grep_replace(
  pattern: "import type { $$$ } from $SOURCE",
  rewrite: "import { $$$ } from $SOURCE",
  lang: "typescript",
  dryRun: true  // 변경 없이 매칭 결과만 확인
)
```

### 2. lang 설정

| 파일 확장자 | lang 값 |
|------------|---------|
| `.ts` | `typescript` |
| `.tsx` | `tsx` (또는 `typescript`) |
| `.js` | `javascript` |

### 3. paths 제한

genp에서 너무 많은 결과가 나오면 paths로 범위 제한:
```typescript
paths: ["apps/frontend/src"]           // frontend만
paths: ["packages/shared/src"]         // shared 패키지만
paths: ["apps/backend/chat-service"]   // 특정 서비스만
```

### 4. 제외할 경로

```typescript
globs: ["!**/node_modules/**", "!**/dist/**", "!**/.git/**"]
```

### 5. 실제 genp 적용 예시

```typescript
// 1. 모든 함수형 컴포넌트 찾기 (tsx)
ast_grep_search(
  pattern: "export default function $NAME($$$) { $$$ }",
  lang: "tsx",
  paths: ["apps/frontend/src"]
)

// 2. 모든 try-catch 찾기
ast_grep_search(
  pattern: "try { $$$ } catch ($ERR) { $$$ }",
  lang: "typescript",
  paths: ["apps"]
)

// 3. import type 패턴 분포 확인
// ast-grep으로는 직접 매칭 어려움 → grep 사용
// grep(pattern: "import type {", include: "*.ts")
```
