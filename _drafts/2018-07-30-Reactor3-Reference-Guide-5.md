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

