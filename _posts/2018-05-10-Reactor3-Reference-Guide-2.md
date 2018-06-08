---
layout: post
title:  "[Reactor3 Reference Guide] 3. Reactive Programming의 소개(1)"
date:   2018-05-10 00:02:05 +0900
tag: Reactor
---

원문 : [https://projectreactor.io/docs/core/release/reference/docs/index.html#intro-reactive](https://projectreactor.io/docs/core/release/reference/docs/index.html#intro-reactive)

# 3. Reactive Programming의 소개
`Reactor`는 리액티브 프로그래밍 패러다임의 구현체인데, 그럼 리액티브 프로그래밍은 뭘까?
> `Reactive Programming`은 데이터 스트림과 변경의 전파에 관심을 갖는 비동기 프로그래밍 패러다임이다.
리액티브 프로그래밍은 정적(배열) 또는 동적(이벤트 퍼블리셔)인 데이터 스트림을 기존의 프로그래밍 언어로 쉽게 표현할 수 있게 한다.

시작은 MS가 `Reactive Extension (Rx)` 라이브러리를 닷넷 진영에 만들었다. 이후 `RxJava`가 등장하였고 시간이 지나 `Reactive Stream` 명세를 만들게 된다. 이 명세는 이후 Java9에 `Flow API`로 통합된다.

객체지향 언어에선 Reactive Programming이 옵저버 디자인 패턴의 확장으로 표현되었다.
또한 이러한 리액티브 프로그래밍 패러다임을 구현한 라이브러리에서 `Iterable-Iterator` 쌍에 이중성이 있기 때문에 리액티브 스트림 패턴을 우리에게 익숙한 Iterator 디자인 패턴과 비교할 수 있다.
다만, 한가지 Iterator 패턴과 리액티브 스트림 패턴의 차이는 Iterator는 **pull** 에 기반하지만, 리액티브 스트림은 **push** 에 기반한다는 것이다.

 `iterator`패턴에서는 값에 접근하는 메소드가 오로지 `Iterable` 의 책임일지라도 언제 `next()` 를 통해 시퀀스의 다음 요소에 접근할지는 개발자에게 달려있기 때문에 명령형 프로그래밍 패턴이라 할 수 있다.

Reactive Streams 에선 `Iterable-next()` 와 같은 쌍이 `Publisher-Subscriber` 이다. 그러나 Subscriber 에게 새로 사용가능한 값을 알려주는 것은 Publisher 이다. 이러한 **push** 는 reactive의 핵심이다.
또한 push된 value에 적용되는 연산은 명령형에 비해 선언적이다. 개발자는 정확한 흐름제어를 설명하기보다 계산 로직을 표현한다.

value를 push할 뿐만아니라 error 핸들링과 완료시점 또한 잘 정의된 방법으로 지원한다.
`Publisher`는 새로운 value를 자신의 `Subscriber` 에게 `onNext()` 를 호출함으로써 push 할 수 있을 뿐만 아니라 `onError()` 를 통해 에러에 대한 신호나 `onComplete`를 통해 완료를 전달할 수 있다. 에러나 완료가 전달되면 시퀀스는 종료된다. 이러한 패턴은 매우 유연하다. 값이 없거나, 한 개 이거나, 여러개일지라도 (심지어 무한대로 늘어난다 할지라도) 이 패턴으로 녹여낼 수 있다.

우리가 비동기 리액티브 라이브러리가 왜 필요했는지 고민해보자.

## 3.1 Blocking은 낭비가 될 수 있다.

프로그램의 성능을 증가시키는 방법은 대체로 두가지가 존재한다. 병렬처리와 좀 더 효과적으로 개선하는것이다.

일반적으로 자바 개발자는 `blocking code`를 작성한다. 이러한 관행은 병목현상이 발생할 때 까지는 잘 작동한다. 이 시점이 오면 비슷한 `blocking code` 를 실행하는 스레드를 더 때려박는다. 그러나 이러한 scaling은 얼마 못가 경쟁 및 동시성 문제를 야기한다.

설상가상으로 blocking은 자원을 낭비한다. 자세히 보면 어느정도 지연이 있는 프로그램 특히 I/O나 database request, network call 같은 경우엔 스레드가 data를 기다리기 위해 idle 상태에 돌입하기 때문에 자원이 낭비된다. 그렇기 때문에 병렬처리가 모든걸 해결해 주진 않는다.

## 3.2 비동기성의 해결??

비동기적이고 논-블럭킹한 코드를 작성함으로써 동일한 자원을 가지고 여러 활성화된 task로 바꿔가며 수행할 수 있고, 비동기적인 작업이 완료되면 현재 프로세스로 돌아올 수 있다.
그렇다면 jvm에서 어떻게 비동기 코드를 구현할 수 있을까. java는 `Callbacks` 와 `Futures`를 제공한다.

CallBack : 콜백은 반환값은 갖지 않지만, 콜백 파라미터를 받아 (람다나 익명클래스) 비동기 작업이 끝나면 해당 콜백 파라미터를 실행하는 비동기 메소드이다.
Future : 비동기 메소드는 `Future<T>` 를 즉시 반환한다. 비동기 작업이 이 `T`를 계산하고, `Future`는 이 값에 접근하는걸 감싼다. 이 값은 즉시 사용할 수 없지만. 사용이 가능해 질때까지
기다릴 수 있다.

이러한 기술들은 몇가지 문제를 가지고 있다.
CallBack은 조합하기가 어렵고 코드를 읽고 유지보수하기 어렵게 만든다. (콜백지옥)
Future는 그나마 조금 낫지만 여전히 조합하기가 어렵다 자바8에 `CompletableFuture`가 추가되어 여러 Future를 조정하는것은 가능하지만 쉽지 않을 뿐더러 Future 객체가 `get()` 메소드를 호출함으로써 block 되는 상황으로 끝나기 쉬우며, multiple value와 고급 에러 핸들링에 대한 지원이 부족하다.
