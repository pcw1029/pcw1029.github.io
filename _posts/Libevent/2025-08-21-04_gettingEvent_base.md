---
layout: single
title:  "Creating an event_base"
date: 2025-08-21
tags: libevent
categories: 
   - Libevent
---

### Creating an event_base

어떤 흥미로운 Libevent 함수를 사용하기 전에, 하나 이상의 `event_base` 구조체를 할당해야 한다.
 각 `event_base` 구조체는 이벤트들의 집합을 보관하며, 어떤 이벤트가 활성 상태인지 확인하기 위해 poll을 수행할 수 있다.

만약 `event_base`가 locking을 사용하도록 설정되었다면, 여러 스레드 사이에서 접근하는 것은 안전하다.
 그러나 루프 자체는 단일 스레드에서만 실행될 수 있다.
 IO를 여러 스레드에서 polling 하고 싶다면, 각 스레드마다 하나의 `event_base`가 필요하다.

> 팁: Libevent의 미래 버전은 여러 스레드에서 이벤트를 실행할 수 있는 `event_base`를 지원할 수도 있다.

각 `event_base`는 "method" 또는 백엔드를 사용하여 어떤 이벤트가 준비되었는지 확인한다.
 인식되는 메서드들은:

- select
- poll
- epoll
- kqueue
- devpoll
- evport
- win32

사용자는 환경 변수로 특정 백엔드를 비활성화할 수 있다.
 예: `kqueue` 백엔드를 끄고 싶다면 `EVENT_NOKQUEUE` 환경 변수를 설정한다.
 프로그램 내부에서 백엔드를 끄고 싶다면, 아래의 `event_config_avoid_method()` 참고.

------

### Setting up a default event_base

`event_base_new()` 함수는 기본 설정으로 새로운 event_base를 할당하고 반환한다.
 이 함수는 환경 변수를 검사하고 새로운 `event_base` 포인터를 반환한다.
 에러가 발생하면 `NULL`을 반환한다.

백엔드 메서드를 선택할 때는, 운영체제가 지원하는 가장 빠른 메서드를 선택한다.

#### Interface

```c
struct event_base *event_base_new(void);
```

대부분의 프로그램에서는 이것이면 충분하다.

이 함수는 `<event2/event.h>`에 선언되어 있으며, Libevent 1.4.3에서 처음 등장했다.

------

### Setting up a complicated event_base

event_base의 종류를 더 세밀하게 제어하고 싶다면 `event_config`를 사용해야 한다.
 `event_config`는 event_base에 대한 선호 정보를 담는 불투명 구조체다.
 event_base를 원할 때, event_config를 `event_base_new_with_config()`에 전달한다.

#### Interface

```c
struct event_config *event_config_new(void);
struct event_base *event_base_new_with_config(const struct event_config *cfg);
void event_config_free(struct event_config *cfg);
```

이 함수들로 `event_base`를 할당하는 절차는:

1. `event_config_new()` 호출 → 새로운 event_config 생성
2. event_config에 필요 조건을 지정
3. `event_base_new_with_config()` 호출 → 새로운 event_base 생성
4. 필요 없으면 `event_config_free()` 호출

------

#### Interface

```c
int event_config_avoid_method(struct event_config *cfg, const char *method);

enum event_method_feature {
    EV_FEATURE_ET = 0x01,
    EV_FEATURE_O1 = 0x02,
    EV_FEATURE_FDS = 0x04,
};
int event_config_require_features(struct event_config *cfg,
                                  enum event_method_feature feature);

enum event_base_config_flag {
    EVENT_BASE_FLAG_NOLOCK = 0x01,
    EVENT_BASE_FLAG_IGNORE_ENV = 0x02,
    EVENT_BASE_FLAG_STARTUP_IOCP = 0x04,
    EVENT_BASE_FLAG_NO_CACHE_TIME = 0x08,
    EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST = 0x10,
    EVENT_BASE_FLAG_PRECISE_TIMER = 0x20
};
int event_config_set_flag(struct event_config *cfg,
    enum event_base_config_flag flag);
```

* `event_config_avoid_method()` : 특정 백엔드를 이름으로 피하게 한다.

* `event_config_require_features()` : 지정한 기능을 제공하지 못하는 백엔드는 사용하지 않는다.

* `event_config_set_flag()` : event_base를 구성할 때 런타임 플래그를 설정한다.

- 

------

#### Recognized feature values (event_config_require_features)

- **EV_FEATURE_ET** : 엣지 트리거 IO를 지원해야 한다.
- **EV_FEATURE_O1** : 이벤트 추가/삭제/활성화가 O(1)인 백엔드여야 한다.
- **EV_FEATURE_FDS** : 소켓 외에 임의의 FD 타입도 지원해야 한다.

#### Recognized option values (event_config_set_flag)

- **EVENT_BASE_FLAG_NOLOCK** : event_base에 락을 만들지 않는다. (멀티스레드에서 안전하지 않음)
- **EVENT_BASE_FLAG_IGNORE_ENV** : EVENT_* 환경 변수를 무시하고 백엔드 선택
- **EVENT_BASE_FLAG_STARTUP_IOCP** : Windows에서 IOCP 로직을 시작 시 바로 초기화
- **EVENT_BASE_FLAG_NO_CACHE_TIME** : 타임아웃마다 시간을 확인하지 않고 콜백 이후에만 확인 (CPU 사용량 증가 가능)
- **EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST** : epoll changelist 최적화 사용 (dup()된 fd에서는 커널 버그 가능)
- **EVENT_BASE_FLAG_PRECISE_TIMER** : 더 느리지만 정밀한 타이머 사용

위 함수들은 성공 시 0, 실패 시 -1을 반환한다.

> 참고: OS에서 제공하지 않는 백엔드를 요구하는 event_config를 만들기는 쉽다.
>  예: Windows에는 O(1) 백엔드 없음 / Linux에는 EV_FEATURE_FDS + EV_FEATURE_O1 조합을 지원하는 백엔드 없음.
>  이런 경우 `event_base_new_with_config()`는 NULL 반환.

------

#### Interface

```c
int event_config_set_num_cpus_hint(struct event_config *cfg, int cpus)
```

- 현재 Windows IOCP에서만 의미가 있다.
- 멀티스레드에서 특정 CPU 개수를 활용하려는 힌트를 준다.
- 단지 힌트일 뿐, 실제 CPU 수는 다를 수 있다.

------

#### Interface

```c
int event_config_set_max_dispatch_interval(struct event_config *cfg,
    const struct timeval *max_interval, int max_callbacks,
    int min_priority);
```

- 낮은 우선순위 콜백들이 오래 실행되면서 높은 우선순위 이벤트가 지연되는 **우선순위 역전** 방지.
- `max_interval` → 일정 시간이 지나면 높은 우선순위 이벤트 확인
- `max_callbacks` → 일정 개수의 콜백 실행 후 높은 우선순위 이벤트 확인
- `min_priority` 이상인 이벤트에 적용

------

### Example: Preferring edge-triggered backends

```c
struct event_config *cfg;
struct event_base *base;
int i;

/* 프로그램은 가능한 경우 edge-triggered 이벤트를 사용하고 싶다. 
   그래서 두 번 시도한다: 한 번은 edge-triggered IO를 강제하고, 
   한 번은 그렇지 않게. */
for (i=0; i<2; ++i) {
    cfg = event_config_new();

    /* select는 싫다. */
    event_config_avoid_method(cfg, "select");

    if (i == 0)
        event_config_require_features(cfg, EV_FEATURE_ET);

    base = event_base_new_with_config(cfg);
    event_config_free(cfg);
    if (base)
        break;

    /* event_base_new_with_config()가 NULL을 반환하면,
       첫 번째 시도에서는 EV_FEATURE_ET 없이 다시 시도한다.
       두 번째 시도에서도 실패하면 포기한다. */
}
```

------

### Example: Avoiding priority-inversion

```c
struct event_config *cfg;
struct event_base *base;

cfg = event_config_new();
if (!cfg)
   /* 에러 처리 */;

/* 우선순위가 2개인 이벤트를 실행하려 한다. 
   우선순위 1 이벤트 중 일부는 콜백이 느릴 수 있으니,
   우선순위 0 이벤트를 확인하기 전에 100ms 이상 
   (혹은 콜백 5개 이상) 지연되지 않도록 하고 싶다. */
struct timeval msec_100 = { 0, 100*1000 };
event_config_set_max_dispatch_interval(cfg, &msec_100, 5, 1);

base = event_base_new_with_config(cfg);
if (!base)
   /* 에러 처리 */;

event_base_priority_init(base, 2);
```

이 함수들과 타입은 `<event2/event.h>`에 선언되어 있다.

- EVENT_BASE_FLAG_IGNORE_ENV → Libevent 2.0.2-alpha
- EVENT_BASE_FLAG_PRECISE_TIMER → Libevent 2.1.2-alpha
- event_config_set_num_cpus_hint() → Libevent 2.0.7-rc
- event_config_set_max_dispatch_interval() → Libevent 2.1.1-alpha
- 나머지는 Libevent 2.0.1-alpha



### event_base의 백엔드 메서드 확인하기

어떤 이벤트 베이스가 특정 메서드를 사용하여 이벤트를 multiplexing 하는지 확인하고 싶을 수도 있다.
 예를 들어, 프로그램이 성능 문제를 보일 때, 프로그램이 빠른 메서드를 쓰지 못하고 select 같은 느린 메서드를 쓰고 있는지 확인할 수 있다.

------

#### Interface

```c
const char *event_base_get_method(const struct event_base *base);
```

- 이 함수는 event_base가 어떤 메서드를 쓰고 있는지 나타내는 문자열을 반환한다.

------

### event_base가 가진 기능 확인하기

또한, event_base가 어떤 특성을 지원하는지도 확인할 수 있다.

------

#### Interface

```c
enum event_method_feature event_base_get_features(const struct event_base *base);
```

이 함수는 event_base가 지원하는 기능을 비트 마스크로 반환한다.

지원 기능에는 다음이 있다:

- **EV_FEATURE_ET** : 엣지 트리거 이벤트 지원
- **EV_FEATURE_O1** : 이벤트 추가/삭제/활성화가 O(1) 시간 복잡도로 가능
- **EV_FEATURE_FDS** : 소켓 외에도 임의의 fd 타입 지원

- - 

------

### 예제: 메서드와 기능 출력하기

```c
struct event_base *base;
const char **methods;
int i;

base = event_base_new();
if (!base)
    /* 에러 처리 */

printf("Libevent는 %s 메서드를 사용 중입니다.\n",
       event_base_get_method(base));

int features = event_base_get_features(base);
if ((features & EV_FEATURE_ET))
    printf("  지원 기능: Edge-triggered IO\n");
if ((features & EV_FEATURE_O1))
    printf("  지원 기능: O(1) add/delete\n");
if ((features & EV_FEATURE_FDS))
    printf("  지원 기능: 임의 fd 지원\n");
```

이 함수들은 `<event2/event.h>`에 선언되어 있으며, Libevent 2.0.1-alpha에 추가되었다.



------

### event_base의 백엔드 메서드 확인하기 (Examining an event_base’s backend method)

가끔은 event_base에서 실제로 어떤 기능이 사용 가능한지, 혹은 어떤 메서드를 쓰고 있는지 확인하고 싶을 수 있다.

------

#### Interface

```c
const char **event_get_supported_methods(void);
```

* `event_get_supported_methods()` 함수는 현재 Libevent 버전에서 지원하는 메서드들의 이름 배열에 대한 포인터를 반환한다.

* 배열의 마지막 요소는 `NULL`이다.

#### Example

```c
int i;
const char **methods = event_get_supported_methods();
printf("Starting Libevent %s.  Available methods are:\n",
    event_get_version());
for (i=0; methods[i] != NULL; ++i) {
    printf("    %s\n", methods[i]);
}
```

#### Note

- 이 함수는 Libevent가 **컴파일 시 지원하도록 빌드된 메서드 목록**을 반환한다.
- 하지만 실제 실행 시 운영체제가 모두 지원하지 않을 수도 있다.
- 예를 들어, OSX의 특정 버전에서는 kqueue가 버그 때문에 사용할 수 없을 수 있다.

------

#### Interface

```c
const char *event_base_get_method(const struct event_base *base);
enum event_method_feature event_base_get_features(const struct event_base *base);
```

- `event_base_get_method()` 호출은 event_base가 실제로 사용 중인 메서드의 이름을 반환한다.
- `event_base_get_features()` 호출은 지원되는 기능을 비트 마스크로 반환한다.

------

#### Example

```c
struct event_base *base;
enum event_method_feature f;

base = event_base_new();
if (!base) {
    puts("Couldn't get an event_base!");
} else {
    printf("Using Libevent with backend method %s.",
        event_base_get_method(base));
    f = event_base_get_features(base);
    if ((f & EV_FEATURE_ET))
        printf("  Edge-triggered events are supported.");
    if ((f & EV_FEATURE_O1))
        printf("  O(1) event notification is supported.");
    if ((f & EV_FEATURE_FDS))
        printf("  All FD types are supported.");
    puts("");
}
```

- 이 함수들은 `<event2/event.h>`에 정의되어 있다.
- `event_base_get_method()`는 Libevent 1.4.3부터 제공되었고, 나머지는 Libevent 2.0.1-alpha부터 제공되었다.

------

### event_base 해제하기 (Deallocating an event_base)

event_base 사용을 끝냈다면, `event_base_free()`로 해제할 수 있다.

------

#### Interface

```c
void event_base_free(struct event_base *base);
```

* 이 함수는 event_base를 해제하지만, 현재 event_base에 연결된 이벤트를 해제하지는 않는다.

* 소켓을 닫거나 이벤트 내부 포인터를 해제하지도 않는다.

* 이 함수는 `<event2/event.h>`에 정의되어 있으며, Libevent 1.2부터 구현되었다.

- 

------

### event_base에서 우선순위 설정하기 (Setting priorities on an event_base)

Libevent는 이벤트에 여러 우선순위를 부여할 수 있다.
 하지만 기본적으로 event_base는 단일 우선순위만 지원한다.
 `event_base_priority_init()`을 호출하여 우선순위 개수를 설정할 수 있다.

------

#### Interface

```c
int event_base_priority_init(struct event_base *base, int n_priorities);
```

- 성공 시 0, 실패 시 -1을 반환한다.
- `base`는 수정할 event_base이고, `n_priorities`는 지원할 우선순위 개수이다. 최소 1 이상이어야 한다.
- 새로운 이벤트의 우선순위는 0(가장 중요)부터 `n_priorities - 1`(가장 덜 중요)까지 번호가 매겨진다.
- `EVENT_MAX_PRIORITIES`라는 상수가 있으며, 이것이 `n_priorities` 값의 상한이다. 이 값을 초과하면 에러가 발생한다.

------

#### Note

- 반드시 이벤트가 활성화되기 전에 이 함수를 호출해야 한다.
- 보통 event_base를 생성한 직후 호출하는 것이 가장 좋다.

------

#### Interface

```c
int event_base_get_npriorities(struct event_base *base);
```

- 반환 값은 해당 base에 설정된 우선순위 개수이다.
- 예를 들어, 반환 값이 3이면 허용되는 우선순위 값은 0, 1, 2이다.

------

#### Example

- 구체적인 예제는 아래 **event_priority_set** 문서에서 제공된다.
- 기본적으로 새로운 이벤트들은 `(n_priorities / 2)`의 우선순위로 초기화된다.
- `event_base_priority_init()`은 `<event2/event.h>`에 정의되어 있으며 Libevent 1.0부터 사용 가능하다.
- `event_base_get_npriorities()`는 Libevent 2.1.1-alpha에서 새로 추가되었다.

------

### fork() 이후 event_base 재초기화하기 (Reinitializing an event_base after fork())

모든 이벤트 백엔드가 fork() 이후에도 정상적으로 유지되는 것은 아니다.
 따라서 프로그램이 fork() 또는 관련 시스템 호출로 새 프로세스를 시작하고,
 그 후에도 event_base를 계속 사용하려면 재초기화가 필요할 수 있다.

------

#### Interface

```c
int event_reinit(struct event_base *base);
```

- 성공 시 0, 실패 시 -1을 반환한다.

------

#### Example

```c
struct event_base *base = event_base_new();

/* ... 일부 이벤트를 event_base에 추가 ... */

if (fork()) {
    /* 부모 프로세스 */
    continue_running_parent(base); /*...*/
} else {
    /* 자식 프로세스 */
    event_reinit(base);
    continue_running_child(base); /*...*/
}
```

- `event_reinit()` 함수는 `<event2/event.h>`에 정의되어 있으며, Libevent 1.4.3-alpha에서 처음 제공되었다.

------

### 오래된 event_base 함수들 (Obsolete event_base functions)

이전 버전의 Libevent는 "현재(current)" event_base라는 개념에 크게 의존했다.
 "현재" event_base는 모든 스레드가 공유하는 전역 설정이었다.
 따라서 어떤 event_base를 사용할지 지정하지 않으면 "현재" event_base가 사용되었다.
 하지만 event_base는 스레드 안전하지 않았기 때문에 오류가 발생하기 쉬웠다.

------

#### Interface

```c
struct event_base *event_init(void);
```

- 이 함수는 `event_base_new()`처럼 동작했으며, 현재 base를 새로 할당한 base로 설정했다.
- 다른 방법으로 현재 base를 바꿀 수 있는 방법은 없었다.

------

- 이 섹션의 몇몇 event_base 함수들은 "현재 base"에서 동작하는 변형을 가지고 있었다.
- 이 함수들은 현재 함수들과 동일하게 동작하지만, base 인자를 받지 않았다.

| 현재 함수 (Current function) | 오래된 current-base 버전 (Obsolete version) |
| ---------------------------- | ------------------------------------------- |
| event_base_priority_init()   | event_priority_init()                       |
| event_base_get_method()      | event_get_method()                          |