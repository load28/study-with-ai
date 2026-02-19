# 함수형 프로그래밍 심화 학습 로드맵

## 학습 목표
프로그래밍의 **수학적 근본 원리**부터 시작하여 Functor, Applicative, Monad, 그리고 카테고리 이론까지 체계적으로 마스터합니다. 모든 개념은 **프론트엔드 프로덕션 코드**에서의 실전 활용과 함께 학습합니다.

## 학습 철학
- **원리부터**: "어떻게" 보다 "왜" 이런 구조가 필요한지부터
- **수학적 기초**: 카테고리 이론의 직관을 코드로 번역
- **프론트엔드 실전**: React, TypeScript 기반 프로덕션 예시 중심
- **점진적 심화**: 각 개념이 다음 개념의 기초가 됨
- **법칙 검증**: 모든 추상화는 법칙(Laws)을 만족해야 의미가 있음

## 사용 언어 & 도구
- **TypeScript** — 메인 언어 (프론트엔드 실무 기준)
- **React** — UI 레이어 실전 예시
- **fp-ts / Effect** — 실무 FP 라이브러리
- **Haskell** (보조) — 개념의 순수한 형태를 볼 때 참고용

## 학습 방법
- 각 주차는 독립적인 문서로 제공됩니다
- **원리 설명 → 법칙 증명 → TypeScript 구현 → 프론트엔드 실전 적용** 순서
- 각 문서는 `week-XXX.md` 형식으로 저장됩니다 (3자리)
- 모든 코드는 직접 실행 가능한 프로덕션 수준

## 전체 학습 계획 (40주 과정 = 약 10개월)

> 💡 **핵심 원칙**: 각 주차의 "왜 필요한가?" 섹션이 가장 중요합니다. 코드 패턴을 외우지 말고 **문제를 이해**하세요.

---

### Phase 1: 함수형 사고의 기초 (1-4주차)
> 명령형에서 함수형으로 패러다임 전환

**1주차: 함수란 무엇인가 — 수학적 함수와 프로그래밍**
- 수학적 함수의 정의 (정의역, 공역, 치역)
- 전사·단사·전단사 함수
- 순수 함수(Pure Function)와 참조 투명성(Referential Transparency)
- 부수 효과(Side Effect)가 왜 문제인가
- **프론트엔드 실전**: React 컴포넌트를 순수 함수로 바라보기, 같은 props → 같은 UI

**2주차: 일급 함수와 고차 함수**
- 일급 시민(First-class Citizen)으로서의 함수
- 고차 함수(Higher-order Function)의 정의와 활용
- 클로저(Closure)와 렉시컬 스코프
- 커링(Currying)과 부분 적용(Partial Application)
- **프론트엔드 실전**: 이벤트 핸들러 팩토리, 설정 가능한 API 클라이언트, HOC 패턴

**3주차: 불변성과 영속 자료구조**
- 불변성(Immutability)의 정의와 가치
- 구조적 공유(Structural Sharing)
- Object.freeze vs Immer vs 영속 자료구조
- 불변 업데이트 패턴과 성능
- **프론트엔드 실전**: React 상태 불변 업데이트, Redux 리듀서, 깊은 객체 렌즈 패턴

**4주차: 합성과 파이프라인**
- 함수 합성(Composition): f ∘ g
- 합성의 결합법칙(Associativity)
- pipe와 flow 구현
- Point-free 스타일
- **프론트엔드 실전**: 데이터 변환 파이프라인, 미들웨어 체이닝, API 응답 정규화

---

### Phase 2: 타입 시스템과 대수적 구조 (5-8주차)
> 타입으로 불가능한 상태를 불가능하게 만들기

**5주차: 대수적 데이터 타입 (ADT)**
- 곱 타입(Product Type): 튜플, 레코드
- 합 타입(Sum Type): 태그드 유니언, 판별 유니언
- 타입 대수: |A × B| = |A| × |B|, |A + B| = |A| + |B|
- 패턴 매칭과 완전성 검사(Exhaustiveness)
- **프론트엔드 실전**: API 응답 모델링(Loading|Success|Error), 폼 상태 머신, 라우트 타입

**6주차: Option/Maybe — null을 타입으로 다루기**
- null/undefined의 근본적 문제 (10억 달러의 실수)
- Option<A> = Some<A> | None 타입 구현
- map, flatMap, getOrElse, fold 연산
- 옵셔널 체이닝(?.)과의 비교와 한계
- **프론트엔드 실전**: 안전한 DOM 접근, 중첩 객체 안전 탐색, 설정값 폴백 체인

**7주차: Either와 Result — 에러를 값으로 다루기**
- 예외(Exception)의 문제점: 타입 시스템을 우회
- Either<E, A> = Left<E> | Right<A> 구현
- 에러 합성과 체이닝
- try-catch를 Either로 변환하기
- **프론트엔드 실전**: API 에러 처리 파이프라인, 폼 유효성 검증 체이닝, 에러 바운더리 대체

**8주차: 재귀와 대수적 재귀 구조**
- 재귀 함수와 꼬리 호출 최적화
- 트램폴린(Trampoline)으로 스택 안전 확보
- 재귀적 데이터 구조: 리스트, 트리
- fold/catamorphism — 재귀 구조의 일반적 소비
- **프론트엔드 실전**: 재귀 컴포넌트(트리 뷰, 중첩 메뉴), JSON 스키마 순회, 가상 DOM diff

---

### Phase 3: Functor와 그 친구들 (9-14주차)
> 컨텍스트 안의 값을 다루는 추상화

**9주차: Functor — 매핑의 일반화**
- Functor의 정의: fmap / map
- **Functor 법칙 2가지**
  - 항등 법칙: map(id) === id
  - 합성 법칙: map(f ∘ g) === map(f) ∘ map(g)
- Array, Option, Either, Promise를 Functor로 바라보기
- TypeScript로 Functor 타입클래스 구현
- **프론트엔드 실전**: React 컴포넌트 트리의 Functor 구조, 상태 변환 map 패턴

**10주차: Bifunctor와 Contravariant Functor**
- Bifunctor: 두 타입 파라미터 모두 매핑
- Either의 Bifunctor 인스턴스 (mapLeft, mapRight, bimap)
- Contravariant Functor: 입력 쪽을 변환
- Profunctor 맛보기
- **프론트엔드 실전**: 에러와 성공을 동시에 변환, 비교 함수(Comparator) 변환, 정렬 로직 합성

**11주차: Applicative Functor — 독립적 효과의 합성**
- Functor의 한계: map으로는 여러 컨텍스트를 합칠 수 없다
- Applicative의 정의: pure + ap (또는 liftA2)
- **Applicative 법칙 4가지** (Identity, Composition, Homomorphism, Interchange)
- Applicative vs Monad: 독립 vs 의존
- **프론트엔드 실전**: 병렬 API 호출 합성, 다중 폼 필드 유효성 검증, 독립적 설정 로딩 합성

**12주차: Traversable과 Sequence**
- Traversable의 정의: traverse + sequence
- "리스트의 Option"을 "Option의 리스트"로 뒤집기
- traverse의 핵심: 효과를 모으면서 구조를 보존
- **Traversable 법칙**
- **프론트엔드 실전**: 배열 내 모든 API 호출을 순서대로/병렬로, 폼 필드 일괄 유효성 검증

**13주차: Semigroup과 Monoid**
- Semigroup: 결합법칙을 만족하는 이항 연산 (concat)
- Monoid: Semigroup + 항등원 (empty)
- **Monoid 법칙**: 결합법칙, 좌항등, 우항등
- 다양한 Monoid: Sum, Product, All, Any, First, Last
- fold/reduce는 Monoid의 활용
- **프론트엔드 실전**: 설정 병합(Config Monoid), 로그 수집, 필터 조건 합성, CSS 클래스 합성

**14주차: Foldable과 구조 소비**
- Foldable의 정의: 구조를 하나의 값으로 접기
- foldMap: Foldable + Monoid의 시너지
- reduce, reduceRight와의 관계
- Foldable에서 다양한 연산 유도 (toArray, length, sum, any, all)
- **프론트엔드 실전**: 트리 구조 집계(장바구니 총액, 권한 체크), 컴포넌트 트리 분석

---

### Phase 4: Monad 완전 정복 (15-22주차)
> 순차적 효과를 합성하는 핵심 추상화

**15주차: Monad의 본질 — 왜 flatMap인가**
- Functor → Applicative → Monad 계층 구조
- Monad의 정의: pure(of) + flatMap(chain/bind)
- **Monad 법칙 3가지** (좌항등, 우항등, 결합법칙)
- flatMap이 해결하는 문제: 중첩 컨텍스트 평탄화
- join: M<M<A>> → M<A>
- Kleisli 합성: A → M<B>를 합성하기
- **프론트엔드 실전**: 비동기 체이닝의 본질, 옵셔널 체이닝 재발견

**16주차: Identity Monad와 Maybe Monad 심화**
- Identity Monad: 가장 단순한 Monad 이해
- Maybe(Option) Monad 직접 구현
- do-notation 스타일 (generator 기반)
- 법칙 검증: 테스트 코드로 증명
- **프론트엔드 실전**: 안전한 데이터 파이프라인, 중첩 API 응답 안전 추출

**17주차: Either Monad와 에러 누적**
- Either Monad: 단락 평가(Short-circuit)
- Validation: Either의 Applicative — 에러를 누적하기
- Either vs Validation: 언제 무엇을 쓰나
- 커스텀 에러 타입 설계
- **프론트엔드 실전**: 단계별 폼 유효성 검증, API 에러 체이닝과 누적 에러 표시

**18주차: Reader Monad — 의존성 주입**
- Reader<R, A>: 환경에서 값을 읽는 컴퓨팅
- ask, local, reader 연산
- Reader Monad로 의존성 주입 패턴 구현
- **Reader 법칙** 검증
- **프론트엔드 실전**: 테마/설정 주입, React Context의 함수형 대안, 테스트 가능한 서비스 레이어

**19주차: Writer Monad — 로깅과 부가 출력**
- Writer<W, A>: 값과 함께 로그를 누적
- tell, listen, pass 연산
- W는 Monoid여야 하는 이유
- **Writer 법칙** 검증
- **프론트엔드 실전**: 디버그 로그 누적, 분석 이벤트 수집, 변환 과정 추적 파이프라인

**20주차: State Monad — 상태를 순수하게**
- State<S, A>: S → [A, S]
- get, put, modify 연산
- State Monad로 상태 기계 구현
- **State 법칙** 검증
- **프론트엔드 실전**: 순수한 폼 상태 관리, 되돌리기(Undo) 구현, 게임 상태 로직

**21주차: IO Monad — 부수 효과를 가두기**
- IO<A>: () → A — 지연된 부수 효과
- IO의 핵심: 실행을 기술(describe)과 분리
- 부수 효과의 합성: IO를 체이닝
- Lazy evaluation과의 관계
- **프론트엔드 실전**: DOM 조작 지연 실행, 로컬 스토리지 안전 접근, 분석 이벤트 배치 전송

**22주차: Task/Future Monad — 비동기의 함수형 처리**
- Task<A>: () → Promise<A> — 지연된 비동기
- Promise의 문제점: 즉시 실행, 참조 투명성 위반
- TaskEither: 비동기 + 에러를 한번에
- 취소(Cancellation)와 경쟁(Race)
- **프론트엔드 실전**: 취소 가능한 API 호출, 데이터 페칭 파이프라인, React Query와의 비교

---

### Phase 5: Monad 변환기와 효과 합성 (23-26주차)
> 여러 효과를 동시에 다루기

**23주차: Monad Transformer 기초**
- 문제: Reader<Either<Option<A>>> 같은 중첩
- Monad Transformer의 아이디어: 효과를 쌓기
- OptionT, EitherT 구현
- lift 연산: 안쪽 Monad를 바깥으로
- **프론트엔드 실전**: 인증 + 에러 + 비동기를 하나의 파이프라인으로

**24주차: ReaderTaskEither — 실전 스택**
- Reader + Task + Either 합성
- fp-ts의 ReaderTaskEither
- 의존성 주입 + 비동기 + 에러 처리를 한 번에
- 환경(Environment) 설계 패턴
- **프론트엔드 실전**: 완전한 API 서비스 레이어 구축, 테스트 가능한 비즈니스 로직

**25주차: Effect 시스템 입문**
- Monad Transformer의 한계: 성능, 복잡성
- Algebraic Effects 개념
- Effect-TS: 차세대 효과 시스템
- Effect<Requirements, Error, Value> 이해
- **프론트엔드 실전**: Effect-TS로 프로덕션 서비스 구축

**26주차: Effect 시스템 심화**
- Layer와 서비스 패턴
- 에러 채널과 결함(Defect) 구분
- Schedule, Queue, Ref 등 동시성 도구
- Effect의 테스트 전략
- **프론트엔드 실전**: 마이크로 프론트엔드 서비스 레이어, 실시간 데이터 스트림 처리

---

### Phase 6: Optics — 불변 데이터 정밀 조작 (27-30주차)
> 복잡한 중첩 구조를 우아하게 다루기

**27주차: Lens — 중첩 구조의 getter/setter**
- Lens의 정의: get + set
- **Lens 법칙**: GetPut, PutGet, PutPut
- Lens 합성: 깊은 중첩 접근
- TypeScript로 Lens 구현
- **프론트엔드 실전**: 깊은 Redux 상태 업데이트, 중첩 폼 데이터 조작

**28주차: Prism, Optional, Traversal**
- Prism: Sum 타입의 한 케이스에 초점
- Optional: 있을 수도 없을 수도 있는 값에 초점
- Traversal: 여러 값에 동시 초점
- Iso: 동형 사상(Isomorphism)
- **프론트엔드 실전**: 유니언 타입 안전 분기, 배열 내 특정 요소 업데이트, 데이터 변환 계층

**29주차: Optics 합성과 실전 패턴**
- Optics 합성 규칙: Lens + Prism = Optional
- monocle-ts / optics-ts 라이브러리 활용
- 불변 상태 관리에서의 Optics 패턴
- **프론트엔드 실전**: 복잡한 대시보드 상태 관리, 설정 에디터, 스프레드시트 셀 조작

**30주차: Optics와 카테고리 이론**
- Optics의 수학적 기초: Profunctor Optics
- Van Laarhoven Lens
- Optics는 왜 합성 가능한가
- **프론트엔드 실전**: 커스텀 Optics 설계, 도메인 특화 데이터 접근 레이어

---

### Phase 7: 카테고리 이론 기초 (31-35주차)
> 함수형 프로그래밍의 수학적 기반

**31주차: 카테고리의 정의**
- 카테고리: 대상(Object)과 사상(Morphism)
- 합성과 항등 사상
- 카테고리 법칙: 결합법칙, 항등법칙
- 프로그래밍에서의 카테고리: **Hask**, **TypeScript**
- **프론트엔드 실전**: 타입과 함수를 카테고리로 보기, 리팩토링의 수학적 근거

**32주차: Functor는 카테고리 간의 사상**
- Functor의 카테고리 이론적 정의
- Endofunctor: 같은 카테고리 내의 Functor
- Functor의 보존 성질
- 자연 변환(Natural Transformation): Functor 간의 사상
- **프론트엔드 실전**: Option → Either 변환, 에러 타입 변환 전략, 데이터 계층 어댑터

**33주차: Monad는 Endofunctor 카테고리의 Monoid**
- "A monad is just a monoid in the category of endofunctors"
- Monoidal Category 기초
- Monad의 카테고리 이론적 정의: η(unit) + μ(join)
- Kleisli 카테고리
- **프론트엔드 실전**: 이 추상화가 코드 설계에 주는 통찰

**34주차: 수반(Adjunction)과 Free 구조**
- 수반(Adjunction)의 정의
- 모든 Monad는 수반에서 온다
- Free Monad: DSL을 위한 도구
- Free Monad의 해석(Interpretation)
- **프론트엔드 실전**: UI 인터랙션 DSL, 테스트 가능한 부수 효과 기술

**35주차: Comonad와 쌍대성**
- 쌍대성(Duality) 원리
- Comonad: extract + extend
- Store Comonad와 Lens의 관계
- Env Comonad
- **프론트엔드 실전**: 컴포넌트를 Comonad로 모델링, 스프레드시트 계산 모델, UI 포커스 패턴

---

### Phase 8: 고급 패턴과 실전 아키텍처 (36-38주차)
> 이론을 프로덕션에 녹이기

**36주차: Tagless Final vs Free Monad**
- 프로그램을 '기술'하는 두 가지 방식
- Tagless Final: 타입클래스 기반 추상화
- Free Monad: 데이터로서의 프로그램
- 트레이드오프: 성능, 확장성, 사용성
- **프론트엔드 실전**: API 클라이언트 추상화, 테스트 더블 자동 생성

**37주차: 함수형 아키텍처 패턴**
- Onion Architecture / Hexagonal Architecture의 FP 버전
- Port & Adapter를 타입클래스로
- 순수 코어 + 불순 셸(Functional Core, Imperative Shell)
- 의존성 역전의 함수형 구현
- **프론트엔드 실전**: React 앱의 함수형 아키텍처, 비즈니스 로직 분리, 테스트 전략

**38주차: 함수형 상태 관리 심화**
- Redux를 넘어서: 함수형 상태 관리의 본질
- Algebraic Effects 기반 상태 관리
- 시간 여행 디버깅의 수학적 기초
- CQRS/Event Sourcing의 함수형 모델
- **프론트엔드 실전**: 대규모 앱 상태 설계, 실시간 협업 에디터 상태 모델

---

### Phase 9: 최종 프로젝트와 통합 (39-40주차)
> 모든 것을 하나로

**39주차: 종합 프로젝트 — 함수형 프론트엔드 앱 설계**
- 실전 프로젝트: 함수형 원칙으로 설계하는 대시보드 앱
- Effect 시스템 + Optics + ADT + 타입 안전 라우팅
- 아키텍처 결정 기록(ADR) 작성
- 테스트 전략: Property-based Testing

**40주차: 회고와 다음 단계**
- 전체 개념 지도(Concept Map) 정리
- 각 추상화 간의 관계 총정리
- Haskell / PureScript / Scala 생태계 탐색 가이드
- 논문 읽기 가이드: 함수형 프로그래밍 핵심 논문 목록
- 커뮤니티와 계속 학습하기

---

## 학습 진도표

| Phase | 주차 | 주제 | 상태 |
|-------|------|------|------|
| 1 | 1-4 | 함수형 사고의 기초 | 🟡 1주차 진행중 |
| 2 | 5-8 | 타입 시스템과 대수적 구조 | ⬜ |
| 3 | 9-14 | Functor와 그 친구들 | ⬜ |
| 4 | 15-22 | Monad 완전 정복 | ⬜ |
| 5 | 23-26 | Monad 변환기와 효과 합성 | ⬜ |
| 6 | 27-30 | Optics — 불변 데이터 정밀 조작 | ⬜ |
| 7 | 31-35 | 카테고리 이론 기초 | ⬜ |
| 8 | 36-38 | 고급 패턴과 실전 아키텍처 | ⬜ |
| 9 | 39-40 | 최종 프로젝트와 통합 | ⬜ |

## 학습 팁

1. **법칙을 항상 검증하세요** — 코드를 작성한 뒤 법칙 테스트를 함께 작성하면 이해가 깊어집니다
2. **타입이 가이드입니다** — 타입 시그니처만 보고 구현을 유추하는 연습을 하세요
3. **급하게 넘어가지 마세요** — Phase 3-4가 핵심입니다. Functor와 Monad를 체화할 때까지 반복하세요
4. **직접 구현하세요** — fp-ts를 쓰기 전에 직접 만들어보고, 그 다음에 라이브러리를 쓰세요
5. **Haskell을 두려워하지 마세요** — 개념의 순수한 형태를 볼 때 Haskell 코드가 큰 도움이 됩니다
