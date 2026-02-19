# 2주차: 일급 함수와 고차 함수

## 왜 이것을 배우는가?

1주차에서 수학적 함수와 순수 함수를 배웠다. 함수형 프로그래밍의 첫 번째 원칙은 "같은 입력 → 같은 출력"이었다. 그런데 이것만으로는 함수형 프로그래밍의 진짜 힘이 발휘되지 않는다.

함수형 프로그래밍의 진짜 힘은 **함수를 값처럼 다루는 것**에서 나온다. 숫자를 변수에 담고, 배열에 넣고, 다른 함수에 전달하듯이 — 함수도 똑같이 할 수 있다면 어떨까? 함수를 조합하고, 변환하고, 생성하는 것이 자유로워진다.

이것이 가능하려면 함수가 **일급 시민(First-class Citizen)** 이어야 한다. 그리고 함수를 받거나 반환하는 **고차 함수(Higher-order Function)** 가 핵심 도구가 된다. 여기에 **클로저(Closure)** 가 합쳐지면 함수로 상태를 캡슐화할 수 있고, **커링(Currying)** 과 **부분 적용(Partial Application)** 을 통해 함수를 더 작은 단위로 분해하고 재조합할 수 있다.

이번 주에 배우는 개념은 앞으로 배울 모든 추상화(Functor, Monad 등)의 전제 조건이다. 함수를 값으로 다루지 못하면 `map`, `flatMap`, `pipe` 어떤 것도 불가능하다.

---

## 1. 일급 시민으로서의 함수

### 1.1 일급 시민의 조건

프로그래밍 언어에서 어떤 것이 **일급 시민(First-class Citizen)** 이라 함은, 그것이 다음을 모두 만족한다는 뜻이다:

1. **변수에 할당**할 수 있다
2. **데이터 구조에 저장**할 수 있다
3. **함수의 인자로 전달**할 수 있다
4. **함수의 반환값으로 반환**할 수 있다
5. **런타임에 생성**할 수 있다

숫자, 문자열, 객체는 대부분의 언어에서 일급 시민이다. JavaScript/TypeScript에서는 **함수도 일급 시민**이다. 이것은 당연해 보이지만, 모든 언어가 이를 지원하는 것은 아니다 (C 언어의 함수는 함수 포인터를 통해 제한적으로만 일급이고, Java는 8 이전에는 함수가 일급이 아니었다).

### 1.2 함수를 변수에 할당

```typescript
// 숫자를 변수에 할당
const x: number = 42;

// 함수도 변수에 할당할 수 있다
const add = (a: number, b: number): number => a + b;
const greet = function(name: string): string {
  return `Hello, ${name}!`;
};

// 함수 선언도 사실 이름 있는 변수 할당과 같다
function multiply(a: number, b: number): number {
  return a * b;
}
// multiply도 값이다 — 다른 변수에 할당 가능
const mul = multiply;
mul(3, 4); // 12
```

**핵심 관찰**: `add`와 `42`는 본질적으로 같은 방식으로 취급된다. 둘 다 "값"이다.

### 1.3 함수를 데이터 구조에 저장

```typescript
// 배열에 함수 저장
const operations: Array<(a: number, b: number) => number> = [
  (a, b) => a + b,
  (a, b) => a - b,
  (a, b) => a * b,
  (a, b) => a / b,
];

operations[0](10, 3); // 13
operations[2](10, 3); // 30

// 객체(맵)에 함수 저장
const validators: Record<string, (value: string) => boolean> = {
  email: (v) => v.includes("@"),
  nonEmpty: (v) => v.length > 0,
  minLength: (v) => v.length >= 8,
};

validators["email"]("test@example.com"); // true
```

이것이 왜 중요한가? **전략 패턴(Strategy Pattern)** 을 클래스 없이 함수만으로 구현할 수 있다. OOP에서는 인터페이스, 클래스, 상속 구조가 필요한 것을 함수 하나로 대체한다.

### 1.4 함수를 인자로 전달

```typescript
// 함수를 인자로 받는 함수
function applyTwice(f: (n: number) => number, x: number): number {
  return f(f(x));
}

applyTwice(n => n + 1, 5);   // 7 — (5 + 1) + 1
applyTwice(n => n * 2, 3);   // 12 — (3 * 2) * 2
applyTwice(n => n * n, 2);   // 16 — (2 * 2) * (2 * 2) = 4 * 4... 아니, (2²)² = 4² = 16
```

`applyTwice`는 **무엇을 할지**를 인자로 받는다. 로직 자체가 데이터처럼 전달된다.

### 1.5 함수를 반환값으로 반환

```typescript
// 함수를 반환하는 함수
function multiplier(factor: number): (n: number) => number {
  return (n: number) => n * factor;
}

const double = multiplier(2);
const triple = multiplier(3);

double(5);  // 10
triple(5);  // 15

// multiplier는 "함수를 만드는 함수" — 함수 팩토리다
```

함수를 반환한다는 것은 **함수를 동적으로 생성**한다는 뜻이다. 실행 시점에 조건에 따라 다른 함수를 만들어낼 수 있다.

---

## 2. 고차 함수 (Higher-order Function)

### 2.1 정의

**고차 함수**는 다음 중 하나 이상을 만족하는 함수다:

1. 함수를 **인자로 받는다**
2. 함수를 **반환한다**

수학에서의 정의: 함수 `f`가 고차 함수라 함은, `f`의 정의역이나 공역에 함수 공간이 포함된다는 것이다.

```
고차 함수의 타입 시그니처 패턴:

함수를 받는 경우:  (f: A → B, ...) → C
함수를 반환하는 경우:  (...) → (A → B)
둘 다:              (f: A → B) → (C → D)
```

고차 함수가 아닌 것: `add(a, b) = a + b`처럼 일반 값만 받고 일반 값만 반환하는 함수는 **일차 함수(First-order Function)** 다.

### 2.2 함수를 인자로 받는 고차 함수

가장 익숙한 형태는 배열 메서드들이다.

```typescript
const numbers = [1, 2, 3, 4, 5];

// map: 각 요소에 함수를 적용하여 변환
const doubled = numbers.map(n => n * 2);
// [2, 4, 6, 8, 10]

// filter: 조건 함수를 만족하는 요소만 선별
const evens = numbers.filter(n => n % 2 === 0);
// [2, 4]

// reduce: 누적 함수로 하나의 값으로 접기
const sum = numbers.reduce((acc, n) => acc + n, 0);
// 15
```

이 세 함수(`map`, `filter`, `reduce`)가 왜 중요한가? 거의 **모든 데이터 변환**을 이 세 가지로 표현할 수 있기 때문이다. 명령형의 `for` 루프가 하는 일을 선언적으로 대체한다.

```typescript
// 명령형: "어떻게" 하는지를 기술
const results: string[] = [];
for (let i = 0; i < users.length; i++) {
  if (users[i].age >= 18) {
    results.push(users[i].name.toUpperCase());
  }
}

// 선언형: "무엇을" 하는지를 기술
const results = users
  .filter(u => u.age >= 18)
  .map(u => u.name.toUpperCase());
```

명령형 코드는 루프 변수 `i`, 중간 배열 `results`, 조건문, 배열 조작이 뒤섞여 있다. 선언형 코드는 "성인을 필터링하고, 이름을 대문자로 변환한다"는 의도가 직접 드러난다.

### 2.3 함수를 반환하는 고차 함수

```typescript
// 비교 함수 생성기
function compareBy<T>(key: keyof T): (a: T, b: T) => number {
  return (a: T, b: T) => {
    const va = a[key];
    const vb = b[key];
    return va < vb ? -1 : va > vb ? 1 : 0;
  };
}

interface User {
  readonly name: string;
  readonly age: number;
}

const users: User[] = [
  { name: "Charlie", age: 25 },
  { name: "Alice", age: 30 },
  { name: "Bob", age: 20 },
];

const byName = compareBy<User>("name");
const byAge = compareBy<User>("age");

[...users].sort(byName);
// [{ name: "Alice", ... }, { name: "Bob", ... }, { name: "Charlie", ... }]

[...users].sort(byAge);
// [{ name: "Bob", age: 20 }, { name: "Charlie", age: 25 }, { name: "Alice", age: 30 }]
```

`compareBy`는 "어떤 키로 비교할지"를 받아서 "비교 함수를 반환"한다. 정렬 로직을 추상화한 것이다.

### 2.4 직접 구현해보기: map, filter, reduce

라이브러리가 제공하는 것을 쓰기 전에 직접 만들어보면 이해가 깊어진다.

```typescript
// map 직접 구현
function map<A, B>(arr: ReadonlyArray<A>, f: (a: A) => B): ReadonlyArray<B> {
  const result: B[] = [];
  for (const item of arr) {
    result.push(f(item));
  }
  return result;
}

// filter 직접 구현
function filter<A>(arr: ReadonlyArray<A>, predicate: (a: A) => boolean): ReadonlyArray<A> {
  const result: A[] = [];
  for (const item of arr) {
    if (predicate(item)) {
      result.push(item);
    }
  }
  return result;
}

// reduce 직접 구현
function reduce<A, B>(arr: ReadonlyArray<A>, f: (acc: B, a: A) => B, initial: B): B {
  let acc = initial;
  for (const item of arr) {
    acc = f(acc, item);
  }
  return acc;
}

// 사용
map([1, 2, 3], n => n * 2);                    // [2, 4, 6]
filter([1, 2, 3, 4], n => n % 2 === 0);        // [2, 4]
reduce([1, 2, 3, 4], (sum, n) => sum + n, 0);  // 10
```

**관찰**: `map`과 `filter`를 `reduce`로 구현할 수 있다. `reduce`는 가장 일반적인(general) 고차 함수다.

```typescript
// reduce로 map 구현
function mapViaReduce<A, B>(arr: ReadonlyArray<A>, f: (a: A) => B): ReadonlyArray<B> {
  return reduce(arr, (acc: B[], item) => [...acc, f(item)], [] as B[]);
}

// reduce로 filter 구현
function filterViaReduce<A>(arr: ReadonlyArray<A>, pred: (a: A) => boolean): ReadonlyArray<A> {
  return reduce(arr, (acc: A[], item) => pred(item) ? [...acc, item] : acc, [] as A[]);
}
```

이것은 단순한 트릭이 아니다. 나중에 배울 **Foldable** (14주차)의 핵심 원리다 — `reduce`(fold)가 있으면 거의 모든 리스트 연산을 유도할 수 있다.

---

## 3. 클로저 (Closure)

### 3.1 렉시컬 스코프 (Lexical Scope)

클로저를 이해하려면 먼저 **렉시컬 스코프**를 이해해야 한다.

렉시컬 스코프란: 함수가 **정의된 위치**에서 변수 접근 범위가 결정되는 것이다. 함수가 **호출되는 위치**가 아니라 **작성된 위치**가 중요하다.

```typescript
const outerVar = "outer";

function outer() {
  const innerVar = "inner";

  function inner() {
    // inner는 자신의 스코프 + outer의 스코프 + 전역 스코프에 접근 가능
    console.log(innerVar); // ✅ "inner" — 바로 바깥 스코프
    console.log(outerVar); // ✅ "outer" — 전역 스코프
  }

  return inner;
}

const fn = outer();
fn(); // inner가 outer() 밖에서 호출되지만, innerVar에 접근할 수 있다
```

`inner` 함수가 `outer` 함수 밖에서 실행되었는데도 `innerVar`에 접근할 수 있다. 왜? **렉시컬 스코프** 때문이다. `inner`가 정의된 위치에서 `innerVar`가 보이므로, 어디서 호출되든 그 변수에 접근할 수 있다.

### 3.2 클로저의 정의

**클로저(Closure)** = 함수 + 그 함수가 정의된 렉시컬 환경(Lexical Environment)

함수가 자신이 정의된 스코프의 변수를 **"캡처(capture)"** 하여 들고 다니는 것이다. 함수가 반환된 후에도 캡처된 변수는 사라지지 않는다.

```typescript
function makeCounter(): { increment: () => number; getCount: () => number } {
  let count = 0; // 이 변수가 클로저에 의해 캡처된다

  return {
    increment: () => ++count,
    getCount: () => count,
  };
}

const counter = makeCounter();
counter.increment(); // 1
counter.increment(); // 2
counter.getCount();  // 2

// count는 makeCounter의 지역 변수였지만,
// 반환된 함수들이 클로저로 캡처하고 있으므로 살아있다.

// 새 카운터는 독립적인 count를 가진다
const counter2 = makeCounter();
counter2.getCount(); // 0 — counter의 count와 별개
```

### 3.3 클로저의 활용 패턴

**패턴 1: 데이터 은닉 (캡슐화)**

```typescript
function createStack<T>() {
  let items: T[] = []; // 외부에서 직접 접근 불가

  return {
    push: (item: T): void => { items = [...items, item]; },
    pop: (): T | undefined => {
      const last = items[items.length - 1];
      items = items.slice(0, -1);
      return last;
    },
    peek: (): T | undefined => items[items.length - 1],
    size: (): number => items.length,
  };
}

const stack = createStack<number>();
stack.push(1);
stack.push(2);
stack.pop();   // 2
stack.size();  // 1
// stack.items — ❌ 접근 불가, items는 클로저 안에 은닉됨
```

**패턴 2: 함수 팩토리**

```typescript
function createLogger(prefix: string) {
  return {
    info: (msg: string) => console.log(`[${prefix}] INFO: ${msg}`),
    warn: (msg: string) => console.warn(`[${prefix}] WARN: ${msg}`),
    error: (msg: string) => console.error(`[${prefix}] ERROR: ${msg}`),
  };
}

const authLogger = createLogger("Auth");
const apiLogger = createLogger("API");

authLogger.info("User logged in");   // [Auth] INFO: User logged in
apiLogger.error("Request failed");   // [API] ERROR: Request failed
```

**패턴 3: 메모이제이션**

1주차에서 순수 함수의 캐싱을 배웠다. 클로저로 캐시를 구현한다.

```typescript
function memoize<A extends string | number, B>(fn: (a: A) => B): (a: A) => B {
  const cache = new Map<A, B>(); // 클로저로 캡처된 캐시

  return (a: A): B => {
    if (cache.has(a)) {
      return cache.get(a)!;
    }
    const result = fn(a);
    cache.set(a, result);
    return result;
  };
}

const expensiveFib = memoize((n: number): number => {
  if (n <= 1) return n;
  return expensiveFib(n - 1) + expensiveFib(n - 2);
});

expensiveFib(40); // 캐시 덕분에 빠르게 계산
```

### 3.4 클로저와 순수 함수의 관계

중요한 질문: 클로저를 사용하면 순수 함수가 아닌 것 아닌가?

```typescript
// 이것은 순수한가?
function multiplier(factor: number): (n: number) => number {
  return (n: number) => n * factor; // factor를 클로저로 캡처
}

const double = multiplier(2);
double(5); // 항상 10
double(5); // 항상 10 — 같은 입력, 같은 출력 ✅
```

`double`은 순수 함수다. `factor`가 `2`로 고정된 후에는 절대 변하지 않는다. 클로저가 캡처한 값이 **불변**이면 순수성이 유지된다.

```typescript
// 이것은 순수하지 않다
function makeAccumulator(initial: number): (n: number) => number {
  let total = initial; // 가변 상태를 클로저로 캡처
  return (n: number) => {
    total += n; // 캡처된 변수를 변경!
    return total;
  };
}

const acc = makeAccumulator(0);
acc(5); // 5
acc(5); // 10 — 같은 입력, 다른 출력 ❌
```

**규칙**: 클로저가 캡처한 변수를 **읽기만** 하면 순수, **변경하면** 비순수.

---

## 4. 커링 (Currying)

### 4.1 수학적 배경

수학에서 커링은 다인자 함수를 일련의 단일 인자 함수로 변환하는 것이다. 이름은 논리학자 **하스켈 커리(Haskell Curry)** 에서 유래했다 (Haskell 언어의 이름도 여기서 왔다).

```
수학적 표현:

f: (A × B) → C
를 다음으로 변환
curry(f): A → (B → C)

즉, 두 인자를 한 번에 받는 함수를
하나씩 받는 함수의 체인으로 바꾼다.
```

이것이 왜 중요한가? **함수 합성**이 쉬워지기 때문이다. 합성은 `A → B`와 `B → C`를 연결하여 `A → C`를 만드는 것인데, 인자가 하나인 함수가 합성하기 가장 쉽다.

### 4.2 TypeScript에서의 커링

```typescript
// 커링되지 않은 함수
function addUncurried(a: number, b: number): number {
  return a + b;
}
addUncurried(1, 2); // 3

// 커링된 함수
function addCurried(a: number): (b: number) => number {
  return (b: number) => a + b;
}
addCurried(1)(2); // 3

// 화살표 함수로 더 간결하게
const add = (a: number) => (b: number): number => a + b;
add(1)(2); // 3
```

커링된 함수의 호출 방식에 주목하자: `add(1)(2)`. 먼저 `add(1)`이 호출되어 `(b: number) => 1 + b`라는 새 함수를 반환한다. 그 다음 이 함수에 `2`를 전달한다.

### 4.3 범용 curry 함수 구현

```typescript
// 2인자 함수를 커링하는 함수
function curry2<A, B, C>(f: (a: A, b: B) => C): (a: A) => (b: B) => C {
  return (a: A) => (b: B) => f(a, b);
}

// 3인자 함수를 커링하는 함수
function curry3<A, B, C, D>(
  f: (a: A, b: B, c: C) => D
): (a: A) => (b: B) => (c: C) => D {
  return (a: A) => (b: B) => (c: C) => f(a, b, c);
}

// 사용
const curriedAdd = curry2((a: number, b: number) => a + b);
curriedAdd(1)(2); // 3

const curriedReplace = curry3(
  (str: string, search: string, replace: string) => str.replace(search, replace)
);
const replaceSpaces = curriedReplace("hello world")(" ");
replaceSpaces("-"); // "hello-world"
```

### 4.4 커링의 실전 활용

**활용 1: 설정의 사전 주입**

```typescript
// API 호출 함수를 커링으로 설계
const fetchFrom = (baseUrl: string) => (endpoint: string) => (params?: Record<string, string>) =>
  fetch(`${baseUrl}${endpoint}?${new URLSearchParams(params).toString()}`);

// 기본 URL을 미리 주입
const api = fetchFrom("https://api.example.com");

// 엔드포인트를 지정한 특화 함수 생성
const getUsers = api("/users");
const getPosts = api("/posts");

// 최종 호출
getUsers({ page: "1", limit: "10" });
getPosts({ category: "tech" });
```

**활용 2: 데이터 변환 파이프라인 준비**

```typescript
// 커링된 유틸리티 함수들
const prop = <T, K extends keyof T>(key: K) => (obj: T): T[K] => obj[key];
const gt = (threshold: number) => (n: number): boolean => n > threshold;
const includes = (search: string) => (str: string): boolean => str.includes(search);

interface Product {
  readonly name: string;
  readonly price: number;
  readonly category: string;
}

const products: Product[] = [
  { name: "Laptop", price: 1200, category: "electronics" },
  { name: "Book", price: 15, category: "education" },
  { name: "Phone", price: 800, category: "electronics" },
];

// 커링된 함수를 조합하여 사용
const expensiveProducts = products
  .filter(p => gt(100)(prop<Product, "price">("price")(p)))
  .map(prop("name"));
// ["Laptop", "Phone"]
```

이 예시는 아직 약간 불편하다. 4주차에서 배울 **pipe/flow**를 사용하면 훨씬 깔끔해진다. 지금은 커링이 "함수를 부분적으로 적용할 수 있게 준비하는 도구"라는 점을 이해하면 된다.

---

## 5. 부분 적용 (Partial Application)

### 5.1 정의

**부분 적용**은 다인자 함수에 일부 인자만 먼저 제공하여, 나머지 인자를 받는 새 함수를 만드는 것이다.

```
원래 함수:     f(a, b, c)
부분 적용:     g = f(1, __, __) → g(b, c) = f(1, b, c)
```

### 5.2 커링과 부분 적용의 차이

이 둘은 자주 혼동되지만 다른 개념이다.

```typescript
// 커링: 다인자 함수를 단일 인자 함수의 체인으로 변환
// f(a, b, c) → f(a)(b)(c)
// 항상 인자를 하나씩 받는다

const curriedAdd = (a: number) => (b: number) => (c: number): number =>
  a + b + c;

curriedAdd(1)(2)(3); // 6


// 부분 적용: 일부 인자를 고정한 새 함수를 생성
// f(a, b, c) → g(c) = f(1, 2, c)
// 한 번에 여러 인자를 고정할 수 있다

function addThree(a: number, b: number, c: number): number {
  return a + b + c;
}

function partial<A, B, C, R>(
  f: (a: A, b: B, c: C) => R,
  a: A,
  b: B
): (c: C) => R {
  return (c: C) => f(a, b, c);
}

const add1and2 = partial(addThree, 1, 2);
add1and2(3); // 6
```

**핵심 차이**:
- **커링**: 함수의 **형태를 변환**한다 `(A, B, C) → D` 를 `A → B → C → D`로
- **부분 적용**: 함수의 일부 인자를 **미리 채운다** `(A, B, C) → D`에서 A를 고정하면 `(B, C) → D`

커링된 함수에서 부분 적용은 자연스럽게 일어난다:

```typescript
const add = (a: number) => (b: number): number => a + b;

// add(1)은 부분 적용이자 커링된 함수의 자연스러운 호출
const increment = add(1); // (b: number) => 1 + b
increment(5); // 6
```

### 5.3 bind를 이용한 부분 적용

JavaScript의 `Function.prototype.bind`도 부분 적용의 한 형태다.

```typescript
function greet(greeting: string, name: string): string {
  return `${greeting}, ${name}!`;
}

const sayHello = greet.bind(null, "Hello");
const sayGoodbye = greet.bind(null, "Goodbye");

sayHello("Alice");    // "Hello, Alice!"
sayGoodbye("Bob");    // "Goodbye, Bob!"
```

하지만 `bind`는 타입 추론이 약하고, `this` 바인딩과 얽혀 있어 함수형 프로그래밍에서는 커링이나 명시적 부분 적용 함수를 선호한다.

---

## 6. 고차 함수의 고급 패턴

### 6.1 함수 합성 맛보기

고차 함수의 궁극적 활용은 **함수 합성**이다 (4주차에서 자세히 다룬다). 간단히 맛보기:

```typescript
// compose: 두 함수를 합성 (오른쪽부터 실행)
function compose<A, B, C>(
  f: (b: B) => C,
  g: (a: A) => B
): (a: A) => C {
  return (a: A) => f(g(a));
}

const toUpper = (s: string): string => s.toUpperCase();
const exclaim = (s: string): string => s + "!";

const shout = compose(exclaim, toUpper);
shout("hello"); // "HELLO!"

// pipe: 왼쪽부터 실행 (읽는 순서대로)
function pipe<A, B, C>(
  f: (a: A) => B,
  g: (b: B) => C
): (a: A) => C {
  return (a: A) => g(f(a));
}

const shout2 = pipe(toUpper, exclaim);
shout2("hello"); // "HELLO!"
```

### 6.2 함수 데코레이터 패턴

고차 함수로 기존 함수의 행동을 감싸서 확장할 수 있다.

```typescript
// 실행 시간을 측정하는 데코레이터
function withTiming<A extends unknown[], R>(
  fn: (...args: A) => R,
  label: string
): (...args: A) => R {
  return (...args: A): R => {
    const start = performance.now();
    const result = fn(...args);
    const elapsed = performance.now() - start;
    console.log(`[${label}] ${elapsed.toFixed(2)}ms`);
    return result;
  };
}

// 재시도 로직을 추가하는 데코레이터
function withRetry<A extends unknown[], R>(
  fn: (...args: A) => Promise<R>,
  maxRetries: number
): (...args: A) => Promise<R> {
  return async (...args: A): Promise<R> => {
    for (let attempt = 0; attempt <= maxRetries; attempt++) {
      try {
        return await fn(...args);
      } catch (error) {
        if (attempt === maxRetries) throw error;
      }
    }
    throw new Error("Unreachable");
  };
}

// 사용
const fetchData = async (url: string) => {
  const res = await fetch(url);
  return res.json();
};

const resilientFetch = withRetry(fetchData, 3);
const timedFetch = withTiming(fetchData, "API Call");
```

핵심: 원래 함수를 수정하지 않고 새로운 능력을 추가한다. 이것이 **합성**의 힘이다.

### 6.3 함수를 다루는 유틸리티 고차 함수들

```typescript
// once: 한 번만 실행되는 함수
function once<A extends unknown[], R>(fn: (...args: A) => R): (...args: A) => R {
  let called = false;
  let result: R;
  return (...args: A): R => {
    if (!called) {
      called = true;
      result = fn(...args);
    }
    return result;
  };
}

const initialize = once(() => {
  console.log("Initializing...");
  return { ready: true };
});

initialize(); // "Initializing..." → { ready: true }
initialize(); // (아무 출력 없음) → { ready: true } — 캐시된 결과

// negate: 술어 함수를 반전
function negate<A extends unknown[]>(
  pred: (...args: A) => boolean
): (...args: A) => boolean {
  return (...args: A) => !pred(...args);
}

const isEven = (n: number): boolean => n % 2 === 0;
const isOdd = negate(isEven);

[1, 2, 3, 4, 5].filter(isOdd); // [1, 3, 5]

// flip: 인자 순서를 뒤집기
function flip<A, B, C>(f: (a: A, b: B) => C): (b: B, a: A) => C {
  return (b: B, a: A) => f(a, b);
}

const divide = (a: number, b: number): number => a / b;
const divideBy = flip(divide);

divideBy(2, 10); // 10 / 2 = 5
```

---

## 7. 프론트엔드 실전

### 7.1 이벤트 핸들러 팩토리

React에서 이벤트 핸들러를 동적으로 생성할 때 고차 함수와 클로저가 핵심이다.

```typescript
// ❌ 안티패턴: 렌더링마다 인라인 함수 생성
function TodoList({ todos, onToggle }: Props) {
  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          {/* 렌더링마다 새 함수가 생성된다 — 하위 컴포넌트 리렌더링 유발 */}
          <button onClick={() => onToggle(todo.id)}>
            {todo.text}
          </button>
        </li>
      ))}
    </ul>
  );
}

// ✅ 고차 함수로 핸들러 팩토리 생성
function TodoList({ todos, onToggle }: Props) {
  // useCallback + 커링 패턴
  const handleToggle = useCallback(
    (id: string) => () => onToggle(id),
    [onToggle]
  );

  return (
    <ul>
      {todos.map(todo => (
        <li key={todo.id}>
          <button onClick={handleToggle(todo.id)}>
            {todo.text}
          </button>
        </li>
      ))}
    </ul>
  );
}
```

더 일반적인 이벤트 핸들러 팩토리:

```typescript
// 폼 필드 변경 핸들러 팩토리
function useFormHandlers<T extends Record<string, string>>(
  initial: T
): {
  values: T;
  handleChange: (field: keyof T) => (e: React.ChangeEvent<HTMLInputElement>) => void;
  reset: () => void;
} {
  const [values, setValues] = useState(initial);

  const handleChange = useCallback(
    (field: keyof T) => (e: React.ChangeEvent<HTMLInputElement>) => {
      setValues(prev => ({ ...prev, [field]: e.target.value }));
    },
    []
  );

  const reset = useCallback(() => setValues(initial), [initial]);

  return { values, handleChange, reset };
}

// 사용
function SignupForm() {
  const { values, handleChange, reset } = useFormHandlers({
    name: "",
    email: "",
    password: "",
  });

  return (
    <form>
      <input value={values.name} onChange={handleChange("name")} />
      <input value={values.email} onChange={handleChange("email")} />
      <input value={values.password} onChange={handleChange("password")} />
      <button type="button" onClick={reset}>초기화</button>
    </form>
  );
}
```

`handleChange`는 커링된 함수다: 필드 이름을 먼저 받아 이벤트 핸들러를 반환한다.

### 7.2 설정 가능한 API 클라이언트

고차 함수로 API 클라이언트를 계층적으로 설정할 수 있다.

```typescript
// 기본 fetch를 감싸는 고차 함수들
type FetchFn = (url: string, init?: RequestInit) => Promise<Response>;

// 기본 헤더를 주입하는 고차 함수
const withHeaders = (headers: Record<string, string>) =>
  (fetchFn: FetchFn): FetchFn =>
    (url, init = {}) =>
      fetchFn(url, {
        ...init,
        headers: { ...headers, ...init.headers },
      });

// 인증 토큰을 주입하는 고차 함수
const withAuth = (getToken: () => string) =>
  (fetchFn: FetchFn): FetchFn =>
    (url, init = {}) =>
      fetchFn(url, {
        ...init,
        headers: {
          ...init.headers,
          Authorization: `Bearer ${getToken()}`,
        },
      });

// 에러를 자동 처리하는 고차 함수
const withErrorHandling = (fetchFn: FetchFn): FetchFn =>
  async (url, init) => {
    const response = await fetchFn(url, init);
    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }
    return response;
  };

// 조합하여 완전한 API 클라이언트 구축
const createApiClient = (baseUrl: string, getToken: () => string): FetchFn => {
  const addHeaders = withHeaders({ "Content-Type": "application/json" });
  const addAuth = withAuth(getToken);

  return withErrorHandling(addAuth(addHeaders(fetch)));
};

// 사용
const api = createApiClient(
  "https://api.example.com",
  () => localStorage.getItem("token") ?? ""
);

const users = await api("/users").then(r => r.json());
```

각 고차 함수(`withHeaders`, `withAuth`, `withErrorHandling`)는 `FetchFn`을 받아 강화된 `FetchFn`을 반환한다. 이것은 4주차에서 배울 **미들웨어 패턴**과 직결된다.

### 7.3 Higher-order Component (HOC) 패턴

HOC는 React에서 고차 함수의 직접적인 적용이다. 컴포넌트(함수)를 받아서 강화된 컴포넌트(함수)를 반환한다.

```typescript
// 로딩 상태를 추가하는 HOC
function withLoading<P extends object>(
  WrappedComponent: React.ComponentType<P>
): React.ComponentType<P & { isLoading: boolean }> {
  return function WithLoadingComponent({ isLoading, ...props }: P & { isLoading: boolean }) {
    if (isLoading) {
      return <div className="spinner">Loading...</div>;
    }
    return <WrappedComponent {...(props as P)} />;
  };
}

// 에러 바운더리를 추가하는 HOC
function withErrorBoundary<P extends object>(
  WrappedComponent: React.ComponentType<P>,
  FallbackComponent: React.ComponentType<{ error: Error }>
): React.ComponentType<P> {
  return function WithErrorBoundaryComponent(props: P) {
    const [error, setError] = useState<Error | null>(null);

    if (error) {
      return <FallbackComponent error={error} />;
    }

    return (
      <ErrorBoundaryWrapper onError={setError}>
        <WrappedComponent {...props} />
      </ErrorBoundaryWrapper>
    );
  };
}

// 조합
const UserList = ({ users }: { users: User[] }) => (
  <ul>{users.map(u => <li key={u.id}>{u.name}</li>)}</ul>
);

const EnhancedUserList = withLoading(UserList);
// <EnhancedUserList users={users} isLoading={loading} />
```

**참고**: 현대 React에서는 HOC 대신 **커스텀 훅(Custom Hook)** 을 선호하지만, HOC는 고차 함수 개념을 컴포넌트 레벨에서 보여주는 좋은 예시다. 둘 다 "함수를 받아 강화된 함수를 반환하는" 고차 함수 패턴이다.

### 7.4 커스텀 훅에서의 고차 함수

```typescript
// 고차 함수 패턴을 커스텀 훅으로
function useFilter<T>(
  items: ReadonlyArray<T>,
  predicates: Record<string, (item: T) => boolean>
) {
  const [activeFilters, setActiveFilters] = useState<Set<string>>(new Set());

  const filteredItems = useMemo(() => {
    if (activeFilters.size === 0) return items;

    // reduce로 활성 필터를 합성하여 적용
    return items.filter(item =>
      Array.from(activeFilters).every(key => predicates[key](item))
    );
  }, [items, activeFilters, predicates]);

  const toggleFilter = useCallback((key: string) => {
    setActiveFilters(prev => {
      const next = new Set(prev);
      if (next.has(key)) {
        next.delete(key);
      } else {
        next.add(key);
      }
      return next;
    });
  }, []);

  return { filteredItems, activeFilters, toggleFilter };
}

// 사용
function ProductPage({ products }: { products: Product[] }) {
  const { filteredItems, toggleFilter, activeFilters } = useFilter(products, {
    cheap: (p) => p.price < 50,
    electronics: (p) => p.category === "electronics",
    inStock: (p) => p.stock > 0,
  });

  return (
    <div>
      <div className="filters">
        {["cheap", "electronics", "inStock"].map(key => (
          <button
            key={key}
            onClick={() => toggleFilter(key)}
            className={activeFilters.has(key) ? "active" : ""}
          >
            {key}
          </button>
        ))}
      </div>
      <ul>
        {filteredItems.map(p => <li key={p.name}>{p.name}</li>)}
      </ul>
    </div>
  );
}
```

`predicates`는 필터 함수들의 맵이다. `useFilter`는 이 함수들을 조합하여 아이템을 필터링한다. 필터 로직이 **데이터(함수 객체)** 로 전달되는 점이 핵심이다.

---

## 연습 문제

### 문제 1: 고차 함수 식별

다음 함수들 중 고차 함수인 것을 모두 고르고 이유를 설명하라.

```typescript
// (a)
function identity<T>(x: T): T {
  return x;
}

// (b)
function apply<A, B>(f: (a: A) => B, x: A): B {
  return f(x);
}

// (c)
function constant<A>(a: A): () => A {
  return () => a;
}

// (d)
function head<T>(arr: T[]): T | undefined {
  return arr[0];
}

// (e)
function on<A, B, C>(
  f: (b1: B, b2: B) => C,
  g: (a: A) => B
): (a1: A, a2: A) => C {
  return (a1: A, a2: A) => f(g(a1), g(a2));
}
```

### 문제 2: 클로저 추론

다음 코드의 실행 결과를 예측하라. 각 단계에서 클로저가 어떤 값을 캡처하고 있는지 설명하라.

```typescript
function createMultipliers(): Array<(n: number) => number> {
  const multipliers: Array<(n: number) => number> = [];
  for (let i = 1; i <= 3; i++) {
    multipliers.push((n: number) => n * i);
  }
  return multipliers;
}

const [times1, times2, times3] = createMultipliers();

console.log(times1(10)); // ?
console.log(times2(10)); // ?
console.log(times3(10)); // ?

// 만약 let 대신 var를 사용했다면?
function createMultipliersVar(): Array<(n: number) => number> {
  const multipliers: Array<(n: number) => number> = [];
  for (var i = 1; i <= 3; i++) {
    multipliers.push((n: number) => n * i);
  }
  return multipliers;
}

const [v1, v2, v3] = createMultipliersVar();

console.log(v1(10)); // ?
console.log(v2(10)); // ?
console.log(v3(10)); // ?
```

### 문제 3: curry 함수 구현

임의 개수의 인자를 가진 함수를 커링하는 `curry` 함수를 구현하라 (TypeScript 타입은 단순화해도 좋다).

```typescript
function curry(fn: Function): Function {
  // 구현하라
}

// 동작 예시:
const add = (a: number, b: number, c: number) => a + b + c;
const curriedAdd = curry(add);

curriedAdd(1)(2)(3);    // 6
curriedAdd(1, 2)(3);    // 6
curriedAdd(1)(2, 3);    // 6
curriedAdd(1, 2, 3);    // 6
```

### 문제 4: 고차 함수로 유효성 검증기 합성

다음 타입에 맞는 `composeValidators` 함수를 구현하라.

```typescript
type Validator<T> = (value: T) => string | null; // null이면 유효, string이면 에러 메시지

// 여러 검증기를 하나로 합성 — 첫 번째 에러를 반환
function composeValidators<T>(...validators: Validator<T>[]): Validator<T> {
  // 구현하라
}

// 동작 예시:
const required: Validator<string> = (v) =>
  v.length === 0 ? "필수 입력입니다" : null;

const minLength = (min: number): Validator<string> => (v) =>
  v.length < min ? `최소 ${min}자 이상이어야 합니다` : null;

const hasNumber: Validator<string> = (v) =>
  /\d/.test(v) ? null : "숫자를 포함해야 합니다";

const passwordValidator = composeValidators(
  required,
  minLength(8),
  hasNumber
);

passwordValidator("");          // "필수 입력입니다"
passwordValidator("abc");       // "최소 8자 이상이어야 합니다"
passwordValidator("abcdefgh");  // "숫자를 포함해야 합니다"
passwordValidator("abcdefg1");  // null (유효)
```

---

## 연습 문제 해답

### 문제 1 해답

**(a) `identity` — ❌ 고차 함수 아님**
일반 값을 받아 일반 값을 반환한다. 함수를 받거나 반환하지 않는다. (단, 제네릭이므로 `T`가 함수 타입일 수는 있지만, 그것은 호출자의 선택이지 `identity`의 정의와 무관하다.)

**(b) `apply` — ✅ 고차 함수**
첫 번째 인자 `f: (a: A) => B`가 함수다. 함수를 인자로 받으므로 고차 함수다.

**(c) `constant` — ✅ 고차 함수**
`() => A` 타입의 함수를 반환한다. 함수를 반환하므로 고차 함수다.

**(d) `head` — ❌ 고차 함수 아님**
배열을 받아 그 첫 번째 원소를 반환한다. 함수를 주고받지 않는다.

**(e) `on` — ✅ 고차 함수**
함수 `f`와 `g`를 인자로 받고, 새 함수 `(a1: A, a2: A) => C`를 반환한다. 함수를 받고 반환하므로 고차 함수다. (이 함수는 Haskell의 `Data.Function.on`에 해당하며, `compareBy`의 일반화다.)

### 문제 2 해답

**`let`을 사용한 경우:**

```
times1(10) → 10   (i = 1을 캡처)
times2(10) → 20   (i = 2를 캡처)
times3(10) → 30   (i = 3을 캡처)
```

`let`은 **블록 스코프**이므로, `for` 루프의 각 반복마다 새로운 `i` 바인딩이 생성된다. 각 클로저는 자신의 반복에서의 `i`를 캡처한다.

**`var`를 사용한 경우:**

```
v1(10) → 40   (i = 4를 참조)
v2(10) → 40   (i = 4를 참조)
v3(10) → 40   (i = 4를 참조)
```

`var`는 **함수 스코프**이므로, `i`는 하나의 변수다. 루프가 끝나면 `i = 4`이다 (`i <= 3`이 거짓이 되어 루프가 종료하는 시점에 `i`는 4). 세 클로저 모두 **같은** `i`를 참조하므로 모두 `n * 4`를 계산한다.

이것은 클로저의 고전적인 함정이다. 클로저는 변수의 **값을 복사**하는 것이 아니라 변수 자체에 대한 **참조를 캡처**한다.

### 문제 3 해답

```typescript
function curry(fn: Function): Function {
  return function curried(...args: unknown[]): unknown {
    // 충분한 인자가 모이면 원래 함수 실행
    if (args.length >= fn.length) {
      return fn(...args);
    }
    // 아직 부족하면, 추가 인자를 기다리는 새 함수 반환
    return (...moreArgs: unknown[]) => curried(...args, ...moreArgs);
  };
}

// 검증
const add = (a: number, b: number, c: number) => a + b + c;
const curriedAdd = curry(add);

console.log(curriedAdd(1)(2)(3));    // 6
console.log(curriedAdd(1, 2)(3));    // 6
console.log(curriedAdd(1)(2, 3));    // 6
console.log(curriedAdd(1, 2, 3));    // 6
```

**핵심 원리**:
- `fn.length`는 함수의 매개변수 개수를 반환한다
- 누적된 인자(`args`)가 필요한 개수(`fn.length`) 이상이면 실행
- 부족하면 클로저로 기존 인자를 캡처하고, 추가 인자를 기다리는 새 함수를 반환
- 이 과정이 재귀적으로 반복된다

### 문제 4 해답

```typescript
type Validator<T> = (value: T) => string | null;

function composeValidators<T>(...validators: Validator<T>[]): Validator<T> {
  return (value: T): string | null => {
    for (const validator of validators) {
      const error = validator(value);
      if (error !== null) {
        return error; // 첫 번째 에러를 만나면 즉시 반환
      }
    }
    return null; // 모든 검증 통과
  };
}
```

또는 `reduce`를 사용한 더 함수형 스타일:

```typescript
function composeValidators<T>(...validators: Validator<T>[]): Validator<T> {
  return (value: T): string | null =>
    validators.reduce<string | null>(
      (error, validator) => error ?? validator(value),
      null
    );
}
```

`reduce` 버전의 동작 원리:
- 초기값은 `null` (에러 없음)
- 각 단계에서 이전 에러(`error`)가 있으면 그대로 유지 (`??` 연산자)
- 이전 에러가 없으면(`null`) 다음 검증기를 실행

**참고**: 이 패턴은 7주차에서 배울 **Either Monad** 의 단락 평가(Short-circuit)와 같은 구조다. 첫 번째 실패에서 멈추는 것이 `Either`의 `flatMap` 체이닝과 동일하다. 반면 **모든** 에러를 모으고 싶다면 17주차에서 배울 **Validation** 패턴이 필요하다.

---

## 핵심 정리

| 개념 | 한 줄 요약 |
|------|-----------|
| 일급 시민 | 변수 할당, 인자 전달, 반환이 모두 가능한 값 |
| 일급 함수 | 함수가 일급 시민인 것 — 값처럼 다룰 수 있음 |
| 고차 함수 | 함수를 인자로 받거나 함수를 반환하는 함수 |
| 클로저 | 함수 + 그 함수가 정의된 렉시컬 환경 (캡처된 변수) |
| 렉시컬 스코프 | 함수가 **정의된 위치**에서 변수 접근 범위가 결정됨 |
| 커링 | `f(a, b, c)` → `f(a)(b)(c)` — 다인자 함수를 단일 인자 함수 체인으로 |
| 부분 적용 | 일부 인자를 미리 고정하여 새 함수를 만듦 |
| map/filter/reduce | 가장 기본적인 고차 함수 — 대부분의 데이터 변환을 표현 가능 |

## 다음 주차 예고

**3주차: 불변성과 영속 자료구조** — 데이터를 절대 변경하지 않고 새로운 버전을 만드는 패러다임을 배운다. 구조적 공유(Structural Sharing)로 성능을 유지하면서 불변성을 지키는 방법, 그리고 React 상태 관리에서의 실전 패턴을 다룬다.
