# 수명(lifetime)

## 한 줄 요약

`'a`는 "빌려온 원본이 살아 있는 기간"에 붙인 이름표다.
구조체가 그 이름표를 달고 있으면, 컴파일러가 구조체가 원본보다 오래 쓰이지 않는지 검사한다.
목적은 댕글링 포인터를 컴파일 단계에서 막는 것이다.

---

## 기본 형태

```rust
pub struct Tokens<'a> {
    tokens: &'a [token_t],
}
```

- `Tokens<'a>` : "이 타입은 `'a`라는 수명 매개변수를 받는다"고 **선언**
- `&'a [token_t]` : "이 필드가 가리키는 원본은 `'a` 동안 유효하다"고 **사용**

제네릭 `struct A<T> { x: T }`와 같은 선언/사용 구조다.

- `'`가 붙으면 수명, 안 붙으면 타입이다. `< >` 안에는 둘 다 들어갈 수 있다. (`struct C<'a, T>`)
- `a`는 그냥 이름이다. `'buf`라고 써도 되지만 선언과 사용이 같아야 한다.
- 선언만 하고 어떤 필드에도 안 쓰면 컴파일 에러가 난다.

---

## 자주 헷갈린 점

| 오해 | 실제 |
|---|---|
| `'a`는 타입이다 | 타입이 아니라 수명 매개변수다 |
| 만들 때 `<'a>`를 넣어줘야 한다 | 정의할 때만 적는다. 만들 때는 `Tokens::new(...)`로 쓰고 컴파일러가 추론한다 |
| 그 수명만큼 살아 있게 된다 | 반대다. 수명은 **상한선**이다. 원본보다 오래 살 수 없다는 제한이고, 수명을 늘려주지 않는다 |
| `<>` 안 변수의 수명이다 | 필드가 **가리키는 원본 데이터**의 수명이다 |

---

## 언제 필요한가

바깥 데이터를 빌려서 가리키는 필드(참조 `&`)가 있을 때만 필요하다.

```rust
struct Example<'a> {
    borrowed: &'a str,   // 바깥 문자열을 가리킴 → 수명 필요
    owned: String,       // 자기가 소유 (C의 malloc한 복사본) → 필요 없음
    count: u32,          // 값 자체 → 필요 없음
}
```

참조 필드가 여러 개면 수명을 따로 줄 수도 있고(`<'a, 'b>`), 하나로 묶을 수도 있다.
같이 살다 같이 사라지는 원본이면 하나로 묶는 경우가 더 흔하다.

---

## 예시

### 정상

```rust
let original = String::from("vop");
let n = Name { text: &original };
println!("{}", n.text);   // OK: original이 아직 살아 있음
```

### 컴파일 에러 (댕글링)

```rust
let n;
{
    let original = String::from("vop");
    n = Name { text: &original };
}                        // original이 여기서 사라짐
println!("{}", n.text);  // 에러: original은 이미 없음
```

C라면 이 코드가 컴파일되어 해제된 메모리를 읽는다.

---

## arcus-vector에서

memcached는 명령 줄을 `tokenize_command()`로 잘라 `token_t { value, length }` 배열을 만들고,
extension의 `accept`/`execute`에 `argv`/`argc`로 넘긴다.

```
"vop create idx 2 0\r\n"
 → [ {"vop",3}, {"create",6}, {"idx",3}, {"2",1}, {"0",1} ]
```

각 `value`는 복사본이 아니라 연결의 읽기 버퍼 안을 직접 가리킨다.
`execute`가 끝나면 버퍼가 재사용되므로, 그 뒤에 쓰면 안 된다.

```rust
unsafe extern "C" fn execute(..., argc: c_int, argv: *mut token_t, ...) -> bool {
    let tokens = unsafe { Tokens::new(argv, argc) };  // memcached의 argv를 빌림
    protocol::parse(&tokens);                         // execute 안에서만 사용 → OK
}
```

`tokens`를 전역에 저장해 다음 요청에서 쓰려고 하면 컴파일 에러가 난다.

단, C 포인터는 Rust가 수명을 알 수 없어서 "`'a` 동안 유효하다"는 건 호출하는 쪽이 약속해야 한다.
그래서 `Tokens::new`는 `unsafe`이고, 약속 내용을 `# Safety` 주석에 적는다.
한 번 `Tokens`로 감싼 뒤부터는 컴파일러가 검사한다.

`text()`의 반환 타입도 `&'a str`이라서, 꺼낸 문자열이 원본 버퍼를 가리킨다는 것(복사 없음)이 타입에 드러난다.
