# 1주차: 함수란 무엇인가 — 수학적 함수와 프로그래밍

## 왜 이것부터 시작하는가?

프로그래밍에서 "함수"라는 단어를 매일 쓰지만, 수학에서 말하는 함수와 프로그래밍의 함수는 근본적으로 다르다. 이 차이를 이해하는 것이 함수형 프로그래밍의 출발점이다.

명령형 프로그래밍의 함수는 **절차(procedure)** 에 가깝다. "이 일을 해라"라는 명령의 묶음이다. 반면 수학의 함수는 **관계(relation)** 다. 입력과 출력 사이의 대응 규칙이다. 함수형 프로그래밍은 이 수학적 함수의 성질을 프로그래밍에 가져오려는 시도다.

---

## 1. 수학적 함수의 정의

### 1.1 집합과 대응

수학에서 함수 `f: A → B`는 다음을 의미한다:

- **A**: 정의역(Domain) — 입력이 될 수 있는 모든 값의 집합
- **B**: 공역(Codomain) — 출력이 될 수 있는 모든 값의 집합
- **f(A)**: 치역(Range/Image) — 실제로 출력되는 값의 집합 (공역의 부분집합)

**핵심 조건**: 정의역의 **모든** 원소는 공역의 **정확히 하나의** 원소에 대응해야 한다.

```
정의역 A = {1, 2, 3}
공역   B = {a, b, c, d}

f(1) = a
f(2) = c
f(3) = a

치역 f(A) = {a, c}  ⊆  B
```

이것이 함수형 프로그래밍에서 중요한 이유: **같은 입력은 항상 같은 출력을 보장**한다는 것이다.

### 1.2 함수가 아닌 것

```
g(1) = a
g(1) = b  ← 같은 입력에 다른 출력 → 함수가 아님!

h(1) = a
h(2) = b
(3은 대응 없음) ← 정의역의 원소가 빠짐 → 함수가 아님!
```

프로그래밍에서 같은 입력에 다른 출력이 나오는 경우:

```typescript
// 이것은 수학적 의미의 함수가 아니다
let count = 0;
function increment(): number {
  return ++count; // 호출할 때마다 다른 값 반환
}

increment(); // 1
increment(); // 2 — 같은 입력(없음)에 다른 출력
```

---

## 2. 전사·단사·전단사 함수

함수의 대응 방식에 따라 세 가지로 분류된다. 이 분류는 나중에 타입 시스템과 동형 사상(Isomorphism)을 이해하는 데 필수적이다.

### 2.1 단사 함수 (Injective / One-to-one)

**정의**: 서로 다른 입력은 항상 서로 다른 출력을 낸다.

```
f(a) = f(b) → a = b  (대우: a ≠ b → f(a) ≠ f(b))
```

```typescript
// 단사 함수 — 서로 다른 입력은 서로 다른 출력
function double(n: number): number {
  return n * 2;
}
// double(1) = 2, double(2) = 4, double(3) = 6 — 모두 다른 출력

// 단사가 아닌 함수
function abs(n: number): number {
  return Math.abs(n);
}
// abs(-3) = 3, abs(3) = 3 — 다른 입력이 같은 출력
```

**프로그래밍에서의 의미**: 단사 함수는 정보를 잃지 않는다. 출력에서 입력을 복원할 수 있다.

### 2.2 전사 함수 (Surjective / Onto)

**정의**: 공역의 모든 원소가 적어도 하나의 입력에 대응된다. 즉 치역 = 공역이다.

```typescript
// TypeScript에서의 전사 예시
// boolean → boolean인 not 함수는 전사다
function not(b: boolean): boolean {
  return !b;
}
// not(true) = false, not(false) = true — 공역 {true, false}의 모든 값이 출력됨

// number → number인 다음 함수는 전사가 아니다
function square(n: number): number {
  return n * n;
}
// 출력이 항상 0 이상 — 음수는 공역에 있지만 치역에 없음
```

### 2.3 전단사 함수 (Bijective)

**정의**: 단사이면서 전사. 입력과 출력이 정확히 1:1 대응된다.

```typescript
// 전단사 함수 — 역함수가 존재한다
function celsiusToFahrenheit(c: number): number {
  return c * 9/5 + 32;
}

function fahrenheitToCelsius(f: number): number {
  return (f - 32) * 5/9;
}

// fahrenheitToCelsius(celsiusToFahrenheit(100)) === 100 ✓
// celsiusToFahrenheit(fahrenheitToCelsius(212)) === 212 ✓
```

**프론트엔드에서의 의미**: 전단사 함수는 **가역적 변환**이다. 직렬화/역직렬화, 인코딩/디코딩이 전단사여야 데이터 손실이 없다.

```typescript
// JSON 직렬화는 전단사여야 안전하다
const data = { name: "Alice", age: 30 };
const serialized = JSON.stringify(data);
const deserialized = JSON.parse(serialized);
// deserialized 깊은 동등성 data ✓

// 하지만 Date 객체에서는 전단사가 깨진다!
const withDate = { created: new Date("2024-01-01") };
const s = JSON.stringify(withDate);  // {"created":"2024-01-01T00:00:00.000Z"}
const d = JSON.parse(s);
// d.created는 string이지 Date가 아님 — 정보가 변형됨
```

---

## 3. 순수 함수 (Pure Function)

### 3.1 정의

순수 함수는 두 가지 조건을 만족하는 함수다:

1. **결정론적(Deterministic)**: 같은 입력이면 항상 같은 출력
2. **부수 효과 없음(No Side Effects)**: 함수 외부의 상태를 변경하지 않음

```typescript
// ✅ 순수 함수
function add(a: number, b: number): number {
  return a + b;
}

function formatName(first: string, last: string): string {
  return `${last}, ${first}`;
}

function filterAdults(users: ReadonlyArray<User>): ReadonlyArray<User> {
  return users.filter(u => u.age >= 18);
}
```

```typescript
// ❌ 순수하지 않은 함수들
let total = 0;
function addToTotal(n: number): number {
  total += n;       // 외부 상태 변경 (부수 효과)
  return total;
}

function getCurrentTimestamp(): number {
  return Date.now(); // 같은 입력(없음)에 매번 다른 출력 (비결정론적)
}

function saveUser(user: User): void {
  db.save(user);     // 데이터베이스에 쓰기 (부수 효과)
  console.log("saved"); // 콘솔 출력 (부수 효과)
}
```

### 3.2 순수 함수의 가치

왜 순수 함수가 중요한가? 구체적인 이유가 있다:

**테스트 용이성**: 외부 환경 설정 없이 입력-출력만 검증하면 된다.

```typescript
// 순수 함수는 테스트가 단순하다
describe("add", () => {
  it("adds two numbers", () => {
    expect(add(2, 3)).toBe(5);
    // 끝. mock 없음. setup 없음. teardown 없음.
  });
});
```

**캐싱 가능(Memoization)**: 같은 입력은 같은 출력이므로 결과를 안전하게 캐시할 수 있다.

```typescript
function memoize<A, B>(fn: (a: A) => B): (a: A) => B {
  const cache = new Map<A, B>();
  return (a: A) => {
    if (cache.has(a)) return cache.get(a)!;
    const result = fn(a);
    cache.set(a, result);
    return result;
  };
}

const expensiveCalc = memoize((n: number) => {
  // 복잡한 계산...
  return fibonacci(n);
});
```

**병렬 실행 안전**: 공유 상태를 변경하지 않으므로 동시 실행해도 안전하다.

**리팩토링 안전**: 참조 투명성 덕분에 순서를 바꾸거나 추출해도 동작이 같다.

---

## 4. 참조 투명성 (Referential Transparency)

### 4.1 정의

표현식이 **참조 투명**하다는 것은, 그 표현식을 그 값으로 치환해도 프로그램의 동작이 바뀌지 않는다는 뜻이다.

```typescript
// 참조 투명한 코드
const x = add(2, 3);        // x = 5
const result = x + x;       // 10

// add(2, 3)을 5로 치환해도 동일하다
const result2 = add(2, 3) + add(2, 3); // 10 ✓
const result3 = 5 + 5;                  // 10 ✓
```

```typescript
// 참조 투명하지 않은 코드
let counter = 0;
function next(): number {
  return ++counter;
}

const a = next();          // a = 1
const b = a + a;           // b = 2

// next()를 그 값(1)으로 치환하면?
const c = next() + next(); // c = 5 (2 + 3) — 다르다!
```

### 4.2 치환 모델 (Substitution Model)

참조 투명성이 보장되면 코드를 **대수적으로 추론**할 수 있다. 수학에서 방정식을 풀듯이 코드를 변환할 수 있다.

```typescript
// 이 파이프라인을 단계별로 추론할 수 있다
const process = (users: User[]) =>
  users
    .filter(u => u.age >= 18)
    .map(u => u.name)
    .sort();

// 각 단계가 순수하므로, 중간 결과를 자유롭게 추출·합성할 수 있다
const adults = (users: User[]) => users.filter(u => u.age >= 18);
const names = (users: User[]) => users.map(u => u.name);
const sorted = (names: string[]) => [...names].sort();

// 동일한 결과가 보장된다
const process2 = (users: User[]) => sorted(names(adults(users)));
```

---

## 5. 부수 효과 (Side Effect)

### 5.1 부수 효과란 무엇인가

함수가 반환값 외에 하는 **모든 것**이 부수 효과다:

| 부수 효과 | 예시 |
|-----------|------|
| 변수 변경 | `count++`, `arr.push(x)` |
| 화면 출력 | `console.log()` |
| DOM 조작 | `document.getElementById()` |
| 네트워크 | `fetch()`, `XMLHttpRequest` |
| 파일 I/O | `fs.readFile()` |
| 예외 발생 | `throw new Error()` |
| 랜덤 값 | `Math.random()` |
| 현재 시간 | `Date.now()`, `new Date()` |

### 5.2 부수 효과가 왜 문제인가

부수 효과 자체가 나쁜 것은 아니다. 프로그램은 결국 어딘가에서 부수 효과를 일으켜야 의미가 있다. 문제는 부수 효과가 **코드 어디에나 흩어져** 있을 때다.

```typescript
// 부수 효과가 흩어진 코드 — 추론이 어렵다
function processOrder(order: Order): void {
  const total = calculateTotal(order.items); // 순수할까?
  applyDiscount(order);     // order를 변경할까?
  sendEmail(order.user);    // 네트워크 호출
  updateInventory(order);   // DB 쓰기
  logAnalytics(order);      // 외부 서비스 호출
  // 어떤 순서로 실행되어야 하나? 하나가 실패하면?
}
```

```typescript
// 부수 효과를 분리한 코드 — 순수 로직과 효과를 나눔
// 1. 순수 로직: 무엇을 해야 하는지 계산
function planOrder(order: Order): OrderPlan {
  const total = calculateTotal(order.items);
  const discount = determineDiscount(order);
  const finalTotal = applyDiscount(total, discount);
  return {
    total: finalTotal,
    emailTo: order.user.email,
    inventoryChanges: order.items.map(i => ({ id: i.id, delta: -i.qty })),
  };
}

// 2. 효과 실행: 계산된 계획을 실행
async function executeOrderPlan(plan: OrderPlan): Promise<void> {
  await db.updateInventory(plan.inventoryChanges);
  await emailService.send(plan.emailTo, formatReceipt(plan));
  await analytics.track("order_completed", { total: plan.total });
}
```

이 분리의 핵심: `planOrder`는 순수하므로 테스트, 캐싱, 추론이 쉽다. `executeOrderPlan`은 효과를 모아놓았으므로 경계가 명확하다.

---

## 6. 프론트엔드 실전: React 컴포넌트를 순수 함수로 바라보기

### 6.1 React 컴포넌트는 함수다

React 함수 컴포넌트는 본질적으로 `Props → JSX` 함수다.

```typescript
// 순수 컴포넌트: 같은 props → 같은 UI
interface UserCardProps {
  readonly name: string;
  readonly email: string;
  readonly role: "admin" | "user";
}

function UserCard({ name, email, role }: UserCardProps): JSX.Element {
  return (
    <div className={`card card--${role}`}>
      <h2>{name}</h2>
      <p>{email}</p>
      {role === "admin" && <span className="badge">Admin</span>}
    </div>
  );
}

// 같은 props를 넣으면 항상 같은 JSX를 반환한다
// UserCard({ name: "Alice", email: "a@b.c", role: "admin" })
// → 항상 동일한 결과
```

### 6.2 순수하지 않은 컴포넌트의 문제

```typescript
// ❌ 순수하지 않은 컴포넌트
function Clock(): JSX.Element {
  // Date.now()는 호출 시점마다 다른 값 → 비결정론적
  return <span>{new Date().toLocaleTimeString()}</span>;
}

// ❌ 렌더링 중 부수 효과
function BadComponent({ id }: { id: string }): JSX.Element {
  // 렌더링 중에 DOM을 직접 조작 — 부수 효과
  document.title = `Item ${id}`;
  return <div>{id}</div>;
}
```

### 6.3 React가 순수성을 활용하는 방법

React는 컴포넌트가 순수하다는 가정 하에 최적화를 수행한다:

```typescript
// React.memo: 같은 props면 리렌더링하지 않음
// 이것은 순수 함수의 memoization과 같은 원리다
const MemoizedUserCard = React.memo(UserCard);

// props가 바뀌지 않으면 이전 결과를 재사용
// 이것이 가능한 이유: 같은 입력 → 같은 출력이 보장되므로

// useMemo: 계산 결과를 캐시
function UserList({ users }: { users: ReadonlyArray<User> }): JSX.Element {
  // users가 바뀌지 않으면 필터링 결과를 재사용
  const activeUsers = useMemo(
    () => users.filter(u => u.isActive),
    [users]
  );

  return (
    <ul>
      {activeUsers.map(u => (
        <li key={u.id}>{u.name}</li>
      ))}
    </ul>
  );
}
```

### 6.4 부수 효과를 useEffect로 분리

React에서 부수 효과를 다루는 공식 방법은 `useEffect`다. 이것은 **순수한 렌더링**과 **부수 효과**를 분리하는 장치다.

```typescript
function UserProfile({ userId }: { userId: string }): JSX.Element {
  const [user, setUser] = useState<User | null>(null);

  // 부수 효과를 렌더링에서 분리
  useEffect(() => {
    // 이 안에서만 부수 효과 수행
    fetchUser(userId).then(setUser);
  }, [userId]);

  // 렌더링은 순수: user 상태에 따라 결정론적으로 UI 반환
  if (!user) return <Loading />;
  return <UserCard name={user.name} email={user.email} role={user.role} />;
}
```

구조를 정리하면:

```
React 컴포넌트 = 순수 함수(렌더링) + 분리된 효과(useEffect)

┌─────────────────────────────┐
│  Props/State → JSX (순수)   │  ← 결정론적, 테스트 가능
├─────────────────────────────┤
│  useEffect (부수 효과)      │  ← API 호출, DOM 조작 등
└─────────────────────────────┘
```

---

## 연습 문제

### 문제 1: 순수 함수 판별

다음 함수들이 순수한지 판별하고, 순수하지 않다면 이유를 설명하라.

```typescript
// (a)
function greet(name: string): string {
  return `Hello, ${name}!`;
}

// (b)
const config = { prefix: "Mr." };
function formalGreet(name: string): string {
  return `Hello, ${config.prefix} ${name}!`;
}

// (c)
function shuffle<T>(arr: T[]): T[] {
  const copy = [...arr];
  for (let i = copy.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [copy[i], copy[j]] = [copy[j], copy[i]];
  }
  return copy;
}

// (d)
function head<T>(arr: ReadonlyArray<T>): T {
  if (arr.length === 0) throw new Error("empty array");
  return arr[0];
}

// (e)
function map<A, B>(arr: ReadonlyArray<A>, f: (a: A) => B): ReadonlyArray<B> {
  const result: B[] = [];
  for (const item of arr) {
    result.push(f(item));
  }
  return result;
}
```

### 문제 2: 참조 투명성 검증

다음 코드에서 `x`를 해당 표현식으로 치환했을 때 결과가 같은지 확인하라.

```typescript
// 케이스 A
const x = [1, 2, 3].map(n => n * 2);
console.log(x.length + x.length);
// vs
console.log([1, 2, 3].map(n => n * 2).length + [1, 2, 3].map(n => n * 2).length);

// 케이스 B
const arr = [3, 1, 2];
const x2 = arr.sort();
console.log(x2[0] + arr[0]);
// vs
console.log(arr.sort()[0] + arr[0]);
```

### 문제 3: 순수 함수로 리팩토링

다음 비순수 함수를 순수 함수로 리팩토링하라.

```typescript
let nextId = 1;

function createUser(name: string, email: string): User {
  const user = {
    id: nextId++,
    name,
    email,
    createdAt: new Date(),
  };
  console.log(`Created user: ${name}`);
  return user;
}
```

### 문제 4: 전사·단사·전단사 분류

다음 함수의 유형을 분류하라 (정의역과 공역은 TypeScript 타입 기준).

```typescript
// (a) number → number
const square = (n: number) => n * n;

// (b) string → number
const length = (s: string) => s.length;

// (c) boolean → boolean
const not = (b: boolean) => !b;

// (d) number → string
const toString = (n: number) => String(n);
```

---

## 연습 문제 해답

### 문제 1 해답

**(a) `greet` — ✅ 순수 함수**
같은 `name`에 항상 같은 문자열을 반환하며 부수 효과가 없다.

**(b) `formalGreet` — ⚠️ 조건부 순수**
`config`가 절대 변하지 않는다면 사실상 순수하다. 하지만 `config.prefix`가 외부에서 변경될 수 있으므로 엄밀히는 순수하지 않다. 외부 가변 상태에 의존하기 때문이다. `config`를 `const config = Object.freeze({ prefix: "Mr." })`로 만들거나 인자로 받으면 순수해진다.

**(c) `shuffle` — ❌ 순수하지 않음**
`Math.random()`을 사용하므로 비결정론적이다. 같은 배열을 넣어도 매번 다른 결과가 나온다. 원본 배열을 변경하지는 않지만(복사 후 조작), 결정론성 조건을 위반한다.

**(d) `head` — ❌ 순수하지 않음 (엄밀한 정의에서)**
빈 배열에 대해 예외를 던진다. 예외는 부수 효과이며, 함수의 반환 타입(`T`)에 표현되지 않는 탈출 경로를 만든다. 정의역(모든 `T[]`)에 대해 완전하지 않다. 순수하게 만들려면 반환 타입을 `T | undefined`나 `Option<T>`로 바꿔야 한다.

**(e) `map` — ✅ 순수 함수 (f가 순수하다면)**
함수 내부에서 `result` 배열을 만들고 변경하지만, 이것은 **지역 변이(local mutation)** 다. 외부에서 관찰할 수 없으므로 순수 함수의 정의를 만족한다. 다만 인자로 받는 `f`가 순수해야 `map` 전체도 순수하다.

### 문제 2 해답

**케이스 A — ✅ 참조 투명**
`[1, 2, 3].map(n => n * 2)`는 순수하므로 항상 `[2, 4, 6]`을 반환한다. 치환해도 `.length`는 3이고 결과는 6이다.

**케이스 B — ❌ 참조 투명하지 않음**
`Array.prototype.sort()`는 **원본 배열을 변경(in-place mutation)** 한다.

```typescript
const arr = [3, 1, 2];
const x2 = arr.sort();      // arr = [1, 2, 3], x2 = [1, 2, 3] (같은 참조)
console.log(x2[0] + arr[0]); // 1 + 1 = 2

// 치환하면:
const arr2 = [3, 1, 2];
console.log(arr2.sort()[0] + arr2[0]); // arr2.sort()가 arr2를 변경 → 1 + 1 = 2
// 이 경우 결과가 같아 보이지만, 핵심 문제는 arr이 변경된다는 것이다
// 더 명확한 예:
// arr.sort()[0] → 1 (sort 후 arr = [1,2,3])
// arr[0] → 1 (이미 정렬됨)
// 하지만 원본이 바뀌므로 이후 arr을 사용하는 코드에 영향을 준다
```

`sort()`의 변이 문제를 피하려면 `[...arr].sort()` 또는 `arr.toSorted()`(ES2023)를 사용한다.

### 문제 3 해답

```typescript
// 순수 함수 버전
interface CreateUserInput {
  readonly name: string;
  readonly email: string;
  readonly id: number;
  readonly createdAt: Date;
}

function createUser(input: CreateUserInput): User {
  return {
    id: input.id,
    name: input.name,
    email: input.email,
    createdAt: input.createdAt,
  };
}

// 부수 효과를 호출자에게 위임
// ID 생성, 시간, 로깅은 순수 함수 바깥에서 처리
const user = createUser({
  name: "Alice",
  email: "alice@example.com",
  id: generateId(),        // 효과: ID 생성
  createdAt: new Date(),   // 효과: 현재 시간
});
console.log(`Created user: ${user.name}`); // 효과: 로깅
```

핵심: `id`, `createdAt` 같은 비결정론적 값과 `console.log` 같은 부수 효과를 함수 바깥으로 밀어냈다. `createUser`는 이제 입력이 같으면 항상 같은 결과를 반환한다.

### 문제 4 해답

**(a) `square: number → number` — 단사 아님, 전사 아님**
- 단사 아님: `square(3) = square(-3) = 9`
- 전사 아님: 음수는 출력될 수 없음 (`square(x) >= 0`)

**(b) `length: string → number` — 단사 아님, 전사 아님**
- 단사 아님: `length("ab") = length("cd") = 2`
- 전사 아님: 음수나 소수는 출력될 수 없음

**(c) `not: boolean → boolean` — 전단사**
- 단사: `not(true) ≠ not(false)` (false ≠ true)
- 전사: `not(true) = false`, `not(false) = true` — 공역 `{true, false}` 모두 커버
- 역함수: `not` 자체가 역함수 (`not(not(x)) = x`)

**(d) `toString: number → string` — 단사, 전사 아님**
- 단사: 서로 다른 숫자는 서로 다른 문자열로 변환됨
- 전사 아님: `"hello"` 같은 문자열은 어떤 숫자의 `String()` 결과도 아님

---

## 핵심 정리

| 개념 | 한 줄 요약 |
|------|-----------|
| 수학적 함수 | 정의역의 각 원소를 공역의 정확히 하나의 원소에 대응 |
| 순수 함수 | 같은 입력 → 같은 출력, 부수 효과 없음 |
| 참조 투명성 | 표현식을 그 값으로 치환해도 프로그램이 동일하게 동작 |
| 부수 효과 | 반환값 외의 모든 관찰 가능한 변화 |
| 단사 | 다른 입력 → 다른 출력 (정보 보존) |
| 전사 | 공역의 모든 원소가 출력됨 |
| 전단사 | 1:1 대응 — 역함수가 존재 |

## 다음 주차 예고

**2주차: 일급 함수와 고차 함수** — 함수를 값으로 다루고, 함수를 받고 반환하는 고차 함수의 세계로 진입한다. 커링과 부분 적용을 통해 함수를 더 유연하게 합성하는 방법을 배운다.
