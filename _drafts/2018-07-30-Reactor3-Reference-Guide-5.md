---
layout: post
title:  "[Reactor3 Reference Guide] 4. Reactor 핵심 특징 (2)"
date:   2018-07-30 00:02:05 +0900
tag: Reactor
---

원문 : [https://projectreactor.io/docs/core/release/reference/docs/index.html#core-features](https://projectreactor.io/docs/core/release/reference/docs/index.html#core-features)

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

* `IGNORE` 는 다운스트림의 역압 요청을 완전히 무시한다. 그래서 다운스트림의 대기열 큐가 가득 차게되면 `IllegalStateException`을 발생한다.
* `ERROR` 는 다운스트림을 더 이상 유지할 수 없을 때 `IllegalStateException`을 발생한다.
* `DROP` 은 다운스트림이 준비되지 않았으면 업스트림으로 부터의 신호를 버린다.
* `LATEST` 는 업스트림의 마지막 신호만을 다운스트림에 전달한다.
* `BUFFER` 는 기본값으로 다운스트림이 유지되지 않으면 모든 신호를 버퍼에 둔다. (버퍼의 제한이 없으므로 `OutOfMemoryError`로 이어질수있다.)

> `Mono` 에도 `create` 메소드가 존재한다. 이 메소드의 `MonoSink`는 다중값의 배출이 가능하지 않기에 첫번째 신호 이후의 것들은 전부 버려진다.

##### Push 모델

하나의 생산자에서 생기는 이벤트를 처리하는데 적합한 `create`의 형태는 `push`이다. `create`와 비슷하게 `push` 또한 비동기로 실행되고 `create`에 지원되는 전략들을 사용해
역압을 관리할 수 있다. 오로지 하나의 생산자 스레드만이 `next`, `complete`, `error` 메소드를 호출할 수 있다.

```java
Flux<String> bridge = Flux.push(sink -> {
  myEventProcessor.register(
    new SingleThreadEventListener<String>() {

      public void onDataChunk(List<String> chunk) {
        for (String s : chunk) {
          sink.next(s);
        }
      }

      public void processComplete() {
        sink.complete();
      }

      public void processError(Throwable e) {
        sink.error(e);
      }      
    });
});
```

##### Hybrid push/pull 모델

`push` 와는 다르게 `create`는 `push` 또는 `pull` 모드로 동작할 수 있다. 이는 데이터가 비동기로 들어오는 리스너 기반의 API로 부터 Flux나 Mono를 생성하는데 유용하다.
`onRequest` 콜백은 요청을 추적하기 위해 `FluxSink`에 등록할 수 있다.

이 콜백은 아마 소스로부터 데이터를 더 요청하기 위해 사용될 것이다. (만약 데이터를 전송하는 속도를 늦추어 역압을 관리해야 한다면, sink 로 요청을 보류한다.)
이것은 push 와 pull 모두 가능한 모델을 만든다. 다운스트림이 업스트림에서 데이터가 이미 준비 되었다면 당겨올 수 있게 하고, 데이터가 아직 준비되지 않았다면 준비되었을 때 다운스트림으로 밀어줄 수 있다.

```java
Flux<String> bridge = Flux.create(sink -> {
  myMessageProcessor.register(
    new MyMessageListener<String> () {
      public void onMessage(List<String> messages) {
        for (String s : messages) {
          sink.next(s); // 데이터가 준비되면 업스트림에서 밀어준다.
        }
      }
    });
  sink.onRequest(n -> {
    List<String> messages = myMessageProcessor.request(n);
    for (String s : messages) {
      sink.next(s); // 이미 데이터가 준비되어 있다면 다운스트림에서 당겨온다.
    }
  })
});
```

##### Cleaning up

두가지 콜백 `onDispose` 과 `onCancel` 은 사용한 리소스의 정리나, 종료나, 제거를 수행한다. `onDispose` 메소드는 `Flux` 가 완료되거나, 에러가 뱉어지거나, 취소되었을 때 정리하는 로직을 수행하는데 사용될 수 있다.

`onCancel` 메소드는 `onDispose` 메소드를 통해 정리하기 전에 취소와 관련된 특정 액션을 수행할 수 있다.

```java
Flux<String> bridge = Flux.create(sink -> {
  sink.onRequest(n -> channel.poll(n))
      .onCancel(() -> channel.cancel()) // cancel 신호에 의해서만 실행
      .onDispose(() -> channel.close()) // complete, error, cancel 전부 실행
});
```
### 4.4.3 Handle

`handle` 메소드는 조금 다르다. `Mono` 와 `Flux` 전부 사용할 수 있고, 인스턴스 메소드이기 때문에 이미 존재하는 소스에 체이닝 되어 사용될 수 있다. (다른 연산자들 처럼)

`handle` 메소드는 `SynchronousSink` 를 사용하고 오로지 1-1 방출만 가능하다는 점에서 `generate` 메소드와 비슷하지만 각 소스 요소에서 임의의 값을 생성하는데 사용될 수 있으며, 일부는 건너 뛸 수 있다. 이런 방법으로 `map` 과 `filter`의 조합으로 제공할 수 있다. 메소드의 시그니처는 다음과 같다.

`handle(BiConsumer<T, SynchronousSink<R>>)`

예제를 보도록 하자. Reactive streams의 명세는 시퀀스 내에 `null` 값을 허용하질 않는다. 만약 기존에 존재하는 맵핑 함수를 사용하여 `map` 메소드를 수행하고 싶을때 이 메소드가 종종 `null`을 리턴한다면 어떻게 해야할까?

예를 들어 다음 메소드는 정수형 소스에 안전하게 적용될 수 있다.

```java
public String alphabet(int letterNumber) {
  if (letterNumber < 1 || letterNumber > 26) {
    return null;
  }
  int letterIndexAscii = 'A' + letterNumber - 1;
  return "" + (char) letterIndexAscii;
}
```
우리는 `handle` 메소드를 모든 null 값을 징는데 사용할 수 있다.

```java
Flux<String> alphabet = Flux.just(-1, 30, 13, 9, 20)
  .handle((i, sink) -> {
    String letter = alphabet(i);
    if (letter != null) sink.next(letter);
  });

  alphabet.subscribe(System.out::println);
```

## 4.5 Schedulers

RxJava와 마찬가지로 Reactor 또한 동시성에 대해 잘 모르더라도 동시성 처리를 가능케 하는 도구~concurrency agnostic~ 로 고려될 수 있다.

that is, it dose not enforce a concurrency model. Rather it leaves you, the developer, in command. However, that does not prevent library from helping you with concurrency ..?

Reactor 에서 실행 모델과 어디서 실행될지는 `Scheduler` 에 의해서 결정된다.
[Scheduler](http://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Scheduler.html) 는 추상화된 인터페이스이고, [Schedulers](http://projectreactor.io/docs/core/release/api/reactor/core/scheduler/Schedulers.html) 클래스는 아래 실행 컨택스트에 대한 접근에 관련된 메소드를 제공한다.

> 이둘의 차이??

* 현재 스레드에서 실행 (`Schedulers.immediate()`).
* 재사용 가능한 싱글스레드에서 실행 (`Schedulers.single()`). 이 메소드는 모든 실행 호출자들이 하나만의 스레드를 해당 스레드가 종료될때까지 재사용한다는걸 명심해라. 만약 각각의 실행 호출자들마다 새로운 싱글스레드에서 실행되길 원한다면 `Schedulers.newSingle()` 메소드를 사용할 것.
* elastic 스레드 풀 (`Schedulers.elastic()`). 이 메소드는 필요하다면 새로운 워커 스레드를 생성하거나 유휴 스레드가 존재한다면 그 스레드를 재활용한다. 스레드가 너무 오랫동안 유 상태에 있다면 (기본 60초) 해당 스레드는 종료된다. 인스턴스의 I/O 블럭킹 작업을 처리하는데 유용하다. `Schedulers.elastic()` 메소드는 블록킹 작업을 별개의 스레드를 할당하여 다른 리소스를 이 작업에 얽매이지 않도록 손쉽게 만들 수 있다.
* 병렬처리를 위한 고정된 스레드풀을 사용 (`Schedulers.parallel()`) 이 메소드는 CPU 코어의 갯수만큼 워커 스레드를 생성한다.

> 레거시 블록킹 코드를 피할 수 없다면, `elastic` 은 해결책이 될 수 있지만 `single` 이나 `parallel`은 그럴 수 없다. 결론만 얘기하자면, 리액터의 블록킹 API들은 (`block()`, `blockFirst()`, `blockList()`, `toIterable()`, `toStream()`) 기본 single 이나 기본 parallel 스케줄러 에서 실행될 경우 `IllegalStateException` 을 던지게 되어있다.
> 사용자 정의 스케줄러는 `NonBlocking` 마커 인터페이스를 구현하는 스레드 인스턴스를 생성하는 방법으로 "논 블럭킹만 가능" 하다는 표현을 할 수 있다.

추가로, `Schedulers.fromExecutorService(ExecutorService)` 메소드를 사용하여 이미 존재하는 `ExecutorService` 로 부터 스케줄러를 생성할 수 있다.
또한 위의 여러 종류의 스케줄러를 `newXXX` 메소드를 사용하여 새로운 스케줄러를 생성할 수 있다. 예를들어 `Schedulers.newElastic(yourScheduleName)` 은 yourScheduleName 라는 이름을 갖는 새로운 elastic 스케줄러를 만든다.

> 논 블럭킹 알고리즘을 사용하여 구현된 연산자는 일부 스케줄러에서 진행죽인 작업을 훔치도록 조정한다.

몇몇 연산자는 기본적으로 `Schedulers`의 특정한 스케줄러를 사용하고 일반적으로는 사용자에게 다른 스케줄러를 사용할 수 있는 선택지를 준다.
예를들어 `Flux.interval(Duration.ofMillis(300))` 은 매 300ms 마다 값을 생산하는 `Flux<Long>` 를 만든다. 이 `Flux`는 기본적으로 `Schedulers.parallel()` 에서 생성되는데 다음과 같이 새로운 싱글 스레드 인스턴스를 가지고 생성하도록 바꿀 수 있다.

```java
Flux.interval(Duration.ofMillis(300), Schedulers.newSingle("test"));
```

리액터는 리액티브 체인에서 실행 컨텍스트 (혹은 스케줄러) 를 변경할 수 있는 `publishOn`, `subscribeOn` 두가지 수단을 제공한다.
둘다 `Scheduler` 를 인자로 받아 실행 컨텍스트를 해당 스케줄러로 변경해주지만, `publishOn` 은 메소드가 호출되는 지점 (라인) 이 중요한 반면에 `subscribeOn` 은 그렇지 않다. 이 둘의 차이를 이해하기 위해서는 우선 [구독하기 전에는 아무것도 일어나지 않는다는 것을 기억해야한다.](http://projectreactor.io/docs/core/release/reference/#reactive.subscribe)

리액터에서 연산자를 체이닝할때, 필요한 만큼 `Flux` 와 `Mono`의 구현체로 연산자 서로의 내부를 감쌀 수 있다 (?) 구독을 시작하면 체인의 `Subscriber` 객체가 생성되어 역으로 (체인의 맨 위로) 전달된다. 이것은 효과적으로 사용자로부터 감추어져있다. 사용자가 볼 수 있는 것은 `Flux` 혹은  `Mono` 와 `Subscription` 의 외부 계층이지만 이런 중간 연산자 구독자들은 실제로 작업이 일어나는 곳이다. (???)

 이러한 사전 지식을 통해 우리는 `publishOn` 과 `subscribeOn` 을 좀 더 자세히 볼 수 있다.

 * `publishOn` 은 구독 연산자들의 체인 중간에 다른 연산자들 처럼 적용한다. 이 연산자는 상위 스트림으로 부터 신호를 받아 주입받은 `Scheduler` 의 워커에서 콜백을 실행한다. 결론적으로 이 연산자는 이후의 연산자들의 실행에 영향을 끼친다 (이후 체인에 다른 `publishOn` 메소드를 만날 때 까지)

 * `subscribe` 은 구독 프로세스에 적용되어, 연산자 체인이 생성되는 시점까지 역으로 전달된다. 즉 어디에 `subscribeOn` 메소드를 명시하든 이 메소드는 항상 소스 방출의 전체적인 실행환경에 영향을 끼친다. 그러나 이 메소드는 이후 시퀀스가 동작하는 중 `publishOn` 메소드의 호출에 영향을 끼치지 못한다. `publishOn` 가 호출되면 이후의 실행환경은 `publishOn` 메소드에 의해 변경되어 수행된다.

 > 오로지 가장먼저 호출된 `subscribeOn` 만이 전체적인 실행환경에 영향을 끼친다.

## 4.6 Threading
