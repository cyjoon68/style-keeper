# 스타일 변종 예시

실제 코드에서 발견되는 스타일 변종들의 before/after 예시.

## 함수 선언

### named function → arrow function

```typescript
// Before
function getTotalPrice(items: Item[]): number {
  return items.reduce((sum, item) => sum + item.price, 0);
}

// After
const getTotalPrice = (items: Item[]): number => {
  return items.reduce((sum, item) => sum + item.price, 0);
}
```

### async function → async arrow

```typescript
// Before
async function fetchUserData(userId: string): Promise<User> {
  const response = await api.get(`/users/${userId}`);
  return response.data;
}

// After
const fetchUserData = async (userId: string): Promise<User> => {
  const response = await api.get(`/users/${userId}`);
  return response.data;
}
```

### export function → export const arrow

```typescript
// Before
export function formatDate(date: Date): string {
  return date.toISOString().split('T')[0];
}

// After
export const formatDate = (date: Date): string => {
  return date.toISOString().split('T')[0];
}
```

## 타입 정의

### type (객체) → interface

```typescript
// Before
type UserProps = {
  name: string;
  email: string;
  age: number;
};

// After
interface UserProps {
  name: string;
  email: string;
  age: number;
}
```

### interface 확장 → type 교차

```typescript
// Before
type AdminUser = UserProps & {
  role: 'admin';
  permissions: string[];
};

// After
interface AdminUser extends UserProps {
  role: 'admin';
  permissions: string[];
}
```

### type import → inline import

```typescript
// Before
import type { User, Product } from './types';
import { formatDate } from './utils';

// After
import { User, Product, formatDate } from './types';
import { formatDate } from './utils';
```

> 참고: type import를 inline으로 바꾸는 것은 번들러의 type-erasure에 영향을 줄 수 있음.
> TypeScript의 `verbatimModuleSyntax` 설정과 충돌할 수 있으므로 사용자에게 먼저 확인.

## 컴포넌트 선언 (React)

### function component → arrow component

```typescript
// Before
export function UserCard({ user, onPress }: UserCardProps) {
  return (
    <View>
      <Text>{user.name}</Text>
    </View>
  );
}

// After
export const UserCard = ({ user, onPress }: UserCardProps) => {
  return (
    <View>
      <Text>{user.name}</Text>
    </View>
  );
}
```

### React.FC 제거

```typescript
// Before
const UserCard: React.FC<UserCardProps> = ({ user, onPress }) => {
  return (
    <View>
      <Text>{user.name}</Text>
    </View>
  );
};

// After
const UserCard = ({ user, onPress }: UserCardProps) => {
  return (
    <View>
      <Text>{user.name}</Text>
    </View>
  );
}
```

### 중간에 export → 하단 export

```typescript
// Before
export const useUser = (id: string) => {
  return useQuery(['user', id], () => fetchUser(id));
};

export const useUsers = () => {
  return useQuery(['users'], () => fetchUsers());
};

// After
const useUser = (id: string) => {
  return useQuery(['user', id], () => fetchUser(id));
};

const useUsers = () => {
  return useQuery(['users'], () => fetchUsers());
};

export { useUser, useUsers };
```

## 에러 핸들링

### console.error → logger

```typescript
// Before
try {
  await saveData(data);
} catch (error) {
  console.error('Failed to save:', error);
  throw error;
}

// After
try {
  await saveData(data);
} catch (error) {
  logger.error('Failed to save:', error);
  throw error;
}
```

### Promise.catch → async/await try-catch

```typescript
// Before
fetchData()
  .then(data => processData(data))
  .catch(error => handleError(error));

// After
try {
  const data = await fetchData();
  processData(data);
} catch (error) {
  handleError(error);
}
```

## 조건문

### 중첩 if → early return

```typescript
// Before
function processOrder(order: Order) {
  if (order) {
    if (order.isValid) {
      if (order.paymentStatus === 'paid') {
        // 실제 로직
        return confirmOrder(order);
      }
    }
  }
  return null;
}

// After
function processOrder(order: Order) {
  if (!order) return null;
  if (!order.isValid) return null;
  if (order.paymentStatus !== 'paid') return null;
  
  return confirmOrder(order);
}
```

## nullish 처리

### && 체이닝 → optional chaining

```typescript
// Before
const userName = user && user.profile && user.profile.name;

// After
const userName = user?.profile?.name;
```

### null 체크 → optional chaining

```typescript
// Before
if (data != null && data.items != null) {
  data.items.forEach(item => processItem(item));
}

// After
data?.items?.forEach(item => processItem(item));
```

## React Native 특화

### StyleSheet.create → unistyles (프로젝트 설정에 따라)

```typescript
// Before
const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 16,
  },
  title: {
    fontSize: 18,
    fontWeight: 'bold',
  },
});

// After (unistyles)
const stylesheet = createStyleSheet((theme) => ({
  container: {
    flex: 1,
    padding: theme.spacing.md,
  },
  title: {
    fontSize: theme.typography.size.lg,
    fontWeight: 'bold',
  },
}));
```

## 주의사항

1. **항상 dry-run 먼저**: ast-grep replace는 dryRun=true로 영향 범위 확인 필수
2. **LSP 검증**: 모든 변경 후 lsp_diagnostics로 타입 안전성 확인
3. **의미 보존**: 동작을 바꾸는 리팩토링은 스타일 통일화 범위가 아님
4. **컨텍스트 의존 변환**: 단순 패턴 매칭으로 안전하지 않은 변환(조건문 재구성 등)은 발견만 하고 사용자에게 보고
5. **프레임워크 의존 패턴**: React, NestJS 등 프레임워크별 패턴은 프로젝트 설정에 따라 적용 여부 결정
