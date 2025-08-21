---
layout: single
title:  "이벤트 루프 다루기"
date: 2025-08-21
tags: libevent
categories: 
   - Libevent
---

### 이벤트 루프 다루기

#### 루프 실행하기

일단 `event_base` 안에 몇몇 이벤트가 등록되었다면(이벤트를 생성하고 등록하는 방법은 다음 섹션 참고), 이제는 Libevent가 이벤트를 기다렸다가 그것을 알려주길 원하게 된다.

------

#### 인터페이스

```c
#define EVLOOP_ONCE             0x01
#define EVLOOP_NONBLOCK         0x02
#define EVLOOP_NO_EXIT_ON_EMPTY 0x04

int event_base_loop(struct event_base *base, int flags);
```

기본적으로 `event_base_loop()` 함수는 `event_base` 안에 더 이상 이벤트가 등록되어 있지 않을 때까지 실행된다. 루프를 실행하는 동안, 등록된 이벤트 중 어떤 것이 트리거되었는지를 반복적으로 확인한다.
 (예: 읽기 이벤트의 파일 디스크립터가 읽을 준비가 되었는지, 타임아웃 이벤트의 시간이 만료되었는지)
 트리거가 발생하면, 해당 이벤트를 “active(활성)” 상태로 표시하고 실행하기 시작한다.

`event_base_loop()`의 동작을 변경하려면 `flags` 인자에 플래그를 하나 이상 넣으면 된다.

- `EVLOOP_ONCE`가 설정되면, 루프는 이벤트가 활성화될 때까지 기다렸다가, 활성 이벤트들을 모두 실행한 후 반환한다.
- `EVLOOP_NONBLOCK`이 설정되면, 루프는 이벤트가 발생하기를 기다리지 않고, 당장 트리거될 준비가 된 이벤트가 있는지만 확인하고 있다면 즉시 실행한다.

일반적으로 루프는 대기 중이거나 활성화된 이벤트가 없으면 종료된다.
 이 동작을 무효화하려면 `EVLOOP_NO_EXIT_ON_EMPTY` 플래그를 설정하면 된다. (예: 다른 스레드에서 이벤트를 추가하려는 경우)
 이 플래그를 설정하면, 누군가 `event_base_loopbreak()` 또는 `event_base_loopexit()`을 호출하거나 에러가 발생할 때까지 루프는 계속 실행된다.

루프가 끝나면 `event_base_loop()`은 정상적으로 종료되었으면 0, 백엔드에서 처리되지 않은 에러 때문에 종료되었으면 -1, 더 이상 대기 중이거나 활성 이벤트가 없어서 종료되었으면 1을 반환한다.

------

#### 이해를 돕기 위한 `event_base_loop` 알고리즘 의사코드

```c
while (any events are registered with the loop,
        or EVLOOP_NO_EXIT_ON_EMPTY was set) {

    if (EVLOOP_NONBLOCK was set, or any events are already active)
        If any registered events have triggered, mark them active.
    else
        Wait until at least one event has triggered, and mark it active.

    for (p = 0; p < n_priorities; ++p) {
       if (any event with priority of p is active) {
          Run all active events with priority of p.
          break; /* 더 낮은 우선순위 이벤트는 실행하지 않는다 */
       }
    }

    if (EVLOOP_ONCE was set or EVLOOP_NONBLOCK was set)
       break;
}
```

---

#### 편의 기능

```c
int event_base_dispatch(struct event_base *base);
```

`event_base_dispatch()`는 `event_base_loop()`을 플래그 없이 호출한 것과 같다. 따라서 이벤트가 더 이상 등록되지 않았거나 `event_base_loopbreak()` 또는 `event_base_loopexit()`이 호출될 때까지 계속 실행된다.

이 함수들은 `<event2/event.h>`에 정의되어 있으며, Libevent 1.0부터 존재했다.

------

### 루프 정지하기

만약 모든 이벤트가 제거되기 전에 활성 이벤트 루프를 멈추고 싶다면, 약간 다른 두 가지 함수를 사용할 수 있다.

------

#### 인터페이스

```c
int event_base_loopexit(struct event_base *base,
                        const struct timeval *tv);
int event_base_loopbreak(struct event_base *base);
```

- `event_base_loopexit()`는 일정 시간이 지난 후 루프를 멈추도록 지시한다. `tv` 인자가 `NULL`이면 즉시 멈춘다. 현재 실행 중인 이벤트 콜백이 있다면 그것들은 끝까지 실행한 후 종료된다.
- `event_base_loopbreak()`는 루프를 즉시 멈추도록 지시한다. `event_base_loopexit(base, NULL)`와 달리, 현재 이벤트 콜백을 실행 중이라면 그것만 끝내고 바로 종료한다.

또한, 루프가 실행 중이지 않을 때 `event_base_loopexit(base, NULL)`와 `event_base_loopbreak(base)`는 다르게 동작한다:

- `loopexit`는 다음 루프 실행이 시작되면 첫 라운드 콜백 후 종료되도록 예약된다(`EVLOOP_ONCE`처럼).
- `loopbreak`는 현재 실행 중인 루프만 멈추며, 루프가 실행 중이지 않으면 아무 효과가 없다.

두 함수 모두 성공 시 0, 실패 시 -1을 반환한다.

------

#### 예제: 즉시 종료하기

```c
#include <event2/event.h>

/* 콜백에서 loopbreak 호출 */
void cb(int sock, short what, void *arg)
{
    struct event_base *base = arg;
    event_base_loopbreak(base);
}

void main_loop(struct event_base *base, evutil_socket_t watchdog_fd)
{
    struct event *watchdog_event;

    /* watchdog 소켓에서 읽을 바이트가 생기면 이벤트를 트리거한다.
       이때 cb()가 호출되고, 루프는 즉시 종료된다. 다른 이벤트들은 실행되지 않는다. */
    watchdog_event = event_new(base, watchdog_fd, EV_READ, cb, base);

    event_add(watchdog_event, NULL);

    event_base_dispatch(base);
}
```

------

#### 예제: 10초 동안만 루프 실행 후 종료

```c
#include <event2/event.h>

void run_base_with_ticks(struct event_base *base)
{
  struct timeval ten_sec;

  ten_sec.tv_sec = 10;
  ten_sec.tv_usec = 0;

  /* event_base를 10초 단위로 실행하고, 각 라운드가 끝날 때 "Tick"을 출력한다.
     (더 나은 방법은 아래 persistent timer 이벤트 참조) */
  while (1) {
     /* 10초 후 종료 예약 */
     event_base_loopexit(base, &ten_sec);

     event_base_dispatch(base);
     puts("Tick");
  }
}
```

가끔은 `event_base_dispatch()` 또는 `event_base_loop()` 호출이 **정상적으로** 종료된 것인지, 아니면 `event_base_loopexit()` 또는 `event_base_break()` 호출 때문에 종료된 것인지 구분하고 싶을 수 있다. 이를 위해 다음 함수를 사용해 loopexit 또는 break가 호출되었는지 확인할 수 있다.

------

#### 인터페이스

```c
int event_base_got_exit(struct event_base *base);
int event_base_got_break(struct event_base *base);
```

위 두 함수는 각각 루프가 `event_base_loopexit()` 또는 `event_base_break()`에 의해 멈췄다면 true를, 그렇지 않다면 false를 반환한다. 이 값들은 다음에 이벤트 루프를 시작할 때 **초기화**된다.

이 함수들은 `<event2/event.h>`에 선언되어 있다. `event_break_loopexit()`는 Libevent 1.0c에, `event_break_loopbreak()`는 Libevent 1.4.3에 처음 구현되었다.

------

## 이벤트 재확인 (Re-checking for events)

일반적으로 Libevent는 이벤트를 확인한 뒤, **가장 높은 우선순위**의 활성 이벤트들을 모두 실행하고, 다시 이벤트를 확인하는 과정을 반복한다. 하지만 어떤 때에는 **현재 콜백을 끝내자마자** Libevent가 즉시 스캔을 다시 하도록 지시하고 싶을 수 있다. `event_base_loopbreak()`에 비유하면, 이를 위해 `event_base_loopcontinue()`를 사용할 수 있다.

------

#### 인터페이스

```c
int event_base_loopcontinue(struct event_base *);
```

`event_base_loopcontinue()`는 **현재 이벤트 콜백을 실행 중이 아닐 때는** 아무 효과가 없다.

이 함수는 Libevent 2.1.2-alpha에 도입되었다.

------

## 내부 시간 캐시 확인 (Checking the internal time cache)

콜백 내부에서 대략적인 **현재 시각**을 얻고 싶은데, 직접 `gettimeofday()`를 호출하고 싶지 않을 때가 있다(운영체제에서 `gettimeofday()`가 시스템 콜로 구현되어 오버헤드가 있을 수 있기 때문).

콜백 내부에서는, 이번 콜백 라운드를 실행하기 **시작할 때** Libevent가 본 현재 시각을 요청할 수 있다.

------

#### 인터페이스

```c
int event_base_gettimeofday_cached(struct event_base *base, struct timeval *tv_out);
```

`event_base_gettimeofday_cached()`는 event_base가 **현재 콜백을 실행 중**이라면, 캐시된 시각을 `tv_out`에 설정한다. 그렇지 않다면 실제 현재 시각을 위해 `evutil_gettimeofday()`를 호출한다. 성공 시 0, 실패 시 음수를 반환한다.

주의: timeval은 Libevent가 콜백 실행을 **시작할 때** 캐시되므로 **약간 부정확**할 수 있다. 콜백 실행이 오래 걸리면 **상당히 부정확**해질 수 있다. 캐시를 즉시 갱신하려면 다음 함수를 호출할 수 있다.

------

#### 인터페이스

```c
int event_base_update_cache_time(struct event_base *base);
```

성공 시 0, 실패 시 -1을 반환하며, base가 이벤트 루프를 실행 중이 **아니라면** 아무 효과가 없다.

`event_base_gettimeofday_cached()`는 Libevent 2.0.4-alpha에서 새로 추가되었고, `event_base_update_cache_time()`은 2.1.1-alpha에서 추가되었다.

------

## event_base 상태 덤프 (Dumping the event_base status)

#### 인터페이스

```c
void event_base_dump_events(struct event_base *base, FILE *f);
```

프로그램(또는 Libevent 자체)을 디버깅할 때, event_base에 추가된 **모든 이벤트와 그 상태**의 전체 목록이 필요할 수 있다. `event_base_dump_events()`를 호출하면, 해당 목록을 지정한 stdio 파일로 출력한다.

이 목록은 사람이 읽기 좋게 만들어졌으며, **향후 버전에서 형식은 변경될 수 있다**.

이 함수는 Libevent 2.0.1-alpha에 도입되었다.

------

## event_base의 모든 이벤트에 대해 함수 실행 (Running a function over every event in an event_base)

#### 인터페이스

```c
typedef int (*event_base_foreach_event_cb)(const struct event_base *,
    const struct event *, void *);

int event_base_foreach_event(struct event_base *base,
                             event_base_foreach_event_cb fn,
                             void *arg);
```

`event_base_foreach_event()`를 사용하면 현재 `event_base`에 연관된 **모든 활성 또는 대기 이벤트**를 순회(iterate)할 수 있다. 제공한 콜백은 각 이벤트마다 정확히 한 번 호출되며, 호출 순서는 **정해져 있지 않다**. `event_base_foreach_event()`의 세 번째 인자는 콜백의 세 번째 인자로 그대로 전달된다.

콜백은 **계속 순회하려면 0**, 순회를 중단하려면 **0이 아닌 값**을 반환해야 한다. 콜백이 최종적으로 반환한 값이 `event_base_foreach_event()`의 반환값이 된다.

콜백 함수는 전달받은 이벤트를 **수정하면 안 되며**, 이벤트를 추가/제거하거나, event_base에 연관된 **어떤 이벤트도 변경하면 안 된다**. 그렇지 않으면 **정의되지 않은 동작**(크래시나 힙 손상까지 포함)이 발생할 수 있다.

`event_base_foreach_event()` 호출 동안에는 **event_base의 락이 잡힌 상태**로 유지된다. 따라서 다른 스레드에서 해당 event_base로 유용한 작업을 수행할 수 없으니, **콜백이 오래 걸리지 않도록** 주의해야 한다.

이 함수는 Libevent 2.1.2-alpha에 추가되었다.

---

## 오래된 이벤트 루프 함수들 (Obsolete event loop functions)

앞서 설명했듯, 예전 Libevent API는 전역적인 "현재(current) event_base" 개념을 가지고 있었다.

이 섹션의 이벤트 루프 함수들 중 일부는 **현재 base**에서 동작하는 변형이 있었으며, 현재 함수와 동일하게 동작하지만 **base 인자를 받지 않는다**.

| 현재 함수 (Current function) | 오래된 current-base 버전 (Obsolete current-base version) |
| ---------------------------- | -------------------------------------------------------- |
| event_base_dispatch()        | event_dispatch()                                         |
| event_base_loop()            | event_loop()                                             |
| event_base_loopexit()        | event_loopexit()                                         |
| event_base_loopbreak()       | event_loopbreak()                                        |