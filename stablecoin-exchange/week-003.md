# 3주차: 러스트 시작하기 — 설치와 첫 프로그램

## 왜 이것을 배우는가?

Phase 0에서 **왜 스테이블코인 거래소인가**를 이해했다. 이제 Phase 1, 즉 거래소의 심장인 **매칭 엔진을 만들 언어**, 러스트를 배우기 시작한다.

왜 러스트인가? 거래소 매칭 엔진은 **초당 수만 건의 주문을 처리**하면서 **단 한 건의 오류도 허용하지 않아야** 한다. C/C++는 빠르지만 메모리 버그가 치명적이고, Java/Go는 안전하지만 가비지 컬렉터(GC) 때문에 예측 불가능한 지연(latency spike)이 발생한다. 러스트는 **C++ 수준의 성능 + 컴파일 타임 메모리 안전성**을 동시에 제공하는 유일한 언어다.

이번 주에는 러스트를 설치하고, 기본 문법을 익히며, 마지막에 **간단한 주문 입력 CLI**를 만들어본다.

---

## 1. 러스트 설치와 개발 환경

### 1.1 rustup으로 러스트 설치

```
rustup: 러스트의 공식 설치/버전 관리 도구

설치 명령어 (Linux/macOS):
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

설치 과정:
  1. rustup 자체 설치
  2. 최신 안정 버전(stable) 러스트 컴파일러 설치
  3. cargo (패키지 매니저 겸 빌드 도구) 설치
  4. rustfmt (코드 포매터) 설치
  5. clippy (린터) 설치

설치 확인:
  $ rustc --version
  rustc 1.84.0 (9fc6b43 2025-01-07)

  $ cargo --version
  cargo 1.84.0 (fb7c438 2024-12-20)
```

> 💡 **핵심**: `rustup`은 Node.js의 `nvm`, Python의 `pyenv`와 같은 역할이다. 여러 버전의 러스트를 설치하고 전환할 수 있다.

### 1.2 rustup의 주요 명령어

```
버전 관리:
  rustup update              # 러스트 업데이트
  rustup default stable      # 안정 버전을 기본으로 설정
  rustup default nightly     # 야간 빌드를 기본으로 설정

도구 체인 관리:
  rustup toolchain list      # 설치된 도구 체인 목록
  rustup component add clippy    # clippy(린터) 설치
  rustup component add rustfmt   # rustfmt(포매터) 설치

타겟 추가 (크로스 컴파일):
  rustup target add wasm32-unknown-unknown   # WebAssembly 타겟
  rustup target list                          # 사용 가능한 타겟 목록
```

### 1.3 IDE 설정

```
추천 에디터: VS Code + rust-analyzer

설치 순서:
  1. VS Code 설치
  2. 확장 프로그램 "rust-analyzer" 설치
  3. (선택) "Even Better TOML" 확장 설치
  4. (선택) "CodeLLDB" 확장 설치 (디버깅용)

rust-analyzer가 제공하는 기능:
  ✅ 자동 완성 (IntelliSense)
  ✅ 인라인 타입 힌트
  ✅ 정의로 이동 (Go to Definition)
  ✅ 실시간 에러 표시
  ✅ 리팩토링 지원
  ✅ 매크로 확장 보기

→ Java의 IntelliJ, TypeScript의 tsserver에 해당하는 수준의 지원
```

### 1.4 러스트 도구 생태계 비교

```
러스트 도구          ↔    기존 스킬 대응

rustup               ↔    nvm (Node.js 버전 관리)
cargo                ↔    npm/yarn + webpack (패키지 관리 + 빌드)
Cargo.toml           ↔    package.json (프로젝트 설정)
crates.io            ↔    npmjs.com (패키지 레지스트리)
rustc                ↔    javac (컴파일러)
rustfmt              ↔    prettier (코드 포매터)
clippy               ↔    eslint (린터)
rust-analyzer        ↔    typescript language server
cargo test           ↔    jest/JUnit (테스트 러너)
cargo bench          ↔    JMH (벤치마크)
```

> 💡 **핵심**: 이미 익숙한 도구들과 1:1 대응된다. `cargo`가 npm의 역할(패키지 관리)과 webpack의 역할(빌드)을 모두 담당한다는 점이 차이다.

---

## 2. Hello World와 Cargo 프로젝트 구조

### 2.1 첫 번째 프로그램: Hello World

```rust
// main.rs
fn main() {
    println!("Hello, World!");
}
```

```
컴파일과 실행:
  $ rustc main.rs    # 컴파일 → main 실행파일 생성
  $ ./main           # 실행
  Hello, World!

→ rustc는 직접 잘 쓰지 않는다. cargo를 사용하는 것이 표준이다.
```

### 2.2 Cargo 프로젝트 생성

```
$ cargo new order-cli
     Creating binary (application) `order-cli` package

생성된 디렉토리 구조:
  order-cli/
  ├── Cargo.toml       # 프로젝트 메타데이터 + 의존성 (= package.json)
  ├── src/
  │   └── main.rs      # 진입점 (= index.js)
  └── .gitignore       # target/ 폴더 자동 제외
```

### 2.3 Cargo.toml 분석

```toml
[package]
name = "order-cli"           # 프로젝트 이름
version = "0.1.0"            # 버전 (SemVer)
edition = "2021"             # 러스트 에디션 (언어 버전과 다름)

[dependencies]
# 여기에 외부 크레이트(라이브러리) 추가
# 예: serde = "1.0"
```

```
package.json과 비교:

Cargo.toml                    package.json
──────────────────────────────────────────────────
[package]                     { "name": "...",
name = "order-cli"              "version": "0.1.0",
version = "0.1.0"             }

[dependencies]                "dependencies": {
serde = "1.0"                   "lodash": "^4.17"
                              }

edition = "2021"              (해당 없음 — 러스트 고유)
```

### 2.4 Cargo 주요 명령어

```
빌드 & 실행:
  cargo build          # 디버그 빌드 (target/debug/)
  cargo build --release  # 릴리스 빌드 (최적화, target/release/)
  cargo run            # 빌드 + 실행 (가장 자주 사용)
  cargo run --release  # 릴리스 모드로 빌드 + 실행

코드 품질:
  cargo check          # 컴파일 에러 확인 (빌드보다 빠름)
  cargo clippy         # 린트 검사 (더 엄격한 경고)
  cargo fmt            # 코드 자동 포매팅

테스트:
  cargo test           # 모든 테스트 실행
  cargo test 이름      # 특정 테스트만 실행

문서:
  cargo doc --open     # API 문서 생성 후 브라우저에서 열기
```

```
cargo run의 동작:

  $ cargo run
     Compiling order-cli v0.1.0 (/path/to/order-cli)
      Finished `dev` profile [unoptimized + debuginfo] target(s)
       Running `target/debug/order-cli`
  Hello, World!

→ 소스 코드 변경이 없으면 재컴파일하지 않음 (증분 빌드)
→ npm start + webpack --watch와 유사한 개념
```

### 2.5 디버그 빌드 vs 릴리스 빌드

```
                 디버그 (cargo build)    릴리스 (cargo build --release)
──────────────────────────────────────────────────────────────────────
최적화 수준      없음 (opt-level = 0)    최대 (opt-level = 3)
컴파일 속도      빠름                    느림 (2~5배)
실행 속도        느림                    빠름 (10~100배 차이 가능)
디버그 정보      포함                    제거
바이너리 크기    큼                      작음
용도             개발 중                 배포/벤치마크

→ 거래소 매칭 엔진은 반드시 --release로 빌드해야 한다
→ 개발 중에는 컴파일 속도를 위해 기본(디버그) 빌드 사용
```

---

## 3. 변수와 상수

### 3.1 변수 선언: let

```rust
fn main() {
    // 기본 변수는 불변(immutable)
    let price = 100;
    // price = 200;  // ❌ 컴파일 에러! 불변 변수에 재할당 불가

    // 가변 변수는 mut 키워드 사용
    let mut quantity = 10;
    quantity = 20;  // ✅ OK

    println!("가격: {}, 수량: {}", price, quantity);
}
```

```
JavaScript/Java와 비교:

러스트                        JavaScript              Java
──────────────────────────────────────────────────────────────
let x = 5;                   const x = 5;            final int x = 5;
let mut x = 5;               let x = 5;              int x = 5;
(재선언 불가)                 var x = 5;              (해당 없음)

→ 러스트의 기본값은 "불변"이다!
→ JavaScript의 const가 러스트의 let에 해당
→ 변경이 필요할 때만 명시적으로 mut을 붙인다
```

> 💡 **왜 기본이 불변인가?** 거래소 매칭 엔진에서 주문 가격이 의도치 않게 변경되면 치명적이다. 러스트는 "변경할 일이 있을 때만 변경 가능하게 만들어라"라는 철학을 가지고 있다. 컴파일러가 의도치 않은 변경을 원천 차단한다.

### 3.2 변수 쉐도잉(Shadowing)

```rust
fn main() {
    let price = 100;          // 정수
    let price = price * 2;    // 같은 이름으로 재선언 (쉐도잉)
    let price = format!("{}원", price);  // 타입도 변경 가능!

    println!("{}", price);  // "200원"
}
```

```
쉐도잉 vs mut:

쉐도잉 (let price = ...):
  ✅ 새로운 변수를 같은 이름으로 생성
  ✅ 타입 변경 가능
  ✅ 이전 값은 더 이상 접근 불가
  → 변환(transform) 과정을 표현할 때 유용

mut (let mut price = ...):
  ✅ 같은 변수의 값만 변경
  ❌ 타입 변경 불가
  → 상태가 변하는 경우에 사용

거래소에서의 활용:
  let raw_price = "100.50";           // 문자열로 받은 가격
  let raw_price = raw_price.trim();   // 공백 제거 (쉐도잉)
  let price: f64 = raw_price.parse().unwrap();  // 숫자로 변환 (쉐도잉)
  → 각 단계마다 같은 이름을 재활용하여 가독성 향상
```

### 3.3 상수(const)

```rust
// 상수: 컴파일 타임에 결정되는 값
const MAX_ORDER_QUANTITY: u64 = 1_000_000;       // 최대 주문 수량
const MIN_PRICE_INCREMENT: f64 = 0.01;           // 최소 가격 단위
const EXCHANGE_NAME: &str = "StableCoin Exchange";

fn main() {
    println!("거래소: {}", EXCHANGE_NAME);
    println!("최대 주문 수량: {}", MAX_ORDER_QUANTITY);
}
```

```
let vs const:

let (변수):
  → 런타임에 결정
  → 함수 내부에서만 선언
  → 타입 추론 가능 (생략 가능)

const (상수):
  → 컴파일 타임에 결정
  → 어디서든 선언 가능 (함수 밖도 OK)
  → 타입 명시 필수
  → 이름은 SCREAMING_SNAKE_CASE 관례
  → 함수 호출 결과를 넣을 수 없음

거래소에서의 활용:
  const FEE_RATE: f64 = 0.001;        // 수수료율 0.1%
  const MAX_OPEN_ORDERS: u32 = 200;   // 계정당 최대 미체결 주문
  const TICK_SIZE: f64 = 0.01;        // 호가 단위
```

---

## 4. 데이터 타입

### 4.1 스칼라 타입

```rust
fn main() {
    // ── 정수형 ──
    let amount: i32 = -100;         // 부호 있는 32비트 정수
    let quantity: u64 = 1000;       // 부호 없는 64비트 정수 (수량은 음수 불가)
    let tiny: u8 = 255;            // 0~255 (1바이트)

    // ── 부동소수점 ──
    let price: f64 = 1.0025;       // 64비트 부동소수점 (기본)
    let rate: f32 = 0.001;         // 32비트 부동소수점

    // ── 불리언 ──
    let is_buy: bool = true;       // 매수 여부
    let is_filled: bool = false;   // 체결 여부

    // ── 문자 ──
    let symbol: char = '₿';       // 유니코드 문자 (4바이트)
}
```

```
러스트 정수형 전체 목록:

부호 있음 (signed):  i8, i16, i32, i64, i128, isize
부호 없음 (unsigned): u8, u16, u32, u64, u128, usize

비트 수   부호 있음 범위                부호 없음 범위
────────────────────────────────────────────────────
8         -128 ~ 127                  0 ~ 255
16        -32,768 ~ 32,767            0 ~ 65,535
32        -2.1B ~ 2.1B                0 ~ 4.2B
64        -9.2E18 ~ 9.2E18            0 ~ 1.8E19
128       (매우 큰 범위)              (매우 큰 범위)
size      플랫폼 의존 (32 또는 64)     플랫폼 의존

거래소에서의 선택:
  주문 ID    → u64 (충분히 크고, 음수 불필요)
  가격       → f64 또는 i64 (정수로 소수점 처리 — 후에 다룸)
  수량       → u64 (음수 수량은 없음)
  타임스탬프 → u64 또는 i64
```

> 💡 **중요**: 실제 거래소에서는 **부동소수점(f64)으로 가격을 다루지 않는다**. `0.1 + 0.2 ≠ 0.3` 문제 때문이다. 대신 정수 기반 고정소수점(예: 가격 × 10^8)을 사용한다. 이 부분은 매칭 엔진 구현(23주차~) 때 다시 다룬다. 지금은 학습 편의를 위해 f64를 사용한다.

### 4.2 정수 리터럴 표기법

```rust
fn main() {
    // 숫자 구분자 (가독성)
    let million = 1_000_000;       // 백만
    let max_qty = 100_000_u64;     // 타입 접미사

    // 다양한 진법
    let hex = 0xFF;                // 16진수 (255)
    let octal = 0o77;             // 8진수 (63)
    let binary = 0b1111_0000;     // 2진수 (240)
    let byte = b'A';              // 바이트 (u8, 65)

    println!("million: {}", million);
    println!("hex: {}", hex);
    println!("binary: {}", binary);
}
```

### 4.3 복합 타입: 튜플과 배열

```rust
fn main() {
    // ── 튜플: 서로 다른 타입의 값을 묶음 ──
    let order: (u64, f64, bool) = (1001, 50_000.0, true);
    //         주문ID  가격    매수여부

    // 구조분해 (destructuring)
    let (id, price, is_buy) = order;
    println!("주문 #{}: {}원 {}", id, price, if is_buy { "매수" } else { "매도" });

    // 인덱스 접근
    let order_id = order.0;    // 1001
    let order_price = order.1; // 50_000.0

    // ── 배열: 같은 타입, 고정 크기 ──
    let prices: [f64; 5] = [100.0, 101.5, 99.8, 102.3, 100.7];
    let first = prices[0];    // 100.0
    let len = prices.len();   // 5

    // 같은 값으로 초기화
    let zeros = [0.0_f64; 10];  // [0.0, 0.0, ..., 0.0] (10개)

    println!("첫 번째 가격: {}", first);
    println!("가격 수: {}", len);
}
```

```
튜플 vs 배열:

튜플 (a, b, c):
  → 서로 다른 타입 가능
  → 크기 고정 (컴파일 타임)
  → 구조분해로 각 요소에 이름 부여
  → 용도: 함수에서 여러 값 반환

배열 [T; N]:
  → 모든 요소 같은 타입
  → 크기 고정 (컴파일 타임)
  → 스택에 할당 (빠름)
  → 용도: 크기가 고정된 데이터 집합

벡터 Vec<T> (나중에 배움):
  → 모든 요소 같은 타입
  → 크기 가변 (런타임)
  → 힙에 할당
  → JavaScript의 Array, Java의 ArrayList에 해당
```

### 4.4 문자열: String vs &str

```rust
fn main() {
    // &str — 문자열 슬라이스 (불변 참조, 스택)
    let symbol: &str = "BTC/USDC";      // 문자열 리터럴은 &str
    let exchange: &str = "StableCoin Exchange";

    // String — 소유된 문자열 (가변, 힙)
    let mut greeting = String::from("Hello");
    greeting.push_str(", Trader!");       // 문자열 추가 가능
    println!("{}", greeting);             // "Hello, Trader!"

    // &str → String 변환
    let owned: String = symbol.to_string();
    let also_owned: String = String::from("ETH/USDC");

    // String → &str 변환 (자동 역참조)
    let borrowed: &str = &owned;

    // 포매팅으로 String 생성
    let message = format!("{}에서 {} 거래", exchange, symbol);
    println!("{}", message);
}
```

```
&str vs String:

&str (문자열 슬라이스):
  → 불변 참조 (읽기 전용)
  → 데이터를 소유하지 않음
  → 크기 고정
  → JavaScript의 const str에 가까움
  → 함수 매개변수로 주로 사용

String (소유된 문자열):
  → 힙에 할당, 소유됨
  → 크기 변경 가능 (push, pop)
  → JavaScript의 let str에 가까움
  → 데이터를 저장할 때 주로 사용

언제 어떤 것을 쓰는가:
  함수 매개변수 → &str (빌려서 읽기)
  구조체 필드   → String (소유해서 저장)
  리터럴 값     → &str (자동)
  동적 생성     → String (format!, push_str 등)
```

> 💡 **핵심**: `&str`과 `String`의 관계는 러스트의 **소유권(Ownership)** 개념과 밀접하다. 4주차에서 소유권을 깊이 다룬다. 지금은 "리터럴은 `&str`, 동적으로 만드는 건 `String`" 정도로 이해하면 된다.

---

## 5. 함수

### 5.1 함수 기본

```rust
// 함수 정의
fn greet(name: &str) {
    println!("안녕하세요, {}님!", name);
}

// 반환값이 있는 함수
fn add(a: i32, b: i32) -> i32 {
    a + b    // 세미콜론 없음 = 이 표현식의 값을 반환
}

// return 키워드로 명시적 반환 (조기 반환 시 사용)
fn calculate_fee(amount: f64, rate: f64) -> f64 {
    if amount <= 0.0 {
        return 0.0;  // 조기 반환
    }
    amount * rate    // 마지막 표현식 = 반환값
}

fn main() {
    greet("트레이더");
    let sum = add(10, 20);
    let fee = calculate_fee(1000.0, 0.001);
    println!("합: {}, 수수료: {}", sum, fee);
}
```

```
러스트 함수의 특징:

1. 매개변수 타입은 반드시 명시
   fn add(a: i32, b: i32) -> i32
   → TypeScript: function add(a: number, b: number): number

2. 반환값은 마지막 표현식 (세미콜론 없이)
   fn double(x: i32) -> i32 { x * 2 }
   → 세미콜론을 붙이면 () (유닛 타입)을 반환 → 컴파일 에러

3. 반환 타입이 없으면 () (유닛 타입) 반환
   fn print_hello() { println!("hello"); }
   → TypeScript의 void에 해당

4. snake_case 관례
   fn calculate_total_price()  ✅
   fn calculateTotalPrice()    ❌ (컴파일은 되지만 경고)
```

### 5.2 표현식 vs 문장

```rust
fn main() {
    // 문장(Statement): 값을 반환하지 않음
    let x = 5;          // let 바인딩은 문장
    // let y = (let z = 10);  // ❌ 에러! 문장은 값이 없음

    // 표현식(Expression): 값을 반환함
    let y = {
        let temp = 10;
        temp + 1         // 세미콜론 없음 → 이 블록의 값은 11
    };

    // if도 표현식이다
    let side = if true { "매수" } else { "매도" };

    println!("x={}, y={}, side={}", x, y, side);
}
```

```
표현식 기반 언어의 장점:

JavaScript (문장 기반):
  let fee;
  if (amount > 1000) {
      fee = amount * 0.0005;
  } else {
      fee = amount * 0.001;
  }

러스트 (표현식 기반):
  let fee = if amount > 1000.0 {
      amount * 0.0005
  } else {
      amount * 0.001
  };

→ 변수 선언과 할당이 한 번에 이루어짐
→ mut가 불필요 (불변으로 충분)
→ 블록 {}도 값을 가지므로 복잡한 초기화도 깔끔하게 처리
```

### 5.3 거래소 맥락의 함수 예제

```rust
/// 주문 유형을 문자열로 반환
fn order_side_label(is_buy: bool) -> &'static str {
    if is_buy { "BID (매수)" } else { "ASK (매도)" }
}

/// 수수료 계산 (등급에 따른 차등 수수료)
fn calculate_fee(trade_amount: f64, is_maker: bool) -> f64 {
    let rate = if is_maker {
        0.001   // Maker: 0.1%
    } else {
        0.002   // Taker: 0.2%
    };
    trade_amount * rate
}

/// 주문 검증
fn validate_order(price: f64, quantity: u64) -> bool {
    if price <= 0.0 {
        println!("에러: 가격은 0보다 커야 합니다");
        return false;
    }
    if quantity == 0 {
        println!("에러: 수량은 0보다 커야 합니다");
        return false;
    }
    true
}

fn main() {
    let price = 50_000.0;
    let quantity = 10_u64;
    let is_buy = true;
    let is_maker = true;

    if validate_order(price, quantity) {
        let total = price * quantity as f64;
        let fee = calculate_fee(total, is_maker);
        println!("{}", order_side_label(is_buy));
        println!("주문 금액: {}원", total);
        println!("수수료: {}원 ({})", fee, if is_maker { "Maker" } else { "Taker" });
        println!("총 비용: {}원", total + fee);
    }
}
```

```
실행 결과:
  BID (매수)
  주문 금액: 500000원
  수수료: 500원 (Maker)
  총 비용: 500500원
```

---

## 6. 제어 흐름

### 6.1 if / else if / else

```rust
fn classify_order_size(quantity: u64) -> &'static str {
    if quantity >= 100_000 {
        "기관 (Institutional)"
    } else if quantity >= 10_000 {
        "대량 (Large)"
    } else if quantity >= 1_000 {
        "중간 (Medium)"
    } else {
        "소량 (Small)"
    }
}

fn main() {
    let qty = 50_000;
    let classification = classify_order_size(qty);
    println!("주문 {}개 → {}", qty, classification);
    // 주문 50000개 → 대량 (Large)
}
```

### 6.2 loop — 무한 루프

```rust
fn main() {
    let mut count = 0;

    // loop: 명시적으로 break할 때까지 반복
    let result = loop {
        count += 1;
        if count == 10 {
            break count * 2;  // loop도 값을 반환할 수 있다!
        }
    };

    println!("result: {}", result);  // 20
}
```

### 6.3 while — 조건부 루프

```rust
fn main() {
    let mut remaining_orders = 5;

    while remaining_orders > 0 {
        println!("미체결 주문 {} 건 처리 중...", remaining_orders);
        remaining_orders -= 1;
    }

    println!("모든 주문 처리 완료!");
}
```

### 6.4 for — 반복자 순회

```rust
fn main() {
    // 범위(Range) 순회
    for i in 1..=5 {
        println!("주문 #{}", i);
    }
    // 1..5  → 1, 2, 3, 4 (마지막 미포함)
    // 1..=5 → 1, 2, 3, 4, 5 (마지막 포함)

    // 배열 순회
    let prices = [100.0, 101.5, 99.8, 102.3, 100.7];
    for price in &prices {
        println!("가격: {:.1}", price);
    }

    // 인덱스와 함께 순회
    for (i, price) in prices.iter().enumerate() {
        println!("[{}] {:.1}", i, price);
    }

    // 역순 순회
    for i in (1..=5).rev() {
        println!("카운트다운: {}", i);
    }
}
```

```
JavaScript와 비교:

JavaScript:                          러스트:
for (let i = 0; i < 5; i++)         for i in 0..5
for (const item of array)           for item in &array
array.forEach((item, i) => ...)     for (i, item) in array.iter().enumerate()
for (let i = 4; i >= 0; i--)        for i in (0..5).rev()

→ 러스트의 for는 항상 반복자(Iterator)를 사용
→ C 스타일 for (init; cond; step)는 없음
```

### 6.5 match — 패턴 매칭

```rust
fn main() {
    let order_type = "limit";

    // match: 값에 따른 분기 (switch보다 강력)
    let description = match order_type {
        "limit" => "지정가 주문",
        "market" => "시장가 주문",
        "stop" => "스탑 주문",
        "stop_limit" => "스탑 리밋 주문",
        _ => "알 수 없는 주문 유형",  // _ 는 default와 같음
    };

    println!("{}: {}", order_type, description);

    // 숫자에도 사용 가능
    let error_code: u32 = 404;
    let message = match error_code {
        200 => "OK",
        400 => "잘못된 요청",
        401 => "인증 필요",
        403 => "접근 거부",
        404 => "주문을 찾을 수 없음",
        500..=599 => "서버 에러",  // 범위 패턴
        _ => "알 수 없는 에러",
    };
    println!("에러 {}: {}", error_code, message);
}
```

```
match vs switch:

Java switch:                         러스트 match:
  switch (x) {                        match x {
      case 1: ... ; break;                1 => ...,
      case 2: ... ; break;                2 => ...,
      default: ...;                       _ => ...,
  }                                   }

차이점:
  ✅ break 불필요 (폴스루 없음)
  ✅ 모든 경우를 처리해야 함 (exhaustive) — 빠뜨리면 컴파일 에러
  ✅ 표현식이므로 값 반환 가능
  ✅ 범위, 튜플, 구조체 등 복잡한 패턴 매칭 가능
  → 6주차에서 enum과 함께 더 깊이 다룸
```

### 6.6 제어 흐름 종합 예제

```rust
fn process_price_levels(prices: &[f64], reference_price: f64) {
    println!("기준가: {:.2}", reference_price);
    println!("─────────────────────────────");

    for (i, &price) in prices.iter().enumerate() {
        let change_pct = (price - reference_price) / reference_price * 100.0;

        let status = if change_pct > 5.0 {
            "급등"
        } else if change_pct > 0.0 {
            "상승"
        } else if change_pct == 0.0 {
            "보합"
        } else if change_pct > -5.0 {
            "하락"
        } else {
            "급락"
        };

        let bar = match change_pct as i32 {
            6..=i32::MAX => "▲▲▲",
            1..=5 => "▲",
            0 => "─",
            -5..=-1 => "▼",
            _ => "▼▼▼",
        };

        println!(
            "[{:2}] {:>10.2} | {:>+6.2}% | {} {}",
            i, price, change_pct, bar, status
        );
    }
}

fn main() {
    let prices = [1.0025, 1.0050, 0.9980, 1.0000, 1.0100, 0.9500, 1.0800];
    process_price_levels(&prices, 1.0000);
}
```

```
실행 결과:
  기준가: 1.00
  ─────────────────────────────
  [ 0]       1.00 |  +0.25% | ▲ 상승
  [ 1]       1.01 |  +0.50% | ▲ 상승
  [ 2]       1.00 |  -0.20% | ─ 보합
  [ 3]       1.00 |  +0.00% | ─ 보합
  [ 4]       1.01 |  +1.00% | ▲ 상승
  [ 5]       0.95 |  -5.00% | ▼▼▼ 급락
  [ 6]       1.08 |  +8.00% | ▲▲▲ 급등
```

---

## 7. 표준 입력과 println! 매크로

### 7.1 println! 포매팅

```rust
fn main() {
    let name = "USDC/KRW";
    let price = 1350.5678_f64;
    let quantity = 1000_u64;

    // 기본 출력
    println!("거래쌍: {}", name);

    // 소수점 자릿수 지정
    println!("가격: {:.2}", price);           // 1350.57
    println!("가격: {:.4}", price);           // 1350.5678

    // 정렬과 패딩
    println!("{:<15} | {:>10.2}", name, price);     // 왼쪽 정렬 | 오른쪽 정렬
    println!("{:>15} | {:>10.2}", name, price);     // 오른쪽 정렬

    // 0 채우기
    println!("주문번호: {:06}", 42);          // 000042

    // 이진수, 8진수, 16진수
    println!("이진: {:b}", 255);             // 11111111
    println!("16진: {:x}", 255);             // ff
    println!("16진(대문자): {:X}", 255);      // FF

    // 디버그 출력 ({:?})
    let order = (1001_u64, 50_000.0_f64, true);
    println!("주문: {:?}", order);            // (1001, 50000.0, true)

    // 여러 줄 디버그 ({:#?})
    println!("주문 상세:\n{:#?}", order);
}
```

### 7.2 표준 입력 받기

```rust
use std::io;

fn main() {
    println!("거래쌍을 입력하세요:");

    let mut input = String::new();
    io::stdin()
        .read_line(&mut input)
        .expect("입력을 읽을 수 없습니다");

    let input = input.trim();  // 개행 문자 제거 (쉐도잉)
    println!("선택한 거래쌍: {}", input);
}
```

```
입력 받기 과정 분석:

let mut input = String::new();
  → 빈 String 생성 (가변이어야 데이터를 넣을 수 있음)

io::stdin().read_line(&mut input)
  → 표준 입력에서 한 줄 읽기
  → &mut input: input의 가변 참조를 전달 (빌림)
  → 반환값: Result<usize, io::Error>

.expect("에러 메시지")
  → Result가 Ok이면 값 반환
  → Result가 Err이면 프로그램 종료 + 메시지 출력
  → 7주차에서 에러 처리를 정식으로 다룸

let input = input.trim();
  → 쉐도잉으로 같은 이름 재사용
  → trim()은 앞뒤 공백과 개행(\n) 제거
```

### 7.3 문자열을 숫자로 변환

```rust
use std::io;

fn read_number(prompt: &str) -> f64 {
    println!("{}", prompt);
    let mut input = String::new();
    io::stdin()
        .read_line(&mut input)
        .expect("입력을 읽을 수 없습니다");

    input.trim().parse::<f64>().expect("유효한 숫자를 입력하세요")
}

fn main() {
    let price = read_number("가격을 입력하세요:");
    let quantity = read_number("수량을 입력하세요:");

    println!("주문 금액: {:.2}원", price * quantity);
}
```

```
parse() 메서드:

"100".parse::<i32>()       → Ok(100)
"3.14".parse::<f64>()      → Ok(3.14)
"hello".parse::<i32>()     → Err(ParseIntError)
"".parse::<f64>()          → Err(ParseFloatError)

::<타입>은 "터보피시(turbofish)" 문법이라 부른다
→ 어떤 타입으로 변환할지 명시

대안: 변수의 타입으로 추론
  let x: i32 = "100".parse().expect("...");
  → 타입 어노테이션으로 parse가 i32를 반환할 것을 추론
```

---

## 8. 실전 프로젝트: 간단한 주문 입력 CLI

### 8.1 프로젝트 생성

```
$ cargo new order-cli
$ cd order-cli
```

### 8.2 전체 코드

```rust
// src/main.rs
use std::io;
use std::io::Write;

// ── 상수 정의 ──
const EXCHANGE_NAME: &str = "StableCoin Exchange";
const MAKER_FEE_RATE: f64 = 0.001;  // 0.1%
const TAKER_FEE_RATE: f64 = 0.002;  // 0.2%
const SUPPORTED_PAIRS: [&str; 3] = ["USDC/KRW", "USDT/KRW", "BTC/USDC"];

// ── 헬퍼 함수: 프롬프트 출력 후 입력 받기 ──
fn prompt(message: &str) -> String {
    print!("{}", message);
    io::stdout().flush().expect("출력 실패");
    let mut input = String::new();
    io::stdin()
        .read_line(&mut input)
        .expect("입력을 읽을 수 없습니다");
    input.trim().to_string()
}

// ── 거래쌍 유효성 검증 ──
fn is_valid_pair(pair: &str) -> bool {
    SUPPORTED_PAIRS.iter().any(|&p| p == pair)
}

// ── 수수료 계산 ──
fn calculate_fee(amount: f64, is_maker: bool) -> f64 {
    let rate = if is_maker {
        MAKER_FEE_RATE
    } else {
        TAKER_FEE_RATE
    };
    amount * rate
}

// ── 주문 정보 출력 ──
fn display_order(
    order_id: u64,
    pair: &str,
    side: &str,
    price: f64,
    quantity: f64,
    is_maker: bool,
) {
    let total = price * quantity;
    let fee = calculate_fee(total, is_maker);
    let fee_type = if is_maker { "Maker" } else { "Taker" };

    println!();
    println!("╔══════════════════════════════════════╗");
    println!("║         주문 확인                     ║");
    println!("╠══════════════════════════════════════╣");
    println!("║ 주문 ID  : {:>24} ║", order_id);
    println!("║ 거래쌍   : {:>24} ║", pair);
    println!("║ 주문 유형: {:>24} ║", side);
    println!("║ 가격     : {:>20.2} 원 ║", price);
    println!("║ 수량     : {:>24.4} ║", quantity);
    println!("║──────────────────────────────────────║");
    println!("║ 주문 금액: {:>20.2} 원 ║", total);
    println!("║ 수수료   : {:>15.2} 원 ({}) ║", fee, fee_type);
    println!("║ 총 비용  : {:>20.2} 원 ║", total + fee);
    println!("╚══════════════════════════════════════╝");
}

// ── 메인 루프 ──
fn main() {
    println!("========================================");
    println!("  {} v0.1.0", EXCHANGE_NAME);
    println!("========================================");
    println!();

    // 지원하는 거래쌍 출력
    println!("지원 거래쌍:");
    for (i, pair) in SUPPORTED_PAIRS.iter().enumerate() {
        println!("  {}. {}", i + 1, pair);
    }
    println!();

    let mut order_count: u64 = 0;

    loop {
        println!("─────────────────────────────────────");
        let command = prompt("명령 입력 (new/list/quit): ");

        match command.as_str() {
            "new" | "n" => {
                // 1. 거래쌍 선택
                let pair = prompt("거래쌍 (예: USDC/KRW): ");
                if !is_valid_pair(&pair) {
                    println!("지원하지 않는 거래쌍입니다: {}", pair);
                    continue;
                }

                // 2. 매수/매도 선택
                let side_input = prompt("주문 유형 (buy/sell): ");
                let side = match side_input.as_str() {
                    "buy" | "b" => "매수 (BID)",
                    "sell" | "s" => "매도 (ASK)",
                    _ => {
                        println!("유효하지 않은 주문 유형입니다");
                        continue;
                    }
                };

                // 3. 가격 입력
                let price_str = prompt("가격: ");
                let price: f64 = match price_str.parse() {
                    Ok(p) if p > 0.0 => p,
                    Ok(_) => {
                        println!("가격은 0보다 커야 합니다");
                        continue;
                    }
                    Err(_) => {
                        println!("유효한 숫자를 입력하세요");
                        continue;
                    }
                };

                // 4. 수량 입력
                let qty_str = prompt("수량: ");
                let quantity: f64 = match qty_str.parse() {
                    Ok(q) if q > 0.0 => q,
                    Ok(_) => {
                        println!("수량은 0보다 커야 합니다");
                        continue;
                    }
                    Err(_) => {
                        println!("유효한 숫자를 입력하세요");
                        continue;
                    }
                };

                // 5. 주문 생성 및 출력
                order_count += 1;
                let is_maker = true; // 지정가 주문은 Maker로 간주
                display_order(order_count, &pair, side, price, quantity, is_maker);
            }

            "list" | "l" => {
                println!("총 {} 건의 주문이 접수되었습니다.", order_count);
                println!("(주문 저장 기능은 추후 구현 예정)");
            }

            "quit" | "q" => {
                println!("{}을 종료합니다. 감사합니다!", EXCHANGE_NAME);
                break;
            }

            _ => {
                println!("알 수 없는 명령입니다. new, list, quit 중 선택하세요.");
            }
        }
    }
}
```

### 8.3 실행 예시

```
$ cargo run

========================================
  StableCoin Exchange v0.1.0
========================================

지원 거래쌍:
  1. USDC/KRW
  2. USDT/KRW
  3. BTC/USDC

─────────────────────────────────────
명령 입력 (new/list/quit): new
거래쌍 (예: USDC/KRW): USDC/KRW
주문 유형 (buy/sell): buy
가격: 1350.50
수량: 100

╔══════════════════════════════════════╗
║         주문 확인                     ║
╠══════════════════════════════════════╣
║ 주문 ID  :                        1 ║
║ 거래쌍   :                 USDC/KRW ║
║ 주문 유형:               매수 (BID) ║
║ 가격     :            1350.50 원 ║
║ 수량     :                 100.0000 ║
║──────────────────────────────────────║
║ 주문 금액:          135050.00 원 ║
║ 수수료   :       135.05 원 (Maker) ║
║ 총 비용  :          135185.05 원 ║
╚══════════════════════════════════════╝

─────────────────────────────────────
명령 입력 (new/list/quit): quit
StableCoin Exchange을 종료합니다. 감사합니다!
```

### 8.4 코드에서 배운 개념 정리

```
이 프로젝트에서 사용한 러스트 개념:

1. 상수 (const)
   → EXCHANGE_NAME, MAKER_FEE_RATE, SUPPORTED_PAIRS

2. 변수와 가변성 (let, let mut)
   → let pair (불변), let mut order_count (가변)

3. 함수 (fn)
   → prompt(), is_valid_pair(), calculate_fee(), display_order()
   → 매개변수 타입 명시, 반환 타입 명시

4. 데이터 타입
   → u64 (주문 카운트), f64 (가격/수량), &str (문자열 참조), String (소유 문자열)
   → bool (is_maker), 배열 [&str; 3]

5. 제어 흐름
   → loop (메인 루프), match (명령 분기), if/else (조건 분기)
   → continue (루프 다시), break (루프 종료)

6. 표준 입출력
   → io::stdin().read_line(), println!(), print!(), io::stdout().flush()

7. 문자열 처리
   → .trim(), .parse::<f64>(), .to_string(), .as_str()
   → format! 매크로, 포매팅 ({:.2}, {:>20})

8. 반복자
   → .iter(), .enumerate(), .any()

9. 에러 처리 (기초)
   → .expect() (패닉), match로 Result 처리
```

### 8.5 다음 단계 확장 포인트

```
이 CLI를 확장하면 거래소 매칭 엔진이 된다:

현재 (3주차):
  ✅ 주문 입력
  ✅ 입력 검증
  ✅ 수수료 계산
  ✅ 주문 정보 출력

향후 추가할 기능:
  Week 4-5: 소유권을 이용한 안전한 주문 데이터 전달
  Week 6:   Order 구조체와 OrderBook 구조체 정의
  Week 7:   Vec<Order>로 주문 저장, HashMap으로 가격 레벨 관리
  Week 8:   트레이트로 다양한 주문 유형 추상화
  Week 14:  완전한 CLI 기반 오더 매칭 엔진
  Week 23+: 실제 매칭 알고리즘 구현
```

---

## 9. 러스트가 컴파일 타임에 잡아주는 버그들

### 9.1 거래소에서 치명적인 버그 시나리오

```rust
// ── 시나리오 1: 의도치 않은 변수 변경 ──

// JavaScript에서 발생할 수 있는 버그:
//   let price = 50000;
//   // ... 100줄 뒤 ...
//   price = -1;  // 실수로 음수 할당 → 런타임에야 발견
//   // 고객의 돈이 사라짐!

// 러스트에서:
fn scenario_1() {
    let price = 50_000;
    // price = -1;  // ❌ 컴파일 에러: "cannot assign twice to immutable variable"
    // → 코드가 실행되기 전에 잡힘
    println!("{}", price);
}
```

```rust
// ── 시나리오 2: 정수 오버플로우 ──

// C/C++에서:
//   uint32_t balance = 4294967295;  // 최대값
//   balance += 1;  // 0이 됨! → 잔액이 사라짐 (Silent Overflow)

// 러스트에서 (디버그 모드):
fn scenario_2() {
    let balance: u32 = u32::MAX;  // 4294967295
    // let new_balance = balance + 1;
    // → 디버그 모드: 패닉 (프로그램 종료)
    // → 릴리스 모드: 래핑 (0이 됨) — 명시적 처리 필요

    // 안전한 처리:
    let new_balance = balance.checked_add(1);  // → None (오버플로우 감지)
    match new_balance {
        Some(val) => println!("새 잔액: {}", val),
        None => println!("오버플로우! 잔액 변경 거부"),
    }
}
```

```rust
// ── 시나리오 3: 배열 범위 초과 ──

// C에서:
//   int prices[5] = {100, 200, 300, 400, 500};
//   int x = prices[10];  // 쓰레기 값 읽힘 (Buffer Overflow)
//   → 보안 취약점의 70%가 이런 메모리 버그

// 러스트에서:
fn scenario_3() {
    let prices = [100, 200, 300, 400, 500];
    // let x = prices[10];
    // → 컴파일 타임: 상수 인덱스면 컴파일 에러
    // → 런타임: 변수 인덱스면 패닉 (절대 쓰레기 값 읽지 않음)

    // 안전한 접근:
    let safe = prices.get(10);  // → None (패닉 없이 안전하게 처리)
    match safe {
        Some(val) => println!("가격: {}", val),
        None => println!("해당 인덱스에 가격 없음"),
    }
}

fn main() {
    scenario_1();
    scenario_2();
    scenario_3();
}
```

```
러스트 컴파일러가 잡아주는 버그 카테고리:

안전성 관련:
  ✅ 불변 변수에 재할당 시도
  ✅ 정수 오버플로우 (디버그 모드)
  ✅ 배열 범위 초과 접근
  ✅ null 참조 (러스트에는 null이 없음 — Option 사용)
  ✅ 댕글링 포인터 (이미 해제된 메모리 접근)
  ✅ 데이터 레이스 (동시에 같은 데이터 수정)
  ✅ Use-after-free (해제 후 사용)

코드 품질 관련 (clippy):
  ✅ 사용하지 않는 변수/함수
  ✅ 불필요한 clone
  ✅ 더 나은 패턴 제안
  ✅ 잠재적 성능 문제

→ Microsoft: "우리 보안 취약점의 70%가 메모리 안전성 문제"
→ Google Chrome: "심각한 보안 버그의 70%가 메모리 관련"
→ 러스트: 이 70%를 컴파일 타임에 원천 차단
```

---

## 10. 러스트 vs 다른 언어 — 거래소 관점

### 10.1 성능 비교

```
매칭 엔진에서 중요한 메트릭: 지연 시간(Latency)

언어별 지정가 주문 매칭 지연 시간 (벤치마크 참고치):

C++     ████                    ~1μs (마이크로초)
Rust    ████                    ~1μs (C++와 동급)
Java    ████████████            ~3-5μs (GC pause 가능)
Go      ████████████████        ~5-10μs (GC pause 가능)
Node.js ████████████████████████ ~10-50μs

GC(가비지 컬렉터) 문제:
  Java/Go는 평균 성능은 좋지만, GC가 동작하는 순간 지연 급증
  → 99th percentile latency가 중요한 거래소에서는 치명적
  → "평균 1ms인데, 100번에 1번은 50ms" ← 트레이더가 싫어함

러스트의 장점:
  → GC 없음 = 예측 가능한 지연 시간
  → Zero-cost abstractions = 추상화해도 성능 저하 없음
  → 컴파일 타임 메모리 관리 = 런타임 오버헤드 없음
```

### 10.2 실제 거래소의 언어 선택

```
주요 거래소/금융사의 기술 스택:

바이낸스 (Binance):
  └── 매칭 엔진: Java → Rust로 마이그레이션 중
  └── 이유: 지연 시간 최적화, 메모리 안전성

1inch (DEX Aggregator):
  └── 핵심 로직: Rust
  └── 이유: 온체인 연산 최적화

Solana 블록체인:
  └── 전체 코드베이스: Rust
  └── 이유: 고성능 합의 메커니즘 구현

Cloudflare:
  └── 핵심 네트워크 프록시: Rust
  └── 이유: C의 성능 + 메모리 안전성

Discord:
  └── Read States 서비스: Go → Rust 마이그레이션
  └── 이유: GC pause 제거, 메모리 사용량 감소

→ 고성능 + 안전성이 중요한 곳에서 러스트 채택이 가속화되고 있다
```

---

## 📝 이번 주 핵심 정리

1. **rustup으로 러스트를 설치하고, cargo로 프로젝트를 관리한다** — cargo는 npm + webpack의 역할을 합친 것이다
2. **러스트의 변수는 기본이 불변(immutable)이다** — 변경이 필요할 때만 `mut`을 명시한다
3. **러스트는 표현식 기반 언어다** — if, match, 블록 {}이 모두 값을 반환한다
4. **정수형은 용도에 맞게 선택한다** — 주문 ID는 u64, 수량은 u64, 가격은 f64(학습용) 또는 i64(실전)
5. **&str과 String은 다르다** — 리터럴은 &str, 동적 생성은 String, 자세한 이유는 4주차(소유권)에서 다룬다
6. **match는 switch의 상위 호환이다** — 모든 경우를 처리해야 하고(exhaustive), 값을 반환할 수 있다
7. **러스트 컴파일러가 메모리 관련 버그의 70%를 컴파일 타임에 잡아준다** — 거래소에서 이것은 돈과 직결된다

---

## 🔗 참고 자료

- **The Rust Programming Language (공식 Book)**: [doc.rust-lang.org/book](https://doc.rust-lang.org/book/)
- **Rust by Example**: [doc.rust-lang.org/rust-by-example](https://doc.rust-lang.org/rust-by-example/)
- **Rust Playground (브라우저에서 실행)**: [play.rust-lang.org](https://play.rust-lang.org/)
- **rustup 공식 문서**: [rustup.rs](https://rustup.rs/)
- **cargo 공식 문서**: [doc.rust-lang.org/cargo](https://doc.rust-lang.org/cargo/)
- **rust-analyzer**: [rust-analyzer.github.io](https://rust-analyzer.github.io/)

---

## 🎯 다음 주차 예고

**4주차: 소유권(Ownership) — 러스트의 핵심**에서는 러스트를 러스트답게 만드는 핵심 개념을 다룹니다. 소유권 규칙 3가지, 이동(Move)과 복사(Copy), 참조(Reference)와 빌림(Borrowing)을 배우고, 주문 데이터를 안전하게 전달하는 함수를 설계합니다.

소유권은 러스트의 가장 독특한 개념이며, 이것을 이해하면 `&str`과 `String`의 차이, 함수에 데이터를 전달하는 방식, 메모리 안전성의 원리를 모두 이해할 수 있습니다.

---

## 💡 실전 과제

1. **환경 설정 확인**: `rustc --version`, `cargo --version`, `cargo clippy --version`을 실행하여 모든 도구가 정상 설치되었는지 확인하세요

2. **order-cli 확장**: 실전 프로젝트의 order-cli에 다음 기능을 추가해보세요
   - 시장가 주문(market) 유형 추가 (가격 입력 없이 수량만 입력)
   - 주문 내역을 `Vec`에 저장하고 `list` 명령으로 출력 (힌트: `Vec`은 가변 크기 배열)

3. **타입 연습**: 다음 값들에 적합한 러스트 타입을 선택하고 이유를 설명하세요
   - 주문 ID
   - 비트코인 가격 (달러)
   - 거래쌍 이름 ("BTC/USDC")
   - 주문이 매수인지 매도인지
   - 주문 수량
   - 타임스탬프 (Unix epoch)

4. **match 연습**: 거래소의 에러 코드를 match로 처리하는 함수를 작성하세요
   - 1000: "잔액 부족"
   - 1001: "최소 주문 수량 미달"
   - 1002: "가격이 호가 단위에 맞지 않음"
   - 2000~2999: "시스템 에러"
   - 그 외: "알 수 없는 에러"

5. **Rust Playground 활용**: [play.rust-lang.org](https://play.rust-lang.org/)에서 이번 주 코드 예제를 직접 실행해보고, 컴파일 에러를 일부러 유발하여 에러 메시지를 읽는 연습을 하세요 (러스트의 에러 메시지는 매우 상세하고 유용합니다)
