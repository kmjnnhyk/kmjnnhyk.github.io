---
layout: post
title: "비동기 실행과 모바일 OS·네트워크 경계 메커니즘 학습기"
date: 2026-04-23
categories: [study, mobile]
tags: [async, microtask, javascript, doze, abort-controller, network-stack]
---

백그라운드에서 API 요청이 사라지고, 60분짜리 세션 종료 알림이 가끔만 도착하고, 어떤 요청은 영원히 pending에 머무는 현상을 디버깅하던 중에 떠오른 의문 — "JS는 분명히 코드가 흐르고 있는데, 왜 데이터는 사라지는가". 이 질문이 async/await의 실제 동작, microtask queue, fetch가 OS에 위임하는 방식, OS background suspend, Android Doze, 그리고 axios 한 줄이 서버 프로세스에 도달하는 전 경로까지를 한 번에 정리해야 풀린다는 결론에 도달함. 이 글은 그 학습의 정리.

## 학습 동기

다음 종류의 의문이 누적되어 있었음.

- async/await에서 "함수를 Call Stack에서 내린다"는 것이 OS로 무엇을 넘긴다는 뜻인지.
- Microtask Queue가 JS와 OS 사이의 다리인지, 아니면 JS 안에서만 도는 큐인지.
- fetch가 pending 상태로 멈춰 있을 때 정확히 어디서 정지되어 있는 것인지.
- Android Doze Mode가 네트워크와 타이머에 어떻게 다르게 영향을 주는지.
- axios의 `config.headers.Authorization = ...` 한 줄이 서버 프로세스의 `request.getHeader(...)`에 도달하기까지 어느 레이어를 거치는지.

비동기와 OS 경계의 작동 메커니즘이 공통 분모. 이걸 정리하지 않으면 디버깅 시 매번 표면 추론에 그치게 된다고 판단.

## 1. async/await는 엔진 내부의 스케줄링

`await`은 OS에 무엇을 넘기는 행위가 아니라 **JS 엔진 내부의 일시 중단 스케줄링**.

`await` 실행 시 일어나는 일은 다음과 같음.

1. await 뒤 표현식(보통 Promise) 평가.
2. 현재 async 함수 실행을 suspend. 나머지 코드를 Promise의 후속 콜백으로 등록.
3. 해당 async 함수를 Call Stack에서 pop.
4. Call Stack에 남은 동기 코드가 계속 실행됨.

OS 개입 여부는 **await 대상**에 따라 달라짐.

- `await fetch(...)` — 네트워크 I/O이므로 OS(libuv·kernel epoll·kqueue) 개입.
- `await sleep(1000)` — 타이머이므로 런타임이 OS 타이머를 활용.
- `await Promise.resolve(42)` — OS 전혀 무관, 순수 JS 엔진 내부 microtask.

즉 `await` 자체는 "Promise가 settle되면 나머지를 실행하라"는 **엔진 레벨 스케줄링 요청**이고, Promise 내부에 무엇이 들어 있느냐가 OS 관여 여부를 결정.

## 2. Microtask Queue는 엔진 내부 큐

Microtask Queue는 OS와 직접 연결된 다리가 아니라, JS 엔진이 Promise 기반 후속 작업을 우선 처리하기 위한 **엔진 내부 큐**.

흐름은 다음과 같음.

1. OS·런타임의 비동기 작업(네트워크, 파일, 타이머)이 완료.
2. 해당 콜백이 Macrotask Queue에 적재.
3. 콜백 안에서 Promise가 resolve되면 후속 코드가 Microtask Queue로.
4. **현재 Macrotask 1개 종료 → Microtask Queue 전부 비움 → 다음 Macrotask**.

이 "한 번에 전부 비운다"는 특성 때문에, 쌓인 await 후속 체인이 일제히 resolve되면 cascade 형태로 폭발하는 현상이 발생할 수 있음.

## 3. fetch는 I/O를 OS에 위임

`fetch(url)` 호출 자체는 JS가 네트워크를 다루지 않음. 엔진은 격리된 샌드박스라 소켓 자체를 모름.

전송 흐름.

1. JS가 fetch 호출.
2. 엔진이 C++ 브릿지(JSI 등)를 통해 OS에 `write` syscall 발행.
3. CPU는 syscall 발행 즉시 복귀(논블로킹).
4. 실제 패킷 송출은 NIC가 DMA로 독립 처리.
5. JS는 `Promise { pending }`만 보유.

수신 흐름.

1. NIC에 응답 도착 → 인터럽트로 CPU 깨움.
2. 커널이 Socket Buffer에 적재.
3. 런타임이 데이터 가용성 알림 수신.
4. Promise가 resolve → Microtask로 후속 코드 예약.

함의 — JS의 non-blocking 장점은 명확하지만, **OS가 I/O를 suspend시키면 JS는 아무것도 못 함**. 이 상태는 "실패"가 아니라 "정지". `try/catch`로 잡히지 않음.

## 4. OS Background Suspend — 이벤트 루프 동결

모바일 OS는 배터리 보호를 위해 background 앱의 CPU 할당을 줄이거나 정지시킴. 그 결과 **JS 이벤트 루프 사이클 자체가 멈춤** → 모든 시간 기반 메커니즘이 동결됨.

- `setTimeout`을 10초 timeout으로 걸어 두어도, 타이머를 fire할 루프가 멈춰 있으면 timeout 자체가 발생하지 않음.
- fetch는 pending 상태로 시간이 얼어붙음.
- AbortController의 abort 호출도 이벤트 루프가 깨어나야 fire되므로 함께 무력화됨.

## 5. AbortController의 본질

`new AbortController()`로 두 객체가 연결됨 — `controller`(신호 송신)와 `controller.signal`(신호 수신, fetch 등에 전달).

`abort()` 호출 시 흐름.

1. `signal.aborted = true` 동기적 변경.
2. fetch 내부 리스너가 OS에 `close` syscall 발행.
3. OS가 TCP 연결 강제 종료.
4. Promise가 `AbortError`로 reject.

본질은 "fetch에게 소켓을 닫으라는 **JS 레벨 신호 채널**". 실제 종료는 OS가 수행하지만, **신호를 트리거하는 명령 자체가 JS 이벤트 루프 안에서 처리**됨. 따라서 background suspend 상황에서는 timeout 타이머도, abort 호출도, 소켓 close도, Promise reject도 전부 동결됨. **AbortController는 포그라운드 전용 방어선**이고 근본 해결이 아님.

## 6. 비동기 디버깅의 세 경계

세션 데이터가 사라지거나 시간을 넘겨 잡히는 현상은 다음 세 경계가 동시에 무너질 때 발생함.

1. **JS ↔ OS** — fetch가 I/O를 OS에 위임. OS가 멈추면 JS는 "실패가 아닌 정지". try/catch 불가.
2. **포그라운드 ↔ 백그라운드** — 이벤트 루프·setTimeout·AbortController가 suspend에서 동결. OS 네이티브 타이머만 일부 생존.
3. **프로세스 생존 ↔ 네트워크 생존** — Foreground Service로 프로세스가 살아 있어도, 제조사 레이어나 OS 정책이 독립적으로 소켓을 닫을 수 있음. "JS는 실행 중인데 데이터만 안 나가는" 패턴이 가능.

이 세 경계가 분리되어야 디버깅 시 어느 레이어의 문제인지 식별 가능. 표면 증상만 보면 "네트워크 에러"처럼 보이지만 실제로는 OS 정책에 의한 정지인 경우가 많음.

## 7. Android Doze와 Maintenance Window

Doze는 Android의 강제 배터리 절약 모드. 화면 꺼짐 + 충전 아님 + 기기 정지 상태가 모두 충족될 때 진입.

두 단계가 있음.

- **Light Doze** — 진입 빠름. 네트워크 제한·JobScheduler 지연. 완전 차단은 아님.
- **Deep Doze** — 네트워크 접근 차단, AlarmManager·JobScheduler 보류, Wake Lock 무시.

Doze 진입 후 OS는 주기적으로 **Maintenance Window**라는 짧은 창을 열어 보류된 작업을 처리함. 창 사이의 간격은 **exponential backoff**로 늘어남(15분 → 30분 → 1시간 → 최대 약 6시간). 창이 닫히면 다시 차단.

특수 케이스로 `setExactAndAllowWhileIdle()` 알람은 Doze 상태에서도 fire 가능 — fire 순간 mini Maintenance Window 효과로 네트워크가 잠깐 열림. 이 메커니즘이 있어, 일반 주기 타이머는 모두 막혀도 종료 알림 같은 단발성 호출이 성공하는 역설이 발생할 수 있음.

## 8. axios → 서버까지의 전 경로

axios의 한 줄이 서버 프로세스에 도달하는 경로를 정리하면 다음과 같음.

```
[앱 JS]  axios(config)
  → XHR polyfill (RN)
  → Native Module bridge (New Arch에서는 JSI 직결)
  → NSURLSession(iOS) / OkHttp(Android)
  → Socket API
  → TCP/IP 스택 → NIC → 라우팅
  → 서버 NIC → 서버 OS → nginx/LB
  → WAS 프로세스 → HTTP 파서 → 컨트롤러
```

각 레이어의 역할이 다르며, 문제는 주로 **레이어 경계**에서 터짐. 몇 가지 핵심.

- **interceptor 시점의 헤더는 JS 객체** — 실제 HTTP 메시지 바이트는 훨씬 뒤(NSURLSession·OkHttp 단계)에서 조립됨. interceptor가 sync로 토큰을 읽어야 하는 이유 — 비동기로 읽으면 "전송 시점의 토큰"이 다중 요청 사이에 어긋나면서 race condition의 토대가 됨.
- **HTTP/1.1 메시지의 라인 구분자는 CRLF**. 헤더 블록 끝의 빈 줄(`\r\n\r\n`)이 본문 시작 신호. HTTP/2는 이진 프레임이라 이 규칙 없음.
- **multipart boundary**는 본문에 절대 안 나올 충분히 긴 랜덤 문자열로 파트를 구분. axios에서 Content-Type을 직접 박지 않고 native에 맡기는 이유 — boundary 생성과 헤더 세팅을 한 주체가 원자적으로 처리하게 만드는 방어.
- **TLS 1.3 handshake**는 1 RTT. 인증서 검증·세션키 합의 후 모든 바이트가 AES-GCM으로 암호화. axios instance를 모듈 최상위 싱글톤으로 두면 OkHttp·NSURLSession connection pool이 공유되어 두 번째 요청부터 handshake 비용을 건너뜀.
- **모바일의 L1·L2 변동성** — Wi-Fi ↔ LTE 전환 시 IP가 바뀌며 기존 TCP 커넥션이 파괴됨. 이 시나리오가 race condition의 실제 발생 토대. HTTP/3(QUIC)는 connection ID로 IP 변경에도 커넥션을 유지하지만 현재 axios + 기본 native client 스택은 HTTP/2 TCP 위에서 동작.

## 9. New Architecture와 Bridge

"Bridge가 사라졌다"는 표현의 정확한 의미.

- **Legacy Bridge** — JS ↔ Native 사이의 비동기 직렬화 레이어. 모든 호출을 JSON으로 직렬화해 큐에 쌓고 프레임 단위로 batch flush. 비동기·batched·serialized.
- **New Architecture** — JSI가 자리 대체. 두 세계가 같은 메모리 공간에서 함수 포인터·객체 참조를 직접 주고받음. JSON 직렬화 없음. 동기 호출 가능.

다만 RCTNetworking의 응답은 여전히 비동기 이벤트 emitter 패턴 — 네트워크 I/O 자체의 비동기성은 그대로이므로 JSI의 동기 호출 이점이 직접 노출되지 않음. 응답 바디가 청크 단위로 JS에 전달되는 구조는 동일.

성능 측면 함의.

- JS 스레드 blocking 가능성 감소.
- 동시 401 같은 race가 압축된 시간 윈도우 대신 분산된 타이밍에서 발생 → 재현이 더 어려워질 수 있음.
- 백그라운드 suspend의 영향은 동일. JSI가 OS 정책을 우회하지 않음.

## 정리

학습 후 정렬된 그림.

1. **async/await는 엔진 내부 스케줄링**. OS는 Promise 내부에 무엇이 있는지에 따라 관여하거나 말거나 함.
2. **Microtask Queue는 엔진 내부**. 한 번에 전부 비우는 특성으로 cascade 폭발 가능.
3. **fetch의 pending은 OS 상태에 종속**. 이벤트 루프 동결 시 timeout·abort 모두 작동 안 함. "실패"가 아니라 "정지"이며 try/catch 불가.
4. **Doze + Maintenance Window**는 네트워크 가용성 문제이지 프로세스 생존 문제가 아님. FGS로 프로세스가 살아도 네트워크는 닫혀 있을 수 있음.
5. **클라이언트 방어는 근본 해결이 아님**. 서버 측 보완(선제 갱신·폴백·멱등성)이 없으면 사일런트 실패를 막을 수 없음.
6. **axios 한 줄이 서버에 도달하기까지 최소 10개 레이어**. 문제는 레이어 경계에서 주로 터지므로, 디버깅의 출발점은 **어느 경계에서 정지되었는가**를 식별하는 것.
