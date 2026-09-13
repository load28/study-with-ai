# 1주차: 렌더와 커밋 — React가 화면을 그리는 3단계

## 왜 이것부터 시작하는가?

React를 쓰면서 마주치는 대부분의 의문 — "왜 리렌더가 안 되지", "왜 두 번 실행되지", "`useEffect` 와 `useLayoutEffect` 는 언제 다른가", "메모이제이션은 왜 안 먹지" — 는 같은 질문으로 환원된다. **지금 React는 어느 단계에 있는가.**

React는 화면을 세 단계로 나눠 갱신한다. 트리거(trigger), 렌더(render), 커밋(commit)이다. 이 구분은 문서상의 비유가 아니라 소스 코드의 구조다. 렌더 단계와 커밋 단계는 별개의 함수 묶음이고, `executionContext` 라는 전역 비트마스크로 지금 어느 단계인지 표시하며, 단계마다 할 수 있는 일이 다르다.

이 문서는 세 단계를 실제 코드로 따라간다.

---

## 이 문서의 근거

| 항목 | 값 |
|------|-----|
| 저장소 | `facebook/react` |
| 커밋 | `019019be403c3269e15b8d7ebefb57d30f84086b` (2026-09-11) |
| `ReactVersion` | `19.3.0` |

경로는 저장소 루트 기준이며, 줄번호는 위 커밋 기준이다. 인용한 코드는 원문 그대로이고, 코드 안의 주석도 React 소스의 주석이다.

---

## 0. 전체 지도

한 번의 화면 갱신이 지나가는 경로다. 이 표가 이 문서의 요약이다.

| 단계 | 진입점 | 파일 |
|------|--------|------|
| Trigger | `scheduleUpdateOnFiber` | `packages/react-reconciler/src/ReactFiberWorkLoop.js:987` |
| (스케줄) | `ensureRootIsScheduled` → `processRootScheduleInMicrotask` | `ReactFiberRootScheduler.js:116`, `:259` |
| Render | `performWorkOnRoot` → `renderRootSync` → `workLoopSync` | `ReactFiberWorkLoop.js:1137`, `:2623`, `:2772` |
| (트리 순회) | `performUnitOfWork` → `beginWork` / `completeUnitOfWork` | `ReactFiberWorkLoop.js:3081`, `:3368` |
| Commit | `completeRoot` → `commitRoot` | `ReactFiberWorkLoop.js:3511`, `:3725` |
| (커밋 3단계) | `commitBeforeMutationEffects` → `flushMutationEffects` → `flushLayoutEffects` | `:3874`, `:4009`, `:4055` |
| paint 요청 | `flushSpawnedWork` 의 `requestPaint()` | `ReactFiberWorkLoop.js:4189` |
| passive 이펙트 | `flushPassiveEffects` | `ReactFiberWorkLoop.js:4691` |

---

## 1. Step 1 — Trigger: 렌더는 어떻게 시작되는가

렌더가 시작되는 경로는 두 가지뿐이다. 초기 렌더와 상태 업데이트다. 둘 다 **같은 함수**로 수렴한다.

### 1.1 초기 렌더

`root.render(<App />)` 는 `react-dom` 에서 리컨실러로 곧바로 넘어간다.

`packages/react-dom/src/client/ReactDOMRoot.js:107`

```js
ReactDOMHydrationRoot.prototype.render = ReactDOMRoot.prototype.render =
  // $FlowFixMe[missing-this-annot]
  function (children: ReactNodeList): void {
    const root = this._internalRoot;
    if (root === null) {
      throw new Error('Cannot update an unmounted root.');
    }
    // ... (DEV 경고 생략)
    updateContainer(children, root, null, null);
  };
```

`updateContainer` 는 `updateContainerImpl` 로 위임하고, 그 끝은 이렇다.

`packages/react-reconciler/src/ReactFiberReconciler.js:433`

```js
  const update = createUpdate(lane);
  // Caution: React DevTools currently depends on this property
  // being called "element".
  update.payload = {element};
```

`packages/react-reconciler/src/ReactFiberReconciler.js:452`

```js
  const root = enqueueUpdate(rootFiber, update, lane);
  if (root !== null) {
    scheduleUpdateOnFiber(root, rootFiber, lane);
    entangleTransitions(root, rootFiber, lane);
  }
```

주목할 점은 `root.render()` 가 **렌더를 실행하지 않는다**는 것이다. 엘리먼트를 payload로 담은 업데이트 객체를 루트 Fiber의 큐에 넣고, 스케줄만 요청한다.

### 1.2 상태 업데이트

`setState` 는 훅이 반환한 `dispatch` 함수다. 실체는 `dispatchSetState` → `dispatchSetStateInternal` 이다.

`packages/react-reconciler/src/ReactFiberHooks.js:3662` (`dispatchSetStateInternal`)

```js
  const update: Update<S, A> = {
    lane,
    revertLane: NoLane,
    gesture: null,
    action,
    hasEagerState: false,
    eagerState: null,
    next: null as any,
  };
```

`packages/react-reconciler/src/ReactFiberHooks.js:3718`

```js
    const root = enqueueConcurrentHookUpdate(fiber, queue, update, lane);
    if (root !== null) {
      scheduleUpdateOnFiber(root, fiber, lane);
      entangleTransitionUpdate(root, queue, lane);
      return true;
    }
```

초기 렌더와 구조가 같다. **업데이트 객체를 큐에 넣고 `scheduleUpdateOnFiber` 를 호출한다.** 여기까지가 트리거 단계 전부다.

### 1.3 `setState` 가 렌더를 건너뛰는 경우

큐가 비어 있으면 React는 다음 상태를 **미리 계산해 본다**. 결과가 현재 상태와 같으면 스케줄 자체를 하지 않는다.

`packages/react-reconciler/src/ReactFiberHooks.js:3675` 이하 (`dispatchSetStateInternal`)

```js
    const alternate = fiber.alternate;
    if (
      fiber.lanes === NoLanes &&
      (alternate === null || alternate.lanes === NoLanes)
    ) {
      // The queue is currently empty, which means we can eagerly compute the
      // next state before entering the render phase. If the new state is the
      // same as the current state, we may be able to bail out entirely.
      const lastRenderedReducer = queue.lastRenderedReducer;
      if (lastRenderedReducer !== null) {
```

`packages/react-reconciler/src/ReactFiberHooks.js:3697`

```js
          update.hasEagerState = true;
          update.eagerState = eagerState;
          if (is(eagerState, currentState)) {
            // Fast path. We can bail out without scheduling React to re-render.
```

`is` 는 `Object.is` 다. 즉 **같은 값으로 `setState` 하면 리렌더가 일어나지 않는다.** 반대로 객체를 얕게 복사해 넣으면(`setUser({...user})`) `Object.is` 가 false이므로 내용이 같아도 렌더가 스케줄된다. 이 두 가지가 하나의 조건문에서 나온다.

조건에 `fiber.lanes === NoLanes` 가 붙어 있는 점도 중요하다. 이미 대기 중인 업데이트가 있으면 미리 계산을 시도하지 않으므로, "같은 값인데 렌더가 일어났다"는 상황도 정상 동작이다.

### 1.4 `scheduleUpdateOnFiber` 가 하는 일

`packages/react-reconciler/src/ReactFiberWorkLoop.js:1026`

```js
  // Mark that the root has a pending update.
  markRootUpdated(root, lane);
```

`packages/react-reconciler/src/ReactFiberWorkLoop.js:1093`

```js
    ensureRootIsScheduled(root);
```

루트에 "처리할 작업이 이 Lane에 있다"고 표시하고, 스케줄을 보장한다. 여기서도 렌더는 실행되지 않는다.

### 1.5 배칭의 실체

`ensureRootIsScheduled` 의 주석이 배칭을 그대로 설명한다.

`packages/react-reconciler/src/ReactFiberRootScheduler.js:116`

```js
export function ensureRootIsScheduled(root: FiberRoot): void {
  // This function is called whenever a root receives an update. It does two
  // things 1) it ensures the root is in the root schedule, and 2) it ensures
  // there's a pending microtask to process the root schedule.
  //
  // Most of the actual scheduling logic does not happen until
  // `scheduleTaskForRootDuringMicrotask` runs.
```

마이크로태스크는 **한 번만** 예약된다.

`packages/react-reconciler/src/ReactFiberRootScheduler.js:164` (`ensureScheduleIsScheduled`)

```js
    if (!didScheduleMicrotask) {
      didScheduleMicrotask = true;
      scheduleImmediateRootScheduleTask();
    }
```

한 이벤트 핸들러 안에서 `setState` 를 세 번 호출해도, 세 번 모두 업데이트 큐에 쌓이고 `didScheduleMicrotask` 플래그 때문에 마이크로태스크는 하나만 예약된다. 그 마이크로태스크(`processRootScheduleInMicrotask`, `:259`)가 루트별로 한 번씩 작업을 스케줄한다. **자동 배칭은 별도의 기능이 아니라 이 플래그 하나의 결과다.**

---

## 2. Step 2 — Render: 컴포넌트를 호출해 트리를 만든다

### 2.1 렌더 단계의 표시

`packages/react-reconciler/src/ReactFiberWorkLoop.js:2628` (`renderRootSync`)

```js
  const prevExecutionContext = executionContext;
  executionContext |= RenderContext;
```

`RenderContext` 비트가 켜져 있는 동안이 렌더 단계다. React 내부는 이 비트로 "지금 렌더 중"을 판단한다. 렌더 중 `setState` 가 호출되면 `scheduleUpdateOnFiber` 가 이 비트를 보고 다른 경로로 분기한다(`ReactFiberWorkLoop.js:1029` 이하).

### 2.2 작업 루프

`packages/react-reconciler/src/ReactFiberWorkLoop.js:2770`

```js
// The work loop is an extremely hot path. Tell Closure not to inline it.
/** @noinline */
function workLoopSync() {
  // Perform work without checking if we need to yield between fiber.
  while (workInProgress !== null) {
    performUnitOfWork(workInProgress);
  }
}
```

렌더는 **Fiber 하나를 단위로 하는 반복문**이다. 동기 버전은 끝까지 돌지만, Concurrent 버전(`workLoopConcurrent`, `:3056`)은 같은 반복문에 양보 검사를 끼워 넣는다. 트리를 재귀 호출로 내려가지 않고 반복문으로 도는 구조 덕분에 중간에 멈출 수 있다.

### 2.3 내려가기: `beginWork`

`packages/react-reconciler/src/ReactFiberWorkLoop.js:3081` (`performUnitOfWork`, DEV 분기 생략)

```js
function performUnitOfWork(unitOfWork: Fiber): void {
  // The current, flushed, state of this fiber is the alternate. Ideally
  // nothing should rely on this, but relying on it here means that we don't
  // need an additional field on the work in progress.
  const current = unitOfWork.alternate;

  let next;
  // ...
      next = beginWork(current, unitOfWork, entangledRenderLanes);
  // ...
  unitOfWork.memoizedProps = unitOfWork.pendingProps;
  if (next === null) {
    // If this doesn't spawn new work, complete the current work.
    completeUnitOfWork(unitOfWork);
  } else {
    workInProgress = next;
  }
}
```

`beginWork` 가 자식 Fiber를 반환하면 그 자식으로 내려간다. 반환값이 없으면(더 내려갈 곳이 없으면) 완료 처리로 넘어간다. 컴포넌트 함수가 실제로 호출되는 지점은 `beginWork` 안쪽의 `updateFunctionComponent` → `renderWithHooks` 다.

### 2.4 올라가기: `completeUnitOfWork`

`packages/react-reconciler/src/ReactFiberWorkLoop.js:3368` (발췌)

```js
    const siblingFiber = completedWork.sibling;
    if (siblingFiber !== null) {
      // If there is more work to do in this returnFiber, do that next.
      workInProgress = siblingFiber;
      return;
    }
    // Otherwise, return to the parent
    // $FlowFixMe[incompatible-type] we bail out when we get a null
    completedWork = returnFiber;
```

순회 규칙은 세 줄로 요약된다. **자식이 있으면 자식으로, 없으면 형제로, 형제도 없으면 부모로.** 이것이 Fiber 트리의 깊이 우선 순회이고, `child` / `sibling` / `return` 세 포인터만으로 재귀 없이 트리를 다 돈다.

### 2.5 리렌더를 건너뛰는 곳: `beginWork` 의 bailout

부모가 리렌더되어도 자식이 항상 다시 호출되는 것은 아니다. 판단은 `beginWork` 진입부에서 이뤄진다.

`packages/react-reconciler/src/ReactFiberBeginWork.js:4247` (`beginWork`)

```js
  if (current !== null) {
    const oldProps = current.memoizedProps;
    const newProps = workInProgress.pendingProps;

    if (
      oldProps !== newProps ||
      hasLegacyContextChanged() ||
      // Force a re-render if the implementation changed due to hot reload:
      (__DEV__ ? workInProgress.type !== current.type : false)
    ) {
      // If props or context changed, mark the fiber as having performed work.
      // This may be unset if the props are determined to be equal later (memo).
      didReceiveUpdate = true;
    } else {
      // Neither props nor legacy context changes. Check if there's a pending
      // update or context change.
      const hasScheduledUpdateOrContext = checkScheduledUpdateOrContext(
        current,
        renderLanes,
      );
      if (
        !hasScheduledUpdateOrContext &&
        // If this is the second pass of an error or suspense boundary, there
        // may not be work scheduled on `current`, so we check for this flag.
        (workInProgress.flags & DidCapture) === NoFlags
      ) {
        // No pending updates or context. Bail out now.
        didReceiveUpdate = false;
        return attemptEarlyBailoutIfNoScheduledUpdate(
          current,
          workInProgress,
          renderLanes,
        );
      }
```

`oldProps !== newProps` — **참조 비교 한 번**이다. props 객체는 JSX가 평가될 때마다 새로 만들어지므로, 부모가 리렌더되면 자식의 props 객체는 (내용이 같아도) 새 객체다. 따라서 기본적으로 자식도 렌더된다. `React.memo` 는 이 비교를 얕은 비교로 바꿔 끼우는 장치다.

건너뛰기가 성립하면 다음 함수로 간다.

`packages/react-reconciler/src/ReactFiberBeginWork.js:3823` (`bailoutOnAlreadyFinishedWork`, 발췌)

```js
  // Check if the children have any pending work.
  if (!includesSomeLane(renderLanes, workInProgress.childLanes)) {
    // The children don't have any work either. We can skip them.
```

`childLanes` 는 서브트리에 대기 중인 작업이 있는지를 담은 요약 정보다. 이게 있어야 "이 컴포넌트는 건너뛰되, 그 아래 깊은 곳의 업데이트는 처리"가 가능하다. 없으면 서브트리 전체를 잘라낸다(`return null`).

### 2.6 렌더 단계는 DOM을 만지지 않는다

렌더 단계에서 만들어지는 것은 새 Fiber 트리와 flags뿐이다. 실제 DOM 조작 함수는 모두 커밋 단계 파일에 있다(`ReactFiberCommitHostEffects.js`). 렌더 도중 `document.querySelector` 로 DOM을 읽으면 **커밋되기 전의 낡은 DOM**을 보게 된다. 화면에 붙어 있는 트리는 여전히 `root.current` 이고, 그 교체는 커밋 단계에서 일어난다(§3.3).

그리고 React는 렌더 단계의 순수성을 개발 모드에서 강제로 검사한다.

`packages/react-reconciler/src/ReactFiberHooks.js:595` (`renderWithHooks`)

```js
  const shouldDoubleRenderDEV =
    __DEV__ && (workInProgress.mode & StrictLegacyMode) !== NoMode;

  shouldDoubleInvokeUserFnsInHooksDEV = shouldDoubleRenderDEV;
  let children = __DEV__
    ? callComponentInDEV(Component, props, secondArg)
    : Component(props, secondArg);
  shouldDoubleInvokeUserFnsInHooksDEV = false;
```

`packages/react-reconciler/src/ReactFiberHooks.js:616`

```js
  if (shouldDoubleRenderDEV) {
    // In development, components are invoked twice to help detect side effects.
    setIsStrictModeForDevtools(true);
    try {
      children = renderWithHooksAgain(
        workInProgress,
        Component,
        props,
        secondArg,
      );
    } finally {
      setIsStrictModeForDevtools(false);
    }
  }
```

**"컴포넌트가 두 번 실행된다"의 정확한 조건**이 여기 있다. `__DEV__` 이고 `StrictMode` 트리 안일 때만, 렌더 단계에서만 두 번 호출된다. 프로덕션 빌드에서는 `__DEV__` 가 false이므로 한 번이다. 두 번 호출해도 결과가 같아야 한다는 요구가 곧 "렌더 단계는 순수해야 한다"는 규칙이다.

---

## 3. Step 3 — Commit: 화면에 반영한다

렌더가 끝나면 `completeRoot`(`:3511`)가 `commitRoot`(`:3725`)를 호출한다(호출 지점 `:3712`). 커밋은 세 단계로 나뉜다.

### 3.1 before mutation

`packages/react-reconciler/src/ReactFiberWorkLoop.js:3871`

```js
      // The first phase a "before mutation" phase. We use this phase to read the
      // state of the host tree right before we mutate it. This is where
      // getSnapshotBeforeUpdate is called.
      commitBeforeMutationEffects(root, finishedWork, lanes);
```

DOM을 바꾸기 **직전**의 상태를 읽는 단계다.

### 3.2 세 단계의 실행

`packages/react-reconciler/src/ReactFiberWorkLoop.js:3915`

```js
  } else {
    // Flush synchronously.
    flushMutationEffects();
    flushLayoutEffects();
    // Skip flushAfterMutationEffects
    flushSpawnedWork();
  }
```

View Transition이 관여하지 않는 일반 경로에서는 세 함수가 **동기로 연달아** 실행된다. 커밋 단계는 중단되지 않는다는 성질이 이 코드에 드러난다. (View Transition이 있으면 같은 함수들이 콜백으로 넘겨져 브라우저 API의 타이밍에 맞춰 실행된다. `:3896` 이하.)

### 3.3 mutation — DOM이 바뀌는 지점

`packages/react-reconciler/src/ReactFiberWorkLoop.js:4030` (`flushMutationEffects`)

```js
      // The next phase is the mutation phase, where we mutate the host tree.
      commitMutationEffects(root, finishedWork, lanes);
```

실제 DOM 갱신은 호스트 설정으로 위임된다.

`packages/react-reconciler/src/ReactFiberCommitHostEffects.js:113` (`commitHostUpdate`, 발췌)

```js
      commitUpdate(
        finishedWork.stateNode,
        finishedWork.type,
        oldProps,
        newProps,
        finishedWork,
      );
```

그리고 mutation 직후, 이 한 줄이 트리를 교체한다.

`packages/react-reconciler/src/ReactFiberWorkLoop.js:4047`

```js
  // The work-in-progress tree is now the current tree. This must come after
  // the mutation phase, so that the previous tree is still current during
  // componentWillUnmount, but before the layout phase, so that the finished
  // work is current during componentDidMount/Update.
  root.current = finishedWork;
```

주석이 위치의 이유까지 설명한다. **정리(cleanup) 시점에는 아직 이전 트리가 current이고, layout 이펙트 시점에는 새 트리가 current다.** 포인터 하나를 바꾸는 것으로 트리 전환이 원자적으로 끝난다.

### 3.4 layout — paint 전에 동기로 실행

`packages/react-reconciler/src/ReactFiberWorkLoop.js:4119` (`flushLayoutEffects`)

```js
      // The next phase is the layout phase, where we call effects that read
      // the host tree after it's been mutated. The idiomatic use case for this is
      // layout, but class component lifecycles also fire here for legacy reasons.
      commitLayoutEffects(finishedWork, root, lanes);
```

`useLayoutEffect` 의 콜백이 실행되는 곳이다. DOM은 이미 갱신됐고(§3.3), 브라우저는 아직 그리지 않았다. 그래서 여기서 레이아웃을 측정하고 곧바로 `setState` 를 해도 사용자는 중간 상태를 보지 못한다.

`packages/react-reconciler/src/ReactFiberCommitEffects.js:96` (`commitHookLayoutEffects`)

```js
export function commitHookLayoutEffects(
  finishedWork: Fiber,
  hookFlags: HookFlags,
) {
  // At this point layout effects have already been destroyed (during mutation phase).
  // This is done to prevent sibling component effects from interfering with each other,
  // e.g. a destroy function in one component should never override a ref set
  // by a create function in another component during the same commit.
```

정리 함수는 mutation 단계에서 이미 전부 실행됐다는 사실도 주석에 있다. **모든 정리가 끝난 뒤 모든 생성이 실행된다**는 순서 보장이다.

### 3.5 paint 요청

`packages/react-reconciler/src/ReactFiberWorkLoop.js:4187` (`flushSpawnedWork`)

```js
  // Tell Scheduler to yield at the end of the frame, so the browser has an
  // opportunity to paint.
  requestPaint();
```

### 3.6 passive — paint 이후

`useEffect` 는 여기 속하지 않는다. 별도 태스크로 예약된다.

`packages/react-reconciler/src/ReactFiberWorkLoop.js:3806` (`completeRoot`)

```js
      scheduleCallback(NormalSchedulerPriority, () => {
```

`packages/react-reconciler/src/ReactFiberWorkLoop.js:3815`

```js
        flushPassiveEffects();
```

`NormalSchedulerPriority` 로 예약된 콜백은 커밋 이후에 실행된다. 그래서 `useEffect` 의 콜백은 브라우저가 화면을 그린 뒤에 돌고, `useLayoutEffect` 는 그 전에 돈다. **두 훅의 차이는 API 이름이 아니라 호출되는 단계가 다른 것이다.**

이 차이가 만드는 결과는 분명하다. `useLayoutEffect` 안의 무거운 작업은 paint를 막아 체감 지연이 되고, `useEffect` 안에서 DOM 레이아웃을 측정해 위치를 고치면 사용자는 고쳐지기 전 프레임을 볼 수 있다.

---

## 4. 세 단계로 현상을 진단하기

| 현상 | 해당 단계 | 코드상 원인 |
|------|-----------|-------------|
| `setState` 했는데 리렌더가 없다 | Trigger | `is(eagerState, currentState)` bailout — `ReactFiberHooks.js:3699` |
| 여러 번 `setState` 했는데 렌더는 한 번 | Trigger | `didScheduleMicrotask` 플래그 — `ReactFiberRootScheduler.js:165` |
| 컴포넌트 함수가 두 번 실행된다 | Render | `__DEV__` + `StrictMode` 이중 호출 — `ReactFiberHooks.js:595` |
| props를 안 바꿨는데 자식이 렌더된다 | Render | `oldProps !== newProps` 참조 비교 — `ReactFiberBeginWork.js:4252` |
| 렌더 중 읽은 DOM이 낡았다 | Render | `root.current` 교체는 커밋 단계 — `ReactFiberWorkLoop.js:4051` |
| 화면 깜빡임이 보인다 | Commit | 측정·보정을 passive 단계에서 함 — `ReactFiberWorkLoop.js:3806` |

진단 절차는 이 순서가 된다. **① 업데이트가 스케줄되긴 했는가 → ② 컴포넌트 함수가 호출되긴 했는가 → ③ DOM에 반영되긴 했는가.** DevTools Profiler로 ②를, Elements 패널로 ③을 확인하면 어느 단계에서 끊겼는지 특정할 수 있다.

---

## 핵심 정리

| 개념 | 한 줄 요약 |
|------|-----------|
| Trigger | 업데이트 객체를 큐에 넣고 `scheduleUpdateOnFiber` 를 호출할 뿐, 렌더는 하지 않는다 |
| 자동 배칭 | 마이크로태스크를 한 번만 예약하는 `didScheduleMicrotask` 플래그의 결과 |
| Render | `workLoopSync` 가 Fiber 단위 반복문으로 `beginWork`/`completeUnitOfWork` 를 돈다 |
| 트리 순회 | 자식 → 형제 → 부모, `child`/`sibling`/`return` 세 포인터로 재귀 없이 |
| Bailout | `oldProps !== newProps` 참조 비교와 `childLanes` 요약으로 서브트리를 건너뛴다 |
| Commit | before mutation → mutation → layout, 세 단계가 동기로 연달아 실행된다 |
| 트리 전환 | `root.current = finishedWork` 한 줄. mutation 뒤, layout 앞 |
| layout vs passive | `useLayoutEffect` 는 paint 전 커밋 단계, `useEffect` 는 paint 후 별도 태스크 |

## 스스로 확인하기

1. `setState(같은 값)` 을 호출했을 때 렌더가 일어나지 않는 조건을 코드로 설명할 수 있는가? 항상 그런가?
2. 자식에게 `props` 를 하나도 넘기지 않아도 부모가 렌더되면 자식이 렌더되는가? 어느 비교 때문인가?
3. `useLayoutEffect` 에서 `setState` 를 호출하면 사용자는 중간 상태를 보는가? 근거는 어느 줄인가?
4. 렌더 단계에서 `document.getElementById` 로 읽은 값이 낡은 이유를 `root.current` 로 설명할 수 있는가?

## 다음 주차 예고

**2주차: Fiber 자료구조와 이중 버퍼링** — 이번 주 내내 등장한 `workInProgress`, `current`, `alternate` 가 정확히 무엇인지 `ReactFiber.js` 를 열어 확인한다. Fiber 객체의 필드를 전수 조사하고, `createWorkInProgress` 가 트리를 어떻게 복제·재활용하는지, `root.current` 스위칭이 왜 안전한지를 코드로 따라간다.

## 참고

- 공개 API 관점의 같은 내용: [react.dev — Render and Commit](https://react.dev/learn/render-and-commit)
- 이 문서가 읽은 코드: `packages/react-reconciler/src/` 의 `ReactFiberWorkLoop.js`, `ReactFiberBeginWork.js`, `ReactFiberHooks.js`, `ReactFiberRootScheduler.js`, `ReactFiberCommitEffects.js`, `ReactFiberCommitHostEffects.js`, `ReactFiberReconciler.js` / `packages/react-dom/src/client/ReactDOMRoot.js`
