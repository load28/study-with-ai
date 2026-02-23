# 4주차: 합성과 파이프라인 — 작은 함수를 조립하는 기술

## 왜 이것을 배우는가?

1주차에서 순수 함수를, 2주차에서 일급 함수와 고차 함수를, 3주차에서 불변성을 배웠다. 이 세 가지를 결합하면 강력한 원칙이 탄생한다: **작고 순수한 함수를 만들고, 이들을 조립하여 복잡한 동작을 만든다.**

```typescript
// 이런 코드를 작성하고 있지 않은가?
function processUser(raw: unknown): string {
  const parsed = JSON.parse(raw as string);
  const validated = validateUser(parsed);
  const normalized = normalizeEmail(validated);
  const formatted = formatDisplayName(normalized);
  return formatted;
}

// 매번 중간 변수를 만들고, 순서를 수동으로 관리한다.
// 만약 단계를 추가하거나 순서를 바꿔야 한다면?
// 만약 조건에 따라 다른 파이프라인을 써야 한다면?
```

함수 합성(Composition)은 이 문제를 해결한다. 함수를 **레고 블록**처럼 조립하는 것이다. 각 블록(함수)은 독립적이고, 조립 방법(합성)은 수학적으로 보장된다.

이번 주차는 Phase 1의 마지막이다. 순수 함수 + 일급 함수 + 불변성 + **합성** — 이 네 가지가 함수형 프로그래밍의 사고 체계를 완성한다.

---

## 1. 함수 합성의 수학적 정의

### 1.1 합성이란

수학에서 함수 합성은 두 함수를 연결하여 새 함수를 만드는 연산이다.

```
f: B → C
g: A → B

f ∘ g: A → C
(f ∘ g)(x) = f(g(x))
```

**g가 먼저** 실행되고 그 결과가 f에 전달된다. 읽는 순서(f ∘ g)와 실행 순서(g → f)가 반대인 점에 주의하라.

```typescript
// 수학적 합성을 TypeScript로
function compose<A, B, C>(
  f: (b: B) => C,
  g: (a: A) => B
): (a: A) => C {
  return (a: A) => f(g(a));
}

// 예시
const double = (x: number): number => x * 2;
const addOne = (x: number): number => x + 1;

const doubleAndAddOne = compose(addOne, double);
// 수학: (addOne ∘ double)(x) = addOne(double(x))

console.log(doubleAndAddOne(3));  // double(3) = 6, addOne(6) = 7
console.log(doubleAndAddOne(5));  // double(5) = 10, addOne(10) = 11
```

### 1.2 합성의 전제 조건: 타입이 맞아야 한다

합성이 가능하려면 **g의 출력 타입이 f의 입력 타입과 일치**해야 한다.

```typescript
// ✅ 합성 가능: number → string → number
const toString = (n: number): string => n.toString();
const length = (s: string): number => s.length;

const digitCount = compose(length, toString);
console.log(digitCount(123));   // "123".length = 3
console.log(digitCount(10000)); // "10000".length = 5

// ❌ 합성 불가능: 타입이 맞지 않음
const toUpper = (s: string): string => s.toUpperCase();
// compose(double, toUpper); // 타입 에러! toUpper의 출력은 string, double의 입력은 number
```

이것이 타입 시스템의 힘이다. **합성할 수 없는 함수를 합성하면 컴파일 타임에 잡힌다.**

### 1.3 항등 함수 (Identity Function)

항등 함수는 합성의 **항등원**이다. 어떤 함수와 합성해도 원래 함수가 된다.

```typescript
// 항등 함수
const identity = <A>(a: A): A => a;

// 항등 법칙
// compose(f, identity) === f    (우항등)
// compose(identity, f) === f    (좌항등)

const double = (x: number): number => x * 2;

const left = compose(identity, double);
const right = compose(double, identity);

console.log(left(5));   // 10
console.log(right(5));  // 10
console.log(double(5)); // 10
// 셋 다 같은 결과
```

"아무것도 안 하는 함수가 왜 필요한가?"라고 생각할 수 있다. 항등 함수는 합성 체계를 완전하게 만든다. 수학에서 0이 덧셈의 항등원이고 1이 곱셈의 항등원인 것과 같다. 나중에 카테고리 이론(31주차)에서 이것이 얼마나 중요한지 보게 될 것이다.

---

## 2. 합성의 결합법칙 (Associativity)

### 2.1 결합법칙이란

세 함수 f, g, h가 있을 때, 합성의 **괄호 위치가 결과에 영향을 주지 않는다**.

```
(f ∘ g) ∘ h === f ∘ (g ∘ h)
```

```typescript
const addOne = (x: number): number => x + 1;
const double = (x: number): number => x * 2;
const square = (x: number): number => x * x;

// 두 가지 괄호 방식
const way1 = compose(compose(addOne, double), square);
// (addOne ∘ double) ∘ square

const way2 = compose(addOne, compose(double, square));
// addOne ∘ (double ∘ square)

// 둘 다 같은 결과
console.log(way1(3)); // square(3)=9, double(9)=18, addOne(18)=19
console.log(way2(3)); // square(3)=9, double(9)=18, addOne(18)=19
```

### 2.2 결합법칙이 중요한 이유

결합법칙 덕분에 **합성을 어떤 순서로 조립해도 결과가 같다**. 이것은 리팩토링의 수학적 근거가 된다.

```typescript
// 데이터 변환 파이프라인
const normalize = (s: string): string => s.trim().toLowerCase();
const removeSpecial = (s: string): string => s.replace(/[^a-z0-9]/g, "");
const truncate = (s: string): string => s.slice(0, 20);

// 결합법칙 덕분에 어떻게 그룹핑해도 같은 결과
const pipeline1 = compose(compose(truncate, removeSpecial), normalize);
const pipeline2 = compose(truncate, compose(removeSpecial, normalize));

// 따라서 부분 파이프라인을 자유롭게 추출할 수 있다
const sanitize = compose(removeSpecial, normalize);
const pipeline3 = compose(truncate, sanitize);

const input = "  Hello, World! @#$ ";
console.log(pipeline1(input)); // "helloworld"
console.log(pipeline2(input)); // "helloworld"
console.log(pipeline3(input)); // "helloworld"
```

결합법칙이 없다면 합성 순서를 바꿀 때마다 결과가 달라질 수 있고, 리팩토링이 불가능해진다.

### 2.3 카테고리의 조건

지금까지 세 가지를 배웠다:
1. **합성**: 두 함수를 연결하여 새 함수를 만든다
2. **항등 함수**: 합성의 항등원
3. **결합법칙**: 합성 순서가 결과에 영향을 주지 않는다

이 세 가지는 **카테고리(Category)** 의 정의 조건이다. TypeScript의 타입과 함수는 카테고리를 형성한다. 31주차에서 이것을 본격적으로 다룬다.

---

## 3. compose 구현

### 3.1 두 함수 합성

```typescript
// 가장 기본적인 compose
function compose2<A, B, C>(
  f: (b: B) => C,
  g: (a: A) => B
): (a: A) => C {
  return (a) => f(g(a));
}
```

### 3.2 다중 함수 합성 — 오버로드

실전에서는 두 개 이상의 함수를 합성해야 한다. TypeScript의 오버로드로 타입 안전한 다중 합성을 만들 수 있다.

```typescript
// 오버로드 시그니처
function compose<A, B>(f: (a: A) => B): (a: A) => B;
function compose<A, B, C>(
  f: (b: B) => C,
  g: (a: A) => B
): (a: A) => C;
function compose<A, B, C, D>(
  f: (c: C) => D,
  g: (b: B) => C,
  h: (a: A) => B
): (a: A) => D;
function compose<A, B, C, D, E>(
  f: (d: D) => E,
  g: (c: C) => D,
  h: (b: B) => C,
  i: (a: A) => B
): (a: A) => E;

// 구현
function compose(...fns: Function[]): Function {
  return (x: unknown) => fns.reduceRight((acc, fn) => fn(acc), x);
}

// 사용
const process = compose(
  (s: string) => s.toUpperCase(),    // 4. 대문자로
  (s: string) => s.trim(),           // 3. 공백 제거
  (n: number) => n.toFixed(2),       // 2. 소수점 2자리
  (n: number) => n * 1.1             // 1. 10% 인상
);

console.log(process(100)); // "110.00" → "110.00" → "110.00"
```

### 3.3 reduceRight의 의미

`compose`의 구현에서 `reduceRight`를 쓴 이유는 **오른쪽에서 왼쪽으로** 실행하기 때문이다.

```typescript
// compose(f, g, h)(x) === f(g(h(x)))
// reduceRight: [f, g, h] → h(x), g(h(x)), f(g(h(x)))

function composeExplained(...fns: Function[]) {
  return (x: unknown) => {
    let result = x;
    // 마지막 함수부터 역순으로 실행
    for (let i = fns.length - 1; i >= 0; i--) {
      result = fns[i](result);
    }
    return result;
  };
}
```

---

## 4. pipe와 flow

### 4.1 compose의 문제: 읽기 방향

`compose`는 수학적으로 자연스럽지만, 코드의 **읽기 방향과 반대**다.

```typescript
// compose: 오른쪽에서 왼쪽 (수학적 표기)
const result = compose(toUpper, trim, addPrefix("Hello, "))(name);
// 실행 순서: addPrefix → trim → toUpper
// 읽는 순서: toUpper ← trim ← addPrefix  ← 반대!
```

프로그래머는 보통 **왼쪽에서 오른쪽, 위에서 아래로** 코드를 읽는다. `pipe`와 `flow`는 이 문제를 해결한다.

### 4.2 pipe: 값부터 시작하는 좌→우 합성

`pipe`는 **초기값**을 첫 인자로 받고, 함수들을 순서대로 적용한다.

```typescript
// pipe 구현 (오버로드 생략, 핵심만)
function pipe<A>(a: A): A;
function pipe<A, B>(a: A, f: (a: A) => B): B;
function pipe<A, B, C>(a: A, f: (a: A) => B, g: (b: B) => C): C;
function pipe<A, B, C, D>(
  a: A, f: (a: A) => B, g: (b: B) => C, h: (c: C) => D
): D;
function pipe<A, B, C, D, E>(
  a: A, f: (a: A) => B, g: (b: B) => C, h: (c: C) => D, i: (d: D) => E
): E;

function pipe(a: unknown, ...fns: Function[]): unknown {
  return fns.reduce((acc, fn) => fn(acc), a);
}

// 사용: 왼쪽에서 오른쪽으로 읽기 자연스러움!
const result = pipe(
  "  hello, world  ",          // 시작값
  (s: string) => s.trim(),     // 1. 공백 제거
  (s: string) => s.split(", "),// 2. 쉼표로 분리
  (arr: string[]) => arr.map(s => s[0].toUpperCase() + s.slice(1)), // 3. 첫 글자 대문자
  (arr: string[]) => arr.join(" ")  // 4. 합치기
);

console.log(result); // "Hello World"
```

### 4.3 flow: 값 없이 함수만 합성 (좌→우)

`flow`는 `compose`와 같이 새 함수를 만들지만, **왼쪽에서 오른쪽** 순서다.

```typescript
// flow 구현
function flow<A, B>(f: (a: A) => B): (a: A) => B;
function flow<A, B, C>(
  f: (a: A) => B, g: (b: B) => C
): (a: A) => C;
function flow<A, B, C, D>(
  f: (a: A) => B, g: (b: B) => C, h: (c: C) => D
): (a: A) => D;
function flow<A, B, C, D, E>(
  f: (a: A) => B, g: (b: B) => C, h: (c: C) => D, i: (d: D) => E
): (a: A) => E;

function flow(...fns: Function[]): Function {
  return (x: unknown) => fns.reduce((acc, fn) => fn(acc), x);
}

// 사용
const processName = flow(
  (s: string) => s.trim(),
  (s: string) => s.toLowerCase(),
  (s: string) => s.replace(/\s+/g, "-")
);

// processName은 새 함수 — 아직 실행되지 않음
console.log(processName("  Hello World  ")); // "hello-world"
console.log(processName("  Foo Bar  "));     // "foo-bar"
```

### 4.4 pipe vs flow vs compose 비교

```typescript
const trim = (s: string): string => s.trim();
const lower = (s: string): string => s.toLowerCase();
const slug = (s: string): string => s.replace(/\s+/g, "-");

// compose: 오른쪽 → 왼쪽, 함수만 합성
const toSlug1 = compose(slug, lower, trim);
// 읽기: slug ← lower ← trim

// flow: 왼쪽 → 오른쪽, 함수만 합성
const toSlug2 = flow(trim, lower, slug);
// 읽기: trim → lower → slug

// pipe: 왼쪽 → 오른쪽, 값을 즉시 적용
const result = pipe("  Hello World  ", trim, lower, slug);
// 읽기: "  Hello World  " → trim → lower → slug

console.log(toSlug1("  Hello World  ")); // "hello-world"
console.log(toSlug2("  Hello World  ")); // "hello-world"
console.log(result);                     // "hello-world"
```

```
┌─────────────────────────────────────────────────────────────────┐
│                   pipe vs flow vs compose                       │
├────────────┬──────────────┬─────────────┬───────────────────────┤
│            │ compose      │ flow        │ pipe                  │
├────────────┼──────────────┼─────────────┼───────────────────────┤
│ 방향       │ 오른쪽→왼쪽  │ 왼쪽→오른쪽 │ 왼쪽→오른쪽           │
│ 첫 인자    │ 마지막 함수  │ 첫 함수     │ 초기값                │
│ 반환       │ 새 함수      │ 새 함수     │ 최종 값               │
│ 사용 시점  │ 함수 정의 시 │ 함수 정의 시│ 값 변환 시            │
│ 수학 표기  │ f ∘ g        │ g ; f       │ x |> f |> g           │
│ 가독성     │ 수학에 익숙  │ 코드에 익숙 │ 가장 직관적           │
└────────────┴──────────────┴─────────────┴───────────────────────┘
```

### 4.5 왜 pipe가 실전에서 더 많이 쓰이는가

```typescript
// pipe는 TypeScript에서 두 가지 실용적 이점이 있다

// 1. 타입 추론이 잘 된다 — 초기값에서 타입이 시작되므로
const result = pipe(
  [1, 2, 3, 4, 5],
  // arr은 number[]로 추론됨
  (arr) => arr.filter(n => n % 2 === 0),
  // arr은 number[]로 추론됨
  (arr) => arr.map(n => n * 10),
  // arr은 number[]로 추론됨
  (arr) => arr.reduce((sum, n) => sum + n, 0)
);
// result: number

// 2. 중간 디버깅이 쉽다
const debug = <T>(label: string) => (value: T): T => {
  console.log(`[${label}]`, value);
  return value;
};

const debugResult = pipe(
  [1, 2, 3, 4, 5],
  debug("입력"),                          // [입력] [1, 2, 3, 4, 5]
  (arr: number[]) => arr.filter(n => n % 2 === 0),
  debug("짝수 필터 후"),                   // [짝수 필터 후] [2, 4]
  (arr: number[]) => arr.map(n => n * 10),
  debug("10배 후"),                        // [10배 후] [20, 40]
  (arr: number[]) => arr.reduce((sum, n) => sum + n, 0),
  debug("합계")                            // [합계] 60
);
```

---

## 5. Point-free 스타일

### 5.1 정의

Point-free 스타일은 **인자를 명시하지 않고** 함수를 정의하는 방식이다. "point"는 수학에서 값(점)을 의미한다.

```typescript
// Pointed 스타일: 인자 x를 명시
const doubleAll = (numbers: number[]): number[] =>
  numbers.map(n => n * 2);

// Point-free 스타일: 인자를 명시하지 않음
const double = (n: number): number => n * 2;
const doubleAllPF = (numbers: number[]): number[] =>
  numbers.map(double);
```

### 5.2 Point-free를 가능하게 하는 도구들

Point-free 스타일을 쓰려면 **커링(2주차)**과 **합성**이 필요하다.

```typescript
// 커링된 유틸리티 함수들
const map = <A, B>(f: (a: A) => B) => (arr: readonly A[]): B[] =>
  arr.map(f);

const filter = <A>(pred: (a: A) => boolean) => (arr: readonly A[]): A[] =>
  arr.filter(pred);

const reduce = <A, B>(f: (acc: B, a: A) => B, init: B) => (arr: readonly A[]): B =>
  arr.reduce(f, init);

const prop = <K extends string>(key: K) =>
  <T extends Record<K, unknown>>(obj: T): T[K] => obj[key];

// Point-free 파이프라인
const isEven = (n: number): boolean => n % 2 === 0;
const double = (n: number): number => n * 2;
const sum = (a: number, b: number): number => a + b;

// Pointed
const processNumbersPointed = (numbers: number[]): number => {
  const evens = numbers.filter(n => n % 2 === 0);
  const doubled = evens.map(n => n * 2);
  return doubled.reduce((a, b) => a + b, 0);
};

// Point-free
const processNumbers = flow(
  filter(isEven),
  map(double),
  reduce(sum, 0)
);

console.log(processNumbers([1, 2, 3, 4, 5])); // 2*2 + 4*2 = 4 + 8 = 12
```

### 5.3 Point-free의 장단점

```typescript
// ✅ Point-free가 좋은 경우: 의도가 명확해질 때
const getNames = map(prop("name"));
const getActiveUsers = filter(prop("isActive"));
const formatEmails = map(flow(prop("email"), toLowerCase, trim));

// ❌ Point-free가 나쁜 경우: 오히려 읽기 어려울 때
// 과도한 point-free
const mystery = flow(
  filter(complement(either(propEq("status", "deleted"), propEq("role", "bot")))),
  sortBy(descend(prop("createdAt"))),
  take(10),
  map(pick(["id", "name", "email"]))
);

// 같은 로직, 적절한 명시적 인자
const getRecentRealUsers = (users: User[]) =>
  pipe(
    users,
    filter(u => u.status !== "deleted" && u.role !== "bot"),
    sortBy(u => -u.createdAt),
    take(10),
    map(u => ({ id: u.id, name: u.name, email: u.email }))
  );
```

**원칙**: Point-free 스타일이 **가독성을 높일 때만** 사용한다. 함수 이름이 충분히 설명적이고, 합성이 자연스러우면 Point-free가 좋다. 복잡한 변환이 여러 겹으로 중첩되면 인자를 명시하는 것이 낫다.

### 5.4 TypeScript에서의 Point-free 한계

TypeScript의 타입 추론은 Point-free 스타일에서 제한적이다.

```typescript
// ❌ 타입 추론 실패 — TypeScript의 한계
const getNames = map(prop("name"));
// 타입을 추론할 수 없다: prop("name")의 반환 타입이 모호

// ✅ 타입 어노테이션 필요
const getNames = map((user: User) => user.name);
// 또는
const getName = (user: User): string => user.name;
const getNames = map(getName);
```

Haskell 같은 언어에서는 전역 타입 추론(Hindley-Milner)이 Point-free를 완벽히 지원한다. TypeScript에서는 타입이 모호해질 때 명시적 인자를 사용하는 것이 실용적이다.

---

## 6. 실전 패턴: 데이터 변환 파이프라인

### 6.1 API 응답 정규화

백엔드에서 받은 데이터를 프론트엔드에서 쓸 형태로 변환하는 것은 가장 흔한 파이프라인이다.

```typescript
// 백엔드 응답 타입
interface ApiUser {
  user_id: number;
  first_name: string;
  last_name: string;
  email_address: string;
  is_active: boolean;
  created_at: string;
  profile_image_url: string | null;
}

// 프론트엔드 모델
interface User {
  readonly id: number;
  readonly displayName: string;
  readonly email: string;
  readonly isActive: boolean;
  readonly createdAt: Date;
  readonly avatarUrl: string;
}

// 각 변환 단계를 순수 함수로 분리
const snakeToCamel = (user: ApiUser) => ({
  id: user.user_id,
  firstName: user.first_name,
  lastName: user.last_name,
  email: user.email_address,
  isActive: user.is_active,
  createdAt: user.created_at,
  profileImageUrl: user.profile_image_url,
});

const buildDisplayName = <T extends { firstName: string; lastName: string }>(
  user: T
) => ({
  ...user,
  displayName: `${user.firstName} ${user.lastName}`,
});

const parseDate = <T extends { createdAt: string }>(user: T) => ({
  ...user,
  createdAt: new Date(user.createdAt),
});

const defaultAvatar = <T extends { profileImageUrl: string | null }>(user: T) => ({
  ...user,
  avatarUrl: user.profileImageUrl ?? "/default-avatar.png",
});

const pickUserFields = (user: {
  id: number;
  displayName: string;
  email: string;
  isActive: boolean;
  createdAt: Date;
  avatarUrl: string;
}): User => ({
  id: user.id,
  displayName: user.displayName,
  email: user.email,
  isActive: user.isActive,
  createdAt: user.createdAt,
  avatarUrl: user.avatarUrl,
});

// 파이프라인 합성
const normalizeUser = flow(
  snakeToCamel,
  buildDisplayName,
  parseDate,
  defaultAvatar,
  pickUserFields
);

// API 응답 전체를 변환
const normalizeUsers = (apiUsers: ApiUser[]): User[] =>
  apiUsers.map(normalizeUser);

// 사용
const apiResponse: ApiUser[] = [
  {
    user_id: 1,
    first_name: "Alice",
    last_name: "Kim",
    email_address: "alice@example.com",
    is_active: true,
    created_at: "2024-01-15T09:00:00Z",
    profile_image_url: null,
  },
];

const users = normalizeUsers(apiResponse);
// [{ id: 1, displayName: "Alice Kim", email: "alice@example.com", ... }]
```

이 접근의 장점:
- 각 변환 단계는 **독립적으로 테스트** 가능
- 단계를 **추가/제거/순서 변경**이 용이
- 각 함수의 **관심사가 분리**됨

### 6.2 조건부 파이프라인

때로는 조건에 따라 다른 변환을 적용해야 한다.

```typescript
// 조건부로 함수를 적용하는 유틸리티
const when = <T>(predicate: (value: T) => boolean, transform: (value: T) => T) =>
  (value: T): T =>
    predicate(value) ? transform(value) : value;

const unless = <T>(predicate: (value: T) => boolean, transform: (value: T) => T) =>
  when((v: T) => !predicate(v), transform);

// 사용
interface Product {
  readonly name: string;
  readonly price: number;
  readonly category: string;
  readonly discount: number;
}

const applyDiscount = (product: Product): Product => ({
  ...product,
  price: product.price * (1 - product.discount),
});

const addTax = (rate: number) => (product: Product): Product => ({
  ...product,
  price: product.price * (1 + rate),
});

const roundPrice = (product: Product): Product => ({
  ...product,
  price: Math.round(product.price * 100) / 100,
});

const isDiscounted = (p: Product): boolean => p.discount > 0;
const isLuxury = (p: Product): boolean => p.category === "luxury";

// 조건부 파이프라인
const calculateFinalPrice = flow(
  when(isDiscounted, applyDiscount),   // 할인이 있으면 적용
  when(isLuxury, addTax(0.2)),         // 럭셔리면 세금 20%
  unless(isLuxury, addTax(0.1)),       // 아니면 세금 10%
  roundPrice
);

console.log(calculateFinalPrice({
  name: "시계", price: 1000, category: "luxury", discount: 0.1
}));
// 1000 * 0.9 = 900, 900 * 1.2 = 1080
```

### 6.3 배열 파이프라인

```typescript
// 커링된 배열 유틸리티
const map = <A, B>(f: (a: A) => B) =>
  (arr: readonly A[]): B[] => arr.map(f);

const filter = <A>(pred: (a: A) => boolean) =>
  (arr: readonly A[]): A[] => arr.filter(pred);

const sort = <A>(compare: (a: A, b: A) => number) =>
  (arr: readonly A[]): A[] => [...arr].sort(compare);

const take = (n: number) =>
  <A>(arr: readonly A[]): A[] => arr.slice(0, n);

const groupBy = <A>(keyFn: (a: A) => string) =>
  (arr: readonly A[]): Record<string, A[]> =>
    arr.reduce<Record<string, A[]>>((groups, item) => {
      const key = keyFn(item);
      return { ...groups, [key]: [...(groups[key] ?? []), item] };
    }, {});

// 실전 예시: 대시보드 데이터 변환
interface Transaction {
  readonly id: string;
  readonly amount: number;
  readonly category: string;
  readonly date: string;
  readonly status: "completed" | "pending" | "failed";
}

// 이번 달 완료된 거래를 카테고리별로 합산하여 상위 5개 표시
const getTopCategories = (transactions: readonly Transaction[]) =>
  pipe(
    transactions,
    filter((t: Transaction) => t.status === "completed"),
    filter((t: Transaction) => t.date.startsWith("2024-01")),
    groupBy((t: Transaction) => t.category),
    (groups) => Object.entries(groups).map(([category, txns]) => ({
      category,
      total: txns.reduce((sum, t) => sum + t.amount, 0),
      count: txns.length,
    })),
    sort((a, b) => b.total - a.total),
    take(5)
  );
```

---

## 7. 프론트엔드 실전

### 7.1 미들웨어 패턴

미들웨어는 **함수 합성의 실전 적용**이다. Express, Redux, Koa 등에서 사용하는 미들웨어 패턴은 본질적으로 함수의 체이닝이다.

```typescript
// 미들웨어 타입 정의
type Middleware<T> = (context: T) => T;

// 미들웨어 합성
function applyMiddleware<T>(...middlewares: Middleware<T>[]): Middleware<T> {
  return (context: T) =>
    middlewares.reduce((ctx, middleware) => middleware(ctx), context);
}

// HTTP 요청 처리 미들웨어
interface RequestContext {
  readonly headers: Record<string, string>;
  readonly body: unknown;
  readonly user: { id: string; role: string } | null;
  readonly timestamp: number;
  readonly logs: readonly string[];
}

const addTimestamp: Middleware<RequestContext> = (ctx) => ({
  ...ctx,
  timestamp: Date.now(),
  logs: [...ctx.logs, "timestamp added"],
});

const authenticate: Middleware<RequestContext> = (ctx) => {
  const token = ctx.headers["authorization"];
  const user = token ? { id: "user-1", role: "admin" } : null;
  return {
    ...ctx,
    user,
    logs: [...ctx.logs, user ? "authenticated" : "anonymous"],
  };
};

const validateBody: Middleware<RequestContext> = (ctx) => ({
  ...ctx,
  logs: [...ctx.logs, ctx.body ? "body valid" : "body empty"],
});

// 미들웨어 파이프라인 합성
const processRequest = applyMiddleware(
  addTimestamp,
  authenticate,
  validateBody
);

const request: RequestContext = {
  headers: { authorization: "Bearer token123" },
  body: { name: "Alice" },
  user: null,
  timestamp: 0,
  logs: [],
};

const processed = processRequest(request);
console.log(processed.logs);
// ["timestamp added", "authenticated", "body valid"]
```

### 7.2 Redux 미들웨어의 본질

Redux 미들웨어는 `store => next => action => result` 형태의 **3중 커링** 함수이며, 합성으로 연결된다.

```typescript
// Redux 미들웨어의 단순화된 타입
type Action = { type: string; payload?: unknown };
type Dispatch = (action: Action) => Action;
type Store = { getState: () => unknown; dispatch: Dispatch };
type ReduxMiddleware = (store: Store) => (next: Dispatch) => Dispatch;

// 로깅 미들웨어
const logger: ReduxMiddleware = (store) => (next) => (action) => {
  console.log("dispatching:", action.type);
  const result = next(action);
  console.log("next state:", store.getState());
  return result;
};

// 비동기 미들웨어
const asyncMiddleware: ReduxMiddleware = (store) => (next) => (action) => {
  if (typeof action.payload === "function") {
    return (action.payload as Function)(store.dispatch, store.getState);
  }
  return next(action);
};

// applyMiddleware는 본질적으로 compose다
function applyReduxMiddleware(
  ...middlewares: ReduxMiddleware[]
): (store: Store) => Dispatch {
  return (store: Store) => {
    const chain = middlewares.map(m => m(store));
    // compose: 미들웨어들을 합성하여 하나의 dispatch 생성
    return chain.reduceRight(
      (next, middleware) => middleware(next),
      store.dispatch
    );
  };
}
```

### 7.3 React 데이터 흐름 파이프라인

```typescript
// 커스텀 훅에서의 데이터 변환 파이프라인
interface RawApiData {
  users: ApiUser[];
  total_count: number;
  page_size: number;
  current_page: number;
}

interface PaginatedUsers {
  readonly users: User[];
  readonly totalCount: number;
  readonly pageSize: number;
  readonly currentPage: number;
  readonly totalPages: number;
}

// 변환 함수들
const normalizeUserList = (data: RawApiData) => ({
  ...data,
  users: data.users.map(normalizeUser), // 6.1에서 만든 normalizeUser 활용
});

const normalizePagination = (data: {
  users: User[];
  total_count: number;
  page_size: number;
  current_page: number;
}) => ({
  users: data.users,
  totalCount: data.total_count,
  pageSize: data.page_size,
  currentPage: data.current_page,
  totalPages: Math.ceil(data.total_count / data.page_size),
});

// 파이프라인
const normalizeResponse = flow(normalizeUserList, normalizePagination);

// React 훅에서 사용
function useUsers(page: number) {
  const [data, setData] = useState<PaginatedUsers | null>(null);

  useEffect(() => {
    fetch(`/api/users?page=${page}`)
      .then(res => res.json())
      .then(raw => {
        // 파이프라인 적용
        const normalized = normalizeResponse(raw);
        setData(normalized);
      });
  }, [page]);

  return data;
}
```

### 7.4 이벤트 핸들러 합성

```typescript
// 이벤트 핸들러를 합성하여 관심사를 분리
type EventHandler<E> = (event: E) => void;

function composeHandlers<E>(...handlers: EventHandler<E>[]): EventHandler<E> {
  return (event: E) => {
    handlers.forEach(handler => handler(event));
  };
}

// 각 관심사별 핸들러
const logClick: EventHandler<React.MouseEvent> = (e) => {
  analytics.track("button_clicked", { x: e.clientX, y: e.clientY });
};

const hapticFeedback: EventHandler<React.MouseEvent> = () => {
  navigator.vibrate?.(10);
};

const handleSubmit: EventHandler<React.MouseEvent> = () => {
  submitForm();
};

// 합성
const onButtonClick = composeHandlers(logClick, hapticFeedback, handleSubmit);

// 사용
function SubmitButton() {
  return <button onClick={onButtonClick}>제출</button>;
}
```

### 7.5 유효성 검증 합성

```typescript
// Validator를 합성하여 복잡한 유효성 검증을 조립
type ValidationResult = { valid: true } | { valid: false; error: string };
type Validator<T> = (value: T) => ValidationResult;

const ok: ValidationResult = { valid: true };
const fail = (error: string): ValidationResult => ({ valid: false, error });

// 개별 검증기
const required: Validator<string> = (value) =>
  value.trim().length > 0 ? ok : fail("필수 입력입니다");

const minLength = (min: number): Validator<string> => (value) =>
  value.length >= min ? ok : fail(`최소 ${min}자 이상 입력하세요`);

const maxLength = (max: number): Validator<string> => (value) =>
  value.length <= max ? ok : fail(`최대 ${max}자까지 입력 가능합니다`);

const matches = (pattern: RegExp, message: string): Validator<string> => (value) =>
  pattern.test(value) ? ok : fail(message);

// 검증기 합성: 모든 조건을 만족해야 통과
function composeValidators<T>(...validators: Validator<T>[]): Validator<T> {
  return (value: T) => {
    for (const validate of validators) {
      const result = validate(value);
      if (!result.valid) return result; // 첫 번째 실패에서 중단
    }
    return ok;
  };
}

// 합성된 검증기
const validateEmail = composeValidators(
  required,
  minLength(5),
  maxLength(254),
  matches(/^[^\s@]+@[^\s@]+\.[^\s@]+$/, "올바른 이메일 형식이 아닙니다")
);

const validatePassword = composeValidators(
  required,
  minLength(8),
  maxLength(100),
  matches(/[A-Z]/, "대문자를 포함해야 합니다"),
  matches(/[0-9]/, "숫자를 포함해야 합니다")
);

// 사용
console.log(validateEmail(""));            // { valid: false, error: "필수 입력입니다" }
console.log(validateEmail("abc"));          // { valid: false, error: "최소 5자 이상 입력하세요" }
console.log(validateEmail("abc@d.e"));      // { valid: true }
console.log(validatePassword("weak"));      // { valid: false, error: "최소 8자 이상 입력하세요" }
console.log(validatePassword("StrongPass1")); // { valid: true }
```

---

## 8. fp-ts의 pipe와 flow

### 8.1 fp-ts가 제공하는 합성 도구

fp-ts는 TypeScript 함수형 프로그래밍의 사실상 표준 라이브러리다. `pipe`와 `flow`가 핵심 합성 도구로 제공된다.

```typescript
import { pipe, flow } from "fp-ts/function";
import * as A from "fp-ts/Array";
import * as O from "fp-ts/Option";

// fp-ts의 pipe: 최대 20개 함수까지 타입 안전
const result = pipe(
  [1, 2, 3, 4, 5],
  A.filter(n => n % 2 === 0),
  A.map(n => n * 10),
  A.reduce(0, (acc, n) => acc + n)
);
// result: 60

// fp-ts의 flow: 함수 합성
const doubleEvens = flow(
  A.filter((n: number) => n % 2 === 0),
  A.map(n => n * 10)
);

console.log(doubleEvens([1, 2, 3, 4])); // [20, 40]
```

### 8.2 fp-ts에서의 파이프라인 실전

```typescript
import { pipe, flow } from "fp-ts/function";
import * as O from "fp-ts/Option";
import * as E from "fp-ts/Either";
import * as A from "fp-ts/Array";

// Option 파이프라인: null-safe 데이터 접근
interface Config {
  database?: {
    host?: string;
    port?: number;
  };
}

const getDbHost = (config: Config): string =>
  pipe(
    O.fromNullable(config.database),
    O.flatMap(db => O.fromNullable(db.host)),
    O.getOrElse(() => "localhost")
  );

// Either 파이프라인: 에러 처리 체인
const parseAndValidate = (input: string): E.Either<string, number> =>
  pipe(
    E.tryCatch(
      () => JSON.parse(input),
      () => "JSON 파싱 실패"
    ),
    E.flatMap(value =>
      typeof value === "number"
        ? E.right(value)
        : E.left("숫자가 아닙니다")
    ),
    E.flatMap(n =>
      n > 0
        ? E.right(n)
        : E.left("양수여야 합니다")
    )
  );

console.log(parseAndValidate("42"));      // { _tag: "Right", right: 42 }
console.log(parseAndValidate('"hello"'));  // { _tag: "Left", left: "숫자가 아닙니다" }
console.log(parseAndValidate("-5"));       // { _tag: "Left", left: "양수여야 합니다" }
```

### 8.3 fp-ts의 설계 철학

fp-ts에서 모든 모듈(Array, Option, Either 등)은 `map`, `filter`, `flatMap` 같은 연산을 **커링된 형태**로 제공한다. 이것은 `pipe`에서 합성하기 위함이다.

```typescript
// fp-ts 스타일: 커링된 연산 + pipe
import * as A from "fp-ts/Array";

// A.map은 (f) => (array) => result 형태
// pipe에서 자연스럽게 연결됨
const result = pipe(
  users,
  A.filter(u => u.isActive),   // (array) => filteredArray
  A.map(u => u.name),           // (array) => names
  A.sort(S.Ord)                 // (array) => sortedNames
);

// 반면 일반 배열 메서드는 체이닝 방식
const result2 = users
  .filter(u => u.isActive)
  .map(u => u.name)
  .sort();

// 차이점:
// - pipe 방식: 어떤 타입(Option, Either, Task 등)이든 동일한 패턴
// - 메서드 체이닝: 프로토타입에 메서드가 있어야 함 (확장 불가)
```

---

## 9. 합성의 한계와 다음 단계

### 9.1 단순 합성이 실패하는 경우

지금까지의 합성은 `A → B` 형태의 순수 함수만 다뤘다. 하지만 현실에는 **실패할 수 있는** 함수가 있다.

```typescript
// 이 함수들은 단순 합성할 수 없다
const parse = (s: string): number | null => {
  const n = Number(s);
  return isNaN(n) ? null : n;
};

const sqrt = (n: number): number | null => {
  return n >= 0 ? Math.sqrt(n) : null;
};

const reciprocal = (n: number): number | null => {
  return n !== 0 ? 1 / n : null;
};

// ❌ 단순 합성 불가 — 타입이 맞지 않음!
// compose(reciprocal, sqrt, parse)
// parse의 출력: number | null
// sqrt의 입력: number (null이면 안 됨)

// ❌ 매번 null 체크를 해야 함
function processManual(input: string): number | null {
  const parsed = parse(input);
  if (parsed === null) return null;

  const sqrtResult = sqrt(parsed);
  if (sqrtResult === null) return null;

  return reciprocal(sqrtResult);
}
// 이것은 합성이 아니라 수동 연결이다
```

### 9.2 이것이 Monad가 필요한 이유

`A → B`가 아닌 `A → M<B>` 형태의 함수(실패할 수 있는, 비동기인, 부수 효과가 있는 등)를 합성하는 것이 **Monad**의 역할이다. Phase 2에서 `Option`과 `Either`로 이 문제를 해결하기 시작하고, Phase 4에서 Monad를 본격적으로 다룬다.

```typescript
// 미리보기: Monad 합성 (15주차에서 본격적으로)
import * as O from "fp-ts/Option";
import { pipe } from "fp-ts/function";

const safeParse = (s: string): O.Option<number> => {
  const n = Number(s);
  return isNaN(n) ? O.none : O.some(n);
};

const safeSqrt = (n: number): O.Option<number> =>
  n >= 0 ? O.some(Math.sqrt(n)) : O.none;

const safeReciprocal = (n: number): O.Option<number> =>
  n !== 0 ? O.some(1 / n) : O.none;

// ✅ flatMap으로 합성 가능!
const process = (input: string): O.Option<number> =>
  pipe(
    safeParse(input),
    O.flatMap(safeSqrt),
    O.flatMap(safeReciprocal)
  );

console.log(process("4"));   // Some(0.5)  — 4 → √4=2 → 1/2=0.5
console.log(process("-1"));  // None       — √(-1)은 불가
console.log(process("abc")); // None       — 파싱 실패
```

지금은 "합성할 수 없는 함수도 합성 가능하게 만드는 도구가 있다"는 것만 기억하면 된다. Phase 1에서 배운 **순수 함수, 일급 함수, 불변성, 합성** — 이 기초 위에 모든 것이 세워진다.

---

## 연습 문제

### 문제 1: compose와 pipe 구현

직접 `compose`와 `pipe`를 구현하라. 각각 최소 3개 함수를 받을 수 있어야 한다.

```typescript
// (a) compose 구현 — 오른쪽에서 왼쪽 실행
function compose<A, B, C>(f: (b: B) => C, g: (a: A) => B): (a: A) => C;
function compose<A, B, C, D>(
  f: (c: C) => D, g: (b: B) => C, h: (a: A) => B
): (a: A) => D;
// 구현하라

// (b) pipe 구현 — 왼쪽에서 오른쪽, 값부터 시작
function pipe<A, B>(a: A, f: (a: A) => B): B;
function pipe<A, B, C>(a: A, f: (a: A) => B, g: (b: B) => C): C;
function pipe<A, B, C, D>(
  a: A, f: (a: A) => B, g: (b: B) => C, h: (c: C) => D
): D;
// 구현하라

// (c) 다음 함수들을 compose와 pipe 각각으로 합성하여 같은 결과를 내라
const exclaim = (s: string): string => s + "!";
const toUpper = (s: string): string => s.toUpperCase();
const repeat = (s: string): string => s + " " + s;

// compose로: "hello" → "HELLO! HELLO!"
// pipe로: "hello" → "HELLO! HELLO!"
```

### 문제 2: 데이터 변환 파이프라인

다음 API 응답을 요구사항에 맞게 변환하는 파이프라인을 `pipe`로 작성하라.

```typescript
interface RawProduct {
  product_id: number;
  product_name: string;
  unit_price: number;
  stock_quantity: number;
  category_name: string;
  is_available: boolean;
}

const rawProducts: RawProduct[] = [
  { product_id: 1, product_name: "노트북", unit_price: 1200000, stock_quantity: 5, category_name: "전자기기", is_available: true },
  { product_id: 2, product_name: "마우스", unit_price: 35000, stock_quantity: 0, category_name: "전자기기", is_available: false },
  { product_id: 3, product_name: "책상", unit_price: 450000, stock_quantity: 12, category_name: "가구", is_available: true },
  { product_id: 4, product_name: "의자", unit_price: 380000, stock_quantity: 8, category_name: "가구", is_available: true },
  { product_id: 5, product_name: "모니터", unit_price: 550000, stock_quantity: 3, category_name: "전자기기", is_available: true },
];

// 요구사항:
// 1. 사용 가능한(is_available) 상품만 필터
// 2. 가격에 10% 세금 추가
// 3. snake_case → camelCase 변환
// 4. 카테고리별로 그룹핑
// 5. 각 카테고리의 총 재고와 평균 가격 계산

// 최종 결과 타입:
interface CategorySummary {
  readonly category: string;
  readonly products: readonly { id: number; name: string; price: number }[];
  readonly totalStock: number;
  readonly averagePrice: number;
}

// pipe를 사용하여 rawProducts → CategorySummary[] 변환을 구현하라
```

### 문제 3: 미들웨어 합성

다음 미들웨어 시스템을 구현하라.

```typescript
interface Context {
  readonly path: string;
  readonly method: string;
  readonly headers: Record<string, string>;
  readonly body: unknown;
  readonly response: {
    readonly status: number;
    readonly body: unknown;
  };
  readonly logs: readonly string[];
}

type Middleware = (ctx: Context) => Context;

// (a) 다음 미들웨어들을 구현하라
// - corsMiddleware: "Access-Control-Allow-Origin"을 "*"로 설정
// - authMiddleware: headers에 "authorization"이 있으면 통과, 없으면 status 401
// - logMiddleware: logs에 `${method} ${path}` 추가
// - notFoundMiddleware: path가 매칭되지 않으면 status 404

// (b) 미들웨어를 합성하는 함수를 구현하라
function applyMiddlewares(...middlewares: Middleware[]): Middleware {
  // 구현하라
}

// (c) 합성된 미들웨어로 다음 요청을 처리한 결과를 예측하라
const handler = applyMiddlewares(
  logMiddleware,
  corsMiddleware,
  authMiddleware
);

const request: Context = {
  path: "/api/users",
  method: "GET",
  headers: { authorization: "Bearer token123" },
  body: null,
  response: { status: 200, body: null },
  logs: [],
};
```

### 문제 4: Point-free 변환

다음 pointed 스타일 함수를 point-free로 변환하라. 필요한 유틸리티 함수도 함께 작성하라.

```typescript
// 유틸리티: 직접 구현 필요
// prop, map, filter, sort, flow, ...

// (a) pointed → point-free
const getActiveNames = (users: User[]) =>
  users
    .filter(u => u.isActive)
    .map(u => u.name)
    .sort();

// (b) pointed → point-free
const sumPrices = (products: Product[]) =>
  products
    .map(p => p.price)
    .reduce((a, b) => a + b, 0);

// (c) pointed → point-free
const formatTitle = (title: string) =>
  title.trim().toLowerCase().replace(/\s+/g, "-");
```

---

## 연습 문제 해답

### 문제 1 해답

```typescript
// (a) compose 구현
function compose<A, B, C>(f: (b: B) => C, g: (a: A) => B): (a: A) => C;
function compose<A, B, C, D>(
  f: (c: C) => D, g: (b: B) => C, h: (a: A) => B
): (a: A) => D;
function compose(...fns: Function[]): Function {
  return (x: unknown) => fns.reduceRight((acc, fn) => fn(acc), x);
}

// (b) pipe 구현
function pipe<A, B>(a: A, f: (a: A) => B): B;
function pipe<A, B, C>(a: A, f: (a: A) => B, g: (b: B) => C): C;
function pipe<A, B, C, D>(
  a: A, f: (a: A) => B, g: (b: B) => C, h: (c: C) => D
): D;
function pipe(a: unknown, ...fns: Function[]): unknown {
  return fns.reduce((acc, fn) => fn(acc), a);
}

// (c) compose와 pipe로 같은 결과
const exclaim = (s: string): string => s + "!";
const toUpper = (s: string): string => s.toUpperCase();
const repeat = (s: string): string => s + " " + s;

// compose: 오른쪽에서 왼쪽 실행
// "hello" → toUpper → exclaim → repeat
const withCompose = compose(repeat, exclaim, toUpper);
console.log(withCompose("hello")); // "HELLO! HELLO!"

// pipe: 왼쪽에서 오른쪽 실행
const withPipe = pipe(
  "hello",
  toUpper,   // "HELLO"
  exclaim,   // "HELLO!"
  repeat     // "HELLO! HELLO!"
);
console.log(withPipe); // "HELLO! HELLO!"
```

### 문제 2 해답

```typescript
// 유틸리티 함수들
const filterBy = <T>(pred: (item: T) => boolean) =>
  (items: readonly T[]): T[] => items.filter(pred);

const mapWith = <A, B>(f: (a: A) => B) =>
  (items: readonly A[]): B[] => items.map(f);

const groupByKey = <T>(keyFn: (item: T) => string) =>
  (items: readonly T[]): Record<string, T[]> =>
    items.reduce<Record<string, T[]>>((groups, item) => {
      const key = keyFn(item);
      return { ...groups, [key]: [...(groups[key] ?? []), item] };
    }, {});

// 변환 파이프라인
const result = pipe(
  rawProducts,

  // 1. 사용 가능한 상품만 필터
  filterBy((p: RawProduct) => p.is_available),

  // 2. 가격에 10% 세금 추가 + 3. camelCase 변환
  mapWith((p: RawProduct) => ({
    id: p.product_id,
    name: p.product_name,
    price: Math.round(p.unit_price * 1.1),
    stock: p.stock_quantity,
    category: p.category_name,
  })),

  // 4. 카테고리별 그룹핑
  groupByKey(p => p.category),

  // 5. 카테고리 요약 계산
  (groups) => Object.entries(groups).map(([category, products]): CategorySummary => ({
    category,
    products: products.map(p => ({ id: p.id, name: p.name, price: p.price })),
    totalStock: products.reduce((sum, p) => sum + p.stock, 0),
    averagePrice: Math.round(
      products.reduce((sum, p) => sum + p.price, 0) / products.length
    ),
  }))
);

// 결과:
// [
//   {
//     category: "전자기기",
//     products: [
//       { id: 1, name: "노트북", price: 1320000 },
//       { id: 5, name: "모니터", price: 605000 }
//     ],
//     totalStock: 8,
//     averagePrice: 962500
//   },
//   {
//     category: "가구",
//     products: [
//       { id: 3, name: "책상", price: 495000 },
//       { id: 4, name: "의자", price: 418000 }
//     ],
//     totalStock: 20,
//     averagePrice: 456500
//   }
// ]
```

### 문제 3 해답

```typescript
// (a) 미들웨어 구현
const logMiddleware: Middleware = (ctx) => ({
  ...ctx,
  logs: [...ctx.logs, `${ctx.method} ${ctx.path}`],
});

const corsMiddleware: Middleware = (ctx) => ({
  ...ctx,
  headers: { ...ctx.headers, "Access-Control-Allow-Origin": "*" },
});

const authMiddleware: Middleware = (ctx) => {
  if (!ctx.headers["authorization"]) {
    return {
      ...ctx,
      response: { status: 401, body: "Unauthorized" },
      logs: [...ctx.logs, "auth failed"],
    };
  }
  return {
    ...ctx,
    logs: [...ctx.logs, "auth passed"],
  };
};

// (b) 미들웨어 합성
function applyMiddlewares(...middlewares: Middleware[]): Middleware {
  return (ctx: Context) =>
    middlewares.reduce((currentCtx, middleware) => middleware(currentCtx), ctx);
}
// 또는 pipe 스타일:
// return (ctx) => pipe(ctx, ...middlewares);

// (c) 결과 예측
const handler = applyMiddlewares(logMiddleware, corsMiddleware, authMiddleware);
const processed = handler(request);

// processed:
// {
//   path: "/api/users",
//   method: "GET",
//   headers: {
//     authorization: "Bearer token123",
//     "Access-Control-Allow-Origin": "*"    ← cors 추가됨
//   },
//   body: null,
//   response: { status: 200, body: null },  ← 인증 통과, 200 유지
//   logs: ["GET /api/users", "auth passed"] ← 로그 2개
// }

// 실행 순서:
// 1. logMiddleware: logs에 "GET /api/users" 추가
// 2. corsMiddleware: headers에 CORS 헤더 추가
// 3. authMiddleware: authorization이 있으므로 통과, logs에 "auth passed" 추가
```

### 문제 4 해답

```typescript
// 유틸리티 함수
const prop = <K extends string>(key: K) =>
  <T extends Record<K, unknown>>(obj: T): T[K] => obj[key];

const map = <A, B>(f: (a: A) => B) =>
  (arr: readonly A[]): B[] => arr.map(f);

const filter = <A>(pred: (a: A) => boolean) =>
  (arr: readonly A[]): A[] => arr.filter(pred);

const sortAsc = (arr: readonly string[]): string[] => [...arr].sort();

const reduce = <A, B>(f: (acc: B, a: A) => B, init: B) =>
  (arr: readonly A[]): B => arr.reduce(f, init);

const sum = (a: number, b: number): number => a + b;

const trim = (s: string): string => s.trim();
const lower = (s: string): string => s.toLowerCase();
const replaceSpaces = (s: string): string => s.replace(/\s+/g, "-");

// (a) point-free 변환
interface User { name: string; isActive: boolean }

const getActiveNames = flow(
  filter((u: User) => u.isActive),
  map(prop("name") as (u: User) => string),
  sortAsc
);

// 검증
const users: User[] = [
  { name: "Charlie", isActive: true },
  { name: "Alice", isActive: true },
  { name: "Bob", isActive: false },
];
console.log(getActiveNames(users)); // ["Alice", "Charlie"]

// (b) point-free 변환
interface Product { price: number }

const sumPrices = flow(
  map((p: Product) => p.price),
  reduce(sum, 0)
);

// 검증
const products: Product[] = [{ price: 100 }, { price: 200 }, { price: 300 }];
console.log(sumPrices(products)); // 600

// (c) point-free 변환
const formatTitle = flow(trim, lower, replaceSpaces);

// 검증
console.log(formatTitle("  Hello World  ")); // "hello-world"
console.log(formatTitle(" Functional Programming Rocks ")); // "functional-programming-rocks"
```

---

## 핵심 정리

| 개념 | 한 줄 요약 |
|------|-----------|
| 함수 합성 | 두 함수를 연결하여 새 함수를 만드는 연산: (f ∘ g)(x) = f(g(x)) |
| 결합법칙 | (f ∘ g) ∘ h === f ∘ (g ∘ h) — 괄호 위치가 결과에 영향 없음 |
| 항등 함수 | identity(x) = x — 합성의 항등원, 아무것도 안 하는 함수 |
| compose | 오른쪽→왼쪽 합성, 수학적 표기와 일치 |
| flow | 왼쪽→오른쪽 합성, 실행 순서와 읽기 순서 일치 |
| pipe | 초기값 + 왼쪽→오른쪽 합성, 즉시 값을 반환 |
| Point-free | 인자를 명시하지 않는 스타일, 가독성 향상 시에만 사용 |
| 미들웨어 | 함수 합성의 실전 패턴, 관심사 분리와 합성을 동시에 달성 |
| 합성의 한계 | A→B 합성은 실패/비동기를 다루지 못함, 이것이 Monad의 출발점 |

## Phase 1 완료!

4주간 배운 핵심을 다시 보자:

```
1주차: 순수 함수 — 같은 입력, 같은 출력, 부수 효과 없음
2주차: 일급 함수 — 함수를 값으로, 고차 함수로 추상화
3주차: 불변성   — 데이터를 변경하지 않고, 새로 만든다
4주차: 합성     — 작은 함수를 조립하여 복잡한 동작을 만든다
```

이 네 가지는 함수형 프로그래밍의 **사고 체계**다. 앞으로 배울 모든 개념(ADT, Functor, Monad, Effect 등)은 이 기초 위에 세워진다.

## 다음 주차 예고

**5주차: 대수적 데이터 타입 (ADT)** — Phase 2의 시작. 곱 타입(Product Type)과 합 타입(Sum Type)으로 **불가능한 상태를 불가능하게** 만든다. `Loading | Success | Error` 같은 판별 유니언으로 API 응답을 모델링하고, 패턴 매칭으로 모든 케이스를 빠짐없이 처리하는 방법을 배운다.
