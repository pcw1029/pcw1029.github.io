---
layout: single
title:  "버퍼이벤트: 개념과 기초"
date: 2025-08-21
tags: libevent
categories: 
   - Libevent
---

# 버퍼이벤트: 개념과 기초

대부분의 경우 애플리케이션은 **이벤트에 반응**하는 것만이 아니라 **데이터 버퍼링**도 어느 정도 필요로 한다. 예를 들어 쓰기를 하고 싶을 때의 전형적인 패턴은 다음과 같다:

- 연결에 쓸 데이터를 쓰기로 결정하고, 그 데이터를 버퍼에 넣는다.
- 연결이 **쓰기 가능**해질 때까지 기다린다.
- 가능한 만큼 데이터를 쓴다.
- 얼마나 썼는지 기억하고, 아직 더 쓸 데이터가 남아 있다면, 다시 연결이 쓰기 가능해질 때까지 기다린다.

이러한 **버퍼링된 IO 패턴**이 충분히 흔하기 때문에, Libevent는 이를 위한 일반 메커니즘을 제공한다. **“버퍼이벤트(bufferevent)”**는 (소켓 같은) **하부 전송(transport)**, **읽기 버퍼**, **쓰기 버퍼**로 구성된다. 기본 전송이 읽기/쓰기에 준비되었을 때 콜백을 주는 **일반 이벤트**와 달리, 버퍼이벤트는 **충분한 데이터를 읽거나 썼을 때** 사용자 콜백을 호출한다.

공통 인터페이스를 공유하는 버퍼이벤트 타입이 여럿 있다. 이 글을 쓰는 시점에 존재하는 타입들은 다음과 같다.

- **소켓 기반 버퍼이벤트**
   event_* 인터페이스를 백엔드로 사용하여, 하부 **스트림 소켓**에서 데이터를 송수신하는 버퍼이벤트.
- **비동기 IO 버퍼이벤트**
   Windows의 **IOCP** 인터페이스를 사용하여 하부 스트림 소켓에 데이터를 송수신하는 버퍼이벤트. (Windows 전용; 실험적)
- **필터링 버퍼이벤트**
   하부 버퍼이벤트에 넘기기 전에, 들어오고 나가는 데이터를 처리(예: 압축/변환)하는 버퍼이벤트.
- **페어드 버퍼이벤트**
   서로에게 데이터를 전송하는 **두 개**의 버퍼이벤트.

> **NOTE**
>  Libevent 2.0.2-alpha 기준으로, 여기의 버퍼이벤트 인터페이스는 **모든 타입에서 완전히 직교적이지 않다**. 즉, 아래에 설명된 모든 인터페이스가 **모든** 버퍼이벤트 타입에서 동작하는 것은 아니다. 향후 버전에서 수정될 예정이다.

> **NOTE ALSO**
>  현재 버퍼이벤트는 **TCP 같은 스트림 지향 프로토콜**에서만 동작한다. 향후에 **UDP 같은 데이터그램 지향 프로토콜** 지원이 추가될 수 있다.

이 절의 모든 함수와 타입은 `event2/bufferevent.h`에 선언되어 있다. **evbuffer** 관련 전용 함수는 `event2/buffer.h`에 선언되어 있으며, 이에 관한 내용은 다음 장에서 다룬다.

------

## 버퍼이벤트와 evbuffer

모든 버퍼이벤트에는 **입력 버퍼**와 **출력 버퍼**가 있다. 타입은 모두 `struct evbuffer`이다. 버퍼이벤트에 **쓸 데이터**가 있으면 **출력 버퍼**에 추가하고, 버퍼이벤트가 **읽을 데이터를 제공**하면 **입력 버퍼에서 소모(drain)**한다.

`evbuffer` 인터페이스는 많은 연산을 지원하며, 자세한 내용은 뒤에서 다룬다.

------

## 콜백과 워터마크

모든 버퍼이벤트에는 데이터 관련 콜백이 **두 개** 있다: **읽기 콜백**과 **쓰기 콜백**. 기본적으로 **읽기 콜백**은 하부 전송으로부터 **어떤 데이터든** 읽힐 때마다 호출되고, **쓰기 콜백**은 출력 버퍼의 데이터가 하부 전송으로 **충분히 비워졌을 때** 호출된다. 이 동작은 버퍼이벤트의 **읽기/쓰기 “워터마크”**를 조정하여 재정의할 수 있다.

버퍼이벤트에는 네 가지 워터마크가 있다:

- **읽기 저워터마크(low-water mark)**
   읽기가 발생하여 **입력 버퍼 길이**가 이 수준 이상이 되면, **읽기 콜백**을 호출한다. 기본값은 **0**이므로, **모든 읽기**가 읽기 콜백을 유발한다.
- **읽기 고워터마크(high-water mark)**
   입력 버퍼가 이 수준에 도달하면, 입력 버퍼에서 충분히 소모되어 다시 그 아래로 내려가기 전까지 **읽기를 중단**한다. 기본값은 **무제한**이므로, 입력 버퍼의 크기 때문에 읽기를 멈추지 않는다.
- **쓰기 저워터마크**
   쓰기가 발생하여 **출력 버퍼 길이**가 이 수준 이하가 되면, **쓰기 콜백**을 호출한다. 기본값은 **0**이므로, **출력 버퍼가 완전히 비워지지 않으면** 쓰기 콜백이 호출되지 않는다.
- **쓰기 고워터마크**
   버퍼이벤트가 **다른 버퍼이벤트의 하부 전송**으로 사용될 때 **특수한 의미**를 가질 수 있다. 필터링 버퍼이벤트에 대한 주석을 참조하라.

또한 버퍼이벤트에는 데이터 중심이 아닌 이벤트(연결 종료, 에러 등)를 애플리케이션에 알리기 위한 **“에러/이벤트 콜백”**이 있다. 다음 이벤트 플래그들이 정의된다:

- **BEV_EVENT_READING** — 읽기 연산 중 이벤트 발생(어떤 이벤트인지는 다른 플래그 참조)
- **BEV_EVENT_WRITING** — 쓰기 연산 중 이벤트 발생(어떤 이벤트인지는 다른 플래그 참조)
- **BEV_EVENT_ERROR** — 버퍼이벤트 연산 중 **에러** 발생. 자세한 에러는 `EVUTIL_SOCKET_ERROR()`를 호출.
- **BEV_EVENT_TIMEOUT** — 버퍼이벤트에서 **타임아웃** 발생.
- **BEV_EVENT_EOF** — 버퍼이벤트에서 **EOF** 수신.
- **BEV_EVENT_CONNECTED** — 버퍼이벤트에서 요청한 **연결 완료**.

(위 이벤트 이름들은 Libevent 2.0.2-alpha에서 새로 도입되었다.)

------

## 지연(deferred) 콜백

기본적으로, 버퍼이벤트(와 evbuffer) 콜백은 **조건이 발생하는 즉시** 실행된다. 하지만 의존성이 복잡하면 문제가 생길 수 있다. 예를 들어, **비어지면 A로 데이터를 옮기는 콜백**과, **가득 차면 A에서 데이터를 처리하는 콜백**이 있을 때, 이런 호출이 **동일 스택에서 연쇄적으로** 일어나면 **스택 오버플로** 위험이 있다.

이를 해결하기 위해, 버퍼이벤트(또는 evbuffer)에 콜백을 **지연(defer)**하도록 지시할 수 있다. 지연 콜백의 조건이 충족되면, 즉시 호출하지 않고 **event_loop()의 일부로 큐잉**하여, **일반 이벤트 콜백들 이후**에 호출된다.

(지연 콜백은 Libevent 2.0.1-alpha에서 도입되었다.)

------

## 버퍼이벤트 옵션 플래그

버퍼이벤트를 생성할 때 **플래그**를 하나 이상 사용해 동작을 바꿀 수 있다. 인식되는 플래그는 다음과 같다.

- **BEV_OPT_CLOSE_ON_FREE** — 버퍼이벤트를 해제할 때 **하부 전송을 닫는다**. (하부 소켓 닫기, 하부 버퍼이벤트 해제 등)
- **BEV_OPT_THREADSAFE** — 버퍼이벤트에 **자동으로 락을 할당**하여, **멀티스레드**에서 안전하게 사용 가능하게 한다.
- **BEV_OPT_DEFER_CALLBACKS** — 설정하면, 위에서 설명한 대로 버퍼이벤트가 **모든 콜백을 지연**한다.
- **BEV_OPT_UNLOCK_CALLBACKS** — 기본적으로 스레드 안전하도록 설정된 버퍼이벤트는 **사용자 콜백을 호출하는 동안** 버퍼이벤트의 **락을 유지**한다. 이 옵션을 설정하면 콜백 호출 시 **락을 해제**한다.

(Libevent 2.0.5-beta에서 `BEV_OPT_UNLOCK_CALLBACKS`가 도입되었다. 나머지 옵션은 2.0.1-alpha에서 새로 추가되었다.)

------

## 소켓 기반 버퍼이벤트 사용

가장 간단한 버퍼이벤트는 **소켓 기반 타입**이다. 소켓 기반 버퍼이벤트는 Libevent의 내부 **이벤트 메커니즘**을 사용해 하부 **네트워크 소켓**이 읽기/쓰기 가능해지는 것을 감지하고, `readv`, `writev`, `WSASend`, `WSARecv` 같은 네트워크 호출을 사용해 데이터를 송수신한다.

### 소켓 기반 버퍼이벤트 생성

`bufferevent_socket_new()`로 소켓 기반 버퍼이벤트를 만들 수 있다.

#### Interface

```c
struct bufferevent *bufferevent_socket_new(
    struct event_base *base,
    evutil_socket_t fd,
    enum bufferevent_options options);
```

- `base`는 `event_base`이고, `options`는 버퍼이벤트 옵션(BEV_OPT_CLOSE_ON_FREE 등)의 비트마스크이다.
- `fd`는 선택적 소켓 파일 디스크립터이다. 나중에 설정하려면 `-1`로 둘 수 있다.

> **Tip**
>  `bufferevent_socket_new`에 제공하는 소켓이 **논블로킹 모드**인지 확인하라. Libevent는 이를 위한 편의 함수 **`evutil_make_socket_nonblocking`**을 제공한다.

성공 시 버퍼이벤트를 반환하고, 실패 시 `NULL`을 반환한다.

`bufferevent_socket_new()`는 Libevent 2.0.1-alpha에서 도입되었다.

### 소켓 기반 버퍼이벤트에서 연결 시작

버퍼이벤트의 소켓이 아직 연결되지 않았다면, **새 연결을 시작**할 수 있다.

#### Interface

```c
int bufferevent_socket_connect(struct bufferevent *bev,
    struct sockaddr *address, int addrlen);
```

`address`, `addrlen`은 표준 `connect()` 호출과 동일하다. 버퍼이벤트에 아직 소켓이 설정되지 않았다면, 이 함수를 호출할 때 **새 스트림 소켓을 할당**하고 **논블로킹**으로 만든다.

이미 소켓이 있는 경우 이 함수를 호출하면, 해당 소켓이 **연결되지 않았음**을 Libevent에 알리고, **연결이 완료될 때까지** 해당 소켓에서 **읽기/쓰기**를 하지 않는다.

연결 완료 전이라도 **출력 버퍼에 데이터를 미리 추가**해도 된다.

성공적으로 연결 시도를 시작하면 **0**, 에러가 발생하면 **-1**을 반환한다.

#### Example

```c
#include <event2/event.h>
#include <event2/bufferevent.h>
#include <sys/socket.h>
#include <string.h>

void eventcb(struct bufferevent *bev, short events, void *ptr)
{
    if (events & BEV_EVENT_CONNECTED) {
         /* 127.0.0.1:8080에 연결됨. 보통 여기서 읽기/쓰기를 시작. */
    } else if (events & BEV_EVENT_ERROR) {
         /* 연결 중 에러 발생. */
    }
}

int main_loop(void)
{
    struct event_base *base;
    struct bufferevent *bev;
    struct sockaddr_in sin;

    base = event_base_new();

    memset(&sin, 0, sizeof(sin));
    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = htonl(0x7f000001); /* 127.0.0.1 */
    sin.sin_port = htons(8080); /* 포트 8080 */

    bev = bufferevent_socket_new(base, -1, BEV_OPT_CLOSE_ON_FREE);

    bufferevent_setcb(bev, NULL, NULL, eventcb, NULL);

    if (bufferevent_socket_connect(bev,
        (struct sockaddr *)&sin, sizeof(sin)) < 0) {
        /* 연결 시작 에러 */
        bufferevent_free(bev);
        return -1;
    }

    event_base_dispatch(base);
    return 0;
}
```

`bufferevent_base_connect()` 함수는 Libevent-2.0.2-alpha에서 도입되었다. 그 전에는 소켓에 직접 `connect()`를 호출해야 했고, 연결 완료 시 버퍼이벤트가 이를 **쓰기 가능**으로 보고했다.

`bufferevent_socket_connect()`로 연결을 시작한 경우에만 **`BEV_EVENT_CONNECTED`** 이벤트를 받는다. 직접 `connect()`를 호출하면, 연결은 **쓰기 이벤트**로 보고된다.

직접 `connect()`를 호출하되, 연결 성공 시 **`BEV_EVENT_CONNECTED`** 이벤트를 여전히 받고 싶다면, `connect()`가 **-1**을 반환하고 `errno`가 **EAGAIN** 또는 **EINPROGRESS**일 때 **`bufferevent_socket_connect(bev, NULL, 0)`**를 호출하라.

이 함수는 Libevent 2.0.2-alpha에서 도입되었다.

### 호스트명으로 연결 시작

DNS **호스트명 해석**과 **연결 시작**을 하나로 묶어 하고 싶을 때가 많다. 이를 위한 인터페이스가 있다.

#### Interface

```c
int bufferevent_socket_connect_hostname(struct bufferevent *bev,
    struct evdns_base *dns_base, int family, const char *hostname,
    int port);
int bufferevent_socket_get_dns_error(struct bufferevent *bev);
```

이 함수는 `hostname`을 `family` 타입 주소로 **DNS 해석**한다. (`family`는 `AF_INET`, `AF_INET6`, `AF_UNSPEC` 중 하나) 이름 해석에 실패하면 **에러 이벤트**로 이벤트 콜백을 호출한다. 성공하면 `bufferevent_connect`와 동일하게 **연결 시도**를 시작한다.

`dns_base`는 선택적이다. **NULL**이면, **이름 해석이 끝날 때까지 Libevent가 블록**한다 — 보통 원하지 않는 동작이다. 제공하면, Libevent가 이를 사용하여 **비동기적으로** 호스트명을 조회한다. DNS에 관한 자세한 내용은 R9장을 보라.

`bufferevent_socket_connect()`와 마찬가지로, 이 함수는 버퍼이벤트의 기존 소켓이 있더라도 **연결되지 않았음**을 알리고, **해석이 끝나고 연결에 성공할 때까지** 읽기/쓰기를 하지 않는다.

에러가 발생하면 그 에러는 DNS 호스트명 조회 에러일 수 있다. 가장 최근 에러는 `bufferevent_socket_get_dns_error()`로 확인할 수 있다. 반환값이 **0**이면 DNS 에러는 감지되지 않은 것이다.

#### Example: 사소한 HTTP v0 클라이언트

> 이 코드는 실제로 복사해 쓰지 말 것: **좋은 HTTP 클라이언트 구현 방법이 아니다**. 대신 **evhttp**를 참고하라.

```c
#include <event2/dns.h>
#include <event2/bufferevent.h>
#include <event2/buffer.h>
#include <event2/util.h>
#include <event2/event.h>

#include <stdio.h>

void readcb(struct bufferevent *bev, void *ptr)
{
    char buf[1024];
    int n;
    struct evbuffer *input = bufferevent_get_input(bev);
    while ((n = evbuffer_remove(input, buf, sizeof(buf))) > 0) {
        fwrite(buf, 1, n, stdout);
    }
}

void eventcb(struct bufferevent *bev, short events, void *ptr)
{
    if (events & BEV_EVENT_CONNECTED) {
         printf("Connect okay.\n");
    } else if (events & (BEV_EVENT_ERROR|BEV_EVENT_EOF)) {
         struct event_base *base = ptr;
         if (events & BEV_EVENT_ERROR) {
                 int err = bufferevent_socket_get_dns_error(bev);
                 if (err)
                         printf("DNS error: %s\n", evutil_gai_strerror(err));
         }
         printf("Closing\n");
         bufferevent_free(bev);
         event_base_loopexit(base, NULL);
    }
}

int main(int argc, char **argv)
{
    struct event_base *base;
    struct evdns_base *dns_base;
    struct bufferevent *bev;

    if (argc != 3) {
        printf("Trivial HTTP 0.x client\n"
               "Syntax: %s [hostname] [resource]\n"
               "Example: %s www.google.com /\n",argv[0],argv[0]);
        return 1;
    }

    base = event_base_new();
    dns_base = evdns_base_new(base, 1);

    bev = bufferevent_socket_new(base, -1, BEV_OPT_CLOSE_ON_FREE);
    bufferevent_setcb(bev, readcb, NULL, eventcb, base);
    bufferevent_enable(bev, EV_READ|EV_WRITE);
    evbuffer_add_printf(bufferevent_get_output(bev), "GET %s\r\n", argv[2]);
    bufferevent_socket_connect_hostname(
        bev, dns_base, AF_UNSPEC, argv[1], 80);
    event_base_dispatch(base);
    return 0;
}
```

`bufferevent_socket_connect_hostname()`는 Libevent 2.0.3-alpha에서, `bufferevent_socket_get_dns_error()`는 2.0.5-beta에서 새로 추가되었다.

------

## 일반 버퍼이벤트 연산

이 절의 함수들은 여러 버퍼이벤트 구현에서 공통으로 동작한다.

### 버퍼이벤트 해제

#### Interface

```c
void bufferevent_free(struct bufferevent *bev);
```

버퍼이벤트를 해제한다. 버퍼이벤트는 내부적으로 **참조 카운트**를 갖기 때문에, **지연 콜백이 보류 중**인 상태에서 해제하더라도, 콜백이 끝날 때까지는 실제로 삭제되지 않는다.

그럼에도 `bufferevent_free()`는 가능한 한 빨리 버퍼이벤트를 해제하려고 한다. **쓰기 대기 중인 데이터**가 있으면, 버퍼이벤트가 해제되기 전에 **flush되지 않을** 수 있다.

`BEV_OPT_CLOSE_ON_FREE` 플래그가 설정되어 있고, 이 버퍼이벤트가 **하부 전송(소켓/버퍼이벤트)**을 가지고 있다면, 버퍼이벤트 해제 시 **그 전송도 함께 닫힌다**.

이 함수는 Libevent 0.8부터 존재한다.

### 콜백, 워터마크, 활성화된 연산 조작

#### Interface

```c
typedef void (*bufferevent_data_cb)(struct bufferevent *bev, void *ctx);
typedef void (*bufferevent_event_cb)(struct bufferevent *bev,
    short events, void *ctx);

void bufferevent_setcb(struct bufferevent *bufev,
    bufferevent_data_cb readcb, bufferevent_data_cb writecb,
    bufferevent_event_cb eventcb, void *cbarg);

void bufferevent_getcb(struct bufferevent *bufev,
    bufferevent_data_cb *readcb_ptr,
    bufferevent_data_cb *writecb_ptr,
    bufferevent_event_cb *eventcb_ptr,
    void **cbarg_ptr);
```

`bufferevent_setcb()`는 버퍼이벤트의 콜백 중 **하나 이상**을 변경한다.

- `readcb`는 **충분한 데이터가 읽혔을 때**,
- `writecb`는 **충분한 데이터가 쓰였을 때**,
- `eventcb`는 **이벤트가 발생했을 때** 호출된다.

각 콜백의 첫 인자는 해당 이벤트가 발생한 **버퍼이벤트**이고, 마지막 인자는 사용자가 `cbarg`로 제공한 값이다(이를 통해 콜백에 데이터를 전달할 수 있다). `eventcb`의 `events` 인자는 **이벤트 플래그의 비트마스크**이다(위 “콜백과 워터마크” 참조).

콜백을 **비활성화**하려면 함수 자리로 **NULL**을 넘기면 된다. **모든 콜백**은 **하나의 `cbarg`** 값을 공유하므로, 이를 바꾸면 **모든 콜백**에 영향을 준다.

현재 설정된 콜백들은 `bufferevent_getcb()`로 조회할 수 있다. `*readcb_ptr`, `*writecb_ptr`, `*eventcb_ptr`, `*cbarg_ptr`에 각각 현재 값이 설정되며, **NULL** 포인터는 무시된다.

`bufferevent_setcb()`는 Libevent 1.4.4에서 도입. 타입 이름 `bufferevent_data_cb`, `bufferevent_event_cb`는 2.0.2-alpha에서 도입. `bufferevent_getcb()`는 2.1.1-alpha에서 추가.

#### Interface

```c
void bufferevent_enable(struct bufferevent *bufev, short events);
void bufferevent_disable(struct bufferevent *bufev, short events);

short bufferevent_get_enabled(struct bufferevent *bufev);
```

버퍼이벤트에서 `EV_READ`, `EV_WRITE`, 또는 `EV_READ|EV_WRITE`를 **활성화/비활성화**할 수 있다. 읽기/쓰기가 비활성화되어 있으면, 버퍼이벤트는 해당 방향으로 데이터를 읽거나 쓰려고 하지 않는다.

출력 버퍼가 비어 있을 때 **굳이 쓰기를 비활성화**할 필요는 없다: 버퍼이벤트는 자동으로 **쓰기를 멈췄다가**, 쓸 데이터가 생기면 **다시 시작**한다. 입력 버퍼가 **고워터마크**에 도달했을 때도 마찬가지다: 자동으로 **읽기를 멈추었다가**, 읽을 공간이 생기면 **다시 시작**한다.

기본적으로 **새로 생성된** 버퍼이벤트는 **쓰기 활성화**, **읽기 비활성화** 상태다.

현재 어떤 이벤트가 활성화되어 있는지는 `bufferevent_get_enabled()`로 확인할 수 있다.

이 함수들은 Libevent 0.8부터 존재하며, `bufferevent_get_enabled()`는 2.0.3-alpha에서 도입되었다.

#### Interface

```c
void bufferevent_setwatermark(struct bufferevent *bufev, short events,
    size_t lowmark, size_t highmark);
```

버퍼이벤트 하나의 **읽기 워터마크**, **쓰기 워터마크**, 또는 **둘 다**를 조정한다. (`events`에 `EV_READ`가 설정되어 있으면 **읽기 워터마크**, `EV_WRITE`가 설정되어 있으면 **쓰기 워터마크**를 조정한다.)

**고워터마크가 0**이면 “**무제한**”과 같다.

이 함수는 Libevent 1.4.4에서 처음 공개되었다.

#### Example

```c
#include <event2/event.h>
#include <event2/bufferevent.h>
#include <event2/buffer.h>
#include <event2/util.h>

#include <stdlib.h>
#include <errno.h>
#include <string.h>

struct info {
    const char *name;
    size_t total_drained;
};

void read_callback(struct bufferevent *bev, void *ctx)
{
    struct info *inf = ctx;
    struct evbuffer *input = bufferevent_get_input(bev);
    size_t len = evbuffer_get_length(input);
    if (len) {
        inf->total_drained += len;
        evbuffer_drain(input, len);
        printf("Drained %lu bytes from %s\n",
             (unsigned long) len, inf->name);
    }
}

void event_callback(struct bufferevent *bev, short events, void *ctx)
{
    struct info *inf = ctx;
    struct evbuffer *input = bufferevent_get_input(bev);
    int finished = 0;

    if (events & BEV_EVENT_EOF) {
        size_t len = evbuffer_get_length(input);
        printf("Got a close from %s.  We drained %lu bytes from it, "
            "and have %lu left.\n", inf->name,
            (unsigned long)inf->total_drained, (unsigned long)len);
        finished = 1;
    }
    if (events & BEV_EVENT_ERROR) {
        printf("Got an error from %s: %s\n",
            inf->name, evutil_socket_error_to_string(EVUTIL_SOCKET_ERROR()));
        finished = 1;
    }
    if (finished) {
        free(ctx);
        bufferevent_free(bev);
    }
}

struct bufferevent *setup_bufferevent(void)
{
    struct bufferevent *b1 = NULL;
    struct info *info1;

    info1 = malloc(sizeof(struct info));
    info1->name = "buffer 1";
    info1->total_drained = 0;

    /* ... 여기서 버퍼이벤트를 설정하고 연결을 보장해야 함 ... */

    /* 입력 버퍼에 데이터가 적어도 128바이트 있을 때만 읽기 콜백을 호출. */
    bufferevent_setwatermark(b1, EV_READ, 128, 0);

    bufferevent_setcb(b1, read_callback, NULL, event_callback, info1);

    bufferevent_enable(b1, EV_READ); /* 읽기 시작 */
    return b1;
}
```

## 버퍼이벤트의 데이터 조작

네트워크에서 데이터를 읽고 쓰는 것만으로는 소용이 없다. 버퍼이벤트는 **쓰기용 데이터 제공**과 **읽기용 데이터 가져오기**를 위한 메서드를 제공한다.

#### Interface

```c
struct evbuffer *bufferevent_get_input(struct bufferevent *bufev);
struct evbuffer *bufferevent_get_output(struct bufferevent *bufev);
```

두 함수는 각각 **입력 버퍼**와 **출력 버퍼**를 반환한다. `evbuffer`에서 수행할 수 있는 모든 연산은 다음 장을 보라.

애플리케이션은 **입력 버퍼에서는 제거만(추가 불가)**, **출력 버퍼에서는 추가만(제거 불가)** 할 수 있다.

쓰기 쪽이 **데이터 부족** 때문에 멈추었거나(읽기 쪽이 **데이터 과다** 때문에 멈추었거나) 했다면, **출력 버퍼에 데이터를 추가**(또는 **입력 버퍼에서 데이터를 제거**)하면 자동으로 다시 시작된다.

이 함수들은 Libevent 2.0.1-alpha에서 도입되었다.

#### Interface

```c
int bufferevent_write(struct bufferevent *bufev,
    const void *data, size_t size);
int bufferevent_write_buffer(struct bufferevent *bufev,
    struct evbuffer *buf);
```

버퍼이벤트의 **출력 버퍼**에 데이터를 추가한다.

- `bufferevent_write()`는 `data`에서 **size 바이트**를 출력 버퍼 **끝에 추가**한다.
- `bufferevent_write_buffer()`는 `buf`의 **모든 내용**을 제거하여 출력 버퍼 **끝에 붙인다**.

둘 다 성공 시 **0**, 에러 시 **-1**을 반환한다.

이 함수들은 Libevent 0.8부터 존재한다.

#### Interface

```c
size_t bufferevent_read(struct bufferevent *bufev, void *data, size_t size);
int bufferevent_read_buffer(struct bufferevent *bufev,
    struct evbuffer *buf);
```

버퍼이벤트의 **입력 버퍼**에서 데이터를 제거한다.

- `bufferevent_read()`는 최대 **size 바이트**를 입력 버퍼에서 제거하여 `data`에 저장하고, **실제 제거한 바이트 수**를 반환한다.
- `bufferevent_read_buffer()`는 입력 버퍼의 **전체 내용을 비우고** 이를 `buf`에 넣는다. 성공 시 **0**, 실패 시 **-1**.

`bufferevent_read()`에서 `data`가 가리키는 메모리는 **size 바이트를 담을 충분한 공간**이 있어야 한다.

`bufferevent_read()`는 Libevent 0.8부터, `bufferevent_read_buffer()`는 2.0.1-alpha에서 도입되었다.

#### Example

```c
#include <event2/bufferevent.h>
#include <event2/buffer.h>

#include <ctype.h>

void
read_callback_uppercase(struct bufferevent *bev, void *ctx)
{
        /* 이 콜백은 입력 버퍼에서 128바이트씩 데이터를 꺼내
           대문자로 바꾸고 다시 보내준다.

           (주의! 실무에서는 현재 로케일이 원하는 로케일임을 확신하지 않는 한
           네트워크 프로토콜을 toupper로 구현하면 안 된다.)
         */

        char tmp[128];
        size_t n;
        int i;
        while (1) {
                n = bufferevent_read(bev, tmp, sizeof(tmp));
                if (n <= 0)
                        break; /* 더 이상 데이터 없음. */
                for (i=0; i<n; ++i)
                        tmp[i] = toupper(tmp[i]);
                bufferevent_write(bev, tmp, n);
        }
}

struct proxy_info {
        struct bufferevent *other_bev;
};
void
read_callback_proxy(struct bufferevent *bev, void *ctx)
{
        /* 단순 프록시를 구현한다면 이런 함수를 쓸 수 있다:
           한 연결(bev)에서 받은 데이터를 다른 연결로
           가능한 한 적게 복사하여 쓴다. */
        struct proxy_info *inf = ctx;

        bufferevent_read_buffer(bev,
            bufferevent_get_output(inf->other_bev));
}

struct count {
        unsigned long last_fib[2];
};

void
write_callback_fibonacci(struct bufferevent *bev, void *ctx)
{
        /* 이 콜백은 bev의 출력 버퍼에 일부 피보나치 수열을 추가한다.
           1KB를 채우면 멈추고, 이 데이터가 비워지면 다시 더 추가한다. */
        struct count *c = ctx;

        struct evbuffer *tmp = evbuffer_new();
        while (evbuffer_get_length(tmp) < 1024) {
                 unsigned long next = c->last_fib[0] + c->last_fib[1];
                 c->last_fib[0] = c->last_fib[1];
                 c->last_fib[1] = next;

                 evbuffer_add_printf(tmp, "%lu", next);
        }

        /* 이제 tmp의 전체 내용을 bev에 추가한다. */
        bufferevent_write_buffer(bev, tmp);

        /* tmp는 더 이상 필요 없다. */
        evbuffer_free(tmp);
}
```

## 읽기/쓰기 타임아웃

다른 이벤트와 마찬가지로, 버퍼이벤트에서 **일정 시간 동안** 읽기/쓰기가 **성공하지 않으면** 타임아웃이 발생하도록 설정할 수 있다.

#### Interface

```c
void bufferevent_set_timeouts(struct bufferevent *bufev,
    const struct timeval *timeout_read, const struct timeval *timeout_write);
```

타임아웃을 **NULL**로 설정하면 **제거**되어야 한다. 그러나 Libevent 2.1.2-alpha 이전에는 **모든 이벤트 타입에서** 이렇게 동작하지 않았다. (구버전의 해결책으로, **며칠짜리 긴 간격**으로 설정하거나, 원하지 않을 때 **`eventcb`에서 `BEV_TIMEOUT`**을 무시하도록 할 수 있다.)

- **읽기 타임아웃**은 버퍼이벤트가 **읽기를 시도**하는 동안 최소 `timeout_read`초를 기다리면 발생한다.
- **쓰기 타임아웃**은 버퍼이벤트가 **쓰기를 시도**하는 동안 최소 `timeout_write`초를 기다리면 발생한다.

타임아웃은 버퍼이벤트가 **읽거나 쓰고자 할 때만** 센다. 즉, 읽기가 비활성화되어 있거나, 입력 버퍼가 **고워터마크**에 차 있는 경우에는 **읽기 타임아웃이 활성화되지 않는다**. 쓰기도 마찬가지로, 쓰기가 비활성화되어 있거나 쓸 데이터가 없으면 **쓰기 타임아웃이 활성화되지 않는다**.

읽기/쓰기 타임아웃이 발생하면, 해당 방향의 **연산이 비활성화**되고, 이벤트 콜백이 **`BEV_EVENT_TIMEOUT|BEV_EVENT_READING`** 또는 **`BEV_EVENT_TIMEOUT|BEV_EVENT_WRITING`**으로 호출된다.

이 함수는 Libevent 2.0.1-alpha부터 존재한다. 2.0.4-alpha까지는 버퍼이벤트 타입에 따라 **일관되게 동작하지 않았다**.

------

## 버퍼이벤트에서 flush 시작

#### Interface

```c
int bufferevent_flush(struct bufferevent *bufev,
    short iotype, enum bufferevent_flush_mode state);
```

버퍼이벤트에서 **flush**를 수행하면, 보통은 쓰여지지 않도록 막고 있는 제약을 무시하고, 가능한 한 많은 바이트를 **하부 전송으로 쓰거나** **하부 전송에서 읽도록** 지시한다. 구체 동작은 **버퍼이벤트 타입**에 따라 다르다.

- `iotype`는 `EV_READ`, `EV_WRITE`, `EV_READ|EV_WRITE` 중 하나로, **읽기/쓰기/둘 다**에 대해 처리할지 지정한다.
- `state`는 `BEV_NORMAL`, `BEV_FLUSH`, `BEV_FINISHED` 중 하나다. `BEV_FINISHED`는 **더 이상 데이터를 보내지 않음을 상대에게 알려야 함**을 의미한다. `BEV_NORMAL`과 `BEV_FLUSH`의 구분은 버퍼이벤트 타입에 따라 다르다.

반환값은 **실패 시 -1**, **플러시된 데이터 없음 0**, **일부 데이터 플러시 1**이다.

현재(2.0.5-beta 기준) `bufferevent_flush()`는 일부 버퍼이벤트 타입에서만 구현되어 있다. 특히 **소켓 기반 버퍼이벤트**에는 없다.

------

## 타입별 버퍼이벤트 함수

다음 버퍼이벤트 함수들은 **모든** 버퍼이벤트 타입에서 지원되지 않는다.

#### Interface

```c
int bufferevent_priority_set(struct bufferevent *bufev, int pri);
int bufferevent_get_priority(struct bufferevent *bufev);
```

`bufev` 구현에 사용되는 이벤트의 **우선순위**를 `pri`로 조정한다. 우선순위에 관한 더 자세한 내용은 `event_priority_set()`을 보라.

성공 시 **0**, 실패 시 **-1**을 반환한다. **소켓 기반 버퍼이벤트에서만** 동작한다.

`bufferevent_priority_set()`은 Libevent 1.0에서, `bufferevent_get_priority()`는 2.1.2-alpha에서 도입되었다.

#### Interface

```c
int bufferevent_setfd(struct bufferevent *bufev, evutil_socket_t fd);
evutil_socket_t bufferevent_getfd(struct bufferevent *bufev);
```

**FD 기반** 이벤트의 파일 디스크립터를 설정/반환한다. `setfd()`는 **소켓 기반 버퍼이벤트에서만** 지원된다. 둘 다 실패 시 **-1**을 반환하며, `setfd()`는 성공 시 **0**을 반환한다.

`bufferevent_setfd()`는 1.4.4에서, `bufferevent_getfd()`는 2.0.2-alpha에서 도입되었다.

#### Interface

```c
struct event_base *bufferevent_get_base(struct bufferevent *bev);
```

버퍼이벤트의 `event_base`를 반환한다. 2.0.9-rc에서 도입되었다.

#### Interface

```c
struct bufferevent *bufferevent_get_underlying(struct bufferevent *bufev);
```

다른 버퍼이벤트를 **전송(transport)**으로 사용하는 경우, 그 **하부 버퍼이벤트**를 반환한다. 이러한 상황은 **필터링 버퍼이벤트** 관련 주석을 보라.

이 함수는 2.0.2-alpha에서 도입되었다.

------

## 버퍼이벤트 수동 락/언락

`evbuffer`와 마찬가지로, 버퍼이벤트에서 여러 연산을 **원자적으로** 수행하고 싶을 때가 있다. Libevent는 버퍼이벤트를 **수동으로 락/언락**할 수 있는 함수를 노출한다.

#### Interface

```c
void bufferevent_lock(struct bufferevent *bufev);
void bufferevent_unlock(struct bufferevent *bufev);
```

버퍼이벤트가 생성 시 `BEV_OPT_THREADSAFE`로 만들어지지 않았거나, Libevent의 스레딩 지원이 활성화되지 않았다면, **락의 효과가 없다**.

이 함수로 버퍼이벤트를 락하면 **연결된 evbuffer들도 함께 락**된다. 함수들은 **재귀적**이어서, 이미 락을 보유한 버퍼이벤트를 다시 락해도 안전하다. 물론 **락 호출 수만큼 언락**을 호출해야 한다.

이 함수들은 Libevent 2.0.6-rc에서 도입되었다.

------

## 오래된(Obsolete) 버퍼이벤트 기능

Libevent 1.4와 2.0 사이에서 버퍼이벤트 백엔드 코드는 크게 개편되었다. 이전 인터페이스에서는 가끔 `struct bufferevent` 내부에 접근하는 것이 정상적이었고, 이 내부 접근에 의존한 매크로들이 사용되었다.

혼란스럽게도, 예전 코드에서는 버퍼이벤트 기능의 이름을 **“evbuffer” 접두사**로 붙여 부르기도 했다.

Libevent 2.0 이전에 무엇이라 불렸는지에 대한 간단한 가이드:

| Current name              | Old name           |
| ------------------------- | ------------------ |
| bufferevent_data_cb       | evbuffercb         |
| bufferevent_event_cb      | everrorcb          |
| BEV_EVENT_READING         | EVBUFFER_READ      |
| BEV_EVENT_WRITE           | EVBUFFER_WRITE     |
| BEV_EVENT_EOF             | EVBUFFER_EOF       |
| BEV_EVENT_ERROR           | EVBUFFER_ERROR     |
| BEV_EVENT_TIMEOUT         | EVBUFFER_TIMEOUT   |
| bufferevent_get_input(b)  | EVBUFFER_INPUT(b)  |
| bufferevent_get_output(b) | EVBUFFER_OUTPUT(b) |

이전 함수들은 `event2/bufferevent.h`가 아니라 **`event.h`**에 정의되어 있었다.

공통 부분의 내부에 **여전히 접근해야 한다면**, `event2/bufferevent_struct.h`를 포함할 수 있다. 그러나 **권장하지 않는다**: `struct bufferevent`의 내용은 Libevent 버전마다 **변한다**. 이 절의 매크로와 이름들은 `event2/bufferevent_compat.h`를 포함하면 사용할 수 있다.

예전 버전에서 버퍼이벤트를 설정하는 인터페이스는 달랐다:

#### Interface

```c
struct bufferevent *bufferevent_new(evutil_socket_t fd,
    evbuffercb readcb, evbuffercb writecb, everrorcb errorcb, void *cbarg);
int bufferevent_base_set(struct event_base *base, struct bufferevent *bufev);
```

`bufferevent_new()`는 **소켓 버퍼이벤트만** 생성하며, 더 이상 사용되지 않는 **“기본” event_base**에서 동작한다. `bufferevent_base_set`은 **소켓 버퍼이벤트만**의 `event_base`를 조정한다.

타임아웃은 `struct timeval`이 아니라 **초 단위 정수**로 설정했다:

#### Interface

```c
void bufferevent_settimeout(struct bufferevent *bufev,
    int timeout_read, int timeout_write);
```

마지막으로, Libevent 2.0 이전의 **하부 evbuffer 구현은 비효율적**이어서, **고성능 앱**에서 버퍼이벤트 사용이 **상당히 의문**시될 정도였다.