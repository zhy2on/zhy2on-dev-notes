# Persistence IO Blocking

sync 로깅 모드에서 worker가 fsync 완료를 기다리는 동안, 해당 커넥션만 재워두고 다른 커넥션을 계속 처리하는 구조를 정리한 문서.
ARCUS에 이미 존재하는 "별도 스레드 작업 → notify → worker 재개" 구조의 대표 예시.

- 선행 문서: [Worker Event Loop와 상태 머신](./worker-event-loop.md)
- 관련 문서: [Persistence](./persistence.md), [Extension 비동기 처리](./extension-async-offloading.md)

## 1. 왜 persistence 스레드를 따로 두는가

- worker 스레드가 하는 캐시 연산(hash table, collection 접근)은 메모리 작업이라 매우 빠름
- persistence는 변경 내용을 command log 파일에 남겨야 하고, 이건 disk I/O(write, fsync)라 느림
- disk I/O를 worker가 직접 하면, 그동안 같은 worker에 붙은 다른 커넥션 요청이 모두 멈춤
- 그래서 **느린 disk 작업은 별도 스레드가 맡고, worker는 빠른 메모리 작업만 함**

## 2. 등장하는 스레드

| 스레드 | 위치 | 하는 일 |
|---|---|---|
| worker | `memcached.c`, `thread.c` | 요청 처리, 캐시 메모리 반영, **log record를 메모리 log buffer에 기록** |
| log flush 스레드 | `cmdlogbuf.c` `log_flush_thread_main()` | log buffer 내용을 log 파일에 `write()` |
| group commit 스레드 | `cmdlogmgr.c` `do_cmdlog_gcommit_thread_main()` | log 파일 `fsync()` 후, 기다리던 커넥션들을 깨움 |
| checkpoint 스레드 | `checkpoint.c` `chkpt_thread_main()` | 주기적으로 snapshot 생성, 오래된 log 파일 정리 (요청 처리 흐름과는 무관) |

포인트: worker도 log를 "쓰긴" 하지만 **메모리 버퍼에만** 씀. 파일 write와 fsync는 다른 스레드가 함.

## 3. 두 가지 모드

`async_logging` 설정에 따라 응답 시점이 다름.

| 모드 | 응답 시점 | 의미 |
|---|---|---|
| async (`async_logging=true`) | 메모리 반영 직후 바로 응답 | 빠르지만, 응답 후 fsync 전에 죽으면 그 변경은 유실될 수 있음 |
| sync (`async_logging=false`) | **fsync 완료 후** 응답 | 클라이언트가 응답을 받았다면 디스크에 반영된 것이 보장됨 |

- async 모드는 worker가 기다릴 일이 없으므로 일반 명령과 똑같이 처리됨
- **비동기 블록/notify 구조가 쓰이는 건 sync 모드**이고, 아래는 sync 모드 기준
- sync 모드에서도 "성공 + noreply 아님 + 아직 fsync 안 됨"일 때만 블록. 그 외에는 바로 응답

## 4. sync 모드 처리 흐름 (`lop insert` 예시)

### 4.1 한 줄 요약

> worker는 일을 다 끝내고 응답("STORED")까지 만들어 둔 뒤, 해당 커넥션만 잠시 재워둔다.
> fsync가 끝나면 group commit 스레드가 깨워주고, worker는 만들어둔 응답을 보내기만 한다.

### 4.2 단계별

```
 [client]          [worker]                         [log flush]     [group commit]
    │  lop insert    │                                   │                │
    │──────────────▶│                                   │                │
    │                │ ① 메모리에 elem 삽입 (결과 확정)    │                │
    │                │ ② log record → log buffer(메모리)  │                │
    │                │ ③ "fsync 끝나면 알려줘" 등록        │                │
    │                │    (waiter를 대기열에 추가)          │                │
    │                │ ④ 응답 "STORED" 미리 작성           │                │
    │                │ ⑤ 이 커넥션만 재움                  │                │
    │                │   (다른 커넥션 요청은 계속 처리)      │                │
    │                │                                   │ ⑥ buffer→파일   │
    │                │                                   │    write()     │
    │                │                                   │                │ ⑦ fsync()
    │                │◀─────────────────── ⑧ notify_io_complete(커넥션) ───│
    │                │ ⑨ 커넥션 깨움                       │                │
    │◀──────────────│ ⑩ 만들어둔 "STORED" 전송            │                │
```

#### ① ~ ③ engine 내부 (worker 스레드)

`engines/default/coll_list.c` `list_elem_insert()`

```c
PERSISTENCE_ACTION_BEGIN(cookie, UPD_LIST_ELEM_INSERT);  // waiter 생성 (이 요청의 대기표)
...
ret = do_list_elem_insert(it, index, elem);              // ① 메모리 반영
                                                         // ② 내부에서 log record를 log buffer에 기록
...
PERSISTENCE_ACTION_END(ret);                             // ③ cmdlog_waiter_end()
```

`cmdlog_waiter_end()` (`cmdlogmgr.c`)에서:

- 성공 + sync 모드 + noreply 아님 + 아직 fsync 안 됨 → 대기 필요
  - log flush 스레드에게 flush 요청
  - waiter를 group commit 대기열에 추가
  - `waitfor_io_complete(cookie)`: 서버 코어에 "이 커넥션은 notify를 기다려야 함" 표시
  - `ENGINE_EWOULDBLOCK` 반환
- 그 외 → 그냥 `ENGINE_SUCCESS` 반환 (대기 없음)

여기서 중요한 점: **EWOULDBLOCK을 반환하는 시점에 작업 결과(성공, created 여부 등)는 이미 다 정해져 있음**. 기다리는 건 결과가 아니라 "디스크에 안전하게 기록됐다"는 사실뿐.

#### ④ ~ ⑤ 서버 코어 (worker 스레드)

`memcached.c` `process_lop_insert_complete()`

```c
ret = mc_engine.v1->list_elem_insert(...);
CONN_CHECK_AND_SET_EWOULDBLOCK(ret, c);   // EWOULDBLOCK이면: c->ewouldblock = true, ret = SUCCESS
switch (ret) {
case ENGINE_SUCCESS:
    out_string(c, created ? "CREATED_STORED" : "STORED");   // ④ 응답 미리 작성, 상태 = conn_write
```

- EWOULDBLOCK을 성공으로 간주하고 평소처럼 응답을 작성
- `c->ewouldblock = true` 표시만 남겨둠
- 명령 처리 함수가 끝나면 `c->ewouldblock`을 보고 커넥션을 재움 (⑤, 아래 5절)

#### ⑥ ~ ⑧ persistence 스레드들

- log flush 스레드: log buffer 내용을 파일에 `write()`
- group commit 스레드:
  - 2ms 기다렸다가 `fsync()` (그 사이 쌓인 여러 요청을 한 번의 fsync로 처리 = group commit)
  - fsync된 위치(LSN)까지의 waiter들을 대기열에서 꺼냄
  - 각 waiter의 커넥션에 대해 `notify_io_complete(cookie, ENGINE_SUCCESS)` 호출

```c
/* cmdlogmgr.c do_cmdlog_callback_and_free_waiters() */
engine->server.core->notify_io_complete(waiter->cookie, ENGINE_SUCCESS);
```

engine은 커넥션 구조체를 모름. `cookie`(= 커넥션 포인터를 감싼 불투명 값)를 받아뒀다가 그대로 돌려줄 뿐.

#### ⑨ ~ ⑩ 다시 worker 스레드

- `notify_io_complete()`는 해당 커넥션을 worker의 "깨울 목록(`pending_io`)"에 넣고, worker에 신호(pipe)를 보냄
- worker는 신호를 받으면 그 커넥션을 libevent에 다시 등록하고, 멈췄던 지점부터 이어서 진행
- 멈췄던 지점 = 응답 전송 단계(`conn_write`) → 미리 만들어둔 "STORED" 전송

## 5. 커넥션을 재우고 깨우는 방법

한 줄 요약: **스레드가 sleep하는 게 아니라, libevent 감시 목록에서 그 커넥션의 이벤트를 빼고(`event_del`) 플래그(`io_blocked`)를 켜두는 것.**

### 5.1 재우기: `event_del` + `return false`

1. 상태 함수에서 `false` 반환 → callback 탈출 → worker는 다른 커넥션 처리
2. 그 커넥션의 이벤트를 `event_del()`로 감시 목록에서 제거

`thread.c` `should_io_blocked()`:

```c
bool should_io_blocked(const void *cookie)
{
    struct conn *c = (struct conn *)cookie;
    ...
    LOCK_THREAD(thr);
    if (c->current_io_wait > 0) {     // 아직 notify가 안 왔으면
        event_del(&c->event);         // 감시 목록에서 제거
        c->io_blocked = true;         // "재워둠" 표시
        blocked = true;
    }
    UNLOCK_THREAD(thr);
    return blocked;
}
```

호출부 `memcached.c` `conn_nread()` (`conn_parse_cmd()`도 동일):

```c
if (c->ewouldblock) {
    c->ewouldblock = false;
    if (should_io_blocked(c)) {
        return false;      // 상태 머신 루프 탈출 → callback return
    }
}
return true;
```

**플래그만 켜두면 안 되고 `event_del()`까지 하는 이유**:
이벤트가 남아 있으면 클라이언트가 다음 요청을 보내는 순간 libevent가 `event_handler(c)`를 다시 부름.
`event_handler`는 상태를 따지지 않고 `while (c->state(c))`만 돌리므로, 상태가 `conn_write`인 채로 fsync가 끝나기도 전에 "STORED"가 전송되어 버림 (sync 모드 보장 깨짐).
그래서 libevent가 이 커넥션을 아예 부르지 않도록 이벤트 자체를 뺌.

### 5.2 블록 중에 들어온 요청은 어디에 있나

```
client ──TCP──▶ [커널 소켓 수신 버퍼] ──read()──▶ [c->rbuf (서버 메모리)] ──▶ 파싱/처리
                     ▲                        ▲
              여기에 쌓여 있음          worker가 read()를 불러야만 이동
```

- 클라이언트가 보낸 바이트는 서버 코드와 무관하게 커널이 소켓 수신 버퍼에 넣어둠
- libevent는 "이 fd에 읽을 게 있다"고 알려주기만 함. 데이터를 가져가지 않음
- `event_del` 상태에서는 아무도 감시하지 않으므로 `read()`가 호출되지 않고, **데이터는 커널 버퍼에 그대로 남음**
- 버퍼가 가득 차면 TCP flow control로 클라이언트 send가 막힘 (유실 없음)
- 서버가 별도 큐를 만든 게 아니라 **아직 읽어가지 않은 상태로 커널에 남아 있는 것**

### 5.3 깨우기: `pending_io` + notify pipe

persistence 스레드는 worker의 상태 머신을 직접 돌릴 수 없음 (그 커넥션의 worker만 돌려야 함). 그래서 두 가지만 함:

1. 공유 목록(`pending_io`)에 "이 커넥션 깨워줘"라고 넣음
2. notify pipe에 1바이트 씀 → pipe도 libevent에 등록된 fd라서 worker 입장에서는 이벤트가 됨

`thread.c` `notify_io_complete()` (persistence 스레드에서 실행):

```c
LOCK_THREAD(thr);
if (conn->current_io_wait > 0) {
    conn->current_io_wait -= 1;
    if (conn->current_io_wait == 0 && conn->io_blocked) {
        conn->io_blocked = false;
        conn->next = thr->pending_io;     // 깨울 목록에 추가
        thr->pending_io = conn;
        notify_thread = true;
    }
}
UNLOCK_THREAD(thr);

if (notify_thread) {
    write(thr->notify_send_fd, "", 1);    // worker 호출
}
```

worker 쪽: libevent가 pipe 이벤트를 감지해 `thread_libevent_process()` 호출 (`thread.c`):

```c
LOCK_THREAD(me);
conn* pending = me->pending_io;       // 깨울 목록 가져오기
me->pending_io = NULL;
UNLOCK_THREAD(me);
while (pending) {
    conn *c = pending;
    pending = pending->next;
    event_add(&c->event, 0);          // 감시 목록에 다시 등록

    while (c->state(c)) {             // 저장된 상태부터 상태 머신 재개
        /* do task */
    }
}
```

persistence 스레드가 건드리는 건 `io_blocked`, `aiostat`, `pending_io`, `current_io_wait` / `premature_io_complete` 정도. **`c->state`는 건드리지 않음.**

### 5.4 깨어난 뒤 처리 순서

이전 요청 응답 전송 → 그 다음 요청 처리.

```
깨어남 (thread_libevent_process: event_add + 상태 머신 재개)
conn_write / conn_mwrite   이전 요청의 "STORED" 전송
conn_new_cmd               reset_cmd_handler()
   ├─ c->rbytes > 0  → conn_parse_cmd   (이미 rbuf에 읽혀 있던 다음 요청을 바로 파싱)
   └─ c->rbytes == 0 → conn_waiting → conn_read
                                       read()로 커널 버퍼에 쌓인 요청을 가져와 처리
```

`memcached.c` `reset_cmd_handler()`:

```c
if (c->rbytes > 0) {
    conn_set_state(c, conn_parse_cmd);
} else {
    conn_set_state(c, conn_waiting);
}
```

다음 요청이 있을 수 있는 위치는 두 곳:

1. **이미 `c->rbuf`에 있음**: 클라이언트가 요청을 연달아 보내서(파이프라이닝) 첫 요청을 읽을 때 함께 읽혀 온 경우 → 바로 `conn_parse_cmd`
2. **아직 커널 버퍼에 있음**: 블록 중 새로 도착한 요청 → `conn_read`에서 `read()`로 가져옴

한 커넥션은 하나의 TCP 스트림이고 worker 하나가 순서대로 처리하므로 **요청 순서 = 응답 순서**가 항상 유지됨.

### 5.5 전체 과정: 상태 / 이벤트 등록 / 플래그 변화

커넥션 A에서 `lop insert`를 sync persistence로 처리하는 경우:

| 시점 | 실행 스레드 | `c->state` | 이벤트 등록 | `io_blocked` | 하는 일 |
|---|---|---|---|---|---|
| 1 | worker | `conn_waiting` | O | false | 요청 대기 |
| 2 | worker | `conn_read` → `conn_parse_cmd` | O | false | 소켓 읽기, 명령 파싱 |
| 3 | worker | `conn_nread` | O | false | value 읽기 → engine 호출 (메모리 반영, log buffer 기록, EWOULDBLOCK) |
| 4 | worker | → `conn_write` | O | false | `out_string("STORED")`로 응답 작성 |
| 5 | worker | `conn_write` (유지) | **X** | **true** | `should_io_blocked()`: `event_del`, `return false`로 callback 탈출 |
| — | worker | (A 정지) | X | true | **worker는 커넥션 B, C 처리** |
| 6 | persistence | `conn_write` | X | false | fsync 후 `notify_io_complete()`: `pending_io`에 A 추가, pipe write |
| 7 | worker | `conn_write` | **O** | false | pipe 이벤트 → `event_add`, 상태 머신 재개 |
| 8 | worker | `conn_write` → `conn_mwrite` | O | false | "STORED" 전송 |
| 9 | worker | `conn_new_cmd` → `conn_waiting` | O | false | 다음 요청 대기 |

핵심:

- **5~7 동안 `c->state`는 `conn_write`로 멈춰 있음.** "재워진 상태"라는 별도 state는 없음
- **재우기 = `event_del` + `io_blocked = true`.** 스레드 sleep이 아님
- **깨우기 = 다른 스레드가 `pending_io`에 넣고 pipe로 신호 → worker가 직접 `event_add` 후 상태 머신 재개.** 상태 머신은 처음부터 끝까지 그 worker만 돌림

### 5.6 참고: notify가 먼저 오는 경우 (premature notify)

- fsync가 매우 빨리 끝나서 5번(`should_io_blocked`)보다 6번(`notify_io_complete`)이 먼저 일어날 수 있음
- 이때 notify는 `premature_io_complete`(MULTI_NOTIFY에서는 카운트)만 기록하고 끝냄
- 이후 `should_io_blocked()`는 이를 보고 재우지 않고 `false` 반환 → 바로 `conn_write`로 진행
- 두 스레드가 같은 플래그를 다루므로 `LOCK_THREAD`로 보호

## 6. 정리: 이 구조가 단순할 수 있었던 이유

| 항목 | persistence |
|---|---|
| 별도 스레드가 하는 일 | fsync (부가 작업) |
| 응답 내용이 별도 스레드 결과에 의존? | X. 결과는 worker가 이미 계산함 |
| notify가 전달하는 정보 | "끝났다"는 신호뿐 (status는 항상 SUCCESS) |
| 깨어난 뒤 할 일 | 미리 만든 응답 전송만 |

- `conn_mwrite()`에는 `aiostat` 실패 시 응답을 바꾸는 자리가 빈 블록으로만 남아 있음 → 현재는 사실상 "끝났다" 신호만 주고받는 구조
- read 명령은 이 경로를 타지 않음 ("The read-only operation do not return ENGINE_EWOULDBLOCK")

vector search는 반대 → [Extension 비동기 처리](./extension-async-offloading.md)

- 별도 스레드가 하는 일 = 검색 자체 (명령의 본체)
- 응답 내용 = 별도 스레드의 결과 (top-k 목록)
- 블록 시점에 응답을 미리 만들 수 없음 → 깨어난 뒤 응답을 만드는 단계가 추가로 필요
