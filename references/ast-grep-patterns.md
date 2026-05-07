# AST-grep 패턴 모음

Style Keeper가 코드 변환에 사용하는 AST-grep 패턴 모음.

## 함수 선언 변환

### named function → arrow function

```typescript
// Pattern
function $NAME($$$) { $$$ }

// Rewrite
const $NAME = ($$$) => { $$$ }
```

### export named function → export const arrow

```typescript
// Pattern
export function $NAME($$$) { $$$ }

// Rewrite
export const $NAME = ($$$) => { $$$ }
```

### async named function → async arrow

```typescript
// Pattern
async function $NAME($$$) { $$$ }

// Rewrite
const $NAME = async ($$$) => { $$$ }
```

## 타입 정의 변환

### type alias (객체 타입) → interface

```typescript
// Pattern  (객체 타입만)
type $NAME = { $$$ }

// Rewrite
interface $NAME { $$$ }
```

### type alias (유니온) → 그대로 (interface로 변환 불가)

유니온 타입은 interface로 변환할 수 없으므로 건너뛴다.
```typescript
// 변환 불가 — 건너뛰기
type Status = 'active' | 'inactive'
type Props = A & B
```

## import 변환

### type import → inline import

```typescript
// Pattern
import type { $NAME } from $SOURCE

// Rewrite
import { $NAME } from $SOURCE
```

### default import → named import (특정 패턴)

```typescript
// Pattern
import $NAME from $SOURCE

// Rewrite (모듈에 named export가 있는 경우만)
import { $NAME } from $SOURCE
```

> 주의: default import를 named import로 바꾸는 것은 모듈의 export 구조를 알아야 하므로
> AST-grep만으로 안전하게 처리하기 어렵다. 가능하면 건너뛰거나 사용자에게 확인.

## 컴포넌트 선언 변환 (React)

### function component → arrow component

```typescript
// Pattern
export function $NAME({ $$$ }: $$$) { $$$ }

// Rewrite
export const $NAME = ({ $$$ }: $$$) => { $$$ }
```

### React.FC 제거

```typescript
// Pattern
const $NAME: React.FC<$$$> = ({ $$$ }) => { $$$ }

// Rewrite
const $NAME = ({ $$$ }: $$$) => { $$$ }
```

## export 변환

### 개별 export → 하단 export

```typescript
// Before (개별 export)
export const foo = () => {}
export const bar = () => {}

// After (하단 export)
const foo = () => {}
const bar = () => {}
export { foo, bar }
```

> AST-grep으로 한 번에 처리하기 어려움. 여러 파일의 export 패턴을 수집한 후
> 하단 export로 통일할 때는 에이전트에 위임하여 처리.

## 조건문 변환

### 중첩 if → early return

```typescript
// Pattern (찾기만 — 변환은 맥락에 따라 다름)
if ($COND) {
  if ($OTHER) {
    $$$BODY
    return $$$RESULT
  }
}

// 보통 다음과 같은 변환 의도:
// Before:
if (x) {
  if (y) {
    doSomething();
    return result;
  }
}

// After:
if (!x || !y) return;
doSomething();
return result;
```

> 조건문 변환은 맥락 의존적이므로 AST-grep으로 자동 변환하지 말고
> 발견만 하고 사용자에게 보고한다. 변환은 에이전트에 위임.

## 사용 팁

1. **항상 dryRun=true로 먼저 실행**하여 영향 범위 확인
2. **언어 설정 정확히**: TypeScript 파일은 `lang: "typescript"`, TSX는 가능하면 `lang: "tsx"`
3. **여러 패턴 매칭**: 같은 변환에 여러 패턴이 필요하면 순차 실행
4. **패턴 검증**: `ast_grep_search`로 먼저 패턴이 정확히 매칭되는지 확인 후 `ast_grep_replace` 실행
5. **메타변수 사용**: `$NAME`은 단일 노드, `$$$`는 여러 노드(표현식, 문장 등)에 사용
