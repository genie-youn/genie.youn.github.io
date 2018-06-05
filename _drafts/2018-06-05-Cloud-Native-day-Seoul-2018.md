---
layout: post
title: "Cloud Native day Seoul 2018 후기"
date: 2018-06-05 00:02:05 +0900
tag: Conference
---

Pivotal이 한국에서 처음 `Cloud Native Day`라는 이름으로 세미나를 진행하였다. Spring의 리드그룹인 Pivotal이 어떤 방향성을 갖고 있는지 궁금하기도 했고, 혹시 향후에 Spring으로 MSA를 개발할 때 도움이 될까 싶어 참여하게 됐다.
히로쿠의 [The Twelve-Factor app](https://12factor.net/ko/) 같은걸 기대하고 갔다.

진행세션은 다음과 같다.

환영사 및 글로벌 트렌드 소개
클라우드 네이티브로의 전환을 위한 여정과 성공사례
넷플릭스 서비스와 피보탈
Spring Project와 Pivotal Cloud Foundry의 최신 업데이트
Google Cloud Platform을 통한 Pivotal Cloud Foundry의 확장
클라우드 네이티브 엔터프라이즈 사례 발표 (Case studies)


## 세션2. 클라우드 네이티브로의 전환을 위한 여정과 성공사례

클라우드 네이티브는 세가지로 구성된다고 할 수 있다. devOps, Cloud Native App(Microservice Architecture), Organazation / Process.

데브옵스

데브옵스는 애플리케이션과 서비스를 빠른 속도로 제공할 수 있도록 조직의 역량을 향상시키는 문화 철학, 방식 및 도구의 조합입니다. 기존의 소프트웨어 개발 및 인프라 관리 프로세스를 사용하는 조직보다 제품을 더 빠르게 혁신하고 개선할 수 있습니다. 이러한 빠른 속도를 통해 조직은 고객을 더 잘 지원하고 시장에서 좀 더 효과적으로 경쟁할 수 있습니다. - _[amazon](https://aws.amazon.com/ko/devops/what-is-devops/)_

이 말이 제일 인상깊었다. Optimizing time to value. 변화하는 요구사항에 맞춰 코드를 작성하고, 이 코드가 실제 사용자에게 도달하여 가치를 창출하는 시간을 줄여라. 그리고 그 사용자의 피드백을 빠르게 받아들여라.

이를 위해 빌드/배포 파이프라인을 구성하고 지속적통합, 지속적전달을 통해 빌드/배포를 자동화하여 빠르고 쉽게 만들것.
Platform팀은 개발자들이 비즈니스 로직을 코드로 구현하는데 집중하게 하기 위해 코드형 인프라/서비스를 제공할것.

클라우드 네이티브 앱
     마이크로서비스 아키텍처
     기존의 monolithic 시스템은 여러 단점이 있다. -> MSA가 기존 monolithic한 시스템의 문제들을 해결해 줄것이다.
     https://www.slideshare.net/saltynut/building-micro-service-architecture

     https://medium.com/coupang-tech/%ED%96%89%EB%B3%B5%EC%9D%84-%EC%B0%BE%EA%B8%B0-%EC%9C%84%ED%95%9C-%EC%9A%B0%EB%A6%AC%EC%9D%98-%EC%97%AC%EC%A0%95-94678fe9eb61


Organazation / Process
     XP를 해라..

그뒤로는 Pivotal이 위에것들을 도와줄 수 있다.. 하는 영업

세션3. 넷플릭스 MSA와 피보탈
    클라우네이티브자바 - 조쉬롱
    카프카 데이터플랫폼의 최강자

    넷플릭스 OSS
    configuration server - Archalus
    service discovery - Eureka
    circuit breaker - Hysrix
    api gateway - zuul
    load balacer - ribobn
    realtime monitoring - atlas / servo / spectator
    distributed tracing -
    zero downtime delivery - (CI/CD)spinnaker (canary)kayenta
    fault injection
    simain army
    evcache
    side car pattern

    유투브 혼돈의제왕을 검색해보래

세션4. 스프링프로젝트와 피보탈 파운더리
피보탈 application service에 관한 세션이다.

세션4. 구글 클라우드 플랫폼
PCF 가 GCP를 워크로드로 쓸꺼다. GCP이 최고다

세션5. Cloud Native Enterprise
USA airforce pivotal

Optimizing time to value

금융권 -> 기간계 코어뱅킹 시스템이 이럴 필요가 있을까?
그런데 대고객 서비스는 달라.

한 이터러블이 끝날때 피쳐가 동작하고 피드백을 받을 수 있어야 해
