---
layout: post
title:  "[Reactor3 Reference Guide] 2. Getting Start"
date:   2018-05-10 00:02:05 +0900
categories: Reactor
---

원문 : [https://projectreactor.io/docs/core/release/reference/docs/index.html#about-doc](https://projectreactor.io/docs/core/release/reference/docs/index.html#about-doc)

# 2. Getting Start

## 2.1 Reactor 소개
Reactor란 `backpressure`를 관리하는 형태의 효율적인 수요관리를 갖춘 JVM을 위한 fully non-blocking reactive programming foundation 이다.
Reactor는 자바8의 함수형 API들을 포함하고, Reactive Extensions Specification의 광범위한 구현체인 구성가능한 비동기 시퀀스 API `Flux` 와 `Mono`를 제공한다.

리액터는 또한 IPC (non-blocking inter-process communication)를 `reactor-ipc` 라는 컴포넌트로 제공한다. Reactor IPC는 MSA에 최적화된 backpressure-ready network engine을 제공한다.

## 2.2 필요조건
Java 8, reactive-stream:1.0.2 이상

## 2.3 Getting Reactor
Reactor를 시작하는 가장 쉬운 방법은 `BOM` 을 사용하는것.
의존성을 추가할 때 버전을 생략해야지만 `BOM` 에 의해 관리된다.
