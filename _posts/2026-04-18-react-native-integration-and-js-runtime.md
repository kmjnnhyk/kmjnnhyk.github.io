---
layout: post
title: "RN 통합 모델 비교와 네이티브 안에서의 JS 실행 메커니즘 학습기"
date: 2026-04-18
categories: [study, mobile]
tags: [react-native, expo, granite, jsi, turbo-modules, fabric]
---

토스 Granite와 Expo가 같은 React Native를 쓰면서도 통합 방식이 근본적으로 다르다는 이야기를 접한 뒤, 두 모델의 동작 원리를 비교하고 싶었음. 그 흐름이 자연스럽게 "네이티브 앱 안에서 JavaScript가 어떻게 실행되는가"라는 더 근본적인 의문으로 이어짐. 평소 RN을 쓰면서 그냥 추상화 뒤에 두고 지나갔던 부분을 한 번 정리해 보는 학습.

## 학습 동기

RN 앱을 운영하면서도 다음 질문에 대한 답이 표면적이었음.

- 앱이 켜진 뒤 JS 번들이 실제로 언제·어떻게 실행되는가.
- JS와 Native가 같은 프로세스 안에 있다는 것이 정확히 어떤 의미인가.
- New Architecture 도입으로 "Bridge가 사라졌다"는 표현이 무엇을 가리키는가.
- Granite 같은 브라운필드 통합과 Expo 같은 그린필드 통합은 어디서 갈라지는가.

이 질문들을 한 번에 정리하지 않으면 라이프사이클·네트워크·성능 디버깅 시 매번 표면 추상화 수준에서 멈추게 된다고 판단.

## Part 0. 누가 앱의 호스트인가

두 통합 모델의 본질적 차이는 "앱의 주인이 누구인가"라는 관점에서 정리됨.

### Expo — JS가 주인, Native는 얇은 shell

`AppDelegate` / `MainActivity`가 거의 비어 있고, RN 런타임을 부트스트랩하는 역할만 수행. 진입점·내비게이션·라우팅·상태관리 모두 JS가 주도. 네이티브 코드는 "JS가 호출할 수 있는 기능 제공자" 위치.

### Granite — Native가 주인, RN은 손님

기존 iOS·Android 앱에 RN 화면을 끼워 넣는 **브라운필드 통합**. 네이티브 앱이 전체 내비게이션 스택을 쥐고, RN 화면은 그 위에 하나의 ViewController·Fragment로 얹힘. 결제·송금·주식 등 수십 개 기능을 담은 슈퍼앱이 기존 네이티브 자산을 보존하면서 RN의 개발 속도를 결합하기 위해 채택하는 모델.

### JS 번들 전략 차이

| | Expo | Granite |
|---|---|---|
| 번들 전략 | 단일 monolithic | feature별 마이크로 번들 |
| 크기 | 수 MB | 수백 KB × N |
| 공통 런타임 | 번들에 통합 | 베이스 번들에 1회 |
| 배포 | 전체 교체 | feature별 독립 배포 |
| 팀 독립성 | 낮음 | 높음 |

핵심은 "앱 하나 = 번들 하나"인가, "앱 하나 = 베이스 + N개의 feature 번들"인가의 차이. 단일 도메인 앱은 Expo가 단순함과 규제 적합성에서 유리. 다도메인·다팀 슈퍼앱은 Granite가 팀 독립 배포·번들 크기에서 유리.

## Part 1. 부트스트랩 체인

iOS 기준으로 정리하면 다음과 같음.

```
iOS kernel
  └─ main()
      └─ UIApplicationMain()
          └─ AppDelegate
              └─ ExpoAppDelegate           JS 번들 URL 결정 (OTA 포함)
                  └─ RCTAppDelegate         RootView 팩토리 생성
                      └─ RCTHost            New Arch 진입점
                          ├─ JSI Runtime
                          ├─ Turbo Module 레지스트리
                          └─ JS 번들 로드·실행
                              └─ Surface 생성 → window 마운트
```

각 레이어의 역할은 다음과 같음.

- **AppDelegate** — 모든 iOS 앱의 공통 진입점. RN 앱은 이걸 얇게 상속해 사용.
- **RCTAppDelegate** — RN이 제공하는 공통 부트스트랩. JS 엔진 켜기, 번들 읽기, 뷰 마운트의 보일러플레이트를 캡슐화.
- **ExpoAppDelegate** — RCTAppDelegate를 한 번 더 감싸 OTA 업데이트(expo-updates), dev client URL, Expo 모듈 자동 등록 등을 추가. 핵심은 **JS 번들 URL 결정 로직을 가로채** 디스크에 저장된 최신 OTA 번들을 우선 로드하는 것.
- **RCTHost** — New Architecture의 심장. 구 아키텍처에서 RCTBridge가 가졌던 역할을 대체하며 JSI Runtime·Turbo Module 매핑·Surface 관리·번들 로더를 소유. **하나의 RCTHost에 여러 Surface(RN 화면)를 띄울 수 있다는 점**이 Granite 같은 브라운필드 통합에서 JS 엔진 하나를 공유하며 여러 RN 화면을 띄우는 설계의 기반.

Android 쪽도 구조는 동일. `MainActivity ≈ AppDelegate`, `ReactActivity ≈ RCTAppDelegate`, `ReactHost ≈ RCTHost`.

## Part 2. 네이티브 프로세스 안에서 JS는 어떻게 실행되는가

한 줄로 압축하면 — **앱 프로세스 안에 JS 엔진이라는 C++ 라이브러리가 있고, 네이티브 코드가 이 라이브러리의 API를 호출해 JS를 실행·조작한다**.

### 1. JS 엔진은 라이브러리

Hermes(`libhermes`)나 JSC는 정적·동적 라이브러리. 앱 빌드 시 앱 바이너리에 링크되어 같은 메모리 공간에서 동작. JS는 별도 프로세스가 아님.

- Hermes는 AOT 컴파일로 JS를 빌드 타임에 바이트코드로 변환. 런타임에는 VM이 바이트코드를 실행.
- JSC는 인터프리터 + JIT.

### 2. 네이티브가 JS 엔진에 번들을 먹임

네이티브 코드가 JS 엔진의 `evaluateJavaScript` 한 번을 호출하면서 모든 게 시작. VM이 번들을 읽어 전역 스코프 코드를 실행. `AppRegistry.registerComponent` 같은 부트스트랩이 이 시점에 실행되어 React 컴포넌트 트리가 메모리에 올라옴.

- 개발 빌드는 Metro 서버에서 fetch.
- 프로덕션은 앱 번들 내부 파일을 read.
- Granite는 CDN에서 feature 번들을 다운로드 후 `evaluateJavaScript`에 넘기는 동적 번들 로딩.

### 3. JSI — JS와 C++ 사이의 유리창

JSI는 엔진 독립적인 C++ 추상 인터페이스. Hermes·JSC 어느 엔진을 쓰든 같은 인터페이스로 JS 값을 조작 가능. 핵심 능력은 **네이티브가 JS 전역에 함수·객체를 직접 꽂아 넣을 수 있다는 것**.

JS에서 그 함수를 호출하면 C++ 람다가 동기적으로 실행됨. 직렬화·크로스-스레드 큐잉·Bridge 어디에도 거치지 않음. 같은 스레드·같은 이벤트 루프 틱에서 결과 반환.

### 4. Turbo Modules — JSI의 모듈 시스템

JSI 호출을 매번 손으로 등록하지 않도록 한 단계 추상화한 것. JS의 `NativeModules.MyModule.doThing()` 호출이 다음 흐름을 거침.

1. JS — TurboModule registry에서 `MyModule` 가져옴 (JSI로 C++ 객체 획득).
2. 메서드 테이블에서 `doThing` 찾음.
3. JSI Function::call → C++ → Objective-C/Swift/Kotlin override 실행.
4. 리턴값을 `jsi::Value`로 감싸 JS로 반환.

Expo Modules API는 이 위에 Swift·Kotlin DSL을 얹은 얇은 레이어. 성능 특성이 Turbo Modules와 유사한 이유가 이것 — 밑바닥은 같음.

### 5. UI는 어떻게 그려지는가 (Fabric)

JS의 `<View>`는 UI 그 자체가 아니라 **명령서**. 흐름은 다음과 같음.

1. React가 JS에서 diff 계산.
2. JSI를 통해 C++ Shadow Tree에 반영.
3. C++ Yoga 레이아웃 엔진이 플렉스박스 계산.
4. UI 스레드로 mount 명령 전달.
5. 실제 `UIView` / `android.view.View` 생성.

RN 앱을 Xcode Instruments로 열면 평범한 UIView 계층이 보이는 이유 — 실제 네이티브 뷰가 그대로 존재하기 때문.

### 6. 스레드 모델

세 종류의 스레드가 동작.

- **JS 스레드** — Hermes VM과 React 코드가 실행되는 곳. 단일 이벤트 루프.
- **UI 스레드** — 터치 수신, UIView 조작.
- **백그라운드 스레드들** — Shadow Tree 계산, Turbo Module의 비동기 작업 등.

JS 스레드에서 무거운 동기 연산을 하면 터치가 씹히는 이유, 그리고 OS가 앱을 suspend하면 JS 이벤트 루프 자체가 멈추는 이유 모두 이 모델에서 직접 도출됨.

## 정리

학습 후 다시 정리되는 그림은 다음과 같음.

- RN 앱은 **단일 프로세스 안에서 두 세계가 같은 메모리를 공유하며 협업**하는 구조. JS는 별도 세계가 아니라 그 프로세스 안에 링크된 C++ 라이브러리(엔진) 위에서 돌아가는 스크립트.
- JSI는 두 세계 사이의 함수 포인터 수준 직접 통로. Turbo Modules·Expo Modules는 그 통로를 편하게 다루도록 만든 모듈 시스템.
- 두 통합 모델의 차이는 **누가 앱 라이프사이클의 호스트인가**로 압축됨. Granite는 native가 호스트이고 RN은 게스트, Expo는 RN이 호스트이고 native는 얇은 shell. 어느 쪽이 더 좋다가 아니라 **앱의 도메인 형태에 따라 다르게 정렬**됨.
- 네트워크·라이프사이클·성능 디버깅 시 "OS가 앱을 멈췄다 = JS 이벤트 루프가 멈췄다"라는 직접 연결이 명확해짐. 이 한 줄 이해가 이후 비동기 디버깅의 출발점이 됨.
