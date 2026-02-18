# React Compiler Style Analysis & Functional Refactoring Report

> **Branch**: `feat/attendance-alarm2` vs `main`
> **Date**: 2026-02-10
> **Method**: React Compiler 파이프라인(AST -> HIR -> SSA -> Effect -> Reactive -> Scope)을 AI가 수동으로 수행

---

## 분석 방법론

React Compiler가 코드를 최적화하는 5단계 분석을 각 파일에 적용:

| 단계 | 설명 | 목적 |
|------|------|------|
| **HIR** | 실행 흐름을 기본 블록으로 분해 | 분기/합류 지점 파악 |
| **SSA** | 변수의 단일 할당 여부 분석 | 재할당/mutation 탐지 |
| **Effect** | Read/Store/Mutate/Freeze 분류 | 캐싱 가능 여부 판단 |
| **Reactive** | props/state/context 의존성 전파 | 불필요한 재계산 탐지 |
| **Scope** | 같은 의존성 공유 연산 그룹화 | 캐시 단위 결정 |

---

## 리팩토링 원칙

1. **순수 함수 추출**: 훅/컴포넌트 내부의 계산 로직을 모듈 수준 순수 함수로 추출
2. **상태 최소화**: 파생 가능한 값은 `useState` 대신 계산으로 도출
3. **불변성 강화**: `let` 제거, `readonly` 타입, `[...arr].sort()` 패턴
4. **Side Effect 격리**: 순수 계산과 부수효과를 명확히 분리
5. **안정적 참조**: `useCallback`, `useMemo`로 불필요한 재생성 방지

---

## 파일별 분석 및 변경사항

### 1. Types Layer

#### `src/types/attendance-alarm.ts`

| 분석 | 결과 |
|------|------|
| HIR | 순수 타입 선언, 실행 흐름 없음 |
| SSA | 모듈 수준 상수만 존재, 재할당 없음 |
| Effect | Read-only (타입은 컴파일 시점에 제거됨) |
| Reactive | 모든 consumer에 전파되는 근본 타입 |
| Scope | 정적 분석 가능, 변경 불필요 |

**변경사항:**
- `AttendanceProduct`를 discriminated union으로 분리 (`PeriodMembershipProduct`, `CountMembershipProduct`, `GroupClassProduct`)
- `AdditionalProduct`를 `LockerProduct` / `EquipmentProduct`로 분리
- `Gender`, `ProductType`, `AdditionalProductType` 타입 별칭 추출
- 모든 interface 속성에 `readonly` 추가
- 상수에 `as const` 적용 (literal type narrowing)

#### `src/types/push.ts`

**변경사항:**
- 미사용 `BasePushPayload` 제거
- `AnyPushPayload` union 타입 추가
- `readonly` 적용

---

### 2. API Layer

#### `src/apis/attendance-alarm/getAttendanceAlarms.ts`

| 분석 | 결과 |
|------|------|
| HIR | 선형: param-build -> fetch -> validate (2개 실패 분기) |
| SSA | `searchParams` 1회 할당 후 `.set()` 3회 mutation |
| Effect | `searchParams.set()` = Mutate, `fetch` = Store, `response.json()` = Read |
| Reactive | 모든 파라미터가 reactive input |
| Scope | param-building과 validation은 독립 scope |

**변경사항:**
- `buildSearchParams()` 순수 함수 추출 (URLSearchParams mutation 격리)
- `validateResponse()` 제네릭 순수 함수 추출 (재사용 가능)
- `readonly` interface 속성

#### `src/apis/push-subscription.ts`

**변경사항:**
- `PUSH_SUBSCRIPTIONS_URL` 상수 추출 (중복 URL 제거)
- `buildAuthHeaders()` 순수 함수 추출 (중복 헤더 구성 제거)
- `assertOk()` 순수 검증 함수 추출

---

### 3. Hooks Layer

#### `src/hooks/attendance-alarm/useAttendanceAlarmsInfiniteQuery.ts`

| 분석 | 결과 |
|------|------|
| HIR | 4개 주요 코드 경로: 메인 쿼리, fetchLatest, fetchAfter, resetWithAnchor |
| SSA | `setQueryData` 콜백 내 중복 제거 로직에 다수 변수 할당 |
| Effect | `getNextPageParam`/`getPreviousPageParam` = Read(순수), `fetchLatestAlarms` = Read+Mutate 혼합 |
| Reactive | 모든 fetch 함수가 `[accessToken, centerId, size, queryClient, queryKey]` 의존 |
| Scope | 페이지네이션 추출, 중복제거, prepend, reset 각각 독립 scope |

**변경사항:**
- `InfiniteQueryData` 인터페이스 정의 (3회 반복 인라인 타입 제거)
- `deriveInitialPageParam()` 순수 함수 추출
- `extractNextPageParam()` / `extractPreviousPageParam()` 순수 함수 추출 (안정적 참조)
- `collectExistingIds()` 순수 Set 구성 함수 추출
- `mergeLatestPage()` 순수 중복제거+prepend 함수 추출 (20+ 라인 콜백 -> 1 함수 호출)
- `prependPage()` / `resetToSinglePage()` 순수 불변 함수 추출
- `queryKey`에 `as const` 적용

#### `src/hooks/useAttendanceAlarms.ts`

**변경사항:**
- `buildSearchParams()`, `buildAlarmsUrl()`, `isAbortError()` 순수 함수 추출
- `abortControllerRef.current?.abort()` 옵셔널 체이닝 적용

#### `src/hooks/useAttendanceBadge.ts`

**변경사항:**
- `isNewAlarm()` 순수 함수 추출 (localStorage 비교 로직)

#### `src/hooks/useScrollAreaVirtualizer.ts`

**변경사항:**
- `findScrollAreaViewport()` 순수 DOM 쿼리 함수 추출
- `estimateSize` 콜백에 `useCallback` 적용 (불필요한 virtualizer 재구성 방지)

---

### 4. Components Layer - Alarm List

#### `src/components/attendance-alarm/attendance-alarm-list/AttendanceAlarmCard.tsx`

| 분석 | 결과 |
|------|------|
| HIR | 단일 선형 경로, 인라인 삼항 다수 |
| SSA | `time`, `genderLabel` 1회 할당 (양호) |
| Effect | 모든 연산 Read (순수 계산), 이벤트 핸들러 Freeze |
| Reactive | `alarm`, `highlighted`, `onClick` props에만 의존 |
| Scope | 전체 컴포넌트가 단일 scope |

**변경사항:**
- `GENDER_LABEL`, `PRODUCT_TYPE_LABEL` 정적 Record 룩업 (삼항 -> 테이블 구동)
- `deriveAdditionalProductLabel()`, `deriveAdditionalProductDetail()` 순수 함수 추출
- `getCardClassName()` 순수 함수 추출
- `ProfileImage`, `MemberInfo`, `UnpaidBadge`, `ProductRow`, `AdditionalProductRow` 서브컴포넌트 분리

#### `src/components/attendance-alarm/attendance-alarm-list/AttendanceAlarmList.tsx`

| 분석 | 결과 |
|------|------|
| HIR | **가장 복잡** - 8+ useEffect, 다수 분기 |
| SSA | 4개 Set 상태 반복 mutation, 3개 ref mutation |
| Effect | 6개 Store, 2개 Mutate, 2개 Read 효과 |
| Reactive | `centerId -> query -> pages -> alarms -> flatItems` 핵심 체인 |
| Scope | flatItems, alarms 메모 + Set 연산 통합 필요 |

**변경사항:**
- `buildFlatItems()` 순수 함수 추출 (mutable `lastDateLabel` closure 제거)
- `addToSet()`, `removeFromSet()`, `hasAnyNotIn()` 제네릭 Set 유틸리티 추출
- `findAlarmIndex()`, `getScrollAlign()` 순수 함수 추출
- `useIntersectionSentinel()` 커스텀 훅 추출 (중복 IntersectionObserver 제거)
- `useScrollAreaViewport()` 커스텀 훅 추출 (DOM 쿼리 격리)

#### `src/components/attendance-alarm/attendance-alarm-list/AttendanceDrawer.tsx`

**변경사항:**
- settings toggle에 `useCallback` + 함수형 업데이터 적용
- prop 인터페이스 `setIsSettingsOpen` -> `onToggleSettings` (액션 기반)

#### `src/components/attendance-alarm/attendance-alarm-list/AttendanceDrawerButton.tsx`

**변경사항:**
- `getButtonClassName()`, `getIconClassName()` 순수 함수 추출
- `handleClick`에 `useCallback` 적용

#### `src/components/attendance-alarm/attendance-alarm-list/AttendanceDrawerFooter.tsx`

**변경사항:**
- `buildAttendanceUrl()` 순수 함수 추출 (URL 계산을 side effect에서 분리)

#### `src/components/attendance-alarm/attendance-alarm-list/AttendanceDrawerHeader.tsx`

**변경사항:**
- `onToggleSettings` 콜백 prop으로 변경 (closure 재생성 제거)
- `formatDateLabel(new Date())` -> `useMemo(() => ..., [])` (매 렌더 Date 생성 방지)
- `DAYS` 모듈 수준 상수 추출

#### `src/components/attendance-alarm/attendance-alarm-list/AttendanceDrawerSettings.tsx`

**변경사항:**
- `getToggleTrackClassName()`, `getToggleThumbClassName()` 순수 함수 추출
- `handleToggle`에 `useCallback` 적용

#### Skeleton, EmptyState, DateDivider, NewIndicator

**변경사항:**
- 불필요한 `'use client'` 지시문 제거 (순수 표현 컴포넌트)
- `DAYS` 배열 모듈 수준 상수화
- `SkeletonPulse` 재사용 서브컴포넌트 추출

---

### 5. Components Layer - Notification

#### `src/components/attendance-alarm/atoms.ts`

| 분석 | 결과 |
|------|------|
| HIR | 8개 atom, push 처리 로직이 컴포넌트에 분산 |
| SSA | `latestPushPayloadAtom`이 다수 consumer에 의해 읽기/쓰기 |
| Effect | `openDrawerActionAtom` = Mutate(3개 atom 원자적 쓰기) |
| Reactive | `isOpenDrawerAtom`, `notiEnabledAtom` = 읽기 전용 projection |
| Scope | push 처리 상태 머신이 컴포넌트에 분산 (문제) |

**변경사항:**
- `processPushPayloadAtom` 추가 (push 처리 상태 머신을 atom 레이어로 이동)
- 네이밍 개선: `internalIsOpenDrawerAtom` -> `drawerOpenAtom`
- 섹션별 구조화: Primitive Atoms, Derived Atoms, Action Atoms

#### `src/components/attendance-alarm/notification/AttendanceToast.tsx`

**변경사항:**
- `useToastLifecycle()` 커스텀 훅 추출 (enter/auto-dismiss/exit 생명주기 캡슐화)
- `getProductLabel()`, `getAdditionalProductLabel()`, `getLockerSuffix()`, `getVisibilityClass()` 순수 함수 추출
- `ProductInfo` 서브컴포넌트 추출
- `AUTO_DISMISS_MS`, `EXIT_ANIMATION_MS` 매직넘버 상수화

#### `src/components/attendance-alarm/notification/AttendanceToastContainer.tsx`

| 분석 | 결과 |
|------|------|
| HIR | useEffect에서 5개 atom 읽고 3방향 분기 |
| SSA | `latestPushPayload` 모든 분기에서 동일 clear 패턴 |
| Effect | 5개 atom 구독, 2개 Store, 조건부 분기 = 불순 효과 |
| Reactive | 5개 reactive 의존성 -> 광범위한 효과 재실행 |
| Scope | Toast 관리와 push 처리가 뒤엉킴 |

**변경사항:**
- **atom 구독 5개 -> 3개로 감소** (processPushPayloadAtom이 분기 로직 담당)
- `createToastId()`, `payloadToToast()`, `addToast()`, `removeToastById()` 순수 함수 추출
- `useEffect` 의존성 배열 5개 -> 2개로 축소
- `useRef`로 stale closure 방지

#### `src/components/attendance-alarm/notification/DevPushSimulator.tsx`

**변경사항:**
- `buildFakePayload()` 순수 데이터 구성 함수 추출
- `payloadToAlarm()` 순수 매퍼 함수 추출
- `createPastDate()` 순수 함수 추출
- 상수 배열에 `as const` 적용

---

### 6. Push & Layout Layer

#### `src/libs/push/PushListener.tsx`

**변경사항:**
- `usePushSubscription()` 커스텀 훅 추출 (push 구독 side effect 격리)
- `useServiceWorkerMessageListener()` 커스텀 훅 추출 (메시지 리스너 격리)
- `canSubscribePush()` 순수 판별 함수 추출

#### `src/libs/push/registry.ts`, Layout 파일들

**변경 없음** - 이미 최적 구조

---

### 7. Utils & Mocks Layer

#### `src/utils/mock-attendance-alarms.ts`

| 분석 | 결과 |
|------|------|
| HIR | `generateMockAlarms`에 4개 분기 (pagination direction) |
| SSA | **심각한 위반**: 7개 `let` 변수, for 루프 카운터, in-place mutation |
| Effect | `allAlarms.sort()` = Mutate(in-place), `allAlarms.push()` = Mutate |
| Reactive | N/A (유틸리티) |
| Scope | 단일 모놀리식 함수를 파이프라인으로 분해 필요 |

**변경사항:**
- `hashString()`: `let hash` + `for` -> `Array.from(str).reduce()` (let 0개)
- `generateAlarm()`: `isSameDay()`, `generateProduct()`, `generateAdditionalProducts()` 순수 헬퍼 추출
- `generateMockAlarms()`: **let 7개 -> 0개**
  - `assignStableIds()` 순수 reduce 함수
  - `computePaginationSlice()` 순수 함수 (early-return 패턴)
- `allAlarms.sort()` -> `[...alarms].sort()` (불변 정렬)
- `allAlarms.push()` -> spread in array literal
- `for` 루프 -> `Array.from({ length }, (_, i) => ...)` 함수형 반복
- **전체 흐름이 함수형 파이프라인**: `generate -> merge -> sort -> assignIds -> paginate -> count`

#### `src/mocks/handlers/attendance-alarm.ts`

**변경사항:**
- `parseAlarmQueryParams()` 순수 파라미터 파싱 함수 추출
- `API_DELAY_MS`에 `as const` 적용

#### `src/mocks/initMsw.ts`

**변경사항:**
- 중첩 `if` -> 순차 guard clause (HIR 평탄화)
- `Promise<void>` 명시적 반환 타입

---

## 종합 메트릭

| 메트릭 | Before | After |
|--------|--------|-------|
| 모듈 수준 순수 함수 | ~5개 | **~45개** |
| 커스텀 훅 추출 | 0개 | **6개** |
| 서브컴포넌트 추출 | 0개 | **7개** |
| `let` 변수 (mock-attendance) | 7개 | **0개** |
| 인라인 타입 반복 | 3회 | **0회** (InfiniteQueryData) |
| 인라인 삼항 (AlarmCard) | 5개 | **0개** (테이블 구동) |
| atom 구독 (ToastContainer) | 5개 | **3개** |
| useEffect 의존성 (ToastContainer) | 5개 | **2개** |
| 불안정한 콜백 참조 | 8+ | **0개** (useCallback 적용) |
| 불필요한 'use client' | 3개 | **0개** |
| 매직넘버 | 5+ | **0개** (상수 추출) |

## 핵심 아키텍처 변화

### Push 처리 상태 머신 이동

```
Before:
  PushListener -> latestPushPayloadAtom -> AttendanceToastContainer useEffect
                                           ├─ read isDrawerOpen
                                           ├─ read notiEnabled
                                           ├─ set hasNewBadge
                                           └─ clear payload (3곳 중복)

After:
  PushListener -> latestPushPayloadAtom -> AttendanceToastContainer useEffect
                                           └─ processPushPayloadAtom (1회 호출)
                                              ├─ read isDrawerOpen (atom 내부)
                                              ├─ read notiEnabled (atom 내부)
                                              ├─ set hasNewBadge (atom 내부)
                                              └─ return payload | null
```

### 함수형 파이프라인 (mock-attendance-alarms)

```
Before (imperative):
  let allAlarms = [...];
  allAlarms.push(...simulatedAlarms);
  allAlarms.sort((a, b) => ...);
  let mockIndex = 0;
  for (const alarm of allAlarms) { alarm.id = ...; mockIndex++; }
  let anchorIndex, items, hasMoreBefore, hasMoreAfter;
  if (direction === 'around') { ... }
  else if (direction === 'before') { ... }

After (functional pipeline):
  const merged = [...generateBatch(count), ...simulatedAlarms];
  const sorted = sortByAttendedAtDesc(merged);
  const withIds = assignStableIds(sorted);
  const slice = computePaginationSlice(withIds, params);
  return { items: slice.items, hasMore: { before: ..., after: ... } };
```

---

## React Compiler 호환성 영향

이 리팩토링으로 React Compiler가 더 효과적으로 최적화할 수 있는 이유:

1. **모듈 수준 순수 함수**: 컴파일러가 "이 함수는 절대 변하지 않는다"고 판단 -> 메모이제이션 skip
2. **안정적 참조 (useCallback)**: 자식 컴포넌트의 reactive scope가 불필요하게 무효화되지 않음
3. **좁은 의존성 배열**: 컴파일러가 더 작은 reactive scope를 생성 -> 정밀한 부분 재계산
4. **불변 데이터 변환**: `[...arr].sort()` 패턴으로 컴파일러가 안전하게 캐싱 가능
5. **서브컴포넌트 분리**: 각각 독립적인 reactive scope로 캐싱 가능
