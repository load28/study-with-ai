# 2주차: 일급 함수와 고차 함수

## 왜 이것을 배우는가?

1주차에서 수학적 함수와 순수 함수를 배웠다. 하지만 함수형 프로그래밍의 진정한 힘은 **함수 자체를 값으로 다룰 수 있다**는 데서 나온다.

명령형 프로그래밍에서는 데이터(숫자, 문자열, 객체)를 변수에 담고, 인자로 전달하고, 반환한다. 함수형 프로그래밍에서는 **함수도 똑같이** 한다. 함수를 변수에 담고, 다른 함수에 인자로 넘기고, 함수에서 함수를 반환한다. 이것이 가능하면 코드의 추상화 수준이 완전히 달라진다.

이번 주에는 이 개념을 체계적으로 이해하고, 커링(Currying)과 부분 적용(Partial Application)을 통해 함수를 조립하는 방법까지 배운다.

---

## 1. 일급 시민으로서의 함수 (First-class Function)

### 1.1 일급 시민의 조건

프로그래밍 언어에서 어떤 것이 **일급 시민(First-class Citizen)** 이라는 것은 다음 조건을 만족한다는 뜻이다:

1. **변수에 할당**할 수 있다
2. **함수의 인자**로 전달할 수 있다
3. **함수의 반환값**으로 사용할 수 있다
4. **자료구조에 저장**할 수 있다

JavaScript/TypeScript에서 숫자, 문자열, 객체는 당연히 일급 시민이다. 그리고 **함수도 일급 시민**이다.

```typescript
// 1. 변수에 할당
const add = (a: number, b: number): number => a + b;

// 2. 함수의 인자로 전달
const numbers = [3, 1, 4, 1, 5];
numbers.filter((n) => n > 2); // [3, 4, 5]

// 3. 함수의 반환값으로 사용
function createGreeter(greeting: string): (name: string) => string {
  return (name) => `${greeting}, ${name}!`;
}

// 4. 자료구조에 저장
const operations: Record<string, (a: number, b: number) => number> = {
  add: (a, b) => a + b,
  sub: (a, b) => a - b,
  mul: (a, b) => a * b,
};
```

### 1.2 일급 함수가 아닌 언어에서의 비교

일급 함수가 없으면 같은 로직을 반복해야 한다:

```typescript
// 일급 함수가 없다고 가정하면...
// 배열에서 짝수만 걸러내기
function filterEven(arr: number[]): number[] {
  const result: number[] = [];
  for (const n of arr) {
    if (n % 2 === 0) result.push(n);
  }
  return result;
}

// 배열에서 양수만 걸러내기
function filterPositive(arr: number[]): number[] {
  const result: number[] = [];
  for (const n of arr) {
    if (n > 0) result.push(n);
  }
  return result;
}

// 배열에서 10보다 큰 것만 걸러내기
function filterGreaterThan10(arr: number[]): number[] {
  const result: number[] = [];
  for (const n of arr) {
    if (n > 10) result.push(n);
  }
  return result;
}
// 루프 구조가 매번 반복된다!
```

```typescript
// 일급 함수가 있으면: 조건을 함수로 추상화
function filter<T>(arr: T[], predicate: (item: T) => boolean): T[] {
  const result: T[] = [];
  for (const item of arr) {
    if (predicate(item)) result.push(item);
  }
  return result;
}

// 한 번 만들어놓으면 조건만 바꿔서 사용
filter(numbers, (n) => n % 2 === 0);   // 짝수
filter(numbers, (n) => n > 0);          // 양수
filter(numbers, (n) => n > 10);         // 10 초과
```

**핵심**: 일급 함수는 **행위(behavior)를 추상화**할 수 있게 한다. 데이터뿐만 아니라 "무엇을 할지"도 파라미터화할 수 있다.

### 1.3 함수 타입 표기

TypeScript에서 함수 타입을 명시적으로 표기하는 것은 함수형 프로그래밍에서 중요하다. 타입이 함수의 의도를 설명하기 때문이다.

```typescript
// 함수 타입 표기 방법들
type Predicate<T> = (value: T) => boolean;
type Mapper<A, B> = (value: A) => B;
type Reducer<T, R> = (accumulator: R, current: T) => R;
type Comparator<T> = (a: T, b: T) => number;

// 타입을 사용한 함수 시그니처
function filter<T>(arr: T[], pred: Predicate<T>): T[] {
  return arr.filter(pred);
}

function map<A, B>(arr: A[], f: Mapper<A, B>): B[] {
  return arr.map(f);
}

function reduce<T, R>(arr: T[], f: Reducer<T, R>, init: R): R {
  return arr.reduce(f, init);
}
```

---

## 2. 고차 함수 (Higher-order Function)

### 2.1 정의

**고차 함수**는 다음 중 하나 이상을 만족하는 함수다:

1. **함수를 인자로 받는다**
2. **함수를 반환한다**

수학에서의 대응: 함수 공간(Function Space). `A → B`가 집합이면, 그 집합의 원소(함수)를 인자로 받거나 반환하는 것도 자연스럽다.

### 2.2 함수를 인자로 받는 고차 함수

가장 기본적인 세 가지: `map`, `filter`, `reduce`.

```typescript
const users: ReadonlyArray<{ name: string; age: number; active: boolean }> = [
  { name: "Alice", age: 30, active: true },
  { name: "Bob", age: 17, active: true },
  { name: "Charlie", age: 25, active: false },
  { name: "Diana", age: 22, active: true },
];

// map: 각 원소를 변환
const names = users.map((u) => u.name);
// ["Alice", "Bob", "Charlie", "Diana"]

// filter: 조건에 맞는 원소만 선택
const activeAdults = users.filter((u) => u.age >= 18 && u.active);
// [{ name: "Alice", ... }, { name: "Diana", ... }]

// reduce: 전체를 하나의 값으로 접기
const totalAge = users.reduce((sum, u) => sum + u.age, 0);
// 94
```

이 세 함수의 공통점: **반복의 구조(어떻게)를 숨기고 변환의 의미(무엇을)만 드러낸다.**

```typescript
// 명령형: "어떻게" 반복하는지가 전면에
const result: string[] = [];
for (let i = 0; i < users.length; i++) {
  if (users[i].active && users[i].age >= 18) {
    result.push(users[i].name);
  }
}

// 함수형: "무엇을" 하는지가 전면에
const result2 = users
  .filter((u) => u.active && u.age >= 18)
  .map((u) => u.name);
```

### 2.3 함수를 반환하는 고차 함수

함수를 반환하는 것은 **행위를 생성**하는 것이다. 설정에 따라 다르게 동작하는 함수를 만들 수 있다.

```typescript
// 비교 함수 생성기
function compareBy<T, K extends string | number>(
  key: (item: T) => K,
): Comparator<T> {
  return (a, b) => {
    const ka = key(a);
    const kb = key(b);
    return ka < kb ? -1 : ka > kb ? 1 : 0;
  };
}

// 사용: 정렬 기준을 동적으로 생성
const byAge = compareBy((u: typeof users[number]) => u.age);
const byName = compareBy((u: typeof users[number]) => u.name);

[...users].sort(byAge);   // 나이순 정렬
[...users].sort(byName);  // 이름순 정렬
```

```typescript
// 유효성 검증기 생성기
type Validator<T> = (value: T) => string | null;

function minLength(min: number): Validator<string> {
  return (value) =>
    value.length < min ? `최소 ${min}자 이상이어야 합니다` : null;
}

function maxLength(max: number): Validator<string> {
  return (value) =>
    value.length > max ? `최대 ${max}자 이하여야 합니다` : null;
}

function matches(pattern: RegExp, message: string): Validator<string> {
  return (value) => (pattern.test(value) ? null : message);
}

// 검증기 합성
function composeValidators<T>(...validators: Validator<T>[]): Validator<T> {
  return (value) => {
    for (const validate of validators) {
      const error = validate(value);
      if (error) return error;
    }
    return null;
  };
}

// 사용
const validateUsername = composeValidators(
  minLength(3),
  maxLength(20),
  matches(/^[a-zA-Z0-9_]+$/, "영문, 숫자, 밑줄만 사용 가능합니다"),
);

validateUsername("ab");          // "최소 3자 이상이어야 합니다"
validateUsername("valid_user");  // null (통과)
validateUsername("no spaces!");  // "영문, 숫자, 밑줄만 사용 가능합니다"
```

### 2.4 직접 구현해보기: map, filter, reduce

라이브러리가 제공하는 것을 쓰기 전에 직접 구현해봐야 고차 함수의 본질을 이해할 수 있다.

```typescript
// map 구현
function myMap<A, B>(arr: ReadonlyArray<A>, f: (a: A) => B): B[] {
  const result: B[] = [];
  for (const item of arr) {
    result.push(f(item));
  }
  return result;
}

// filter 구현
function myFilter<T>(
  arr: ReadonlyArray<T>,
  pred: (item: T) => boolean,
): T[] {
  const result: T[] = [];
  for (const item of arr) {
    if (pred(item)) result.push(item);
  }
  return result;
}

// reduce 구현
function myReduce<T, R>(
  arr: ReadonlyArray<T>,
  f: (acc: R, item: T) => R,
  init: R,
): R {
  let acc = init;
  for (const item of arr) {
    acc = f(acc, item);
  }
  return acc;
}

// reduce로 map과 filter를 구현할 수 있다 — reduce가 가장 일반적이다
function mapViaReduce<A, B>(arr: ReadonlyArray<A>, f: (a: A) => B): B[] {
  return myReduce(arr, (acc: B[], item) => [...acc, f(item)], []);
}

function filterViaReduce<T>(
  arr: ReadonlyArray<T>,
  pred: (item: T) => boolean,
): T[] {
  return myReduce(
    arr,
    (acc: T[], item) => (pred(item) ? [...acc, item] : acc),
    [],
  );
}
```

**핵심 통찰**: `reduce`는 가장 일반적인 반복 추상화다. `map`과 `filter`는 `reduce`의 특수한 경우다. 이것은 나중에 배울 `Foldable`(14주차)의 기초가 된다.

---

## 3. 클로저 (Closure)와 렉시컬 스코프

### 3.1 렉시컬 스코프란

변수의 유효 범위가 **코드가 작성된 위치**에 의해 결정되는 것을 렉시컬 스코프(Lexical Scope, 또는 정적 스코프)라 한다.

```typescript
const outer = "나는 바깥 변수";

function printOuter(): void {
  // 이 함수가 어디서 호출되든, outer는 정의된 위치의 스코프를 참조한다
  console.log(outer);
}

function wrapper(): void {
  const outer = "나는 wrapper 안의 변수"; // 이 변수가 아니라
  printOuter(); // "나는 바깥 변수" — 정의된 시점의 outer를 참조
}
```

### 3.2 클로저의 정의

**클로저(Closure)** 는 함수와 그 함수가 정의된 렉시컬 환경(Lexical Environment)의 조합이다. 함수가 자신이 생성된 스코프의 변수를 "기억"하는 것이다.

```typescript
function createCounter(start: number): {
  get: () => number;
  increment: () => void;
} {
  let count = start; // 이 변수가 클로저에 의해 캡처된다

  return {
    get: () => count,          // count를 읽음
    increment: () => { count++; }, // count를 수정
  };
}

const counter = createCounter(0);
counter.get();       // 0
counter.increment();
counter.get();       // 1
// count는 외부에서 직접 접근 불가 — 캡슐화
```

### 3.3 함수형 프로그래밍에서의 클로저

함수형 프로그래밍에서 클로저는 주로 **설정을 캡처하여 특화된 함수를 만드는 데** 사용된다. 위의 가변 카운터와 달리, 순수 함수형에서는 캡처하는 값이 불변인 경우가 대부분이다.

```typescript
// 설정을 캡처하는 순수한 클로저
function createFormatter(locale: string, currency: string) {
  // locale과 currency가 클로저에 캡처됨 (불변)
  const formatter = new Intl.NumberFormat(locale, {
    style: "currency",
    currency,
  });

  return (amount: number): string => formatter.format(amount);
}

const formatKRW = createFormatter("ko-KR", "KRW");
const formatUSD = createFormatter("en-US", "USD");

formatKRW(10000);  // "₩10,000"
formatUSD(99.99);  // "$99.99"
// 같은 amount를 넣으면 항상 같은 결과 — 순수 함수
```

```typescript
// 클로저로 "환경"을 캡처하는 패턴
function createLogger(prefix: string) {
  return (message: string): string => `[${prefix}] ${message}`;
}

const authLog = createLogger("AUTH");
const dbLog = createLogger("DB");

authLog("login attempt");  // "[AUTH] login attempt"
dbLog("query executed");   // "[DB] query executed"
```

### 3.4 클로저의 함정: 가변 변수 캡처

클로저가 가변 변수를 캡처하면 예상치 못한 결과가 생길 수 있다.

```typescript
// ❌ 클래식한 실수: var와 클로저
function createHandlers(): (() => void)[] {
  const handlers: (() => void)[] = [];
  for (var i = 0; i < 3; i++) {
    handlers.push(() => console.log(i));
  }
  return handlers;
}

const handlers = createHandlers();
handlers[0](); // 3 (0이 아님!)
handlers[1](); // 3
handlers[2](); // 3
// 모든 클로저가 같은 i를 참조하고, 루프 종료 후 i = 3

// ✅ 해결 1: let (블록 스코프)
function createHandlersFixed(): (() => void)[] {
  const handlers: (() => void)[] = [];
  for (let i = 0; i < 3; i++) {
    handlers.push(() => console.log(i)); // 각 반복마다 새로운 i
  }
  return handlers;
}

// ✅ 해결 2: 함수형 — 인덱스를 즉시 바인딩
const handlers2 = [0, 1, 2].map((i) => () => console.log(i));
```

**교훈**: 클로저가 캡처하는 값은 가능하면 불변이어야 한다. 함수형 프로그래밍에서는 이 문제가 자연스럽게 해소된다.

---

## 4. 커링 (Currying)

### 4.1 정의

**커링(Currying)** 은 여러 인자를 받는 함수를 **하나의 인자만 받는 함수의 연쇄**로 변환하는 것이다.

수학적으로: `f: (A, B) → C`를 `f': A → (B → C)`로 변환한다.

```typescript
// 일반 함수: 두 인자를 한번에 받음
function add(a: number, b: number): number {
  return a + b;
}
add(2, 3); // 5

// 커링된 함수: 하나씩 받음
function curriedAdd(a: number): (b: number) => number {
  return (b) => a + b;
}
curriedAdd(2)(3); // 5

// 화살표 함수로 간결하게
const curriedAdd2 = (a: number) => (b: number): number => a + b;
```

### 4.2 왜 커링이 유용한가

커링의 핵심 가치는 **부분 적용(Partial Application)** 이 자연스럽게 된다는 것이다.

```typescript
const add = (a: number) => (b: number): number => a + b;

// 부분 적용: 첫 번째 인자만 넣어서 새 함수 생성
const add10 = add(10);  // (b: number) => 10 + b
const add100 = add(100);

add10(5);   // 15
add100(5);  // 105

// map과 함께 사용
[1, 2, 3].map(add(10)); // [11, 12, 13]
```

```typescript
// 실전 예시: API 엔드포인트 빌더
const buildUrl =
  (baseUrl: string) =>
  (path: string) =>
  (params: Record<string, string>): string => {
    const query = new URLSearchParams(params).toString();
    return `${baseUrl}${path}${query ? `?${query}` : ""}`;
  };

// 환경별 API 클라이언트 생성
const devApi = buildUrl("http://localhost:3000");
const prodApi = buildUrl("https://api.example.com");

// 리소스별 URL 생성기
const devUsers = devApi("/api/users");
const prodUsers = prodApi("/api/users");

devUsers({ page: "1", limit: "10" });
// "http://localhost:3000/api/users?page=1&limit=10"

prodUsers({ page: "1", limit: "10" });
// "https://api.example.com/api/users?page=1&limit=10"
```

### 4.3 범용 curry 함수 구현

직접 커링된 형태로 함수를 작성할 수도 있지만, 기존 함수를 자동으로 커링하는 유틸리티를 만들 수 있다.

```typescript
// 2인자 함수 커링
function curry2<A, B, C>(f: (a: A, b: B) => C): (a: A) => (b: B) => C {
  return (a) => (b) => f(a, b);
}

// 3인자 함수 커링
function curry3<A, B, C, D>(
  f: (a: A, b: B, c: C) => D,
): (a: A) => (b: B) => (c: C) => D {
  return (a) => (b) => (c) => f(a, b, c);
}

// 사용 예시
const multiply = (a: number, b: number): number => a * b;
const curriedMultiply = curry2(multiply);

const double = curriedMultiply(2);
const triple = curriedMultiply(3);

[1, 2, 3].map(double); // [2, 4, 6]
[1, 2, 3].map(triple); // [3, 6, 9]
```

### 4.4 커링 vs 부분 적용

커링과 부분 적용은 관련이 깊지만 다른 개념이다.

```typescript
// 커링: 인자를 하나씩 받는 함수로 변환
// f(a, b, c) → f(a)(b)(c)
const curriedFn = (a: number) => (b: number) => (c: number): number =>
  a + b + c;

// 부분 적용: 일부 인자를 미리 고정
// f(a, b, c) → f_with_a(b, c)
function partial<A, B, C, R>(
  f: (a: A, b: B, c: C) => R,
  a: A,
): (b: B, c: C) => R {
  return (b, c) => f(a, b, c);
}

const add3 = (a: number, b: number, c: number) => a + b + c;
const add10And = partial(add3, 10);
add10And(20, 30); // 60

// 커링된 함수는 부분 적용이 자연스럽다
const add10AndCurried = curriedFn(10);
add10AndCurried(20)(30); // 60
```

**정리**:
- **커링**: 함수의 **형태를 변환**한다 (`(a, b, c) → R`을 `a → b → c → R`로)
- **부분 적용**: 인자의 **일부를 미리 채운** 새 함수를 만든다

커링된 함수에서는 부분 적용이 그냥 함수 호출이다. 이것이 커링의 편리함이다.

### 4.5 인자 순서의 중요성

커링할 때 인자 순서가 매우 중요하다. **가장 변하기 어려운(설정, 환경) 인자를 앞에**, **가장 자주 변하는(데이터) 인자를 뒤에** 놓아야 부분 적용이 유용하다.

```typescript
// ❌ 나쁜 인자 순서: 데이터가 앞에 있으면 부분 적용이 어색하다
const badFilter = (arr: number[]) => (pred: (n: number) => boolean) =>
  arr.filter(pred);
// badFilter([1,2,3])(n => n > 1) — 배열을 먼저 넣어야 함

// ✅ 좋은 인자 순서: 설정/행위가 앞, 데이터가 뒤
const goodFilter = (pred: (n: number) => boolean) => (arr: number[]) =>
  arr.filter(pred);

const positives = goodFilter((n) => n > 0);
const evens = goodFilter((n) => n % 2 === 0);

positives([1, -2, 3, -4]); // [1, 3]
evens([1, 2, 3, 4]);       // [2, 4]
// 필터 조건을 먼저 고정하고, 다양한 배열에 재사용
```

이 원칙은 fp-ts나 Ramda 같은 함수형 라이브러리에서 일관되게 적용되는 "**data-last**" 패턴이다.

---

## 5. 실전 패턴: 고차 함수 활용

### 5.1 이벤트 핸들러 팩토리

React에서 이벤트 핸들러를 만들 때 고차 함수는 매우 유용하다.

```typescript
// ❌ 흔한 문제: 핸들러 안에서 인라인 함수 생성
function TodoList({ todos, onToggle, onDelete }: TodoListProps) {
  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          {/* 매 렌더링마다 새 함수 생성 → 최적화 방해 */}
          <input
            type="checkbox"
            onChange={() => onToggle(todo.id)}
          />
          <span>{todo.text}</span>
          <button onClick={() => onDelete(todo.id)}>삭제</button>
        </li>
      ))}
    </ul>
  );
}
```

```typescript
// ✅ 고차 함수로 핸들러 팩토리 패턴
// 커링된 핸들러를 미리 만들어서 안정적인 참조를 유지
const createToggleHandler =
  (onToggle: (id: string) => void) =>
  (id: string) =>
  (): void => {
    onToggle(id);
  };

const createDeleteHandler =
  (onDelete: (id: string) => void) =>
  (id: string) =>
  (): void => {
    onDelete(id);
  };

function TodoList({ todos, onToggle, onDelete }: TodoListProps) {
  const handleToggle = createToggleHandler(onToggle);
  const handleDelete = createDeleteHandler(onDelete);

  return (
    <ul>
      {todos.map((todo) => (
        <li key={todo.id}>
          <input type="checkbox" onChange={handleToggle(todo.id)} />
          <span>{todo.text}</span>
          <button onClick={handleDelete(todo.id)}>삭제</button>
        </li>
      ))}
    </ul>
  );
}
```

```typescript
// 더 일반적인 패턴: 이벤트 핸들러 팩토리
function handleFieldChange(
  setState: React.Dispatch<React.SetStateAction<FormData>>,
) {
  return (field: keyof FormData) =>
    (e: React.ChangeEvent<HTMLInputElement>): void => {
      setState((prev) => ({ ...prev, [field]: e.target.value }));
    };
}

function SignupForm() {
  const [form, setForm] = useState<FormData>({
    name: "",
    email: "",
    password: "",
  });

  const onChange = handleFieldChange(setForm);

  return (
    <form>
      <input value={form.name} onChange={onChange("name")} />
      <input value={form.email} onChange={onChange("email")} />
      <input value={form.password} onChange={onChange("password")} />
    </form>
  );
}
```

### 5.2 설정 가능한 API 클라이언트

```typescript
// 고차 함수로 API 클라이언트 구성
interface ApiConfig {
  readonly baseUrl: string;
  readonly headers: Record<string, string>;
  readonly timeout: number;
}

type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";

const createApiClient = (config: ApiConfig) => {
  const request =
    (method: HttpMethod) =>
    <T>(path: string, body?: unknown): Promise<T> =>
      fetch(`${config.baseUrl}${path}`, {
        method,
        headers: {
          "Content-Type": "application/json",
          ...config.headers,
        },
        body: body ? JSON.stringify(body) : undefined,
        signal: AbortSignal.timeout(config.timeout),
      }).then((res) => {
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return res.json();
      });

  return {
    get: request("GET"),
    post: request("POST"),
    put: request("PUT"),
    delete: request("DELETE"),
  };
};

// 환경별 클라이언트 생성
const devClient = createApiClient({
  baseUrl: "http://localhost:3000",
  headers: {},
  timeout: 5000,
});

const prodClient = createApiClient({
  baseUrl: "https://api.example.com",
  headers: { Authorization: "Bearer token" },
  timeout: 10000,
});

// 사용
const users = await devClient.get<User[]>("/api/users");
const newUser = await prodClient.post<User>("/api/users", {
  name: "Alice",
});
```

### 5.3 미들웨어 패턴

```typescript
// 함수를 감싸서 기능을 추가하는 미들웨어 패턴
type AsyncFn<A, B> = (arg: A) => Promise<B>;

// 로깅 미들웨어
function withLogging<A, B>(
  label: string,
  fn: AsyncFn<A, B>,
): AsyncFn<A, B> {
  return async (arg) => {
    console.log(`[${label}] 시작`, arg);
    try {
      const result = await fn(arg);
      console.log(`[${label}] 성공`, result);
      return result;
    } catch (error) {
      console.error(`[${label}] 실패`, error);
      throw error;
    }
  };
}

// 재시도 미들웨어
function withRetry<A, B>(
  retries: number,
  fn: AsyncFn<A, B>,
): AsyncFn<A, B> {
  return async (arg) => {
    let lastError: unknown;
    for (let i = 0; i <= retries; i++) {
      try {
        return await fn(arg);
      } catch (error) {
        lastError = error;
        if (i < retries) {
          await new Promise((r) => setTimeout(r, 2 ** i * 1000));
        }
      }
    }
    throw lastError;
  };
}

// 캐싱 미들웨어
function withCache<A extends string | number, B>(
  ttlMs: number,
  fn: AsyncFn<A, B>,
): AsyncFn<A, B> {
  const cache = new Map<A, { value: B; expiry: number }>();
  return async (arg) => {
    const cached = cache.get(arg);
    if (cached && Date.now() < cached.expiry) {
      return cached.value;
    }
    const result = await fn(arg);
    cache.set(arg, { value: result, expiry: Date.now() + ttlMs });
    return result;
  };
}

// 미들웨어 합성: 안쪽부터 바깥으로 감싸기
const fetchUser: AsyncFn<string, User> = (id) =>
  fetch(`/api/users/${id}`).then((r) => r.json());

const enhancedFetchUser = withLogging(
  "fetchUser",
  withRetry(3, withCache(60000, fetchUser)),
);
// 실행 순서: 로깅 → 재시도 → 캐시 → 원본 fetch
```

### 5.4 HOC (Higher-Order Component) 패턴

React의 Higher-Order Component는 고차 함수의 컴포넌트 버전이다.

```typescript
// HOC: 컴포넌트를 받아 강화된 컴포넌트를 반환하는 함수
function withAuth<P extends object>(
  WrappedComponent: React.ComponentType<P>,
): React.FC<P> {
  return function AuthenticatedComponent(props: P) {
    const { user, isLoading } = useAuth();

    if (isLoading) return <Loading />;
    if (!user) return <Navigate to="/login" />;

    return <WrappedComponent {...props} />;
  };
}

// HOC: 에러 바운더리 래핑
function withErrorBoundary<P extends object>(
  WrappedComponent: React.ComponentType<P>,
  fallback: React.ReactNode,
): React.FC<P> {
  return function WithErrorBoundary(props: P) {
    return (
      <ErrorBoundary fallback={fallback}>
        <WrappedComponent {...props} />
      </ErrorBoundary>
    );
  };
}

// HOC 합성
const DashboardPage = withAuth(
  withErrorBoundary(Dashboard, <ErrorFallback />),
);
```

> **참고**: 현대 React에서는 HOC보다 커스텀 Hook이 선호되지만, HOC는 고차 함수의 개념을 이해하는 좋은 예시이며 여전히 횡단 관심사(cross-cutting concerns)를 처리하는 데 유용하다.

---

## 6. 고차 함수의 타입 안전성

TypeScript에서 고차 함수를 작성할 때 제네릭을 활용하면 타입 안전성을 유지할 수 있다.

```typescript
// 제네릭 고차 함수: 타입이 전파된다
function pipe2<A, B, C>(f: (a: A) => B, g: (b: B) => C): (a: A) => C {
  return (a) => g(f(a));
}

const parseAndDouble = pipe2(
  (s: string) => parseInt(s, 10),  // string → number
  (n: number) => n * 2,            // number → number
);
// parseAndDouble의 타입: (s: string) => number ✓

// 타입 에러가 자동 감지된다
const invalid = pipe2(
  (s: string) => parseInt(s, 10),  // string → number
  (s: string) => s.toUpperCase(),  // string → string ← number를 기대하는데!
  // TypeScript 에러: 'number'는 'string'에 할당할 수 없습니다
);
```

```typescript
// 실전 타입 패턴: 함수 오버로드와 고차 함수
type EventMap = {
  click: { x: number; y: number };
  keydown: { key: string; code: string };
  submit: { formData: FormData };
};

function on<K extends keyof EventMap>(
  event: K,
  handler: (payload: EventMap[K]) => void,
): () => void {
  // 이벤트 리스너 등록 로직...
  return () => {
    // 구독 해제
  };
}

// 타입 안전한 이벤트 핸들링
on("click", (payload) => {
  // payload는 자동으로 { x: number; y: number }
  console.log(payload.x, payload.y);
});

on("keydown", (payload) => {
  // payload는 자동으로 { key: string; code: string }
  console.log(payload.key);
});
```

---

## 연습 문제

### 문제 1: 고차 함수 작성

다음 요구사항에 맞는 고차 함수를 작성하라.

```typescript
// (a) once: 함수를 한 번만 실행되도록 감싸는 고차 함수
// 두 번째 이후 호출에서는 첫 번째 호출의 결과를 반환
declare function once<A extends unknown[], R>(
  fn: (...args: A) => R,
): (...args: A) => R;

// 기대 동작:
const initialize = once(() => {
  console.log("초기화!");
  return 42;
});
initialize(); // "초기화!" 출력, 42 반환
initialize(); // 아무것도 출력 안 함, 42 반환
```

```typescript
// (b) debounce: 마지막 호출 후 delay ms 뒤에 실행
declare function debounce<A extends unknown[]>(
  fn: (...args: A) => void,
  delay: number,
): (...args: A) => void;
```

```typescript
// (c) negate: 조건 함수의 결과를 반전시키는 고차 함수
declare function negate<T>(pred: (value: T) => boolean): (value: T) => boolean;

// 기대 동작:
const isEven = (n: number) => n % 2 === 0;
const isOdd = negate(isEven);
isOdd(3); // true
isOdd(4); // false
```

### 문제 2: 커링 변환

다음 함수들을 커링된 형태로 변환하고, 부분 적용의 예시를 작성하라.

```typescript
// (a)
function formatPrice(currency: string, decimals: number, amount: number): string {
  return `${currency}${amount.toFixed(decimals)}`;
}
// formatPrice("$", 2, 19.99) → "$19.99"

// (b)
function between(min: number, max: number, value: number): boolean {
  return value >= min && value <= max;
}
// between(0, 100, 50) → true
```

### 문제 3: 클로저 동작 예측

다음 코드의 출력을 예측하고, 그 이유를 설명하라.

```typescript
function createFunctions(): Array<() => number> {
  const fns: Array<() => number> = [];
  for (let i = 0; i < 5; i++) {
    const captured = i * 10;
    fns.push(() => captured);
  }
  return fns;
}

const functions = createFunctions();
console.log(functions[0]()); // ?
console.log(functions[2]()); // ?
console.log(functions[4]()); // ?
```

### 문제 4: 미들웨어 합성

다음 요구사항을 만족하는 `compose` 함수를 작성하라.

```typescript
// 여러 미들웨어를 하나로 합성하는 함수
// compose(f, g, h)(x) === f(g(h(x)))
declare function compose<T>(...fns: Array<(arg: T) => T>): (arg: T) => T;

// 테스트:
const addExclamation = (s: string) => s + "!";
const toUpper = (s: string) => s.toUpperCase();
const trim = (s: string) => s.trim();

const process = compose(addExclamation, toUpper, trim);
process("  hello  "); // "HELLO!"
```

---

## 연습 문제 해답

### 문제 1 해답

**(a) once 구현**

```typescript
function once<A extends unknown[], R>(
  fn: (...args: A) => R,
): (...args: A) => R {
  let called = false;
  let result: R;

  return (...args: A): R => {
    if (!called) {
      result = fn(...args);
      called = true;
    }
    return result;
  };
}
```

클로저로 `called`와 `result`를 캡처한다. 첫 호출에서만 원본 함수를 실행하고, 이후에는 캐시된 결과를 반환한다.

**(b) debounce 구현**

```typescript
function debounce<A extends unknown[]>(
  fn: (...args: A) => void,
  delay: number,
): (...args: A) => void {
  let timerId: ReturnType<typeof setTimeout> | null = null;

  return (...args: A): void => {
    if (timerId !== null) {
      clearTimeout(timerId);
    }
    timerId = setTimeout(() => {
      fn(...args);
      timerId = null;
    }, delay);
  };
}
```

클로저로 `timerId`를 캡처한다. 새 호출마다 이전 타이머를 취소하고 새 타이머를 설정한다. `delay` ms 동안 추가 호출이 없으면 원본 함수가 실행된다.

**(c) negate 구현**

```typescript
function negate<T>(pred: (value: T) => boolean): (value: T) => boolean {
  return (value: T) => !pred(value);
}
```

고차 함수의 가장 단순한 형태. 원본 함수의 결과를 반전시키는 새 함수를 반환한다. `pred`가 순수하면 `negate(pred)`도 순수하다.

### 문제 2 해답

**(a) formatPrice 커링**

```typescript
const formatPrice =
  (currency: string) =>
  (decimals: number) =>
  (amount: number): string =>
    `${currency}${amount.toFixed(decimals)}`;

// 부분 적용 예시
const formatUSD = formatPrice("$")(2);
const formatKRW = formatPrice("₩")(0);

formatUSD(19.99);   // "$19.99"
formatUSD(100);     // "$100.00"
formatKRW(15000);   // "₩15000"

// map과 함께 사용
[19.99, 29.99, 39.99].map(formatUSD);
// ["$19.99", "$29.99", "$39.99"]
```

인자 순서: 통화 → 소수점 → 금액. 통화와 소수점이 설정(변하지 않음)이고 금액이 데이터(자주 변함)이므로 이 순서가 자연스럽다.

**(b) between 커링**

```typescript
const between =
  (min: number) =>
  (max: number) =>
  (value: number): boolean =>
    value >= min && value <= max;

// 부분 적용 예시
const isPercentage = between(0)(100);
const isTeenager = between(13)(19);
const isWorkingAge = between(18)(65);

isPercentage(50);   // true
isPercentage(150);  // false
isTeenager(15);     // true
isWorkingAge(30);   // true

// filter와 함께 사용
const ages = [5, 15, 25, 35, 70];
ages.filter(isWorkingAge); // [25, 35]
```

### 문제 3 해답

```typescript
console.log(functions[0]()); // 0
console.log(functions[2]()); // 20
console.log(functions[4]()); // 40
```

**이유**: `let`은 블록 스코프이므로 각 반복에서 새로운 `i`가 만들어진다. `captured = i * 10`은 각 반복에서 새로운 `const`로 선언되므로, 각 클로저는 자신만의 `captured` 값을 캡처한다.

- `i = 0` → `captured = 0` → `fns[0]`은 0을 반환
- `i = 1` → `captured = 10` → `fns[1]`은 10을 반환
- `i = 2` → `captured = 20` → `fns[2]`은 20을 반환
- `i = 3` → `captured = 30` → `fns[3]`은 30을 반환
- `i = 4` → `captured = 40` → `fns[4]`은 40을 반환

만약 `var`를 사용하고 `captured` 없이 직접 `i`를 참조했다면, 모든 함수가 5를 반환했을 것이다.

### 문제 4 해답

```typescript
function compose<T>(...fns: Array<(arg: T) => T>): (arg: T) => T {
  return (arg: T): T => fns.reduceRight((acc, fn) => fn(acc), arg);
}
```

`reduceRight`를 사용하여 오른쪽부터 왼쪽으로 함수를 적용한다. `compose(f, g, h)(x)`는 `f(g(h(x)))`가 된다.

동작 추적:
```
compose(addExclamation, toUpper, trim)("  hello  ")
→ reduceRight를 "  hello  "부터 시작
→ trim("  hello  ")     = "hello"
→ toUpper("hello")      = "HELLO"
→ addExclamation("HELLO") = "HELLO!"
```

`reduce`를 사용하면 왼쪽부터 적용되므로 `pipe`가 된다:
```typescript
function pipe<T>(...fns: Array<(arg: T) => T>): (arg: T) => T {
  return (arg: T): T => fns.reduce((acc, fn) => fn(acc), arg);
}
// pipe(trim, toUpper, addExclamation)("  hello  ") === "HELLO!"
```

`compose`는 수학의 함수 합성 순서(f ∘ g는 g를 먼저 적용)와 같고, `pipe`는 데이터 흐름 순서(왼쪽에서 오른쪽)와 같다. 4주차에서 이것을 더 깊이 다룬다.

---

## 핵심 정리

| 개념 | 한 줄 요약 |
|------|-----------|
| 일급 함수 | 함수를 값처럼 변수에 담고, 전달하고, 반환할 수 있다 |
| 고차 함수 | 함수를 인자로 받거나 함수를 반환하는 함수 |
| 클로저 | 함수가 자신이 정의된 스코프의 변수를 기억하는 것 |
| 렉시컬 스코프 | 변수의 유효 범위가 코드가 작성된 위치에 의해 결정됨 |
| 커링 | 다인자 함수를 단인자 함수의 연쇄로 변환 |
| 부분 적용 | 함수의 일부 인자를 미리 고정하여 새 함수 생성 |
| data-last | 설정을 앞에, 데이터를 뒤에 놓는 인자 순서 규칙 |

## 다음 주차 예고

**3주차: 불변성과 영속 자료구조** — 값을 변경하지 않고 새 값을 만드는 불변 프로그래밍의 원리를 배운다. 구조적 공유(Structural Sharing)를 통해 불변성과 성능을 동시에 달성하는 방법, 그리고 React 상태 관리에서의 불변 업데이트 패턴을 다룬다.
