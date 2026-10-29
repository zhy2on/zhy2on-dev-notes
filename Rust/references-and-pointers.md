# 참조와 원시 포인터

## 한 줄 요약

Rust의 포인터는 컴파일러가 검사하는 **참조**(`&T`, `&mut T`)와 검사하지 않는 **원시 포인터**(`*const T`, `*mut T`)로 나뉜다.
평소에는 참조를 쓰고, 컴파일러가 수명이나 공유 관계를 따라갈 수 없는 곳(FFI, 침입형 리스트)에서만 원시 포인터로 내려간다.

---

## 포인터 4종류

| 종류 | 뜻 | C 비유 | 컴파일러 검사 |
|---|---|---|---|
| `&T` | 공유 참조. 읽기만 함 | `const T *` | 있음 |
| `&mut T` | 독점 참조. 읽고 씀 | `T *restrict` | 있음 |
| `*const T` | 원시 포인터 (읽기 의도) | `const T *` | 없음 (`unsafe`) |
| `*mut T` | 원시 포인터 (쓰기 의도) | `T *` | 없음 (`unsafe`) |

참조로 읽기만 하면 `&`를 쓴다. 고치려면 `&mut`가 필요하고, 그동안에는 같은 값을 가리키는 다른 참조가 없어야 한다.
이 규칙은 스레드와 상관없이 한 함수 안에서도 똑같이 적용된다.

C의 `restrict`는 "이 포인터 말고는 이 메모리를 안 가리킨다"는 **약속**이고,
Rust의 `&mut`는 그걸 컴파일러가 **강제**하는 것이다.

---

## borrow checker

컴파일러가 참조 규칙을 확인하는 방식이다.
C 컴파일러가 "초기화 안 된 변수 사용" 경고를 낼 때 쓰는 흐름 분석과 같은 종류다.

1. `&x`를 만들면 "x를 빌림"으로 기록한다.
2. 그 빌림은 **마지막으로 쓰이는 지점까지만** 살아 있다. 변수를 다른 값으로 덮어쓰면 거기서 끝난다.
3. 빌림이 살아 있는 구간에 `&mut x`가 나오면 에러다.

```rust
let mut x = 5;
let y = 6;
let mut b = &x;
b = &y;            // x를 빌린 상태가 여기서 끝남
let c = &mut x;    // OK
*c += 1;
println!("{b}");
```

경로가 갈리면 "가능한 경로 중 하나라도" 기준으로 보수적으로 판단한다.

```rust
let mut b = &x;
if cond { b = &y; }
let c = &mut x;    // 에러: cond가 false면 b가 아직 x를 가리킴
println!("{b}");
```

실제로는 안전해도 컴파일러가 증명하지 못하면 거부한다.
그런 구조를 짜야 할 때 원시 포인터와 `unsafe`를 쓴다.

---

## `*const`와 `*mut`

둘 다 검사가 없고 서로 자유롭게 캐스트할 수 있어서, 강제되는 차이는 작다.

- `*const`로는 `*p = x`처럼 직접 쓸 수 없다. 쓰려면 캐스트가 필요하다.
- 나머지는 **의도 표시**다. `*mut`는 `&mut`를 본떠 "이걸로 고친다", `*const`는 `&`를 본떠 "읽기만 한다(공유)".

주의할 점: 공유 참조(`&T`)에서 나온 포인터를 `*mut`로 캐스트해 `UnsafeCell` 밖의 값을 쓰면 UB다.
C에서 const를 캐스트로 떼고 쓰는 것과 달리, Rust 컴파일러는 "공유된 값은 안 바뀐다"고 가정하고 최적화하기 때문이다.

---

## unsafe

`unsafe`는 검사를 끄는 게 아니라, 원래 금지된 몇 가지 동작을 허용한다.
그 안에서도 참조 규칙과 타입 검사는 그대로 적용된다.

`unsafe`가 여는 동작:

- 원시 포인터 따라가기 (`*p`). 포인터를 **만드는 것**은 안전하다.
- `unsafe fn` 호출 (C 함수, `CStr::from_ptr` 같은 표준 라이브러리 함수 포함)
- `static mut` 접근
- `unsafe impl` (`unsafe impl Send for List {}`)
- `union` 필드 읽기

어디에 `unsafe`가 필요한지는 외울 필요가 없다. 빠뜨리면 컴파일러가 E0133 에러를 낸다.
리뷰할 때는 `// SAFETY:` 주석의 근거가 맞는지를 본다.

### unsafe 블록과 unsafe fn

| 형태 | 뜻 | 근거를 적는 곳 |
|---|---|---|
| `unsafe { ... }` | 여기서 위험한 동작을 하고, 내가 확인했다 | 바로 위 `// SAFETY:` |
| `unsafe fn` | 부르는 쪽이 지켜야 할 조건이 있다 | 함수 문서의 `/// # Safety` |

함수 안에 `unsafe` 블록이 있다고 해서 `unsafe fn`이 되는 건 아니다.
기준은 **호출자에게 요구할 조건이 있느냐**다.

```rust
/// # Safety
///
/// `entry` must stay alive until run and must not be in any list.
unsafe fn push_back(&mut self, entry: *const Entry)   // 넘어온 포인터를 함수가 검증할 수 없음

fn pop_front(&mut self) -> Option<*const Entry>      // 리스트 안의 entry만 읽음. 조건 없음
```

- `push_back`은 바깥에서 받은 `entry`가 살아 있는지, 다른 리스트에 없는지 알 수 없다. 그래서 호출자에게 맡긴다.
- `pop_front`는 이미 `push_back`을 거쳐 들어온 entry만 읽는다. 넣을 때 보장됐으니 스스로 근거를 댈 수 있다.
  안에서 원시 포인터를 따라가니 `unsafe` 블록은 필요하다.

이렇게 `unsafe` 조건이 `push_back` → `submit` → `aio::begin`으로 호출자를 따라 올라가고,
마지막에 새로 할당한 Job을 넣는 곳에서 보장된다.

---

## UnsafeCell: 공유된 상태에서 고치기

```rust
pub struct Entry {
    next: UnsafeCell<*const Self>,
    run: unsafe fn(*const Self),
}
```

entry는 `*const`(읽기 전용, 공유)로 돌려 쓰지만 `next`만은 고쳐야 한다.
`UnsafeCell`은 "이 필드는 공유된 상태에서도 바뀔 수 있다"고 컴파일러에게 알리는 유일한 표시다.
C++의 `mutable` 멤버와 비슷하다.

`get()`은 값이 아니라 **안쪽 값의 주소**(`*mut T`)를 준다. `T`가 무엇이든 항상 포인터다.

```rust
unsafe { *(*self.tail).next.get() = entry };
```

| 식 | 타입 | C |
|---|---|---|
| `self.tail` | `*const Entry` | `tail` |
| `(*self.tail).next` | `UnsafeCell<*const Entry>` | `tail->next` |
| `(*self.tail).next.get()` | `*mut *const Entry` | `&tail->next` |
| `*(...) = entry` | 대입 | `tail->next = entry` |

`*const Entry`의 `const`는 **가리키는 다음 entry**에 붙은 것이지 `next` 필드가 상수라는 뜻이 아니다.

값으로 다루는 다른 방법도 있지만 이 코드에서는 못 쓴다.

| 방법 | 조건 |
|---|---|
| `into_inner(self)` | UnsafeCell을 통째로 소비 |
| `get_mut(&mut self)` | 독점 참조 필요 (entry는 공유라 불가) |
| `Cell<T>`의 `get`/`set` | 한 스레드용(`!Sync`) |

`Cell`, `Mutex`, `Atomic*`은 모두 `UnsafeCell` 위에 만들어져 있다.

---

## `#[repr(C)]`와 첫 필드 트릭

```rust
#[repr(C)]
struct Job {
    entry: Entry,      // 첫 필드
    conn: Connection,
    reply: &'static [u8],
}
```

`#[repr(C)]`는 필드를 적은 순서대로 C와 같은 레이아웃으로 배치한다.
첫 필드의 주소가 구조체의 주소와 같으니 `&job.entry`를 `Job *`으로 되돌릴 수 있다.

```rust
let conn = unsafe { (*entry.cast::<Job>()).conn };
```

C의 `container_of`나 리눅스 `list_head`를 구조체 첫 필드에 넣는 방식과 같다.
리스트 노드를 job 안에 넣어 두니(침입형 리스트) 큐에 넣을 때 할당이 없다.

### 함수 포인터 필드 추적하기

`entry.run`이 실제로 어떤 함수인지 알려면, 필드가 비공개이니 **값을 넣는 생성자**(`Entry::new`)를 부르는 곳을 찾으면 된다.
`Job::new`가 `Entry::new(run)`을 부르므로 레인에서 부르는 `entry.run`은 `aio::run`이다.

---

## Box와 직접 할당

| Rust | C |
|---|---|
| `Box::new(v)` | `malloc` + 대입. 할당 실패 시 abort |
| `alloc(Layout::new::<T>())` + `raw.write(v)` | `malloc` + 대입. 실패 시 NULL 확인 가능 |
| `Box::into_raw(b)` | 소유권을 원시 포인터로 넘김 (free 책임은 내가) |
| `Box::from_raw(p)` | 원시 포인터를 다시 Box로. 스코프를 벗어나면 free |
| `Box::leak(b)` | 끝까지 free하지 않는 malloc. `&'static` 참조를 얻음 |

arcus-vector의 `Job::new`는 메모리 부족 시 서버를 죽이지 않고 에러 응답을 보내려고 `Box::new` 대신 `alloc`을 쓴다.
