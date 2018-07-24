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

## 4.1 0-N개 요소의 비동기 시퀀스 Flux

`Flux<T>`는 0부터 N개의 방출된 요소의 비동기 시퀀스를 나타내고, 완료 신호와 에러로 인해 종료될 수 있는 표준 `Publisher<T>` 이다. 그러므로, `Flux`가 가질 수 있는 값은 값과 완료신호와 에러이다. Reactive Streams 스펙에서 이러한 세가지 신호의 타입은 downstream 객체의 `onNext`, `onComplete`, `onError`를 호출한다.

이처럼 처리 가능한 신호가 많기때문에, `Flux`는 범용적인 반응형 타입이라 할 수 있다. 모든 이벤트는 (심지어 종료하는 이벤트일지라도) 선택적이라는걸 명심해라.
`onComplete` 이벤트가 존재하는데  `onNext`가 없다는건 비어있는 유한한 시퀀스를 의미한다. 이 시퀀스에 `onComplete` 를 지운다면 비어있는 무한한 시퀀스를 의미하게 된다. 비슷하게 무한한 시퀀스가 꼭 비어있을 필요는 없다. 예를들어 `Flux.interval(Duration)` 은 무한하고 주기적으로 Long의 값을 배출하는 `Flux<Long>`를 만들어낸다.


## 4.2 0 혹은 1개의 비동기 결과를 갖는 Mono

`Mono<T>` 는 보통 한개의 요소를 갖고, `onComplete` 나 `onError`로 종료될 수 있는 특별한 `Publisher<T>` 이다.

`Flux`에 제공되는 연산자중 일부만 제공되는데, 예를들면
combination operators can either ignore the right hand-side emissions and return another mono or emit values from both sides ?
후자의 경우, 양쪽에서 배출된 값은 `Flux`로 변경된다. 예를들어 `Mono#then(Mono)`는 다른 `Mono`를 반환하는 반면에 `Mono#concatWith(Publisher)`는 `Flux`를 리턴한다.

`Mono` 는 반환하는 값이 없이 오로지 완료의 의미를 갖는 비동기 작업(얘를들면 `Runnable` 같은)을 나타낼 수 있다는걸 명심해라.

## 4.3 Flux와 Mono를 만들고 구독하는 가장 쉬운 방법

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

완료 이벤트를 처리하는 핸들러는 `Runnable` 인터페이스의 파라밑터가 없는 `run` 메소드를 나타내기 때문에 비어있는 괄호의 람다로 표현된다.
앞선 코드는 다음과 같은 출력을 갖는다.

```
1
2
3
4
Done
```

마지막 `subscribe`의 메소드 시그니처는 임의의 `Subscriber`를 포함한다. 해당 내용은 다음 섹션에서 자세히 설명한다.

```java
SampleSubscriber<Integer> ss = new SampleSubscriber<>();
Flux<Integer> ints = Flux.range(1,4);
ints.subscribe(i -> System.out.println(i),
               error -> System.out.println("Error: " + error),
               () -> {System.out.println("Done");},
               s -> ss.request(100));
ints.subscribe(ss);
```

앞선 예제에서 우리는 임의의 `Subscriber`를 메소드의 마지막 인자로 넘겨주었다. 다음 예제에서 인자로 넘겨주었던 `Subscriber`의 가장 간단한 구현체를 보자.

```java
package io.projectreactor.samples;
import org.reactivestreams.Subscription;
import reactor.core.publisher.BaseSubscriber;

public class SampleSubscriber<T> extends BaseSubscriber<T> {
  public void hookOnSubscriber(Subscription subscription) {
    System.out.println("Subscribed");
    request(1);
  }

  public void hookOnNext(T value) {
    System.out.println(value);
    request(1);
  }
}
```

SampleSubscriber 클래스는 `BaseSubscriber` 클래스를 상속받고 있는데, 이 클래스는 Reactor 에서 사용자가 `Subscriber`를 정의할 때 권장하는 추상클래스이다.
이 클래스는 Subscriber의 행위를 조정할 수 있는 오버라이드 가능한 훅(hook)을 제공한다. 기본적으로 이 훅은 제한이 없는 요청을 발생시키고 `subscribe()` 메소드와 같이 동작한다.
그러나 `BaseSubscriber`를 상속받으면 임의로 요청의 양을 조절하는데 유용하다.

최소한 `hookOnSubscribe(Subscription subscription)` 과 `hookOnNext(T value)`는 구현해야 한다. 이 예제에서 `hookOnSubscribe` 메소드는 문자열을 출력하고, 첫번째 요청을 만들어낸다. 그런후 `hookOnNext` 메소드는 현재의 값을 출력하고 다음 요청을 만들어낸다.

`SimpleSubscriber` 클래스는 다음과 같은 출력을 만든다.
```
Subscribed
1
2
3
4
```
> `hookOnError`, `hookOnCancel`,`hookOnComplete`, `hookFinally` 메소드 또한 구현할 수 있다.

Reactive Streams 스펙은 또 다른 형태의 `subscribe` 메소드를 정의한다. 다음과 같이 임의의 Subscriber를 별다른 옵션 없이 받는다.

```Java
subscribe(Subscriber<? super T> subscriber);
```

이러한 형태의 subscribe 메소드는 이미 유용한 `Subscriber`를 가지고 있을때 편리하다.
하지만 그것보다 더 이 메소드는 다른 콜백과 관련된 구독 작업을 하기위해 필요하다. 아마도 당신은 backpressure를 처리하고 스스로 요청을 실행해야하기 때문이다.

이 상황에서, backpressure를 핸들링하기 위한 유용한 메소드를 제공하는 `BaseSubscriber` 추상 클래스를 사용한다면, 쉽게 문제를 해결할 수 있다.

**backpressure를 조정하기 위한 BaseSubscriber의 사용**
```java
Flux<String> source = someStringSource();

source.map(String::toUpperCase)
      .subscribe(new BaseSubscriber<String> () {
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
          request(1);
        }

        @Override
        protected void hookOnNext(String value) {
          request(1);
        }
      });
```

## 4.4 프로그램으로 시퀀스 만들기

이번 섹션에서는 `onNext`, `onError`, `onComplete`를 프로그램적으로 정의하여 `Flux`와 `Mono`를 생성하는 방법을 소개한다.
이러한 메소드들은 우리가 `sink`라고 부르는 이벤트를 발생시키기 위한 API를 공통적으로 갖고 있다. 몇가지 형태의 메소드를 알아보자.


### 4.4.1 생성 (Generate)

프로그램적으로 `Flux`를 생성하는 가장 쉬운 방법은 생성 함수를 받는 `generate` 메소드를 사용하는 것이다.

이 메소드는 `synchronous` 처리방식과 `one-by-one` 방출, 즉 `SynchronousSink` 와 한번의 콜백에 한번의 `next()` 메소드가 호출되는것을 의미한다.
또한 부가적으로 `error(Throwable)` 과 `complete()` 메소드를 호출 할 수 있다.

가장 유용한 형태는 `sink`를 참조하여 사용할 수 있는 상태를 유지하며 다음에 방출할 것을 결정하는 것이다.
생성 함수는 `BiFunction<S, SynchronousSink<T>, S>`의 형태를 가지며 `<S>`는 상태 객체의 타입이다. 또한 상태의 초기값을 제공하는 `Supplier<S>` 를 제공해야하며, 생성함수는 각 스텝마다 새로운 상태를 반환해야한다.

`int`의 상태를 갖는 예를 보자.

```java
Flux<String> flux = Flux.generate(
  () -> 0,
  (state, sink) -> {
    sink.next(" 3 x " + state + " = " + 3*state);
    if (state == 10) sink.complete();
    return state + 1;
  });
```

또한 mutable한 <S>를 사용할 수 있다.

```java
Flux<String> flux = Flux.generate(
  AtomicLong::new,
  (state, sink) -> {
    long i = state.getAndIncrement();
    sink.next(" 3 x " + i + " = " + 3*i);
    if (i == 10) sink.complete();
    return state;
  });
```

또한 마지막 상태 객체를 정리하고 싶다면 `generate(Supplier<S>, BiFunction, Consumer<S>)`를 사용하면 된다.

```java
Flux<String> flux = Flux.generate(
  AtomicLong::new,
  (state, sink) -> {
    long i = state.getAndIncrement();
    sink.next(" 3 x " + i + " = " + 3*i);
    if (i == 10) sink.complete();
    return state;
  }, (state) -> System.out.println("state :" + state));
```

### 4.4.2 생성 (Create)

`Flux`의 좀 더 프로그램틱한 생성방법은 `create` 메소드를 사용하는 것이다. 이 메소드는 비동기/동기 모두 가능하고, 한번에 여러 값을 방출하는데 적합하다.
이 메소드는 `next`, `error`, `complete` 메소드를 갖는 `FluxSink` 를 사용하는데, `generate` 와는 대조적으로 상태 기반의 변수를 갖지 않는다. 이는 콜백에서 여러
이벤트를 트리거 할 수 있다는 것을 의미한다 (심지어 다른 스레드에서도) (?)

> 또한 `create` 메소드는 기존의 API를 reactive 하게 바꾸는데 유용합니다.

리스너 기반의 API가 있다고 가정해보자. 이 API는 데이터를 묶음으로 처리하면서 두개의 이벤트를 갖고 있는데, 하나는 데이터의 묶음이 준비되었다는 이벤틑와, 처리가 완료 되었다는 이벤ㅌ트이다.

```java
interface MyEventListener<T> {
  void onDataChunk(List<T> chunk);
  void processComplete();
}
```

이때 `create` 메소드를 사용해서 `Flux<T>` 로 바꿀 수 있다.
```java
Flux<String> bridge = Flux.create(sink -> {
  myEventProcessor.register(
    new MyEventListener<String>() {
      public void onDataChunk(List<String> chunk) {
        for (String s : chunk) {
          sink.next(s);
        }
      }

      public void processComplete() {
        sink.complete();
      }
    }
  );
});
```

또한 `create` 는 비동기로 동작하고, `OverflowStrategy`를 재 정의 하여 역압이 어떻게 동작할지를 관리할 수 있다.
