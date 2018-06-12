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
