---
layout: single
title:  "Bufferevents: 고급 주제"
date: 2025-08-21
tags: libevent
categories: 
   - Libevent
---

# Bufferevents: 고급 주제

이 장은 일반적인 사용에는 필요하지 않은 Libevent의 **버퍼이벤트(bufferevent)** 고급 기능을 설명한다. 버퍼이벤트 사용법을 이제 막 배우는 중이라면, 이 장은 건너뛰고 다음의 **evbuffer** 장을 읽는 것이 좋다.

------

## 페어드 버퍼이벤트 (Paired bufferevents)

가끔 **스스로와 통신해야 하는 네트워킹 프로그램**이 있다. 예를 들어, 어떤 프로토콜 위로 사용자 연결을 터널링하도록 작성된 프로그램이 있는데, 때로는 자기 자신의 연결도 그 프로토콜 위로 터널링하고자 할 수 있다. 물론 **자기 리스닝 포트에 연결**을 열어 자기 자신을 사용하도록 할 수도 있지만, 그러면 **네트워크 스택을 통해 자기 자신과 대화**하느라 자원이 낭비된다.

대신, 실제 플랫폼 소켓을 전혀 쓰지 않으면서, **한쪽에서 쓴 모든 바이트가 반대쪽에서 수신**(그리고 그 반대도)되도록 **페어드 버퍼이벤트** 쌍을 만들 수 있다.

#### Interface

```c
int bufferevent_pair_new(struct event_base *base, int options,
    struct bufferevent *pair[2]);
```

`bufferevent_pair_new()`를 호출하면 `pair[0]`과 `pair[1]`에 서로 연결된 두 개의 버퍼이벤트가 설정된다. **BEV_OPT_CLOSE_ON_FREE**는 의미가 없고, **BEV_OPT_DEFER_CALLBACKS**는 **항상 켜져** 있다는 점을 제외하면 보통의 옵션이 모두 지원된다.

왜 **버퍼이벤트 페어**는 **지연 콜백**으로 동작해야 할까? 페어 중 한쪽의 연산이 그 버퍼이벤트를 변경하는 콜백을 유발하고, 그 결과 반대쪽 버퍼이벤트의 콜백이 호출되고, 이것이 여러 단계로 이어지는 경우가 흔하다. 콜백을 지연하지 않으면 이러한 호출 체인이 **스택 오버플로**를 자주 일으키고, **다른 연결을 굶기며**, **모든 콜백이 재진입 가능**해야만 했다.

페어드 버퍼이벤트는 **flush**를 지원한다. 모드를 **BEV_NORMAL** 또는 **BEV_FLUSH**로 설정하면, 보통이라면 워터마크 때문에 제한될 데이터를 무시하고 **관련 데이터를 한쪽에서 다른 쪽으로 강제로 전송**한다. 모드를 **BEV_FINISHED**로 설정하면 반대편 버퍼이벤트에 **EOF 이벤트**도 발생시킨다.

페어 중 **한 멤버를 free**하더라도 자동으로 다른 멤버가 free되거나 EOF 이벤트가 발생하지 않는다. 단지 **다른 멤버가 연결 해제(unlinked)**될 뿐이다. 연결이 해제된 이후에는 그 버퍼이벤트는 더 이상 데이터를 읽거나 쓸 수 없고, 아무 이벤트도 생성하지 않는다.

#### Interface

```c
struct bufferevent *bufferevent_pair_get_partner(struct bufferevent *bev)
```

가끔 한 멤버만 가지고 **페어의 반대편 멤버**가 필요할 수 있다. 이때 `bufferevent_pair_get_partner()`를 호출하면 된다. `bev`가 페어의 멤버이고 반대편이 아직 존재한다면, **반대 멤버를 반환**한다. 그렇지 않으면 **NULL**을 반환한다.

페어드 버퍼이벤트는 Libevent **2.0.1-alpha**에서 도입되었고, `bufferevent_pair_get_partner()`는 **2.0.6**에서 도입되었다.

------

## 필터링 버퍼이벤트 (Filtering bufferevents)

때때로 버퍼이벤트 객체를 통과하는 **모든 데이터에 변환을 적용**하고 싶을 때가 있다. 예를 들어 **압축 레이어**를 추가하거나, 어떤 프로토콜을 다른 프로토콜로 **래핑**하여 전송하고자 할 수 있다.

#### Interface

```c
enum bufferevent_filter_result {
        BEV_OK = 0,
        BEV_NEED_MORE = 1,
        BEV_ERROR = 2
};
typedef enum bufferevent_filter_result (*bufferevent_filter_cb)(
    struct evbuffer *source, struct evbuffer *destination, ev_ssize_t dst_limit,
    enum bufferevent_flush_mode mode, void *ctx);


struct bufferevent *bufferevent_filter_new(struct bufferevent *underlying,
        bufferevent_filter_cb input_filter,
        bufferevent_filter_cb output_filter,
        int options,
        void (*free_context)(void *),
        void *ctx);
```

`bufferevent_filter_new()`는 기존의 “**하부**(underlying) 버퍼이벤트”를 감싸는 **새로운 필터링 버퍼이벤트**를 생성한다. 하부 버퍼이벤트로 **수신된 모든 데이터**는 **입력 필터**를 거쳐 필터링 버퍼이벤트에 도착하고, 필터링 버퍼이벤트를 통해 **전송되는 모든 데이터**는 **출력 필터**를 거쳐 하부 버퍼이벤트로 보내진다.

하부 버퍼이벤트에 필터를 추가하면 **하부 버퍼이벤트의 콜백이 교체**된다. 여전히 하부 버퍼이벤트의 **evbuffer 콜백**은 추가할 수 있지만, **필터가 제대로 동작하길 원한다면 버퍼이벤트 자체의 콜백은 설정할 수 없다**.

`input_filter`와 `output_filter` 함수는 아래에 설명한다. `options`에는 통상적인 옵션이 모두 지원된다. **BEV_OPT_CLOSE_ON_FREE**가 설정되어 있으면, **필터링 버퍼이벤트를 free**할 때 **하부 버퍼이벤트도 함께 free**된다. `ctx`는 필터 함수로 전달되는 임의의 포인터이며, `free_context`가 제공되면 **필터링 버퍼이벤트가 닫히기 직전**에 `ctx`를 인자로 호출된다.

**입력 필터** 함수는 **하부 입력 버퍼에 새로운 읽기 가능 데이터가 생길 때마다** 호출된다. **출력 필터** 함수는 **필터의 출력 버퍼에 새로운 쓰기 가능 데이터가 생길 때마다** 호출된다. 각 필터는 한 쌍의 evbuffer를 받는다: **읽을 소스**와 **쓸 목적지**. `dst_limit`은 목적지에 **추가할 수 있는 바이트 상한**을 나타낸다. 필터는 이 값을 **무시할 수** 있지만, 그러면 **고워터마크나 속도 제한**을 위반할 수 있다. `dst_limit`이 **-1**이면 제한이 없다. `mode`는 필터가 **얼마나 공격적으로 쓸지**를 알려준다. **`BEV_NORMAL`**이면 **적절히 변환 가능한 만큼**만 쓰면 된다. **`BEV_FLUSH`**는 **가능한 한 많이** 쓰라는 뜻이고, **`BEV_FINISHED`**는 **스트림 종료 시 필요한 정리(cleanup)**도 수행하라는 뜻이다. 마지막 인자 `ctx`는 생성자에 전달했던 그 포인터다.

필터 함수는 목적지 버퍼에 데이터를 **성공적으로 썼다면 `BEV_OK`**, **더 입력이 필요하거나 다른 flush 모드가 필요해서 더는 쓸 수 없으면 `BEV_NEED_MORE`**, **복구 불가능한 에러면 `BEV_ERROR`**를 반환해야 한다.

필터를 생성하면 **하부 버퍼이벤트의 읽기와 쓰기가 모두 활성화**된다. 별도로 읽기/쓰기를 관리할 필요는 없다: 필터는 원치 않을 때 **하부 버퍼이벤트에서의 읽기를 자동으로 중단**한다. **2.0.8-rc** 이후에는 하부 버퍼이벤트의 읽기/쓰기를 **필터와 독립적으로** 활성/비활성화하는 것이 허용된다. 다만 그렇게 하면 필터가 **원하는 데이터를 제대로 받지 못할 수 있음**에 유의하라.

입력 필터와 출력 필터를 **둘 다 지정할 필요는 없다**. 생략한 필터는 **데이터를 변환 없이 통과**시키는 필터로 대체된다.

------

## 단일 읽기/쓰기의 최대 크기 제한

기본적으로, 버퍼이벤트는 이벤트 루프가 호출될 때마다 **가능한 최대 바이트 수**를 읽거나 쓰지 않는다. 그렇게 하면 **이상한 불공정성**이나 **리소스 굶주림**이 생길 수 있기 때문이다. 반대로, **기본값이 모든 상황에 합리적이지 않을 수도** 있다.

#### Interface

```c
int bufferevent_set_max_single_read(struct bufferevent *bev, size_t size);
int bufferevent_set_max_single_write(struct bufferevent *bev, size_t size);

ev_ssize_t bufferevent_get_max_single_read(struct bufferevent *bev);
ev_ssize_t bufferevent_get_max_single_write(struct bufferevent *bev);
```

두 “set” 함수는 각각 현재의 **읽기/쓰기 최대치**를 **교체**한다. `size`가 **0**이거나 **EV_SSIZE_MAX 이상**이면, 최대치를 **기본값으로 설정**한다. 성공 시 **0**, 실패 시 **-1**을 반환한다.

두 “get” 함수는 각각 현재의 **루프당 읽기/쓰기 최대치**를 반환한다.

이 함수들은 **2.1.1-alpha**에서 추가되었다.

------

## 버퍼이벤트와 **속도 제한 (Rate-limiting)**

일부 프로그램은 단일 버퍼이벤트나 **버퍼이벤트 그룹**의 **대역폭**을 제한하고자 한다. Libevent **2.0.4-alpha**와 **2.0.5-alpha**는 **개별 버퍼이벤트에 한도를 설정**하거나, 버퍼이벤트를 **속도 제한 그룹**에 할당하는 기본 기능을 추가했다.

### 속도 제한 모델

Libevent의 속도 제한은 **토큰 버킷 알고리즘**을 사용하여 **한 번에 읽거나 쓸 바이트 수**를 결정한다. 임의의 시점에서, 속도 제한 객체는 **“읽기 버킷”**과 **“쓰기 버킷”**을 갖고 있으며, 이 크기가 **즉시 읽거나 쓸 수 있는 바이트 수**를 결정한다. 각 버킷에는 **리필 속도**, **최대 버스트 크기**, **타이밍 단위(“tick”)**가 있다. 타이밍 단위가 경과할 때마다 버킷은 리필 속도에 **비례하여 채워지며**, **버스트 크기를 초과**하면 초과 바이트는 **버려진다**.

따라서, **리필 속도**는 객체가 **평균적으로** 전송/수신할 최대 속도를 결정하고, **버스트 크기**는 **단일 버스트에서 보낼/받을 수 있는 최대 바이트 수**를 결정한다. **타이밍 단위**는 트래픽의 **평탄함(smoothness)**을 결정한다.

### 버퍼이벤트에 속도 제한 설정

#### Interface

```c
#define EV_RATE_LIMIT_MAX EV_SSIZE_MAX
struct ev_token_bucket_cfg;
struct ev_token_bucket_cfg *ev_token_bucket_cfg_new(
        size_t read_rate, size_t read_burst,
        size_t write_rate, size_t write_burst,
        const struct timeval *tick_len);
void ev_token_bucket_cfg_free(struct ev_token_bucket_cfg *cfg);
int bufferevent_set_rate_limit(struct bufferevent *bev,
    struct ev_token_bucket_cfg *cfg);
```

`ev_token_bucket_cfg` 구조체는 **단일 버퍼이벤트** 또는 **버퍼이벤트 그룹**의 읽기/쓰기를 제한하는 **쌍의 토큰 버킷 설정**을 나타낸다. 생성하려면 `ev_token_bucket_cfg_new`를 호출하고, **최대 평균 읽기 속도**, **최대 읽기 버스트**, **최대 평균 쓰기 속도**, **최대 쓰기 버스트**, **틱 길이**를 제공한다. `tick_len`이 **NULL**이면 틱 길이는 **1초**가 기본값이다. 에러 시 **NULL**을 반환할 수 있다.

`read_rate`와 `write_rate`는 **초당 바이트**가 아니라 **틱당 바이트** 단위라는 점에 주의하라. 즉, 틱이 **0.1초**이고 `read_rate`가 **300**이면, 최대 평균 읽기 속도는 **초당 3000 바이트**다. **EV_RATE_LIMIT_MAX**를 넘는 속도/버스트 값은 지원되지 않는다.

버퍼이벤트의 전송 속도를 제한하려면, 해당 버퍼이벤트에 대해 `bufferevent_set_rate_limit()`을 호출하고 `ev_token_bucket_cfg`를 넘겨라. 성공 시 **0**, 실패 시 **-1**을 반환한다. **여러 버퍼이벤트에 동일한 `ev_token_bucket_cfg`를 공유**시킬 수 있다. **속도 제한을 제거**하려면 `cfg`에 **NULL**을 넘겨 `bufferevent_set_rate_limit()`을 호출하라.

`ev_token_bucket_cfg`를 free하려면 `ev_token_bucket_cfg_free()`를 호출한다. 단, **아직 그 설정을 사용 중인 버퍼이벤트가 있다면 안전하지 않다**는 점에 유의하라.

### 버퍼이벤트 **그룹**에 속도 제한 설정

버퍼이벤트들의 **총 대역폭 사용량**을 제한하려면, 버퍼이벤트를 **속도 제한 그룹**에 할당할 수 있다.

#### Interface

```c
struct bufferevent_rate_limit_group;

struct bufferevent_rate_limit_group *bufferevent_rate_limit_group_new(
        struct event_base *base,
        const struct ev_token_bucket_cfg *cfg);
int bufferevent_rate_limit_group_set_cfg(
        struct bufferevent_rate_limit_group *group,
        const struct ev_token_bucket_cfg *cfg);
void bufferevent_rate_limit_group_free(struct bufferevent_rate_limit_group *);
int bufferevent_add_to_rate_limit_group(struct bufferevent *bev,
    struct bufferevent_rate_limit_group *g);
int bufferevent_remove_from_rate_limit_group(struct bufferevent *bev);
```

속도 제한 그룹을 만들려면 `bufferevent_rate_limit_group_new()`에 `event_base`와 초기 `ev_token_bucket_cfg`를 넘긴다. 그룹에는 `bufferevent_add_to_rate_limit_group()`로 추가하고, `bufferevent_remove_from_rate_limit_group()`로 제거한다. 두 함수는 성공 시 **0**, 에러 시 **-1**을 반환한다.

하나의 버퍼이벤트는 **동시에 오직 한 개의 그룹**에만 속할 수 있다. 또한 버퍼이벤트는 **개별 속도 제한**(`bufferevent_set_rate_limit()`로 설정)과 **그룹 속도 제한**을 **둘 다** 가질 수 있다. **둘 다 설정되면 더 낮은 한도가 적용**된다.

기존 그룹의 속도 제한을 바꾸려면 `bufferevent_rate_limit_group_set_cfg()`를 호출한다. 성공 시 **0**, 실패 시 **-1**. `bufferevent_rate_limit_group_free()`는 **그룹을 해제**하고 **모든 멤버를 제거**한다.

**2.0 버전** 기준으로, Libevent의 그룹 속도 제한은 **총합적으로 공정**하려 노력하지만, **아주 작은 시간 규모**에서는 **불공정**할 수 있다. 스케줄링 공정성이 매우 중요하다면, 향후 버전을 위한 패치에 기여해주길 바란다.

### 현재 속도 제한 값 조회

가끔 코드에서 특정 버퍼이벤트나 그룹에 적용되는 **현재 속도 제한**을 조회하고 싶을 때가 있다. 이를 위한 함수들이 제공된다.

#### Interface

```c
ev_ssize_t bufferevent_get_read_limit(struct bufferevent *bev);
ev_ssize_t bufferevent_get_write_limit(struct bufferevent *bev);
ev_ssize_t bufferevent_rate_limit_group_get_read_limit(
        struct bufferevent_rate_limit_group *);
ev_ssize_t bufferevent_rate_limit_group_get_write_limit(
        struct bufferevent_rate_limit_group *);
```

위 함수들은 **버퍼이벤트** 또는 **그룹**의 **읽기/쓰기 토큰 버킷의 현재 크기(바이트)**를 반환한다. 버퍼이벤트가 **할당량을 초과하도록 강제**(예: flush)된 경우, 이 값들이 **음수**가 될 수 있음을 유의하라.

#### Interface

```c
ev_ssize_t bufferevent_get_max_to_read(struct bufferevent *bev);
ev_ssize_t bufferevent_get_max_to_write(struct bufferevent *bev);
```

이 함수들은 Libevent 전역의 **단일 호출당 최대 읽기/쓰기 제한**, 버퍼이벤트의 속도 제한, **그룹 속도 제한(있다면)**을 모두 고려할 때, **지금 당장** 버퍼이벤트가 **읽거나 쓸 의향이 있는** 바이트 수를 반환한다.

#### Interface

```c
void bufferevent_rate_limit_group_get_totals(
    struct bufferevent_rate_limit_group *grp,
    ev_uint64_t *total_read_out, ev_uint64_t *total_written_out);
void bufferevent_rate_limit_group_reset_totals(
    struct bufferevent_rate_limit_group *grp);
```

각 `bufferevent_rate_limit_group`는 **그룹 전체**에서 전송된 바이트 총량을 추적한다. 이를 통해 그룹 내 **여러 버퍼이벤트의 총 사용량**을 추적할 수 있다. `bufferevent_rate_limit_group_get_totals()`를 호출하면, `*total_read_out`과 `*total_written_out`에 각각 **총 읽기/쓰기 바이트 수**가 설정된다. 이 값들은 **그룹 생성 시 0**에서 시작하며, `bufferevent_rate_limit_group_reset_totals()`를 호출하면 **0으로 리셋**된다.

### 속도 제한 **수동 조정**

상당히 복잡한 요구사항을 가진 프로그램에서는, **토큰 버킷의 현재 값**을 **직접 조정**하고 싶을 수 있다. 예를 들어, 버퍼이벤트가 아닌 방법으로 트래픽을 생성하는 경우가 그러하다.

#### Interface

```c
int bufferevent_decrement_read_limit(struct bufferevent *bev, ev_ssize_t decr);
int bufferevent_decrement_write_limit(struct bufferevent *bev, ev_ssize_t decr);
int bufferevent_rate_limit_group_decrement_read(
        struct bufferevent_rate_limit_group *grp, ev_ssize_t decr);
int bufferevent_rate_limit_group_decrement_write(
        struct bufferevent_rate_limit_group *grp, ev_ssize_t decr);
```

이 함수들은 버퍼이벤트나 속도 제한 그룹의 **현재 읽기/쓰기 버킷**을 **감소**시킨다. 감소량은 **부호 있는 값**이므로, **증가**시키고 싶다면 **음수를 전달**하면 된다.

### 그룹에서 가능한 **최소 할당량(minimum share)** 설정

종종, 매 틱마다 그룹의 사용 가능 바이트를 **모든 버퍼이벤트에 균등 분배**하고 싶지 않을 때가 있다. 예를 들어, **10,000개의 활성 버퍼이벤트**가 있고 그룹에 매 틱 **10,000 바이트의 쓰기 가능량**이 있다면, 각 버퍼이벤트에 **매 틱 1바이트**만 쓰도록 허용하는 것은 **시스템 콜/ TCP 헤더 오버헤드** 때문에 비효율적이다.

이를 해결하기 위해, 각 속도 제한 그룹은 **“최소 할당량”**이라는 개념을 갖는다. 위 상황에서, 모든 버퍼이벤트가 매 틱 1바이트씩 쓰는 대신, 매 틱 **10,000/SHARE 개의 버퍼이벤트**가 **각각 SHARE 바이트**를 쓰도록 허용되고, 나머지는 **아무것도 쓰지 못한다**. 어떤 버퍼이벤트들이 먼저 쓰는지는 **매 틱 무작위**로 선택된다.

기본 최소 할당량은 성능을 고려해 선택되며, 현재(2.0.6-rc 기준) **64**로 설정되어 있다. 다음 함수로 이 값을 조정할 수 있다.

#### Interface

```c
int bufferevent_rate_limit_group_set_min_share(
        struct bufferevent_rate_limit_group *group, size_t min_share);
```

`min_share`를 **0**으로 설정하면 **최소 할당량 코드를 완전히 비활성화**한다.

Libevent의 속도 제한은 처음 도입될 때부터 최소 할당량을 갖고 있었고, 이를 변경하는 함수는 Libevent **2.0.6-rc**에서 처음 공개되었다.

### 속도 제한 구현의 제약

Libevent **2.0** 기준으로, 속도 제한 구현에는 다음과 같은 제약이 있다.

- 모든 버퍼이벤트 타입이 속도 제한을 **잘**(혹은 **전혀**) 지원하는 것은 아니다.
- 버퍼이벤트 **속도 제한 그룹은 중첩될 수 없고**, 버퍼이벤트는 **한 번에 하나의 그룹**에만 속할 수 있다.
- 속도 제한 구현은 **TCP 패킷에서 데이터로 전송된 바이트**만 계산하고, **TCP 헤더는 포함하지 않는다**.
- 읽기 제한 구현은 **애플리케이션이 일정 속도로만 데이터를 소비**한다는 사실을 TCP 스택이 인지하고, 버퍼가 가득 차면 **상대방에 back-pressure**를 가하는 것에 의존한다.
- 일부 버퍼이벤트 구현(특히 Windows IOCP 구현)은 **과할당(over-commit)**할 수 있다.
- 버킷은 시작 시 **한 틱 분량만큼 가득** 차 있다. 즉, 버퍼이벤트는 **즉시 읽기/쓰기**를 시작할 수 있고, **한 틱을 기다릴 필요가 없다**. 하지만, **N.1 틱** 동안 제한되었던 버퍼이벤트가 **N+1 틱 분량**의 트래픽을 전송할 가능성도 있다는 뜻이기도 하다.
- 틱은 **1밀리초보다 작을 수 없고**, **밀리초의 분수는 무시**된다.

```cpp
/// TODO: Write an example for rate-limiting
```

## 버퍼이벤트와 **SSL**

버퍼이벤트는 **OpenSSL** 라이브러리를 사용해 **SSL/TLS 보안 전송 계층**을 구현할 수 있다. 많은 애플리케이션이 OpenSSL을 링크하기를 원치 않거나 필요로 하지 않기 때문에, 이 기능은 **별도 라이브러리**인 `"libevent_openssl"`에 구현되어 있다. 향후 버전에서 **NSS**나 **GnuTLS** 같은 다른 SSL/TLS 라이브러리를 지원할 수 있지만, 현재는 **OpenSSL만** 제공된다.

OpenSSL 기능은 Libevent **2.0.3-alpha**에서 도입되었으나, **2.0.5-beta**나 **2.0.6-rc** 이전에는 그리 잘 동작하지 않았다.

이 절은 **OpenSSL/SSL/TLS/암호학 튜토리얼**이 아니다.

이들 함수는 모두 `"event2/bufferevent_ssl.h"` 헤더에 선언되어 있다.

### OpenSSL 기반 버퍼이벤트 설정 및 사용

#### Interface

```c
enum bufferevent_ssl_state {
        BUFFEREVENT_SSL_OPEN = 0,
        BUFFEREVENT_SSL_CONNECTING = 1,
        BUFFEREVENT_SSL_ACCEPTING = 2
};

struct bufferevent *
bufferevent_openssl_filter_new(struct event_base *base,
    struct bufferevent *underlying,
    SSL *ssl,
    enum bufferevent_ssl_state state,
    int options);

struct bufferevent *
bufferevent_openssl_socket_new(struct event_base *base,
    evutil_socket_t fd,
    SSL *ssl,
    enum bufferevent_ssl_state state,
    int options);
```

SSL 버퍼이벤트는 두 가지 방식으로 만들 수 있다.

- 다른 **하부 버퍼이벤트 위에서 통신**하는 **필터 기반** 버퍼이벤트,
- OpenSSL이 네트워크와 **직접 통신**하도록 하는 **소켓 기반** 버퍼이벤트.

어느 경우이든 **`SSL\*` 객체**와 그 객체의 **상태**를 제공해야 한다. 상태는 클라이언트로서 협상을 수행 중이면 **`BUFFEREVENT_SSL_CONNECTING`**, 서버로서 협상을 수행 중이면 **`BUFFEREVENT_SSL_ACCEPTING`**, 핸드셰이크가 완료되었다면 **`BUFFEREVENT_SSL_OPEN`**이어야 한다.

보통의 옵션들이 허용된다. **BEV_OPT_CLOSE_ON_FREE**를 사용하면, **openssl 버퍼이벤트가 닫힐 때** SSL 객체와 하부 fd나 버퍼이벤트도 함께 닫힌다.

핸드셰이크가 완료되면, **새 버퍼이벤트의 이벤트 콜백**이 **`BEV_EVENT_CONNECTED`** 플래그와 함께 호출된다.

소켓 기반 버퍼이벤트를 만들 때, **SSL 객체에 이미 소켓이 설정되어 있다면** 소켓을 직접 제공할 필요가 없다: **`-1`**을 넘기면 된다. 이후 **`bufferevent_setfd()`**로 fd를 설정할 수도 있다.

```less
/// TODO: Remove this once bufferevent_shutdown() API has been finished.
```

**중요**: SSL 버퍼이벤트에 **BEV_OPT_CLOSE_ON_FREE**가 설정된 경우, SSL 연결에서 **클린 셧다운이 수행되지 않는다**. 이는 두 가지 문제를 유발한다.
 첫째, 상대방 입장에서는 연결이 **“깨진(broken)”** 것으로 보이며, **의도적으로 닫혔는지** 혹은 **공격자나 제3자에 의해 손상되었는지**를 구분할 수 없다.
 둘째, OpenSSL은 세션을 **“나쁜(bad)”** 것으로 취급하여 세션 캐시에서 제거한다. 이는 **부하가 큰 SSL 애플리케이션**에서 **중대한 성능 저하**를 일으킬 수 있다.

현재 유일한 우회책은 **지연(lazy) SSL 셧다운을 수동으로 수행**하는 것이다. 이는 **TLS RFC를 위반**하지만, **세션이 닫힌 뒤에도 캐시에 남도록** 해준다. 다음 코드는 이 우회책을 구현한다.

#### Example

```c
SSL *ctx = bufferevent_openssl_get_ssl(bev);

/*
 * SSL_RECEIVED_SHUTDOWN tells SSL_shutdown to act as if we had already
 * received a close notify from the other end.  SSL_shutdown will then
 * send the final close notify in reply.  The other end will receive the
 * close notify and send theirs.  By this time, we will have already
 * closed the socket and the other end's real close notify will never be
 * received.  In effect, both sides will think that they have completed a
 * clean shutdown and keep their sessions valid.  This strategy will fail
 * if the socket is not ready for writing, in which case this hack will
 * lead to an unclean shutdown and lost session on the other end.
 */
SSL_set_shutdown(ctx, SSL_RECEIVED_SHUTDOWN);
SSL_shutdown(ctx);
bufferevent_free(bev);
```

#### Interface

```c
SSL *bufferevent_openssl_get_ssl(struct bufferevent *bev);
```

이 함수는 OpenSSL 버퍼이벤트에서 사용 중인 **SSL 객체를 반환**한다. `bev`가 OpenSSL 기반 버퍼이벤트가 아니라면 **NULL**을 반환한다.

#### Interface

```c
unsigned long bufferevent_get_openssl_error(struct bufferevent *bev);
```

이 함수는 주어진 버퍼이벤트 연산에 대해 **대기 중인 첫 번째 OpenSSL 에러**를 반환한다. 에러 형식은 OpenSSL 라이브러리의 `ERR_get_error()`가 반환하는 형식과 같다. 에러가 없으면 **0**을 반환한다.

#### Interface

```c
int bufferevent_ssl_renegotiate(struct bufferevent *bev);
```

이 함수를 호출하면 SSL에 **재협상(renegotiate)**을 지시하고, 버퍼이벤트에 **적절한 콜백을 호출**하게 한다. 이는 **고급 주제**이며, 특히 많은 SSL 버전에서 **재협상 관련 보안 이슈**가 알려져 있으므로, **정말로 무엇을 하는지 아는 경우**가 아니라면 **피하는 것이 좋다**.

#### Interface

```c
int bufferevent_openssl_get_allow_dirty_shutdown(struct bufferevent *bev);
void bufferevent_openssl_set_allow_dirty_shutdown(struct bufferevent *bev,
    int allow_dirty_shutdown);
```

모든 “좋은” SSL 프로토콜 버전(SSLv3 및 모든 TLS 버전)은 **인증된 셧다운**을 지원하여, 고의적인 종료와 **우발적/악의적 종료**를 구분할 수 있게 한다. 기본적으로 우리는 **정상 셧다운 외의 모든 종료를 에러**로 처리한다. 그러나 `allow_dirty_shutdown` 플래그를 **1**로 설정하면, 연결의 종료를 **`BEV_EVENT_EOF`**로 취급한다.

`allow_dirty_shutdown` 함수는 Libevent **2.1.1-alpha**에서 추가되었다.

#### Example: 단순 SSL 기반 에코 서버

```c
/* Simple echo server using OpenSSL bufferevents */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#include <openssl/ssl.h>
#include <openssl/err.h>
#include <openssl/rand.h>

#include <event.h>
#include <event2/listener.h>
#include <event2/bufferevent_ssl.h>

static void
ssl_readcb(struct bufferevent * bev, void * arg)
{
    struct evbuffer *in = bufferevent_get_input(bev);

    printf("Received %zu bytes\n", evbuffer_get_length(in));
    printf("----- data ----\n");
    printf("%.*s\n", (int)evbuffer_get_length(in), evbuffer_pullup(in, -1));

    bufferevent_write_buffer(bev, in);
}

static void
ssl_acceptcb(struct evconnlistener *serv, int sock, struct sockaddr *sa,
             int sa_len, void *arg)
{
    struct event_base *evbase;
    struct bufferevent *bev;
    SSL_CTX *server_ctx;
    SSL *client_ctx;

    server_ctx = (SSL_CTX *)arg;
    client_ctx = SSL_new(server_ctx);
    evbase = evconnlistener_get_base(serv);

    bev = bufferevent_openssl_socket_new(evbase, sock, client_ctx,
                                         BUFFEREVENT_SSL_ACCEPTING,
                                         BEV_OPT_CLOSE_ON_FREE);

    bufferevent_enable(bev, EV_READ);
    bufferevent_setcb(bev, ssl_readcb, NULL, NULL, NULL);
}

static SSL_CTX *
evssl_init(void)
{
    SSL_CTX  *server_ctx;

    /* Initialize the OpenSSL library */
    SSL_load_error_strings();
    SSL_library_init();
    /* We MUST have entropy, or else there's no point to crypto. */
    if (!RAND_poll())
        return NULL;

    server_ctx = SSL_CTX_new(SSLv23_server_method());

    if (! SSL_CTX_use_certificate_chain_file(server_ctx, "cert") ||
        ! SSL_CTX_use_PrivateKey_file(server_ctx, "pkey", SSL_FILETYPE_PEM)) {
        puts("Couldn't read 'pkey' or 'cert' file.  To generate a key\n"
           "and self-signed certificate, run:\n"
           "  openssl genrsa -out pkey 2048\n"
           "  openssl req -new -key pkey -out cert.req\n"
           "  openssl x509 -req -days 365 -in cert.req -signkey pkey -out cert");
        return NULL;
    }
    SSL_CTX_set_options(server_ctx, SSL_OP_NO_SSLv2);

    return server_ctx;
}

int
main(int argc, char **argv)
{
    SSL_CTX *ctx;
    struct evconnlistener *listener;
    struct event_base *evbase;
    struct sockaddr_in sin;

    memset(&sin, 0, sizeof(sin));
    sin.sin_family = AF_INET;
    sin.sin_port = htons(9999);
    sin.sin_addr.s_addr = htonl(0x7f000001); /* 127.0.0.1 */

    ctx = evssl_init();
    if (ctx == NULL)
        return 1;
    evbase = event_base_new();
    listener = evconnlistener_new_bind(
                         evbase, ssl_acceptcb, (void *)ctx,
                         LEV_OPT_CLOSE_ON_FREE | LEV_OPT_REUSEABLE, 1024,
                         (struct sockaddr *)&sin, sizeof(sin));

    event_base_loop(evbase, 0);

    evconnlistener_free(listener);
    SSL_CTX_free(ctx);

    return 0;
}

```

### 스레딩과 OpenSSL에 관한 몇 가지 노트

Libevent의 내장 스레딩 메커니즘은 **OpenSSL 락킹**을 커버하지 않는다. OpenSSL은 **수많은 전역 변수**를 사용하므로, **OpenSSL을 스레드 안전하게 구성**해야 한다. 이 과정은 Libevent의 범위를 벗어나지만, 자주 등장하는 주제이므로 간단히 다룬다.

#### Example: OpenSSL 스레드 안전을 켜는 **아주 단순한** 예시

> OpenSSL 문서를 참고하여 **올바르게 구성**했는지 확인하라.
>  Libevent는 이 코드가 **완전한 그림**을 보장하지 않으며, **오직 예시**로만 사용되어야 한다.

```c
/*
 * Please refer to OpenSSL documentation to verify you are doing this correctly,
 * Libevent does not guarantee this code is the complete picture, but to be used
 * only as an example.
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#include <pthread.h>
#include <openssl/ssl.h>
#include <openssl/crypto.h>

pthread_mutex_t * ssl_locks;
int ssl_num_locks;

/* Implements a thread-ID function as requied by openssl */
static unsigned long
get_thread_id_cb(void)
{
    return (unsigned long)pthread_self();
}

static void
thread_lock_cb(int mode, int which, const char * f, int l)
{
    if (which < ssl_num_locks) {
        if (mode & CRYPTO_LOCK) {
            pthread_mutex_lock(&(ssl_locks[which]));
        } else {
            pthread_mutex_unlock(&(ssl_locks[which]));
        }
    }
}

int
init_ssl_locking(void)
{
    int i;

    ssl_num_locks = CRYPTO_num_locks();
    ssl_locks = malloc(ssl_num_locks * sizeof(pthread_mutex_t));
    if (ssl_locks == NULL)
        return -1;

    for (i = 0; i < ssl_num_locks; i++) {
        pthread_mutex_init(&(ssl_locks[i]), NULL);
    }

    CRYPTO_set_id_callback(get_thread_id_cb);
    CRYPTO_set_locking_callback(thread_lock_cb);

    return 0;
}
```