---
layout: single
title:  "Libevent Reference Manual 기초"
date: 2025-08-21
tags: libevent
categories: 
   - Libevent
---

# Libevent Reference Manual: 기초

## Libevent 한눈에 보기

Libevent는 빠르고 이식성 있는 비동기(nonblocking) IO를 작성하기 위한 라이브러리이다.

그 설계 목표는 다음과 같다:

### 이식성 (Portability)

Libevent로 작성된 프로그램은 Libevent가 지원하는 모든 플랫폼에서 동작해야 한다.
 비동기 IO를 할 수 있는 좋은 방법이 없는 경우에도, 제한된 환경에서 실행될 수 있도록 그럭저럭 가능한 방법을 지원해야 한다.

### 속도 (Speed)

Libevent는 각 플랫폼에서 가능한 가장 빠른 비동기 IO 구현을 사용하려 하며, 큰 오버헤드를 발생시키지 않도록 설계되었다.

### 확장성 (Scalability)

수만 개의 활성 소켓을 동시에 처리해야 하는 프로그램에서도 잘 동작하도록 설계되었다.

### 편의성 (Convenience)

가능한 한, Libevent로 프로그램을 작성하는 가장 자연스러운 방식이 안정적이고 이식성 있는 방식이어야 한다.

------

## Libevent의 구성 요소

- **evutil**
   서로 다른 플랫폼의 네트워킹 구현 차이를 추상화하는 범용 기능.
- **event와 event_base**
   Libevent의 핵심.
   플랫폼별 이벤트 기반 비동기 IO 백엔드에 대한 추상 API를 제공한다.
   소켓이 읽기/쓰기 가능할 때 알려주고, 기본적인 타임아웃 기능과 OS 신호 감지를 수행한다.
- **bufferevent**
   이벤트 기반 코어를 더 편리하게 감싸는 래퍼(wrapper).
   소켓 준비 여부를 알리는 대신 실제 IO가 발생했을 때를 알려준다.
   여러 백엔드를 지원하므로 Windows IOCP 같은 빠른 비동기 IO 방식도 활용할 수 있다.
- **evbuffer**
   bufferevent의 기반이 되는 버퍼를 구현하며, 효율적이고 편리한 접근을 위한 함수 제공.
- **evhttp**
   간단한 HTTP 클라이언트/서버 구현.
- **evdns**
   간단한 DNS 클라이언트/서버 구현.
- **evrpc**
   간단한 RPC 구현.

------

## 라이브러리

Libevent를 빌드하면 기본적으로 다음 라이브러리가 설치된다:

- **libevent_core**
   모든 코어 이벤트 및 버퍼 기능 포함.
   event_base, evbuffer, bufferevent, 유틸리티 함수 포함.
- **libevent_extra**
   HTTP, DNS, RPC 같은 프로토콜 특화 기능 정의.
- **libevent**
   역사적 이유로 존재하며 libevent_core와 libevent_extra의 내용을 모두 포함.
   앞으로 사라질 수도 있으므로 사용하지 않는 것이 좋다.

일부 플랫폼에서만 설치되는 라이브러리:

- **libevent_pthreads**
   pthreads 기반의 스레딩과 락 기능 추가.
   멀티스레드 기능을 쓰지 않는다면 pthreads에 링크할 필요 없도록 분리되어 있다.
- **libevent_openssl**
   bufferevent와 OpenSSL을 통한 암호화 통신 지원.
   암호화 연결을 쓰지 않는다면 OpenSSL에 링크할 필요 없도록 분리되어 있다.

------

## 헤더 파일

모든 공개 Libevent 헤더는 event2 디렉토리에 설치된다.
 헤더는 세 가지 범주로 나뉜다:

- **API 헤더**
   현재 공개된 Libevent 인터페이스 정의. 접미사 없음.
- **호환성 헤더**
   사용 중단된 함수 정의 포함. 구버전 프로그램 포팅할 때만 사용.
- **구조체 헤더**
   자주 변하는 구조체 레이아웃 정의.
   일부는 빠른 접근이나 역사적 이유로 노출.
   직접 의존하면 다른 버전과 바이너리 호환성 깨질 수 있음.
   접미사 "_struct.h".

(event2 디렉토리가 없는 구버전 헤더도 있으며, “구버전 Libevent를 다뤄야 하는 경우” 참고)

------

## 구버전 Libevent를 다뤄야 하는 경우

Libevent 2.0은 API를 더 합리적이고 오류 발생 가능성이 적게 개정했다.
 가능하다면 새로운 프로그램은 2.0 API를 사용해야 한다.
 하지만 기존 애플리케이션 업데이트나 2.0 이상 설치 불가 환경 지원 때문에 구버전 API를 다뤄야 할 수 있다.

구버전은 헤더 파일이 적었고 event2 디렉토리를 사용하지 않았다:

| 구버전 헤더 | 현재 헤더 |
 | event.h | event2/event*.h, event2/buffer*.h, event2/bufferevent*.h, event2/tag*.h |
 | evdns.h | event2/dns*.h |
 | evhttp.h | event2/http*.h |
 | evrpc.h | event2/rpc*.h |
 | evutil.h | event2/util*.h |

Libevent 2.0 이상에서는 구버전 헤더도 새 헤더를 불러오는 래퍼로 남아 있다.

추가 참고 사항:

- 1.4 이전에는 libevent 하나의 라이브러리에 지금의 libevent_core와 libevent_extra 기능이 모두 포함.
- 2.0 이전에는 락 지원이 없었으며, 스레드 안전하려면 동시에 같은 구조체를 쓰지 않도록 직접 보장해야 했다.

------

## 버전 상태 참고

- 1.4.7 이전 버전은 완전히 구식으로 간주해야 한다.
- 1.3e 이전 버전은 버그가 매우 많으므로 사용하지 않아야 한다.

또한, 1.4.x 이하 버전에는 새로운 기능을 추가하지 않는다. 안정 릴리스로 유지되기 때문이다.
 1.3x 이하 버전에서 버그를 발견한다면 최신 안정 버전에서도 존재하는지 반드시 확인해야 한다.
 후속 릴리스가 나온 데에는 이유가 있기 때문이다.