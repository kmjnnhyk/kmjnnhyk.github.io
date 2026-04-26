---
layout: post
title: "네이티브 로그 수집을 통한 앱 디버깅 및 성능 분석기"
date: 2026-04-10
categories: [debugging, mobile]
tags: [android, logcat, react-native, expo, foreground-service, doze]
---

Samsung Galaxy 5대에서 60분 오디오 세션이 시간을 넘겨 재생되거나, 도중에 앱 프로세스가 죽거나, 종료 후 로그인 화면으로 튀는 보고가 누적됨. 단일 기기 로그만으로는 패턴 분리가 어려워 5대를 동시에 무선 디버깅으로 연결하고 1분 단위로 자동 분석 문서를 누적하는 인프라부터 구축. 이 위에서 두 번의 세션을 돌려 근본 원인을 좁혀 나간 회차.

## 수집 인프라 설계

여러 접근 중 하이브리드 방식 채택.

- 5대에 대해 `adb logcat -v threadtime`을 백그라운드 스트림으로 띄워 기기별 raw 로그 파일에 적재.
- Python 모니터 스크립트가 매 60초 단위로 각 raw 로그의 델타만 읽어 노이즈 필터링 후 기기별 분석 마크다운에 누적.
- `offsets.json`으로 각 기기 raw 로그의 마지막 읽은 byte offset을 추적하여 중복 처리 방지.
- 매 분 Claude를 호출해 분석시키는 `/loop` 방식은 토큰 비용 측면에서 기각. 모니터링은 결정적 스크립트가 맡고, 사람의 분석은 1시간 단위로 batch 처리.

## 필터 설계 단계의 시행착오

초기 필터에 두 가지 함정이 발견됨.

- **`401`을 HTTP status로 잡으려 했으나** PID `4401`(wpa_supplicant 등) 라인이 전부 신호로 매치됨. `\b401\b`로 word boundary 적용하여 해결.
- **`SurfaceFlinger` 라인이 신호로 잡힘** — 앱이 단순히 foreground 상태라 언급될 뿐인 라인까지 포함됨. `SurfaceFlinger`/`SurfaceComposerClient`/`RenderEngine`의 앱 패키지 언급은 NOISE 분류로 이관.

다기기 동시 로그 수집에서 PID·status code의 우연 일치는 흔한 함정으로 확인됨. word boundary와 NOISE 분류를 사전 설계 단계에서 잡아두지 않으면 분석 문서가 잡음으로 가득 차 활용 불가.

## Session 1: 기본 60분 테스트

관심사별 병렬 grep으로 로그를 분해.

- Network policy / Doze / Standby
- ReactNativeJS / Timer
- HTTP / SSL / Socket errors
- FATAL / ANR / tombstone
- Activity / FOREGROUND_SERVICE
- AudioTrack / MediaPlayer

### 핵심 발견 5건

1. **Foreground service 미등록 — 5대 전부.** R8 난독화된 클래스명으로 `START_FOREGROUND` 호출 시 `Unable to start service ... not found`. AndroidManifest에 service 선언 누락 또는 R8 keep 규칙 누락이 원인 후보.
2. **DNS 차단 — 5대 전부, 60분 마크 직후 +10~37초.** `NetdEventListenerService`에서 `isBlocked=true` 패턴. 앱이 foreground로 복귀하면 차단 해제, idle 시 재차단되는 전형적 **App Standby Bucket / Background Network Restriction** 동작.
3. **1대만 60분 초과 재생.** AudioTrack `frames delivered`로 정확 측정 시 4대는 약 60:10, 1대는 약 61:17까지 재생. cleanup 간격이 4대는 6~7초, 1대만 약 72초로 측정됨.
4. **React Navigation route not found 경고가 세션 내내 반복.** 세션 종료 후 로그인 화면으로 튀는 증상의 유력 후보로 분류.
5. **FATAL / ANR / crash 0건.** 실제 크래시 없이 OS 정책과 cleanup 타이밍 만으로 증상이 발생함을 확인.

### 인과 체인 도출

```
AndroidManifest 의 foreground service 선언 또는 R8 keep 누락
  → START_FOREGROUND 실패
  → foreground service 없이 60분 재생 진행
  → Android가 background-only로 간주
  → App Standby Bucket 진입
  → 60분 마크에 DNS 차단 발동
  → session-end cleanup 의 await network 호출 hang
  → 4대는 차단 직전에 cleanup 완료(타이밍 운)
  → 1대는 차단 진입 후 cleanup 시도 → 72초 지연
```

## Session 2: 배터리 최적화 OFF 대조 테스트

"App Standby Bucket이 진짜 원인인가"라는 가설 검증을 위해 5대 전부 OS 설정에서 "배터리 사용 - 제한 없음" 적용. baseline에서 doze whitelist에 앱이 추가됨을 확인 후 동일 시나리오 재실행.

T+2분 시점 중간 점검 결과:

| 항목 | Session 1 | Session 2 (T+2) |
|---|---|---|
| doze whitelist | none | user-added |
| DNS 차단 (테라피 중) | 발생 | 0건 |
| Foreground service 등록 | 실패 | 실패 |
| React Nav route 경고 | 지속 | 지속 |

테라피 시작 직후부터 DNS 차단이 사라진 것을 통해, 배터리 최적화 해제가 네트워크 차단을 실제로 막고 있음이 확인됨. 단, T+60에서의 결정적 비교는 **5개 logcat stream 전부 exit 255로 끊겨 미수집**. ADB 무선 디버깅의 장시간 안정성 한계가 드러남.

## 권장 수정 우선순위

| 우선순위 | 항목 |
|---|---|
| HIGH | AndroidManifest에 foreground service 등록 + R8 keep + `FOREGROUND_SERVICE_MEDIA_PLAYBACK` permission |
| HIGH | React Navigation의 종료 라우트 등록 확인 |
| MED | cleanup에서 오디오 정지를 네트워크 호출 이전에 수행 + AbortController timeout |
| LOW | 중복 `abandonAudioFocus` 호출 정리 |

## 교훈

- 단일 기기 로그로는 분리 불가능한 증상이 다기기 동시 수집으로 패턴화됨. 동일 시각 5대의 동일 동작이 OS 정책에 의한 것임을 강하게 시사.
- 사용자 보고된 "타이머가 안 멈춤"이 실제로는 **cleanup의 네트워크 호출이 차단된 DNS에 hang되어 audio stop이 지연된 결과**였음. 증상과 원인 사이에 OS 정책 레이어가 끼어 있음을 확인.
- Foreground service 미등록은 dev 빌드에서는 거의 드러나지 않고 R8 난독화·릴리즈 빌드 + 60분 background 시점에서야 외부로 노출되는 종류의 문제. 릴리즈 빌드 기반 시나리오 테스트의 중요성 재확인.
- 장시간 ADB 무선 디버깅은 supervisord 같은 auto-reconnect wrapper 없이 60분 단위 시나리오 검증에 부적합함을 확인. 다음 회차의 선결 과제로 식별.
