# Cargo와 프로젝트 구조

## 한 줄 요약

Cargo는 Rust의 빌드 도구이자 패키지 관리자다.
C로 치면 `make` + `configure` + 라이브러리 설치를 하나로 합친 것이고,
`Cargo.toml` 하나에 빌드 설정, 의존성, 린트 규칙을 모두 적는다.

---

## 이름으로 정해진 파일들

Cargo는 몇몇 파일을 이름만 보고 역할을 정한다.

| 파일 | 역할 | C 비유 |
|---|---|---|
| `Cargo.toml` | 패키지 설정, 의존성, 린트 | `Makefile` + `configure.ac` |
| `src/lib.rs` | 라이브러리의 시작 파일 | 라이브러리의 메인 `.c` |
| `src/main.rs` | 실행 파일의 시작 파일 | `main()`이 있는 `.c` |
| `build.rs` | 컴파일 전에 먼저 실행되는 빌드 스크립트 | `configure` 단계에서 헤더/코드를 생성하는 스크립트 |

`build.rs`가 쓰는 라이브러리는 `[dependencies]`가 아니라 `[build-dependencies]`에 적는다.
arcus-vector는 `build.rs`에서 bindgen으로 memcached C 헤더를 읽어 Rust 타입을 자동 생성한다.

---

## crate-type

```toml
[lib]
crate-type = ["cdylib"]
```

| 값 | 결과물 | 용도 |
|---|---|---|
| `rlib` | Rust 전용 라이브러리 | 다른 Rust 코드가 가져다 씀 |
| `cdylib` | C에서 쓸 수 있는 공유 라이브러리 (`.so`) | memcached가 `dlopen`으로 로드 |

---

## mod와 use

Rust에는 헤더 파일이 없다. 정의가 곧 선언이다.

- `mod ascii;` : `src/ascii.rs`를 빌드에 포함한다. `mod`로 선언하지 않은 파일은 컴파일되지 않는다.
- `use crate::guard;` : 다른 모듈에 있는 이름을 이 파일에서 짧게 부를 수 있게 한다.

```rust
use std::os::raw::c_int;            // 표준 라이브러리
use super::Error;                   // 한 단계 위 모듈
use crate::memcached::ffi::token_t; // 크레이트 최상위(lib.rs)부터 찾는 절대 경로
```

C의 `#include`는 파일 내용을 그대로 붙여 넣지만, `use`는 이름만 가져온다.
파일을 포함하는 역할은 `mod`가 한다.

하위 모듈은 `protocol.rs` 안에서 `mod tokens;`라고 쓰면 `protocol/tokens.rs`를 찾는다.

---

## 공개 범위

| 표기 | 보이는 범위 |
|---|---|
| (없음) | 같은 모듈 안 |
| `pub(crate)` | 같은 크레이트 안 |
| `pub` | 어디서든 (단, 바깥 모듈이 비공개면 거기까지) |

`.so`에서 어떤 심볼이 export되는지는 `pub`이 아니라 `#[unsafe(no_mangle)]`이 정한다.

---

## 테스트

### 단위 테스트

같은 파일 아래쪽에 둔다. Rust에서 일반적인 관례다.

```rust
#[cfg(test)]          // 테스트 빌드에서만 컴파일 (#ifdef TEST)
mod tests {
    use super::*;     // 바깥 모듈의 이름을 전부 가져옴

    #[test]           // 테스트 함수로 등록
    fn accepts_only_vop_lines() {
        assert!(accepts(&["vop", "create"]));
        assert!(!accepts(&["get", "key"]));
    }
}
```

- 같은 모듈 안이라 비공개 함수도 테스트할 수 있다.
- `#[test]`만 붙이면 `cargo test`가 알아서 찾아 실행한다. 따로 등록하는 `main`이 필요 없다.
- 테스트 내용, mock 함수는 직접 작성한다. 러너만 언어가 제공한다.

### 통합 테스트

`tests/` 폴더에 둔다. 외부 사용자처럼 `pub` API만 쓸 수 있다.

### 테스트 작성 시 주의

결과의 원인을 하나로 좁혀야 한다.
예를 들어 "플래그가 켜져 있으면 false 반환"을 테스트하는데 응답 함수로 `None`을 넘기면,
`None` 분기에서도 false가 나오기 때문에 플래그 검사를 지워도 테스트가 통과한다.
정상 응답 함수를 넘겨야 false의 원인이 플래그뿐이라는 게 확실해진다.

---

## 린트

`Cargo.toml`의 `[lints.rust]`, `[lints.clippy]`에서 경고 수준을 정한다.

- `allow` / `warn` / `deny` : 무시 / 경고 / 에러
- `clippy::pedantic` 같은 그룹은 `priority = -1`로 깔고, 개별 린트로 덮어쓴다.
- 코드에서 예외를 둘 때는 `#[allow(...)]`보다 `#[expect(..., reason = "...")]`를 쓴다.
  `expect`는 경고가 더 이상 안 나면 "불필요한 예외"라고 다시 알려준다.
