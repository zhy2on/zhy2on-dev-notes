# 스레드와 동기화

## 한 줄 요약

Rust는 "이 값을 다른 스레드로 넘겨도 되는가", "여러 스레드가 같이 가리켜도 되는가"를
`Send`/`Sync`라는 표시로 컴파일 시점에 검사한다.
공유 데이터는 `Mutex` 안에 넣어서 락을 잡지 않으면 접근 자체가 안 되게 만든다.

---

## 스레드 만들기

```rust
std::thread::Builder::new()
    .name(format!("arcus-vector-{n}"))
    .spawn(move || {
        ready.wait();
        run(lane);
    });
```

- `spawn`은 표준 라이브러리 함수다. 내부에서 `pthread_create`를 부른다.
- 넘기는 클로저가 C의 `pthread_create(fn, arg)`에서 **fn + arg**를 합친 것이다.
- `move`는 클로저가 쓰는 바깥 변수(`ready`, `lane`)를 클로저 안으로 옮긴다는 뜻이다.

`spawn`은 클로저에 두 조건을 요구한다.

| 조건 | 뜻 |
|---|---|
| `Send` | 클로저(와 클로저가 가진 값)를 새 스레드로 넘겨도 된다 |
| `'static` | 클로저가 빌린 값이 프로그램 끝까지 살아 있다. 새 스레드가 언제 끝날지 모르기 때문 |

그래서 레인 배열은 `Box::leak`으로 `&'static`을 만들어 넘긴다. free하지 않는 malloc이다.

---

## Send와 Sync

| 표시 | 뜻 |
|---|---|
| `Send` | 값을 다른 스레드로 **옮겨도** 된다 |
| `Sync` | 여러 스레드가 **같이 가리켜도** 된다. `T: Sync`는 `&T: Send`와 같은 말 |

- 메서드가 없는 **표시용 트레이트**(marker trait)다. Java의 `Serializable`처럼 "이 타입은 이런 성질이 있다"는 표시만 한다.
- 컴파일 시점에만 쓰이고 런타임 비용이 없다. C++ 템플릿의 type trait 특수화와 비슷하다.
- 직접 구현하지 않아도 **필드가 전부 `Send`면 구조체도 자동으로 `Send`**다. 누가 정의한 구조체인지와는 상관없다.
- 원시 포인터는 `Send`도 `Sync`도 아니다. 그래서 원시 포인터 필드가 있으면 자동으로 붙지 않는다.

### unsafe impl

```rust
struct List {
    head: *const Entry,
    tail: *const Entry,
}

// SAFETY: a `List` is only reached through its lane's mutex, and its entries stay alive until run.
unsafe impl Send for List {}
```

- 몸통이 비어 있는 건 구현할 메서드가 없기 때문이다. "List는 Send다"라는 선언만 한다.
- `unsafe`인 이유는 컴파일러가 확인할 수 없는 걸 개발자가 보장하기 때문이다. 근거는 SAFETY 주석에 적는다.

### 검사가 이어지는 방식

레인 스레드에 `&Lane`을 넘기려면 `Lane: Sync`여야 한다. 컴파일러는 필드를 따라 내려간다.

```
Lane: Sync
 → Mutex<List>: Sync    (Lane의 필드라서)
 → List: Send           (Mutex 규칙: T가 Send면 Mutex<T>는 Sync)
 → 원시 포인터 때문에 자동으로는 안 됨 → unsafe impl Send for List
```

`Mutex`는 락을 잡은 스레드에게 차례로 `&mut List`를 빌려준다.
`List` 자체는 제자리에 있지만 접근 권한이 스레드 사이를 오가니 `List: Send`를 요구한다.

---

## Mutex

```rust
pub struct Lane {
    queue: Mutex<List>,
    wake: Condvar,
    load: AtomicUsize,
}
```

C의 `pthread_mutex_t`와 다른 점은 **보호할 데이터가 Mutex 안에 들어 있다**는 것이다.

- `lock()`을 거치지 않으면 `List`에 접근할 수 없다. C에서는 락 없이 `lane->list`를 만져도 컴파일된다.
- `lock()`은 가드(`MutexGuard`)를 준다. 가드를 통해 `&mut List`처럼 쓴다.
- 가드가 스코프를 벗어나면(`drop`) 자동으로 unlock된다. RAII다.
- 락을 잡은 스레드가 패닉 나면 Mutex가 poison 상태가 된다. `unwrap_or_else(PoisonError::into_inner)`는 그래도 데이터를 꺼내 쓰겠다는 뜻이다.

```rust
fn queue(lane: &Lane) -> MutexGuard<'_, List> {
    lane.queue.lock().unwrap_or_else(PoisonError::into_inner)
}
```

`&Lane`(공유 참조)인데 큐를 고칠 수 있는 건, Mutex가 내부적으로 `UnsafeCell`을 쓰기 때문이다.
`load`도 `Atomic`이라 `&Lane`으로 `fetch_add`할 수 있다.

---

## Condvar

C의 `pthread_cond_t`다.

```rust
// 레인 스레드: 꺼낼 게 없으면 잠듦
let mut queue = queue(lane);
loop {
    if let Some(entry) = queue.pop_front() { break entry; }
    queue = lane.wake.wait(queue).unwrap_or_else(PoisonError::into_inner);
}

// worker: 넣고 깨움
let mut queue = queue(lane);
unsafe { queue.push_back(entry) };
drop(queue);
lane.wake.notify_one();
```

- `wait(guard)`는 락을 풀고 잠들었다가, 깨어나면 락을 다시 잡은 가드를 돌려준다.
- 이유 없이 깨는 경우(spurious wakeup)가 있어서 C처럼 루프로 조건을 다시 확인한다.

C로 쓰면 이렇다.

```c
pthread_mutex_lock(&lane->lock);
while ((entry = list_pop_front(&lane->list)) == NULL)
    pthread_cond_wait(&lane->wake, &lane->lock);
pthread_mutex_unlock(&lane->lock);
entry->run(entry);   /* 락 밖에서 실행 */
```

---

## OnceLock

한 번만 쓸 수 있고, 그 뒤로는 여러 스레드가 락 없이 읽는 칸이다.

```rust
static API: OnceLock<Api> = OnceLock::new();

API.set(api)   // 처음이면 Ok(()), 이미 차 있으면 Err(api) — 넣지 못한 값을 돌려줌
API.get()      // 차 있으면 Some(&Api), 비어 있으면 None
```

- 여러 스레드가 동시에 `set`을 불러도 하나만 성공한다. 그 동기화는 내부에서 한다.
- 채워진 뒤로는 바뀌지 않으니 읽을 때 락이 없다.
- C로 치면 `static struct api api;` + `pthread_once_t`다.
- 로드할 때 한 번 채우고 이후로는 읽기만 하는 설정값에 맞다. 계속 바뀌는 값은 `Mutex`나 `Atomic`을 쓴다.

---

## 그 밖의 도구

| 도구 | 뜻 | C 비유 |
|---|---|---|
| `AtomicUsize`, `AtomicPtr` | 락 없는 원자적 읽기/쓰기 | `_Atomic`, `__atomic_*` |
| `Arc<T>` | 스레드 간 공유하는 참조 카운트 포인터 | refcount + free |
| `Barrier` | N개 스레드가 모두 도착할 때까지 대기 | `pthread_barrier_t` |
| `catch_unwind` | 패닉을 잡아서 스레드가 죽지 않게 함 | `setjmp`/`longjmp` 래퍼 |

---

## 예: arcus-vector 레인 풀

```
Lane × 32
 ├─ 레인 스레드 1개
 ├─ queue: Mutex<List>   (Entry 연결 리스트)
 ├─ wake: Condvar
 └─ load: AtomicUsize    (대기 + 실행 중인 job 수)
```

| 스레드 | 하는 일 |
|---|---|
| main 스레드 | 로드할 때 Lane 32개를 만들고, 레인 스레드를 띄우며 `&Lane`을 하나씩 넘김 |
| memcached worker 스레드 | 가장 한가한 레인(`idlest`)을 골라 job을 넣음(`submit`) |
| 레인 스레드 | 자기 큐에서 job을 꺼내 실행(`run`) |

Job 하나를 두 곳이 동시에 가리킨다.

| 보관 위치 | 값 | 쓰는 곳 |
|---|---|---|
| 연결 `c->aiocb_data` | `job` | worker가 `wake`에서 응답을 보내고 free |
| 레인 큐 | `&job->entry` (같은 주소) | 레인 스레드가 `run`에서 `notify_io` |

`run`은 `conn`을 먼저 복사한 뒤 `notify_io`를 부른다.
notify 순간 다른 worker에서 `wake`가 돌아 Job을 free할 수 있어서, 그 뒤로는 Job에 손대면 안 된다.

### fork와 스레드

`-d`(daemonize)는 `-X` 로드 **이후에** fork한다. fork하면 부른 스레드 하나만 자식에 복사되고 레인 스레드는 따라오지 않는다.
그래서 `pthread_atfork`의 child 핸들러에서 레인을 새로 만든다.

- POSIX는 멀티스레드 프로세스를 fork한 자식에서 async-signal-safe 함수만 허용한다. 할당과 스레드 생성은 여기에 들지 않는다.
- 실제로는 glibc, jemalloc 등이 fork 때 malloc 락을 정리해 줘서 동작한다.
- `Barrier`로 레인이 시작 도중(전역 락을 잡은 채)에 fork되지 않게 한다.
- 더 깔끔한 방법은 core에 daemonize 이후 호출되는 시작 이벤트를 두고 거기서 스레드를 한 번만 만드는 것이다.
