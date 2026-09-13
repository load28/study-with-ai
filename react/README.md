# React 내부 동작 학습 로드맵

## 학습 목표
React를 "사용법"이 아니라 **구현**으로 이해한다. 렌더링 파이프라인, 훅의 자료구조, 스케줄러와 Lane 모델, Concurrent 기능, 서버 컴포넌트, React Compiler까지 **실제 소스 코드를 읽으며** 체계적으로 파악한다.

## 학습 철학
- **소스 코드가 근거다**: 모든 설명은 `facebook/react` 저장소의 실제 코드로 뒷받침한다. 블로그·추측·"아마 이렇게 동작할 것"은 쓰지 않는다
- **경로와 함수명을 남긴다**: 인용한 코드는 `파일:줄번호` 와 함수명을 함께 표기해 직접 찾아볼 수 있게 한다
- **관찰 가능한 동작과 연결한다**: 코드를 읽은 뒤 "그래서 화면에서 무엇이 달라지는가"로 되돌아온다
- **점진적 심화**: 렌더 파이프라인 → 훅 → 스케줄링 → Concurrent → 서버/컴파일러 순으로 각 단계가 다음의 전제가 된다

## 기준 버전

| 항목 | 값 |
|------|-----|
| 저장소 | `facebook/react` |
| 커밋 | `019019be403c3269e15b8d7ebefb57d30f84086b` (2026-09-11) |
| 브랜치 | `main` |
| `ReactVersion` | `19.3.0` (`ReactVersions.js`) |

줄번호는 이 커밋 기준이다. 다른 커밋에서는 줄번호가 달라지므로, 각 문서는 인용한 **함수명**을 함께 적어 위치를 다시 찾을 수 있게 한다.

```bash
git clone --depth 1 --branch main https://github.com/facebook/react.git
cd react && git checkout 019019be403c3269e15b8d7ebefb57d30f84086b
```

## 읽게 될 주요 패키지

| 패키지 | 역할 |
|--------|------|
| `packages/react` | 공개 API 표면 (`useState` 등은 디스패처로 위임만 한다) |
| `packages/react-reconciler` | Fiber 트리, 렌더·커밋 루프, 훅 구현, Lane 모델 — 학습의 중심 |
| `packages/react-dom` | 호스트 설정(DOM). `createRoot`, DOM 변경·이벤트 시스템 |
| `packages/scheduler` | 타임 슬라이싱과 우선순위 큐 (`MessageChannel` 기반) |
| `packages/react-server` | 서버 컴포넌트 직렬화(Flight) |
| `compiler/` | React Compiler (별도 워크스페이스) |

## 학습 방법
- 각 주차는 독립적인 문서로 제공된다
- **동작 관찰 → 진입점 찾기 → 코드 읽기 → 원리 정리 → 실전 진단** 순서
- 각 문서는 `week-XXX.md` 형식으로 저장된다 (3자리)
- 인용 코드는 원문 그대로 옮기고, 설명은 코드 블록 밖에 둔다

---

## 전체 학습 계획 (24주 과정 = 약 6개월)

> 💡 **핵심 원칙**: 함수 하나의 세부 구현을 외우는 것이 목표가 아니다. **"이 동작은 어느 단계에서 결정되는가"** 를 말할 수 있게 되는 것이 목표다.

---

### Phase 1: 렌더 파이프라인 (1-5주차)
> React가 화면을 그리는 전체 경로를 코드로 따라간다

**1주차: 렌더와 커밋 — React가 화면을 그리는 3단계** ✅
- Trigger: `root.render` / `setState` 가 스케줄로 바뀌는 경로
- Render: `workLoopSync` → `beginWork` → `completeUnitOfWork` 순회
- Commit: before mutation / mutation / layout 3단계와 paint 이후의 passive
- 실전: "화면이 안 바뀐다 / 두 번 그려진다"를 단계별로 진단

**2주차: Fiber 자료구조와 이중 버퍼링**
- `Fiber` 객체의 필드 전수 조사 (`ReactFiber.js`)
- `return` / `child` / `sibling` 로 만드는 트리와 순회
- `alternate` 와 `createWorkInProgress` — current 트리와 WIP 트리
- `root.current` 스위칭이 "원자적 업데이트"인 이유

**3주차: beginWork와 completeWork**
- 태그(`WorkTag`)별 분기 구조
- `didReceiveUpdate` 와 조기 bailout 조건
- `completeWork` 에서 하는 일 (인스턴스 생성, flags 집계)
- `subtreeFlags` 로 커밋 단계 순회를 줄이는 방법

**4주차: 리컨실리에이션과 key**
- `ReactChildFiber.js` 의 `reconcileChildFibers`
- 단일 자식 / 배열 / 이터러블 경로 분리
- `reconcileChildrenArray` 의 두 패스 알고리즘과 `mapRemainingChildren`
- key가 없을 때 발생하는 재생성, 상태 유실의 코드상 원인

**5주차: 커밋 단계 심층 — flags와 이펙트**
- `ReactFiberFlags.js` 의 비트 플래그 체계
- `commitMutationEffects` 순회와 삭제·삽입·업데이트
- `commitHostUpdate` → 호스트 설정의 `commitUpdate`
- layout 이펙트와 passive 이펙트의 실행 순서·타이밍 차이

### Phase 2: 훅의 구현 (6-10주차)
> 훅이 "마법"이 아니라 연결 리스트임을 확인한다

**6주차: 디스패처와 훅 호출 규칙**
- `ReactSharedInternals.H` 와 디스패처 교체
- 마운트/업데이트 디스패처 분리 (`HooksDispatcherOnMount` vs `OnUpdate`)
- 훅 순서가 바뀌면 안 되는 이유 — `workInProgressHook` 커서
- 규칙 위반 감지 코드와 DEV 경고

**7주차: useState와 useReducer의 업데이트 큐**
- `Hook` / `UpdateQueue` / `Update` 구조체
- 순환 연결 리스트로 쌓이는 업데이트
- `updateReducerImpl` 의 재생(replay) 로직과 Lane 건너뛰기
- eager state 계산과 bailout

**8주차: useEffect의 자료구조와 실행 시점**
- `Effect` 객체와 `HookFlags` (`HookPassive`, `HookLayout`, `HookHasEffect`)
- deps 비교 (`areHookInputsEqual`)
- unmount → mount 순서 보장
- `useLayoutEffect` / `useInsertionEffect` 와의 차이

**9주차: 메모이제이션 훅과 bailout 체계**
- `useMemo` / `useCallback` 의 실제 구현 (놀랄 만큼 짧다)
- `React.memo` 와 `didReceiveUpdate` 의 상호작용
- `bailoutOnAlreadyFinishedWork` 와 `childLanes`
- "메모가 안 먹는" 상황의 코드상 원인

**10주차: ref, context, 외부 스토어**
- `useRef` 와 커밋 단계의 ref attach/detach
- `useContext` 와 컨텍스트 전파 (`propagateContextChange`)
- `useSyncExternalStore` 와 tearing 방지 (`isRenderConsistentWithExternalStores`)

### Phase 3: 스케줄링 (11-15주차)
> 언제, 어떤 우선순위로 렌더가 실행되는지 결정하는 층

**11주차: Lane 모델**
- `ReactFiberLane.js` 의 비트마스크 설계
- Lane과 Lanes, `mergeLanes` / `includesSomeLane`
- `requestUpdateLane` — 업데이트의 우선순위는 어디서 정해지는가
- entanglement의 의미

**12주차: 루트 스케줄러와 배칭**
- `ensureRootIsScheduled` → 마이크로태스크 → `processRootScheduleInMicrotask`
- `scheduleTaskForRootDuringMicrotask` 의 우선순위 결정
- 자동 배칭(automatic batching)의 실제 구현 위치
- `flushSync` 가 하는 일

**13주차: Scheduler 패키지**
- `MessageChannel` 기반 태스크 루프
- `shouldYieldToHost` 와 프레임 예산
- 우선순위별 타임아웃과 만료(expiration)
- `requestPaint` 의 역할

**14주차: 이벤트 시스템과 우선순위**
- `react-dom` 의 이벤트 위임과 루트 리스너
- `DiscreteEventPriority` / `ContinuousEventPriority` / `DefaultEventPriority`
- 이벤트 우선순위가 Lane으로 번역되는 지점

**15주차: 중단과 재시작**
- `prepareFreshStack` 과 WIP 트리 폐기
- `workLoopConcurrent` 의 양보 지점
- 높은 우선순위 업데이트가 진행 중 렌더를 가로채는 경로
- 렌더 단계가 순수해야 하는 구조적 이유

### Phase 4: Concurrent 기능 (16-20주차)
> Phase 1-3의 기반 위에 올라간 사용자 기능들

**16주차: Suspense의 구현**
- thenable throw와 `ReactFiberThrow.js`
- `attachPingListener` 와 재시도 Lane
- fallback 커밋과 `SuspenseComponent` 상태
- `use()` 훅과 `ReactFiberThenable.js`

**17주차: Transition**
- `startTransition` / `useTransition`
- `requestTransitionLane` 과 transition Lane 배정
- `useDeferredValue` 와 `requestDeferredLane`
- 진행 중 transition이 버려지는 조건

**18주차: 서버 렌더링과 하이드레이션**
- `react-dom/server` 의 스트리밍 구조
- `ReactFiberHydrationContext.js` — 트리 대조
- 선택적 하이드레이션(selective hydration)
- 하이드레이션 불일치 경고가 나오는 지점

**19주차: 서버 컴포넌트와 Flight**
- `react-server` 의 직렬화 포맷
- 클라이언트 참조(client reference)와 모듈 맵
- 서버/클라이언트 경계가 번들러와 맞물리는 방식

**20주차: Activity와 View Transition**
- Offscreen/Activity 컴포넌트와 상태 보존
- `ReactFiberCommitViewTransitions.js`
- 커밋 단계가 비동기로 쪼개지는 경로 (`startViewTransition`)

### Phase 5: React Compiler와 종합 (21-24주차)

**21주차: React Compiler 개요**
- `compiler/` 워크스페이스 구조
- 무엇을 입력으로 받아 무엇을 출력하는가
- 메모이제이션 자동 삽입의 조건

**22주차: Compiler 내부 — HIR**
- 파싱 → HIR → 최적화 패스 → 코드 생성
- 추론하는 불변식과 포기(bailout) 조건
- 컴파일 결과와 손으로 쓴 `useMemo` 비교

**23주차: 디버깅과 성능 측정**
- DevTools Profiler가 읽는 데이터의 출처 (`ReactFiberPerformanceTrack.js`)
- `enableProfilerTimer` 계열 플래그
- 실제 성능 문제를 단계별로 좁히는 절차

**24주차: 회고와 지도 그리기**
- 전체 파이프라인 개념 지도 정리
- 기능별 "어느 단계에서 결정되는가" 표 완성
- React RFC / 실험 플래그(`ReactFeatureFlags.js`) 읽기 가이드

---

## 학습 진도표

| Phase | 주차 | 주제 | 상태 |
|-------|------|------|------|
| 1 | 1-5 | 렌더 파이프라인 | 🟡 1주차 완료 |
| 2 | 6-10 | 훅의 구현 | ⬜ |
| 3 | 11-15 | 스케줄링 | ⬜ |
| 4 | 16-20 | Concurrent 기능 | ⬜ |
| 5 | 21-24 | React Compiler와 종합 | ⬜ |

## 학습 팁

1. **코드를 먼저 열어라** — 설명을 읽고 이해한 기분이 드는 것과, `ReactFiberWorkLoop.js` 를 열어 함수를 찾는 것은 다르다
2. **`__DEV__` 블록은 건너뛰며 읽어라** — 개발 전용 경고가 본류 로직보다 길다. 처음 읽을 때는 무시하고 골격만 본다
3. **feature flag를 확인하라** — `enableViewTransition` 처럼 플래그로 감싼 코드는 릴리스 빌드에서 빠질 수 있다. `ReactFeatureFlags.js` 를 함께 본다
4. **주석을 신뢰하라** — React 소스의 주석은 밀도가 높고 "왜"를 담고 있다. 인용할 가치가 있는 1차 자료다
5. **버전을 고정하라** — main은 계속 움직인다. 기준 커밋을 정하고, 갱신할 때는 문서의 줄번호를 함께 갱신한다

## 참고 자료

- [react.dev — Learn React](https://react.dev/learn) : 공개 API의 정확한 계약(contract)
- [facebook/react](https://github.com/facebook/react) : 구현의 1차 자료
- [React RFCs](https://github.com/reactjs/rfcs) : 설계 의도와 논의 과정
