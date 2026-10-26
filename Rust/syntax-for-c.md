# C 개발자를 위한 기본 문법

## 한 줄 요약

Rust는 C에서 "주석이나 관례로 지키던 약속"(NULL 검사, 에러 코드 확인, 범위 검사)을
타입과 문법으로 강제한다.

---

## 마지막 식이 반환값

세미콜론 없는 마지막 식이 함수의 반환값이다.

```rust
fn is_vop(name: &[u8]) -> bool {
    name == b"vop"        // return name == "vop"; 와 같음
}
```

중간에 빠져나갈 때만 `return`을 쓴다.

---

## 클로저 `|| { ... }`

이름 없는 작은 함수다. 바깥 변수를 그대로 가져다 쓸 수 있다.

```rust
let on_panic = || {
    if !ndata.is_null() { unsafe { *ndata = 0 }; }   // 바깥 변수 ndata 사용
    true
};

let add_one = |x| x + 1;   // 인자가 있으면 | | 사이에 적음
```

C에는 대응하는 문법이 없다. 함수 포인터 + `void *ctx`로 흉내 내던 것을 한 번에 쓴다.

---

## Option: NULL일 수 있는 값

Rust의 참조와 함수 포인터는 NULL이 될 수 없다.
"없을 수도 있음"은 `Option`으로 표현한다.

- 값이 있으면 `Some(값)`
- 없으면 `None`

bindgen은 C의 nullable 함수 포인터를 `Option<fn>`으로 만든다.
`Option` 상태로는 호출할 수 없어서, 꺼내는 과정에서 NULL 검사가 강제된다.

```rust
let Some(respond) = response_handler else {
    return false;
};
respond(cookie, len, ptr);
```

```c
// C로 치면
if (response_handler == NULL) return false;
handler_t respond = response_handler;
respond(cookie, len, ptr);
```

- 꺼낸 `respond`는 그 함수 안에서는 계속 쓸 수 있는 지역 변수다.
- 경계에서 한 번 꺼내고, 안쪽 함수에는 `Option`이 아닌 값을 넘기면 다시 검사할 필요가 없다.
- 반대로 `Some(record)`처럼 감싸면 "NULL 아닌 함수 포인터"를 넘기는 것이다.

---

## Shadowing

같은 이름으로 다시 선언하면 이전 변수가 가려진다.

```rust
let Some(get_server_api) = get_server_api else { return FATAL; };
```

위 줄 이후로 `get_server_api`는 `Option`이 아니라 꺼낸 함수 포인터다.
C에서 안쪽 블록에 같은 이름 변수를 선언하는 것과 비슷하다.

---

## Result와 `?`

에러가 날 수 있는 함수는 `Result<성공 타입, 에러 타입>`을 반환한다.

- 성공: `Ok(값)`
- 실패: `Err(에러)`

C의 "에러 코드 반환 + 출력 포인터 인자"를 하나로 합친 것이다.

```rust
fn text(&self, i: usize) -> Result<&str, Error> {
    let token = self.tokens.get(i).ok_or(Error::MALFORMED)?;
    ...
}
```

`?`는 에러면 그대로 반환하는 연산자다.

```c
// C로 치면
if (i >= argc) return ERR_MALFORMED;
```

자주 쓰는 변환:

| 메서드 | 의미 |
|---|---|
| `.ok_or(e)` | `None`이면 `Err(e)`로 바꿈 |
| `.map_err(\|_\| e)` | 에러 종류를 `e`로 바꿈 |
| `.unwrap_or(v)` | 실패면 기본값 `v` |
| `.unwrap()` | 실패면 패닉 (프로덕션 코드에서는 린트로 막는 편) |

---

## 슬라이스 `&[T]`

"주소 + 길이"를 함께 가진 배열 참조다.
C의 `T *ptr; size_t len;`를 하나로 묶은 것이다.

- `.len()`으로 길이를 알 수 있다.
- `s[i]`는 범위를 벗어나면 패닉, `s.get(i)`는 `None`을 반환한다.
- 슬라이스끼리 `==`로 비교하면 길이까지 비교한다. (`"vopx" != "vop"`)

C 포인터로 슬라이스를 만들 때는 `unsafe`가 필요하다.

```rust
let name = unsafe { std::slice::from_raw_parts(ptr, len) };
```

---

## 바이트 문자열과 C 문자열

| 표기 | 의미 |
|---|---|
| `"abc"` | UTF-8 문자열 (`&str`) |
| `b"abc"` | 바이트 배열 (`&[u8]`), 끝에 `\0` 없음 |
| `c"abc"` | C 문자열, 끝에 `\0` 있음 |

---

## 속성 `#[...]`

바로 아래 항목에 대해 컴파일러에게 지시하는 표시다.
C의 `__attribute__`, `#pragma`, `#ifdef`를 합친 역할이다.

| 속성 | 의미 | C 비유 |
|---|---|---|
| `#[derive(Clone, Copy)]` | 복사 기능 코드를 자동 생성 | C 구조체는 기본으로 대입 복사가 되지만 Rust는 허용해야 함 |
| `#[cfg(test)]` | 테스트 빌드에서만 포함 | `#ifdef TEST` |
| `#[test]` | 테스트 함수로 등록 | - |
| `#[unsafe(no_mangle)]` | 심볼 이름을 그대로 export | - |
| `#[expect(lint, reason = "...")]` | 이 경고는 예상된 것 | `#pragma GCC diagnostic ignored` |
| `#[repr(C)]` | C와 같은 메모리 레이아웃 | - |

---

## thread_local과 Cell

```rust
thread_local! {
    static ACCEPT_PANICKED: Cell<bool> = const { Cell::new(false) };
}
```

- `thread_local!` : 스레드마다 따로 있는 전역 변수다. C의 `__thread bool x = false;`
- `Cell<bool>` : 겉은 불변이어도 안의 값을 `.set()`, `.replace()`로 바꿀 수 있게 하는 상자다.
  Rust의 `static`은 원래 수정할 수 없어서, `static mut` 없이 값을 바꾸려면 이런 타입이 필요하다.
- `const { ... }` : Cell이 상수라는 뜻이 아니라, 초기값을 컴파일 시점에 정한다는 뜻이다.
- `.replace(false)` : 이전 값을 꺼내면서 새 값으로 바꾼다. "확인하고 지우기"를 한 번에 한다.
