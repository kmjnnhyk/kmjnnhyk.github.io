---
layout: post
title: "모바일 로컬 저장소가 SQLite로 수렴하는 이유 학습기"
date: 2026-04-17
categories: [study, mobile]
tags: [sqlite, expo, react-native, embedded-database, jsi]
---

세션 종료 알림 실패 문제를 해결하기 위해 로컬 저장소 기반 아키텍처를 검토하던 도중, "왜 모바일 앱은 거의 예외 없이 SQLite를 쓰는가"가 단순 관행을 넘어선 근원적 이유에 대한 의문으로 번졌음. 단순히 "쓰면 된다"가 아니라, 네트워크·시스템·런타임 세 층위에서 왜 다른 후보가 살아남지 못하는지를 이해하기 위한 학습.

## 학습 동기

세션 데이터의 영속화를 클라이언트에 둘지, 어떤 방식으로 둘지 결정하는 단계에서 SQLite 외의 후보(AsyncStorage, JSON 파일, Realm 등)를 모두 검토함. 그러나 각 후보가 왜 부적합하고, SQLite가 왜 사실상 표준의 위치에 있는지에 대한 답이 표면적이었음. 이걸 네트워크 모델·OS·RN 아키텍처 수준에서 다시 정리해야, 도입 결정의 근거가 견고해진다고 판단.

## 1. 네트워크 관점 — 모바일은 파티션이 일상

서버-클라이언트 모델의 전통적 가정은 "클라이언트는 얇고, 상태는 서버에 있다". 웹에서는 성립하지만 모바일에서는 다음 이유로 성립하지 않음.

- **Latency** — 셀룰러·Wi-Fi RTT는 수십~수백 ms. 모든 read를 원격으로 보내면 60 fps UI(16.6 ms/frame) 유지 불가.
- **Partition** — 지하철·엘리베이터·터널·비행기 등 네트워크 단절 시나리오는 예외가 아니라 **전제**.
- **라디오 비용** — 셀룰러 모뎀 활성화는 배터리 소모의 가장 큰 원인 중 하나.

결론적으로 모바일은 **offline-first가 기본값**이고, 데이터의 primary copy가 디바이스에 있어야 함. 서버는 sync 대상의 위치가 됨.

## 2. 시스템 관점 — 임베디드 DB가 유일한 답

서버형 DB(MySQL, PostgreSQL)는 별도 프로세스 + TCP 소켓 모델. 다음 이유로 모바일에 들어오지 않음.

- **프로세스 모델 충돌** — 모바일 OS는 background 앱 프로세스를 suspend·kill하므로, 앱 안에 데몬을 띄울 수 없음.
- **배포 모델 충돌** — 서버 DB 바이너리는 수백 MB. 앱에 번들하면 스토어 정책 위반.
- **보안 모델 충돌** — multi-user concurrency·네트워크 노출 전제의 인증 체계는 싱글 유저 로컬 환경에 과잉.

남는 카테고리는 **in-process embedded DB**. 이 안에서 SQLite의 위치는 다음 특성으로 정리됨.

- **OS 내장** — iOS·Android 시스템에 SQLite가 이미 포함되어 있음. 앱 바이너리 사이즈 증가 0.
- **단일 파일** — 하나의 `.db` 파일이 DB 전체. 백업·삭제·이관이 단순.
- **ACID** — key-value 저장소나 JSON 파일과 달리 트랜잭션·롤백이 보장됨.
- **Public domain + 장기 지원** — 라이브러리 벤더 리스크가 낮음. 의료기기 같은 규제 산업의 SOUP 등록 시 방어가 쉬움.

대안 비교를 정리하면 다음과 같음.

| 후보 | 한계 |
|---|---|
| AsyncStorage | key-value 구조라 조건 검색 불가. 부분 수정 시 read-modify-write race. |
| JSON 파일 | atomic write 불가. 동시성 보장 없음. |
| Realm / WatermelonDB | 추가 native 라이브러리 필요. 학습 곡선·규제 문서화 부담. 내부적으로 SQLite를 쓰는 경우도 있음. |
| LevelDB / RocksDB | key-value. 관계형 쿼리 부재. |

## 3. 런타임 관점 — 왜 Expo가 SDK로 번들하는가

RN의 JS 스레드는 단일 스레드. 과거 bridge 시절에는 JS와 Native 사이가 JSON 직렬화로 연결되어 수십 행만 읽어도 수 ms가 직렬화·역직렬화로 사라지는 구조였음.

JSI(JavaScript Interface) 도입 이후 SQLite C API가 **JS의 HostObject로 직접 바인딩**됨. 직렬화 없이 함수 포인터 호출 수준으로 동작하여 bridge 오버헤드가 사실상 0. Expo가 이걸 SDK 레벨로 묶어 단일 import로 Android·iOS·Web(WASM) 전반을 커버함.

이 지점이 중요한 이유 — embedded DB의 ACID·관계형 쿼리 같은 강점은, JSI 같은 메커니즘으로 JS에 노출될 때만 RN 환경에서 실용적으로 쓰일 수 있음. 두 층(임베디드 DB의 시스템적 강점 + JSI의 런타임적 노출)이 동시에 성립해야 비로소 모바일 표준으로 자리잡음.

## 4. ORM과의 카테고리 정리

학습 도중 헷갈리기 쉬운 지점이 "Prisma vs SQLite" 같은 비교. Prisma·Drizzle 등은 **DB가 아니라 ORM**. SQLite·MySQL 같은 DB 엔진 위에 올라가는 타입 세이프 레이어임. RN 스택을 다시 정리하면 다음과 같음.

```
[앱 코드 — TypeScript]
   ↓
[ORM — Drizzle / Prisma]   타입 세이프, 마이그레이션
   ↓
[SDK 래퍼 — expo-sqlite]    JSI 바인딩
   ↓
[OS 내장 — libsqlite3]      C 라이브러리
   ↓
[파일시스템]                 .db 파일
```

서버에서 PostgreSQL + Prisma를 쓰던 DX를 모바일의 SQLite + Drizzle로 거의 동일하게 가져갈 수 있다는 점이, embedded DB 선택의 부담을 추가로 낮춤.

## 정리

근원은 세 층의 중첩.

1. **네트워크** — 모바일에서 로컬 primary storage는 물리적 필연.
2. **시스템** — in-process embedded DB만이 low-latency + ACID + 모바일 프로세스 모델 호환을 동시에 만족.
3. **런타임** — JSI가 embedded DB의 강점을 JS 레이어에 손실 없이 노출하는 유일한 길.

세 층 중 하나라도 무너지면 SQLite의 위치가 흔들리지만, 셋 다 SQLite 쪽으로 정렬되어 있어 사실상 경쟁자가 없음. 도입 결정 자체보다, 도입의 근거가 다층에서 정렬되어 있다는 점이 의료·규제 도메인에서 방어 비용을 낮추는 근거가 됨.
