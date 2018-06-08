---
layout: post
title:  "[Reactor3 Reference Guide] 3. Reactive Programming의 소개(2)"
date:   2018-05-18 00:02:05 +0900
tag: Reactor
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
이와같이 전체적인 체인은 연결된다. 데이텉는 첫번째 Publisher로 부터 체인을 따라 내려가면 각각에 체인에 의해 변경된다. 결국에 Subscriber가 작업을 끝낸다.

### 3.3.4 `subscribe()` 가 호출되기 전엔 아무것도 일어나지 않는다.

Subscriber가 Publisher를 구독하기 전까지 아무일도 일어나지 않는다는것을 명심해라.

### 3.3.5 BackPressure

`backPressure`는 앞서 생산라인에 비교한다면 하위 작업장이 상위 작업장에 조금 더 천천히 보내 하고 신호를 주는것과 같다. 즉, upstream으로 신호를 전파하는것이다.

Reactive Streams Specification에 정의된 실제 메커니즘은 유추에 가깝다.
`Subscriber`는 `unbounded`로 동작하면서 소스가 모든 데이터를 푸쉬할수 있게 할 수도 있고, `Subscriber`가 n개의 요소를 처리할 준비가 되었을 때 소스에게 n개 만큼을 요청하는 방식으로도 동작할 수 있다.

중간연산자 (`Intermediate Operators`) 는 또한 요청을 운송중에 바꿀수 있다. 작업중 10개의 요소를 그룹화하는 버퍼 연산자가 있다고 가정해보자. 만약 Subscriber가 1개의 버퍼를 요청하면 이것은 소스에게 10개의 요소를 생산하라고 받아들여질 수 있다. 만약 요소가 필요하기 전에 생산되는게 효율적이지 않다면 Prefetching Strategies 또한 주어질 수 있다.

이것은 push model로 부터 하위작업장이 상위 작업장으로부터 그들이 처리가능할때 n개의 요소를 요청하고, 만약 요소가 준비되지 않았다면 요소가 생산되었을 때 상위작업장으로부터 push를 받을 수 도 있는 push-pull 하이브리드 모델로 변환한다.

### 3.3.6 Hot vs Cold

리액티브 라이브러에서는 리액티브 시퀀스를 `hot` 과 `cold` 두가지로 구분하낟. 이러한 구분은 리액티브 스트림이 `Subscriber`에게 어떻게 반응하는지와 주로 관계가 있다.
데이터 소스를 포함하여 `Cold Sequence`는 각각의 Subscriber에 대해 새로 시작된다. 만약 소스가 HTTP 요청으로 감싸져 있다면 각각의 구독자에 대하여 새로운 HTTP 요청이 생성된다.

`Hot Sequence`는 각각의 구독자에 대하여 아무것도 없는 상태부텉 시작하지 않는다. 오히려 이후의 구독자들은 그들이 구독된 이후에 방출되는 신호를 받는다. 그러나 일부의 `Hot Sequence`들은 방출의 전체나 일부 히스토리를 캐시해두거나 반복할 수 있다.

일반적인 관점에서 `Hot Sequence`는 구독하는 Subscriber가 없을지라도 방출할 수 있다. ("Noting happens before you subscribe" 조건의 예외이다.)
