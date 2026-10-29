# enum과 에러 처리

## 한 줄 요약

Rust의 enum은 variant마다 값을 담을 수 있는 태그 달린 union이다.
`Result`와 `Option`도 표준 라이브러리에 정의된 평범한 enum이고,
`?`는 "실패면 여기서 return"을 한 글자로 줄인 연산자다.

---

## 값을 담는 enum

```rust
/// Why the server cannot take arcus-vector.
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum Unusable {
    OtherVersion,
    /// The server handle member it lacks.
    Missing(&'static str),
    AlreadyLoaded,
}
```

C enum과 달리 variant가 값을 들고 있을 수 있다. `Missing("core")`처럼 이름을 함께 담는다.

```c
// C로 치면
struct unusable {
    enum { OTHER_VERSION, MISSING, ALREADY_LOADED } kind;
    const char *missing;   /* kind == MISSING일 때만 의미 있음 */
};
```

### derive

컴파일러가 기능을 자동으로 만들어 주라는 표시다.

| derive | 생기는 기능 |
|---|---|
| `Clone, Copy` | C enum처럼 값 복사 |
| `Debug` | `{:?}`로 출력 (`Missing("core")`) |
| `PartialEq, Eq` | `==` 비교. 테스트의 `assert_eq!`에서 필요 |

---

## Result와 Option은 그냥 enum이다

```rust
enum Result<T, E> { Ok(T), Err(E) }
enum Option<T>    { Some(T), None }
```

- 문법 키워드가 아니라 표준 라이브러리 타입이다.
- 모든 파일에 자동으로 import되는 prelude에 들어 있어서 `Ok`, `Err`, `Some`, `None`을 바로 쓴다.
- 문법에 속하는 건 `?` 연산자와 `match` 쪽이다.

### 그냥 값을 반환하는 것과의 차이

```rust
fn new(server: &SERVER_HANDLE_V1) -> Api                     // 실패를 표현할 방법이 없음
fn new(server: &SERVER_HANDLE_V1) -> Result<Api, Unusable>   // 실패할 수 있다는 게 타입에 드러남
```

호출자는 `Api`를 바로 쓸 수 없고 반드시 꺼내는 과정(`?`, `match`)을 거쳐야 한다.
C의 `int api_new(..., struct api *out)`은 반환값을 무시하고 `out`을 써도 컴파일되지만, Rust는 그 실수를 막는다.

### 패닉, Option, Result 고르기

| 상황 | 선택 | 예 |
|---|---|---|
| 실패가 **버그** | 패닉 | `a / b`는 `b == 0`이면 패닉 (`assert`와 비슷) |
| 정상적인 실패, 이유 하나 | `Option` | `a.checked_div(b)` → `None` |
| 정상적인 실패, 이유 여러 개 | `Result` | `Api::new` → `Err(Missing(..))`, `Err(OtherVersion)` |

---

## `?` 연산자

```rust
let api = Api::new(server)?;
```

```rust
// 위와 같은 코드
let api = match Api::new(server) {
    Ok(v) => v,
    Err(e) => return Err(e),   // 현재 함수에서 바로 반환
};
```

```c
// C로 치면
if ((err = api_new(server, &api)) != 0)
    return err;
```

`?`를 붙일 수 있는 조건:

1. 붙이는 값이 `Result`나 `Option`이어야 한다. `bool`에는 못 붙인다.
2. 감싸고 있는 함수의 반환 타입과 맞아야 한다. `Result`를 반환하는 함수에서는 `Result`에, `Option`을 반환하는 함수에서는 `Option`에.
   에러 타입이 달라도 `From`으로 변환할 수 있으면 자동으로 변환된다.

그래서 `Option`을 `Result` 함수 안에서 쓰려면 `ok_or`로 바꿔야 한다.

```rust
fn new(...) -> Result<Self, Unusable> {
    server.core.as_ref()?                          // 에러: Option인데 함수는 Result
    server.core.as_ref().ok_or(Missing("core"))?   // OK
```

`bool`을 돌려주는 검사는 `?`를 못 쓰니 `if` + `return Err`로 쓴다.

```rust
if !built_for(server_version) {
    return Err(Unusable::OtherVersion);
}
```

---

## 체이닝 읽는 법

```rust
let core = unsafe { server.core.as_ref() }.ok_or(Missing("core"))?;
if !built_for(core.server_version.ok_or(Missing("server_version"))?) {
    return Err(Unusable::OtherVersion);
}
```

- `.ok_or(X)`는 "NULL이면 X 에러"라고 읽는다.
- `?`는 "실패하면 여기서 return"이라고 끊어 읽는다.

그러면 이 세 줄에 실패 지점이 세 개 있다.

| 위치 | 실패 조건 | 반환 |
|---|---|---|
| 1번 줄 `?` | `core` 포인터가 NULL | `Missing("core")` |
| 2번 줄 `?` | `server_version` 함수 포인터가 NULL | `Missing("server_version")` |
| 3번 줄 `return` | 버전이 다름 | `OtherVersion` |

```c
// C로 풀면
const SERVER_CORE_API *core = server->core;
if (!core) return MISSING("core");
if (!core->server_version) return MISSING("server_version");
if (!built_for(core->server_version)) return OTHER_VERSION;
```

중첩이 읽기 어려우면 이렇게 나눠 써도 같은 코드다.

```rust
let core = unsafe { server.core.as_ref() }.ok_or(Missing("core"))?;
let server_version = core.server_version.ok_or(Missing("server_version"))?;
if !built_for(server_version) {
    return Err(Unusable::OtherVersion);
}
```

### map_err

```rust
API.set(Api::new(server)?)
    .map_err(|_| Unusable::AlreadyLoaded)
```

1. `Api::new(server)?` : 실패면 `Unusable`을 반환, 성공이면 `Api`를 꺼냄
2. `API.set(api)` : `Result<(), Api>`. 이미 차 있으면 넣지 못한 `api`를 `Err`로 돌려줌
3. `.map_err(|_| ...)` : `Err` 안의 값만 바꿈. 돌려받은 `api`는 `_`로 버리고 `AlreadyLoaded`로

마지막 줄에 세미콜론이 없으니 이 식의 결과가 함수의 반환값이다.

---

## C 문자열 비교

```rust
fn built_for(server_version: unsafe extern "C" fn() -> *const c_char) -> bool {
    // SAFETY: the server's own function; it returns its static version string.
    let running = unsafe { server_version() };
    // SAFETY: a non-null version is the server's static NUL-terminated string.
    !running.is_null() && unsafe { CStr::from_ptr(running) }.to_bytes_with_nul() == VERSION
}
```

```c
// C로 치면
const char *running = server_version();
return running != NULL && strcmp(running, VERSION) == 0;
```

- `CStr::from_ptr` : C 문자열 포인터를 감싼다. 내부에서 `strlen`처럼 `\0`까지 길이를 잰다.
- `to_bytes_with_nul()` : 끝의 `\0`까지 포함한 바이트. bindgen이 만든 `VERSION`이 `b"0.12.1-E-30\0"`처럼 `\0`을 포함하기 때문이다.
- `&&`는 C처럼 단락 평가라서 NULL이면 `from_ptr`을 부르지 않는다.
- `unsafe`가 두 번인 이유는 둘 다 `unsafe fn` 호출이기 때문이다. `char *`를 **받는** 것 자체는 안전하다.

---

## 정의 순서

Rust는 함수, 구조체, `impl`, `static`을 파일 안 어디에 두든 쓸 수 있다. 전방 선언이 없다.
컴파일러가 모듈 전체를 읽고 나서 이름을 찾기 때문이다.

관례상 큰 흐름을 위에, 돕는 함수를 아래에 둔다.
단 함수 안의 `let` 변수는 C처럼 선언한 뒤에만 쓸 수 있다.

---

## 린트 예외: `#[expect]`

```rust
#[expect(
    clippy::expect_used,
    reason = "`set` checks the API at load, before any command"
)]
fn api() -> &'static Api {
    API.get().expect("the API is checked at load")
}
```

- 속성을 붙인 항목(함수, 구조체, `impl`, 문장)에만 적용된다. 함수 하나에만 예외를 둘 수 있다.
- `#[allow]`와 달리, 그 안에서 린트가 더 이상 걸리지 않으면 `unfulfilled_lint_expectations` 경고를 낸다.
  그래서 필요 없어진 예외가 방치되지 않는다.
- `#[expect]`는 "이 안에서 이 린트가 실제로 걸리는가"만 본다. 안 걸리는 이유(코드 삭제, 테스트 안, 린트 꺼짐)는 따지지 않는다.
- `clippy::` 린트에 대한 예외는 `cargo clippy`를 돌릴 때만 확인된다. `cargo build`에서는 안 보인다.
- 이름이 같아 헷갈리는데, `#[expect(...)]` **속성**(린트 예외)과 `.expect("...")` **메서드**(None이면 패닉)는 다른 것이다.

arcus-vector는 `Cargo.toml`에서 `eprintln!`(`print_stderr`), `.unwrap()`, `.expect()`를 린트로 막고,
꼭 필요한 곳에만 `#[expect(.., reason = "..")]`로 예외를 둔다.
