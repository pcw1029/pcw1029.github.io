---
layout: single
title:  "Evbuffers: 버퍼링된 IO를 위한 유틸리티 기능"
date: 2025-08-21
tags: libevent
categories: 
   - Libevent
---

# Evbuffers: 버퍼링된 IO를 위한 유틸리티 기능

Libevent의 evbuffer 기능은 바이트 큐를 구현하며, 끝에 데이터를 추가하고 앞에서 제거하는 데 최적화되어 있습니다.

evbuffer는 버퍼링된 네트워크 IO에서 “버퍼” 역할을 일반적으로 수행하도록 설계되었습니다. IO를 스케줄링하거나 준비되었을 때 IO를 트리거하는 함수들은 제공하지 않습니다. 그런 일은 bufferevent가 담당합니다.

이 장의 함수들은 별도 언급이 없는 한 `event2/buffer.h`에 선언되어 있습니다.
## Creating or freeing an evbuffer  
### Interface

```c
struct evbuffer *evbuffer_new(void);
void evbuffer_free(struct evbuffer *buf);
```

이 함수들은 비교적 명확합니다. `evbuffer_new()`는 새 빈 evbuffer를 할당하여 반환하고, `evbuffer_free()`는 해당 evbuffer와 그 안의 모든 내용을 삭제합니다.

이 함수들은 Libevent 0.8부터 존재했습니다.
## Evbuffers and Thread-safety  
### Interface

```c
int evbuffer_enable_locking(struct evbuffer *buf, void *lock);
void evbuffer_lock(struct evbuffer *buf);
void evbuffer_unlock(struct evbuffer *buf);
```

기본적으로 하나의 evbuffer를 여러 스레드에서 동시에 접근하는 것은 안전하지 않습니다. 이런 필요가 있다면 해당 evbuffer에 `evbuffer_enable_locking()`을 호출할 수 있습니다. `lock` 인자가 NULL이면, Libevent는 `evthread_set_lock_creation_callback`에 제공된 락 생성 함수를 사용해 새 락을 할당합니다. NULL이 아니면 그 인자를 락으로 사용합니다.

`evbuffer_lock()`/`evbuffer_unlock()`은 각각 evbuffer의 락을 획득/해제합니다. 여러 연산을 원자적으로 묶고 싶을 때 사용할 수 있습니다. evbuffer에서 락킹을 활성화하지 않았다면 이 함수들은 아무 일도 하지 않습니다.

(참고: 개별 연산 주변에 일일이 `evbuffer_lock()`/`evbuffer_unlock()`을 감쌀 필요는 없습니다. evbuffer에 락킹이 켜져 있으면 개별 연산은 이미 원자적입니다. 다만 여러 연산을 다른 스레드가 끼어들지 못하게 묶어야 할 때 수동으로 락을 거세요.)

이 함수들은 모두 Libevent 2.0.1-alpha에 도입되었습니다.
## Inspecting an evbuffer  

### Interface

```c
size_t evbuffer_get_length(const struct evbuffer *buf);
```

이 함수는 evbuffer에 저장된 바이트 수를 반환합니다.

Libevent 2.0.1-alpha에서 처음 도입되었습니다.
### Interface

```c
size_t evbuffer_get_contiguous_space(const struct evbuffer *buf);
```

이 함수는 evbuffer의 **앞부분**에 연속적으로 저장된 바이트 수를 반환합니다. evbuffer의 바이트들은 여러 개의 분리된 메모리 청크에 들어 있을 수 있습니다. 이 함수는 첫 번째 청크에 현재 연속 저장된 바이트 수를 알려줍니다.

Libevent 2.0.1-alpha에서 처음 도입되었습니다.
## Adding data to an evbuffer: basics  
### Interface

```c
int evbuffer_add(struct evbuffer *buf, const void *data, size_t datlen);
```

`data`의 `datlen` 바이트를 `buf`의 **끝**에 덧붙입니다. 성공 시 0, 실패 시 -1을 반환합니다.
### Interface

```c
int evbuffer_add_printf(struct evbuffer *buf, const char *fmt, ...);
int evbuffer_add_vprintf(struct evbuffer *buf, const char *fmt, va_list ap);
```

서식화된 데이터를 `buf`의 끝에 덧붙입니다. `printf`/`vprintf`와 동일한 방식으로 동작합니다. 반환값은 추가된 바이트 수입니다.
### Interface

```c
int evbuffer_expand(struct evbuffer *buf, size_t datlen);
```

버퍼의 마지막 청크를 조정하거나 새 청크를 추가하여, 추가 할당 없이 `datlen` 바이트를 담을 수 있도록 버퍼를 확장합니다.
### Examples

```c
/* "Hello world 2.0.1"를 버퍼에 추가하는 두 가지 방법 */
/* 직접 추가: */
evbuffer_add(buf, "Hello world 2.0.1", 17);

/* printf 사용: */
evbuffer_add_printf(buf, "Hello %s %d.%d.%d", "world", 2, 0, 1);
```

`evbuffer_add()`와 `evbuffer_add_printf()`는 Libevent 0.8에 도입, `evbuffer_expand()`는 0.9에, `evbuffer_add_vprintf()`는 1.1에서 처음 등장했습니다.
## Moving data from one evbuffer to another

효율을 위해, Libevent는 evbuffer 간 데이터를 옮기는 최적화된 함수를 제공합니다.
### Interface

```c
int evbuffer_add_buffer(struct evbuffer *dst, struct evbuffer *src);
int evbuffer_remove_buffer(struct evbuffer *src, struct evbuffer *dst,
    size_t datlen);
```

`evbuffer_add_buffer()`는 `src`의 모든 데이터를 `dst`의 끝으로 **이동**합니다. 성공 시 0, 실패 시 -1.

`evbuffer_remove_buffer()`는 `src`에서 정확히 `datlen` 바이트를 가능한 적게 복사하면서 `dst`의 끝으로 이동시킵니다. 옮길 바이트가 부족하면 있는 만큼만 옮깁니다. 반환값은 실제 옮긴 바이트 수입니다.

`evbuffer_add_buffer()`는 Libevent 0.8, `evbuffer_remove_buffer()`는 2.0.1-alpha에서 도입되었습니다.
## Adding data to the front of an evbuffer  
### Interface

```c
int evbuffer_prepend(struct evbuffer *buf, const void *data, size_t size);
int evbuffer_prepend_buffer(struct evbuffer *dst, struct evbuffer* src);
```

각각 `evbuffer_add()`/`evbuffer_add_buffer()`와 같지만, **대상 버퍼의 앞부분**에 데이터를 배치합니다.

주의해서 사용해야 하며, bufferevent와 공유되는 evbuffer에는 사용하지 마세요. 2.0.1-alpha에서 도입되었습니다.
## Rearranging the internal layout of an evbuffer

가끔 evbuffer의 앞쪽 N바이트를 **연속적인 바이트 배열**로 들여다보고 싶을 수 있습니다. 그러려면 먼저 앞부분이 실제로 연속적이어야 합니다.
### Interface

```c
unsigned char *evbuffer_pullup(struct evbuffer *buf, ev_ssize_t size);
```

`evbuffer_pullup()`은 `buf`의 처음 `size` 바이트를 “선형화(linearize)”합니다. 즉, 모두 연속적이고 같은 메모리 청크를 차지하도록 필요 시 복사/이동합니다. `size`가 음수면 전체 버퍼를 선형화합니다. `size`가 버퍼 길이보다 크면 NULL을 반환합니다. 그 외에는 `buf`의 첫 바이트 포인터를 반환합니다.

큰 `size`로 호출하면 버퍼 전체를 복사해야 할 수도 있어 **매우 느릴 수 있습니다**.
### Example

```c
#include <event2/buffer.h>
#include <event2/util.h>

#include <string.h>

int parse_socks4(struct evbuffer *buf, ev_uint16_t *port, ev_uint32_t *addr)
{
    /* SOCKS4 요청의 시작부를 파싱해 보자!
     * 포맷: 버전 1바이트, 명령 1바이트, 목적지 포트 2바이트, 목적지 IP 4바이트 */
    unsigned char *mem;

    mem = evbuffer_pullup(buf, 8);

    if (mem == NULL) {
        /* 버퍼에 데이터가 부족 */
        return 0;
    } else if (mem[0] != 4 || mem[1] != 1) {
        /* 알 수 없는 프로토콜/명령 */
        return -1;
    } else {
        memcpy(port, mem+2, 2);
        memcpy(addr, mem+4, 4);
        *port = ntohs(*port);
        *addr = ntohl(*addr);
        /* 이제 마음에 든다는 걸 확인했으니 데이터를 실제로 제거 */
        evbuffer_drain(buf, 8);
        return 1;
    }
}
```

Note

`evbuffer_get_contiguous_space()`가 돌려준 값과 **같은** `size`로 `evbuffer_pullup()`을 호출하면 데이터 복사/이동이 발생하지 않습니다.

`evbuffer_pullup()`은 2.0.1-alpha에서 새로 도입되었습니다. 그 이전의 Libevent는 비용과 상관없이 항상 evbuffer 데이터를 연속적으로 유지했습니다.
## Removing data from an evbuffer  
### Interface

```c
int evbuffer_drain(struct evbuffer *buf, size_t len);
int evbuffer_remove(struct evbuffer *buf, void *data, size_t datlen);
```

`evbuffer_remove()`는 `buf`의 **앞**에서 `datlen` 바이트를 복사하여 `data`에 넣고, 그만큼 제거합니다. 가용 바이트가 더 적으면 있는 만큼만 복사합니다. 실패 시 -1, 그 외에는 복사한 바이트 수를 반환합니다.

`evbuffer_drain()`은 데이터를 **복사하지 않고** 앞에서 제거만 한다는 점을 제외하면 동일합니다. 성공 시 0, 실패 시 -1.

`evbuffer_drain()`은 0.8, `evbuffer_remove()`는 0.9에 도입되었습니다.
## Copying data out from an evbuffer

버퍼의 **앞부분** 데이터를 비우지 않고 사본만 얻고 싶을 때가 있습니다. 예컨대 어떤 레코드가 완전히 도착했는지 보되(= 들여다보되), `evbuffer_remove()`처럼 데이터를 빼거나, `evbuffer_pullup()`처럼 내부를 재배치하고 싶진 않을 때입니다.
### Interface

```c
ev_ssize_t evbuffer_copyout(struct evbuffer *buf, void *data, size_t datlen);
ev_ssize_t evbuffer_copyout_from(struct evbuffer *buf,
     const struct evbuffer_ptr *pos,
     void *data_out, size_t datlen);
```

`evbuffer_copyout()`은 동작은 `evbuffer_remove()`와 같지만 데이터를 **비우지 않습니다**. 즉 앞에서 `datlen` 바이트를 `data`에 복사합니다. 가용 바이트가 더 적으면 있는 만큼만 복사합니다. 실패 시 -1, 그 외에는 복사한 바이트 수를 반환합니다.

`evbuffer_copyout_from()`은 복사 시작점을 버퍼의 앞이 아니라 `pos`(아래 “Searching within an evbuffer” 참고)로 지정합니다.

복사가 느리면 대신 `evbuffer_peek()`을 사용하세요.
### Example

```c
#include <event2/buffer.h>
#include <event2/util.h>
#include <stdlib.h>
#include <stdlib.h>

int get_record(struct evbuffer *buf, size_t *size_out, char **record_out)
{
    /* 네트워크 바이트 순서의 4바이트 길이 필드 + 그 길이만큼의 데이터로
       구성된 레코드를 처리한다고 가정.
       레코드가 모두 도착했으면 1을 반환하고 out 파라미터를 채움.
       아직이면 0, 오류면 -1. */
    size_t buffer_len = evbuffer_get_length(buf);
    ev_uint32_t record_len;
    char *record;

    if (buffer_len < 4)
       return 0; /* 길이 필드가 아직 안 옴 */

   /* 길이 필드가 버퍼에 남아 있도록 copyout 사용 */
    evbuffer_copyout(buf, &record_len, 4);
    /* 호스트 바이트 순서로 변환 */
    record_len = ntohl(record_len);
    if (buffer_len < record_len + 4)
        return 0; /* 레코드가 아직 다 안 옴 */

    /* 이제 레코드를 제거해도 됨 */
    record = malloc(record_len);
    if (record == NULL)
        return -1;

    evbuffer_drain(buf, 4);
    evbuffer_remove(buf, record, record_len);

    *record_out = record;
    *size_out = record_len;
    return 1;
}
```

`evbuffer_copyout()`은 2.0.5-alpha, `evbuffer_copyout_from()`은 2.1.1-alpha에 추가되었습니다.
## Line-oriented input  

### Interface

```c
enum evbuffer_eol_style {
        EVBUFFER_EOL_ANY,
        EVBUFFER_EOL_CRLF,
        EVBUFFER_EOL_CRLF_STRICT,
        EVBUFFER_EOL_LF,
        EVBUFFER_EOL_NUL
};
char *evbuffer_readln(struct evbuffer *buffer, size_t *n_read_out,
    enum evbuffer_eol_style eol_style);
```

많은 인터넷 프로토콜은 **라인 기반** 형식을 사용합니다. `evbuffer_readln()`은 evbuffer의 앞에서 한 줄을 꺼내 새로 할당한 NUL-종단 문자열로 반환합니다. `n_read_out`이 NULL이 아니면, 반환된 문자열의 바이트 수를 거기에 저장합니다. 완전한 한 줄이 없다면 NULL을 반환합니다. **줄 구분자는 복사된 문자열에 포함되지 않습니다.**

`evbuffer_readln()`이 이해하는 줄 구분 형식은 4가지입니다:

EVBUFFER_EOL_LF  
한 개의 라인피드(`\n`, 0x0A)가 줄의 끝.

EVBUFFER_EOL_CRLF_STRICT  
캐리지 리턴 + 라인피드(`\r\n`, 0x0D 0x0A)가 줄의 끝.

EVBUFFER_EOL_CRLF  
선택적 캐리지 리턴 + 라인피드. 즉 `\r\n` 또는 `\n`. 표준은 보통 `\r\n`을 규정하지만 불완전 구현이 `\n`만 보내는 경우가 있어 유용합니다.

EVBUFFER_EOL_ANY  
임의 개수의 `\r`/`\n` 시퀀스면 무엇이든 줄의 끝로 간주. 주로 하위 호환을 위해 존재.

EVBUFFER_EOL_NUL  
단일 바이트 값 0(ASCII NUL)이 줄의 끝.

(참고: `event_set_mem_functions()`로 기본 malloc을 바꿨다면, `evbuffer_readln`이 반환하는 문자열은 지정한 malloc 대체 함수를 통해 할당됩니다.)
### Example

```c
char *request_line;
size_t len;

request_line = evbuffer_readln(buf, &len, EVBUFFER_EOL_CRLF);
if (!request_line) {
    /* 첫 번째 줄이 아직 안 옴 */
} else {
    if (!strncmp(request_line, "HTTP/1.0 ", 9)) {
        /* HTTP 1.0 감지 ... */
    }
    free(request_line);
}
```

`evbuffer_readln()`은 Libevent 1.4.14-stable 이상에서 제공됩니다. `EVBUFFER_EOL_NUL`은 2.1.1-alpha에 추가되었습니다.
## Searching within an evbuffer

`evbuffer_ptr`는 evbuffer 내의 한 위치를 가리키며, evbuffer를 순회(iterate)하는 데 쓸 수 있는 정보를 담습니다.
### Interface

```c
struct evbuffer_ptr {
        ev_ssize_t pos;
        struct {
                /* internal fields */
        } _internal;
};
```

공개 필드는 `pos`뿐입니다. 나머지는 사용하지 마세요. `pos`는 버퍼 시작으로부터의 오프셋입니다.
### Interface

```c
struct evbuffer_ptr evbuffer_search(struct evbuffer *buffer,
    const char *what, size_t len, const struct evbuffer_ptr *start);
struct evbuffer_ptr evbuffer_search_range(struct evbuffer *buffer,
    const char *what, size_t len, const struct evbuffer_ptr *start,
    const struct evbuffer_ptr *end);
struct evbuffer_ptr evbuffer_search_eol(struct evbuffer *buffer,
    struct evbuffer_ptr *start, size_t *eol_len_out,
    enum evbuffer_eol_style eol_style);
```

`evbuffer_search()`는 길이 `len`의 문자열 `what`을 버퍼에서 검색합니다. 발견하면 그 위치를 담은 `evbuffer_ptr`을, 없으면 `pos == -1`을 반환합니다. `start`가 주어지면 그 위치부터 검색하고, 없으면 버퍼의 처음부터 검색합니다.

`evbuffer_search_range()`는 검색을 `end` 이전으로 제한합니다.

`evbuffer_search_eol()`은 `evbuffer_readln()`과 같은 방식으로 줄 끝을 찾되, 복사 대신 줄 끝 문자(들)의 **시작**을 가리키는 `evbuffer_ptr`을 돌려줍니다. `eol_len_out`이 NULL이 아니면 줄 끝 시퀀스의 길이를 거기에 저장합니다.
### Interface

```c
enum evbuffer_ptr_how {
        EVBUFFER_PTR_SET,
        EVBUFFER_PTR_ADD
};
int evbuffer_ptr_set(struct evbuffer *buffer, struct evbuffer_ptr *pos,
    size_t position, enum evbuffer_ptr_how how);
```

`evbuffer_ptr_set()`은 `pos`를 조작합니다. `how`가 `EVBUFFER_PTR_SET`이면 절대 위치 `position`으로, `EVBUFFER_PTR_ADD`이면 앞으로 `position` 바이트 이동합니다. 성공 시 0, 실패 시 -1.
### Example

```c
#include <event2/buffer.h>
#include <string.h>

/* 'buf'에서 'str'이 나타나는 총 횟수를 센다. */
int count_instances(struct evbuffer *buf, const char *str)
{
    size_t len = strlen(str);
    int total = 0;
    struct evbuffer_ptr p;

    if (!len)
        /* 길이 0 문자열의 출현 횟수는 세지 않는다. */
        return -1;

    evbuffer_ptr_set(buf, &p, 0, EVBUFFER_PTR_SET);

    while (1) {
         p = evbuffer_search(buf, str, len, &p);
         if (p.pos < 0)
             break;
         total++;
         evbuffer_ptr_set(buf, &p, 1, EVBUFFER_PTR_ADD);
    }

    return total;
}
```

## WARNING

evbuffer를 **수정**하거나 레이아웃을 바꾸는 모든 호출은, 존재하는 모든 `evbuffer_ptr` 값을 무효화합니다.
이 인터페이스들은 Libevent 2.0.1-alpha에서 도입되었습니다.
## Inspecting data without copying it

복사(`evbuffer_copyout()`)도 하지 않고, 내부 메모리 재배치(`evbuffer_pullup()`)도 하지 않으면서 evbuffer의 데이터를 읽고 싶을 때가 있습니다. 혹은 중간 지점의 데이터를 보고 싶을 때도요.

다음으로 가능합니다:
### Interface

```c
struct evbuffer_iovec {
        void *iov_base;
        size_t iov_len;
};

int evbuffer_peek(struct evbuffer *buffer, ev_ssize_t len,
    struct evbuffer_ptr *start_at,
    struct evbuffer_iovec *vec_out, int n_vec);
```

`evbuffer_peek()`에 `evbuffer_iovec` 배열(`vec_out`, 길이 `n_vec`)을 주면, evbuffer 내부 RAM의 청크들에 대한 포인터(`iov_base`)와 그 길이(`iov_len`)를 채워줍니다.

`len < 0`이면 가능한 한 많은 `evbuffer_iovec`을 채우고, 그렇지 않으면 `len` 바이트 이상을 가시화할 때까지(또는 배열이 다 찰 때까지) 채웁니다. 요청한 데이터를 전부 제공할 수 있으면 실제 사용한 구조체 개수를 반환하고, 그렇지 않으면 **필요한 개수**를 반환합니다.

`start_at`가 NULL이면 버퍼의 시작에서, 아니면 지정한 위치에서 시작합니다.
### Examples

```c
{
    /* buf의 처음 두 청크를 보고 stderr로 쓴다. */
    int n, i;
    struct evbuffer_iovec v[2];
    n = evbuffer_peek(buf, -1, NULL, v, 2);
    for (i=0; i<n; ++i) { /* 가용 청크가 2개 미만일 수 있다. */
        fwrite(v[i].iov_base, 1, v[i].iov_len, stderr);
    }
}
```

```c
{
    /* 처음 4096바이트를 write로 stdout에 보낸다. */
    int n, i, r;
    struct evbuffer_iovec *v;
    size_t written = 0;

    /* 필요한 청크 수를 구한다. */
    n = evbuffer_peek(buf, 4096, NULL, NULL, 0);
    /* 청크 배열을 할당 (alloca를 쓸 수도 있음) */
    v = malloc(sizeof(struct evbuffer_iovec)*n);
    /* 실제로 채움 */
    n = evbuffer_peek(buf, 4096, NULL, v, n);
    for (i=0; i<n; ++i) {
        size_t len = v[i].iov_len;
        if (written + len > 4096)
            len = 4096 - written;
        r = write(1 /* stdout */, v[i].iov_base, len);
        if (r<=0)
            break;
        /* 마지막 청크에서 한도를 넘겨 더 쓰는 일을 막기 위해
           별도로 누적 바이트를 추적한다. */
        written += len;
    }
    free(v);
}
```

```c
{
    /* "start\n"의 첫 출현 뒤의 16KiB 데이터를 얻어 consume()에 전달 */
    struct evbuffer_ptr ptr;
    struct evbuffer_iovec v[1];
    const char s[] = "start\n";
    int n_written;

    ptr = evbuffer_search(buf, s, strlen(s), NULL);
    if (ptr.pos == -1)
        return; /* 시작 문자열 없음 */

    /* 시작 문자열을 건너뛴다. */
    if (evbuffer_ptr_set(buf, &ptr, strlen(s), EVBUFFER_PTR_ADD) < 0)
        return; /* 버퍼 끝을 넘어감 */

    while (n_written < 16*1024) {
        /* 한 청크를 들여다봄 */
        if (evbuffer_peek(buf, -1, &ptr, v, 1) < 1)
            break;
        /* 사용자 정의 consume()에 전달 */
        consume(v[0].iov_base, v[0].iov_len);
        n_written += v[0].iov_len;

        /* 다음 반복에서 다음 청크를 보도록 포인터 이동 */
        if (evbuffer_ptr_set(buf, &ptr, v[0].iov_len, EVBUFFER_PTR_ADD)<0)
            break;
    }
}
```

Notes

- `evbuffer_iovec`가 가리키는 데이터를 수정하면 **정의되지 않은 동작**입니다.
- evbuffer를 수정하는 어떤 함수라도 `evbuffer_peek()`이 돌려준 포인터들을 무효화할 수 있습니다.
- 여러 스레드에서 사용할 수 있는 evbuffer라면, `evbuffer_peek()` 호출 전 `evbuffer_lock()`, 사용이 끝난 뒤 `evbuffer_unlock()`으로 보호하세요.

이 함수는 Libevent 2.0.2-alpha에서 새로 도입되었습니다.
## Adding data to an evbuffer directly

가끔은 먼저 임시 버퍼에 쓰고 `evbuffer_add()`로 복사하지 않고, **바로** evbuffer 내부로 쓰고 싶을 수 있습니다. 이를 위한 고급 함수 쌍이 `evbuffer_reserve_space()`와 `evbuffer_commit_space()`입니다. `evbuffer_peek()`과 마찬가지로 내부 메모리에 직접 접근하기 위해 `evbuffer_iovec`을 사용합니다.
### Interface

```c
int evbuffer_reserve_space(struct evbuffer *buf, ev_ssize_t size,
    struct evbuffer_iovec *vec, int n_vecs);
int evbuffer_commit_space(struct evbuffer *buf,
    struct evbuffer_iovec *vec, int n_vecs);
```

`evbuffer_reserve_space()`는 evbuffer 내부 공간의 포인터를 제공합니다. 최소 `size` 바이트를 줄 수 있도록 필요 시 버퍼를 확장합니다. 이 포인터들과 길이를 `vec` 배열(길이 `n_vecs`)에 채워줍니다.

`n_vecs`는 최소 1이어야 합니다. 단 하나만 제공하면 요청한 바이트를 **하나의 연속 영역**으로 보장하지만, 그 때문에 내부 재배치/메모리 낭비가 생길 수 있습니다. 성능을 위해서는 최소 2개 이상을 제공하세요. 반환값은 실제로 사용한 벡터 개수입니다.

이 벡터들에 쓴 데이터는 `evbuffer_commit_space()`를 호출해야 비로소 버퍼의 일부가 됩니다. 요청했던 것보다 적은 공간만 커밋하려면 각 `evbuffer_iovec`의 `iov_len`을 줄이거나, 전달하는 벡터 수를 줄이면 됩니다. 성공 시 0, 실패 시 -1을 반환합니다.
## Notes and Caveats

- evbuffer를 재배치하거나 데이터를 추가하는 **모든 함수**는 `evbuffer_reserve_space()`로 받은 포인터를 무효화합니다.
- 현재 구현은 사용자가 몇 개를 주든 **최대 2개**만 사용합니다(향후 바뀔 수 있음).
- `evbuffer_reserve_space()`는 여러 번 호출해도 안전합니다.
- 멀티스레드에서 쓸 수 있는 evbuffer라면 `reserve` 전 `evbuffer_lock()`, 커밋 후 `evbuffer_unlock()`을 호출하세요.
### Example

```c
/* generate_data()의 2048바이트 출력을 복사 없이 버퍼에 채우기 */
struct evbuffer_iovec v[2];
int n, i;
size_t n_to_add = 2048;

/* 2048바이트 예약 */
n = evbuffer_reserve_space(buf, n_to_add, v, 2);
if (n<=0)
   return; /* 예약 실패 */

for (i=0; i<n && n_to_add > 0; ++i) {
   size_t len = v[i].iov_len;
   if (len > n_to_add) /* 남은 양을 넘기지 않기 */
      len = n_to_add;
   if (generate_data(v[i].iov_base, len) < 0) {
      /* 생성 중 문제면 여기서 중단: 커밋하지 않으면 데이터는 추가되지 않음 */
      return;
   }
   /* 실제로 쓴 바이트 수만 커밋되게 iov_len 조정 */
   v[i].iov_len = len;
}

/* 사용한 벡터 수(i)만큼 커밋 (사용 가능 수 n이 아님) */
if (evbuffer_commit_space(buf, v, i) < 0)
   return; /* 커밋 실패 */
```

### Bad Examples

```c
/* evbuffer_reserve() 사용 시 흔한 실수 — 따라하지 마세요. */
struct evbuffer_iovec v[2];

{
  /* 버퍼를 수정하는 함수 호출 후 reserve 포인터를 사용하지 말 것 */
  evbuffer_reserve_space(buf, 1024, v, 2);
  evbuffer_add(buf, "X", 1);
  /* 잘못: 위 evbuffer_add가 내부 레이아웃을 바꿨다면 다음 줄은 실패/크래시 가능.
     항상 데이터를 추가한 뒤 reserve를 하세요. */
  memset(v[0].iov_base, 'Y', v[0].iov_len-1);
  evbuffer_commit_space(buf, v, 1);
}

{
  /* iov_base 포인터를 수정하지 말 것 */
  const char *data = "Here is some data";
  evbuffer_reserve_space(buf, strlen(data), v, 1);
  /* 잘못: data의 내용을 v[0].iov_base로 '복사'해야 합니다. 포인터만 바꾸지 마세요. */
  v[0].iov_base = (char*) data;
  v[0].iov_len = strlen(data);
  /* 운이 좋으면 commit에서 에러가 납니다. */
  evbuffer_commit_space(buf, v, 1);
}
```

이 함수들의 현재 인터페이스는 Libevent 2.0.2-alpha부터입니다.
## Network IO with evbuffers

evbuffer의 가장 흔한 용도는 네트워크 IO입니다. evbuffer로 네트워크 IO를 수행하는 인터페이스:
### Interface

```c
int evbuffer_write(struct evbuffer *buffer, evutil_socket_t fd);
int evbuffer_write_atmost(struct evbuffer *buffer, evutil_socket_t fd,
        ev_ssize_t howmuch);
int evbuffer_read(struct evbuffer *buffer, evutil_socket_t fd, int howmuch);
```

`evbuffer_read()`는 소켓 `fd`에서 최대 `howmuch` 바이트를 읽어 `buffer` **끝**에 붙입니다. 성공 시 읽은 바이트 수, EOF면 0, 오류면 -1을 반환합니다. 이때 오류가 **진짜** 오류인지, 비블로킹에서 지금은 불가(EAGAIN/Windows의 WSAEWOULDBLOCK)인지를 구분해야 합니다. `howmuch`가 음수면, 적절한 크기를 **추정**해서 읽습니다.

`evbuffer_write_atmost()`는 `buffer`의 **앞**에서 최대 `howmuch` 바이트를 소켓 `fd`로 씁니다. 성공 시 쓴 바이트 수, 실패 시 -1. 읽기와 마찬가지로, 지금은 불가인 비블로킹 조건인지 실제 오류인지 확인해야 합니다. `howmuch`가 음수면 버퍼의 모든 내용을 쓰려 시도합니다.

`evbuffer_write()`는 `howmuch < 0`인 `evbuffer_write_atmost()`와 같습니다: 가능한 한 많이 플러시합니다.

유닉스에선 읽기/쓰기를 지원하는 파일 디스크립터라면 무엇이든 동작합니다. Windows에선 소켓만 지원됩니다.

**bufferevent**를 사용 중이면 이 IO 함수들을 직접 호출할 필요가 없습니다. bufferevent 코드가 처리합니다.

`evbuffer_write_atmost()`는 2.0.1-alpha에서 도입되었습니다.
## Evbuffers and callbacks

evbuffer 사용자들은 데이터가 추가되거나 제거될 때 이를 알고 싶어 합니다. 이를 위해 Libevent는 일반적인 evbuffer 콜백 메커니즘을 제공합니다.
### Interface

```c
struct evbuffer_cb_info {
        size_t orig_size;
        size_t n_added;
        size_t n_deleted;
};

typedef void (*evbuffer_cb_func)(struct evbuffer *buffer,
    const struct evbuffer_cb_info *info, void *arg);
```

evbuffer가 변경될 때마다 콜백이 호출됩니다. 콜백은 버퍼, `evbuffer_cb_info` 포인터, 사용자 인자(arg)를 받습니다. `orig_size`는 변경 전 길이, `n_added`는 추가된 바이트 수, `n_deleted`는 제거된 바이트 수입니다.
### Interface

```c
struct evbuffer_cb_entry;
struct evbuffer_cb_entry *evbuffer_add_cb(struct evbuffer *buffer,
    evbuffer_cb_func cb, void *cbarg);
```

`evbuffer_add_cb()`는 evbuffer에 콜백을 추가하고, 나중에 이 콜백을 가리키는 불투명 포인터를 반환합니다. `cb`는 호출될 함수, `cbarg`는 콜백에 전달할 사용자 포인터입니다.

하나의 evbuffer에 여러 콜백을 둘 수 있습니다. 새 콜백을 추가해도 기존 콜백은 제거되지 않습니다.
### Example

```c
#include <event2/buffer.h>
#include <stdio.h>
#include <stdlib.h>

/* 버퍼에서 총 제거한 바이트 수를 기억하고,
   1MB가 지날 때마다 점을 출력하는 콜백 */
struct total_processed {
    size_t n;
};
void count_megabytes_cb(struct evbuffer *buffer,
    const struct evbuffer_cb_info *info, void *arg)
{
    struct total_processed *tp = arg;
    size_t old_n = tp->n;
    int megabytes, i;
    tp->n += info->n_deleted;
    megabytes = ((tp->n) >> 20) - (old_n >> 20);
    for (i=0; i<megabytes; ++i)
        putc('.', stdout);
}

void operation_with_counted_bytes(void)
{
    struct total_processed *tp = malloc(sizeof(*tp));
    struct evbuffer *buf = evbuffer_new();
    tp->n = 0;
    evbuffer_add_cb(buf, count_megabytes_cb, tp);

    /* evbuffer 사용 … 끝나면: */
    evbuffer_free(buf);
    free(tp);
}
```

참고: **비어있지 않은** evbuffer를 free해도 “드레인”으로 계산되지 않습니다. 또한 evbuffer를 free해도 콜백의 사용자 포인터를 free해주지 않습니다.

콜백을 영구적으로 활성화하고 싶지 않다면, 제거(완전히 삭제)하거나 비활성화(잠시 꺼두기)할 수 있습니다:
### Interface

```c
int evbuffer_remove_cb_entry(struct evbuffer *buffer,
    struct evbuffer_cb_entry *ent);
int evbuffer_remove_cb(struct evbuffer *buffer, evbuffer_cb_func cb,
    void *cbarg);

#define EVBUFFER_CB_ENABLED 1
int evbuffer_cb_set_flags(struct evbuffer *buffer,
                          struct evbuffer_cb_entry *cb,
                          ev_uint32_t flags);
int evbuffer_cb_clear_flags(struct evbuffer *buffer,
                          struct evbuffer_cb_entry *cb,
                          ev_uint32_t flags);
```

콜백은 추가 시 받은 `evbuffer_cb_entry`로 제거할 수도 있고, `(cb, cbarg)` 쌍으로 제거할 수도 있습니다. 성공 시 0, 실패 시 -1.

`evbuffer_cb_set_flags()`/`evbuffer_cb_clear_flags()`는 콜백의 플래그를 설정/해제합니다. 현재 사용자에게 보이는 플래그는 `EVBUFFER_CB_ENABLED` 하나뿐입니다(기본 활성). 해제하면 evbuffer 변경 시 이 콜백은 호출되지 않습니다.
### Interface

```c
int evbuffer_defer_callbacks(struct evbuffer *buffer, struct event_base *base);
```

bufferevent 콜백과 마찬가지로, evbuffer 콜백을 즉시 실행하지 않고 지정한 event base의 이벤트 루프 일부로 **지연 실행**할 수 있습니다. 여러 evbuffer의 콜백이 상호 간에 데이터를 추가/제거하는 상황에서 스택 폭주를 피하는 데 유용합니다.

evbuffer의 콜백이 지연되면, 실제 호출 시 여러 연산 결과를 **요약**해 전달할 수도 있습니다.

bufferevent처럼 evbuffer도 내부적으로 참조 카운팅을 사용하므로, 지연된 콜백이 아직 실행되지 않았더라도 evbuffer를 안전하게 free할 수 있습니다.

이 콜백 시스템 전체는 2.0.1-alpha에서 도입되었습니다. `evbuffer_cb_(set|clear)_flags()`는 2.0.2-alpha부터 현재 인터페이스로 제공됩니다.
## Avoiding data copies with evbuffer-based IO

**복사를 최소화**하는 것은 고성능 네트워크 프로그래밍의 핵심입니다. Libevent는 이를 돕는 메커니즘을 제공합니다.
### Interface

```c
typedef void (*evbuffer_ref_cleanup_cb)(const void *data,
    size_t datalen, void *extra);

int evbuffer_add_reference(struct evbuffer *outbuf,
    const void *data, size_t datlen,
    evbuffer_ref_cleanup_cb cleanupfn, void *extra);
```

이 함수는 데이터를 **복사하지 않고** 포인터 **참조**로 evbuffer 끝에 추가합니다. 따라서 포인터는 evbuffer가 그 데이터를 쓰는 동안 **유효**해야 합니다. 더 이상 필요 없게 되면 `cleanupfn(data, datlen, extra)`를 호출합니다. 성공 0, 실패 -1.
### Example

```c
#include <event2/buffer.h>
#include <stdlib.h>
#include <string.h>

/* 1MiB 리소스를 여러 evbuffer로 네트워크에 스풀(spool)하고자 할 때,
   메모리에 불필요한 복사본을 남기지 않고 처리하는 예 */
#define HUGE_RESOURCE_SIZE (1024*1024)
struct huge_resource {
    /* 이 구조체에 대한 참조 수를 추적하여
       언제 free할 수 있는지 알기 위함 */
    int reference_count;
    char data[HUGE_RESOURCE_SIZE];
};

struct huge_resource *new_resource(void) {
    struct huge_resource *hr = malloc(sizeof(struct huge_resource));
    hr->reference_count = 1;
    /* 실제 환경에선 파일을 읽거나 복잡한 계산을 하겠지만,
       여기선 그냥 0xEE로 채운다 */
    memset(hr->data, 0xEE, sizeof(hr->data));
    return hr;
}

void free_resource(struct huge_resource *hr) {
    --hr->reference_count;
    if (hr->reference_count == 0)
        free(hr);
}

static void cleanup(const void *data, size_t len, void *arg) {
    free_resource(arg);
}

/* 리소스를 실제로 버퍼에 추가하는 함수 */
void spool_resource_to_evbuffer(struct evbuffer *buf,
    struct huge_resource *hr)
{
    ++hr->reference_count;
    evbuffer_add_reference(buf, hr->data, HUGE_RESOURCE_SIZE,
        cleanup, hr);
}
```

`evbuffer_add_reference()`의 현재 인터페이스는 2.0.2-alpha부터입니다.
## Adding a file to an evbuffer

일부 OS는 데이터를 **유저 공간으로 복사하지 않고** 파일을 네트워크로 쓰는 기능을 제공합니다. 가능하다면 다음 간단한 인터페이스로 접근할 수 있습니다.
### Interface

```c
int evbuffer_add_file(struct evbuffer *output, int fd, ev_off_t offset,
    size_t length);
```

읽기 가능한 **파일 디스크립터**(소켓이 아님!) `fd`가 있다고 가정합니다. 파일의 `offset`부터 `length` 바이트를 `output`의 끝에 추가합니다. 성공 0, 실패 -1.
WARNING

Libevent 2.0.x에서 이렇게 추가한 데이터에 대해 **신뢰할 수 있는 동작**은 `evbuffer_write*()`로 네트워크에 보내거나, `evbuffer_drain()`으로 비우거나, `evbuffer_*_buffer()`로 다른 evbuffer로 옮기는 것뿐이었습니다. `evbuffer_remove()`로 꺼내거나, `evbuffer_pullup()`으로 선형화하는 등은 신뢰하기 어려웠습니다. Libevent 2.1.x는 이 제한을 개선하려고 합니다.

OS가 `splice()`나 `sendfile()`을 지원하면, `evbuffer_write()` 호출 시 유저 RAM으로 복사하지 않고 파일에서 네트워크로 **직접 전송**합니다. 이들이 없지만 `mmap()`이 있으면 커널이 유저 공간 복사 없이 전송할 수 있음을 파악할 수 있습니다. 둘 다 없으면 디스크에서 RAM으로 읽습니다.

파일 디스크립터는 evbuffer에서 데이터가 플러시되거나 evbuffer가 해제될 때 **닫힙니다**. 그게 원치 않거나 더 세밀한 제어가 필요하면 아래의 **file_segment** 기능을 보세요.

이 함수는 2.0.1-alpha에서 도입되었습니다.
## Fine-grained control with file segments

`evbuffer_add_file()`은 파일의 소유권을 가져가기 때문에, **같은 파일을 여러 번** 추가하기에는 비효율적입니다.
### Interface

```c
struct evbuffer_file_segment;

struct evbuffer_file_segment *evbuffer_file_segment_new(
        int fd, ev_off_t offset, ev_off_t length, unsigned flags);
void evbuffer_file_segment_free(struct evbuffer_file_segment *seg);
int evbuffer_add_file_segment(struct evbuffer *buf,
    struct evbuffer_file_segment *seg, ev_off_t offset, ev_off_t length);
```

`evbuffer_file_segment_new()`는 파일 `fd`의 `offset`에서 시작해 `length` 바이트를 나타내는 새 `evbuffer_file_segment`를 생성/반환합니다. 오류 시 NULL.

파일 세그먼트는 `sendfile`, `splice`, `mmap`, `CreateFileMapping`, 또는 `malloc()+read()`로 구현됩니다. 가장 가벼운 메커니즘으로 생성되며, 필요해질 때 더 무거운 방식으로 **전환**됩니다(예: OS가 `sendfile`과 `mmap` 모두 지원하면, 내용 검사가 필요해질 때까지 `sendfile`로만 두고, 그때 `mmap()`으로 전환).

세부 동작은 다음 플래그로 제어합니다:

- **EVBUF_FS_CLOSE_ON_FREE**  
  세그먼트를 `evbuffer_file_segment_free()`로 해제할 때, **기저 파일을 닫습니다**.

- **EVBUF_FS_DISABLE_MMAP**  
  적합하더라도 **매핑 메모리(mmap/CreateFileMapping)** 백엔드를 사용하지 않습니다.

- **EVBUF_FS_DISABLE_SENDFILE**  
  적합하더라도 **sendfile(splice)** 백엔드를 사용하지 않습니다.

- **EVBUF_FS_DISABLE_LOCKING**  
  잠금을 할당하지 않습니다. 여러 스레드에서 볼 가능성이 있는 사용에는 **안전하지 않습니다**.

세그먼트를 만들었으면, `evbuffer_add_file_segment()`로 그 일부/전체를 evbuffer에 추가할 수 있습니다. 여기서의 `offset`은 **세그먼트 내부** 오프셋입니다(파일 전체가 아님).

더 이상 세그먼트를 사용하지 않으면 `evbuffer_file_segment_free()`로 해제하세요. 실제 저장소는 어떤 evbuffer도 이 세그먼트 조각을 참조하지 않을 때까지 해제되지 않습니다.
### Interface

```c
typedef void (*evbuffer_file_segment_cleanup_cb)(
    struct evbuffer_file_segment const *seg, int flags, void *arg);

void evbuffer_file_segment_add_cleanup_cb(struct evbuffer_file_segment *seg,
        evbuffer_file_segment_cleanup_cb cb, void *arg);
```

세그먼트에 최종 참조가 해제되어 곧 free되기 직전에 호출될 콜백을 등록할 수 있습니다. 이 콜백에서 세그먼트를 되살리거나 버퍼에 다시 추가하는 등의 동작은 **금지**됩니다.

이들 파일 세그먼트 함수는 2.1.1-alpha에서 처음 등장했으며, `evbuffer_file_segment_add_cleanup_cb()`는 2.1.2-alpha에 추가되었습니다.
## Adding an evbuffer to another by reference

한 evbuffer의 **현재 내용**을 다른 evbuffer에 **참조**로 추가할 수도 있습니다. 즉, 내용을 빼서 복사하지 않고, 복사한 것처럼 동작하게 만듭니다.
### Interface

```c
int evbuffer_add_buffer_reference(struct evbuffer *outbuf,
    struct evbuffer *inbuf);
```

`evbuffer_add_buffer_reference()`는 `inbuf`의 **현재 내용**을 복사한 것처럼 보이게 하지만 불필요한 복사를 수행하지 않습니다. 성공 0, 실패 -1.

이후 `inbuf`의 **내용 변경**은 `outbuf`에 반영되지 않습니다. evbuffer **자체**가 아니라, 호출 시점의 **내용**을 참조로 추가하는 것입니다.

또한 **참조의 중첩**은 불가합니다. 이미 어떤 호출에서 `outbuf`였던 버퍼는 다른 호출에서 `inbuf`가 될 수 없습니다.

이 함수는 2.1.1-alpha에서 도입되었습니다.
## Making an evbuffer add- or remove-only  

### Interface

```c
int evbuffer_freeze(struct evbuffer *buf, int at_front);
int evbuffer_unfreeze(struct evbuffer *buf, int at_front);
```

evbuffer의 **앞(front)** 또는 **뒤(end)**에 대한 변경을 일시적으로 막을 수 있습니다. bufferevent 코드는 내부적으로, 출력 버퍼의 **앞**(= 이미 보낼 데이터)에 대한 실수 수정이나, 입력 버퍼의 **끝**에 대한 실수 수정을 막기 위해 이 함수를 사용합니다.

`evbuffer_freeze()`는 2.0.1-alpha에 도입되었습니다.
## Obsolete evbuffer functions

evbuffer 인터페이스는 2.0에서 크게 바뀌었습니다. 그 이전에는 모든 evbuffer가 **연속 RAM**으로 구현되어 접근이 매우 비효율적이었습니다.

또한 `event.h`는 `struct evbuffer`의 내부를 노출했는데, 1.4와 2.0 사이에 내부가 너무 많이 바뀌어 그런 코드들은 더 이상 동작하지 않습니다.

evbuffer의 바이트 수는 `EVBUFFER_LENGTH()` 매크로로, 실제 데이터는 `EVBUFFER_DATA()`로 접근했습니다. 이들은 `event2/buffer_compat.h`에 있습니다. 다만 `EVBUFFER_DATA(b)`는 `evbuffer_pullup(b, -1)`의 별칭이므로 **매우 비용이 큽니다**.

기타 폐기된 인터페이스:
### Deprecated Interface

```c
char *evbuffer_readline(struct evbuffer *buffer);
unsigned char *evbuffer_find(struct evbuffer *buffer,
    const unsigned char *what, size_t len);
```

`evbuffer_readline()`은 현재의 `evbuffer_readln(buffer, NULL, EVBUFFER_EOL_ANY)`와 유사하게 동작했습니다.

`evbuffer_find()`는 문자열의 **첫 번째** 출현만 찾고 그 포인터를 반환했습니다. `evbuffer_search()`와 달리 하나만 찾을 수 있었죠. 구 버전 호환을 위해 현재는 찾은 문자열 끝까지 버퍼를 **전체 선형화**합니다.

콜백 인터페이스도 달랐습니다:
### Deprecated Interface

```c
typedef void (*evbuffer_cb)(struct evbuffer *buffer,
    size_t old_len, size_t new_len, void *arg);
void evbuffer_setcb(struct evbuffer *buffer, evbuffer_cb cb, void *cbarg);
```

evbuffer는 한 번에 **단 하나의 콜백**만 가질 수 있었습니다. 새 콜백을 설정하면 이전 것이 꺼졌고, NULL을 설정하는 것이 콜백 해제의 표준 방법이었습니다.

현재의 `evbuffer_cb_info` 대신, 콜백에는 **옛 길이/새 길이**만 전달되었습니다. 따라서 `old_len > new_len`이면 드레인, 반대면 추가로 해석했습니다. 콜백 지연 실행도 불가했고, 추가/삭제가 하나의 콜백으로 요약되는 일도 없었습니다.

여기의 폐기된 함수들은 `event2/buffer_compat.h`에서 여전히 사용할 수 있습니다.