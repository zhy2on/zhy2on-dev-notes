# PR #1025 리뷰: INTERNAL: Support offloading work asynchronously via callbacks

- PR: naver/arcus-memcached#1025 (ing-eoking:aio → develop, 커밋 1955575)
- 관련 이슈: jam2in/arcus-works#867 `[Cache Server] vector search 전용의 처리 스레드 필요성 검토`
- 배경 지식: [Worker Event Loop와 상태 머신](./worker-event-loop.md), [Persistence IO Blocking](./persistence-io-blocking.md), [Extension 비동기 처리](./extension-async-offloading.md)

## 1. 배경

### 1.1 목표

- ARCUS에 USearch(HNSW) 기반 vector search 기능을 추가하려 함
- engine에 새 API를 추가하지 않고, **extension(`EXTENSION_ASCII_PROTOCOL_DESCRIPTOR`)으로 명령어를 추가**하는 방식
- vector search는 처리 시간이 길어질 수 있음

### 1.2 문제

- ARCUS memcached는 별도의 I/O 스레드 없이, worker 스레드가 요청 수신 → 명령 처리 → 응답까지 모두 담당
- 하나의 worker 스레드는 여러 커넥션을 libevent로 처리하므로, long query가 worker를 점유하면 **같은 worker에 붙은 다른 커넥션의 가벼운 요청(get/set 등)까지 지연되어 timeout 발생**
- 따라서 목적은 "검색 속도 향상"이 아니라 **"long query가 다른 요청을 막지 않도록 하는 것"** (jhpark816 코멘트)

### 1.3 이슈에서 합의된 방향

- 오래 걸리는 작업(vector search)은 extension 내부의 전용 스레드 풀로 넘겨 처리(offloading)
- worker 스레드는 해당 커넥션을 event loop에서 잠시 제외(블록)하고 다른 커넥션 처리를 계속함
- 작업이 끝나면 `notify_io_complete()`로 worker를 깨우고, worker가 결과를 클라이언트에게 응답
- 클라이언트 측에서는 vector search 전용 ArcusClient를 분리해 사용하도록 권고
- 기존에 replication async 복제, persistence group commit 등이 이미 `waitfor_io_complete()` / `notify_io_complete()` 기반으로 같은 형태의 동작을 하고 있음

### 1.4 기존 비동기 처리 구조 (persistence 예시)

자세한 내용은 [Persistence IO Blocking](./persistence-io-blocking.md).

1. engine 명령 처리 중 cmdlog fsync를 기다려야 하면 `cmdlog_waiter_end()`에서 `waitfor_io_complete(cookie)` 호출 후 `ENGINE_EWOULDBLOCK` 반환
2. 서버 코어는 `CONN_CHECK_AND_SET_EWOULDBLOCK(ret, c)`로 `c->ewouldblock = true` 설정 후 ret을 `ENGINE_SUCCESS`로 바꾸어 **응답 메시지를 미리 구성** (`out_string()` → `conn_write`)
3. `conn_parse_cmd()` / `conn_nread()`에서 `c->ewouldblock`을 확인하고 `should_io_blocked()` 호출 → `event_del()`로 커넥션을 event loop에서 제외, `io_blocked = true`
4. group commit 스레드가 fsync 완료 후 `notify_io_complete(cookie, ENGINE_SUCCESS)` 호출 → 커넥션을 해당 worker의 `pending_io` 리스트에 넣고 notify pipe로 worker를 깨움
5. worker가 `pending_io`의 커넥션을 `event_add()`로 다시 등록하고, **블록 직전 상태(`conn_write`)부터** state machine을 이어서 실행 → 이미 구성된 응답을 전송

즉 기존 구조는 "응답은 블록 전에 이미 만들어 두고, 깨어나면 전송만 이어서 한다"는 형태임.

## 2. PR 내용

### 2.1 개요

- 참고용(PoC) PR로, extension 명령의 작업을 다른 스레드에 넘기고 완료 후 응답하는 구조를 보여줌
- 기존 구조와의 차이: **응답을 블록 전에 만들 수 없으므로**, 깨어난 뒤 실행할 콜백(`aiocb`)을 등록해 두고 깨어났을 때 그 콜백이 응답을 구성함
- 변경 파일: `include/memcached/extension.h`, `include/memcached/types.h`, `memcached.c`, `memcached.h` (+49 / -2)

### 2.2 변경 사항

#### (1) 콜백 타입 추가 — `include/memcached/types.h`

```c
typedef void (*AIO_CALLBACK)(const void *cookie);
```

#### (2) extension descriptor에 `pending` 메소드 추가 — `include/memcached/extension.h`

```c
AIO_CALLBACK (*pending)(const void *cmd_cookie, const void *cookie);
```

- `execute()`가 true를 반환한 직후, 같은 커넥션에 대해 호출됨
- `NULL` 반환: 응답이 이미 response buffer(dynamic buffer)에 있고 명령이 끝났음을 의미 (기존 동작과 동일, 멤버를 NULL로 두면 하위 호환)
- non-NULL 반환: extension이 나중에 `notify_io_complete()`로 깨울 때 코어가 호출할 콜백. 그 전까지 커넥션은 `conn_waking` 상태로 event loop에서 빠져 있음

#### (3) 커넥션 구조체에 `aiocb` 추가 — `memcached.h`, `conn_new()`

```c
ENGINE_ERROR_CODE aiostat;
AIO_CALLBACK aiocb;   /* 추가 */
bool ewouldblock;
```

- `conn_new()`에서 `c->aiocb = NULL`로 초기화

#### (4) extension 명령 처리 후 블록 진입 — `complete_nread_ascii()`, `process_extension_command()`

```c
if (!cmd->execute(cmd->cookie, c, ntokens, tokens, ascii_response_handler)) {
    conn_set_state(c, conn_closing);
} else {
    if (cmd->pending != NULL && (c->aiocb = cmd->pending(cmd->cookie, c)) != NULL) {
        c->ewouldblock = true;
        conn_set_state(c, conn_waking);
    } else if (c->dynamic_buffer.buffer != NULL) {
        write_and_free(c, c->dynamic_buffer.buffer, c->dynamic_buffer.offset);
        c->dynamic_buffer.buffer = NULL;
    } else {
        ...
    }
}
```

- `pending()`이 콜백을 반환하면 `ewouldblock = true`, 상태를 `conn_waking`으로 설정
- 이후 `conn_parse_cmd()` / `conn_nread()`의 기존 `ewouldblock` 처리 경로를 그대로 타서 `should_io_blocked()` → event loop에서 제외

#### (5) 새 상태 `conn_waking` — `memcached.c`

```c
bool conn_waking(conn *c)
{
    if (c->aiocb != NULL) {
        c->aiocb(c);
        c->aiocb = NULL;
        if (c->dynamic_buffer.buffer != NULL) {
            write_and_free(c, c->dynamic_buffer.buffer, c->dynamic_buffer.offset);
            c->dynamic_buffer.buffer = NULL;
        }
        return true;
    }

    conn_set_state(c, conn_waiting);
    return false;
}
```

- `notify_io_complete()`로 깨어나면 worker가 `conn_waking`부터 state machine을 재개
- `aiocb`를 호출해 응답을 구성하고, dynamic buffer가 있으면 `write_and_free()`로 `conn_write` 전환
- `state_text()`에 `"conn_waking"` 추가

### 2.3 전체 흐름

```
[worker]                                   [extension thread pool]
execute()  ── 작업 enqueue ───────────────▶ vector search 수행
pending()  → aiocb 반환
c->ewouldblock = true, conn_waking
should_io_blocked() → event_del()
  (다른 커넥션 계속 처리)
                                           작업 완료
                         ◀──────────────── notify_io_complete(cookie)
pending_io → event_add()
conn_waking: aiocb(c)
  → ascii_response_handler()로 dynamic buffer에 응답 구성
  → write_and_free() → conn_write → 전송
```

### 2.4 응답 구성 방식 (PR 코멘트 내용)

- 콜백 내부에서 `ascii_response_handler()`를 호출해 응답을 dynamic buffer에 **복사**하여 구성

```c
/* 예시 */
void aiocb() {
  ascii_response_handler(c, strlen(value), value);
  ascii_response_handler(c, 2, "\r\n");
  ascii_response_handler(c, 5, "END\r\n");
}
```

- jhpark816: value 크기가 크면 비효율적이므로, element 포인터 배열을 받아 직접 write(iov)하는 방식이 가능한지 질문
- ing-eoking: 콜백이 element 같은 engine 아이템만 다룬다면 가능. 다만 extension이 반환하는 데이터가 engine 아이템이 아닌 extension 내부 데이터라면, write 완료 시점까지 그 데이터의 lifetime이 보장되어야 함

## 3. 리뷰 포인트

### 3.1 dynamic buffer 복사 대신 포인터 배열(iov)로 응답을 전달하는 방법

(작성 예정)

### 3.2 기존 비동기 구조와의 일관성

방식별 상태 흐름 그림과 상세 분석은 [Extension 비동기 처리 5~6절](./extension-async-offloading.md#5-큰-그림-상태는-책갈피).

#### 공통 전제

- 재우기/깨우기 메커니즘(`ewouldblock` → `should_io_blocked` → `notify_io_complete` → `pending_io` → 저장된 상태부터 재개)은 PR도 그대로 재사용함
- 차이는 **"어느 상태에서 멈추고(책갈피 위치), 깨어나서 무엇을 하는가"**
- 기존(persistence)은 응답을 블록 전에 만들 수 있어서 `conn_write`에서 멈추면 충분했음
- vector search는 블록 시점에 결과가 없으므로 기존 상태 중 그대로 이어갈 지점이 없음

#### 방식 비교

| | (A) persistence | (B) PR: 상태 추가 | (C) 재실행 | (D) 구조 변경 통일 |
|---|---|---|---|---|
| 멈추는 상태 | `conn_write` | `conn_waking` (신규) | `conn_parse_cmd` | 공통 응답 작성 상태 (신규) |
| 깨어나서 하는 일 | 전송 | 콜백으로 응답 작성 → 전송 | 명령 재실행 → 응답 작성 → 전송 | 저장된 결과로 응답 작성 → 전송 |
| 새 상태 | - | 필요 | 불필요 | 필요 |
| 기존 명령 코드 영향 | - | 없음 | 명령 라인 보존 처리 필요 | 모든 변경 명령의 응답 작성 시점 이동 |
| extension 구현 부담 | - | `pending()` + 콜백 | `execute()` 2단계 구현 | - |

#### 의견

- (B) 유지 추천
  - (A)와 (B)는 깨어난 뒤 `conn_write`로 합류한다는 점에서 이미 일관됨. (A)는 "콜백이 없는 (B)"
  - 일관성은 "모든 명령이 같은 시점에 응답 작성"보다 "블록/재개 메커니즘과 재개 후 합류 지점이 같다" 수준에서 확보하는 게 적절
- (D)는 비용 대비 이득이 작음
  - persistence는 조건부로만 블록 → 통일해도 "즉시 응답 / 나중에 응답" 두 경로가 남음
  - 명령마다 응답에 필요한 결과(created, incr 값, getrim element 등)를 conn에 보관해야 함
  - `CONN_CHECK_AND_SET_EWOULDBLOCK` 사용처 수십 곳 변경, persistence 입장에서 기능상 이득 없음
- (C)는 ascii 경로의 명령 라인(데이터 블록 포함) 보존이 필요해 (B)보다 서버 코어 변경이 큼
- 리뷰 질문: (C)처럼 기존 "깨어나면 저장된 상태부터 재개" 원칙과 기존 API(`store/get_engine_specific`)로 처리하는 방식 대비, 새 상태와 `pending()` 메소드를 추가한 이유

#### (B)를 유지할 때 확인/명시할 점

- `waitfor_io_complete()` 호출 계약: extension이 작업 enqueue **전에** 호출해야 함 (`MULTI_NOTIFY_IO_COMPLETE`에서 `should_io_blocked()`는 `current_io_wait > 0`일 때만 블록, premature notify 대비)
- 실패 처리: `notify_io_complete()`의 status(`aiostat`)를 콜백에 전달해 실패 응답 처리
- 커넥션 종료: 작업 중 커넥션이 닫히면 extension 스레드가 무효화된 cookie로 notify하게 됨. conn 객체 재사용 시 다른 커넥션을 깨울 위험 확인
- 결과 데이터 lifetime: 콜백 이후 write 완료까지 유지 보장 (3.1과 연결)
