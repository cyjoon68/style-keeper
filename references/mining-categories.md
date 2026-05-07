# 마이닝 카테고리 상세 — genp 실제 발견 사례

Style Keeper Phase 1에서 탐색하는 각 카테고리의 상세 설명과
실제 genp 코드베이스에서 발견된 패턴 현황.

---

## Miner A: 함수/메서드 선언

### genp 발견 현황

| 패턴 ID | 스타일 | count | 파일 예시 | 비고 |
|---------|-------|-------|----------|------|
| `func-declaration` | `export function useXxx()` | **25** | `apps/frontend/src/features/*/api/hooks.ts` | ✅ 주력 패턴 |
| `func-declaration` | `async function bootstrap()` | **6** | `apps/backend/*/src/main.ts` | ✅ 백엔드 표준 |
| `func-declaration` | `function buildXxx()` | **3** | `apps/backend/*/src/__tests__/*.spec.ts` | ✅ 테스트 헬퍼 |
| `arrow-const` | `const mutate = (...) => { }` | **4** | `apps/frontend/src/features/auth/api/hooks.ts` | ✅ 내부 콜백 |
| `export-const-arrow` | `export const Xxx = () => {}` | **0** | 발견되지 않음 | - |

### 주요 발견: frontend hooks — 일관된 패턴 (25개)

모든 TanStack Query hook이 `export function useXxx()` 스타일:
```typescript
export function useProducts(query?: ProductQuery) { ... }
export function useProduct(id: string) { ... }
export function useCreateProduct() { ... }
export function useUpdateProduct() { ... }
export function useDeleteProduct() { ... }
```
완전히 일관됨. 변경 불필요.

### 주요 발견: callback — arrow function 혼용

hook 내부에서 callback이 필요한 경우 `const $NAME = (...) => { }` 사용:
```typescript
export function useSignup() {
  const mutate = (variables: ..., callbacks?: ...) => {
    return apolloMutate({ ... });
  };
  return { mutate, ... };
}
```
선언(`export function`)과 할당(`const = () =>`)의 이분법적 패턴.

### ast-grep 패턴
```typescript
// named export function (frontend hooks)
ast_grep_search(pattern: "export function use$NAME($$$) { $$$ }", lang: "typescript", paths: ["apps/frontend"])

// async function bootstrap (backend main)
ast_grep_search(pattern: "async function bootstrap() { $$$ }", lang: "typescript", paths: ["apps/backend"])

// arrow function assigned to const
ast_grep_search(pattern: "const $NAME = ($$$) => { $$$ }", lang: "typescript")
```

---

## Miner B: 타입/인터페이스/import

### genp 발견 현황

| 패턴 ID | 스타일 | count | 비고 |
|---------|-------|-------|------|
| `interface-def` | `interface X { ... }` | **43** | ✅ 압도적 표준 |
| `type-object` | `type X = { ... }` | **1** | 🔍 드묾 (AppThemes) |
| `type-union` | `type X = A \| B` | 소수 | 특수 목적 |
| `type-import` | `import type { X }` | **31** | ✅ 자주 사용 |
| `inline-type` | 함수 파라미터 인라인 타입 | 가끔 | 간단한 경우 |

### interface — 43개, 압도적 표준

shared DTO, frontend type, backend type 전부 interface:
```typescript
// packages/shared/src/dtos/auth.dto.ts
export interface ISignupInput { ... }
export interface IAuthResponse { ... }
export interface IUserProfile { ... }

// apps/frontend/src/features/product/type.ts
export interface Product { ... }
export interface ProductListResponse { ... }
```

`type X = { ... }`는 단 1회 사용. interface로 통일되어 있음.

### import type — 31개 파일에서 사용

```typescript
// frontend
import type { Product, ProductListResponse } from '../type';
import type { ApolloError } from '@apollo/client';

// backend
import type { DrizzleDB } from '@genp/shared';
import type { GqlContext } from '../gql-context.interface';
```

다만 일반 `import`와 혼용되고 있어 일관화 가능 여부 확인 필요.

### ast-grep 패턴
```typescript
// interface
ast_grep_search(pattern: "interface $NAME { $$$ }", lang: "typescript")

// type alias with object
ast_grep_search(pattern: "type $NAME = { $$$ }", lang: "typescript")

// type import
grep(pattern: "import type {", include: "*.ts")
```

---

## Miner C: 컴포넌트/모듈/export

### genp 발견 현황

| 패턴 ID | 스타일 | count | 비고 |
|---------|-------|-------|------|
| `style-def` | `StyleSheet.create({})` | **44** | ✅ 표준 |
| `style-def` | `createStyleSheet((theme) => ({}))` | 소수 | unistyles 마이그레이션 |
| `export-style` | `export const styles = ...` | **44** | ✅ 스타일 export |
| `export-hook` | `export function useXxx()` | **25** | ✅ hooks export |
| `tsx-component` | component in `*.tsx` | - | 확인 필요 |

### StyleSheet.create — 44개 파일

모든 screen/component 스타일이 별도 `.styles.ts` 파일에 정의:
```typescript
// apps/frontend/src/screens/HomeScreen.styles.ts
import { StyleSheet } from 'react-native';
export const styles = StyleSheet.create({ ... });
```

`createStyleSheet` (unistyles)도 소수 사용 중 — 마이그레이션 진행 흔적.

### ast-grep 패턴
```typescript
// StyleSheet.create
ast_grep_search(pattern: "StyleSheet.create({ $$$ })", lang: "typescript")

// createStyleSheet (unistyles)
ast_grep_search(pattern: "createStyleSheet($$$)", lang: "typescript")

// component patterns in tsx
grep(pattern: "export default function", include: "*.tsx")
grep(pattern: "export const", include: "*.tsx")
```

---

## Miner D: 에러 핸들링/조건문/nullish

### genp 발견 현황

| 패턴 ID | 스타일 | count | 비고 |
|---------|-------|-------|------|
| `optional-chaining` | `data?.field` | **34** | ✅ 표준 |
| `and-chaining` | `data && data.field` | **~1** | 거의 사용 안 함 |
| `try-catch-console` | `catch(e) { console.error(e) }` | **4** | frontend |
| `try-catch-logger` | `catch(e) { this.logger.error(e) }` | **1** | backend |
| `bare-catch` | `catch { console.warn('...') }` | **1** | 에러 무시 |
| `catch-instanceof` | `catch(e) { if(e instanceof HTTPError) }` | **1** | 상세 분기 |

### Optional chaining — 34회, 압도적 표준

```typescript
// data?.field 패턴
return { data: data?.product, isLoading: loading, error };
return { data: data?.conversation, isLoading: loading, error };
callbacks?.onSuccess?.(result.signup);
```

### Try-catch 스타일 분화 (일관화 가능 영역)

**frontend:**
```typescript
// console.error 사용
catch (e) { console.error('File upload failed:', e); }

// console.warn 사용
catch { console.warn('[Auth] Logout error suppressed'); }

// instanceof 분기
catch (error) {
  if (error instanceof HTTPError) {
    // status별 처리
  }
  console.warn('[API] Network error during refresh');
}
```

**backend:**
```typescript
// NestJS Logger 사용
catch (error) { this.logger.error('Database seeding failed'); }
```

### ast-grep 패턴
```typescript
// try-catch with console
ast_grep_search(pattern: "try { $$$ } catch ($ERR) { $$$ }", lang: "typescript")

// bare catch (no variable)
grep(pattern: "catch \\{", include: "*.ts")

// optional chaining
grep(pattern: "\\?\\.\\w+", include: "*.ts")

// instanceof error check
grep(pattern: "instanceof HTTPError", include: "*.ts")
```

---

## Miner E (확장): 프레임워크별 특화 — NestJS & React

### NestJS Controller 패턴

모든 controller가 `@MessagePattern()` + `@Payload()` 사용:
```typescript
@MessagePattern(CHAT_PATTERNS.CREATE_CONVERSATION)
async createConversation(@Payload() data: CreateConversationDto) { ... }
```

### NestJS Service 패턴

service는 `@Injectable()` + constructor injection:
```typescript
@Injectable()
export class ChatService {
  constructor(
    @Inject(DRIZZLE_DB) private db: DrizzleDB,
    private configService: ConfigService,
  ) {}
}
```

이러한 패턴들은 ESLint가 잡지 못하는 프레임워크 컨벤션 레벨의 일관성 체크 대상.

---

## 발견 패턴 우선순위 (genp 기준)

| 순위 | 카테고리 | 발견된 변종 수 | ask_user | 이유 |
|------|---------|---------------|----------|------|
| 1️⃣ | 함수 선언 방식 (hooks) | **25개 모두 일관** | ❌ | 이미 완전 통일 |
| 2️⃣ | 타입 정의 (interface vs type) | **43:1 비율** | ❌ | 거의 통일됨 |
| 3️⃣ | type import 방식 | `import type` 31회 + 일반 import 혼용 | ✅ | 혼용 중 |
| 4️⃣ | optional chaining | **34:1 비율** | ❌ | 거의 통일됨 |
| 5️⃣ | 에러 핸들링 스타일 | console vs logger, bare catch 등 다양 | ✅ | 변종 많음 |
| 6️⃣ | 스타일 정의 (StyleSheet vs unistyles) | 44:소수 | ✅ | 마이그레이션 중 |
