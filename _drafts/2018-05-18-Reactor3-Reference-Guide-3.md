---
layout: post
title:  "[Reactor3 Reference Guide] 3. Reactive Programming의 소개(2)"
date:   2018-05-18 00:02:05 +0900
categories: Reactor
---

원문 : [https://projectreactor.io/docs/core/release/reference/docs/index.html#intro-reactive](https://projectreactor.io/docs/core/release/reference/docs/index.html#intro-reactive)

## 3.3 절차적 프로그래밍에서 리액티브 프로그래밍으로

리액터를 비롯한 리액티브 라이브러리들은 JVM 환경에서의 기존의 고전적인 비동기에 대한 접근방식의 단점을 해결하는것 뿐만 아니라 몇가지 새로운 관점에서도 관심을 가지고 있다.
* 조합가능하고 읽기 편리함
* 데이터를 하나의 흐름으로써 다루고, 이를 위한 풍부한 연산자의 제공
* 구독전까지 아무일도 일어나지 않음
* `consumer` 가 `producer`에게 배출 속도가 너무 높다고 신호를 보낼 수 있는 `BackPressure`
* 고수준, 고가치의 동시성 추상화

### 3.3.1 조합성과 가독성
`Composability` 는 여러 비동기 작업을 이전 작업의 결과를 다른 작업의 입력으로 사용하거나 fork-join 스타일로 여러 작업으로 나누고 고수준 시스템에서 재사용가능한 별개의 컴포넌트로 실행시키도록 조정하는 능력을 의미한다.

`Composability`는 `Readabbility`와 강하게  연관되어 있다. 비동기 작업이 늘어나고 복잡해질수록 코드를 작성하고 읽는 일은 더욱 어려워진다. 콜백 자체는 매우 심플하지만 콜백 내에서 콜백을 호출하며 콜맥 지옥을 만드는것을 보았다.

리액터는 풍부한 조합 옵션을 제공한다.

### 3.3.2 조립라인에 비유

리액티브 어플리케이션에서 데이터 처리는 조립라인을 통한 움직임으로 생각할 수 있다. Reactor는 컨테이너 벨트이자 작업장이다. 원재료는 다양한 변환과 중간 단계 또는 다른 중간단계의 조각을 합치는 큰 조립라인의 일부를 거칠 수 있다. 어떤 지점에 작은 문제나 막힘이 있더라도 (예를 들어 상품을 포장하는 데 불규칙하게 긴 시간이 걸린다던지) 고통받는 작업장은 위의 작업장에서 흘려보낼 원재료를 제한하라는 신호를 보낼 수 있다.

### 3.3.3 Operators 연산자
`Operator`는 위의 생산라인 비유에서 작업장에 해당된다. 각각의 연산들은 Publisher에게 행동을 더하고 이전단계의 `Publisher를 새로운 인스턴스로 감싼다?`
이와같ㅇ티 전체적인 체인은 연결된다. 데이텉는 첫번째 Publisher로 부터 체인을 따라 내려가면 각각에 체인에 의해 변경된다. 결국에 Subscriber가 작업을 끝낸다. Subscriber가 Publisher를 구독하기 전까지 아무일도 일어나지 않는다는것을 명심해라.
