---
layout: single
title:  "비동기 IO에 대한 작은 소개"
date: 2025-08-21
tags: libevent
categories: 
   - Libevent
---

# 비동기 IO에 대한 작은 소개

대부분의 초보 프로그래머는 블로킹 IO 호출부터 시작합니다. **동기(synchronous) IO 호출**이란, 호출이 완료되거나(성공/실패) 네트워크 스택이 타임아웃으로 포기할 때까지 **함수가 반환되지 않는** 호출을 말합니다. 예를 들어 TCP 연결에서 `connect()`를 호출하면, 운영체제는 상대 호스트로 SYN 패킷을 보낸 뒤 **SYN/ACK**를 받을 때까지(또는 포기할 때까지) 애플리케이션에 제어권을 돌려주지 않습니다.

아래는 블로킹 네트워크 호출을 사용하는 아주 간단한 클라이언트 예제입니다. 이 프로그램은 `www.google.com`에 연결하고, 간단한 HTTP 요청을 보낸 뒤, 응답을 stdout으로 출력합니다.

### 예제: 간단한 블로킹 HTTP 클라이언트

``` c
/* For sockaddr_in */
#include <netinet/in.h>
/* For socket functions */
#include <sys/socket.h>
/* For gethostbyname */
#include <netdb.h>

#include <unistd.h>
#include <string.h>
#include <stdio.h>

int main(int c, char **v)
{
    const char query[] =
        "GET / HTTP/1.0\r\n"
        "Host: www.google.com\r\n"
        "\r\n";
    const char hostname[] = "www.google.com";
    struct sockaddr_in sin;
    struct hostent *h;
    const char *cp;
    int fd;
    ssize_t n_written, remaining;
    char buf[1024];

    /* Look up the IP address for the hostname.   Watch out; this isn't
       threadsafe on most platforms. */
    h = gethostbyname(hostname);
    if (!h) {
        fprintf(stderr, "Couldn't lookup %s: %s", hostname, hstrerror(h_errno));
        return 1;
    }
    if (h->h_addrtype != AF_INET) {
        fprintf(stderr, "No ipv6 support, sorry.");
        return 1;
    }

    /* Allocate a new socket */
    fd = socket(AF_INET, SOCK_STREAM, 0);
    if (fd < 0) {
        perror("socket");
        return 1;
    }

    /* Connect to the remote host. */
    sin.sin_family = AF_INET;
    sin.sin_port = htons(80);
    sin.sin_addr = *(struct in_addr*)h->h_addr;
    if (connect(fd, (struct sockaddr*) &sin, sizeof(sin))) {
        perror("connect");
        close(fd);
        return 1;
    }

    /* Write the query. */
    /* XXX Can send succeed partially? */
    cp = query;
    remaining = strlen(query);
    while (remaining) {
      n_written = send(fd, cp, remaining, 0);
      if (n_written <= 0) {
        perror("send");
        return 1;
      }
      remaining -= n_written;
      cp += n_written;
    }

    /* Get an answer back. */
    while (1) {
        ssize_t result = recv(fd, buf, sizeof(buf), 0);
        if (result == 0) {
            break;
        } else if (result < 0) {
            perror("recv");
            close(fd);
            return 1;
        }
        fwrite(buf, 1, result, stdout);
    }

    close(fd);
    return 0;
}
```

위 코드의 모든 네트워크 호출은 **블로킹**입니다. `gethostbyname`은 호스트 이름을 성공/실패로 해석할 때까지 반환하지 않고, `connect`는 연결이 성립될 때까지 반환하지 않으며, `recv`는 데이터나 종료 신호를 받을 때까지, `send`는 최소한 커널의 송신 버퍼에 데이터가 들어갈 때까지 반환하지 않습니다.

블로킹 IO가 꼭 나쁜 것은 아닙니다. **그 사이에 프로그램이 할 일이 없다면** 블로킹 IO만으로도 충분히 동작합니다. 하지만 **여러 연결을 동시에** 처리해야 한다면 문제가 됩니다. 예를 들어, 두 연결에서 들어오는 입력을 읽되, **어느 쪽이 먼저 올지 모르는 상황**이라고 합시다. 다음과 같이 쓸 수는 없습니다.

### 잘못된 예

``` c
/* This won't work. */
char buf[1024];
int i, n;
while (i_still_want_to_read()) {
    for (i=0; i<n_sockets; ++i) {
        n = recv(fd[i], buf, sizeof(buf), 0);
        if (n==0)
            handle_close(fd[i]);
        else if (n<0)
            handle_error(fd[i], errno);
        else
            handle_input(fd[i], buf, n);
    }
}
```

왜냐하면 `fd[2]`에 데이터가 먼저 도착해도, 프로그램은 `fd[0]`, `fd[1]`의 `recv`가 반환될 때까지 `fd[2]`를 시도조차 하지 않기 때문입니다.

이를 해결하려고 **멀티스레딩**이나 **멀티프로세스 서버**를 쓰는 경우가 많습니다. 가장 간단한 방식 중 하나는 **연결마다 별도의 프로세스(또는 스레드)**를 만들어 처리하는 것입니다. 각 연결이 독립 실행되므로, 한 연결의 블로킹 IO가 다른 연결을 막지 않습니다.

아래는 또 다른 예제입니다. 포트 **40713**에서 TCP를 수신하고, 입력을 **줄 단위**로 읽어 **ROT13**으로 변환해 돌려주는 **포킹(forking) 서버**입니다. 들어오는 연결마다 `fork()`로 새 프로세스를 만듭니다.

### 예제: 포킹 ROT13 서버

``` c
/* For sockaddr_in */
#include <netinet/in.h>
/* For socket functions */
#include <sys/socket.h>

#include <unistd.h>
#include <string.h>
#include <stdio.h>
#include <stdlib.h>

#define MAX_LINE 16384

char
rot13_char(char c)
{
    /* We don't want to use isalpha here; setting the locale would change
     * which characters are considered alphabetical. */
    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
        return c + 13;
    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
        return c - 13;
    else
        return c;
}

void
child(int fd)
{
    char outbuf[MAX_LINE+1];
    size_t outbuf_used = 0;
    ssize_t result;

    while (1) {
        char ch;
        result = recv(fd, &ch, 1, 0);
        if (result == 0) {
            break;
        } else if (result == -1) {
            perror("read");
            break;
        }

        /* We do this test to keep the user from overflowing the buffer. */
        if (outbuf_used < sizeof(outbuf)) {
            outbuf[outbuf_used++] = rot13_char(ch);
        }

        if (ch == '\n') {
            send(fd, outbuf, outbuf_used, 0);
            outbuf_used = 0;
            continue;
        }
    }
}

void
run(void)
{
    int listener;
    struct sockaddr_in sin;

    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = 0;
    sin.sin_port = htons(40713);

    listener = socket(AF_INET, SOCK_STREAM, 0);

#ifndef WIN32
    {
        int one = 1;
        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
    }
#endif

    if (bind(listener, (struct sockaddr*)&sin, sizeof(sin)) < 0) {
        perror("bind");
        return;
    }

    if (listen(listener, 16)<0) {
        perror("listen");
        return;
    }



    while (1) {
        struct sockaddr_storage ss;
        socklen_t slen = sizeof(ss);
        int fd = accept(listener, (struct sockaddr*)&ss, &slen);
        if (fd < 0) {
            perror("accept");
        } else {
            if (fork() == 0) {
                child(fd);
                exit(0);
            }
        }
    }
}

int
main(int c, char **v)
{
    run();
    return 0;
}
```

그렇다면 이 방식이 여러 연결을 동시에 처리하는 **완벽한 해결책**일까요? 아쉽게도 아닙니다. 첫째, 일부 플랫폼에서는 **프로세스/스레드 생성 비용**이 상당합니다. 실제로는 매번 새로 만들기보다 **스레드 풀**을 쓰는 편이 낫습니다. 더 근본적으로, **매 연결당 스레드** 방식은 수천~수만 연결 규모에서 **스케일이 잘 안 됩니다**. CPU당 소수 스레드로 운영하는 편이 더 효율적입니다.

그렇다면 스레딩 말고 대안은? **유닉스 패러다임**에서는 소켓을 **논블로킹**으로 만듭니다:

``` c
fcntl(fd, F_SETFL, O_NONBLOCK);
```

이후 `fd`에 대한 네트워크 호출은 **즉시 완료**되거나, 지금은 진행할 수 없으니 **나중에 다시 시도하라**는 에러(예: `EAGAIN`)로 **바로 반환**합니다. 이를 단순히 모든 소켓에 반복 적용하면 아래와 같은 **바쁜 대기(busy-polling)** 코드가 됩니다.

### 나쁜 예: 모든 소켓 바쁜-폴링

``` c
/* This will work, but the performance will be unforgivably bad. */
int i, n;
char buf[1024];
for (i=0; i < n_sockets; ++i)
    fcntl(fd[i], F_SETFL, O_NONBLOCK);

while (i_still_want_to_read()) {
    for (i=0; i < n_sockets; ++i) {
        n = recv(fd[i], buf, sizeof(buf), 0);
        if (n == 0) {
            handle_close(fd[i]);
        } else if (n < 0) {
            if (errno == EAGAIN)
                 ; /* The kernel didn't have any data for us to read. */
            else
                 handle_error(fd[i], errno);
         } else {
            handle_input(fd[i], buf, n);
         }
    }
}
```

이 코드는 **동작은 하지만 성능이 최악**입니다. 이유는 두 가지입니다. (1) 읽을 데이터가 없을 때 루프가 **쓸데없이 CPU를 계속 사용**하고, (2) 실제 데이터 유무와 상관없이 **모든 소켓마다 커널 호출**을 하기 때문입니다.
 따라서 우리는 커널에 **“이 소켓들 중 하나가 준비되면 알려줘”**라고 말할 방법이 필요합니다.

가장 오래되고 널리 쓰이는 해결책이 **`select()`**입니다. `select()`는 읽기/쓰기/예외용 **fd 집합**(비트 배열)을 받아, **준비된 소켓만** 집합에 남겨줍니다.

다음은 `select()`를 사용하는 예시입니다.

### 예제: `select` 사용

``` c
/* If you only have a couple dozen fds, this version won't be awful */
fd_set readset;
int i, n;
char buf[1024];

while (i_still_want_to_read()) {
    int maxfd = -1;
    FD_ZERO(&readset);

    /* Add all of the interesting fds to readset */
    for (i=0; i < n_sockets; ++i) {
         if (fd[i]>maxfd) maxfd = fd[i];
         FD_SET(fd[i], &readset);
    }

    /* Wait until one or more fds are ready to read */
    select(maxfd+1, &readset, NULL, NULL, NULL);

    /* Process all of the fds that are still set in readset */
    for (i=0; i < n_sockets; ++i) {
        if (FD_ISSET(fd[i], &readset)) {
            n = recv(fd[i], buf, sizeof(buf), 0);
            if (n == 0) {
                handle_close(fd[i]);
            } else if (n < 0) {
                if (errno == EAGAIN)
                     ; /* The kernel didn't have any data for us to read. */
                else
                     handle_error(fd[i], errno);
             } else {
                handle_input(fd[i], buf, n);
             }
        }
    }
}
```

아래는 `select()` 기반으로 다시 구현한 ROT13 서버입니다.

### 예제: `select()` 기반 ROT13 서버

``` c
/* For sockaddr_in */
#include <netinet/in.h>
/* For socket functions */
#include <sys/socket.h>
/* For fcntl */
#include <fcntl.h>
/* for select */
#include <sys/select.h>

#include <assert.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>

#define MAX_LINE 16384

char
rot13_char(char c)
{
    /* We don't want to use isalpha here; setting the locale would change
     * which characters are considered alphabetical. */
    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
        return c + 13;
    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
        return c - 13;
    else
        return c;
}

struct fd_state {
    char buffer[MAX_LINE];
    size_t buffer_used;

    int writing;
    size_t n_written;
    size_t write_upto;
};

struct fd_state *
alloc_fd_state(void)
{
    struct fd_state *state = malloc(sizeof(struct fd_state));
    if (!state)
        return NULL;
    state->buffer_used = state->n_written = state->writing =
        state->write_upto = 0;
    return state;
}

void
free_fd_state(struct fd_state *state)
{
    free(state);
}

void
make_nonblocking(int fd)
{
    fcntl(fd, F_SETFL, O_NONBLOCK);
}

int
do_read(int fd, struct fd_state *state)
{
    char buf[1024];
    int i;
    ssize_t result;
    while (1) {
        result = recv(fd, buf, sizeof(buf), 0);
        if (result <= 0)
            break;

        for (i=0; i < result; ++i)  {
            if (state->buffer_used < sizeof(state->buffer))
                state->buffer[state->buffer_used++] = rot13_char(buf[i]);
            if (buf[i] == '\n') {
                state->writing = 1;
                state->write_upto = state->buffer_used;
            }
        }
    }

    if (result == 0) {
        return 1;
    } else if (result < 0) {
        if (errno == EAGAIN)
            return 0;
        return -1;
    }

    return 0;
}

int
do_write(int fd, struct fd_state *state)
{
    while (state->n_written < state->write_upto) {
        ssize_t result = send(fd, state->buffer + state->n_written,
                              state->write_upto - state->n_written, 0);
        if (result < 0) {
            if (errno == EAGAIN)
                return 0;
            return -1;
        }
        assert(result != 0);

        state->n_written += result;
    }

    if (state->n_written == state->buffer_used)
        state->n_written = state->write_upto = state->buffer_used = 0;

    state->writing = 0;

    return 0;
}

void
run(void)
{
    int listener;
    struct fd_state *state[FD_SETSIZE];
    struct sockaddr_in sin;
    int i, maxfd;
    fd_set readset, writeset, exset;

    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = 0;
    sin.sin_port = htons(40713);

    for (i = 0; i < FD_SETSIZE; ++i)
        state[i] = NULL;

    listener = socket(AF_INET, SOCK_STREAM, 0);
    make_nonblocking(listener);

#ifndef WIN32
    {
        int one = 1;
        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
    }
#endif

    if (bind(listener, (struct sockaddr*)&sin, sizeof(sin)) < 0) {
        perror("bind");
        return;
    }

    if (listen(listener, 16)<0) {
        perror("listen");
        return;
    }

    FD_ZERO(&readset);
    FD_ZERO(&writeset);
    FD_ZERO(&exset);

    while (1) {
        maxfd = listener;

        FD_ZERO(&readset);
        FD_ZERO(&writeset);
        FD_ZERO(&exset);

        FD_SET(listener, &readset);

        for (i=0; i < FD_SETSIZE; ++i) {
            if (state[i]) {
                if (i > maxfd)
                    maxfd = i;
                FD_SET(i, &readset);
                if (state[i]->writing) {
                    FD_SET(i, &writeset);
                }
            }
        }

        if (select(maxfd+1, &readset, &writeset, &exset, NULL) < 0) {
            perror("select");
            return;
        }

        if (FD_ISSET(listener, &readset)) {
            struct sockaddr_storage ss;
            socklen_t slen = sizeof(ss);
            int fd = accept(listener, (struct sockaddr*)&ss, &slen);
            if (fd < 0) {
                perror("accept");
            } else if (fd > FD_SETSIZE) {
                close(fd);
            } else {
                make_nonblocking(fd);
                state[fd] = alloc_fd_state();
                assert(state[fd]);/*XXX*/
            }
        }

        for (i=0; i < maxfd+1; ++i) {
            int r = 0;
            if (i == listener)
                continue;

            if (FD_ISSET(i, &readset)) {
                r = do_read(i, state[i]);
            }
            if (r == 0 && FD_ISSET(i, &writeset)) {
                r = do_write(i, state[i]);
            }
            if (r) {
                free_fd_state(state[i]);
                state[i] = NULL;
                close(i);
            }
        }
    }
}

int
main(int c, char **v)
{
    setvbuf(stdout, NULL, _IONBF, 0);

    run();
    return 0;
}
```

하지만 이것도 완벽하진 않습니다. `select()`의 비트 배열 생성/해석 비용이 **가장 큰 fd 값에 비례**하기 때문에, 소켓 수가 많아지면 `select()`의 **스케일링이 매우 나쁩니다**.

운영체제들은 `select` 대체로 **`poll()`**, **`epoll()`(리눅스), **`kqueue()`(BSD/다윈), **`evports`·`/dev/poll`**(솔라리스) 등을 제공합니다. 이들은 `select`보다 성능이 더 좋고, `poll()`을 제외하면 소켓 추가/삭제/준비 감지가 **O(1)**에 가깝습니다.

문제는 **표준이 제각각**이라는 점입니다. 리눅스는 `epoll`, BSD는 `kqueue`, 솔라리스는 `evports`/`/dev/poll`… 서로 호환되지 않습니다. **이식성 높은 고성능 비동기 애플리케이션**을 원한다면, 각 인터페이스 위를 감싸서 **가용한 최적의 것**을 쓰게 해주는 **추상화**가 필요합니다.

그 역할을 **Libevent**의 **가장 낮은 레벨 API**가 수행합니다. Libevent는 가능한 경우 **가장 효율적인 구현**(select/poll/epoll/kqueue 등)을 사용하면서 **일관된 인터페이스**를 제공합니다.

아래는 Libevent 2를 사용해 만든 ROT13 서버입니다. `fd_set`이 사라지고, 대신 **`struct event_base`에 이벤트를 등록/해제**합니다. 내부적으로는 select/poll/epoll/kqueue 중 하나가 쓰입니다.

### 예제: Libevent를 이용한 저수준 ROT13 서버

``` c
/* For sockaddr_in */
#include <netinet/in.h>
/* For socket functions */
#include <sys/socket.h>
/* For fcntl */
#include <fcntl.h>

#include <event2/event.h>

#include <assert.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>

#define MAX_LINE 16384

void do_read(evutil_socket_t fd, short events, void *arg);
void do_write(evutil_socket_t fd, short events, void *arg);

char
rot13_char(char c)
{
    /* We don't want to use isalpha here; setting the locale would change
     * which characters are considered alphabetical. */
    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
        return c + 13;
    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
        return c - 13;
    else
        return c;
}

struct fd_state {
    char buffer[MAX_LINE];
    size_t buffer_used;

    size_t n_written;
    size_t write_upto;

    struct event *read_event;
    struct event *write_event;
};

struct fd_state *
alloc_fd_state(struct event_base *base, evutil_socket_t fd)
{
    struct fd_state *state = malloc(sizeof(struct fd_state));
    if (!state)
        return NULL;
    state->read_event = event_new(base, fd, EV_READ|EV_PERSIST, do_read, state);
    if (!state->read_event) {
        free(state);
        return NULL;
    }
    state->write_event =
        event_new(base, fd, EV_WRITE|EV_PERSIST, do_write, state);

    if (!state->write_event) {
        event_free(state->read_event);
        free(state);
        return NULL;
    }

    state->buffer_used = state->n_written = state->write_upto = 0;

    assert(state->write_event);
    return state;
}

void
free_fd_state(struct fd_state *state)
{
    event_free(state->read_event);
    event_free(state->write_event);
    free(state);
}

void
do_read(evutil_socket_t fd, short events, void *arg)
{
    struct fd_state *state = arg;
    char buf[1024];
    int i;
    ssize_t result;
    while (1) {
        assert(state->write_event);
        result = recv(fd, buf, sizeof(buf), 0);
        if (result <= 0)
            break;

        for (i=0; i < result; ++i)  {
            if (state->buffer_used < sizeof(state->buffer))
                state->buffer[state->buffer_used++] = rot13_char(buf[i]);
            if (buf[i] == '\n') {
                assert(state->write_event);
                event_add(state->write_event, NULL);
                state->write_upto = state->buffer_used;
            }
        }
    }

    if (result == 0) {
        free_fd_state(state);
    } else if (result < 0) {
        if (errno == EAGAIN) // XXXX use evutil macro
            return;
        perror("recv");
        free_fd_state(state);
    }
}

void
do_write(evutil_socket_t fd, short events, void *arg)
{
    struct fd_state *state = arg;

    while (state->n_written < state->write_upto) {
        ssize_t result = send(fd, state->buffer + state->n_written,
                              state->write_upto - state->n_written, 0);
        if (result < 0) {
            if (errno == EAGAIN) // XXX use evutil macro
                return;
            free_fd_state(state);
            return;
        }
        assert(result != 0);

        state->n_written += result;
    }

    if (state->n_written == state->buffer_used)
        state->n_written = state->write_upto = state->buffer_used = 1;

    event_del(state->write_event);
}

void
do_accept(evutil_socket_t listener, short event, void *arg)
{
    struct event_base *base = arg;
    struct sockaddr_storage ss;
    socklen_t slen = sizeof(ss);
    int fd = accept(listener, (struct sockaddr*)&ss, &slen);
    if (fd < 0) { // XXXX eagain??
        perror("accept");
    } else if (fd > FD_SETSIZE) {
        close(fd); // XXX replace all closes with EVUTIL_CLOSESOCKET */
    } else {
        struct fd_state *state;
        evutil_make_socket_nonblocking(fd);
        state = alloc_fd_state(base, fd);
        assert(state); /*XXX err*/
        assert(state->write_event);
        event_add(state->read_event, NULL);
    }
}

void
run(void)
{
    evutil_socket_t listener;
    struct sockaddr_in sin;
    struct event_base *base;
    struct event *listener_event;

    base = event_base_new();
    if (!base)
        return; /*XXXerr*/

    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = 0;
    sin.sin_port = htons(40713);

    listener = socket(AF_INET, SOCK_STREAM, 0);
    evutil_make_socket_nonblocking(listener);

#ifndef WIN32
    {
        int one = 1;
        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
    }
#endif

    if (bind(listener, (struct sockaddr*)&sin, sizeof(sin)) < 0) {
        perror("bind");
        return;
    }

    if (listen(listener, 16)<0) {
        perror("listen");
        return;
    }

    listener_event = event_new(base, listener, EV_READ|EV_PERSIST, do_accept, (void*)base);
    /*XXX check it */
    event_add(listener_event, NULL);

    event_base_dispatch(base);
}

int
main(int c, char **v)
{
    setvbuf(stdout, NULL, _IONBF, 0);

    run();
    return 0;
}
```

코드에서 눈여겨볼 점: 소켓 타입을 `int` 대신 **`evutil_socket_t`**로, 논블로킹 설정도 `fcntl(O_NONBLOCK)` 대신 **`evutil_make_socket_nonblocking`**을 사용합니다. 이렇게 하면 **Win32 네트워킹 API 차이**까지 고려한 이식성을 확보할 수 있습니다.



## 편의성은 어떨까? (그리고 Windows는?)

코드가 **효율적일수록 복잡해진** 것도 눈치채셨을 겁니다. 포킹할 때는 연결마다 **스택 버퍼**를 쓰면 됐고, **읽기/쓰기 상태 관리**도 코드 흐름으로 자연스럽게 표현됐습니다. 부분 완료 진행률도 **루프와 지역변수**로 충분했죠.

또한 Windows 전문가라면 위 Libevent 저수준 예제가 **Windows 최적 성능(IOCP)**을 직접 활용하지 못한다는 걸 알 겁니다. Windows의 고성능 비동기는 select류가 아니라 **IOCP(Completion Port)**를 사용합니다. IOCP는 **소켓 준비 알림**이 아니라 **작업 완료 알림** 모델이죠.

다행히 **Libevent 2의 `bufferevent` 인터페이스**가 이 두 문제를 해결합니다. 코드가 훨씬 단순해지고, **Windows/Unix 모두에서 효율적**인 구현을 사용합니다.

아래는 `bufferevent` API로 다시 작성한 ROT13 서버입니다.

## 예제: Libevent `bufferevent`로 더 단순한 ROT13 서버

``` c
/* For sockaddr_in */
#include <netinet/in.h>
/* For socket functions */
#include <sys/socket.h>
/* For fcntl */
#include <fcntl.h>

#include <event2/event.h>
#include <event2/buffer.h>
#include <event2/bufferevent.h>

#include <assert.h>
#include <unistd.h>
#include <string.h>
#include <stdlib.h>
#include <stdio.h>
#include <errno.h>

#define MAX_LINE 16384

void do_read(evutil_socket_t fd, short events, void *arg);
void do_write(evutil_socket_t fd, short events, void *arg);

char
rot13_char(char c)
{
    /* We don't want to use isalpha here; setting the locale would change
     * which characters are considered alphabetical. */
    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
        return c + 13;
    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
        return c - 13;
    else
        return c;
}

void
readcb(struct bufferevent *bev, void *ctx)
{
    struct evbuffer *input, *output;
    char *line;
    size_t n;
    int i;
    input = bufferevent_get_input(bev);
    output = bufferevent_get_output(bev);

    while ((line = evbuffer_readln(input, &n, EVBUFFER_EOL_LF))) {
        for (i = 0; i < n; ++i)
            line[i] = rot13_char(line[i]);
        evbuffer_add(output, line, n);
        evbuffer_add(output, "\n", 1);
        free(line);
    }

    if (evbuffer_get_length(input) >= MAX_LINE) {
        /* Too long; just process what there is and go on so that the buffer
         * doesn't grow infinitely long. */
        char buf[1024];
        while (evbuffer_get_length(input)) {
            int n = evbuffer_remove(input, buf, sizeof(buf));
            for (i = 0; i < n; ++i)
                buf[i] = rot13_char(buf[i]);
            evbuffer_add(output, buf, n);
        }
        evbuffer_add(output, "\n", 1);
    }
}

void
errorcb(struct bufferevent *bev, short error, void *ctx)
{
    if (error & BEV_EVENT_EOF) {
        /* connection has been closed, do any clean up here */
        /* ... */
    } else if (error & BEV_EVENT_ERROR) {
        /* check errno to see what error occurred */
        /* ... */
    } else if (error & BEV_EVENT_TIMEOUT) {
        /* must be a timeout event handle, handle it */
        /* ... */
    }
    bufferevent_free(bev);
}

void
do_accept(evutil_socket_t listener, short event, void *arg)
{
    struct event_base *base = arg;
    struct sockaddr_storage ss;
    socklen_t slen = sizeof(ss);
    int fd = accept(listener, (struct sockaddr*)&ss, &slen);
    if (fd < 0) {
        perror("accept");
    } else if (fd > FD_SETSIZE) {
        close(fd);
    } else {
        struct bufferevent *bev;
        evutil_make_socket_nonblocking(fd);
        bev = bufferevent_socket_new(base, fd, BEV_OPT_CLOSE_ON_FREE);
        bufferevent_setcb(bev, readcb, NULL, errorcb, NULL);
        bufferevent_setwatermark(bev, EV_READ, 0, MAX_LINE);
        bufferevent_enable(bev, EV_READ|EV_WRITE);
    }
}

void
run(void)
{
    evutil_socket_t listener;
    struct sockaddr_in sin;
    struct event_base *base;
    struct event *listener_event;

    base = event_base_new();
    if (!base)
        return; /*XXXerr*/

    sin.sin_family = AF_INET;
    sin.sin_addr.s_addr = 0;
    sin.sin_port = htons(40713);

    listener = socket(AF_INET, SOCK_STREAM, 0);
    evutil_make_socket_nonblocking(listener);

#ifndef WIN32
    {
        int one = 1;
        setsockopt(listener, SOL_SOCKET, SO_REUSEADDR, &one, sizeof(one));
    }
#endif

    if (bind(listener, (struct sockaddr*)&sin, sizeof(sin)) < 0) {
        perror("bind");
        return;
    }

    if (listen(listener, 16)<0) {
        perror("listen");
        return;
    }

    listener_event = event_new(base, listener, EV_READ|EV_PERSIST, do_accept, (void*)base);
    /*XXX check it */
    event_add(listener_event, NULL);

    event_base_dispatch(base);
}

int
main(int c, char **v)
{
    setvbuf(stdout, NULL, _IONBF, 0);

    run();
    return 0;
}
```

## 성능은 실제로 어느 정도일까?

XXXX 이 부분(효율성 섹션)은 작성 예정입니다. Libevent 페이지의 벤치마크는 꽤 오래되었습니다.



## Libevent를 이용한 저수준 ROT13 서버 와 예제: Libevent bufferevent로 더 단순한 ROT13 서버 의 차이

- **프로그래밍 레벨**
  - 저수준 ROT13: `event_new`로 **EV_READ/EV_WRITE 이벤트를 직접** 만들고, **소켓 비동기/버퍼/부분전송 상태**를 모두 수작업 관리.
  - bufferevent ROT13: `bufferevent_socket_new`로 **읽기/쓰기/에러 콜백만** 제공하면, **버퍼 관리·부분전송·상태 전이**를 Libevent가 대신 처리.
- **버퍼 관리**
  - 저수준: `struct fd_state`에 직접 `buffer`, `buffer_used`, `n_written`, `write_upto` 등을 들고 **언제 쓰기 이벤트를 켜고 끌지(event_add/del)**까지 코딩.
  - bufferevent: `struct evbuffer`(input/output)를 사용. **줄 읽기(`evbuffer_readln`)**, **부분 읽기/쓰기**, **워터마크(저/고 수위)** 같은 고급 버퍼 기능이 **내장**.
- **상태·흐름 제어**
  - 저수준: `EV_READ`에서 데이터 모으고, 개행 만나면 `EV_WRITE`를 **수동으로 활성화**. 전송이 끝나면 다시 **수동 비활성화**. 상태값도 직접 갱신.
  - bufferevent: 읽기 콜백에서 `input`을 가공해 `output`에 넣으면 **자동으로** 쓰기 가능 시점에 송신. **워터마크**로 혼잡 제어/역압(backpressure)도 쉬움.
- **에러·종료 처리**
  - 저수준: `recv/send` 에러 코드별 처리 + 소켓/이벤트/메모리 **직접 해제**.
  - bufferevent: `errorcb` 한 곳에서 **EOF/ERROR/TIMEOUT**를 구분 처리, `bufferevent_free()`로 자원 정리 **일원화**.
- **플랫폼 적합성(특히 Windows)**
  - 저수준: 내부적으로 select/epoll/kqueue 등을 쓰지만, **Windows의 IOCP** 특성을 **직접 활용**하긴 어려움.
  - bufferevent: Libevent가 같은 API로 **Windows(IOCP)**, **Unix(epoll/kqueue)** 모두에서 **최적화된 백엔드**를 붙여줌 → **휴대성 + 성능** 동시 확보.
- **타이머/시간 제한**
  - 저수준: 별도 `struct event`로 타이머를 만들어 직접 관리.
  - bufferevent: **읽기/쓰기 타임아웃**을 옵션으로 쉽게 설정 가능(`bufferevent_set_timeouts`).
- **TLS/필터/레이트리밋**
  - 저수준: 직접 OpenSSL 소켓 감싸고 읽기/쓰기/버퍼를 구현해야 함.
  - bufferevent: `bufferevent_openssl_*`로 **TLS** 추가가 간단. **필터 bufferevent**로 스트림 변환(예: 압축/암복호) 가능, **전송 레이트 제한**도 제공.
- **코드 복잡도 & 유지보수성**
  - 저수준: 라인 수 많고, 상태 버그(경계조건, 부분전송, 이벤트 토글) 발생 위험 ↑.
  - bufferevent: 코드가 **짧고 읽기 쉬움**. 실수 여지 ↓, 구현 속도 ↑.

### 언제 무엇을 쓸까?

- **저수준(event/EV_READ/EV_WRITE) 권장 상황**
  - 극단적으로 특이한 IO 패턴을 직접 최적화해야 할 때
  - 버퍼 정책을 **완전히 커스텀**해야 할 때
  - 학습/디버깅 목적으로 이벤트 루프의 **미시적 동작**을 보고 싶을 때
- **bufferevent 권장 상황(대부분)**
  - 일반적인 TCP 서버/클라이언트
  - **라인/프레임 단위** 처리, 스트림 변환
  - **Windows+Linux**를 모두 지원해야 할 때
  - **TLS**, **타임아웃**, **역압**, **레이트리밋**이 필요할 때
  - “빠르게 안정적인 코드”가 목표일 때

### 예제를 기준으로 본 차이 포인트

- **저수준 ROT13**:
  - `struct fd_state`에 버퍼와 진행 상태를 직접 보관
  - `event_add(state->write_event)` / `event_del(...)`를 **직접 호출**
  - `recv/send`의 **부분 완료**를 루프/인덱스로 수동 처리
  - 종료 시 **read/write event**와 **state free**를 각각 처리
- **bufferevent ROT13**:
  - `readcb()`에서 `evbuffer_readln()`으로 **줄 단위 파싱**을 간단 처리
  - 결과는 `output`에 `evbuffer_add()`로 쓰면 끝 → 송신은 프레임워크가 처리
  - **워터마크**로 입력이 너무 길어질 때의 처리(혼잡 제어)가 쉬움
  - `errorcb()` 한 곳에서 **EOF/ERROR/TIMEOUT** 처리 및 `bufferevent_free()`로 정리

### 결론 (요약 한 줄)

- **저수준 이벤트**: “**모든 것을 직접 제어**하고 싶은 파워 유저용 스패너”
- **bufferevent**: “**안전 장치가 많은 전동 공구**—빠르고 실수 적고 이식성 좋음”

