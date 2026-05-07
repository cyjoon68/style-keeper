# 스타일 변종 예시 — genp 코드베이스 실제 사례

아래 예시들은 실제 genp 프로젝트 코드에서 발견된 스타일 변종들입니다.

---

## 1. 함수 선언 방식

### 발견: frontend hooks — `export function` (일관됨)

frontend의 모든 TanStack Query hook은 `export function useXxx()` 스타일로 통일되어 있습니다.
```typescript
// apps/frontend/src/features/product/api/hooks.ts
export function useProducts(query?: ProductQuery) {
  const { data, loading, error } = useQuery<
    { products: ProductListResponse },
    { query?: ProductQuery }
  >(PRODUCTS_QUERY, { variables: { query } });
  return { data: data?.products, isLoading: loading, error, isError: !!error };
}

export function useProduct(id: string) {
  const { data, loading, error } = useQuery<{ product: Product }, { id: string }>(
    PRODUCT_QUERY, { variables: { id }, skip: !id }
  );
  return { data: data?.product, isLoading: loading, error, isError: !!error };
}
```

### 발견: backend services — `async function` (일관됨)

모든 backend 마이크로서비스의 bootstrap 함수가 동일한 패턴:
```typescript
// apps/backend/auth-service/src/main.ts
async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    { transport: Transport.TCP, options: { host: '0.0.0.0', port: SERVICE_PORTS.AUTH_SERVICE } },
  );
  const logger = new Logger('Bootstrap');
  await app.listen();
  logger.log(`Auth Service running on TCP port ${SERVICE_PORTS.AUTH_SERVICE}`);
  // Graceful shutdown...
}
```

### 발견: callback 함수 — `const mutate = (...)` (arrow)

hook 내부의 callback은 `const mutate = (...) => { ... }` arrow function 사용:
```typescript
// apps/frontend/src/features/auth/api/hooks.ts
const mutate = (
  variables: { email: string; password: string; name: string },
  callbacks?: { onSuccess?: (data: AuthResponse) => void; onError?: (err: Error) => void },
) => {
  return apolloMutate({
    variables: { input: variables },
    onCompleted: (result: { signup: AuthResponse }) => {
      callbacks?.onSuccess?.(result.signup);
    },
    onError: callbacks?.onError,
  });
};
```

### 패턴 요약

| 패턴 | 위치 | count | 일관성 |
|------|------|-------|--------|
| `export function useXxx()` | frontend hooks | 25 | ✅ 완전 일관 |
| `async function bootstrap()` | backend main.ts | 6 | ✅ 완전 일관 |
| `function buildXxx()` | test helpers | 3 | ✅ 일관 |
| `const $NAME = (...) => { ... }` | callback 변수 | 4 | ✅ 일관 |
| `export const useXxx = () => {}` | 발견되지 않음 | 0 | - |

> **결론**: 함수 선언 스타일은 이미 상당히 일관됨.
> 다만 `const mutate = (...) => {` 스타일이 일부 내부 콜백에 사용되고
> `export function`이 메인 훅 선언에 사용되는 패턴 분화가 있음.

---

## 2. 타입 정의 방식

### 발견: interface 압도적 우세

shared DTO와 frontend type 모두 `interface` 사용:
```typescript
// packages/shared/src/dtos/auth.dto.ts
export interface ISignupInput {
  email: string & tags.Format<"email">;
  password: string & tags.MinLength<8>;
  name: string & tags.MinLength<1>;
}

export interface IAuthResponse {
  id: string;
  email: string;
  name: string;
  accessToken: string;
  refreshToken?: string;
}

export interface IUserProfile {
  id: string;
  email: string;
  name: string;
  avatarUrl?: string;
  createdAt: string;
  points: number;
  couponsCount: number;
  membershipGrade: string;
}
```

```typescript
// apps/frontend/src/features/product/type.ts
export interface Product {
  id: string;
  name: string;
  price: number;
  brand: string;
  imageUrl?: string;
  description?: string;
  userId: string;
  createdAt: string;
  updatedAt: string;
}

export interface ProductListResponse {
  items: Product[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}
```

### 발견: type alias (드물게 사용)

`type`은 주로 유니온/교차 타입이나 간단한 매핑에만 사용:
```typescript
// apps/frontend/src/theme/unistyles.ts
type AppThemes = {
  light: typeof lightTheme;
  dark: typeof darkTheme;
};
```

```typescript
// apps/backend/ai-service/src/ai.service.ts
interface GenerationRecord {  // ← interface (not type)
  id: string;
  conversationId: string;
  status: 'pending' | 'processing' | 'completed' | 'failed';
  // ...
}
```

### 패턴 요약

| 패턴 | count | 비율 |
|------|-------|------|
| `interface X { ... }` | 43 | ~98% |
| `type X = { ... }` | 1 | ~2% |
| `type X = A \| B` (유니온) | 소수 | 특수 케이스 |

> **결론**: interface가 압도적. type alias는 유니온/유틸리티 타입에만 사용.
> 이미 잘 통일되어 있음.

---

## 3. Type import 방식

### 발견: `import type { X }` — 31회 사용

```typescript
// apps/backend/product-service/src/product.service.ts
import type { DrizzleDB } from '@genp/shared';
import type { ICreateProductInput, IUpdateProductInput } from '@genp/shared';

// apps/backend/api-gateway/src/graphql/resolvers/chat.resolver.ts
import type { GqlContext } from '../gql-context.interface';

// apps/frontend/src/features/product/api/hooks.ts
import type { Product, ProductListResponse, ProductQuery } from '../type';
import type { ApolloError } from '@apollo/client';
```

31개 파일에서 `import type` 사용. 상당히 널리 퍼져있지만 모든 import가 type인 것은 아님.

### 발견: 일반 `import`도 혼용

```typescript
// 같은 파일에서 일반 import와 type import 혼용
import { useQuery, useMutation, useSubscription } from '@apollo/client';  // values
import type { ApolloError } from '@apollo/client';  // type only
```

### 패턴 요약

| 패턴 | count |
|------|-------|
| `import type { X } from '...'` | 31 |
| `import { X } from '...'` (type 포함) | 다수 |

> **결론**: `import type`과 일반 import가 혼용됨.
> TypeScript `verbatimModuleSyntax` 설정에 따라 일관화 가능한 영역.

---

## 4. 옵셔널 체이닝 스타일

### 발견: optional chaining `?.` — 압도적 우세

```typescript
// apps/frontend/src/features/product/api/hooks.ts
return { data: data?.product, isLoading: loading, error, isError: !!error };

// apps/frontend/src/features/chat/api/hooks.ts
const base = data?.messages ?? [];

// apps/frontend/src/features/auth/api/hooks.ts
callbacks?.onSuccess?.(result.signup);
```

optional chaining이 34회 사용됨. nullish 접근의 표준 패턴.

### 발견: `&&` chaining — 거의 사용 안 함

```typescript
// 유일한 사례: useFileUpload.ts
if (asset.base64) {  // if + property access, not &&
  // ...
}
```

`&&` 체이닝은 거의 발견되지 않음. 대부분 `?.`로 대체됨.

### 패턴 요약

| 패턴 | count |
|------|-------|
| `data?.field` (optional chaining) | 34 |
| `data && data.field` (&& chaining) | ~0 |
| `if (data != null)` (명시적 null check) | ~0 |

> **결론**: 이미 optional chaining으로 거의 완전히 통일됨.

---

## 7. React 컴포넌트 선언 — genp 실제 사례

### 발견: 100% `function` 키워드, arrow component 0개

genp frontend의 모든 React 컴포넌트는 `function` 키워드:

```typescript
// app/(tabs)/chat.tsx — Route: export default function
export default function ChatRoute() { return <ChatScreen />; }

// src/screens/HomeScreen.tsx — Screen: export function
export function HomeScreen() { /* ... */ }

// src/features/chat/ui/ChatFooter.tsx — UI: export function
export function ChatFooter({ inputValue, onSend }: ChatFooterProps) { /* ... */ }
```

Arrow function으로 선언된 컴포넌트는 단 1건도 없음. `const Comp = () => {}` 패턴은 존재하지 않음.

### 발견: Route → Screen re-export 패턴

```typescript
// app/notifications.tsx
export default NotificationsScreen;  // 단순 re-export

// app/settings.tsx
export default SettingsScreen;
```

6개 route 파일에서 이 패턴 사용.

### 발견: 100% 시그니처 destructure

```typescript
// ✅ 표준
export function ChatFooter({ inputValue, onInputChange, onSend }: ChatFooterProps) { ... }
// ❌ 발견 안 됨: function ChatFooter(props: Props) { const { ... } = props; }
```

### 패턴 요약

| 패턴 | count | 일관성 |
|------|-------|--------|
| `export function Comp()` | ~31 (screens/ui) | ✅ |
| `export default function Comp()` | ~25 (routes) | ✅ |
| `function Helper()` (file-private) | ~8 | ✅ |
| `const Comp = () => {}` | 0 | - |
| `React.memo()` | 0 | - |
| 시그니처 destructure | 100% | ✅ |

---

## 8. Guard Clause & Early Return — genp 실제 사례

### Guard: `if (!x) { throw ... }` — 38건, 13개 파일

```typescript
// auth.service.ts
if (!isValid) { throw new UnauthorizedException('Invalid credentials'); }
if (!row) { throw new NotFoundException('Product not found'); }
if (!user) { throw new NotFoundException('User not found'); }
```

### Early return: `if (!x) return` — 64건, 18개 파일

```typescript
// frontend validation
if (!val.trim()) return '이메일을 입력해주세요';
if (error) { setPhoneError(error); return; }

// ai-service
const r = this.generations.get(id);
if (!r || r.status === 'failed') return;
```

### 발견: `!= null` 체크는 0건

모든 nullish 체크는 `!x` (falsy) 또는 `?.` (optional chaining)로 처리. 명시적 `!= null`은 전혀 없음.

### 패턴 요약

| 패턴 | count | 비고 |
|------|-------|------|
| `if (!x) { throw ... }` | 38건 (13개 파일) | NestJS guard 표준 |
| `if (!x) return` | 64건 (18개 파일) | frontend/backend 공통 |
| `if (x != null)` | 0건 | 전혀 사용 안 됨 |

---

## 5. 에러 핸들링 스타일

### 발견: try-catch + console.error/warn

```typescript
// apps/frontend/src/lib/apiClient.ts — refresh token 로직
try {
  const res = await ky.post(`${API_BASE_URL}/auth/refresh`, {
    json: { refreshToken },
    headers: { 'Content-Type': 'application/json' },
  }).json<AuthResponse>();
  useAuthStore.getState().setAuth(res.id, res.accessToken, res.refreshToken);
  return true;
} catch (error) {
  if (error instanceof HTTPError) {
    const status = error.response.status;
    if (status === 401) {
      useAuthStore.getState().clearAuth();
      notifyRefreshFailed('expired');
      return false;
    }
    if (status === 429) {
      console.warn('[API] Refresh rate limited (429), keeping current tokens');
      return false;
    }
    console.warn(`[API] Refresh failed with status ${status}, keeping current tokens`);
    return false;
  }
  console.warn('[API] Network error during refresh, keeping current tokens');
  return false;
}
```

### 발견: bare catch (변수 없이)

```typescript
// apps/frontend/src/features/auth/api/hooks.ts — logout
try {
  await apolloClient.mutate({
    mutation: LOGOUT_MUTATION,
    variables: { input: { refreshToken } },
  });
} catch {
  console.warn('[Auth] Logout error suppressed');  // bare catch, 변수 없음
}
```

### 발견: catch + console.error (단순)

```typescript
// apps/frontend/src/features/file/hooks/useFileUpload.ts
try {
  const res = await uploadFile({ ... });
  // ...
} catch (e) {
  console.error('File upload failed:', e);
}
```

### 발견: backend logger 사용

```typescript
// apps/backend/chat-service/src/chat.service.ts
try {
  await this.db.transaction(async (tx) => {
    await this.seedConversation(tx);
    await this.seedMessages(tx);
  });
} catch (error) {
  this.logger.error('Database seeding failed');
}
```

### 패턴 요약

| 패턴 | 발견 위치 | 특징 |
|------|----------|------|
| `catch (error) { if (error instanceof HTTPError) ... }` | apiClient.ts | 상세 에러 분기 |
| `catch { console.warn('...') }` (bare catch) | hooks.ts | 에러 무시 |
| `catch (e) { console.error('...', e) }` | useFileUpload.ts | 단순 로깅 |
| `catch (error) { this.logger.error('...') }` | chat.service.ts | NestJS Logger |

> **결론**: 에러 핸들링 스타일에 차이가 있음.
> - frontend: `console.error`/`console.warn` 사용
> - backend: `this.logger.error` 사용
> - bare catch vs 변수 있는 catch 혼용

---

## 6. 스타일 정의 방식

### 발견: `StyleSheet.create()` — 44개 파일

```typescript
// apps/frontend/src/screens/HomeScreen.styles.ts
import { StyleSheet } from 'react-native';

export const styles = StyleSheet.create({
  container: { flex: 1, padding: 16 },
  // ...
});
```

### 발견: `createStyleSheet()` — unistyles 패턴

```typescript
// apps/frontend/src/screens/SignupScreen.styles.ts
import { createStyleSheet } from '@/theme';

export const stylesheet = createStyleSheet((theme) => ({
  container: {
    flex: 1,
    padding: theme.spacing.md,
  },
  // ...
}));
```

### 패턴 요약

| 패턴 | count |
|------|-------|
| `StyleSheet.create({})` | 44 |  
| `createStyleSheet((theme) => ({}))` | 소수 (신규 migration) |

> **결론**: `StyleSheet.create()`가 표준. `createStyleSheet`는 unistyles 마이그레이션 진행 중.
> 이건 ESLint로 잡을 수 없는 "프레임워크 패턴 전환" 영역.
