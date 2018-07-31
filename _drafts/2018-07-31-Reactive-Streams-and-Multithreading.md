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
