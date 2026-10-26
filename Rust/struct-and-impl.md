# struct와 impl

## 한 줄 요약

Rust는 데이터(`struct`)와 그 데이터를 다루는 함수(`impl`)를 따로 적는다.
Java나 Python의 클래스 메서드와 같은 개념인데, 쓰는 위치만 떼어놓은 것이다.

---

## C와 비교

```c
// C
struct tokens { token_t *argv; size_t argc; };

struct tokens tokens_new(token_t *argv, int argc) { ... }
size_t tokens_len(const struct tokens *t) { ... }
```

```rust
// Rust
pub struct Tokens<'a> {
    tokens: &'a [token_t],
}

impl<'a> Tokens<'a> {
    pub unsafe fn new(argv: *const token_t, argc: c_int) -> Self { ... }
    pub fn len(&self) -> usize { self.tokens.len() }
}
```

- `struct` : 데이터 구조의 완전한 정의. 헤더처럼 선언만 따로 두는 부분은 없다.
- `impl` : 그 구조체를 다루는 함수들. C에서 `struct tokens *`를 첫 인자로 받는 함수들을 모아둔 것과 같다.
- `t.len()`은 C의 `tokens_len(&t)`를 점 문법으로 쓴 것뿐이다.

Go도 비슷하게 데이터와 메서드를 나눠서 적는다.
나눠두면 `impl` 블록을 여러 개 둘 수 있고, 트레이트(인터페이스) 구현을 따로 붙이기 쉽다.
`#[derive(Clone, Copy)]`도 그런 `impl`을 자동으로 만들어주는 것이다.

---

## impl 안의 두 종류 함수

| 종류 | 예 | 부르는 법 | 비유 |
|---|---|---|---|
| `self`를 받는 함수 | `fn len(&self)` | `t.len()` | 인스턴스 메서드 |
| `self`가 없는 함수 | `fn new(...)` | `Tokens::new(...)` | Java의 `static` 메서드 |

- `&self` : 읽기 전용으로 받음. C의 `const struct tokens *self`
- `&mut self` : 수정 가능하게 받음. C의 `struct tokens *self`
- `Self` : 지금 `impl` 중인 타입 자체 (`Tokens<'a>`)

`new`는 특별한 생성자 키워드가 아니라 관례상 붙인 이름이다.
실제로 구조체를 만드는 건 필드를 채우는 부분이다.

```rust
Self { tokens: &[] }   // C의 (struct tokens){ .tokens = ... }
Self { tokens }        // 변수 이름과 필드 이름이 같으면 줄여 쓸 수 있음
```

---

## 비공개 필드 = 불투명 구조체

```rust
pub struct Tokens<'a> {
    tokens: &'a [token_t],   // pub 없음
}
```

구조체는 `pub`이지만 필드는 비공개다.
다른 모듈은 `Tokens` 타입은 쓸 수 있지만 `tokens` 필드를 직접 건드릴 수 없고,
`new`, `len`, `text` 같은 메서드로만 접근한다.

C로 치면 헤더에 `typedef struct tokens Tokens;`만 두고 내용은 `.c`에 숨기는 opaque struct와 같다.
이렇게 하면 위험한 코드(`unsafe`)를 `new` 한 곳에 모아둘 수 있다.
