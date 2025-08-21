---
layout: single
title:  "Libevent 라이브러리 설정하기"
date: 2025-08-21
tags: libevent
categories: 
   - Libevent
---

# Libevent 라이브러리 설정하기

Libevent는 전체 프로세스에서 공유되는 몇 가지 전역 설정을 가진다.
 이 설정들은 라이브러리 전체에 영향을 미친다.

다른 Libevent API를 호출하기 전에 반드시 이 설정들을 변경해야 한다.
 그렇지 않으면 Libevent가 일관성 없는 상태에 빠질 수 있다.

------

## Libevent의 로그 메시지

Libevent는 내부 오류와 경고를 기록할 수 있다.
 또한, 로깅 지원을 활성화하여 컴파일하면 디버깅 메시지도 기록한다.
 기본적으로 이 메시지들은 `stderr`로 출력된다.
 원한다면 직접 만든 로깅 함수를 제공하여 이 동작을 바꿀 수 있다.

------

### Interface

```c
#define EVENT_LOG_DEBUG 0
#define EVENT_LOG_MSG   1
#define EVENT_LOG_WARN  2
#define EVENT_LOG_ERR   3

/* Deprecated; see note at the end of this section */
#define _EVENT_LOG_DEBUG EVENT_LOG_DEBUG
#define _EVENT_LOG_MSG   EVENT_LOG_MSG
#define _EVENT_LOG_WARN  EVENT_LOG_WARN
#define _EVENT_LOG_ERR   EVENT_LOG_ERR

typedef void (*event_log_cb)(int severity, const char *msg);

void event_set_log_callback(event_log_cb cb);
```

Libevent의 로깅 동작을 재정의하려면 `event_log_cb` 시그니처를 따르는 함수를 작성하고,
 그 함수를 `event_set_log_callback()`에 전달한다.
 Libevent가 메시지를 기록할 필요가 있을 때마다, 작성자가 제공한 함수가 호출된다.
 다시 Libevent의 기본 동작으로 되돌리려면 `NULL`을 인자로 넣어 `event_set_log_callback()`을 호출하면 된다.

------

### Examples

```c
#include <event2/event.h>
#include <stdio.h>

static void discard_cb(int severity, const char *msg)
{
    /* This callback does nothing. */
}

static FILE *logfile = NULL;
static void write_to_file_cb(int severity, const char *msg)
{
    const char *s;
    if (!logfile)
        return;
    switch (severity) {
        case _EVENT_LOG_DEBUG: s = "debug"; break;
        case _EVENT_LOG_MSG:   s = "msg";   break;
        case _EVENT_LOG_WARN:  s = "warn";  break;
        case _EVENT_LOG_ERR:   s = "error"; break;
        default:               s = "?";     break; /* never reached */
    }
    fprintf(logfile, "[%s] %s\n", s, msg);
}

/* Turn off all logging from Libevent. */
void suppress_logging(void)
{
    event_set_log_callback(discard_cb);
}

/* Redirect all Libevent log messages to the C stdio file 'f'. */
void set_logfile(FILE *f)
{
    logfile = f;
    event_set_log_callback(write_to_file_cb);
}
```

- `suppress_logging()`: Libevent의 모든 로깅을 끈다.
- `set_logfile(FILE *f)`: Libevent의 로그 메시지를 C 표준 입출력 파일 `f`로 리다이렉트한다.

------

### NOTE

사용자가 제공한 `event_log_cb` 콜백 안에서 Libevent 함수를 호출하는 것은 안전하지 않다.
 예를 들어, 로그 콜백에서 bufferevent를 사용해 네트워크 소켓으로 경고 메시지를 전송하려 한다면,
 이상하고 진단하기 어려운 버그를 만날 가능성이 크다.
 이 제한은 Libevent의 향후 버전에서 일부 함수에 대해서는 제거될 수 있다.

일반적으로 디버그 로그는 기본적으로 꺼져 있으며, 로깅 콜백에도 전달되지 않는다.
 Libevent가 디버그 로깅을 지원하도록 빌드되었다면, 이를 수동으로 켤 수 있다.

------

### Interface

```c
#define EVENT_DBG_NONE 0
#define EVENT_DBG_ALL 0xffffffffu

void event_enable_debug_logging(ev_uint32_t which);
```

디버깅 로그는 장황하며 대부분의 경우 유용하지 않다.
 `event_enable_debug_logging(EVENT_DBG_NONE)` → 기본 동작
 `event_enable_debug_logging(EVENT_DBG_ALL)` → 모든 디버깅 로그 활성화

향후 버전에서는 더 세밀한 옵션이 지원될 수 있다.

이 함수들은 `<event2/event.h>`에 선언되어 있다.
 Libevent 1.0c에서 처음 등장했으며, `event_enable_debug_logging()`은 2.1.1-alpha에서 처음 추가되었다.

------

### COMPATIBILITY NOTE

Libevent 2.0.19-stable 이전에는 `EVENT_LOG_*` 매크로 이름이 밑줄(_)로 시작했다.
 즉, `_EVENT_LOG_DEBUG`, `_EVENT_LOG_MSG`, `_EVENT_LOG_WARN`, `_EVENT_LOG_ERR`.
 이 이름들은 더 이상 권장되지 않으며, 2.0.18-stable 이하와의 호환성 때문에만 사용해야 한다.
 앞으로는 제거될 수 있다.

------

## 치명적 오류 처리

Libevent가 복구 불가능한 내부 오류(예: 손상된 자료구조)를 감지하면,
 기본 동작은 `exit()` 또는 `abort()`를 호출하여 현재 프로세스를 종료하는 것이다.
 이러한 오류는 거의 항상 버그(사용자 코드 혹은 Libevent 내부)를 의미한다.

만약 애플리케이션이 치명적 오류를 더 우아하게 처리하도록 하고 싶다면,
 종료 대신 호출할 함수를 Libevent에 제공할 수 있다.

------

### Interface

```c
typedef void (*event_fatal_cb)(int err);
void event_set_fatal_callback(event_fatal_cb cb);
```

사용 방법:

1. 치명적 오류 시 호출할 새 함수를 정의한다.
2. `event_set_fatal_callback()`에 전달한다.
3. 이후 Libevent가 치명적 오류를 만나면 제공한 함수가 호출된다.

주의:

- 사용자 함수는 Libevent로 제어를 반환해서는 안 된다. (정의되지 않은 동작 발생)
- 함수 호출 후에는 다른 Libevent 함수를 호출하지 말아야 한다.

이 함수들은 `<event2/event.h>`에 선언되어 있으며, Libevent 2.0.3-alpha에서 처음 추가되었다.

------

## 메모리 관리

기본적으로 Libevent는 C 라이브러리의 메모리 관리 함수(`malloc`, `realloc`, `free`)를 사용한다.
 하지만 사용자가 직접 만든 메모리 관리 함수로 대체할 수 있다.
 예를 들어:

- 더 효율적인 메모리 할당기를 쓰고 싶을 때
- 메모리 누수를 잡기 위해 계측된 할당기를 쓰고 싶을 때

------

### Interface

```c
void event_set_mem_functions(void *(*malloc_fn)(size_t sz),
                             void *(*realloc_fn)(void *ptr, size_t sz),
                             void (*free_fn)(void *ptr));
```

다음은 Libevent의 할당 함수를 할당된 총 바이트 수를 계산하는 변형 함수로 대체하는 간단한 예제입니다. 실제로는 Libevent가 여러 스레드에서 실행될 때 오류를 방지하기 위해 여기에 잠금 기능을 추가하는 것이 좋습니다.

------

### Example

```c
#include <event2/event.h>
#include <sys/types.h>
#include <stdlib.h>

/* This union's purpose is to be as big as the largest of all the
 * types it contains. */
union alignment {
    size_t sz;
    void *ptr;
    double dbl;
};
/* We need to make sure that everything we return is on the right
   alignment to hold anything, including a double. */
#define ALIGNMENT sizeof(union alignment)

/* We need to do this cast-to-char* trick on our pointers to adjust
   them; doing arithmetic on a void* is not standard. */
#define OUTPTR(ptr) (((char*)ptr)+ALIGNMENT)
#define INPTR(ptr) (((char*)ptr)-ALIGNMENT)

static size_t total_allocated = 0;
static void *replacement_malloc(size_t sz)
{
    void *chunk = malloc(sz + ALIGNMENT);
    if (!chunk) return chunk;
    total_allocated += sz;
    *(size_t*)chunk = sz;
    return OUTPTR(chunk);
}
static void *replacement_realloc(void *ptr, size_t sz)
{
    size_t old_size = 0;
    if (ptr) {
        ptr = INPTR(ptr);
        old_size = *(size_t*)ptr;
    }
    ptr = realloc(ptr, sz + ALIGNMENT);
    if (!ptr)
        return NULL;
    *(size_t*)ptr = sz;
    total_allocated = total_allocated - old_size + sz;
    return OUTPTR(ptr);
}
static void replacement_free(void *ptr)
{
    ptr = INPTR(ptr);
    total_allocated -= *(size_t*)ptr;
    free(ptr);
}
void start_counting_bytes(void)
{
    event_set_mem_functions(replacement_malloc,
                            replacement_realloc,
                            replacement_free);
}
```



------

### NOTES

- 메모리 관리 함수 교체는 이후의 모든 Libevent 메모리 할당/해제 호출에 영향을 준다.
   따라서, 반드시 다른 Libevent 함수보다 먼저 호출해야 한다.
- `malloc`/`realloc`은 C 라이브러리와 동일한 메모리 정렬(alignment)을 제공해야 한다.
- `realloc(NULL, sz)`는 `malloc(sz)`로 처리해야 한다.
- `realloc(ptr, 0)`는 `free(ptr)`로 처리해야 한다.
- `free(NULL)`을 처리할 필요는 없다.
- `malloc(0)`을 처리할 필요는 없다.
- 멀티스레드 환경에서 사용하려면 교체한 메모리 관리 함수는 스레드 안전해야 한다.
- Libevent가 반환한 메모리를 해제할 때는, 교체한 `free` 함수를 써야 한다.

이 함수는 `<event2/event.h>`에 선언되어 있으며, Libevent 2.0.1-alpha에서 처음 추가되었다.

Libevent는 `event_set_mem_functions()` 비활성화 상태로 빌드될 수도 있다.
 그 경우, 이를 사용하는 프로그램은 컴파일/링크되지 않는다.
 2.0.2-alpha 이상에서는 `EVENT_SET_MEM_FUNCTIONS_IMPLEMENTED` 매크로가 정의되어 있는지 확인하여 지원 여부를 알 수 있다.

------

## 락과 스레딩

멀티스레드 프로그램에서 동일 데이터를 여러 스레드가 동시에 접근하는 것은 안전하지 않다.

Libevent 구조체는 일반적으로 세 가지 방식으로 멀티스레드에서 동작한다:

1. **단일 스레드 전용**: 동시에 여러 스레드에서 쓰면 절대 안 됨.
2. **옵션으로 락 가능**: 필요 시 객체 단위로 멀티스레드 사용 가능.
3. **항상 락 지원**: 락 지원 빌드에서는 언제나 멀티스레드 안전.

Libevent에서 락을 쓰려면, 어떤 락 함수들을 사용할지 알려줘야 한다.
 이 작업은 반드시, 스레드 간 공유될 구조체를 만드는 다른 Libevent 함수보다 먼저 해야 한다.

만약 pthreads나 Windows 기본 스레딩 코드를 쓴다면, 미리 정의된 함수로 쉽게 설정할 수 있다.

------

### Interface

```c
#ifdef WIN32
int evthread_use_windows_threads(void);
#define EVTHREAD_USE_WINDOWS_THREADS_IMPLEMENTED
#endif
#ifdef _EVENT_HAVE_PTHREADS
int evthread_use_pthreads(void);
#define EVTHREAD_USE_PTHREADS_IMPLEMENTED
#endif
```

이 함수들은 성공 시 0, 실패 시 -1을 반환한다.

------

만약 다른 스레딩 라이브러리를 사용해야 한다면, 더 많은 작업이 필요하다.
 당신은 사용하는 라이브러리를 이용하여 다음을 구현하는 함수들을 정의해야 한다:

- **Locks (락)**
  - 락 획득(locking)
  - 락 해제(unlocking)
  - 락 할당(lock allocation)
  - 락 파괴(lock destruction)
- **Conditions (조건 변수)**
  - 조건 변수 생성
  - 조건 변수 파괴
  - 조건 변수 대기(waiting)
  - 조건 변수 신호 보내기/브로드캐스트
- **Threads (스레드)**
  - 스레드 ID 감지

그 후 `evthread_set_lock_callbacks` 와 `evthread_set_id_callback` 인터페이스를 통해  이 함수들을 Libevent에 알려야 한다.

### Interface

```c
#define EVTHREAD_WRITE  0x04
#define EVTHREAD_READ   0x08
#define EVTHREAD_TRY    0x10

#define EVTHREAD_LOCKTYPE_RECURSIVE 1
#define EVTHREAD_LOCKTYPE_READWRITE 2

#define EVTHREAD_LOCK_API_VERSION 1

struct evthread_lock_callbacks {
       int lock_api_version;
       unsigned supported_locktypes;
       void *(*alloc)(unsigned locktype);
       void (*free)(void *lock, unsigned locktype);
       int (*lock)(unsigned mode, void *lock);
       int (*unlock)(unsigned mode, void *lock);
};

int evthread_set_lock_callbacks(const struct evthread_lock_callbacks *);

void evthread_set_id_callback(unsigned long (*id_fn)(void));

struct evthread_condition_callbacks {
        int condition_api_version;
        void *(*alloc_condition)(unsigned condtype);
        void (*free_condition)(void *cond);
        int (*signal_condition)(void *cond, int broadcast);
        int (*wait_condition)(void *cond, void *lock,
            const struct timeval *timeout);
};

int evthread_set_condition_callbacks(
        const struct evthread_condition_callbacks *);
```



------

`evthread_lock_callbacks` 구조체는 락 콜백과 그 기능을 설명한다.
 위에서 설명한 버전을 사용할 경우,

- `lock_api_version` 필드는 `EVTHREAD_LOCK_API_VERSION`으로 설정해야 한다.
- `supported_locktypes` 필드는 어떤 락 타입을 지원하는지를 `EVTHREAD_LOCKTYPE_*` 비트마스크로 지정해야 한다.
   (2.0.4-alpha 기준, `EVTHREAD_LOCK_RECURSIVE`는 필수이고 `EVTHREAD_LOCK_READWRITE`는 사용되지 않는다.)
- `alloc` 함수는 지정된 타입의 새 락을 반환해야 한다.
- `free` 함수는 락의 모든 리소스를 해제해야 한다.
- `lock` 함수는 지정된 모드로 락을 획득하려 시도해야 하며, 성공 시 0, 실패 시 0 이외 값을 반환해야 한다.
- `unlock` 함수는 락 해제를 시도해야 하며, 성공 시 0, 실패 시 0 이외 값을 반환해야 한다.

------

### 인식되는 락 타입 (Recognized lock types)

- **0**
   일반적인 (재귀적일 필요가 없는) 락.
- **EVTHREAD_LOCKTYPE_RECURSIVE**
   이미 락을 보유 중인 스레드가 다시 락을 요청해도 블록되지 않는 락.
   다른 스레드들은 보유 중인 스레드가 처음 획득한 횟수만큼 해제할 때까지 기다려야 한다.
- **EVTHREAD_LOCKTYPE_READWRITE**
   여러 스레드가 동시에 읽기용으로는 획득 가능하지만, 쓰기용으로는 한 번에 하나의 스레드만 가능하다.
   쓰기 락은 모든 읽기 락을 배제한다.

------

### 인식되는 락 모드 (Recognized lock modes)

- **EVTHREAD_READ**
   읽기/쓰기 락에서, 읽기용으로 락을 획득하거나 해제.
- **EVTHREAD_WRITE**
   읽기/쓰기 락에서, 쓰기용으로 락을 획득하거나 해제.
- **EVTHREAD_TRY**
   락을 즉시 획득할 수 있는 경우에만 획득.

------

### 스레드 ID 콜백

`id_fn` 인자는 현재 호출 중인 스레드를 식별하는 `unsigned long` 값을 반환해야 한다.
 같은 스레드에서는 항상 같은 값을 반환해야 하며, 동시에 실행 중인 서로 다른 두 스레드에서는 같은 값을 반환해서는 안 된다.

------

### 조건 변수 콜백

`evthread_condition_callbacks` 구조체는 조건 변수 관련 콜백을 설명한다.
 위에서 설명한 버전을 사용할 경우,

- `condition_api_version` 필드는 `EVTHREAD_CONDITION_API_VERSION`으로 설정해야 한다.
- `alloc_condition` 함수는 새 조건 변수를 반환해야 한다. 인자로 0을 받는다.
- `free_condition` 함수는 조건 변수가 사용하는 메모리와 리소스를 해제해야 한다.
- `wait_condition` 함수는 3개의 인자를 받는다:
  - `alloc_condition`으로 생성된 조건 변수
  - 사용자가 제공한 `evthread_lock_callbacks.alloc`으로 만든 락
  - (선택적) 타임아웃
     함수가 호출될 때는 락이 잡혀 있어야 하며, 이 함수는 락을 해제하고 조건이 시그널 되거나 타임아웃이 될 때까지 기다려야 한다.
     반환 값은 다음과 같다:
  - 오류 시 -1
  - 조건 시그널 시 0
  - 타임아웃 시 1
     반환하기 전에 반드시 다시 락을 보유한 상태여야 한다.
- `signal_condition` 함수는 조건 대기 중인 스레드를 깨워야 한다.
  - `broadcast` 인자가 거짓이면, 대기 중인 하나의 스레드만 깨운다.
  - `broadcast` 인자가 참이면, 대기 중인 모든 스레드를 깨운다.
     이 함수는 조건과 연결된 락을 보유한 상태에서만 호출된다.

------

조건 변수에 대한 더 많은 정보는 pthreads의 `pthread_cond_*` 함수들,
 또는 Windows의 `CONDITION_VARIABLE` 함수 문서를 참고하라.

------

### Examples

이 함수들을 어떻게 사용하는지는 Libevent 소스 배포판의 `evthread_pthread.c` 와 `evthread_win32.c`를 참고하라.

------

이 섹션의 함수들은 `<event2/thread.h>`에 선언되어 있다.
 대부분은 Libevent 2.0.4-alpha에서 처음 등장했다.
 Libevent 2.0.1-alpha ~ 2.0.3-alpha 버전은 예전 방식의 인터페이스를 사용했다.
 `event_use_pthreads()` 함수를 사용하려면 프로그램을 `event_pthreads` 라이브러리와 링크해야 한다.

조건 변수 함수들은 Libevent 2.0.7-rc에서 새로 추가되었으며,
 이는 해결 불가능했던 교착 상태 문제를 해결하기 위해 도입되었다.

Libevent는 락 지원을 비활성화한 상태로 빌드될 수도 있다.
 그 경우, 위 스레드 관련 함수들을 사용하는 프로그램은 실행되지 않는다.

------

## 락 사용 디버깅 (Debugging lock usage)

Libevent는 락 사용을 디버깅할 수 있는 선택적 "락 디버깅" 기능을 제공한다.
 이는 락 호출을 감싸서 전형적인 락 오류를 잡아낸다.
 예:

- 보유하지 않은 락을 해제하려는 경우
- 재귀적이지 않은 락을 다시 획득하려는 경우

이런 오류가 발생하면, Libevent는 assertion failure로 종료한다.

------

### Interface

``` c
void evthread_enable_lock_debugging(void);
#define evthread_enable_lock_debuging() evthread_enable_lock_debugging()
```

------

### Note

이 함수는 반드시 어떤 락이 생성되거나 사용되기 전에 호출해야 한다.
 안전하게 사용하려면, 스레드 함수를 설정한 직후에 호출하라.

이 함수는 Libevent 2.0.4-alpha에서 잘못된 이름(`evthread_enable_lock_debuging`)으로 처음 추가되었다.
 Libevent 2.1.2-alpha에서 올바른 철자(`evthread_enable_lock_debugging`)로 수정되었으며, 현재는 두 이름 모두 지원된다.

------

## 이벤트 사용 디버깅 (Debugging event usage)

Libevent는 이벤트 사용에서 흔히 발생하는 오류들을 감지하고 보고할 수 있다.
 예:

- 초기화되지 않은 `struct event`를 초기화된 것처럼 다루는 경우
- 대기(pending) 중인 `struct event`를 다시 초기화하려는 경우

이벤트 초기화를 추적하려면 Libevent가 추가적인 메모리와 CPU를 사용해야 하므로,
 프로그램을 실제로 디버깅할 때만 디버그 모드를 켜야 한다.

------

### Interface

```c
void event_enable_debug_mode(void);
```

------

이 함수는 반드시 어떤 `event_base`가 생성되기 전에만 호출할 수 있다.

디버그 모드에서 `event_assign()`으로 많은 이벤트를 만들면 메모리가 부족해질 수 있다.
 이는 Libevent가 그 이벤트가 더 이상 사용되지 않는 시점을 알 수 없기 때문이다.
 (`event_new()`로 만든 이벤트는 `event_free()`를 호출하면 Libevent가 추적을 중단할 수 있다.)
 디버깅 중에 메모리 부족을 피하려면, 더 이상 사용하지 않는 `event_assign()` 이벤트를 명시적으로 알려야 한다.

------

### Interface

```c
void event_debug_unassign(struct event *ev);
```

------

`event_debug_unassign()`은 디버깅이 활성화되지 않았을 때는 아무 효과가 없다.

------

### Example

```c
#include <event2/event.h>
#include <event2/event_struct.h>

#include <stdlib.h>

void cb(evutil_socket_t fd, short what, void *ptr)
{
    /* We pass 'NULL' as the callback pointer for the heap allocated
     * event, and we pass the event itself as the callback pointer
     * for the stack-allocated event. */
    struct event *ev = ptr;

    if (ev)
        event_debug_unassign(ev);
}

/* Here's a simple mainloop that waits until fd1 and fd2 are both
 * ready to read. */
void mainloop(evutil_socket_t fd1, evutil_socket_t fd2, int debug_mode)
{
    struct event_base *base;
    struct event event_on_stack, *event_on_heap;

    if (debug_mode)
       event_enable_debug_mode();

    base = event_base_new();

    event_on_heap = event_new(base, fd1, EV_READ, cb, NULL);
    event_assign(&event_on_stack, base, fd2, EV_READ, cb, &event_on_stack);

    event_add(event_on_heap, NULL);
    event_add(&event_on_stack, NULL);

    event_base_dispatch(base);

    event_free(event_on_heap);
    event_base_free(base);
}
```



------

자세한 이벤트 디버깅은 컴파일 타임에 `-DUSE_DEBUG` 플래그로만 활성화할 수 있다.
 이 플래그를 켜고 컴파일된 프로그램은 백엔드의 저수준 활동을 매우 장황하게 기록한다.

로그에는 다음과 같은 정보가 포함된다 (하지만 이들로 제한되지 않는다):

- 이벤트 추가
- 이벤트 삭제
- 플랫폼 특정 이벤트 알림 정보

이 기능은 API 호출로 켜거나 끌 수 없으며, 개발자 빌드에서만 사용해야 한다.

이 디버깅 기능들은 Libevent 2.0.4-alpha에서 추가되었다.

### Libevent 버전 감지하기

Libevent의 새로운 버전은 기능을 추가하고 버그를 제거할 수 있다.
 때때로 다음과 같은 이유로 Libevent 버전을 감지하고 싶을 때가 있다:

- 설치된 Libevent 버전이 프로그램을 빌드하기에 충분히 좋은지 감지하기 위해.
- 디버깅을 위해 Libevent 버전을 표시하기 위해.
- Libevent 버전을 감지하여 사용자에게 버그를 경고하거나 우회하기 위해.

------

### Interface

```c
#define LIBEVENT_VERSION_NUMBER 0x02000300
#define LIBEVENT_VERSION "2.0.3-alpha"
const char *event_get_version(void);
ev_uint32_t event_get_version_number(void);
```

- 매크로는 **컴파일 시점**의 Libevent 버전을 제공한다.
- 함수는 **실행 시점**의 Libevent 버전을 반환한다.

주의할 점은, 프로그램이 Libevent를 동적 링크했다면 두 버전이 서로 다를 수도 있다는 것이다.

Libevent 버전은 두 가지 형식으로 얻을 수 있다:

- 사용자에게 표시하기 좋은 문자열 형식
- 숫자 비교에 적합한 4바이트 정수 형식

정수 형식은 다음과 같이 바이트를 사용한다:

- 최상위 바이트: major 버전
- 두 번째 바이트: minor 버전
- 세 번째 바이트: patch 버전
- 최하위 바이트: 릴리스 상태 (0 = 릴리스, 0이 아니면 특정 릴리스 이후의 개발 버전)

예를 들어, 릴리스된 Libevent 2.0.1-alpha 는 버전 번호 [02 00 01 00], 즉 0x02000100을 가진다.
 2.0.1-alpha와 2.0.2-alpha 사이의 개발 버전은 [02 00 01 08], 즉 0x02000108일 수 있다.

------

### 예제: 컴파일 시점 검사

```c
#include <event2/event.h>

#if !defined(LIBEVENT_VERSION_NUMBER) || LIBEVENT_VERSION_NUMBER < 0x02000100
#error "This version of Libevent is not supported; Get 2.0.1-alpha or later."
#endif
```

```c
int make_sandwich(void)
{
    /* 예를 들어 Libevent 6.0.5에 새로운 함수가 추가되었다고 가정 */
#if LIBEVENT_VERSION_NUMBER >= 0x06000500
    evutil_make_me_a_sandwich();
    return 0;
#else
    return -1;
#endif
}

```

------

### 예제: 실행 시점 검사

```c
#include <event2/event.h>
#include <string.h>

int check_for_old_version(void)
{
    const char *v = event_get_version();
    /* Libevent 2.0 이전에는 이렇게밖에 못한다 */
    if (!strncmp(v, "0.", 2) ||
        !strncmp(v, "1.1", 3) ||
        !strncmp(v, "1.2", 3) ||
        !strncmp(v, "1.3", 3)) {

        printf("사용 중인 Libevent 버전이 매우 오래되었습니다. 버그가 발생하면 업그레이드를 고려하세요.\n");
        return -1;
    } else {
        printf("현재 Libevent 버전: %s\n", v);
        return 0;
    }
}
```

```c
int check_version_match(void)
{
    ev_uint32_t v_compile, v_run;
    v_compile = LIBEVENT_VERSION_NUMBER;
    v_run = event_get_version_number();
    if ((v_compile & 0xffff0000) != (v_run & 0xffff0000)) {
        printf("실행 중인 Libevent 버전(%s)이 빌드 시점의 버전(%s)과 크게 다릅니다.\n",
               event_get_version(), LIBEVENT_VERSION);
        return -1;
    }
    return 0;
}

```

- 이 매크로와 함수들은 `<event2/event.h>`에 정의되어 있다.
- `event_get_version()` 함수는 Libevent 1.0c에서 처음 등장했다.
- 나머지는 Libevent 2.0.1-alpha에서 처음 도입되었다.

------

### 전역 Libevent 구조체 해제하기

Libevent로 할당했던 객체들을 모두 해제했더라도, 몇몇 전역적으로 할당된 구조체들이 남아있을 수 있다.
 보통 이는 문제가 되지 않는다. 프로세스가 종료되면 모두 정리되기 때문이다.

하지만 일부 디버깅 도구들은 이를 **메모리 누수**로 오인할 수 있다.
 Libevent가 내부의 전역 데이터 구조를 모두 해제했는지 확실히 하려면 다음 함수를 호출할 수 있다:

------

### Interface

```c
void libevent_global_shutdown(void);
```

- 이 함수는 Libevent 함수가 반환한 구조체들을 해제하지는 않는다.
- 종료 전에 모든 `event`, `event_base`, `bufferevent` 등을 직접 해제해야 한다.

`libevent_global_shutdown()`을 호출하면 이후의 다른 Libevent 함수들은 예측 불가능하게 동작한다.
 따라서 이 함수는 프로그램에서 호출하는 **마지막 Libevent 함수**로만 사용해야 한다.

예외적으로, `libevent_global_shutdown()`은 **idempotent**하다.
 즉, 이미 호출된 후에 다시 호출해도 문제는 없다.

이 함수는 `<event2/event.h>`에 선언되어 있으며, Libevent 2.1.1-alpha에서 도입되었다.