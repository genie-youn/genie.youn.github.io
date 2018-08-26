---
layout: post
title: "Reactive Streams and Multithreading"
date: 2018-07-31 00:02:05 +0900
tag: Reactor
---

원문 : [https://itnext.io/reactive-streams-and-multithreading-efd63b67de9a](https://itnext.io/reactive-streams-and-multithreading-efd63b67de9a)

Reactive Streams는 논-블럭킹에다가 비동기적이고, 멀티스레드에, 빠르고, 커피도 내리고…
물론 커피를 내려주진 않습니다. (ㅎㅎ) 그러나 다른 특징들은 reactive streams라고 하면 자주 언급되곤 합니다.
그러나 개발자들이 처음 reactive streams를 시도할때 자주 블럭킹, 동기화, 싱글스레드의 문제와 마주치게 됩니다.
그렇다면 reactive streams는 거품이 낀걸까요? 한번 확인해보겠습니다.

# The blocking stream

`Spring Reactor`와 `Java` 로 되어있는 자주 마주칠 수 있는 간단한 예제를 보겠습니다.

```java
source = Flux.just(1,2);
source
   .map(i -> i + ". element: " + i)
   .subscribe(System.out::println);

System.out.println("Before or after?");
// Output:
// 1. element: 1
// 2. element: 2
// Before or after?
```

위 예제는 동기로 동작하는것 처럼 보입니다.
왜 이렇게 동작하는지 이해하기 위해서는 reactive streams가 어떻게 동작하는지를 이해해야 합니다.
첫번째 라인은 처리할 기초 데이터의 스트림을 만듭니다. 이 경우 배열을 가지고 만듭니다.
두번째 라인은 맵핑 연산자를 추가하고 이 연산자에 의해 맵핑된 다른 값으로 변환된 스트림을 만듭니다.
아마 구독하기 전까진 아무런 일도 일어나지 않는 다는 사실을 알고 있을겁니다.
이 스트림을 구독할 때 다음과 같은 일이 벌어집니다.

* `subscribe` 메소드는 맵핑된 결과의 스트림이 준비되면 값을 출력합니다.
* 맵핑된 스트림은 상위 스트림에게 구독 요청을 전달합니다.
* 처음 생성했던 기초 데이터 스트림은 상위 스트림을 가지고 있지 않으며 즉시 완료될 때까지 값을 방출합니다.

모든 작업은 한 스레드에서 일어납니다. 이것이 마지막 출력문이 스트림이 모든 처리를 끝내고 실행되는 이유입니다.

# Let's go async

기초 데이터 스트림을 타이머로 변경해봅시다.

```java
source = Flux.interval(Duration.ofSeconds(1))
  .take(2);

source
  .map(i -> i + ". element: " + i)
  .subscribe(System.out::println);

System.out.println("Before or after?");

// Output:
// Before or after?
// 0. element: 0
// 1. element: 1
```

우리는 이 스트림이 비동기로 동작하는걸 볼 수 있습니다.
왜냐구요? 기초 데이터 스트림이 값을 즉시 배출하지 않기 때문입니다.
이로써 기초 데이터 소스가 스트림이 동기로 동작할지 비동기로 동작할지 결정한다는 것을 명확히 볼 수 있습니다.

뿐만아니라 동기 혹은 비동기로 동작할지는 스트림의 스레딩관련 처리에 의해 영향을 받을 수 있습니다.
다음과 같이 예제를 변경해보겠습니다.

```java
source = Flux.<Integer>create(emitter -> {
    System.out.println(Thread.currentThread().getName());
    emitter.next(1);
    emitter.complete();
});

source
   .map(i -> i + ". element: " + i)
   .subscribe(System.out::println);

System.out.println("Before or after?");

// Output:
// main
// 1. element: 1
// Before or after?
```

여기서 우리는 기초 데이터 스트림의 연산자가 메인 스레드에서 수행되는걸 볼 수 있습니다. 이제 `publishOn` 메소드를 기초 데이터 스트림 이후에 추가하고 코드를 수행해보겠습니다.

```java
source = Flux.<Integer>create(emitter -> {
    System.out.println(Thread.currentThread().getName());
    emitter.next(1);
    emitter.complete();
});

source.publishOn(Schedulers.single());

source
   .map(i -> i + ". element: " + i)
   .subscribe(System.out::println);

System.out.println("Before or after?");

//Output:
// main
// Before or after?
// 1. element: 1
```

처음은 이전 예제와 완전히 똑같지만, `publishOn` 메소드는 이후에 실행되는 모든 연산자를 다른 스레드에서 실행되게끔 보장합니다.
그래서 마지막 출력문이 먼저 찍히게 되죠.

이제 구독전에 `subscribeOn` 메소드를 추가해서 어떻게 다르게 동작하는지 확인해봅시다.

```java
source = Flux.<Integer>create(emitter -> {
    System.out.println(Thread.currentThread().getName());
    emitter.next(1);
    emitter.complete();
});

source.publishOn(Schedulers.single());

source
   .map(i -> i + ". element: " + i)
   .subscribeOn(Schedulers.single())
   .subscribe(System.out::println);

System.out.println("Before or after?");

// Output:
// Before or after?
// single-1
// 1. element: 1
```

As we can see from the output even the subscription process now runs on another thread.
출력에서 볼 수 있드시 구독의 모든 프로세스가 다른 스레드에서 수행됩니다. (값을 만들어내는 과정까지)
이제 우리는 실제로 비동기적이고, 논블럭킹으로 스트림을 동작하게 할 수 있습니다.

# Conclusion

리액티브 스트림이 항상 비동기로 동작하는건 아닙니다.
스레딩관련 처리는 기초 데이터 소스에 따라 다르며, 연산자에도 영향을 받을 수 있습니다.
리액티브 스트림의 장점은 같은 코드 스타일을 유지하면서도 비동기/논블럭킹의 개발을 할 수 있다는 것입니다. 우리는 동일한 프로그래밍 모델과 동일한 파이프라인을 동기적으로 실행되는 스트림에도, 비동기적으로 동작하는 스트림에도 사용할 수 있습니다.
