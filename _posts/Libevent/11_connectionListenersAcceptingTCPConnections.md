---
layout: single
title:  "Connection listeners: TCP 연결 수락"
date: 2025-08-21
tags: libevent
categories: 
   - Libevent
---

# Connection listeners: TCP 연결 수락

evconnlistener 메커니즘은 들어오는 TCP 연결을 수신하고 수락할 수 있는 방법을 제공합니다.

이 섹션의 모든 함수와 타입은 event2/listener.h에 선언되어 있으며, 특별한 언급이 없는 한 Libevent 2.0.2-alpha에서 처음 등장했습니다.

---

## evconnlistener 생성 또는 해제
### 인터페이스 

```c
struct evconnlistener *evconnlistener_new(struct event_base *base,
    evconnlistener_cb cb, void *ptr, unsigned flags, int backlog,
    evutil_socket_t fd);
struct evconnlistener *evconnlistener_new_bind(struct event_base *base,
    evconnlistener_cb cb, void *ptr, unsigned flags, int backlog,
    const struct sockaddr *sa, int socklen);
void evconnlistener_free(struct evconnlistener *lev);
```

두 evconnlistener_new*() 함수는 모두 새로운 connection listener 객체를 할당하고 반환합니다. 
connection listener는 event_base를 사용하여 지정된 리스닝 소켓에 새로운 TCP 연결이 생겼을 때 이를 감지합니다. 
새로운 연결이 들어오면, 지정한 콜백 함수가 호출됩니다. 

두 함수 모두에서 **base** 파라미터는 연결을 감시하기 위해 listener가 사용할 event_base입니다. 
**cb** 함수는 새 연결이 수신되었을 때 호출되는 콜백입니다. cb가 NULL이면, 콜백이 설정될 때까지 listener는 비활성 상태로 취급됩니다. 
**ptr** 포인터는 콜백에 전달됩니다. 
**flags** 인자는 listener의 동작을 제어합니다. 
**backlog** 파라미터는 네트워크 스택이 아직 accept되지 않은 상태에서 대기할 수 있는 최대 연결 수를 제어합니다. 시스템의 listen() 함수 문서를 참조하십시오. 
backlog가 음수이면, Libevent는 적절한 값을 선택합니다. 0이면 사용자가 이미 listen()을 호출했다고 가정합니다. 

두 함수의 차이점은 listener 소켓을 어떻게 설정하는가입니다. 
- **evconnlistener_new()** → 이미 포트에 소켓을 바인딩했으며 그 소켓 fd를 전달해야 함 
- **evconnlistener_new_bind()** → Libevent가 소켓을 직접 할당하고 바인딩하도록 할 때 사용 

팁: evconnlistener_new를 사용할 때는 리스닝 소켓이 반드시 논블로킹 모드여야 합니다. (evutil_make_socket_nonblocking 사용 권장) 블로킹 모드일 경우 정의되지 않은 동작이 발생할 수 있습니다. 

evconnlistener를 해제하려면 evconnlistener_free()를 호출하십시오. 

---

## 인식되는 플래그
evconnlistener_new()의 flags 인자에 전달할 수 있는 값들입니다. 여러 개를 OR로 조합할 수 있습니다. 

- **LEV_OPT_LEAVE_SOCKETS_BLOCKING** 
  기본적으로 listener는 새 소켓을 수락하면 논블로킹으로 설정합니다. 이 플래그를 사용하면 해당 동작을 하지 않습니다. 

- **LEV_OPT_CLOSE_ON_FREE** 
  listener가 해제될 때 소켓도 닫습니다. 

- **LEV_OPT_CLOSE_ON_EXEC** 
  listener 소켓에 close-on-exec 플래그를 설정합니다. 

- **LEV_OPT_REUSEABLE** 
  일부 플랫폼에서는 소켓이 닫힌 후 일정 시간 동안 같은 포트에 다시 바인딩할 수 없습니다. 이 옵션을 설정하면 소켓을 재사용 가능하게 설정합니다.

- **LEV_OPT_THREADSAFE** 
  멀티스레드 환경에서 안전하게 사용할 수 있도록 락을 할당합니다. (Libevent 2.0.8-rc 도입) 

- **LEV_OPT_DISABLED** 
  listener를 비활성 상태로 초기화합니다. 이후 evconnlistener_enable()로 활성화할 수 있습니다. (2.1.1-alpha) 

- **LEV_OPT_DEFERRED_ACCEPT** 
  가능하다면 커널이 데이터가 수신되기 전까지 연결을 알리지 않도록 설정합니다. 프로토콜이 클라이언트 전송 시작형일 경우에만 사용해야 하며, 일부 OS에서만 동작합니다. (2.1.1-alpha) 

---

## connection listener 콜백 
### 인터페이스 

```c
typedef void (*evconnlistener_cb)(struct evconnlistener *listener,
    evutil_socket_t sock, struct sockaddr *addr, int len, void *ptr);
```

새로운 연결이 수신되면 콜백 함수가 호출됩니다. 
- **listener**: 연결을 수신한 evconnlistener 
- **sock**: 새 소켓 
- **addr, len**: 연결을 수신한 주소와 길이 
- **ptr**: evconnlistener_new()에서 전달한 사용자 포인터 

---

## evconnlistener 활성화 및 비활성화 
### 인터페이스 

```c
int evconnlistener_disable(struct evconnlistener *lev); 
int evconnlistener_enable(struct evconnlistener *lev);  
```

listener를 임시로 비활성화하거나 다시 활성화할 수 있습니다. 

---

## evconnlistener의 콜백 변경 
### 인터페이스 

```c
void evconnlistener_set_cb(struct evconnlistener *lev, evconnlistener_cb cb, void *arg);
```

기존 evconnlistener의 콜백과 인자를 변경합니다. (2.0.9-rc) 

---

## evconnlistener 정보 조회 
### 인터페이스 

```c
evutil_socket_t evconnlistener_get_fd(struct evconnlistener *lev);  
struct event_base *evconnlistener_get_base(struct evconnlistener *lev);
```

- evconnlistener_get_fd() → listener의 소켓 FD 반환 (2.0.3-alpha) 
- evconnlistener_get_base() → listener의 event_base 반환 

---

## 에러 감지 
accept() 호출 실패 시 알림을 받을 수 있도록 에러 콜백을 설정할 수 있습니다. 
### 인터페이스 

```c
typedef void (*evconnlistener_errorcb)(struct evconnlistener *lis, void *ptr);  
void evconnlistener_set_error_cb(struct evconnlistener *lev, evconnlistener_errorcb errorcb);
```

에러 콜백은 listener와 evconnlistener_new()에서 전달한 사용자 포인터를 인자로 받습니다. 
(Libevent 2.0.8-rc 도입) 

---

## 예제 코드: 에코 서버

```c
#include <event2/listener.h>
#include <event2/bufferevent.h>
#include <event2/buffer.h>

#include <arpa/inet.h>

#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>

static void
echo_read_cb(struct bufferevent *bev, void *ctx)
{
        /* This callback is invoked when there is data to read on bev. */
        struct evbuffer *input = bufferevent_get_input(bev);
        struct evbuffer *output = bufferevent_get_output(bev);

        /* Copy all the data from the input buffer to the output buffer. */
        evbuffer_add_buffer(output, input);
}

static void
echo_event_cb(struct bufferevent *bev, short events, void *ctx)
{
        if (events & BEV_EVENT_ERROR)
                perror("Error from bufferevent");
        if (events & (BEV_EVENT_EOF | BEV_EVENT_ERROR)) {
                bufferevent_free(bev);
        }
}

static void
accept_conn_cb(struct evconnlistener *listener,
    evutil_socket_t fd, struct sockaddr *address, int socklen,
    void *ctx)
{
        /* We got a new connection! Set up a bufferevent for it. */
        struct event_base *base = evconnlistener_get_base(listener);
        struct bufferevent *bev = bufferevent_socket_new(
                base, fd, BEV_OPT_CLOSE_ON_FREE);

        bufferevent_setcb(bev, echo_read_cb, NULL, echo_event_cb, NULL);

        bufferevent_enable(bev, EV_READ|EV_WRITE);
}

static void
accept_error_cb(struct evconnlistener *listener, void *ctx)
{
        struct event_base *base = evconnlistener_get_base(listener);
        int err = EVUTIL_SOCKET_ERROR();
        fprintf(stderr, "Got an error %d (%s) on the listener. "
                "Shutting down.\n", err, evutil_socket_error_to_string(err));

        event_base_loopexit(base, NULL);
}

int
main(int argc, char **argv)
{
        struct event_base *base;
        struct evconnlistener *listener;
        struct sockaddr_in sin;

        int port = 9876;

        if (argc > 1) {
                port = atoi(argv[1]);
        }
        if (port<=0 || port>65535) {
                puts("Invalid port");
                return 1;
        }

        base = event_base_new();
        if (!base) {
                puts("Couldn't open event base");
                return 1;
        }

        /* Clear the sockaddr before using it, in case there are extra
         * platform-specific fields that can mess us up. */
        memset(&sin, 0, sizeof(sin));
        /* This is an INET address */
        sin.sin_family = AF_INET;
        /* Listen on 0.0.0.0 */
        sin.sin_addr.s_addr = htonl(0);
        /* Listen on the given port. */
        sin.sin_port = htons(port);

        listener = evconnlistener_new_bind(base, accept_conn_cb, NULL,
            LEV_OPT_CLOSE_ON_FREE|LEV_OPT_REUSEABLE, -1,
            (struct sockaddr*)&sin, sizeof(sin));
        if (!listener) {
                perror("Couldn't create listener");
                return 1;
        }
        evconnlistener_set_error_cb(listener, accept_error_cb);

        event_base_dispatch(base);
        return 0;
}
```



