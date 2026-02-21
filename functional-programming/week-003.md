# 3주차: 불변성과 영속 자료구조 — 변하지 않는 것의 힘

## 왜 이것을 배우는가?

1주차에서 순수 함수를, 2주차에서 일급 함수와 고차 함수를 배웠다. 순수 함수의 핵심 조건 중 하나는 **부수 효과가 없다**는 것이었다. 그런데 부수 효과의 가장 흔한 형태가 바로 **데이터의 변경(mutation)** 이다.

```typescript
// 이 함수는 순수한가?
function addItem(cart: string[], item: string): string[] {
  cart.push(item); // 원본 배열을 변경!
  return cart;
}

const myCart = ["사과", "바나나"];
const newCart = addItem(myCart, "우유");

console.log(myCart);   // ["사과", "바나나", "우유"] — 원본이 바뀌었다!
console.log(newCart);  // ["사과", "바나나", "우유"]
console.log(myCart === newCart); // true — 같은 참조
```

`addItem`은 입력 데이터를 변경했다. 이제 `myCart`를 사용하는 다른 코드는 예상치 못한 결과를 얻게 된다. 이것이 **공유된 가변 상태(shared mutable state)** 의 문제다. 함수형 프로그래밍은 이 문제를 **불변성(immutability)** 으로 해결한다.

---

## 1. 불변성의 정의

### 1.1 불변성이란

불변성은 단순하다: **한 번 생성된 데이터는 절대 변경하지 않는다.** 변경이 필요하면 새로운 데이터를 만든다.

```typescript
// ❌ 가변 방식: 원본을 변경
function addItemMutable(cart: string[], item: string): string[] {
  cart.push(item);
  return cart;
}

// ✅ 불변 방식: 새 배열을 반환
function addItemImmutable(cart: readonly string[], item: string): readonly string[] {
  return [...cart, item]; // 새 배열 생성
}

const myCart = ["사과", "바나나"];
const newCart = addItemImmutable(myCart, "우유");

console.log(myCart);   // ["사과", "바나나"] — 원본 그대로!
console.log(newCart);  // ["사과", "바나나", "우유"] — 새 배열
console.log(myCart === newCart); // false — 다른 참조
```

### 1.2 값(Value)과 참조(Reference)

불변성을 이해하려면 **값**과 **참조**의 차이를 알아야 한다.

```typescript
// 원시 값(primitive)은 본래 불변이다
let a = 5;
let b = a;
a = 10;
console.log(b); // 5 — b는 영향 없음

// 객체(object)는 참조로 공유된다
const obj1 = { name: "Alice", age: 30 };
const obj2 = obj1;        // 같은 객체를 가리킴
obj2.age = 31;
console.log(obj1.age);    // 31 — obj1도 변경됨!
```

**참조 공유 + 변경 가능성 = 버그의 온상**이다. 불변성은 이 조합을 깨뜨린다. 참조를 공유하더라도 변경이 불가능하면 안전하다.

### 1.3 불변성의 세 가지 수준

TypeScript에서 불변성을 적용하는 수준은 세 단계로 나뉜다.

```typescript
// 수준 1: 변수 바인딩 불변 — const
const x = 5;
// x = 10; // ❌ 컴파일 에러

const arr = [1, 2, 3];
arr.push(4); // ✅ 하지만 이건 된다! const는 참조만 고정한다

// 수준 2: 타입 수준 불변 — readonly
function sum(numbers: readonly number[]): number {
  // numbers.push(4); // ❌ 컴파일 에러
  return numbers.reduce((a, b) => a + b, 0);
}

interface User {
  readonly name: string;
  readonly age: number;
  readonly address: {
    readonly city: string;    // 중첩 객체도 readonly 필요
    readonly zipCode: string;
  };
}

// 수준 3: 런타임 불변 — Object.freeze
const config = Object.freeze({
  apiUrl: "https://api.example.com",
  timeout: 5000,
});
// config.timeout = 10000; // 런타임에 조용히 무시됨 (strict mode에서는 에러)
```

**주의**: `const`는 불변이 아니다. 변수가 다른 값을 가리키는 것만 막을 뿐, 가리키는 대상의 변경은 허용한다.

---

## 2. 불변성이 해결하는 문제들

### 2.1 시간 여행 문제 (Temporal Coupling)

가변 데이터는 **언제** 읽느냐에 따라 결과가 달라진다.

```typescript
// ❌ 가변: 순서에 의존
const user = { name: "Alice", role: "user" };

function checkPermission(user: { role: string }): boolean {
  return user.role === "admin";
}

function promote(user: { role: string }): void {
  user.role = "admin";
}

console.log(checkPermission(user)); // false
promote(user);
console.log(checkPermission(user)); // true — 같은 객체, 다른 결과

// ✅ 불변: 순서에 무관
interface ImmutableUser {
  readonly name: string;
  readonly role: string;
}

function promoteImmutable(user: ImmutableUser): ImmutableUser {
  return { ...user, role: "admin" };
}

const user1: ImmutableUser = { name: "Alice", role: "user" };
const user2 = promoteImmutable(user1);

console.log(user1.role); // "user" — 변하지 않음
console.log(user2.role); // "admin" — 새 객체
```

### 2.2 앨리어싱 문제 (Aliasing)

같은 데이터를 여러 곳에서 참조할 때 하나가 변경하면 나머지에 영향을 준다.

```typescript
// ❌ 위험: 앨리어싱 + 변이
class ShoppingCart {
  items: string[] = [];

  getItems(): string[] {
    return this.items; // 내부 참조를 그대로 노출!
  }
}

const cart = new ShoppingCart();
cart.items.push("사과");

const items = cart.getItems();
items.push("해킹된 상품"); // 외부에서 내부 상태를 변경!
console.log(cart.items);   // ["사과", "해킹된 상품"] — 캡슐화 파괴

// ✅ 안전: 불변 데이터
class ImmutableCart {
  constructor(private readonly items: readonly string[] = []) {}

  getItems(): readonly string[] {
    return this.items; // 참조를 줘도 안전 — 변경 불가
  }

  add(item: string): ImmutableCart {
    return new ImmutableCart([...this.items, item]); // 새 인스턴스 반환
  }
}

const cart2 = new ImmutableCart();
const cart3 = cart2.add("사과");
const cart4 = cart3.add("바나나");

console.log(cart2.getItems()); // [] — 원본 그대로
console.log(cart3.getItems()); // ["사과"]
console.log(cart4.getItems()); // ["사과", "바나나"]
```

### 2.3 동시성 안전 (Concurrency Safety)

불변 데이터는 락(lock) 없이 안전하게 공유할 수 있다.

```typescript
// ❌ 가변 데이터 공유 — 경쟁 조건 가능
let sharedCounter = 0;

async function incrementMany(): Promise<void> {
  const promises = Array.from({ length: 100 }, async () => {
    const current = sharedCounter;
    await someAsyncWork();
    sharedCounter = current + 1; // 경쟁 조건!
  });
  await Promise.all(promises);
}

// ✅ 불변 방식 — 각자 새 값을 생성
function incrementAll(counters: readonly number[]): readonly number[] {
  return counters.map(c => c + 1); // 새 배열 반환, 공유 상태 없음
}
```

---

## 3. TypeScript의 불변성 도구

### 3.1 readonly와 Readonly 유틸리티 타입

```typescript
// readonly 배열
function processItems(items: readonly string[]): string {
  // items.push("x");    // ❌ 컴파일 에러
  // items[0] = "y";     // ❌ 컴파일 에러
  // items.sort();       // ❌ 컴파일 에러 — sort는 in-place 변이
  return items.join(", "); // ✅ 읽기 전용 메서드는 OK
}

// ReadonlyArray<T>와 readonly T[]는 같다
const nums: ReadonlyArray<number> = [1, 2, 3];

// Readonly<T> — 객체의 모든 속성을 readonly로
interface Config {
  apiUrl: string;
  timeout: number;
  retries: number;
}

function createApp(config: Readonly<Config>): void {
  // config.timeout = 10000; // ❌ 컴파일 에러
}
```

### 3.2 DeepReadonly — 깊은 불변성

`Readonly<T>`는 **얕은(shallow)** 불변이다. 중첩 객체까지 보호하려면 재귀 타입이 필요하다.

```typescript
// Readonly는 얕다
interface AppState {
  user: {
    name: string;
    settings: {
      theme: string;
    };
  };
}

const state: Readonly<AppState> = {
  user: { name: "Alice", settings: { theme: "dark" } },
};

// state.user = { ... };              // ❌ 1단계는 막힘
state.user.name = "Bob";              // ✅ 2단계는 가능! — 구멍

// 깊은 불변성을 위한 재귀 타입
type DeepReadonly<T> =
  T extends (infer U)[]
    ? DeepReadonlyArray<U>
    : T extends object
    ? DeepReadonlyObject<T>
    : T;

interface DeepReadonlyArray<T> extends ReadonlyArray<DeepReadonly<T>> {}

type DeepReadonlyObject<T> = {
  readonly [K in keyof T]: DeepReadonly<T[K]>;
};

// 이제 깊은 수준까지 보호
const deepState: DeepReadonly<AppState> = {
  user: { name: "Alice", settings: { theme: "dark" } },
};

// deepState.user.name = "Bob";           // ❌ 컴파일 에러
// deepState.user.settings.theme = "light"; // ❌ 컴파일 에러
```

### 3.3 as const — 리터럴 타입 고정

```typescript
// as const는 가장 좁은 타입 + readonly를 동시에 적용
const DIRECTIONS = ["north", "south", "east", "west"] as const;
// 타입: readonly ["north", "south", "east", "west"]

// DIRECTIONS.push("up"); // ❌ 컴파일 에러

type Direction = typeof DIRECTIONS[number]; // "north" | "south" | "east" | "west"

const CONFIG = {
  api: {
    baseUrl: "https://api.example.com",
    version: 3,
  },
  features: {
    darkMode: true,
    beta: false,
  },
} as const;

// 모든 속성이 readonly, 모든 값이 리터럴 타입
// CONFIG.api.version = 4; // ❌ 컴파일 에러
```

---

## 4. 불변 업데이트 패턴

### 4.1 스프레드 연산자 기반 업데이트

가장 기본적인 불변 업데이트 방법이다.

```typescript
interface User {
  readonly name: string;
  readonly age: number;
  readonly address: {
    readonly city: string;
    readonly zipCode: string;
  };
  readonly tags: readonly string[];
}

const user: User = {
  name: "Alice",
  age: 30,
  address: { city: "Seoul", zipCode: "06000" },
  tags: ["developer", "reader"],
};

// 1단계 업데이트: 간단
const older = { ...user, age: 31 };

// 2단계 중첩 업데이트: 점점 복잡해진다
const moved = {
  ...user,
  address: {
    ...user.address,
    city: "Busan",
  },
};

// 배열 업데이트: 추가, 제거, 수정
const withTag = { ...user, tags: [...user.tags, "writer"] };
const withoutTag = { ...user, tags: user.tags.filter(t => t !== "reader") };
const updatedTag = {
  ...user,
  tags: user.tags.map(t => t === "developer" ? "senior-developer" : t),
};
```

### 4.2 깊은 중첩의 고통

스프레드만으로는 깊은 중첩이 고통스러워진다.

```typescript
interface Company {
  readonly name: string;
  readonly departments: readonly {
    readonly name: string;
    readonly teams: readonly {
      readonly name: string;
      readonly members: readonly {
        readonly name: string;
        readonly role: string;
      }[];
    }[];
  }[];
}

// Alice의 역할을 변경하려면... 😱
function promoteAlice(company: Company): Company {
  return {
    ...company,
    departments: company.departments.map(dept =>
      dept.name !== "Engineering" ? dept : {
        ...dept,
        teams: dept.teams.map(team =>
          team.name !== "Frontend" ? team : {
            ...team,
            members: team.members.map(member =>
              member.name !== "Alice" ? member : {
                ...member,
                role: "Senior Developer",
              }
            ),
          }
        ),
      }
    ),
  };
}
// 이것이 바로 Lens가 필요한 이유다 (27주차에서 다룸)
```

### 4.3 Object.freeze vs Object.assign

```typescript
// Object.freeze — 런타임 불변성
const frozen = Object.freeze({ x: 1, y: { z: 2 } });
// frozen.x = 10;   // strict mode에서 TypeError

// ⚠️ 하지만 얕은(shallow) freeze다!
frozen.y.z = 99;    // 이건 된다! 중첩 객체는 동결되지 않음

// 깊은 freeze 유틸리티
function deepFreeze<T extends object>(obj: T): Readonly<T> {
  Object.freeze(obj);
  for (const value of Object.values(obj)) {
    if (typeof value === "object" && value !== null && !Object.isFrozen(value)) {
      deepFreeze(value);
    }
  }
  return obj;
}

const deepFrozen = deepFreeze({ x: 1, y: { z: 2 } });
// deepFrozen.y.z = 99; // TypeError!
```

---

## 5. 구조적 공유 (Structural Sharing)

### 5.1 매번 전체 복사하면 비효율적이지 않나?

불변성에 대한 가장 흔한 반론: "매번 복사하면 메모리와 성능을 낭비하지 않나?"

```typescript
// 단순한 전체 복사 — O(n) 비용
const bigArray = Array.from({ length: 10000 }, (_, i) => i);
const newArray = [...bigArray, 10000]; // 10000개를 전부 복사!
```

이 비용을 해결하는 것이 **구조적 공유**다.

### 5.2 구조적 공유의 원리

구조적 공유는 변경되지 않은 부분은 원본과 **같은 참조를 공유**하고, 변경된 부분만 새로 만든다.

```
원본 트리:          수정 후 (C를 C'로):
    A                   A'
   / \                 / \
  B   C               B   C'    ← B는 공유, C만 새로 생성
 / \   \             / \   \
D   E   F           D   E   F'  ← D, E는 공유, F만 새로
```

변경 경로에 있는 노드만 새로 만들고, 나머지는 원본과 같은 참조를 쓴다.

### 5.3 불변 리스트로 이해하는 구조적 공유

```typescript
// 함수형 연결 리스트 — 자연스러운 구조적 공유
type List<T> = null | { readonly head: T; readonly tail: List<T> };

function cons<T>(head: T, tail: List<T>): List<T> {
  return { head, tail };
}

function toArray<T>(list: List<T>): T[] {
  const result: T[] = [];
  let current = list;
  while (current !== null) {
    result.push(current.head);
    current = current.tail;
  }
  return result;
}

// 구조적 공유의 실제
const list1 = cons(1, cons(2, cons(3, null)));
// list1: 1 → 2 → 3 → null

const list2 = cons(0, list1);
// list2: 0 → 1 → 2 → 3 → null
//             ↑
//         list1과 공유!

// list2를 만들 때 list1은 복사되지 않았다
// 0 노드만 새로 만들고, tail은 list1을 그대로 가리킨다
console.log(toArray(list1)); // [1, 2, 3]
console.log(toArray(list2)); // [0, 1, 2, 3]
```

### 5.4 불변 트리맵 (Persistent TreeMap)

실전에서 사용되는 영속 자료구조의 핵심 원리를 간단한 이진 검색 트리로 이해해보자.

```typescript
// 불변 이진 검색 트리
interface TreeNode<K, V> {
  readonly key: K;
  readonly value: V;
  readonly left: TreeNode<K, V> | null;
  readonly right: TreeNode<K, V> | null;
}

function insert<K, V>(
  node: TreeNode<K, V> | null,
  key: K,
  value: V,
  compare: (a: K, b: K) => number
): TreeNode<K, V> {
  if (node === null) {
    return { key, value, left: null, right: null };
  }

  const cmp = compare(key, node.key);

  if (cmp === 0) {
    // 같은 키: 값만 교체한 새 노드 (left, right는 공유)
    return { ...node, value };
  } else if (cmp < 0) {
    // 왼쪽으로: 오른쪽 서브트리는 공유
    return { ...node, left: insert(node.left, key, value, compare) };
  } else {
    // 오른쪽으로: 왼쪽 서브트리는 공유
    return { ...node, right: insert(node.right, key, value, compare) };
  }
}

// 사용 예시
const numCompare = (a: number, b: number) => a - b;

let tree: TreeNode<number, string> | null = null;
tree = insert(tree, 5, "five", numCompare);
tree = insert(tree, 3, "three", numCompare);
tree = insert(tree, 7, "seven", numCompare);

const tree2 = insert(tree, 3, "THREE", numCompare);
// tree2에서 key=3의 값만 바뀜
// key=5, key=7 노드는 tree와 공유됨
```

### 5.5 성능 비교

```
┌───────────────────────────────────────────────────────────┐
│              연산별 시간 복잡도 비교                         │
├────────────────┬──────────────┬───────────────────────────┤
│ 연산           │ 가변 배열     │ 불변 (구조적 공유 트리)     │
├────────────────┼──────────────┼───────────────────────────┤
│ 읽기 (인덱스)   │ O(1)         │ O(log n)                  │
│ 앞에 추가       │ O(n)         │ O(1) — 연결 리스트         │
│ 뒤에 추가       │ O(1) 분할상환 │ O(log n)                  │
│ 업데이트        │ O(1)         │ O(log n)                  │
│ 동등성 비교     │ O(n)         │ O(1) — 참조 비교로 충분    │
└────────────────┴──────────────┴───────────────────────────┘
```

핵심 트레이드오프: 개별 연산은 약간 느려질 수 있지만, **변경 감지가 O(1)** 이라는 것이 프론트엔드에서 극도로 중요하다. React의 리렌더링 최적화가 바로 이것에 의존한다.

---

## 6. Immer — 불변 업데이트의 실용적 해법

### 6.1 Immer의 아이디어

Immer는 **가변 코드처럼 작성하되, 불변 결과를 생성**하는 라이브러리다.

```typescript
import { produce } from "immer";

interface State {
  readonly user: {
    readonly name: string;
    readonly address: {
      readonly city: string;
      readonly zipCode: string;
    };
  };
  readonly items: readonly string[];
}

const state: State = {
  user: {
    name: "Alice",
    address: { city: "Seoul", zipCode: "06000" },
  },
  items: ["사과", "바나나"],
};

// Immer: 가변 코드처럼 작성
const nextState = produce(state, draft => {
  draft.user.address.city = "Busan";  // 가변처럼 보이지만...
  draft.items.push("우유");
});

// 결과는 불변!
console.log(state.user.address.city);     // "Seoul" — 원본 유지
console.log(nextState.user.address.city); // "Busan" — 새 객체

// 구조적 공유도 자동!
console.log(state.user.name === nextState.user.name); // true — 문자열 공유
console.log(state.user === nextState.user);           // false — 경로상의 객체는 새로 생성
```

### 6.2 Immer의 동작 원리 — Proxy

Immer는 JavaScript의 `Proxy`를 활용한다.

```typescript
// Immer가 내부적으로 하는 일 (단순화)
function simpleProduce<T extends object>(base: T, recipe: (draft: T) => void): T {
  // 1. 변경 추적을 위한 Proxy 생성
  const changes = new Map<string, unknown>();
  const proxy = new Proxy(base, {
    set(target, prop, value) {
      changes.set(String(prop), value); // 변경 기록
      return true;
    },
    get(target, prop) {
      if (changes.has(String(prop))) {
        return changes.get(String(prop));
      }
      return (target as Record<string, unknown>)[String(prop)];
    },
  });

  // 2. recipe 실행 — 변경 사항 수집
  recipe(proxy);

  // 3. 변경된 부분만 새 객체로 생성 (구조적 공유)
  if (changes.size === 0) return base; // 변경 없으면 원본 반환
  return { ...base, ...Object.fromEntries(changes) } as T;
}
```

### 6.3 Immer vs 스프레드 vs Object.freeze

```
┌──────────────────────────────────────────────────────────────┐
│                    불변성 도구 비교                            │
├───────────────┬───────────┬───────────┬──────────────────────┤
│               │ 스프레드   │ freeze    │ Immer                │
├───────────────┼───────────┼───────────┼──────────────────────┤
│ 보호 시점      │ 없음       │ 런타임    │ 런타임               │
│ 깊은 보호      │ ❌         │ 얕음      │ ✅ 자동              │
│ 구조적 공유    │ 수동       │ ❌        │ ✅ 자동              │
│ 중첩 업데이트  │ 고통스러움  │ 불가      │ ✅ 직관적            │
│ 타입 안전성    │ ✅         │ 부분적    │ ✅                   │
│ 번들 크기      │ 0          │ 0        │ ~6KB (gzip)          │
│ 학습 비용      │ 없음       │ 없음      │ 낮음                │
│ 디버깅         │ 쉬움       │ 보통      │ Proxy 때문에 약간 복잡│
└───────────────┴───────────┴───────────┴──────────────────────┘
```

---

## 7. 영속 자료구조 (Persistent Data Structure)

### 7.1 정의

영속 자료구조는 **수정 후에도 이전 버전이 보존되는** 자료구조다. 모든 버전에 접근 가능하다.

```
일반 자료구조 (in-place 변이):
  v1: [1, 2, 3]
  v1.push(4)
  v1: [1, 2, 3, 4]  ← v1의 원래 상태는 사라짐

영속 자료구조:
  v1: [1, 2, 3]
  v2 = v1.append(4)
  v1: [1, 2, 3]      ← 여전히 접근 가능
  v2: [1, 2, 3, 4]   ← 새 버전
```

### 7.2 Hash Array Mapped Trie (HAMT)

실전에서 사용되는 영속 자료구조의 핵심은 **HAMT**다. Clojure, Scala, Immutable.js 등에서 사용한다.

```
HAMT의 핵심 아이디어:
- 32-way branching trie (각 노드가 최대 32개 자식)
- 해시값을 5비트씩 잘라서 트리 경로 결정
- 비트맵으로 희소 노드 압축

깊이 7이면 32^7 = 약 350억 개의 원소 저장 가능
대부분의 연산: O(log32 n) ≈ O(7) → 사실상 O(1)

  수정 시:
  ┌──────────────────────────────────┐
  │     Root                         │
  │    /    \                        │
  │   A      B     ← B를 수정하면    │
  │  / \    / \                      │
  │ C   D  E   F                     │
  └──────────────────────────────────┘
            ↓
  ┌──────────────────────────────────┐
  │     Root'                        │
  │    /    \                        │
  │   A      B'    ← B'와 Root'만    │
  │  / \    / \       새로 생성       │
  │ C   D  E   F' ← A,C,D,E는 공유  │
  └──────────────────────────────────┘
```

### 7.3 간단한 영속 벡터 구현

HAMT의 핵심 원리를 간단한 형태로 구현해보자.

```typescript
// 간단한 영속 벡터 (branching factor = 4 로 단순화)
const BITS = 2;        // log2(4) = 2
const WIDTH = 1 << BITS; // 4
const MASK = WIDTH - 1;  // 0b11

type Node<T> = T[] | Node<T>[];

class PersistentVector<T> {
  private constructor(
    private readonly root: Node<T>,
    private readonly depth: number,
    readonly length: number
  ) {}

  static empty<T>(): PersistentVector<T> {
    return new PersistentVector<T>([], 0, 0);
  }

  // 인덱스로 값 읽기 — O(log4 n)
  get(index: number): T | undefined {
    if (index < 0 || index >= this.length) return undefined;

    let node = this.root;
    for (let level = this.depth * BITS; level > 0; level -= BITS) {
      const idx = (index >> level) & MASK;
      node = (node as Node<T>[])[idx];
    }
    return (node as T[])[index & MASK];
  }

  // 끝에 추가 — O(log4 n), 구조적 공유
  append(value: T): PersistentVector<T> {
    const newLength = this.length + 1;

    if (this.length === 0) {
      return new PersistentVector<T>([value], 0, 1);
    }

    // 현재 깊이에서 공간이 있는 경우
    if (this.length < Math.pow(WIDTH, this.depth + 1)) {
      const newRoot = this.appendToNode(this.root, this.depth, this.length, value);
      return new PersistentVector<T>(newRoot, this.depth, newLength);
    }

    // 트리 깊이를 늘려야 하는 경우
    const newRoot: Node<T>[] = [this.root];
    const path = this.createPath(this.depth, value);
    newRoot.push(path);
    return new PersistentVector<T>(newRoot, this.depth + 1, newLength);
  }

  private appendToNode(node: Node<T>, depth: number, index: number, value: T): Node<T> {
    const newNode = [...node]; // 경로 상의 노드만 복사 (구조적 공유!)

    if (depth === 0) {
      newNode.push(value);
      return newNode;
    }

    const subIndex = (index >> (depth * BITS)) & MASK;
    if (subIndex < newNode.length) {
      newNode[subIndex] = this.appendToNode(
        (newNode as Node<T>[])[subIndex], depth - 1, index, value
      );
    } else {
      (newNode as Node<T>[]).push(this.createPath(depth - 1, value));
    }

    return newNode;
  }

  private createPath(depth: number, value: T): Node<T> {
    if (depth === 0) return [value];
    return [this.createPath(depth - 1, value)];
  }
}

// 사용
let vec = PersistentVector.empty<number>();
vec = vec.append(1);
vec = vec.append(2);
vec = vec.append(3);

console.log(vec.get(0)); // 1
console.log(vec.get(1)); // 2
console.log(vec.get(2)); // 3
console.log(vec.length); // 3
```

---

## 8. 프론트엔드 실전

### 8.1 React 상태의 불변 업데이트

React는 상태가 **참조적으로 다른 객체**일 때만 리렌더링한다. 이것이 불변 업데이트가 필수인 이유다.

```typescript
interface TodoState {
  readonly todos: readonly {
    readonly id: number;
    readonly text: string;
    readonly done: boolean;
  }[];
  readonly filter: "all" | "active" | "done";
}

function TodoApp(): JSX.Element {
  const [state, setState] = useState<TodoState>({
    todos: [],
    filter: "all",
  });

  // ❌ 가변 업데이트 — React가 변경을 감지하지 못함
  const addTodoBad = (text: string) => {
    state.todos.push({ id: Date.now(), text, done: false });
    setState(state); // 같은 참조! 리렌더링 안 됨!
  };

  // ✅ 불변 업데이트 — 새 객체 생성
  const addTodo = (text: string) => {
    setState(prev => ({
      ...prev,
      todos: [...prev.todos, { id: Date.now(), text, done: false }],
    }));
  };

  const toggleTodo = (id: number) => {
    setState(prev => ({
      ...prev,
      todos: prev.todos.map(todo =>
        todo.id === id ? { ...todo, done: !todo.done } : todo
      ),
    }));
  };

  const removeTodo = (id: number) => {
    setState(prev => ({
      ...prev,
      todos: prev.todos.filter(todo => todo.id !== id),
    }));
  };

  // 필터링은 파생 데이터 — 상태를 변경하지 않고 계산
  const filteredTodos = useMemo(() => {
    switch (state.filter) {
      case "all": return state.todos;
      case "active": return state.todos.filter(t => !t.done);
      case "done": return state.todos.filter(t => t.done);
    }
  }, [state.todos, state.filter]);

  return (
    <ul>
      {filteredTodos.map(todo => (
        <li key={todo.id} onClick={() => toggleTodo(todo.id)}>
          {todo.done ? "✓" : "○"} {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

### 8.2 useReducer와 Redux 리듀서

리듀서는 본질적으로 `(State, Action) → State` 순수 함수이며, 반드시 불변 업데이트를 해야 한다.

```typescript
// 액션 타입 정의
type Action =
  | { type: "ADD_TODO"; payload: { text: string } }
  | { type: "TOGGLE_TODO"; payload: { id: number } }
  | { type: "REMOVE_TODO"; payload: { id: number } }
  | { type: "SET_FILTER"; payload: { filter: "all" | "active" | "done" } };

// 순수 함수 리듀서 — 상태를 직접 변경하지 않음
function todoReducer(state: TodoState, action: Action): TodoState {
  switch (action.type) {
    case "ADD_TODO":
      return {
        ...state,
        todos: [
          ...state.todos,
          { id: Date.now(), text: action.payload.text, done: false },
        ],
      };

    case "TOGGLE_TODO":
      return {
        ...state,
        todos: state.todos.map(todo =>
          todo.id === action.payload.id
            ? { ...todo, done: !todo.done }
            : todo
        ),
      };

    case "REMOVE_TODO":
      return {
        ...state,
        todos: state.todos.filter(todo => todo.id !== action.payload.id),
      };

    case "SET_FILTER":
      return { ...state, filter: action.payload.filter };
  }
}

// useReducer로 사용
function TodoAppWithReducer(): JSX.Element {
  const [state, dispatch] = useReducer(todoReducer, {
    todos: [],
    filter: "all",
  });

  // 리듀서 덕분에 상태 로직이 컴포넌트에서 분리됨
  // 리듀서는 순수 함수이므로 테스트가 간단함
  return (
    <button onClick={() => dispatch({ type: "ADD_TODO", payload: { text: "새 할 일" } })}>
      추가
    </button>
  );
}

// 리듀서 테스트 — 순수 함수이므로 mock 불필요
describe("todoReducer", () => {
  it("adds a todo", () => {
    const state: TodoState = { todos: [], filter: "all" };
    const next = todoReducer(state, {
      type: "ADD_TODO",
      payload: { text: "테스트" },
    });

    expect(next.todos).toHaveLength(1);
    expect(next.todos[0].text).toBe("테스트");
    expect(next.todos[0].done).toBe(false);
    expect(state.todos).toHaveLength(0); // 원본 불변!
  });

  it("toggles a todo", () => {
    const state: TodoState = {
      todos: [{ id: 1, text: "할 일", done: false }],
      filter: "all",
    };
    const next = todoReducer(state, {
      type: "TOGGLE_TODO",
      payload: { id: 1 },
    });

    expect(next.todos[0].done).toBe(true);
    expect(state.todos[0].done).toBe(false); // 원본 불변!
  });
});
```

### 8.3 React에서 불변성이 중요한 이유 — 참조 동등성

```typescript
// React.memo와 불변성
const TodoItem = React.memo(function TodoItem({
  todo,
  onToggle,
}: {
  todo: { id: number; text: string; done: boolean };
  onToggle: (id: number) => void;
}) {
  console.log(`Rendering: ${todo.text}`);
  return (
    <li onClick={() => onToggle(todo.id)}>
      {todo.done ? "✓" : "○"} {todo.text}
    </li>
  );
});

// ✅ 불변 업데이트: 변경되지 않은 todo 객체는 같은 참조
// → React.memo가 리렌더링을 건너뜀
// → 100개 todo 중 1개만 토글하면 1개만 리렌더링

// ❌ 전체 복사: 모든 todo가 새 객체
// → React.memo가 전부 다른 것으로 판단
// → 100개 전부 리렌더링

// useMemo와 불변성
function Dashboard({ data }: { data: readonly DataPoint[] }) {
  // data 참조가 같으면 재계산하지 않음
  const summary = useMemo(() => {
    return {
      total: data.reduce((sum, d) => sum + d.value, 0),
      average: data.reduce((sum, d) => sum + d.value, 0) / data.length,
      max: Math.max(...data.map(d => d.value)),
    };
  }, [data]); // 참조 비교 — O(1)

  return <SummaryCard summary={summary} />;
}
```

### 8.4 깊은 객체의 렌즈 패턴 맛보기

27주차에서 본격적으로 다루지만, 핵심 아이디어를 미리 맛보자.

```typescript
// 렌즈: getter와 setter의 쌍
interface Lens<S, A> {
  get: (s: S) => A;
  set: (a: A) => (s: S) => S;
}

// 렌즈 생성 헬퍼
function lens<S, A>(
  get: (s: S) => A,
  set: (a: A) => (s: S) => S
): Lens<S, A> {
  return { get, set };
}

// modify: get으로 읽고, 변환하고, set으로 쓰기
function modify<S, A>(l: Lens<S, A>, f: (a: A) => A): (s: S) => S {
  return s => l.set(f(l.get(s)))(s);
}

// 렌즈 합성 — 깊은 접근을 연결
function compose<S, A, B>(outer: Lens<S, A>, inner: Lens<A, B>): Lens<S, B> {
  return {
    get: s => inner.get(outer.get(s)),
    set: b => s => outer.set(inner.set(b)(outer.get(s)))(s),
  };
}

// 사용 예시
interface Address {
  readonly city: string;
  readonly zipCode: string;
}

interface Person {
  readonly name: string;
  readonly address: Address;
}

// 각 단계의 렌즈 정의
const addressLens: Lens<Person, Address> = lens(
  p => p.address,
  a => p => ({ ...p, address: a })
);

const cityLens: Lens<Address, string> = lens(
  a => a.city,
  c => a => ({ ...a, city: c })
);

// 합성: Person → Address → city
const personCityLens = compose(addressLens, cityLens);

const alice: Person = {
  name: "Alice",
  address: { city: "Seoul", zipCode: "06000" },
};

// 깊은 중첩도 깔끔하게!
const movedAlice = personCityLens.set("Busan")(alice);
console.log(movedAlice.address.city); // "Busan"
console.log(alice.address.city);      // "Seoul" — 원본 불변

// modify로 변환 적용
const upperCity = modify(personCityLens, city => city.toUpperCase());
console.log(upperCity(alice).address.city); // "SEOUL"
```

---

## 연습 문제

### 문제 1: 불변 업데이트 작성

다음 상태를 불변하게 업데이트하는 함수를 작성하라 (스프레드 연산자 사용).

```typescript
interface AppState {
  readonly user: {
    readonly name: string;
    readonly preferences: {
      readonly theme: "light" | "dark";
      readonly language: string;
      readonly notifications: {
        readonly email: boolean;
        readonly push: boolean;
      };
    };
  };
  readonly posts: readonly {
    readonly id: number;
    readonly title: string;
    readonly likes: number;
  }[];
}

// (a) theme을 "dark"로 변경
function setTheme(state: AppState, theme: "light" | "dark"): AppState {
  // 구현하라
}

// (b) 특정 post의 likes를 1 증가
function likePost(state: AppState, postId: number): AppState {
  // 구현하라
}

// (c) email 알림을 토글
function toggleEmailNotification(state: AppState): AppState {
  // 구현하라
}
```

### 문제 2: 구조적 공유 확인

다음 코드에서 `===` 비교의 결과를 예측하고 이유를 설명하라.

```typescript
const original = {
  a: { x: 1, y: 2 },
  b: { x: 3, y: 4 },
  c: [1, 2, 3],
};

const modified = {
  ...original,
  a: { ...original.a, x: 10 },
};

// (a) original === modified
// (b) original.a === modified.a
// (c) original.b === modified.b
// (d) original.c === modified.c
// (e) original.a.y === modified.a.y
```

### 문제 3: 불변 스택 구현

연결 리스트 기반의 불변 스택을 구현하라. 모든 연산이 원본을 변경하지 않아야 한다.

```typescript
interface ImmutableStack<T> {
  push(value: T): ImmutableStack<T>;
  pop(): { value: T; rest: ImmutableStack<T> } | null;
  peek(): T | undefined;
  isEmpty(): boolean;
  toArray(): T[];
}

// 구현하라
function createStack<T>(): ImmutableStack<T> {
  // ...
}
```

### 문제 4: Immer vs 스프레드 비교

다음 Immer 코드와 동일한 결과를 내는 스프레드 기반 코드를 작성하라.

```typescript
import { produce } from "immer";

interface GameState {
  readonly players: readonly {
    readonly id: number;
    readonly name: string;
    readonly score: number;
    readonly inventory: readonly string[];
  }[];
  readonly round: number;
  readonly settings: {
    readonly difficulty: "easy" | "normal" | "hard";
    readonly maxPlayers: number;
  };
}

const nextState = produce(gameState, draft => {
  // 라운드 증가
  draft.round += 1;
  // 플레이어 1의 점수 증가 및 아이템 추가
  const player = draft.players.find(p => p.id === 1);
  if (player) {
    player.score += 100;
    player.inventory.push("마법 검");
  }
  // 난이도 변경
  draft.settings.difficulty = "hard";
});

// 위와 동일한 결과를 스프레드로 구현하라
function updateGameState(state: GameState): GameState {
  // 구현하라
}
```

---

## 연습 문제 해답

### 문제 1 해답

```typescript
// (a) theme 변경 — 3단계 중첩 업데이트
function setTheme(state: AppState, theme: "light" | "dark"): AppState {
  return {
    ...state,
    user: {
      ...state.user,
      preferences: {
        ...state.user.preferences,
        theme,
      },
    },
  };
}

// (b) 특정 post의 likes 증가 — 배열 내 특정 요소 업데이트
function likePost(state: AppState, postId: number): AppState {
  return {
    ...state,
    posts: state.posts.map(post =>
      post.id === postId
        ? { ...post, likes: post.likes + 1 }
        : post
    ),
  };
}

// (c) email 알림 토글 — 4단계 중첩!
function toggleEmailNotification(state: AppState): AppState {
  return {
    ...state,
    user: {
      ...state.user,
      preferences: {
        ...state.user.preferences,
        notifications: {
          ...state.user.preferences.notifications,
          email: !state.user.preferences.notifications.email,
        },
      },
    },
  };
}
// 이런 코드가 바로 Immer나 Lens가 필요한 이유다
```

### 문제 2 해답

```typescript
// (a) original === modified → false
// 스프레드로 새 객체를 만들었으므로 참조가 다르다

// (b) original.a === modified.a → false
// a를 새로 스프레드했으므로 새 객체다

// (c) original.b === modified.b → true ✨
// b는 건드리지 않았다. 스프레드는 얕은 복사이므로
// modified.b는 original.b와 같은 참조를 공유한다 — 이것이 구조적 공유!

// (d) original.c === modified.c → true ✨
// c도 건드리지 않았으므로 같은 참조. 배열도 마찬가지로 공유된다.

// (e) original.a.y === modified.a.y → true
// 숫자(원시값)는 값으로 비교된다. 둘 다 2이므로 true.
// 단, 여기서 '==='는 값 비교가 아닌 원시값 동등 비교다.
// 만약 y가 객체였다면? a를 새로 만들 때 y의 참조는 복사되므로 여전히 true.
```

### 문제 3 해답

```typescript
type StackNode<T> = { readonly value: T; readonly next: StackNode<T> | null };

function createStack<T>(): ImmutableStack<T> {
  return makeStack<T>(null, 0);
}

function makeStack<T>(top: StackNode<T> | null, size: number): ImmutableStack<T> {
  return {
    push(value: T): ImmutableStack<T> {
      // 새 노드가 기존 top을 가리킴 — 구조적 공유!
      return makeStack({ value, next: top }, size + 1);
    },

    pop(): { value: T; rest: ImmutableStack<T> } | null {
      if (top === null) return null;
      return {
        value: top.value,
        rest: makeStack(top.next, size - 1), // 기존 노드 재사용
      };
    },

    peek(): T | undefined {
      return top?.value;
    },

    isEmpty(): boolean {
      return top === null;
    },

    toArray(): T[] {
      const result: T[] = [];
      let current = top;
      while (current !== null) {
        result.push(current.value);
        current = current.next;
      }
      return result;
    },
  };
}

// 검증
const s0 = createStack<number>();          // []
const s1 = s0.push(1);                     // [1]
const s2 = s1.push(2);                     // [2, 1]
const s3 = s2.push(3);                     // [3, 2, 1]

console.log(s3.toArray());                 // [3, 2, 1]
console.log(s3.peek());                    // 3

const popped = s3.pop()!;
console.log(popped.value);                 // 3
console.log(popped.rest.toArray());        // [2, 1]

// 원본은 변하지 않음!
console.log(s1.toArray());                 // [1]
console.log(s2.toArray());                 // [2, 1]
console.log(s3.toArray());                 // [3, 2, 1]
```

### 문제 4 해답

```typescript
function updateGameState(state: GameState): GameState {
  return {
    ...state,
    round: state.round + 1,
    players: state.players.map(player =>
      player.id === 1
        ? {
            ...player,
            score: player.score + 100,
            inventory: [...player.inventory, "마법 검"],
          }
        : player
    ),
    settings: {
      ...state.settings,
      difficulty: "hard",
    },
  };
}

// Immer 버전과 비교:
// - Immer: 9줄, 직관적, "무엇을 바꿀지"에 집중
// - 스프레드: 15줄, 구조를 따라가야 함, 중첩이 깊으면 고통
// - 두 방식 모두 구조적 공유가 일어남 (변경되지 않은 player는 같은 참조)
// - Immer는 Proxy로 자동 처리, 스프레드는 개발자가 수동으로 보장
```

---

## 핵심 정리

| 개념 | 한 줄 요약 |
|------|-----------|
| 불변성 | 한 번 생성된 데이터는 변경하지 않고, 새 데이터를 만든다 |
| const | 변수 바인딩만 고정, 내부 값 변경은 허용 — 불변이 아님 |
| readonly | 타입 수준의 불변성, 컴파일 타임에만 보호 |
| Object.freeze | 런타임 불변, 얕은 보호만 제공 |
| 구조적 공유 | 변경되지 않은 부분은 원본과 같은 참조를 공유 |
| 영속 자료구조 | 수정 후에도 이전 버전이 보존되는 자료구조 |
| HAMT | 32-way 트라이 기반 영속 자료구조, 사실상 O(1) 연산 |
| Immer | Proxy 기반, 가변 코드처럼 작성하되 불변 결과 생성 |
| 렌즈 | getter/setter 쌍, 깊은 중첩 불변 업데이트를 합성 가능하게 |
| React와 불변성 | 참조 비교로 변경 감지, React.memo와 useMemo의 전제 조건 |

## 다음 주차 예고

**4주차: 합성과 파이프라인** — 순수 함수를 작은 단위로 만들고, 이들을 합성(compose)하여 복잡한 동작을 만든다. `pipe`와 `flow`를 직접 구현하고, 데이터 변환 파이프라인을 설계하는 방법을 배운다.
