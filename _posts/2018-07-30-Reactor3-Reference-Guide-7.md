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

> 만약 `onError` 를 따로 정의하지 않는다면, `UnSupportedOperationException`을 뱉게 되어있다. `Exceptions.isErrorCallbackNotImplemented` 메소드로 예외를 좀 더 특정할 수 있다.

리액터는 또한 체인 중간에 에러를 처리하는 수단으로서 에러 핸들링 연산자를 제공한다.

> 에러 핸들링 연산자에 대해 학습하기 전에 항상 *리액티브 시퀀스에서의 모든 에러는 내부 이벤트* 로 처리된다는 것을 잊으면 안된다. 심지어 에러 핸들링 연산자를 사용하더라도, *원본의* 시퀀스가 이어서 실행되도록 할 수는 없다 (왜?) 대신 `onError` 신호를 통해 새로운 시퀀스를 시작하게 할 수 있다. (fallback) 즉 종료된 이전 스트림을 대체하는 것이다.

### 4.7.1 Error Handling Operators

try-catch 블럭에서 예외를 처리하는 몇가지 방법을 알고있을 것이다. 대표적으로 다음과 같은 방법이 있다.

1. 예외를 잡고 스태틱한 기본 값을 던진다.
2. 예외를 잡고 대안이 될 수 있는 fallback 메소드를 실행한다.
3. 예외를 잡고 동적으로 fallback 값을 계산한다.
4. 예외를 잡고 비즈니스 레이어의 예외로 감싸서 다시 던진다.
5. 예외를 잡고 에러 메세지를 남긴 후 고대로 던진다.
6. `finally` 블럭을 사용하여 사용하던 리소스를 정리하거나 자바 7의 `try-with-resource` 구조를 사용한다.

이 모든 방법이 리액터에서는 에러 핸들링 연산자로 구현되어 있다.

이러한 연산자들을 알아보기 전에, 우선 리액티브 체인과 try-catch 구문을 비교해보자.
구독시에 체인의 가장 마지막에 위치한 `onError` 콜백은 `catch` 블럭과 흡사하다.

```java
Flux<String> s = Flux.range(1, 10)
  .map(v -> doSomethingDangerous(v)) // 예외가 발생할 수 있는 변환
  .map(v -> doSecondTransform(v)); // 위에서 예외가 발생된다면 실행되지 않는다.

s.subscribe(value -> System.out.println("RECEIVED : " + value), // 예외가 발생하지 않는다면 (성공한다면) 출력
            error -> System.err.println("CAUGHT : " + error)) // 예외가 발생하면 에러메세지 출력
```

이 예제는 다음 try-catch 구문과 비슷하다.

```java
try {
  for (int i = 1; i < 11; i++) {
    String v1 = doSomethingDangerous(i);
    String v2 = doSecondTransform(v1);
    System.out.println("RECEIVED : " + v2);
  }
} catch (Throwable t) {
  System.err.println("CAUGHT : " + t);
}
```
