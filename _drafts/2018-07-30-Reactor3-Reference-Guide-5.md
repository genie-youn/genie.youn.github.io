---
layout: post
title:  "[Reactor3 Reference Guide] 4. Reactor 핵심 특징 (4)"
date:   2018-07-30 00:02:05 +0900
tag: Reactor
---

## 4.6 Threading

`Flux`와 `Mono`는 스레드를 만들지 않는다. `publishOn` 과 같은 몇몇 메소드가 스레드를 만든다.
또한 작업을 공유함으로써 이러한 메소드들은 다른 스레드풀에서 유휴상태의 스레드를 훔칠수도 있다. 결론적으로 `Flux` 와 `Mono` 둘 다 `Subscriber` 객체가 아니기 때문에 스레드에 대해 관심을 가질 필요가 없다.
`Flux`와 `Mono` 는 연산자에 의존해 스레드와 작업 풀을 관리한다,

`publishOn` 은 다른 스레드에서 다음 연산자를 실행하도록 강제한다. 비슷하게, `subscribeOn` 는 다른 스레드에서 이전의 연사자들을 실행하도록 강제한다. 명심해야 할 것은 구독할때까지 프로세스는 정의되지만 시작되진 않는다는 것이다. 그 이유는 리액터는 이러한 규칙을 사용하여 처리가 어떻게 진행되어야 하는지를 결정하기 때문이다. 그런다음 구독을 시작하면 모든 프로세스가 동작하기 시작한다.

다중 스레드가 작업 공유를 지원하는 예제를 보자

```java
Flux.range(1, 10000)
  .publishOn(Schedulers.parallel())
  .subscribe(result)
```

`Scheduler.parallel()` 은 싱글 스레드로 이루어진 `ExecutorService` 기반의 워커로 구성된 고정 스레드풀을 생성한다. 한 두개의 스레드가 문제를 일으켜도 괜찮도록 최소 네개의 스레드를 생성한다. 그런 다음 `publishOn` 메소드는 이러한 워커 스레드들을 공유하여 요소를 요청할 때 스레드가 방출된 요소를 가져온다. 이러한 방식에서 우리는 자원을 공유함으로써 작업을 공유할 수 있다. 리액터는 자원을 공유하는데 편리한 몇가지 방법을 제공하는데 자세한 내용은 [Schedulers](https://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html) 를 참고해라.

`Scheduler.elastic()` 또한 스레드를 생성하는데 특히 블록킹 리소스 (예를 들면 동기로 동작하는 서비스라던지)를 논-블럭킹으로 만들기 위한 스레드를 쉽게 만들수 있다. 다음 섹션을 참고해라 [동기/블럭킹 호출을 감싸는 방법](https://projectreactor.io/docs/core/release/reference/#faq.wrap-blocking)

이러한 연산자들은 증가 카운터 ~incremental counter~ 와 상태 보호 ~guard conditions~ 를 통해 스레드-안전 ~thread safety~ 을 보장한다. 예를 들어 하나의 스트림을 네개의 스레드에서 동작시키면 (이전의 예제와 같이) 각각의 요청은 카운터를 증가시키며 스레드들이 한 요소에 동시에 접근하지 않게 한다.

## 4.7 Handling Errors

> 에러 핸들링에 관련된 연산자를 빠르게 찾아보고 싶다면, [에러와 관련된 연산자 결정 트리](https://projectreactor.io/docs/core/release/reference/#which.errors) 를 확인해라.

Reactive Streams 에서 에러는 내부적인 이벤트이다. 에러가 발생하면, 서브시퀀스를 중단하고 연산자 체인의 마지막 단계까지 전파되어 `Subscriber` 의 `onError` 메소드를 호출한다.
이러한 에러는 어플리케이션 레벨에서 처리해야 한다. 예를들어 UI에서 에러를 알려주거나, REST 엔드 포인트로 의미있는 에러 페이로드를 전송해야 한다. 이러한 이유로 구독자의 `onError` 메소드는 항상 정의되어야 한다.
