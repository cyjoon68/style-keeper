# 마이닝 카테고리 상세

Style Keeper Phase 1에서 탐색하는 각 카테고리의 상세 설명과 탐색 패턴.

## Miner A: 함수/메서드 선언

### 탐색 대상
| 패턴 ID | 설명 | 예시 A | 예시 B |
|---------|------|--------|--------|
| func-declaration | 함수 선언 방식 | `function foo() {}` | `const foo = () => {}` |
| async-func | 비동기 함수 방식 | `async function foo() {}` | `const foo = async () => {}` |
| method-definition | 객체 메서드 방식 | `{ foo() {} }` | `{ foo: () => {} }` |
| generator-func | 제너레이터 방식 | `function* foo() {}` | 아직 없음 |
| getter-setter | getter/setter 스타일 | `get foo() {}` | - |

### AST-grep 패턴
```typescript
// named function
ast_grep_search(pattern: "function $NAME($$$) { $$$ }", lang: "typescript")

// arrow function (const)
ast_grep_search(pattern: "const $NAME = ($$$) => { $$$ }", lang: "typescript")

// async named function
ast_grep_search(pattern: "async function $NAME($$$) { $$$ }", lang: "typescript")

// async arrow function
ast_grep_search(pattern: "const $NAME = async ($$$) => { $$$ }", lang: "typescript")
```

## Miner B: 타입/인터페이스/import

### 탐색 대상
| 패턴 ID | 설명 | 예시 A | 예시 B |
|---------|------|--------|--------|
| type-definition | 타입 정의 방식 | `interface Foo {}` | `type Foo = {}` |
| type-import | 타입 import 방식 | `import type { X }` | `import { X }` (type 키워드) |
| inline-type | 인라인 타입 vs 추출 | `props: { name: string }` | `interface Props { name: string }` |
| generic-constraint | 제네릭 제약 스타일 | `<T extends Foo>` | `<T extends Foo = Bar>` |
| enum-style | 열거형 방식 | `enum X {}` | `const X = {} as const` |

### AST-grep 패턴
```typescript
// interface
ast_grep_search(pattern: "interface $NAME { $$$ }", lang: "typescript")

// type alias (object)
ast_grep_search(pattern: "type $NAME = { $$$ }", lang: "typescript")

// type alias (union/intersection)
ast_grep_search(pattern: "type $NAME = $$$", lang: "typescript")

// type import
grep(pattern: "import type {", include: "*.ts")

// inline type in function params
ast_grep_search(pattern: "function $NAME($$$: { $$$ }) { $$$ }", lang: "typescript")
```

## Miner C: 컴포넌트/모듈/export

### 탐색 대상
| 패턴 ID | 설명 | 예시 A | 예시 B |
|---------|------|--------|--------|
| component-declaration | 컴포넌트 선언 방식 | `function Comp(){}` | `const Comp = () => {}` |
| export-style | export 방식 | `export default Comp` | `export { Comp }` |
| named-vs-default | named vs default export | `export const foo` | `export default foo` |
| index-rexport | index.ts re-export 방식 | `export * from './foo'` | `export { foo } from './foo'` |
| props-destructure | props 구조분해 방식 | `({ name }: Props)` | `(props: Props)` + 내부 destructure |
| memo-pattern | React.memo 사용 | `memo(Comp)` | `const Comp = memo(() => {})` |

### AST-grep 패턴
```typescript
// component function
ast_grep_search(pattern: "function $NAME($$$: $$$) { $$$JSX", lang: "tsx")

// component arrow
ast_grep_search(pattern: "const $NAME = ($$$: $$$) => { $$$JSX", lang: "tsx")

// export default
grep(pattern: "export default", include: "*.tsx")

// named export
grep(pattern: "export const | export function | export interface | export type", include: "*.ts")
```

## Miner D: 에러 핸들링/조건문/nullish

### 탐색 대상
| 패턴 ID | 설명 | 예시 A | 예시 B |
|---------|------|--------|--------|
| try-catch-style | 에러 처리 패턴 | `try { ... } catch(e) { console.error(e) }` | `try { ... } catch(e) { /* log */ throw e }` |
| early-return | early return vs 중첩 | `if (!x) return;` | `if (x) { if (y) { ... } }` |
| nullish-style | nullish 처리 방식 | `x?.y` | `x && x.y` | `if (x != null) { x.y }` |
| promise-style | Promise 처리 방식 | `await fn().catch(handle)` | `try { await fn() } catch ...` |
| error-class | 에러 클래스 사용 | `new Error()` | `new HttpException()` | `new AppError()` |

### Grep 패턴
```typescript
// try-catch with console.error
grep(pattern: "catch\\(\\$\\w+\\)\\s*\\{\\s*console\\.", include: "*.ts")

// early return
grep(pattern: "if \\(!\\w+\\) return", include: "*.ts")

// optional chaining
grep(pattern: "\\?\\..*\\?", include: "*.ts")  // 중첩 optional chaining

// null check patterns
grep(pattern: "!= null|!== null|!= undefined", include: "*.ts")

// Promise.catch
grep(pattern: "\\.catch\\(\\(", include: "*.ts")

// error class usage
grep(pattern: "extends Error|extends HttpException", include: "*.ts")
```

## 프레임워크별 특화 패턴 (프로젝트에 따라 추가)

### React
| 패턴 ID | 설명 | 예시 |
|---------|------|------|
| hook-naming | 커스텀 훅 네이밍 | `useXxx()` 컨벤션 준수 여부 |
| state-style | 상태 선언 방식 | `useState` vs `useReducer` |
| effect-deps | useEffect 의존성 배열 | 빈 배열, 명시적 deps 패턴 |
| callback-style | 콜백 최적화 | `useCallback` vs inline arrow |

### NestJS
| 패턴 ID | 설명 | 예시 |
|---------|------|------|
| controller-method | 컨트롤러 메서드 스타일 | `@Get()` 데코레이터 + 메서드 패턴 |
| service-injection | 서비스 주입 방식 | constructor injection vs `@Inject()` |
| dto-validation | DTO 검증 방식 | `class-validator` 데코레이터 vs 수동 검증 |
| guard-style | 가드 구현 방식 | `CanActivate` 구현 패턴 |

### React Native (Expo)
| 패턴 ID | 설명 | 예시 |
|---------|------|------|
| style-definition | 스타일 정의 방식 | `StyleSheet.create()` vs inline style |
| navigation-type | 네비게이션 타입 | `NativeStackScreenProps` vs 직접 타입 |
| glass-usage | Glass UI 패턴 | `isGlassEffectAPIAvailable()` 체크 패턴 |

## 검증 체크리스트
- [ ] 각 Miner가 자기 카테고리를 빠짐없이 탐색했는가
- [ ] 다른 Miner와 중복 탐색한 패턴이 병합되었는가
- [ ] ESLint/Prettier 영역을 침범하지 않았는가
- [ ] 변종이 1개뿐인 카테고리는 `ask_user=false`로 처리되었는가
- [ ] 모든 결과에 파일 경로 + 라인 번호가 포함되었는가
