# 출석 알림 시스템 - 테스트 플로우 분석

## 전체 아키텍처

```
┌─────────────────┐     postMessage      ┌──────────────┐     message event     ┌────────────────────┐
│ DevPushSimulator │ ──────────────────► │ Service Worker │ ──────────────────► │ useWebPush listener │
│                  │                     │               │                     │                      │
│ · simulateLatest │                     │ SIMULATE_PUSH │                     │ handlePushMessage()  │
│ · simulatePast   │                     │ → postMessage │                     │                      │
└────────┬─────────┘                     └───────────────┘                     └──────────┬───────────┘
         │                                                                                │
         │ addSimulatedAlarm()                                                            │
         ▼                                                                                ▼
┌─────────────────┐     fetchAlarms()    ┌──────────────────────────────────────────────────┐
│ Mock 저장소      │ ◄────────────────── │              AttendanceContext                    │
│ (메모리 배열)    │                     │                                                  │
│                  │ ────────────────── ►│ State:                                           │
│ · simulatedAlarms│    mock 응답 반환   │  alarms, sortedAlarmIds, isDrawerOpen,           │
│ · generateAlarm  │                     │  anchorAlarmId, highlightedIds,                  │
└──────────────────┘                     │  newIndicatorCount, isAtTop, latestPushPayload,  │
                                         │  toastEnabled, hasNewBadge                       │
                                         └──────┬──────────┬──────────┬────────────────────┘
                                                 │          │          │
                                    ┌────────────┘          │          └────────────┐
                                    ▼                       ▼                       ▼
                           ┌────────────────┐  ┌─────────────────────┐  ┌──────────────────┐
                           │ ToastContainer │  │   AttendanceDrawer  │  │ DrawerButton     │
                           │                │  │                     │  │ (GNB)            │
                           │ · Toast 렌더   │  │ · Header            │  │                  │
                           │ · 5초 자동닫힘 │  │ · Settings          │  │ · hasNewBadge    │
                           │ · 상세보기     │  │ · AlarmList         │  │ · toggleDrawer   │
                           └────────────────┘  │   · NewIndicator    │  └──────────────────┘
                                               │   · AlarmCard       │
                                               │   · DateDivider     │
                                               │ · Footer            │
                                               └─────────────────────┘
```

---

## 핵심 상태 변수

| 변수 | 타입 | 설명 | 변경 시점 |
|------|------|------|-----------|
| `isDrawerOpen` | `boolean` | 드로어 열림 여부 | GNB 클릭, 토스트 상세보기, 닫기 버튼 |
| `isAtTop` | `boolean` | 스크롤 최상단 여부 (≤10px) | viewport scroll 이벤트 |
| `anchorAlarmId` | `string \| null` | 스크롤 이동 대상 알림 ID | `openDrawer(id)` 호출 시 |
| `highlightedIds` | `Set<string>` | 하이라이트 대상 알림 ID | 새 알림 도착, 3초 후 자동 해제 |
| `newIndicatorCount` | `number` | 미확인 새 알림 건수 | 스크롤 중간 + 새 푸시 도착 시 |
| `latestPushPayload` | `Payload \| null` | 토스트 생성용 푸시 데이터 | 드로어 닫힌 상태에서 푸시 수신 |
| `toastEnabled` | `boolean` | 토스트 표시 여부 (localStorage) | Settings 토글 |
| `hasNewBadge` | `boolean` | GNB 빨간 점 표시 여부 | 푸시 수신 / 드로어 열기 |
| `alarms` | `Map<id, Alarm>` | 알림 데이터 저장소 | fetchAlarms 응답, 드로어 닫기 시 초기화 |
| `sortedAlarmIds` | `string[]` | attendedAt 기준 내림차순 정렬 | alarms 변경 시 재계산 |

---

## 케이스별 플로우

### 케이스 1: 드로어 닫힘 → 푸시 도착 → 토스트 표시

> 가장 기본적인 플로우. 드로어가 닫혀있을 때 푸시가 도착하면 토스트가 뜬다.

```
사용자: [최신 알림 테스트] 클릭
         │
         ▼
┌─ simulatePush(new Date().toISOString()) ─────────────────────────────────┐
│                                                                          │
│  1. addSimulatedAlarm(payload)  →  mock 저장소에 알림 데이터 추가        │
│                                                                          │
│  2. SW.postMessage({ type: 'SIMULATE_PUSH', payload })                  │
│     → Service Worker가 수신                                              │
│     → clients.matchAll()로 메인 페이지 찾음                              │
│     → client.postMessage(payload) 전달                                   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─ handlePushMessage(payload) ─────────────────────────────────────────────┐
│                                                                          │
│  조건 확인: isDrawerOpenRef.current === false  ✅                        │
│                                                                          │
│  → badge.setHasNewBadge(true)         GNB 빨간 점 표시                   │
│  → setLatestPushPayload(payload)      토스트 생성 트리거                 │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─ AttendanceToastContainer useEffect ─────────────────────────────────────┐
│                                                                          │
│  의존성: [latestPushPayload, toastEnabled, isDrawerOpen]                 │
│                                                                          │
│  조건 확인:                                                               │
│    latestPushPayload !== null  ✅                                        │
│    toastEnabled === true       ✅                                        │
│    isDrawerOpen === false      ✅                                        │
│                                                                          │
│  → ToastItem 생성 (payload → toast 데이터 변환)                          │
│  → setToasts(prev => [newToast, ...prev].slice(0, 5))                   │
│  → clearLatestPush()                                                     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─ AttendanceToast 렌더 ───────────────────────────────────────────────────┐
│                                                                          │
│  Mount → requestAnimationFrame → setIsVisible(true)                      │
│  → CSS transition: translate-x-full → translate-x-0 (slide-in)          │
│                                                                          │
│  setTimeout(5000ms)                                                      │
│  → setIsVisible(false) → slide-out 애니메이션                            │
│  → setTimeout(300ms) → onClose() → toasts 배열에서 제거                  │
│                                                                          │
│  사용자 액션:                                                             │
│  · [X] 클릭 → 즉시 slide-out → onClose()                                │
│  · [출석 알림 상세 보기] 클릭 → 케이스 2로 이동                          │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**최종 상태:**
- GNB 빨간 점: ✅ 표시
- 토스트: ✅ 화면 우측에 5초간 표시
- 드로어: ❌ 닫힘 유지

---

### 케이스 2: 토스트 [상세 보기] 클릭 → 드로어 열림 + 해당 알림으로 이동

> 토스트에서 "출석 알림 상세 보기"를 클릭하면 드로어가 열리고 해당 알림 위치로 스크롤된다.

```
사용자: 토스트의 [출석 알림 상세 보기] 클릭
         │
         ▼
┌─ handleDetailClick(alarmId) in ToastContainer ───────────────────────────┐
│                                                                          │
│  1. openDrawer(alarmId)  호출                                            │
│  2. setToasts([])        모든 토스트 즉시 제거                            │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─ openDrawer(alarmId) in AttendanceContext ────────────────────────────────┐
│                                                                          │
│  setIsDrawerOpen(true)                                                   │
│  setIsSettingsOpen(false)                                                │
│  badge.setHasNewBadge(false)              GNB 빨간 점 제거               │
│  setAnchorAlarmId(alarmId)                스크롤 대상 지정               │
│  setIsLoading(true)                                                      │
│                                                                          │
│  fetchAlarms({                                                           │
│    centerId,                                                             │
│    anchorId: alarmId,                                                    │
│    direction: 'around',     ← 해당 알림 기준 앞뒤 약 10개씩             │
│    size: 20                                                              │
│  })                                                                      │
│         │                                                                │
│         ▼                                                                │
│  응답 수신:                                                               │
│  → mergeAlarms(new Map(), response.items)                                │
│  → getSortedAlarmIds(merged)                                             │
│  → setHighlightedIds(new Set([alarmId]))  해당 알림 하이라이트           │
│  → setIsLoading(false)                                                   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─ AlarmList useEffect: anchorAlarmId 변경 감지 ───────────────────────────┐
│                                                                          │
│  의존성: [anchorAlarmId, sortedAlarmIds]                                 │
│                                                                          │
│  const isFirst = sortedAlarmIds[0] === anchorAlarmId                     │
│  const el = document.querySelector(`[data-alarm-id="${anchorAlarmId}"]`) │
│                                                                          │
│  el.scrollIntoView({                                                     │
│    behavior: 'smooth',                                                   │
│    block: isFirst ? 'start' : 'center'                                   │
│  })                ▲               ▲                                     │
│                    │               │                                     │
│          최신 알림이면         과거 알림이면                               │
│          리스트 최상단에       화면 중앙에                                 │
│          위치                 위치                                        │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─ 하이라이트 자동 해제 useEffect ─────────────────────────────────────────┐
│                                                                          │
│  의존성: [highlightedIds]                                                │
│                                                                          │
│  highlightedIds.size > 0 이면:                                           │
│  → setTimeout(3000ms) → setHighlightedIds(new Set())                    │
│  → AlarmCard 배경색 복원 (bg-[--main-color-varients-50] → 투명)         │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**최종 상태:**
- GNB 빨간 점: ❌ 제거됨
- 토스트: ❌ 모두 제거됨
- 드로어: ✅ 열림
- 스크롤: 해당 알림 위치 (최신이면 상단, 과거면 중앙)
- 하이라이트: ✅ 3초간 표시 후 자동 해제

---

### 케이스 3: 최신 알림이 상단에 보이는 상태 → 새 푸시 도착

> 드로어가 열려있고 스크롤이 최상단일 때 새 알림이 오면 자동으로 하이라이트된다.

```
[현재 상태]
  드로어: 열림
  스크롤: 최상단 (isAtTop = true)
  최신 알림이 리스트 맨 위에 위치

사용자: [최신 알림 테스트] 클릭
         │
         ▼
┌─ simulatePush → SW → handlePushMessage(payload) ────────────────────────┐
│                                                                          │
│  조건 확인: isDrawerOpenRef.current === true                             │
│                                                                          │
│  latestId = sortedAlarmIdsRef.current[0]   현재 최신 알림 ID             │
│                                                                          │
│  fetchAlarms({                                                           │
│    centerId,                                                             │
│    anchorId: latestId,                                                   │
│    direction: 'after',      ← 현재 최신보다 새로운 알림 조회             │
│    size: 20                                                              │
│  })                                                                      │
│         │                                                                │
│         ▼                                                                │
│  응답 수신:                                                               │
│  → mergeAlarms(기존 alarms, response.items)                              │
│  → newIds = 기존 Map에 없던 ID들 필터링                                  │
│                                                                          │
│  ┌─ 분기: isAtTopRef.current ────────────────────────────────────────┐   │
│  │                                                                    │   │
│  │  === true (최상단)                                                 │   │
│  │  → setHighlightedIds(prev => prev ∪ newIds)                       │   │
│  │  → 새 알림이 리스트 최상단에 추가 + 하이라이트                     │   │
│  │  → 3초 후 하이라이트 자동 해제                                     │   │
│  │                                                                    │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**최종 상태:**
- 새 알림: 리스트 최상단에 추가됨
- 하이라이트: ✅ 3초간 표시 후 해제
- 인디케이터: ❌ 표시 안 됨 (이미 상단이므로)
- 토스트: ❌ 드로어 열려있으므로 미표시

---

### 케이스 4: 과거 알림 토스트 클릭 → 특정 알림으로 포커스 이동

> 과거 시간의 알림이 토스트로 뜨고, 상세보기를 클릭하면 리스트 중간에 위치한다.

```
사용자: [과거 알림 테스트] 클릭
         │
         ▼
┌─ simulatePast() ─────────────────────────────────────────────────────────┐
│                                                                          │
│  const past = new Date()                                                 │
│  past.setHours(past.getHours() - random(3~8))                           │
│  simulatePush(past.toISOString())                                        │
│                                                                          │
│  → attendedAt = 현재 시간 - 3~8시간 (과거 시간)                          │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
         │
         ▼
(케이스 1과 동일: 토스트 표시)
         │
         ▼
사용자: 토스트의 [출석 알림 상세 보기] 클릭
         │
         ▼
┌─ openDrawer(alarmId) ────────────────────────────────────────────────────┐
│                                                                          │
│  fetchAlarms({ direction: 'around', anchorId: alarmId })                │
│  → 해당 알림 기준 앞뒤 약 10개씩 로드                                    │
│                                                                          │
│  setHighlightedIds(new Set([alarmId]))                                   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─ AlarmList 스크롤 처리 ──────────────────────────────────────────────────┐
│                                                                          │
│  isFirst = sortedAlarmIds[0] === anchorAlarmId                           │
│         = false  (과거 알림이므로 최신이 아님)                            │
│                                                                          │
│  el.scrollIntoView({                                                     │
│    behavior: 'smooth',                                                   │
│    block: 'center'     ← 화면 중앙에 위치                                │
│  })                                                                      │
│                                                                          │
│  스크롤 결과: viewport.scrollTop > 10                                    │
│  → setIsAtTop(false)                                                     │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**최종 상태:**
- 드로어: ✅ 열림
- 스크롤: 화면 중앙에 해당 알림 위치
- 하이라이트: ✅ 해당 알림 3초간 표시
- isAtTop: `false` (스크롤이 중간에 위치)

---

### 케이스 5: 스크롤 중간 → 새 푸시 → 인디케이터 표시 → 클릭하여 상단 이동

> 케이스 4 이후 상태에서 새 알림이 도착하면 인디케이터가 표시되고, 클릭 시 상단으로 이동한다.

```
[현재 상태 (케이스 4 이후)]
  드로어: 열림
  스크롤: 중간 (isAtTop = false)
  과거 알림에 포커스 중

사용자: [최신 알림 테스트] 클릭
         │
         ▼
┌─ handlePushMessage(payload) ─────────────────────────────────────────────┐
│                                                                          │
│  조건: isDrawerOpenRef === true                                          │
│                                                                          │
│  fetchAlarms({ direction: 'after' })                                    │
│  → mergeAlarms → newIds 추출                                             │
│                                                                          │
│  ┌─ 분기: isAtTopRef.current ────────────────────────────────────────┐   │
│  │                                                                    │   │
│  │  === false (스크롤 중간)                                           │   │
│  │  → setNewIndicatorCount(prev + newIds.length)                     │   │
│  │  → 인디케이터 표시 (하이라이트 아님!)                              │   │
│  │                                                                    │   │
│  └────────────────────────────────────────────────────────────────────┘   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌─ AttendanceNewIndicator 렌더 ────────────────────────────────────────────┐
│                                                                          │
│  조건: newIndicatorCount > 0 && !isAtTop                                │
│                                                                          │
│  ┌──────────────────────────────────────┐                                │
│  │  "새 출석 알림 N건 보기"  ▲          │  ← 리스트 상단 고정 위치       │
│  └──────────────────────────────────────┘                                │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
         │
         ▼
사용자: 인디케이터 클릭
         │
         ▼
┌─ handleIndicatorClick() in AlarmList ────────────────────────────────────┐
│                                                                          │
│  1. scrollToTopAndHighlight()                                            │
│     → setNewIndicatorCount(0)          인디케이터 숨김                    │
│                                                                          │
│  2. viewportRef.current.scrollTo({                                       │
│       top: 0,                                                            │
│       behavior: 'smooth'                                                 │
│     })                                                                   │
│         │                                                                │
│         ▼                                                                │
│  [viewport scroll 이벤트 발생]                                           │
│  → viewport.scrollTop ≤ 10                                              │
│  → setIsAtTop(true)                                                      │
│                                                                          │
│  새 알림은 이미 DOM에 존재 (리스트 최상단)                                │
│  → 스크롤이 올라가면서 자연스럽게 보임                                    │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**최종 상태:**
- 인디케이터: ❌ 숨김 (count = 0)
- 스크롤: 최상단
- isAtTop: `true`
- 새 알림: 리스트 최상단에 표시됨

---

## 스크롤 감지 메커니즘

```
┌─ AlarmList 컴포넌트 ─────────────────────────────────────────────────────┐
│                                                                          │
│  ┌─ ScrollArea (@aptplayinc/ui) ──────────────────────────────────────┐  │
│  │                                                                     │  │
│  │  ┌─ [data-as-scroll-area-viewport] ← 실제 스크롤 컨테이너 ──────┐ │  │
│  │  │                                                                │ │  │
│  │  │  ┌─ div ref={scrollRef} ────────────────────────────────────┐  │ │  │
│  │  │  │                                                          │  │ │  │
│  │  │  │  · AttendanceNewIndicator (absolute 위치)                │  │ │  │
│  │  │  │  · DateDivider ("오늘")                                  │  │ │  │
│  │  │  │  · AlarmCard (data-alarm-id="...")                       │  │ │  │
│  │  │  │  · AlarmCard                                             │  │ │  │
│  │  │  │  · ...                                                   │  │ │  │
│  │  │  │  · DateDivider ("어제")                                  │  │ │  │
│  │  │  │  · AlarmCard                                             │  │ │  │
│  │  │  │  · ...                                                   │  │ │  │
│  │  │  │  · <div ref={bottomSentinelRef} /> (IntersectionObserver)│  │ │  │
│  │  │  │                                                          │  │ │  │
│  │  │  └──────────────────────────────────────────────────────────┘  │ │  │
│  │  │                                                                │ │  │
│  │  │  scroll 이벤트 → setIsAtTop(scrollTop ≤ 10)                   │ │  │
│  │  │                                                                │ │  │
│  │  └────────────────────────────────────────────────────────────────┘ │  │
│  │                                                                     │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  viewportRef: [data-as-scroll-area-viewport] 요소 직접 참조              │
│  → useEffect에서 찾아서 addEventListener('scroll') 등록                  │
│  → handleIndicatorClick에서 viewportRef.scrollTo() 사용                  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**주의:** `ScrollArea` 내부의 div에 `onScroll`을 걸면 동작하지 않음. 반드시 `[data-as-scroll-area-viewport]` 요소에 직접 리스너를 등록해야 함.

---

## 데이터 소스 비교

### 테스트 환경

```
DevPushSimulator
  │
  ├── addSimulatedAlarm(data)     → 메모리 배열에 알림 추가
  │                                  fetchAlarms() 호출 시 mock 응답에 포함
  │
  └── SW.postMessage(payload)     → Service Worker → 메인 페이지
                                     handlePushMessage() → 토스트/인디케이터
```

- **푸시 데이터**와 **알림 리스트 데이터**는 독립적
- `addSimulatedAlarm()`이 없으면 토스트는 뜨지만 드로어 리스트에 해당 알림이 없음
- 같은 `alarmId`로 양쪽에 모두 등록해야 정상 동작

### 프로덕션 환경

```
서버
  │
  ├── Web Push API → 브라우저 Push → Service Worker → handlePushMessage()
  │
  └── REST API (fetchAlarms) → 알림 리스트 데이터
```

- 서버가 Push와 API 데이터를 동일하게 관리
- `addSimulatedAlarm()` 불필요 (서버 DB에서 조회)

---

## fetchAlarms 방향(direction) 정리

| direction | 용도 | 호출 시점 |
|-----------|------|-----------|
| 없음 (기본 `after`) | 최신 알림 20개 조회 | GNB 버튼 → `openDrawer()` |
| `'around'` | anchor 기준 앞뒤 ~10개씩 | 토스트 클릭 → `openDrawer(alarmId)` |
| `'after'` | anchor보다 새로운 알림 조회 | 드로어 열린 상태에서 푸시 수신 |
| `'before'` | anchor보다 오래된 알림 조회 | 하단 스크롤 → `loadMoreBefore()` |

---

## 조건부 동작 요약

| 상황 | isDrawerOpen | isAtTop | 결과 |
|------|:---:|:---:|------|
| 푸시 도착 | `false` | - | badge ✅ + toast ✅ (toastEnabled일 때) |
| 푸시 도착 | `true` | `true` | 리스트 상단에 추가 + 하이라이트 |
| 푸시 도착 | `true` | `false` | 인디케이터 카운트 증가 |
| 인디케이터 클릭 | `true` | `false` → `true` | 상단으로 스크롤 + 인디케이터 숨김 |
| 토스트 상세보기 | `false` → `true` | - | 드로어 열림 + anchor 스크롤 + 하이라이트 |
| 드로어 닫기 | `true` → `false` | - | 모든 상태 초기화 (alarms, highlights 등) |
