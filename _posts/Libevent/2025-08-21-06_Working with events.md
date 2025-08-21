---
layout: single
title:  "Working with events"
date: 2025-08-21
tags: libevent
categories: 
   - Libevent
---

## Working with events

Libevent의 기본 동작 단위는 **이벤트(event)** 이다. 각 이벤트는 다음과 같은 조건 집합을 표현한다:

- 파일 디스크립터에서 **읽기/쓰기 가능** 상태가 됨.
- (엣지 트리거 IO에서만) 파일 디스크립터가 **읽기/쓰기 가능 상태로 변함**.
- **타임아웃**이 만료됨.
- **시그널**이 발생함.
- **사용자 트리거**로 이벤트가 활성화됨.

이벤트는 유사한 생명주기를 가진다. Libevent 함수로 이벤트를 설정하고 특정 event base에 **연결(associate)** 하면, 이벤트는 **초기화**된다. 이 시점에서 이벤트를 **추가(add)** 하면 해당 base 안에서 **대기(pending)** 상태가 된다. 이벤트가 대기 중일 때, 이벤트를 트리거할 조건(예: fd 상태 변화, 타임아웃 만료)이 발생하면 이벤트가 **활성(active)** 상태가 되고, (사용자가 제공한) **콜백 함수**가 실행된다. 이벤트가 **persistent(지속)** 로 설정되어 있으면, 활성화 후에도 계속 대기 상태로 남는다. persistent가 아니면, 콜백 실행 직전에 대기 상태를 멈춘다. 대기 중인 이벤트를 **삭제(delete)** 하면 비대기 상태가 되고, 비대기 이벤트를 다시 **추가(add)** 하면 다시 대기 상태로 만든다.

------

## Constructing event objects

새로운 이벤트를 만들려면 `event_new()` 인터페이스를 사용한다.

### Interface

```c
#define EV_TIMEOUT      0x01
#define EV_READ         0x02
#define EV_WRITE        0x04
#define EV_SIGNAL       0x08
#define EV_PERSIST      0x10
#define EV_ET           0x20

typedef void (*event_callback_fn)(evutil_socket_t, short, void *);

struct event *event_new(struct event_base *base, evutil_socket_t fd,
    short what, event_callback_fn cb,
    void *arg);

void event_free(struct event *event);
```

`event_new()`는 `base`에서 사용할 새 이벤트를 **할당/구성**하려 시도한다. `what` 인자는 위에 나열된 플래그들의 **조합**이다(구체적 의미는 아래에 설명). `fd`가 0 이상이면, 읽기/쓰기 이벤트 감시 대상이 되는 파일(디스크립터)이다. 이벤트가 활성화되면 Libevent는 제공된 콜백 `cb`를 호출하며, 인자로 **fd**, 트리거된 모든 이벤트의 **비트필드(what)**, 그리고 생성 시 전달한 **arg** 값을 넘긴다.

내부 오류나 잘못된 인자일 경우 `event_new()`는 **NULL**을 반환한다.

모든 새 이벤트는 **초기화되었지만 비대기** 상태다. 이벤트를 **대기 상태**로 만들려면 `event_add()`(아래 문서) 를 호출한다.

이벤트를 해제하려면 `event_free()`를 호출한다. 대기 중이거나 활성 상태인 이벤트에 대해 `event_free()`를 호출해도 안전하다: 해제 전에 이벤트를 **비대기/비활성**으로 만든다.

### Example

```c
#include <event2/event.h>

void cb_func(evutil_socket_t fd, short what, void *arg)
{
        const char *data = arg;
        printf("Got an event on socket %d:%s%s%s%s [%s]",
            (int) fd,
            (what&EV_TIMEOUT) ? " timeout" : "",
            (what&EV_READ)    ? " read" : "",
            (what&EV_WRITE)   ? " write" : "",
            (what&EV_SIGNAL)  ? " signal" : "",
            data);
}

void main_loop(evutil_socket_t fd1, evutil_socket_t fd2)
{
        struct event *ev1, *ev2;
        struct timeval five_seconds = {5,0};
        struct event_base *base = event_base_new();

        /* 호출자는 이미 fd1, fd2를 적절히 준비했고, 논블로킹으로 설정했다. */

        ev1 = event_new(base, fd1, EV_TIMEOUT|EV_READ|EV_PERSIST, cb_func,
           (char*)"Reading event");
        ev2 = event_new(base, fd2, EV_WRITE|EV_PERSIST, cb_func,
           (char*)"Writing event");

        event_add(ev1, &five_seconds);
        event_add(ev2, NULL);
        event_base_dispatch(base);
}
```

위 함수들은 `<event2/event.h>`에 정의되어 있으며, Libevent 2.0.1-alpha에서 처음 등장했다. `event_callback_fn` 타입 별칭은 Libevent 2.0.4-alpha에서 추가되었다.

------

## The event flags

**EV_TIMEOUT**
 타임아웃이 경과하면 이벤트가 활성화됨을 나타낸다.
 이 플래그는 **이벤트 생성 시에는 무시**된다: 이벤트를 추가할 때 타임아웃을 줄 수도 있고, 주지 않을 수도 있다. 타임아웃 발생 시 콜백의 `what` 인자에 **설정되어 온다**.

**EV_READ**
 제공된 fd가 **읽기 준비**가 되었을 때 이벤트가 활성화됨을 나타낸다.

**EV_WRITE**
 제공된 fd가 **쓰기 준비**가 되었을 때 이벤트가 활성화됨을 나타낸다.

**EV_SIGNAL**
 시그널 감지를 구현하는 데 사용. 아래 “Constructing signal events” 참조.

**EV_PERSIST**
 이벤트가 **지속(persistent)** 임을 나타낸다. 아래 “About Event Persistence” 참조.

**EV_ET**
 하부 `event_base` 백엔드가 **엣지 트리거**를 지원한다면, 이벤트를 엣지 트리거로 처리하도록 지시한다. `EV_READ`/`EV_WRITE`의 의미에 영향을 준다.

Libevent 2.0.1-alpha 이후, **동일 조건**에 대해 동시에 여러 이벤트가 대기(pending)할 수 있다. 예를 들어, 특정 fd의 **읽기 가능**을 기다리는 이벤트가 둘 이상 있어도 된다. 이때 콜백 실행 **순서는 정의되지 않는다**.

이 플래그들은 `<event2/event.h>`에 정의되어 있다. `EV_ET`는 Libevent 2.0.1-alpha에서 도입되었고, 나머지는 1.0 이전부터 존재했다.

------

## About Event Persistence

기본적으로, 대기 중인 이벤트가 활성화되면(예: fd가 읽기/쓰기 가능, 혹은 타임아웃 만료), **콜백 실행 직전**에 **비대기** 상태가 된다. 따라서 콜백 내부에서 다시 `event_add()`를 호출해 **재대기** 시킬 수 있다.

하지만 이벤트에 `EV_PERSIST`가 설정되어 있으면, 이벤트는 **지속적**이다. 즉, 콜백이 실행되어 활성화되더라도 **여전히 대기 상태**로 남는다. 콜백 내부에서 비대기 상태로 만들고 싶다면 `event_del()`을 호출하면 된다.

**지속 이벤트의 타임아웃**은 해당 이벤트의 콜백이 **실행될 때마다** **리셋**된다. 예를 들어, `EV_READ|EV_PERSIST` 플래그와 **5초 타임아웃**을 가진 이벤트는 다음과 같이 활성화된다:

- 소켓이 **읽기 준비**가 될 때마다.
- 마지막으로 활성화된 시점으로부터 **5초**가 지났을 때마다.

------

## Creating an event as its own callback argument

자주 필요한 패턴으로, **자기 자신(struct event*)을 콜백 인자**로 받는 이벤트를 만들고 싶을 수 있다. 하지만 이벤트가 아직 존재하지 않으므로 `event_new()`에 곧장 그 포인터를 넘길 수 없다. 이 문제를 해결하기 위해 `event_self_cbarg()`를 사용할 수 있다.

### Interface

```c
void *event_self_cbarg();
```

`event_self_cbarg()`는 **“매직” 포인터**를 반환한다. 이 포인터를 이벤트의 콜백 인자로 전달하면, `event_new()`가 **해당 이벤트 자신을 콜백 인자**로 받도록 만들어준다.

### Example

```c
#include <event2/event.h>

static int n_calls = 0;

void cb_func(evutil_socket_t fd, short what, void *arg)
{
    struct event *me = arg;

    printf("cb_func called %d times so far.\n", ++n_calls);

    if (n_calls > 100)
       event_del(me);
}

void run(struct event_base *base)
{
    struct timeval one_sec = { 1, 0 };
    struct event *ev;
    /* 100번 호출되는 반복 타이머를 설정한다. */
    ev = event_new(base, -1, EV_PERSIST, cb_func, event_self_cbarg());
    event_add(ev, &one_sec);
    event_base_dispatch(base);
}
```

이 함수는 `event_new()`, `evtimer_new()`, `evsignal_new()`, `event_assign()`, `evtimer_assign()`, `evsignal_assign()`과 함께 사용할 수 있다. **이벤트가 아닌 다른 객체**의 콜백 인자로는 사용할 수 없다.

`event_self_cbarg()`는 Libevent 2.1.1-alpha에서 도입되었다.

------

## Timeout-only events

가독성을 위해, **순수 타임아웃 이벤트**를 만들고 다루는 `evtimer_` 접두사 매크로들이 제공된다. 기능상의 이점은 없고, 코드가 **더 명확**해진다.

### Interface

```c
#define evtimer_new(base, callback, arg) \
    event_new((base), -1, 0, (callback), (arg))
#define evtimer_add(ev, tv) \
    event_add((ev),(tv))
#define evtimer_del(ev) \
    event_del(ev)
#define evtimer_pending(ev, tv_out) \
    event_pending((ev), EV_TIMEOUT, (tv_out))
```

이 매크로들은 Libevent 0.6부터 있었으며, `evtimer_new()`는 2.0.1-alpha에서 처음 등장했다.

------

## Constructing signal events

Libevent는 POSIX 스타일 **시그널**을 감시할 수도 있다. 시그널 핸들러를 만들려면 다음을 사용한다.

### Interface

```c
#define evsignal_new(base, signum, cb, arg) \
    event_new(base, signum, EV_SIGNAL|EV_PERSIST, cb, arg)
```

인자들은 `event_new`와 동일하되, **파일 디스크립터 대신 시그널 번호**를 제공한다.

### Example

```c
struct event *hup_event;
struct event_base *base = event_base_new();

/* HUP 시그널이 오면 sighup_function을 호출한다 */
hup_event = evsignal_new(base, SIGHUP, sighup_function, NULL);
```

시그널 콜백은 **시그널 발생 직후에** 이벤트 루프 안에서 실행되므로, 일반적인 POSIX 시그널 핸들러에서 호출하면 안 되는 함수들을 **안전하게 호출**할 수 있다.

### Warning

시그널 이벤트에 **타임아웃을 설정하지 말라**. 지원되지 않을 수 있다. [FIXME: 실제 여부 확인 필요]

시그널 이벤트에 편의 매크로들도 제공된다.

### Interface

```c
#define evsignal_add(ev, tv) \
    event_add((ev),(tv))
#define evsignal_del(ev) \
    event_del(ev)
#define evsignal_pending(ev, what, tv_out) \
    event_pending((ev), (what), (tv_out))
```

`evsignal_*` 매크로들은 Libevent 2.0.1-alpha부터 있었으며, 이전 버전에서는 `signal_add()`, `signal_del()` 등의 이름을 사용했다.

### Caveats when working with signals

현 Libevent와 대부분의 백엔드에서는, **프로세스당 한 번에 하나의 event_base만** 시그널을 감시할 수 있다. 두 개의 event_base에 동시에 시그널 이벤트를 추가하면(시그널이 서로 달라도!), **오직 하나의 event_base만** 시그널을 받는다.

`kqueue` 백엔드는 **이 제한이 없다**.

------

## Setting up events without heap-allocation

성능 등의 이유로, 어떤 사람들은 이벤트를 **더 큰 구조체의 일부**로 할당하길 선호한다. 이렇게 하면 각 사용 시 다음을 절약할 수 있다:

- 작은 객체를 힙에 할당할 때 드는 **메모리 할당기 오버헤드**.
- `struct event` 포인터를 역참조할 때의 **시간 오버헤드**.
- 이벤트가 캐시에 없을 경우 발생할 수 있는 **추가 캐시 미스**.

이 방법은 **Libevent의 이진 호환성**을 깨뜨릴 위험이 있다(버전에 따라 `struct event` 크기가 다를 수 있음).

이 비용들은 **매우 작으며**, 대부분의 애플리케이션에서는 중요하지 않다. 특별한 경우가 아니라면 `event_new()`를 사용하는 것이 좋다. `event_assign()` 사용은, 훗날 Libevent가 더 큰 이벤트 구조체를 사용할 경우 **디버깅이 어려운 문제**를 야기할 수 있다.

### Interface

```c
int event_assign(struct event *event, struct event_base *base,
    evutil_socket_t fd, short what,
    void (*callback)(evutil_socket_t, short, void *), void *arg);
```

`event_assign()`의 인자들은 `event_new()`와 동일하되, 첫 번째 인자인 `event`는 **초기화되지 않은 이벤트 메모리**를 가리켜야 한다. 성공 시 0, 내부 오류/잘못된 인자 시 -1을 반환한다.

### Example

```c
#include <event2/event.h>
/* 주의! event_struct.h를 포함하면, 향후 Libevent 버전과의 이진 호환성이 깨진다. */
#include <event2/event_struct.h>
#include <stdlib.h>

struct event_pair {
         evutil_socket_t fd;
         struct event read_event;
         struct event write_event;
};
void readcb(evutil_socket_t, short, void *);
void writecb(evutil_socket_t, short, void *);
struct event_pair *event_pair_new(struct event_base *base, evutil_socket_t fd)
{
        struct event_pair *p = malloc(sizeof(struct event_pair));
        if (!p) return NULL;
        p->fd = fd;
        event_assign(&p->read_event, base, fd, EV_READ|EV_PERSIST, readcb, p);
        event_assign(&p->write_event, base, fd, EV_WRITE|EV_PERSIST, writecb, p);
        return p;
}
```

`event_assign()`는 **스택 할당** 또는 **정적 할당**된 이벤트를 초기화하는 데에도 사용할 수 있다.

### WARNING

이미 **대기 중인 이벤트**에 `event_assign()`를 호출하지 말라. **매우 디버그가 어려운 오류**로 이어질 수 있다. 이미 초기화되어 대기 중인 이벤트라면, `event_assign()`를 다시 호출하기 전에 **반드시 `event_del()`** 을 호출하라.

아래와 같은 편의 매크로로 **타임아웃 전용** 또는 **시그널 이벤트**에 대해 `event_assign()`를 쉽게 호출할 수 있다.

### Interface

```c
#define evtimer_assign(event, base, callback, arg) \
    event_assign(event, base, -1, 0, callback, arg)
#define evsignal_assign(event, base, signum, callback, arg) \
    event_assign(event, base, signum, EV_SIGNAL|EV_PERSIST, callback, arg)
```

`event_assign()`를 사용하면서도 **향후 이진 호환성**을 유지해야 한다면, 런타임에 `struct event`가 **얼마나 커야 하는지** Libevent에 물어볼 수 있다.

### Interface

```c
size_t event_get_struct_event_size(void);
```

이 함수는 `struct event`를 위해 확보해야 하는 **바이트 수**를 반환한다. 앞서 말했듯, 힙 할당이 정말로 큰 문제가 아니라면 이 함수 사용은 코드의 **가독성과 유지보수성**을 해친다.

미래에는 `event_get_struct_event_size()`가 `sizeof(struct event)`보다 **작은** 값을 반환할 수도 있다. 이 경우, `struct event`의 끝부분에 있는 여분의 바이트는 **미래 버전**에서 사용하기 위해 예약된 **패딩**일 뿐이다.

아래는 앞선 예제를 **런타임 크기 질의**를 통해 구현한 버전이다.

### Example

```c
#include <event2/event.h>
#include <stdlib.h>

/* event_pair를 메모리에 할당할 때, 실제로는 구조체 끝에 더 많은 공간을 할당한다.
 * 접근을 덜 오류-prone하게 만들기 위한 매크로들. */
struct event_pair {
         evutil_socket_t fd;
};

/* 매크로: 'p' 시작점에서 'offset' 바이트 뒤의 struct event를 가리킨다 */
#define EVENT_AT_OFFSET(p, offset) \
            ((struct event*) ( ((char*)(p)) + (offset) ))
/* 매크로: event_pair의 read event 포인터 */
#define READEV_PTR(pair) \
            EVENT_AT_OFFSET((pair), sizeof(struct event_pair))
/* 매크로: event_pair의 write event 포인터 */
#define WRITEEV_PTR(pair) \
            EVENT_AT_OFFSET((pair), \
                sizeof(struct event_pair)+event_get_struct_event_size())

/* 매크로: event_pair를 위해 실제로 할당해야 할 크기 */
#define EVENT_PAIR_SIZE() \
            (sizeof(struct event_pair)+2*event_get_struct_event_size())

void readcb(evutil_socket_t, short, void *);
void writecb(evutil_socket_t, short, void *);
struct event_pair *event_pair_new(struct event_base *base, evutil_socket_t fd)
{
        struct event_pair *p = malloc(EVENT_PAIR_SIZE());
        if (!p) return NULL;
        p->fd = fd;
        event_assign(READEV_PTR(p), base, fd, EV_READ|EV_PERSIST, readcb, p);
        event_assign(WRITEEV_PTR(p), base, fd, EV_WRITE|EV_PERSIST, writecb, p);
        return p;
}
```

`event_assign()`는 `<event2/event.h>`에 정의되어 있으며 2.0.1-alpha부터 존재한다. 2.0.3-alpha부터 **int 반환**으로 바뀌었으며, 그 전에는 `void`를 반환했다. `event_get_struct_event_size()`는 2.0.4-alpha에서 도입되었다. 이벤트 구조체는 `<event2/event_struct.h>`에 정의되어 있다.

------

## Making events pending and non-pending

이벤트를 구성한 뒤에는, **추가(add)** 하여 **대기(pending)** 상태로 만들기 전까지는 아무 일도 하지 않는다. 이를 위해 `event_add`를 사용한다.

### Interface

```c
int event_add(struct event *ev, const struct timeval *tv);
```

비대기 이벤트에 대해 `event_add`를 호출하면, 그 이벤트는 연결된 base에서 **대기 상태**가 된다. 성공 시 0, 실패 시 -1을 반환한다. `tv`가 **NULL**이면 **타임아웃 없이** 추가된다. 그렇지 않으면 `tv`는 **초 단위와 마이크로초 단위**의 타임아웃 크기를 나타낸다.

이미 대기 중인 이벤트에 대해 `event_add()`를 호출하면, 이벤트는 계속 대기 상태로 유지되며, **제공된 타임아웃으로 재스케줄**된다. 이미 대기 중인 이벤트를 **`tv == NULL`** 로 재추가하면, `event_add()`는 **아무 변화도 일으키지 않는다**.

### Note

`tv`를 “실행하고 싶은 절대 시각”으로 지정하면 안 된다. 예를 들어, 2010-01-01에 `tv->tv_sec = time(NULL) + 10;`이라 지정하면, 타임아웃은 **10초가 아니라 40년**을 기다리게 된다.

### Interface

```c
int event_del(struct event *ev);
```

**초기화된** 이벤트에 대해 `event_del`을 호출하면, 이벤트를 **비대기/비활성** 상태로 만든다. 이벤트가 대기/활성 상태가 아니면 **영향이 없다**. 성공 시 0, 실패 시 -1을 반환한다.

### Note

이벤트가 **활성**된 후 콜백이 실행되기 전에 이벤트를 **삭제**하면, 해당 콜백은 **실행되지 않는다**.

### Interface

```c
int event_remove_timer(struct event *ev);
```

마지막으로, **이벤트의 IO/시그널 구성은 유지**한 채 **대기 중 타임아웃만 제거**할 수 있다. 이벤트에 대기 중 타임아웃이 없으면 아무 효과가 없다. 이벤트가 **타임아웃만 있고 IO/시그널이 없다면**, `event_remove_timer()`는 `event_del()`과 **동일한 효과**가 있다. 성공 시 0, 실패 시 -1.

이 API들은 `<event2/event.h>`에 정의되어 있다. `event_add()`/`event_del()`은 Libevent 0.1부터 있었고, `event_remove_timer()`는 2.1.2-alpha에 추가되었다.

------

## Events with priorities

여러 이벤트가 동시에 트리거될 때, Libevent는 **콜백 실행 순서**를 정의하지 않는다. **우선순위**로 이벤트의 중요도를 지정할 수 있다.

앞서 설명했듯, 각 `event_base`는 하나 이상의 **우선순위 값**을 가진다. **이벤트를 추가하기 전에**, 하지만 **초기화 후에**, 이벤트의 우선순위를 설정할 수 있다.

### Interface

```c
int event_priority_set(struct event *event, int priority);
```

이벤트의 우선순위는 `0`부터, 해당 event_base의 **우선순위 개수 - 1** 사이의 값이다. 성공 시 0, 실패 시 -1을 반환한다.

여러 우선순위의 이벤트가 동시에 활성화되면, **낮은 우선순위 이벤트는 실행되지 않는다**. Libevent는 **높은 우선순위 이벤트**를 먼저 실행한 뒤 다시 이벤트를 확인한다. **활성화된 높은 우선순위 이벤트가 없을 때만** 낮은 우선순위 이벤트를 실행한다.

### Example

```c
#include <event2/event.h>

void read_cb(evutil_socket_t, short, void *);
void write_cb(evutil_socket_t, short, void *);

void main_loop(evutil_socket_t fd)
{
  struct event *important, *unimportant;
  struct event_base *base;

  base = event_base_new();
  event_base_priority_init(base, 2);
  /* 이제 base에는 우선순위 0과 1이 있다 */
  important = event_new(base, fd, EV_WRITE|EV_PERSIST, write_cb, NULL);
  unimportant = event_new(base, fd, EV_READ|EV_PERSIST, read_cb, NULL);
  event_priority_set(important, 0);
  event_priority_set(unimportant, 1);

  /* 이제 fd가 쓰기 가능이면 write 콜백이 read 콜백보다 먼저 실행된다.
     write 콜백이 활성인 동안에는 read 콜백은 전혀 실행되지 않는다. */
}
```

우선순위를 설정하지 않으면, 기본값은 해당 base의 **큐 개수 / 2** 이다.

이 함수는 `<event2/event.h>`에 선언되어 있으며 Libevent 1.0부터 존재한다.

------

## Inspecting event status

때때로 이벤트가 **추가되었는지** 확인하거나, **무엇을 참조**하는지 알고 싶을 수 있다.

### Interface

```c
int event_pending(const struct event *ev, short what, struct timeval *tv_out);

#define event_get_signal(ev) /* ... */
evutil_socket_t event_get_fd(const struct event *ev);
struct event_base *event_get_base(const struct event *ev);
short event_get_events(const struct event *ev);
event_callback_fn event_get_callback(const struct event *ev);
void *event_get_callback_arg(const struct event *ev);
int event_get_priority(const struct event *ev);

void event_get_assignment(const struct event *event,
        struct event_base **base_out,
        evutil_socket_t *fd_out,
        short *events_out,
        event_callback_fn *callback_out,
        void **arg_out);
```

`event_pending`은 주어진 이벤트가 **대기 또는 활성** 상태인지 판별한다. 그렇고, `what` 인자에 `EV_READ`, `EV_WRITE`, `EV_SIGNAL`, `EV_TIMEOUT` 중 하나라도 설정되어 있다면, 함수는 이벤트가 현재 **대기 또는 활성** 중인 **모든 플래그**를 반환한다. `tv_out`이 제공되고, `what`에 `EV_TIMEOUT`이 설정되어 있으며, 이벤트가 현재 **타임아웃**으로 대기/활성 상태라면, `tv_out`에는 그 타임아웃이 **만료될 시각**이 설정된다.

`event_get_fd()` / `event_get_signal()`은 이벤트에 설정된 **fd/시그널 번호**를 반환한다.
 `event_get_base()`는 연결된 **event_base**를 반환한다.
 `event_get_events()`는 이벤트의 플래그(`EV_READ`, `EV_WRITE` 등)를 반환한다.
 `event_get_callback()` / `event_get_callback_arg()`는 콜백 함수/인자 포인터를 반환한다.
 `event_get_priority()`는 이벤트의 **현재 우선순위**를 반환한다.

`event_get_assignment()`는 이벤트에 할당된 모든 필드를 전달된 포인터들로 **복사**한다. 포인터가 `NULL`이면 해당 항목은 **무시**된다.

### Example

```c
#include <event2/event.h>
#include <stdio.h>

/* 'ev'의 콜백과 콜백 인자를 변경한다. 단, 'ev'는 대기 상태가 아니어야 한다. */
int replace_callback(struct event *ev, event_callback_fn new_callback,
    void *new_callback_arg)
{
    struct event_base *base;
    evutil_socket_t fd;
    short events;

    int pending;

    pending = event_pending(ev, EV_READ|EV_WRITE|EV_SIGNAL|EV_TIMEOUT,
                            NULL);
    if (pending) {
        /* 대기 중인 이벤트를 재할당하는 일은 매우 위험하므로 여기서 걸러낸다. */
        fprintf(stderr,
                "Error! replace_callback called on a pending event!\n");
        return -1;
    }

    event_get_assignment(ev, &base, &fd, &events,
                         NULL /* 기존 콜백은 무시 */ ,
                         NULL /* 기존 콜백 인자도 무시 */);

    event_assign(ev, base, fd, events, new_callback, new_callback_arg);
    return 0;
}
```

이 함수들은 `<event2/event.h>`에 선언되어 있다.
 `event_pending()`은 Libevent 0.1부터 존재한다.
 2.0.1-alpha에서 `event_get_fd()`/`event_get_signal()`이 추가되었고,
 2.0.2-alpha에서 `event_get_base()`,
 2.1.2-alpha에서 `event_get_priority()`,
 나머지는 2.0.4-alpha에서 추가되었다.

------

## Finding the currently running event

디버깅 등 목적으로, **현재 실행 중인 이벤트**의 포인터를 가져올 수 있다.

### Interface

```c
struct event *event_base_get_running_event(struct event_base *base);
```

이 함수는 **해당 base의 루프 내부**에서 호출될 때만 동작이 정의된다. **다른 스레드**에서 호출하는 것은 지원되지 않으며, **정의되지 않은 동작**을 일으킬 수 있다.

이 함수는 `<event2/event.h>`에 선언되어 있으며 2.1.1-alpha에서 도입되었다.

------

## Configuring one-off events

이벤트를 **한 번만 추가**하면 되고, 추가한 후 **삭제할 필요가 없으며**, **지속적일 필요가 없다면**, `event_base_once()`를 사용할 수 있다.

### Interface

```c
int event_base_once(struct event_base *, evutil_socket_t, short,
  void (*)(evutil_socket_t, short, void *), void *, const struct timeval *);
```

이 함수의 인터페이스는 `event_new()`와 유사하지만, **EV_SIGNAL**과 **EV_PERSIST**는 지원하지 않는다. 예약된 이벤트는 **기본 우선순위**로 삽입/실행된다. 콜백이 끝나면 Libevent가 내부 이벤트 구조체를 **자동으로 해제**한다. 성공 시 0, 실패 시 -1.

`event_base_once`로 넣은 이벤트는 **삭제하거나 수동 활성화할 수 없다**. 취소 가능해야 한다면 `event_new()` 또는 `event_assign()`을 사용하라.

또한 Libevent 2.0까지는 이벤트가 **트리거되지 않으면 내부 메모리가 해제되지 않았다**. 2.1.2-alpha부터는, 활성화되지 않았더라도 **event_base가 해제될 때** 이 이벤트들도 해제된다. 다만 콜백 인자에 연동된 저장소가 있다면, 프로그램이 따로 **추적/해제**하지 않는 한 해제되지 않는다는 점을 유의하라.

------

## Manually activating an event

드물게, 이벤트 조건이 만족되지 않았더라도 **수동으로 활성화**하고 싶을 수 있다.

### Interface

```c
void event_active(struct event *ev, int what, short ncalls);
```

이 함수는 이벤트 `ev`를 `what` 플래그 조합(`EV_READ`, `EV_WRITE`, `EV_TIMEOUT`)으로 **활성화**한다. 이 이벤트는 이전에 **대기 상태일 필요가 없으며**, 활성화해도 **대기 상태로 바뀌지는 않는다**.

**경고**: 동일 이벤트에 대해 콜백 내부에서 **재귀적으로 event_active()** 를 호출하면 **리소스 고갈**을 일으킬 수 있다. 아래는 잘못된 사용 예시이다.

### Bad Example: event_active()로 무한 루프 만들기

```c
struct event *ev;

static void cb(int sock, short which, void *arg) {
        /* 실수: 콜백 안에서 무조건 같은 이벤트를 다시 활성화하면
           다른 이벤트가 전혀 실행되지 않을 수 있다! */

        event_active(ev, EV_WRITE, 0);
}

int main(int argc, char **argv) {
        struct event_base *base = event_base_new();

        ev = event_new(base, -1, EV_PERSIST | EV_READ, cb, NULL);

        event_add(ev, NULL);

        event_active(ev, EV_WRITE, 0);

        event_base_loop(base, 0);

        return 0;
}
```

이 코드는 이벤트 루프를 **한 번만** 실행시키고, 그 후 `cb`만 **영원히 반복 호출**되는 상황을 만든다.

### Example: 타이머로 위 문제를 해결하는 대안

```c
struct event *ev;
struct timeval tv;

static void cb(int sock, short which, void *arg) {
   if (!evtimer_pending(ev, NULL)) {
       event_del(ev);
       evtimer_add(ev, &tv);
   }
}

int main(int argc, char **argv) {
   struct event_base *base = event_base_new();

   tv.tv_sec = 0;
   tv.tv_usec = 0;

   ev = evtimer_new(base, cb, NULL);

   evtimer_add(ev, &tv);

   event_base_loop(base, 0);

   return 0;
}
```

### Example: event_config_set_max_dispatch_interval()로 해결하는 대안

```c
struct event *ev;

static void cb(int sock, short which, void *arg) {
        event_active(ev, EV_WRITE, 0);
}

int main(int argc, char **argv) {
        struct event_config *cfg = event_config_new();
        /* 다른 이벤트를 확인하기 전에 최대 16개의 콜백만 실행 */
        event_config_set_max_dispatch_interval(cfg, NULL, 16, 0);
        struct event_base *base = event_base_new_with_config(cfg);
        ev = event_new(base, -1, EV_PERSIST | EV_READ, cb, NULL);

        event_add(ev, NULL);

        event_active(ev, EV_WRITE, 0);

        event_base_loop(base, 0);

        return 0;
}
```

이 함수는 `<event2/event.h>`에 정의되어 있으며 Libevent 0.3부터 존재한다.

------

## Optimizing common timeouts

현재 Libevent는 **이진 힙**으로 대기 중 타임아웃을 관리한다. 이진 힙은 타임아웃 추가/삭제에 대해 **O(lg n)** 성능을 제공한다. 이는 타임아웃이 **무작위로 분포**해 있는 경우에는 최적이지만, **동일 타임아웃**을 가진 이벤트가 매우 많다면 최적이 아니다.

예를 들어, **1만 개**의 이벤트가 각각 추가된 시점으로부터 **정확히 5초** 후에 타임아웃되어야 한다고 하자. 이런 경우에는 **이중 연결 리스트 큐**를 사용하면 각 타임아웃에 대해 **O(1)** 성능을 얻을 수 있다.

물론 **모든** 타임아웃 값을 큐로 처리하고 싶지는 않을 것이다. 큐는 **상수 타임아웃 값**에만 빠르다. 타임아웃이 **랜덤하게 분포**한다면, 그런 타임아웃을 큐에 넣는 것은 **O(n)** 이 되어 이진 힙보다 훨씬 나빠진다.

Libevent는 이를 해결하기 위해, 일부 타임아웃은 **큐**, 나머지는 **이진 힙**에 두도록 한다. 이를 위해 Libevent에서 특별한 **“공통 타임아웃(common timeout)”** `timeval`을 받아, 해당 `timeval`로 이벤트를 추가하면 된다. **아주 많은 이벤트가 단일 공통 타임아웃**을 가진다면, 이 최적화를 사용해 타임아웃 성능이 향상될 것이다.

### Interface

```c
const struct timeval *event_base_init_common_timeout(
    struct event_base *base, const struct timeval *duration);
```

이 함수는 `event_base`와 공통 타임아웃 **지속시간**을 인자로 받는다. 반환값은 **특별한 struct timeval** 포인터로, 이를 사용해 이벤트를 **O(1) 큐**에 추가하도록 지시할 수 있다(기본 이진 힙 O(lg n) 대신). 이 특별한 `timeval`은 코드에서 자유롭게 복사/대입할 수 있다. 다만 **생성한 base에서만** 동작한다. 그 **실제 내용에 의존하지 말라**: Libevent는 내부적으로 어떤 큐를 사용할지 구분하기 위해 이 값을 사용한다.

### Example

```c
#include <event2/event.h>
#include <string.h>

/* 매우 많은 이벤트가 동일한 base에 등록되며, 그중 거의 대부분은 10초 타임아웃을 가진다.
 * initialize_timeout이 호출되면, 10초 타임아웃 이벤트들을 O(1) 큐로 보내도록 한다. */
struct timeval ten_seconds = { 10, 0 };

void initialize_timeout(struct event_base *base)
{
    struct timeval tv_in = { 10, 0 };
    const struct timeval *tv_out;
    tv_out = event_base_init_common_timeout(base, &tv_in);
    memcpy(&ten_seconds, tv_out, sizeof(struct timeval));
}

int my_event_add(struct event *ev, const struct timeval *tv)
{
    /* ev는 반드시 initialize_timeout에 넘긴 것과 같은 event_base를 가져야 한다 */
    if (tv && tv->tv_sec == 10 && tv->tv_usec == 0)
        return event_add(ev, &ten_seconds);
    else
        return event_add(ev, tv);
}
```

다른 모든 최적화 기능과 마찬가지로, **실제로 중요**하지 않다면 common_timeout을 **사용하지 않는 것**이 좋다.

이 기능은 Libevent 2.0.4-alpha에서 도입되었다.

------

## Telling a good event apart from cleared memory

Libevent는 **초기화된 이벤트**와, `calloc()`/`memset()`/`bzero()` 등으로 **0으로 지워진 메모리**를 구분하는 데 도움이 되는 함수를 제공한다.

### Interface

```c
int event_initialized(const struct event *ev);

#define evsignal_initialized(ev) event_initialized(ev)
#define evtimer_initialized(ev) event_initialized(ev)
```

### Warning

이 함수들은 **초기화된 이벤트**와 **초기화되지 않은 임의 메모리**를 **안정적으로** 구분하지 못한다. 해당 메모리가 **0으로 지워졌거나** **이벤트로 초기화**되었음이 확실한 경우에만 사용하라.

일반적으로, 특별한 목적이 없다면 이 함수들을 사용할 필요가 없다. `event_new()`가 반환한 이벤트는 **항상 초기화**되어 있다.

### Example

```c
#include <event2/event.h>
#include <stdlib.h>

struct reader {
    evutil_socket_t fd;
};

#define READER_ACTUAL_SIZE() \
    (sizeof(struct reader) + \
     event_get_struct_event_size())

#define READER_EVENT_PTR(r) \
    ((struct event *) (((char*)(r))+sizeof(struct reader)))

struct reader *allocate_reader(evutil_socket_t fd)
{
    struct reader *r = calloc(1, READER_ACTUAL_SIZE());
    if (r)
        r->fd = fd;
    return r;
}

void readcb(evutil_socket_t, short, void *);
int add_reader(struct reader *r, struct event_base *b)
{
    struct event *ev = READER_EVENT_PTR(r);
    if (!event_initialized(ev))
        event_assign(ev, b, r->fd, EV_READ, readcb, r);
    return event_add(ev, NULL);
}
```

`event_initialized()`는 Libevent 0.3부터 존재한다.

------

## Obsolete event manipulation functions

Libevent 2.0 이전에는 `event_assign()`/`event_new()`가 없었다. 대신 **현재(current) base**에 이벤트를 연결하는 `event_set()`을 사용했다. base가 둘 이상이면, 원하는 base에 연결되도록 `event_base_set()`을 **반드시** 호출해야 했다.

### Interface

```c
void event_set(struct event *event, evutil_socket_t fd, short what,
        void(*callback)(evutil_socket_t, short, void *), void *arg);
int event_base_set(struct event_base *base, struct event *event);
```

`event_set()`은 사실상 `event_assign()`과 비슷하지만, **현재 base**를 사용한다는 점이 다르다. `event_base_set()`은 이벤트가 연관된 base를 **변경**한다.

타이머/시그널을 편리하게 다루기 위한 `event_set()`의 변형도 있었다: `evtimer_set()`은 대략 `evtimer_assign()`에, `evsignal_set()`은 대략 `evsignal_assign()`에 대응한다.

2.0 이전 버전에서는 시그널 관련 변형의 접두사가 `evsignal_`이 아니라 `signal_`이었다. (즉, `signal_set()`, `signal_add()`, `signal_del()`, `signal_pending()`, `signal_initialized()`.) 아주 오래된 버전(0.6 이전)은 `evtimer_` 대신 `timeout_`을 사용했다. 그래서 코드 고고학을 하다 보면 `timeout_add()`, `timeout_del()`, `timeout_initialized()`, `timeout_set()`, `timeout_pending()` 등을 볼 수 있다.

`event_get_fd()`/`event_get_signal()` 대신, 2.0 이전 버전에서는 이벤트 구조체 내용을 직접 들여다보는 `EVENT_FD()`/`EVENT_SIGNAL()` 매크로를 썼다. 이 매크로들은 구조체 내부에 직접 접근했기 때문에 **이진 호환성**을 해쳤다. 2.0 이후에는 이 매크로들이 `event_get_fd()`/`event_get_signal()`의 **별칭**이다.

또한 2.0 이전에는 **락 지원이 없었기 때문에**, base의 이벤트 상태를 변경하는 함수들을 base를 실행 중인 스레드 **외부에서** 호출하면 **안전하지 않았다**. 여기에는 `event_add()`, `event_del()`, `event_active()`, `event_base_once()` 등이 포함된다.

`event_base_once()` 역할을 하는 `event_once()` 함수도 있었지만, **현재 base**를 사용했다.

`EV_PERSIST`는 2.0 이전에는 타임아웃과 **제대로 상호작용하지 않았다**. 이벤트가 활성화될 때 타임아웃을 **리셋하지 않았고**, 타임아웃과는 아무 상호작용도 하지 않았다.

2.0 이전 버전의 Libevent는 **같은 fd**에 대해 **같은 READ/WRITE** 조건을 가진 **여러 이벤트를 동시에 삽입**하는 것을 지원하지 않았다. 즉, 각 fd에 대해 동시에 기다릴 수 있는 **읽기 이벤트는 하나**, **쓰기 이벤트도 하나**뿐이었다.

