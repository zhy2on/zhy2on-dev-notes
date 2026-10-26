# FFI, unsafe, 패닉

## 한 줄 요약

C와 주고받는 경계(FFI)에서는 Rust의 안전 검사가 닿지 않는다.
그래서 위험한 코드는 `unsafe` 블록에 모으고 왜 안전한지 주석으로 남기며,
Rust의 패닉이 C 쪽으로 넘어가지 않게 경계에서 잡는다.

---

## C에서 부를 수 있는 함수

```rust
#[unsafe(no_mangle)]
pub extern "C" fn memcached_extensions_initialize(
    _config: *const c_char,
    get_server_api: GET_SERVER_API,
) -> EXTENSION_ERROR_CODE { ... }
```

- `extern "C"` : C 호출 규약을 쓴다.
- `#[unsafe(no_mangle)]` : 심볼 이름을 바꾸지 않고 그대로 export한다. memcached가 `dlsym`으로 찾는다.
- `unsafe extern "C" fn` : 호출하는 쪽이 지켜야 할 조건이 있는 함수다. 조건은 `# Safety` 문서 주석에 적는다.

---

## 원시 포인터와 unsafe

| Rust | C |
|---|---|
| `*const T` | `const T *` |
| `*mut T` | `T *` |
| `ptr::null()` | `NULL` |
| `p.is_null()` | `p == NULL` |
| `p.cast::<U>()` | `(U *)p` |
| `&raw mut X` | `&X` (참조를 만들지 않고 주소만) |

원시 포인터를 역참조하거나 C 함수를 부르는 건 `unsafe` 블록 안에서만 할 수 있다.
프로젝트 린트로 `unsafe` 블록마다 `// SAFETY:` 주석을 강제한다.

```rust
// SAFETY: `server` was null-checked above and points to the core's static handle.
let extension = unsafe { (*server).extension };
```

좋은 구조는 `unsafe`를 경계 한 곳(예: `Tokens::new`)에 모으고, 그 뒤로는 안전한 타입만 쓰는 것이다.

---

## static mut

수정 가능한 전역 변수다.
memcached에 등록하는 descriptor는 코어가 `next` 필드를 써넣기 때문에 `static mut`이어야 한다.

edition 2024부터 `static mut`에 대한 참조(`&DESCRIPTOR`)는 만들 수 없고,
원시 포인터(`&raw mut DESCRIPTOR`)로만 다룬다.

---

## 패닉

패닉은 Rust의 "복구하지 않는 에러"다. 배열 범위 초과, `unwrap()` 실패 같은 버그에서 난다.

- 기본 동작은 스택을 거슬러 올라가며 정리하는 unwinding이다.
- Rust 1.81부터 패닉이 `extern "C"` 함수 밖(C 쪽)으로 나가려 하면 프로세스가 abort된다.
  extension이라면 memcached 서버 전체가 죽는다.
- 메모리 할당 실패는 패닉이 아니라 바로 abort라서 잡을 수 없다.

### catch_unwind로 경계에서 잡기

```rust
pub(crate) fn guard<T>(on_panic: impl FnOnce() -> T, f: impl FnOnce() -> T) -> T {
    catch_unwind(AssertUnwindSafe(f)).unwrap_or_else(|_| on_panic())
}
```

- `f`를 실행하다 패닉이 나면 `on_panic`의 결과를 대신 반환한다.
- C 비유: `setjmp`로 감싸서 `longjmp`가 오면 기본값을 반환하는 래퍼.
- C가 부르는 콜백(`accept`, `execute`, 초기화 함수)마다 이걸로 감싼다.
- `Cargo.toml`에 `panic = "abort"`를 설정하면 `catch_unwind`가 동작하지 않는다.

### 패닉 메시지

기본 패닉 훅은 stderr로 출력한다.
memcached를 `-v` 없이 데몬 모드로 띄우면 stderr가 `/dev/null`이라 메시지가 사라진다.
남기려면 `std::panic::set_hook`으로 memcached 로거에 쓰도록 바꿔야 한다.

### 패닉 후 연결 처리

`accept`에서 패닉이 나면 "내 명령이 아니다"(false)가 아니라
"내 명령이다"(true)로 받고, 이어지는 `execute`에서 false를 반환해 연결을 닫는다.

false를 반환하면 본문이 있는 명령에서 본문이 다음 명령으로 해석되는 문제가 생긴다.
memcached 텍스트 프로토콜은 클라이언트가 명령 줄과 본문을 한 번에 보내기 때문에,
서버가 본문을 읽어서 버리거나(swallow) 연결을 닫아야 한다.
