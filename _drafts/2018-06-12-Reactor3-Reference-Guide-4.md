---
layout: post
title:  "[Reactor3 Reference Guide] 4. Reactor 핵심 특징"
date:   2018-06-12 00:02:05 +0900
tag: Reactor
---

원문 : [https://projectreactor.io/docs/core/release/reference/docs/index.html#core-features](https://projectreactor.io/docs/core/release/reference/docs/index.html#core-features)

리액터 프로젝트의 핵심 아티팩트는 Reactive Streams Specification 을 구현하고 자바 8을 지원하는 리액티브 라이브러리인 `reactor-core` 이다.

리액터는 방대한 양의 연산자들을 제공하고, `Mono`와 `Flux` 를 비롯한 조합하기에 용이한 `Publisher` 구현체 타입들을 제공한다. `Flux` 객체는 0부터 N개의 요소를 갖는 리액티브 시퀀스를 나타내고, `Mono` 객체는 0 혹은 1개의 요소를 갖는 리액티브 시퀀스를 나타낸다.

이러한 구분은 타입에 비동기 작업의 관계 수(cardinality) 에 대한 의미를 부여한다. 예를들어 HTTP 요청은 오로지 하나의 HTTP 응답을 생산하기 때문에 `count` 연산자는 의미가 없다. 그러므로 HTTP 요청을 `Mono<HttpResponse>` 로 나타내는 것이 `Flux<HttpResponse>` 로 나타내는 것보다 적절하며, 0개 혹은 1개의 요소를 가질때 필요한 연산자만을 제공한다.

작업의 관계 수를 변경하는 연산자는 타입 또한 변경한다. 예를들어 `Flux`의 `count`연산자는 `Mono<Long>`을 반환한다.

### 4.1 0-N개 요소의 비동기 시퀀스 Flux

`Flux<T>`는 0부터 N개의 방출된 요소의 비동기 시퀀스를 나타내고, 완료 신호와 에러로 인해 종료될 수 있는 표준 `Publisher<T>` 이다. 그러므로, `Flux`가 가질 수 있는 값은 값과 완료신호와 에러이다. Reactive Streams 스펙에서 이러한 세가지 신호의 타입은 downstream 객체의 `onNext`, `onComplete`, `onError`를 호출한다.

이처럼 처리 가능한 신호가 많기때문에, `Flux`는 범용적인 반응형 타입이라 할 수 있다. 모든 이벤트는 (심지어 종료하는 이벤트일지라도) 선택적이라는걸 명심해라.
`onComplete` 이벤트가 존재하는데  `onNext`가 없다는건 비어있는 유한한 시퀀스를 의미한다. 이 시퀀스에 `onComplete` 를 지운다면 비어있는 무한한 시퀀스를 의미하게 된다. 비슷하게 무한한 시퀀스가 꼭 비어있을 필요는 없다. 예를들어 `Flux.interval(Duration)` 은 무한하고 주기적으로 Long의 값을 배출하는 `Flux<Long>`를 만들어낸다.


### 4.2 0 혹은 1개의 비동기 결과를 갖는 Mono

`Mono<T>` 는 보통 한개의 요소를 갖고, `onComplete` 나 `onError`로 종료될 수 있는 특별한 `Publisher<T>` 이다.

`Flux`에 제공되는 연산자중 일부만 제공되는데, 예를들면
combination operators can either ignore the right hand-side emissions and return another mono or emit values from both sides ?
후자의 경우, 양쪽에서 배출된 값은 `Flux`로 변경된다. 예를들어 `Mono#then(Mono)`는 다른 `Mono`를 반환하는 반면에 `Mono#concatWith(Publisher)`는 `Flux`를 리턴한다.

`Mono` 는 반환하는 값이 없이 오로지 완료의 의미를 갖는 비동기 작업(얘를들면 `Runnable` 같은)을 나타낼 수 있다는걸 명심해라.

### 4.3 Flux와 Mono를 만들고 구독하는 가장 쉬운 방법

`Flux`와 `Mono`를 시작하는 가장 쉬운 방법은 각각의 클래스에 정의된 수많은 팩토리 메소드를 사용하는 것이다.

예를들어, `String` 으로부터 어떠한 시퀀스를 생성할 때, 문자열들을 열거하거나 컬랙션에 담아서 이로부터 Flux를 생성하는 방법이 있다.

```java
Flux<String> seq1 = Flux.just("foo", "bar", "foobar");

List<String> iterable = Arrays.asList("foo", "bar", "foobar");
Flux<String> seq2 = Flux.fromIterabble(iterable);
```
이 외에도 다른 팩토리 메소드도 있다.

```java
Mono<String> noData = Mono.empty();
Mono<String> data = Mono.just("foo");
Flux<Integer> numbersFromFiveToSeven = Flux.range(5, 3);
```

Flux나 Mono가 구독될때 Java 8의 람다를 사용한다. 다양한 콜백의 람다들을 조합할 수 있도록 다음과 같은 메소드 시그니처의 `.subscribe()` 메소드를 제공한다.

```java
// 시퀀스 구독을 실행
subscribe()

// 생산된 각각의 요소들을 제어
subscribe(Consumer<? super T> consumer);

// 요소들을 제어하고 예외에 반응
subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer);

// 요소들을 제어하고, 예외에 반응하며 시퀀스가 성공적으로 완료되었을때 무언가 액션을 수행
subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer,
          Runnable completeConsumer);

// 요소들을 제어하고, 예외에 반응하며 시퀀스가 성공적으로 완료되었을때 무언가 액션을 수행할 뿐만 아니라, 이 구독으로 생산된걸 다시 구독(Subscription)
subscribe(Consumer<? super T> consumer,
          Consumer<? super Throwable> errorConsumer,
          Runnable completeConsumer,
          Consumer<? super Subscription> subscriptionConsumer);
```

### 4.3.1 `subscribe` 메소드 예제

이번 세션은 `subscribe`의 다섯가지 메소드 시그니처에 대한 예제를 담고있다. 우선 인자가 없는 메소드의 기본적인 예제다.

```java
// 구독자가 땡겨갈때 3개의 값을 생산해내는 Flux를 준비한다.
Flux<Integer> ints = Flux.range(1, 3);

// 땡긴다.
ints.subscribe();
```

앞선 코드는 눈에 보이는 출력은 없지만, `Flux`는 실제로 동작하며 세개의 값을 만들어낸다.
만약 우리가 람다를 주입시켜 주면 우리는 눈에 보이는 결과를 만들 수 있다. 다음 예제는 생산한 값을 보여줄수 있도록 `subscribe` 메소드를 호출한다.

```java
Flux<Integer> ints = Flux.range(1,3);
ints.subscribe(i -> System.out.println(i));
```

위 코드는 다음과 같은 출력을 만든다 :
```
1
2
3
```

다음 메소드 시그니처를 설명하기 위해, 우리는 임의로 예외를 발생시킬 것이다. 다음 예제를 보자.

```java
// 4개의 값을 갖는 Flux 를 만들자.
Flux<Integer> ints = Flux.range(1,4)
// 특정 값을 다르게 핸들링 하기 위해 map 메소드를 사용한다.
  .map(i -> {
    // 대부분의 값은 그대로 반환하지만
    if (i <= 3) return i
    // 4일경우 예외를 발생한다.
    throw new RuntimeException("Got to 4");
  });

// 에러 핸들러와 함께 구독한다.
ints.subscribe(i -> System.out.println(i),
  error -> System.out.pringln("Error: " + error));
```

생산된 값을 구독하는 람다와, 예외가 발생했을때 이를 처리하는 람다, 총 두개의 람다로  `subscribe()` 를 작성했다. 출력은 다음과 같다.

```
1
2
3
Error: java.lang.RuntimeException: Got to 4
```

`subscribe`의 다음 메소드 시그니처는 에러 핸들러와 완료 이벤트를 처리할 핸들러 두가지를 갖는 시그니처다. 다음 예제를 보자

```java
Flux<Integer> ints = Flux.range(1,4);
ints.subscribe(i -> System.out.println(i),
  error -> System.out.println("Error: " + error),
  () -> {System.out.println("Done");}});
```

에러 신호와 완료 신호는 마지막 이벤트이면서 서로 상호 배타적이다. (동시에 존재할 수 없다)
소비하는 작업을 완료시키기 위해 우리는 반드시 에러 신호를 발생시키지 않도록 주의해야한다.
