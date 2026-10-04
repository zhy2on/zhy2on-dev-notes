# Extension 비동기 처리 (vector search offloading)

extension 명령이 처리되는 구조와, 오래 걸리는 작업(vector search)을 별도 스레드로 넘기고 완료 후 응답하는 방식을 정리한 문서.
PR #1025(INTERNAL: Support offloading work asynchronously via callbacks) 리뷰를 위해 정리함.

- 선행 문서: [Worker Event Loop와 상태 머신](./worker-event-loop.md), [Persistence IO Blocking](./persistence-io-blocking.md)
- 리뷰 문서: [PR #1025 리뷰](./pr-1025-review.md)

## 1. 프로세스 구조: 모두 한 프로세스 안

- **worker 스레드는 engine이 아니라 서버 코어의 스레드**
- engine, extension은 별도 프로세스가 아니라 memcached 프로세스가 `dlopen()`으로 로드하는 공유 라이브러리(.so)

```
memcached 프로세스 (하나)
 ├─ 서버 코어 (memcached.c, thread.c)
 │    ├─ main 스레드: accept 후 커넥션을 worker에 배정
 │    └─ worker 스레드들: libevent 루프, 상태 머신, 명령 파싱/응답
 │
 ├─ engine (default_engine.so, -E 옵션으로 로드)
 │    ├─ 캐시 메모리 (hash table, collection)    ← worker가 함수 호출로 사용
 │    └─ persistence 스레드들 (log flush, group commit, checkpoint)  ← engine이 pthread_create로 생성
 │
 └─ extension (예: vector.so, -X 옵션으로 로드)
      ├─ ascii 명령 descriptor (accept / execute / pending ...)  ← worker가 함수 호출로 사용
      └─ vector search 스레드 풀  ← extension이 pthread_create로 생성
```

- `memcached.c` `load_extension()`: `dlopen()` → `memcached_extensions_initialize()` 호출 → extension이 `register_extension()`으로 명령 descriptor 등록
- persistence 스레드, extension 스레드 풀 모두 같은 프로세스의 스레드 → 메모리 공유, `notify_io_complete()` 등 서버 코어 함수 직접 호출 가능
- engine/extension 함수를 worker가 호출하면 **worker 스레드 위에서 실행됨**. 다른 스레드에서 도는 건 engine/extension이 직접 만든 스레드뿐

## 2. extension 명령이 처리되는 시점

`conn_parse_cmd`까지는 일반 명령과 동일. 명령 이름 확인 단계에서 갈라짐.

`memcached.c` `process_command()`:

```c
    if (unknown_command) {                       // 내장 명령(get, set, lop ...)에 없으면
        if (settings.extensions.ascii != NULL) {
            process_extension_command(c, tokens, ntokens);
        } else {
            out_string(c, "ERROR unknown command");
        }
    }
```

`memcached.c` `process_extension_command()`:

```c
for (cmd = settings.extensions.ascii; cmd != NULL; cmd = cmd->next) {
    if (cmd->accept(cmd->cookie, c, ntokens, tokens, &nbytes, &ptr)) {   // "이 명령 네 거야?"
        break;
    }
}
...
if (nbytes == 0) {
    cmd->execute(cmd->cookie, c, ntokens, tokens, ascii_response_handler);  // 명령 실행
    ...
}
```

- "extension으로 보낸다"가 아니라 **worker가 extension이 등록한 함수(`accept`, `execute`)를 직접 호출**
- `execute()` 안에서 검색까지 하면 그동안 worker가 묶임
- 그래서 `execute()`는 작업을 스레드 풀에 넘기기만 하고 바로 return → PR #1025의 구조

## 3. vector search 흐름 (PR #1025)

```
[worker 스레드]
conn_read → conn_parse_cmd
  process_command(): 내장 명령 아님 → process_extension_command()
    accept()  : vector 명령 확인
    execute() : 검색 작업을 스레드 풀 큐에 넣고 바로 return    ← 검색은 안 함
    pending() : 깨어나면 실행할 콜백(aiocb) 반환
    → c->aiocb 저장, c->ewouldblock = true, 상태 = conn_waking
  conn_parse_cmd로 복귀: ewouldblock 확인 → should_io_blocked()
    → event_del, return false   (커넥션 재움, worker는 다른 커넥션 처리)

[extension 스레드 풀]
  큐에서 작업 꺼내 vector search 수행
  결과를 extension 내부에 보관
  notify_io_complete(cookie)  → pending_io에 추가, pipe write

[worker 스레드]
thread_libevent_process(): event_add → 상태 머신 재개
conn_waking : aiocb(c) 호출 → 검색 결과로 응답 작성 (ascii_response_handler → dynamic buffer)
              write_and_free() → 상태 = conn_write
conn_write → conn_mwrite → conn_new_cmd
```

## 4. persistence와 비교

| | persistence (`lop insert`) | vector search (PR) |
|---|---|---|
| 명령 처리 코드 위치 | 서버 코어 + engine | 서버 코어 + extension |
| worker가 블록 전에 하는 일 | 메모리 반영 완료 + **응답 작성** | 작업을 스레드 풀에 넘김만 |
| 별도 스레드가 하는 일 | fsync (부가 작업) | **검색 자체** (명령의 본체) |
| 블록 시점의 상태 | `conn_write` (기존 상태) | `conn_waking` (**새로 추가**) |
| 깨어난 뒤 하는 일 | 만들어둔 응답 전송 | **콜백으로 응답 작성** 후 전송 |
| 블록/깨우기 메커니즘 | `ewouldblock` → `should_io_blocked` → `notify_io_complete` → `pending_io` | 동일 |

- 재우고 깨우는 메커니즘은 그대로 재사용
- 새로 생긴 건 "깨어난 뒤 응답을 만드는 단계" → `conn_waking` 상태 + `aiocb` 콜백

## 5. 큰 그림: 상태는 "책갈피"

### 5.1 개념

- 커넥션을 재울 때 worker는 `c->state`에 **"깨어나면 여기서부터 이어서 해"라는 책갈피**를 꽂아둠
- 깨어나면 worker는 이전 과정을 기억하지 않음. **책갈피가 가리키는 상태 함수부터 실행**할 뿐
- 그래서 비동기 처리 설계는 두 가지를 정하는 문제
  1. 재울 때 책갈피를 **어디에** 꽂을 것인가
  2. 그 위치에서 시작할 때 **필요한 재료가 다 있는가**

보통 명령은 아래 한 바퀴를 끊김 없이 돔:

```
conn_waiting → conn_read → conn_parse_cmd → conn_write → conn_mwrite → conn_new_cmd → (다시 conn_waiting)
                           [명령 처리+응답 작성]  [전송]
```

비동기 처리 = 이 한 바퀴 중간을 끊는 것. 방식마다 **끊는 위치(책갈피 위치)만 다름**.

### 5.2 (A) 기존 persistence: 책갈피 = `conn_write`

```
conn_parse_cmd: 명령 처리 + 응답 작성 완료 → 책갈피를 conn_write에 꽂음
   │
   ╳ 정지 ◀── persistence 스레드: fsync → notify
   │
conn_write → conn_mwrite → conn_new_cmd      (응답이 이미 있으니 전송만)
```

- `conn_write`에서 시작하려면 "응답"이라는 재료가 필요
- persistence는 결과가 블록 전에 이미 확정 → 응답을 미리 만들 수 있었음 → 이 위치가 가능

### 5.3 (B) 현재 PR: 책갈피 = `conn_waking` (새 상태)

- vector search는 블록 시점에 결과가 없음 → `conn_write`에 꽂으면 보낼 응답이 없음
- 기존 상태 중 "결과를 받아 응답을 만드는" 상태도 없음 → 그 일을 하는 상태를 새로 만듦

```
conn_parse_cmd: execute()로 작업만 넘김, pending()으로 콜백 받음 → 책갈피를 conn_waking에 꽂음
   │
   ╳ 정지 ◀── extension 스레드: search → notify
   │
conn_waking: aiocb()로 응답 작성 → conn_write → conn_mwrite → conn_new_cmd
```

→ "응답 작성" 단계를 블록 뒤로 옮기기 위해 상태 하나 + 콜백을 추가한 것

### 5.4 (C) 재실행 방식: 책갈피 = `conn_parse_cmd` (명령 처리 전)

새 상태 없이 책갈피를 명령 처리 시작 지점으로 되돌려 놓고, 깨어나면 같은 명령을 한 번 더 실행.

```
conn_parse_cmd (1회차)
   execute(): "맡겨둔 작업 없음" → 작업 enqueue, store_engine_specific(c, job), EWOULDBLOCK
   명령 라인 미소비, 책갈피를 conn_parse_cmd에 그대로 둠
   │
   ╳ 정지 ◀── extension 스레드: search, job에 결과 저장 → notify
   │
conn_parse_cmd (2회차, 같은 명령 다시 파싱)
   execute(): get_engine_specific(c) → job 결과 있음 → 응답 작성, store_engine_specific(c, NULL)
   │
conn_write → conn_mwrite → conn_new_cmd
```

- 원조 memcached 엔진 인터페이스가 EWOULDBLOCK을 다루도록 설계된 방식
- ARCUS에도 `store_engine_specific` / `get_engine_specific` 서버 API가 남아 있음 (`include/memcached/server_api.h`)
- 깨어난 뒤 필요한 재료(검색 결과)는 cookie에 붙여둔 `engine_specific`에서 찾음
- 서버 코어 입장에서는 "같은 명령을 다시 실행했더니 이번엔 바로 성공"일 뿐 → 새 상태, 콜백 불필요
- 비용
  - ARCUS ascii 경로는 파싱 시 명령 라인을 소비(`rcurr` 이동) → 재실행하려면 명령 라인 보존 필요
  - 데이터 블록이 있는 명령(`conn_nread` 경유)이면 value까지 보존해야 함
  - extension `execute()`를 "첫 호출 = 작업 등록, 두 번째 호출 = 응답" 2단계로 구현해야 함

### 5.5 (D) 구조 변경 통일

- persistence 포함 모든 비동기 명령을 "블록 전엔 결과만 저장, 깨어난 뒤 공통 상태에서 응답 작성"으로 통일
- `aiostat` 처리 자리(`conn_mwrite`의 빈 블록)도 자연스럽게 채울 수 있음
- 단, 변경 범위 최대 (6.2 참고)

### 5.6 한 장으로 비교

| | 책갈피 위치 | 깨어나서 하는 일 | 새로 필요한 것 |
|---|---|---|---|
| (A) persistence | `conn_write` | 전송 | 없음 (응답을 미리 만들 수 있어서) |
| (B) PR | `conn_waking` | 콜백으로 응답 작성 → 전송 | 새 상태 + `pending()` + 콜백 |
| (C) 재실행 | `conn_parse_cmd` | 명령 재실행 → 응답 작성 → 전송 | 명령 라인 보존 + 2단계 `execute()` |
| (D) 구조 변경 통일 | 공통 응답 작성 상태 (신규) | 저장된 결과로 응답 작성 → 전송 | 모든 비동기 명령의 응답 작성 시점 이동 |

## 6. 어떤 방식이 나은가

### 6.1 (A)와 (B)는 이미 "같은 길로 합류"하고 있음

```
(A) persistence:            ╳ 정지 ──▶                          conn_write → conn_mwrite → ...
(B) vector:                 ╳ 정지 ──▶ conn_waking(응답 작성) ──▶ conn_write → conn_mwrite → ...
                                       └── 이 단계만 추가 ──┘     └──────── 공통 합류 ────────┘
```

- 둘 다 깨어난 뒤 결국 `conn_write`로 합류해 응답을 보냄
- 차이는 "응답이 이미 있으면 바로 합류, 없으면 만들고 합류"뿐
- (A)는 "콜백이 없는 (B)"로 볼 수 있음 → 개념적으로 이미 일관된 구조

### 6.2 (D)가 품이 큰 이유

**1. persistence는 "가끔만" 블록됨**

```
블록되는 경우: sync 모드 && noreply 아님 && 아직 fsync 안 됨
그 외(대부분): 바로 응답
```

| 선택 | 결과 |
|---|---|
| 블록될 때만 응답을 나중에 작성 | 명령마다 "즉시 응답" / "나중에 응답" 두 경로가 생김. **오히려 덜 통일됨** |
| 항상 나중에 작성 | 블록 안 되는 대부분의 요청까지 결과 저장 후 작성하는 우회 경로를 거침 |

**2. 블록 전후로 결과를 들고 있어야 함 (명령마다 다름)**

| 명령 | 응답에 필요한 정보 |
|---|---|
| `set` / `lop insert` | ret, created 여부 |
| `incr` / `decr` | 계산된 숫자 값 |
| `bop insert ... getrim` | trim된 element (item 참조라 lifetime 관리까지 필요) |
| `delete`, `sop delete` 등 | ret, dropped 여부 |

`CONN_CHECK_AND_SET_EWOULDBLOCK`을 쓰는 수십 곳에서 응답 작성 코드를 떼어내고, 결과를 저장했다가 나중에 꺼내 쓰도록 바꿔야 함.

**3. persistence 입장에서 얻는 게 없음**

- 결과는 worker에서 이미 나왔고, 블록 전에 응답을 만드는 게 가장 자연스러움
- 일관성을 위해 뒤로 미루는 건 불필요한 비용. 런타임 오버헤드는 작지만(저장 후 꺼내 쓰기) 코드 구조상 비용이 큼

### 6.3 (C)가 ARCUS와 잘 맞지 않는 부분

- ascii 파싱 단계에서 명령 라인을 소비 → 보존하도록 파싱 경로 변경 필요
- 데이터 블록이 있는 extension 명령이면 value까지 보존 필요
- (B)보다 서버 코어 변경이 큼

### 6.4 결론

| 방식 | 통일성 | 변경 범위 | persistence 영향 | 추천 |
|---|---|---|---|---|
| (B) PR | 깨어난 뒤 `conn_write` 합류는 공통. 응답 작성 단계만 선택적 | 작음 | 없음 | **O** (계약 보완 전제) |
| (C) 재실행 | 기존 상태 흐름 그대로 | 중간 (명령 라인 보존) | 없음 | △ |
| (D) 전체 통일 | 형식상 최고. 단, 조건부 블록 때문에 두 경로가 남음 | 큼 (수십 곳) | 불필요한 비용 | X |

일관성은 "모든 명령이 같은 시점에 응답을 만든다"보다 **"블록/재개 메커니즘과 재개 후 합류 지점이 같다"** 수준에서 확보하는 게 맞음. (B)는 이미 그 수준을 만족함.

### 6.5 (B)를 유지할 때 명시해야 할 계약

| 항목 | 내용 |
|---|---|
| `waitfor_io_complete` 호출 시점 | 작업 스레드가 아주 빨리 끝나 `pending()` 반환 전에 notify가 올 수 있음. extension이 **작업 enqueue 전에** `waitfor_io_complete()`를 호출해야 함. `MULTI_NOTIFY_IO_COMPLETE`에서 `should_io_blocked()`는 `current_io_wait > 0`일 때만 블록. 서버가 대신 해줄 수 없는 순서라 계약으로 명시 필요 |
| 실패 처리 | `notify_io_complete(cookie, status)`의 status가 콜백에 전달되지 않음. 검색 실패 시 응답 처리 방식 필요 (`aiostat` 활용) |
| 커넥션 종료 | 작업 중 클라이언트가 끊으면 extension 스레드가 무효화된 cookie로 notify. conn 객체는 재사용되므로 다른 커넥션을 깨울 위험이 없는지 확인 필요 |
| 결과 데이터 lifetime | 콜백 이후 write 완료 시점까지 결과 데이터가 유지되어야 함 (dynamic buffer 복사 대신 iov 사용 시 특히) |
