# Effect-TS 학습 가이드

> 비트코인 차트 메모 기능 구현을 위한 Effect-TS 완벽 가이드

## 목차

1. [Phase 1: Effect 기초 개념](#phase-1-effect-기초-개념)
2. [Phase 2: Schema와 데이터 검증](#phase-2-schema와-데이터-검증)
3. [Phase 3: Service와 Layer](#phase-3-service와-layer)
4. [Phase 4: React 통합](#phase-4-react-통합)
5. [학습 완료 체크리스트](#학습-완료-체크리스트)

---

## Phase 1: Effect 기초 개념 (1-2일)

### 1.1 Effect란 무엇인가?

Effect는 TypeScript를 위한 강력한 함수형 프로그래밍 라이브러리입니다. 비동기 작업, 에러 핸들링, 의존성 관리를 타입 안전하게 처리할 수 있습니다.

**핵심 개념: Effect 타입**

```typescript
Effect<Success, Error, Requirements>
```

- **Success**: 성공 시 반환되는 값의 타입
- **Error**: 실패 시 발생할 수 있는 에러의 타입
- **Requirements**: 이 Effect를 실행하는데 필요한 의존성 (Context)

### 1.2 Effect 생성하기

#### 성공 케이스

```typescript
import { Effect } from "effect"

// 단순 값을 Effect로 감싸기
const success = Effect.succeed(42)
// Effect<number, never, never>

// 실행
const result = await Effect.runPromise(success)
console.log(result) // 42
```

#### 동기 함수를 Effect로

```typescript
import { Effect } from "effect"

// 동기 함수를 Effect로 변환
const getCurrentTime = Effect.sync(() => new Date())
// Effect<Date, never, never>

const getRandomNumber = Effect.sync(() => Math.random())
// Effect<number, never, never>

// 실행
const time = await Effect.runPromise(getCurrentTime)
console.log(time) // 현재 시간
```

#### 실패 가능한 Effect

```typescript
import { Effect } from "effect"

// 에러를 발생시킬 수 있는 Effect
const divide = (a: number, b: number) =>
  b === 0
    ? Effect.fail("Division by zero")
    : Effect.succeed(a / b)

// Effect<number, string, never>
//        ↑      ↑
//     성공 타입  에러 타입

// 성공 케이스
const result1 = await Effect.runPromise(divide(10, 2))
console.log(result1) // 5

// 실패 케이스 (에러 발생)
try {
  await Effect.runPromise(divide(10, 0))
} catch (error) {
  console.log(error) // "Division by zero"
}
```

#### 비동기 Promise를 Effect로

```typescript
import { Effect } from "effect"

// Promise를 Effect로 변환
const fetchUser = (id: string) =>
  Effect.tryPromise({
    try: () => fetch(`https://api.example.com/users/${id}`)
      .then(res => res.json()),
    catch: (error) => new Error(`Failed to fetch user: ${error}`)
  })

// Effect<User, Error, never>

// 사용 예시
const user = await Effect.runPromise(fetchUser("123"))
```

### 1.3 파이프라인 (pipe)

Effect는 `pipe` 함수를 사용하여 여러 변환을 체이닝할 수 있습니다.

```typescript
import { Effect, pipe } from "effect"

const program = pipe(
  Effect.succeed(10),
  Effect.map(x => x * 2),      // 10 * 2 = 20
  Effect.map(x => x + 5),      // 20 + 5 = 25
  Effect.map(x => x.toString()) // "25"
)

const result = await Effect.runPromise(program)
console.log(result) // "25"
```

### 1.4 에러 핸들링

#### catchAll - 모든 에러 처리

```typescript
import { Effect, pipe } from "effect"

const riskyOperation = Effect.fail("Something went wrong")

const safe = pipe(
  riskyOperation,
  Effect.catchAll(error => {
    console.log("Caught error:", error)
    return Effect.succeed("Default value")
  })
)

const result = await Effect.runPromise(safe)
console.log(result) // "Default value"
```

#### orElse - 대체 Effect 제공

```typescript
import { Effect, pipe } from "effect"

const primary = Effect.fail("Primary failed")
const fallback = Effect.succeed("Fallback value")

const result = pipe(
  primary,
  Effect.orElse(() => fallback)
)

const value = await Effect.runPromise(result)
console.log(value) // "Fallback value"
```

#### match - 성공/실패 분기

```typescript
import { Effect, pipe } from "effect"

const operation = divide(10, 2)

const result = pipe(
  operation,
  Effect.match({
    onFailure: (error) => `Error: ${error}`,
    onSuccess: (value) => `Result: ${value}`
  })
)

const message = await Effect.runPromise(result)
console.log(message) // "Result: 5"
```

### 1.5 실습 예제

다음 요구사항을 Effect로 구현해보세요:

#### 연습 문제 1: 사용자 검증

```typescript
// TODO: 다음 함수를 Effect로 구현하세요
// - 이메일이 비어있으면 "Email is required" 에러
// - 이메일에 @가 없으면 "Invalid email format" 에러
// - 성공 시 이메일 반환

interface User {
  email: string
}

const validateEmail = (email: string): Effect.Effect<string, string, never> => {
  // 여기에 구현
  if (!email) {
    return Effect.fail("Email is required")
  }
  if (!email.includes("@")) {
    return Effect.fail("Invalid email format")
  }
  return Effect.succeed(email)
}

// 테스트
await Effect.runPromise(validateEmail("test@example.com")) // "test@example.com"
await Effect.runPromise(validateEmail("invalid")) // Error: "Invalid email format"
```

#### 연습 문제 2: API 호출 재시도

```typescript
// TODO: fetch를 Effect로 감싸고, 실패 시 기본값 반환
const fetchWithFallback = (url: string, fallback: any) =>
  pipe(
    Effect.tryPromise({
      try: () => fetch(url).then(res => res.json()),
      catch: (error) => new Error(`Fetch failed: ${error}`)
    }),
    Effect.catchAll(() => Effect.succeed(fallback))
  )

// 테스트
const data = await Effect.runPromise(
  fetchWithFallback("https://invalid-url.com", { default: true })
)
console.log(data) // { default: true }
```

### 1.6 다음 단계로 넘어가는 기준

다음을 이해했다면 Phase 2로 넘어가세요:

- ✅ Effect 타입의 세 가지 매개변수 (Success, Error, Requirements) 이해
- ✅ Effect 생성 방법 (succeed, fail, sync, tryPromise)
- ✅ pipe를 사용한 체이닝
- ✅ 기본적인 에러 핸들링 (catchAll, orElse, match)

---

## Phase 2: Schema와 데이터 검증 (1-2일)

### 2.1 @effect/schema란?

Effect Schema는 런타임 타입 검증과 파싱을 제공하는 라이브러리입니다. TypeScript 타입과 런타임 검증을 동시에 보장합니다.

### 2.2 기본 Schema 정의

```typescript
import { Schema } from "@effect/schema"

// 문자열 스키마
const StringSchema = Schema.String
type StringType = typeof StringSchema.Type // string

// 숫자 스키마
const NumberSchema = Schema.Number
type NumberType = typeof NumberSchema.Type // number

// 불린 스키마
const BooleanSchema = Schema.Boolean
type BooleanType = typeof BooleanSchema.Type // boolean
```

### 2.3 객체 Schema (Struct)

```typescript
import { Schema } from "@effect/schema"

// 사용자 스키마
const UserSchema = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  age: Schema.Number,
  email: Schema.String
})

// 타입 추론
type User = typeof UserSchema.Type
// {
//   id: string
//   name: string
//   age: number
//   email: string
// }
```

### 2.4 검증 규칙 추가

```typescript
import { Schema } from "@effect/schema"

const UserSchema = Schema.Struct({
  id: Schema.String.pipe(Schema.uuid()), // UUID 형식
  name: Schema.String.pipe(
    Schema.minLength(2),
    Schema.maxLength(50)
  ),
  age: Schema.Number.pipe(
    Schema.int(),          // 정수만
    Schema.positive(),     // 양수만
    Schema.lessThanOrEqualTo(150)
  ),
  email: Schema.String.pipe(
    Schema.pattern(/^[^\s@]+@[^\s@]+\.[^\s@]+$/) // 이메일 형식
  )
})

type User = typeof UserSchema.Type
```

### 2.5 데이터 파싱 (decode)

```typescript
import { Schema } from "@effect/schema"
import { Effect } from "effect"

const UserSchema = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  age: Schema.Number
})

// 파싱 함수 생성
const parseUser = Schema.decodeUnknown(UserSchema)

// 성공 케이스
const validData = {
  id: "123",
  name: "John",
  age: 30
}

const userEffect = parseUser(validData)
// Effect<User, ParseError, never>

const user = await Effect.runPromise(userEffect)
console.log(user) // { id: "123", name: "John", age: 30 }

// 실패 케이스
const invalidData = {
  id: "123",
  name: "John",
  age: "thirty" // 숫자가 아님!
}

try {
  await Effect.runPromise(parseUser(invalidData))
} catch (error) {
  console.log("Parse error:", error)
}
```

### 2.6 선택적 필드 (optional)

```typescript
import { Schema } from "@effect/schema"

const UserSchema = Schema.Struct({
  id: Schema.String,
  name: Schema.String,
  age: Schema.Number,
  bio: Schema.optional(Schema.String) // 선택적 필드
})

type User = typeof UserSchema.Type
// {
//   id: string
//   name: string
//   age: number
//   bio?: string
// }

const user1 = { id: "1", name: "John", age: 30 } // OK
const user2 = { id: "1", name: "John", age: 30, bio: "Developer" } // OK
```

### 2.7 배열 Schema

```typescript
import { Schema } from "@effect/schema"

const UserArraySchema = Schema.Array(UserSchema)

type Users = typeof UserArraySchema.Type // User[]

const parseUsers = Schema.decodeUnknown(UserArraySchema)

const users = await Effect.runPromise(
  parseUsers([
    { id: "1", name: "John", age: 30 },
    { id: "2", name: "Jane", age: 25 }
  ])
)
```

### 2.8 실습 예제: 메모 Schema

#### 연습 문제 3: 메모 스키마 작성

```typescript
import { Schema } from "@effect/schema"

// TODO: 다음 요구사항을 만족하는 메모 스키마를 작성하세요
// - id: UUID 형식의 문자열
// - userId: UUID 형식의 문자열
// - timestamp: 양수인 숫자 (Unix timestamp)
// - content: 1-1000자 사이의 문자열
// - createdAt: Date 객체
// - updatedAt: Date 객체

const MemoSchema = Schema.Struct({
  id: Schema.String.pipe(Schema.uuid()),
  userId: Schema.String.pipe(Schema.uuid()),
  timestamp: Schema.Number.pipe(Schema.positive()),
  content: Schema.String.pipe(
    Schema.minLength(1),
    Schema.maxLength(1000)
  ),
  createdAt: Schema.Date,
  updatedAt: Schema.Date
})

type Memo = typeof MemoSchema.Type

// 테스트
const parseMemo = Schema.decodeUnknown(MemoSchema)

const validMemo = {
  id: "550e8400-e29b-41d4-a716-446655440000",
  userId: "550e8400-e29b-41d4-a716-446655440001",
  timestamp: 1638360000000,
  content: "BTC 가격 상승 예상",
  createdAt: new Date(),
  updatedAt: new Date()
}

const memo = await Effect.runPromise(parseMemo(validMemo))
console.log(memo)
```

### 2.9 다음 단계로 넘어가는 기준

- ✅ Schema.Struct로 객체 타입 정의 가능
- ✅ pipe를 사용한 검증 규칙 추가 이해
- ✅ Schema.decodeUnknown으로 데이터 파싱 가능
- ✅ Schema.optional, Schema.Array 사용 가능

---

## Phase 3: Service와 Layer (2-3일)

### 3.1 왜 Service와 Layer가 필요한가?

**문제점:**

```typescript
// 나쁜 예: 직접 의존성 사용
async function createUser(name: string) {
  const db = new Database() // 하드코딩된 의존성
  return db.insert({ name })
}

// 테스트하기 어려움!
```

**해결책: 의존성 주입**

```typescript
// 좋은 예: 의존성을 주입받음
async function createUser(db: Database, name: string) {
  return db.insert({ name })
}

// 테스트 시 Mock DB를 주입 가능
```

Effect의 Service와 Layer는 이를 타입 안전하게 구현합니다.

### 3.2 Context.Tag로 서비스 정의

```typescript
import { Context, Effect } from "effect"

// 1. 서비스 인터페이스 정의
interface DatabaseService {
  readonly query: (sql: string) => Effect.Effect<any[], Error>
  readonly insert: (data: any) => Effect.Effect<void, Error>
}

// 2. Context Tag 생성
const DatabaseService = Context.GenericTag<DatabaseService>("@services/Database")
```

### 3.3 서비스 사용하기

```typescript
import { Effect } from "effect"

// 서비스를 사용하는 Effect
const getUsers = Effect.gen(function* () {
  // DatabaseService를 요청
  const db = yield* DatabaseService

  // 서비스 메서드 호출
  const users = yield* db.query("SELECT * FROM users")

  return users
})

// Effect<User[], Error, DatabaseService>
//                            ↑
//                      Requirements에 DatabaseService 필요
```

### 3.4 Layer로 서비스 구현 제공

```typescript
import { Layer, Effect } from "effect"

// 실제 데이터베이스 구현
const DatabaseServiceLive = Layer.succeed(DatabaseService, {
  query: (sql: string) =>
    Effect.tryPromise({
      try: async () => {
        // 실제 DB 쿼리
        const result = await realDatabase.query(sql)
        return result
      },
      catch: (error) => new Error(`Query failed: ${error}`)
    }),

  insert: (data: any) =>
    Effect.tryPromise({
      try: async () => {
        await realDatabase.insert(data)
      },
      catch: (error) => new Error(`Insert failed: ${error}`)
    })
})
```

### 3.5 Layer 제공 및 실행

```typescript
import { Effect } from "effect"

const program = Effect.gen(function* () {
  const db = yield* DatabaseService
  const users = yield* db.query("SELECT * FROM users")
  return users
})

// Layer를 제공하고 실행
const result = await Effect.runPromise(
  program.pipe(Effect.provide(DatabaseServiceLive))
)
```

### 3.6 실습 예제: MemoService

#### 연습 문제 4: MemoService 구현

```typescript
import { Context, Effect, Layer } from "effect"

// 1. 서비스 인터페이스
interface MemoService {
  readonly create: (
    userId: string,
    timestamp: number,
    content: string
  ) => Effect.Effect<Memo, Error>

  readonly findByTimestamp: (
    userId: string,
    timestamp: number
  ) => Effect.Effect<Memo[], Error>
}

// 2. Context Tag
const MemoService = Context.GenericTag<MemoService>("@services/MemoService")

// 3. Mock 구현 (테스트용)
const MemoServiceMock = Layer.succeed(MemoService, {
  create: (userId, timestamp, content) =>
    Effect.succeed({
      id: "mock-id",
      userId,
      timestamp,
      content,
      createdAt: new Date(),
      updatedAt: new Date()
    }),

  findByTimestamp: (userId, timestamp) =>
    Effect.succeed([])
})

// 4. 서비스 사용
const program = Effect.gen(function* () {
  const memoService = yield* MemoService

  const memo = yield* memoService.create(
    "user-1",
    Date.now(),
    "테스트 메모"
  )

  console.log("Created memo:", memo)

  const memos = yield* memoService.findByTimestamp(
    "user-1",
    Date.now()
  )

  return memos
})

// 실행
const memos = await Effect.runPromise(
  program.pipe(Effect.provide(MemoServiceMock))
)
```

### 3.7 여러 서비스 조합

```typescript
import { Layer } from "effect"

// SupabaseService가 필요한 MemoService
const MemoServiceLive = Layer.effect(
  MemoService,
  Effect.gen(function* () {
    // SupabaseService 의존성 요청
    const { client } = yield* SupabaseService

    return {
      create: (userId, timestamp, content) =>
        Effect.tryPromise({
          try: async () => {
            const { data, error } = await client
              .from("memos")
              .insert({ user_id: userId, timestamp, content })
              .select()
              .single()

            if (error) throw error
            return data
          },
          catch: (error) => new Error(`Create failed: ${error}`)
        }),

      findByTimestamp: (userId, timestamp) =>
        Effect.tryPromise({
          try: async () => {
            const { data, error } = await client
              .from("memos")
              .select("*")
              .eq("user_id", userId)
              .eq("timestamp", timestamp)

            if (error) throw error
            return data
          },
          catch: (error) => new Error(`Find failed: ${error}`)
        })
    }
  })
)

// Layer 조합
const AppLayer = MemoServiceLive.pipe(
  Layer.provide(SupabaseServiceLive)
)

// 사용
const result = await Effect.runPromise(
  program.pipe(Effect.provide(AppLayer))
)
```

### 3.8 다음 단계로 넘어가는 기준

- ✅ Context.GenericTag로 서비스 정의 가능
- ✅ Layer.succeed 또는 Layer.effect로 구현 제공 가능
- ✅ Effect.gen에서 yield*로 서비스 사용 가능
- ✅ Layer.provide로 의존성 조합 이해

---

## Phase 4: React 통합 (2-3일)

### 4.1 Effect를 React Hook으로

Effect를 React 컴포넌트에서 사용하려면 Hook으로 변환해야 합니다.

```typescript
import { Effect, Runtime, Exit } from "effect"
import { useState, useEffect } from "react"

interface UseEffectState<A, E> {
  data: A | null
  error: E | null
  loading: boolean
}

function useRunEffect<A, E>(
  effect: Effect.Effect<A, E, never>,
  deps: React.DependencyList = []
): UseEffectState<A, E> {
  const [state, setState] = useState<UseEffectState<A, E>>({
    data: null,
    error: null,
    loading: true
  })

  useEffect(() => {
    setState({ data: null, error: null, loading: true })

    const fiber = Runtime.runFork(Runtime.defaultRuntime)(effect)

    fiber.await.then((exit) => {
      if (Exit.isSuccess(exit)) {
        setState({ data: exit.value, error: null, loading: false })
      } else if (Exit.isFailure(exit)) {
        setState({ data: null, error: exit.cause as E, loading: false })
      }
    })

    return () => {
      Runtime.Fiber.interrupt(fiber)
    }
  }, deps)

  return state
}
```

### 4.2 데이터 조회 Hook

```typescript
import { Effect } from "effect"
import { MemoService } from "@/effect/services/MemoService"
import { AppLayer } from "@/effect/layers/AppLayer"

function useMemos(userId: string, timestamp: number) {
  const memosEffect = Effect.gen(function* () {
    const memoService = yield* MemoService
    return yield* memoService.findByTimestamp(userId, timestamp)
  }).pipe(Effect.provide(AppLayer))

  return useRunEffect(memosEffect, [userId, timestamp])
}

// 컴포넌트에서 사용
function MemoList({ userId, timestamp }) {
  const { data: memos, loading, error } = useMemos(userId, timestamp)

  if (loading) return <div>Loading...</div>
  if (error) return <div>Error: {error.message}</div>
  if (!memos) return null

  return (
    <ul>
      {memos.map(memo => (
        <li key={memo.id}>{memo.content}</li>
      ))}
    </ul>
  )
}
```

### 4.3 데이터 변경 Hook (Mutation)

```typescript
import { Effect } from "effect"
import { useCallback, useState } from "react"
import { MemoService } from "@/effect/services/MemoService"
import { AppLayer } from "@/effect/layers/AppLayer"

function useCreateMemo() {
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<Error | null>(null)

  const createMemo = useCallback(
    async (userId: string, timestamp: number, content: string) => {
      setLoading(true)
      setError(null)

      const effect = Effect.gen(function* () {
        const memoService = yield* MemoService
        return yield* memoService.create(userId, timestamp, content)
      }).pipe(Effect.provide(AppLayer))

      try {
        const memo = await Effect.runPromise(effect)
        setLoading(false)
        return memo
      } catch (err) {
        setError(err as Error)
        setLoading(false)
        return null
      }
    },
    []
  )

  return { createMemo, loading, error }
}

// 컴포넌트에서 사용
function MemoForm({ userId, timestamp }) {
  const [content, setContent] = useState("")
  const { createMemo, loading } = useCreateMemo()

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()

    const memo = await createMemo(userId, timestamp, content)

    if (memo) {
      setContent("") // 성공 시 폼 초기화
      alert("메모가 생성되었습니다!")
    }
  }

  return (
    <form onSubmit={handleSubmit}>
      <textarea
        value={content}
        onChange={(e) => setContent(e.target.value)}
        disabled={loading}
      />
      <button type="submit" disabled={loading}>
        {loading ? "생성 중..." : "메모 추가"}
      </button>
    </form>
  )
}
```

### 4.4 실습 예제: 완전한 메모 컴포넌트

#### 연습 문제 5: MemoPanel 구현

```typescript
"use client"

import { useState } from "react"
import { useMemos } from "../hooks/useMemos"
import { useCreateMemo } from "../hooks/useCreateMemo"
import { useDeleteMemo } from "../hooks/useDeleteMemo"

interface MemoPanelProps {
  userId: string
  timestamp: number
  onClose: () => void
}

export function MemoPanel({ userId, timestamp, onClose }: MemoPanelProps) {
  const { data: memos, loading, error } = useMemos(userId, timestamp)
  const { createMemo, loading: creating } = useCreateMemo()
  const { deleteMemo, loading: deleting } = useDeleteMemo()

  const [content, setContent] = useState("")

  const handleCreate = async () => {
    if (!content.trim()) return

    const memo = await createMemo(userId, timestamp, content.trim())

    if (memo) {
      setContent("")
      // useMemos가 자동으로 데이터 새로고침
    }
  }

  const handleDelete = async (id: string) => {
    if (!confirm("정말 삭제하시겠습니까?")) return
    await deleteMemo(id)
  }

  return (
    <div className="memo-panel">
      <header>
        <h3>메모</h3>
        <button onClick={onClose}>닫기</button>
      </header>

      <div className="timestamp">
        {new Date(timestamp).toLocaleString("ko-KR")}
      </div>

      <div className="memo-form">
        <textarea
          value={content}
          onChange={(e) => setContent(e.target.value)}
          placeholder="메모를 입력하세요..."
          maxLength={1000}
        />
        <div className="form-footer">
          <span>{content.length} / 1000</span>
          <button onClick={handleCreate} disabled={creating || !content.trim()}>
            추가
          </button>
        </div>
      </div>

      <div className="memo-list">
        {loading && <div>로딩 중...</div>}
        {error && <div>에러: {error.message}</div>}
        {memos && memos.length === 0 && (
          <div className="empty">메모가 없습니다</div>
        )}
        {memos?.map((memo) => (
          <div key={memo.id} className="memo-item">
            <p>{memo.content}</p>
            <div className="memo-footer">
              <small>{new Date(memo.createdAt).toLocaleTimeString()}</small>
              <button
                onClick={() => handleDelete(memo.id)}
                disabled={deleting}
              >
                삭제
              </button>
            </div>
          </div>
        ))}
      </div>
    </div>
  )
}
```

### 4.5 다음 단계로 넘어가는 기준

- ✅ useRunEffect로 Effect를 React 상태로 변환 가능
- ✅ 조회 Hook (useMemos) 구현 가능
- ✅ 변경 Hook (useCreateMemo) 구현 가능
- ✅ 컴포넌트에서 Effect 기반 Hook 사용 가능

---

## 학습 완료 체크리스트

모든 Phase를 완료했다면 다음을 확인하세요:

### Phase 1: Effect 기초

- [ ] Effect<Success, Error, Requirements> 타입 이해
- [ ] Effect.succeed, Effect.fail, Effect.sync 사용 가능
- [ ] Effect.tryPromise로 Promise 변환 가능
- [ ] pipe를 사용한 체이닝
- [ ] catchAll, orElse, match로 에러 핸들링

### Phase 2: Schema

- [ ] Schema.Struct로 객체 타입 정의
- [ ] pipe를 사용한 검증 규칙 추가
- [ ] Schema.decodeUnknown으로 데이터 파싱
- [ ] 커스텀 검증 규칙 작성 가능

### Phase 3: Service와 Layer

- [ ] Context.GenericTag로 서비스 정의
- [ ] Layer.succeed/effect로 구현 제공
- [ ] Effect.gen에서 yield*로 서비스 사용
- [ ] Layer.provide로 의존성 조합

### Phase 4: React 통합

- [ ] useRunEffect 구현 이해
- [ ] Effect 기반 조회 Hook 작성
- [ ] Effect 기반 변경 Hook 작성
- [ ] React 컴포넌트에서 Hook 사용

---

## 다음 단계

학습을 완료했다면, 이제 실제 구현을 시작할 준비가 되었습니다!

구현은 다음 단계로 진행됩니다:

1. **Phase 1: 기초 설정** (1-2일)
   - Effect-TS 패키지 설치
   - 디렉토리 구조 생성
   - Supabase 마이그레이션 실행

2. **Phase 2: Effect Service 구현** (2-3일)
   - MemoService 인터페이스 정의
   - Layer 구현
   - 에러 타입 정의

3. **Phase 3: React 통합** (2-3일)
   - 커스텀 Hook 구현
   - UI 컴포넌트 구현
   - 차트 페이지 통합

4. **Phase 4: 테스트 및 최적화** (1-2일)
   - 단위 테스트 작성
   - 에러 핸들링 개선
   - 성능 최적화

---

## 학습 리소스

### 공식 문서

- **Effect-TS Introduction**: https://effect.website/docs/introduction
- **Effect Schema**: https://effect.website/docs/schema/introduction
- **Context Management with Layers**: https://effect.website/docs/guides/context-management/layers
- **Error Handling**: https://effect.website/docs/guides/error-management

### 커뮤니티

- **Effect Discord**: https://discord.gg/effect-ts
- **GitHub Repository**: https://github.com/Effect-TS/effect

### 추가 자료

- **From React to Effect** (공식 블로그): https://effect.website/blog/from-react-to-effect
- **Effect Examples**: https://github.com/Effect-TS/effect/tree/main/packages/examples

---

## 출처

본 학습 가이드는 다음 공식 자료를 기반으로 작성되었습니다:

- [Official Documentation] Effect-TS Introduction: https://effect.website/docs/introduction
- [Official Documentation] Effect Schema: https://effect.website/docs/schema/introduction
- [Official Documentation] Context Management with Layers: https://effect.website/docs/guides/context-management/layers
- [Official Documentation] Error Handling: https://effect.website/docs/guides/error-management
- [Official Blog] From React to Effect: https://effect.website/blog/from-react-to-effect
- [GitHub Repository] Effect-TS Official Examples: https://github.com/Effect-TS/effect/tree/main/packages/examples
- [Community] Effect Discord: https://discord.gg/effect-ts
