# Worker Event Loop와 상태 머신

worker 스레드가 libevent 위에서 여러 커넥션을 어떻게 처리하는지, 커넥션별 상태 머신이 어떻게 도는지 정리한 문서.

- 관련 문서: [Persistence IO Blocking](./persistence-io-blocking.md), [Extension 비동기 처리](./extension-async-offloading.md)

## 1. 한 줄 요약

> worker 스레드는 libevent 루프 하나를 돌며, 이벤트가 온 커넥션의 상태 머신을 `false`가 나올 때까지 돌린다.
> 상태(`c->state`)는 "다음에 여기서부터 이어서 해"라는 책갈피 역할을 한다.

## 2. libevent 기본

libevent는 "이 fd에서 이런 일이 생기면 이 함수를 불러줘"를 등록해두는 라이브러리.

```c
event_set(&ev, fd, EV_READ | EV_PERSIST, callback, arg);  // fd가 읽기 가능해지면 callback(arg) 호출
event_add(&ev, NULL);    // 감시 목록에 등록
event_del(&ev);          // 감시 목록에서 제거
event_base_loop(base);   // 무한 루프: 등록된 fd들을 epoll 등으로 기다리다가, 준비된 fd의 callback 호출
```

worker 스레드의 본체는 이 루프 하나 (`thread.c` `worker_libevent()`):

```c
static void *worker_libevent(void *arg)
{
    ...
    event_base_loop(me->base, 0);   // 서버 종료까지 여기서 안 나옴
}
```

worker 하나의 감시 목록:

```
worker 1의 감시 목록
 ├─ notify pipe fd   → thread_libevent_process()   (다른 스레드가 worker를 부르는 통로)
 ├─ 커넥션 A 소켓 fd  → event_handler(A)
 ├─ 커넥션 B 소켓 fd  → event_handler(B)
 └─ 커넥션 C 소켓 fd  → event_handler(C)
```

커넥션이 생기면 소켓을 등록 (`memcached.c` `conn_new()`):

```c
event_set(&c->event, sfd, event_flags, event_handler, (void *)c);
event_add(&c->event, timeout);
```

중요: **worker는 스레드 하나라서 한 번에 callback 하나만 실행**.
A의 callback이 실행 중이면 B, C는 대기. A가 return해야 루프로 돌아가 B, C를 처리할 수 있음.
→ A 처리 중에 오래 걸리는 작업(fsync, vector search 등)을 직접 하면 B, C까지 다 멈춤.

## 3. callback 안에서 상태 머신이 돈다

커넥션 소켓에 데이터가 오면 libevent가 `event_handler(c)` 호출 (`memcached.c`):

```c
void event_handler(const int fd, const short which, void *arg)
{
    conn *c = (conn *)arg;
    ...
    while (c->state(c)) {
        /* do task */
    }
}   // return → libevent 루프 복귀 → 다른 커넥션 처리
```

- `c->state`: 커넥션의 현재 단계를 나타내는 함수 포인터 (`conn_read`, `conn_nread`, `conn_write` 등)
- 상태 함수가 `true` 반환 → 계속 진행 (보통 다음 상태로 바꾼 뒤)
- `false` 반환 → 이번 차례 끝. callback이 return하고 worker는 다른 커넥션으로
- 다음에 이 커넥션 callback이 다시 불리면 `c->state`에 저장된 단계부터 이어서 진행
- 상태는 **커넥션마다 따로** 있고, 읽고 바꾸는 건 **그 커넥션을 맡은 worker 스레드뿐**
- 상태 전이는 `conn_set_state(c, 다음상태)`로 하고, 다음 상태를 정하는 건 현재 실행 중인 상태 함수(또는 그 안에서 호출된 명령 처리 함수)

즉 상태 머신 덕분에 **커넥션 처리를 중간에 끊었다가 나중에 이어서 할 수 있음**. 비동기 블록은 이 성질을 이용함.

## 4. 언제까지 진행되나: `false`가 나올 때까지

상태는 순환 구조라 "마지막 state"는 없음. `false`가 나오는 지점이 이번 차례의 끝.

```
conn_waiting → conn_read → conn_parse_cmd → (conn_nread) → conn_write → conn_mwrite → conn_new_cmd ─┐
      ▲                                                                                             │
      └─────────────────────────────────────────────────────────────────────────────────────────────┘
```

- 데이터 블록이 없는 명령(`delete`, `incr` 등): `conn_parse_cmd`에서 명령 처리 + 응답 작성
- 데이터 블록이 있는 명령(`set`, `lop insert` 등): `conn_parse_cmd`에서 명령 라인 파싱 후, `conn_nread`에서 value를 마저 읽고 명령 처리 + 응답 작성

`false`를 반환하는 대표적인 경우:

| 상황 | 상태 함수 | 의미 |
|---|---|---|
| 더 읽을 데이터 없음 | `conn_read`가 데이터 없음 확인 → `conn_waiting`이 `false` | 가장 흔한 종료. 다음 요청이 오면 다시 호출됨 |
| 한 번에 처리할 요청 수 초과 | `conn_new_cmd` (`nevents` 소진) | 다른 커넥션이 굶지 않도록 양보 (`reqs_per_event`) |
| 응답을 다 못 보냄 | `conn_mwrite` (`TRANSMIT_SOFT_ERROR`) | 소켓 송신 버퍼 가득 참(EAGAIN). `EV_WRITE`로 바꿔 등록하고 쓸 수 있게 되면 다시 호출됨 |
| 비동기 작업 대기 | `conn_nread` / `conn_parse_cmd` (`should_io_blocked`) | 커넥션 블록. [Persistence IO Blocking](./persistence-io-blocking.md) 참고 |

→ "요청 하나 처리 후 다른 커넥션으로"가 아니라, 보통 **그 커넥션에 와 있는 요청을 (한도 내에서) 다 처리하고 더 읽을 게 없을 때** 다른 커넥션으로 넘어감.

## 5. 도중에 다른 커넥션으로 넘어가나: 같은 worker 안에서는 없음

- worker는 스레드 하나이고, libevent는 callback이 return해야 다음 callback을 부름
- 중간에 끼어드는 선점이 없는 협력형(cooperative) 구조
- 그래서 상태 함수가 오래 걸리면 같은 worker의 다른 커넥션이 전부 기다림 → **vector search를 별도 스레드로 빼려는 이유**
- 단, worker 스레드는 여러 개이고 각자 자기 커넥션을 병렬 처리. 커넥션은 생성 시 worker 하나에 배정되고 이후 바뀌지 않음

## 6. 그 사이 다른 커넥션의 이벤트는 어디에: 커널

- **데이터**: 각 커넥션 소켓의 커널 수신 버퍼
- **"읽을 게 있다"는 준비 상태**: 커널이 fd별로 관리 (epoll이면 커널 내부 ready list)

callback이 return하면:

```
event_base_loop:
  while (1) {
      epoll_wait()           // 커널에 "지금 준비된 fd 목록" 요청 (A 처리 중 B, C가 준비됐다면 여기서 나옴)
      준비된 fd들을 libevent 내부 active 목록에 넣음
      active 목록의 callback을 하나씩 호출   // event_handler(B), event_handler(C), ...
  }
```

- libevent active 목록은 이번 루프 한 바퀴에서 처리할 callback 목록일 뿐, 이벤트를 저장해두는 버퍼가 아님
- memcached는 level-triggered(`EV_READ | EV_PERSIST`) 사용 → 소켓 버퍼에 데이터가 남아 있는 한 epoll이 계속 "준비됨"을 알려줌
- 따라서 A 처리 중 B에 요청이 와도 놓치지 않고, 루프가 돌아오면 B의 callback이 호출됨

## 7. 정리

> 한 worker는 커넥션 하나의 상태 머신을 `false`가 나올 때까지(보통 더 읽을 데이터가 없을 때까지) 돌림.
> 그동안 같은 worker의 다른 커넥션은 끼어들지 못함.
> 그 사이 도착한 데이터와 준비 상태는 커널이 들고 있다가, callback이 return하면 `epoll_wait`로 받아와 차례로 처리함.
