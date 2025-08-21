---
layout: single
title:  "Libevent를 위한 도우미 함수와 타입"
date: 2025-08-21
tags: libevent
categories: 
   - Libevent
---

# Libevent를 위한 도우미 함수와 타입

`<event2/util.h>` 헤더는 Libevent를 사용해 **이식성 있는 애플리케이션**을 구현할 때 유용할 수 있는 많은 함수를 정의한다. Libevent 자체도 내부적으로 이 타입들과 함수들을 사용한다.

## Basic types

### evutil_socket_t

Windows를 제외한 대부분의 환경에서 소켓은 `int`이며, 운영체제는 이를 숫자 순서로 할당한다. 그러나 Windows 소켓 API에서는 소켓이 `SOCKET` 타입(실제로는 **포인터 같은 OS 핸들**)이고, 이를 받는 순서는 정의되어 있지 않다. 우리는 Windows에서 포인터 축소 위험 없이 `socket()` 또는 `accept()`의 결과를 담을 수 있는 정수형인 `evutil_socket_t` 타입을 정의한다.

#### Definition

```c
#ifdef WIN32
#define evutil_socket_t intptr_t
#else
#define evutil_socket_t int
#endif
```

이 타입은 **Libevent 2.0.1-alpha**에서 도입되었다.

### Standard integer types

종종 C99의 표준 헤더인 `stdint.h`를 구현하지 않은, 21세기의 혜택을 받지 못한 C 시스템을 만나게 된다. 이런 상황을 위해 Libevent는 `stdint.h`의 **비트 폭 지정 정수**들을 자체 버전으로 정의한다:

| Type        | Width | Signed | Maximum       | Minimum      |
| ----------- | ----- | ------ | ------------- | ------------ |
| ev_uint64_t | 64    | No     | EV_UINT64_MAX | 0            |
| ev_int64_t  | 64    | Yes    | EV_INT64_MAX  | EV_INT64_MIN |
| ev_uint32_t | 32    | No     | EV_UINT32_MAX | 0            |
| ev_int32_t  | 32    | Yes    | EV_INT32_MAX  | EV_INT32_MIN |
| ev_uint16_t | 16    | No     | EV_UINT16_MAX | 0            |
| ev_int16_t  | 16    | Yes    | EV_INT16_MAX  | EV_INT16_MIN |
| ev_uint8_t  | 8     | No     | EV_UINT8_MAX  | 0            |
| ev_int8_t   | 8     | Yes    | EV_INT8_MAX   | EV_INT8_MIN  |

C99 표준과 마찬가지로, 각 타입은 **지정된 비트 폭을 정확히** 가진다.

이 타입들은 **Libevent 1.4.0-beta**에서 도입되었다. `MAX/MIN` 상수는 **Libevent 2.0.4-alpha**에서 처음 등장했다.

### Miscellaneous compatibility types

- `ev_ssize_t` 타입은 해당 플랫폼에 `ssize_t`(부호 있는 `size_t`)가 있으면 그것으로, 없다면 합리적인 기본값으로 정의된다. 가능한 최대/최소값은 각각 `EV_SSIZE_MAX`, `EV_SSIZE_MIN`이다. (플랫폼이 `SIZE_MAX`를 정의하지 않았다면, `size_t`의 가능한 최대값으로 `EV_SIZE_MAX`를 사용할 수 있다.)
- `ev_off_t` 타입은 파일이나 메모리 블록 내의 **오프셋**을 표현한다. 합리적인 `off_t` 정의가 있는 플랫폼에서는 `off_t`로, Windows에서는 `ev_int64_t`로 정의된다.
- 일부 소켓 API 구현은 길이 타입 `socklen_t`를 제공하고, 일부는 제공하지 않는다. `ev_socklen_t`는 존재하는 경우 해당 타입으로, 그렇지 않으면 합리적인 기본값으로 정의된다.
- `ev_intptr_t`/`ev_uintptr_t`는 **포인터를 비트 손실 없이 담을 만큼 충분히 큰** 부호/무부호 정수이다.

도입 시기:

- `ev_ssize_t` — **2.0.2-alpha**
- `ev_socklen_t` — **2.0.3-alpha**
- `ev_intptr_t`, `ev_uintptr_t`, `EV_SSIZE_MAX/MIN` — **2.0.4-alpha**
- `ev_off_t` — **2.0.9-rc**

------

## Timer portability functions

모든 플랫폼이 표준 `timeval` 조작 함수를 정의하지는 않으므로, Libevent는 자체 구현을 제공한다.

### Interface

```c
#define evutil_timeradd(tvp, uvp, vvp) /* ... */
#define evutil_timersub(tvp, uvp, vvp) /* ... */
```

이 매크로들은 각각 **첫 두 인자를 더하거나 빼고**, 결과를 **세 번째 인자**에 저장한다.

### Interface

```c
#define evutil_timerclear(tvp) /* ... */
#define evutil_timerisset(tvp) /* ... */
```

`timeval`을 **지우는 것**은 그 값을 0으로 설정하는 것이다. **설정 여부 확인**은 0이 아니면 `true`, 아니면 `false`를 반환한다.

### Interface

```c
#define evutil_timercmp(tvp, uvp, cmp)
```

`evutil_timercmp` 매크로는 두 `timeval`을 비교하며, 비교 연산자 `cmp`로 지정한 관계가 성립하면 `true`를 낸다. 예: `evutil_timercmp(t1, t2, ⇐)`는 “t1이 t2보다 **⇐** 인가?”를 의미한다. 일부 OS 버전과 달리, Libevent의 `timercmp`는 **모든 C 관계 연산자**(`<, >, ==, !=, ⇐, >=`)를 지원한다.

### Interface

```c
int evutil_gettimeofday(struct timeval *tv, struct timezone *tz);
```

`evutil_gettimeofday`는 `tv`를 **현재 시각**으로 설정한다. `tz` 인자는 **사용되지 않는다**.

### Example

```c
struct timeval tv1, tv2, tv3;

/* tv1 = 5.5초로 설정 */
tv1.tv_sec = 5; tv1.tv_usec = 500*1000;

/* tv2 = 현재 시각 */
evutil_gettimeofday(&tv2, NULL);

/* tv3 = 5.5초 후 */
evutil_timeradd(&tv1, &tv2, &tv3);

/* 세 줄 모두 true를 출력해야 한다 */
if (evutil_timercmp(&tv1, &tv1, ==))  /* == "tv1 == tv1 이면" */
   puts("5.5 sec == 5.5 sec");
if (evutil_timercmp(&tv3, &tv2, >=))  /* == "tv3 >= tv2 이면" */
   puts("The future is after the present.");
if (evutil_timercmp(&tv1, &tv2, <))   /* == "tv1 < tv2 이면" */
   puts("It is no longer the past.");
```

이 함수들은 **Libevent 1.4.0-beta**에서 도입되었고, `evutil_gettimeofday()`는 **Libevent 2.0**에서 도입되었다.

> **Note**
>  Libevent 1.4.4 이전에는 `timercmp`에서 **⇐** 또는 **>=**를 사용하는 것이 안전하지 않았다.

------

## Socket API compatibility

역사적 이유로 Windows는 Berkeley 소켓 API를 **완전 호환**으로 구현하지 않았다. 다음은 마치 그렇게 구현된 것처럼 사용할 수 있게 해주는 몇 가지 함수들이다.

### Interface

```c
int evutil_closesocket(evutil_socket_t s);

#define EVUTIL_CLOSESOCKET(s) evutil_closesocket(s)
```

이 함수는 소켓을 닫는다. **Unix**에서는 `close()`의 별칭이고, **Windows**에서는 `closesocket()`을 호출한다. (Windows에서는 소켓에 `close()`를 사용할 수 없고, 다른 환경에서는 `closesocket()`을 정의하지 않는다.)

`evutil_closesocket`은 **Libevent 2.0.5-alpha**에서 도입. 그 이전에는 `EVUTIL_CLOSESOCKET` 매크로를 사용.

### Interface

```c
#define EVUTIL_SOCKET_ERROR()
#define EVUTIL_SET_SOCKET_ERROR(errcode)
#define evutil_socket_geterror(sock)
#define evutil_socket_error_to_string(errcode)
```

소켓 오류 코드를 **접근/조작**한다.

- `EVUTIL_SOCKET_ERROR()` — 이 스레드의 **마지막 소켓 연산** 전역 오류 코드
- `evutil_socket_geterror(sock)` — **특정 소켓**의 오류 코드
- `EVUTIL_SET_SOCKET_ERROR()` — 현재 소켓 오류 코드 변경 (Unix의 `errno` 설정과 유사)
- `evutil_socket_error_to_string()` — 주어진 소켓 오류의 **문자열 표현**(Unix의 `strerror()` 유사)

> Windows는 소켓 오류에 `errno`가 아닌 `WSAGetLastError()`를 사용한다.
>  또한 Windows의 소켓 오류 코드는 표준 C의 `errno` 값과 **동일하지 않다.**

### Interface

```c
int evutil_make_socket_nonblocking(evutil_socket_t sock);
```

소켓을 **논블로킹**으로 만든다. (Unix: `O_NONBLOCK`, Windows: `FIONBIO` 설정)

### Interface

```c
int evutil_make_listen_socket_reuseable(evutil_socket_t sock);
```

리스너 소켓이 사용한 주소가 **닫힌 직후 다른 소켓에서 재사용 가능**하도록 한다. (Unix: `SO_REUSEADDR`, Windows: 아무 것도 하지 않음 — Windows에서 `SO_REUSEADDR`의 의미가 다름)

### Interface

```c
int evutil_make_socket_closeonexec(evutil_socket_t sock);
```

향후 `exec()` 호출 시 해당 소켓이 **닫히도록** 설정. (Unix: `FD_CLOEXEC`, Windows: 아무 것도 하지 않음)

### Interface

```c
int evutil_socketpair(int family, int type, int protocol,
        evutil_socket_t sv[2]);
```

Unix의 `socketpair()`처럼 **서로 연결된 두 소켓**을 생성해 `sv[0]`, `sv[1]`에 담는다. 성공 시 0, 실패 시 -1.

> Windows에서는 `AF_INET`/`SOCK_STREAM`/`protocol=0`만 지원.
>  일부 Windows 호스트에서는 방화벽이 127.0.0.1을 차단해 실패할 수 있음.

이 함수들은 **Libevent 1.4.0-beta**에서 도입되었고, `evutil_make_socket_closeonexec()`만 **2.0.4-alpha**에서 새로 추가되었다.

------

## Portable string manipulation functions

### Interface

```c
ev_int64_t evutil_strtoll(const char *s, char **endptr, int base);
```

`strtol`처럼 동작하지만 **64비트 정수**를 처리한다. 일부 플랫폼에서는 **10진(Base 10)**만 지원.

### Interface

```c
int evutil_snprintf(char *buf, size_t buflen, const char *format, ...);
int evutil_vsnprintf(char *buf, size_t buflen, const char *format, va_list ap);
```

표준 `snprintf`/`vsnprintf`처럼 동작. **버퍼가 충분히 길었다면**(끝의 NUL 제외) *얼마나 많은 바이트가 쓰였는지* 그 **개수**를 반환한다. (C99 `snprintf()` 표준 준수; Windows의 `_snprintf()`는 문자열이 버퍼에 다 못 들어가면 **음수**를 반환하는 점과 대조적)

도입 시기:

- `evutil_strtoll()` — **1.4.2-rc**
- 그 외 — **1.4.5**

## Locale-independent string manipulation functions

ASCII 기반 프로토콜 구현 시, **현재 로케일과 무관하게** ASCII 기준으로 문자열을 비교하고 싶을 때.

### Interface

```c
int evutil_ascii_strcasecmp(const char *str1, const char *str2);
int evutil_ascii_strncasecmp(const char *str1, const char *str2, size_t n);
```

`strcasecmp()`/`strncasecmp()`와 같지만 **항상 ASCII**로 비교. **Libevent 2.0.3-alpha**에서 처음 공개.

------

## IPv6 helper and portability functions

### Interface

```c
const char *evutil_inet_ntop(int af, const void *src, char *dst, size_t len);
int evutil_inet_pton(int af, const char *src, void *dst);
```

RFC3493의 `inet_ntop()`/`inet_pton()`과 동일하게 동작하여 IPv4/IPv6 주소를 **포맷/파싱**한다.

- IPv4 포맷: `af=AF_INET`, `src=struct in_addr*`, `dst=버퍼`
- IPv6 포맷: `af=AF_INET6`, `src=struct in6_addr*`
- IPv4/IPv6 파싱: `af=AF_INET` 또는 `AF_INET6`, `src=문자열`, `dst=in_addr*` 혹은 `in6_addr*`

반환값:

- `evutil_inet_ntop()` — 실패 시 `NULL`, 성공 시 `dst`
- `evutil_inet_pton()` — 성공 `0`, 실패 `-1`

### Interface

```c
int evutil_parse_sockaddr_port(const char *str, struct sockaddr *out, int *outlen);
```

`str`에서 주소를 파싱해 `out`에 기록. `outlen`은 `out`의 **가용 바이트 수**를 담은 정수를 가리켜야 하며, **실제 사용 바이트 수**로 갱신된다. 성공 0, 실패 -1. 인식 형식:

- `[ipv6]:port` (예: `[ffff::]:80`)
- `ipv6` (예: `ffff::`)
- `[ipv6]` (예: `[ffff::]`)
- `ipv4:port` (예: `1.2.3.4:80`)
- `ipv4` (예: `1.2.3.4`)

포트가 없으면 결과 `sockaddr`의 포트는 **0**으로 설정.

### Interface

```c
int evutil_sockaddr_cmp(const struct sockaddr *sa1,
    const struct sockaddr *sa2, int include_port);
```

두 주소를 비교해 `sa1 < sa2`면 음수, 같으면 0, `sa1 > sa2`면 양수 반환. `AF_INET`, `AF_INET6`에서 동작하며 다른 주소군은 **동작이 정의되지 않음**. 전순서(total order)를 보장하지만, **정렬 기준은 버전에 따라 달라질 수 있음**.

`include_port`가 **false**면 포트만 다른 `sockaddr`는 **동일**로 취급. **true**면 포트가 다르면 **다름**.

도입 시기:

- 대부분 — **2.0.1-alpha**
- `evutil_sockaddr_cmp()` — **2.0.3-alpha**

------

## Structure macro portability functions

### Interface

```c
#define evutil_offsetof(type, field) /* ... */
```

표준 `offsetof` 매크로와 동일: `type`의 시작부터 `field`가 위치한 **바이트 수**를 반환.

이 매크로는 **2.0.1-alpha**에서 도입되었고, **2.0.3-alpha 이전 버전**에서는 **버그**가 있었다.

------

## Secure random number generator

많은 애플리케이션(예: `evdns`)은 보안을 위해 **예측하기 어려운 난수**가 필요하다.

### Interface

```c
void evutil_secure_rng_get_bytes(void *buf, size_t n);
```

`buf`가 가리키는 **n바이트 버퍼**를 **난수 n바이트**로 채운다.

플랫폼이 `arc4random()`을 제공하면 그걸 사용하고, 아니면 OS의 엔트로피 풀(Windows: `CryptGenRandom`, 그 외: `/dev/urandom`)로 시드한 **자체 `arc4random()` 구현**을 사용.

### Interface

```c
int evutil_secure_rng_init(void);
void evutil_secure_rng_add_bytes(const char *dat, size_t datlen);
```

수동 초기화는 **필수는 아니지만**, 초기화를 보장하려면 `evutil_secure_rng_init()`을 호출할 수 있다. (아직 시드되지 않았다면) RNG에 시드를 주고 **성공 시 0**을 반환. **-1**이면 OS에서 적절한 엔트로피 소스를 찾지 못해, 직접 초기화하지 않으면 **안전하게 사용할 수 없다**는 뜻.

권한을 낮출 가능성 있는 환경(예: `chroot()`)에서는 그 전에 `evutil_secure_rng_init()`을 호출할 것.

추가 엔트로피를 `evutil_secure_rng_add_bytes()`로 **직접 주입**할 수도 있지만, 일반 사용에서는 **불필요**.

이 함수들은 **Libevent 2.0.4-alpha**에서 새로 추가되었다.