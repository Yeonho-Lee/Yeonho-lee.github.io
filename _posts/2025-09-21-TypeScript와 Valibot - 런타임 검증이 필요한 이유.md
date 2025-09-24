---
title: TypeScript와 Valibot - 런타임 검증이 필요한 이유
categories: TypeScript, Valibot
tags: valibot, runtime, typescript, validation
date created: 2025-09-21-22:10
date modified: 2025-09-21-22:49
---

## TypeScript와 런타임 검증

많은 개발자들이 "타입스크립트를 쓰면 런타임 오류가 사라질 것"이라고 기대하지만, TypeScript의 주요 역할은 **컴파일 단계에서 타입 관련 실수를 미리 알려주는 것**이다. 따라서 프로그램이 실행되는 시점(런타임)에 들어오는 **데이터의 형태까지 보장해주지는 않기 때문에** 이를 검증하는 단계가 필요한 경우들이 존재한다.

## TypeScript의 `as` 키워드와 그 위험성

TypeScript에는 개발자의 확신을 기반으로 타입을 강제로 지정하는 **`as`** 키워드가 있다. 이를 **타입 단언(Type Assertion)** 이라고 부른다. 이는 TypeScript가 return type을 정확히 추론하지 못할 때 사용된다.

```ts
const myCanvas = document.getElementById("main_canvas") as HTMLCanvasElement;
```

위의 예시에서 TypeScript는 `getElementById`가 `HTMLElement`의 한 종류를 반환한다는 사실만 알고 있지만, 개발자는 이 요소가 항상 `HTMLCanvasElement`라는 것을 알고 있다. 이럴 때 `as`를 사용해 TypeScript에게 타입을 알려주면 `myCanvas`에서 `getContext()` 같은 메서드를 타입 오류 없이 사용할 수 있다.
하지만 이 개발자의 확신이 틀렸을 경우, 예를 들어서 `main_canvas`가 `HTMLCanvasElement`가 아닌경우, 런타임 오류가 발생할 수 있다. 이 때문에 `as`키워드의 무분별한 사용이 위험한 것이다.

## Valibot의 등장: 런타임 데이터 검증의 필요성


여기서 필요한 것이 바로 **Valibot** 같은 **런타임 유효성 검사** 라이브러리다. Valibot는 프로그램이 실행되는 동안 데이터가 실제로 우리가 지정한 형식과 일치하는지 검증한다. TypeScript와 Valibot의 차이를 살펴보자.

### TypeScript

우리가 TypeScript를 사용해 다음과 같이 타입을 정의한다고 가정해보자.
```ts
// TypeScript가 예상하는 타입
type MyObject = {
  num: number;
};

// 런타임에 들어온 데이터
const myData = { num: 123, str: 'hello' };

// TypeScript는 str의 존재를 무시
const d: MyObject = myData;
```
TypeScript는 컴파일 시점에 `myData`를 `MyObject` 타입으로 간주한다. `MyObject`에 정의된 `num` 속성만 보고 `str` 속성은 없는 것으로 취급하는 것이다. 따라서 이대로 코드를 진행하면 예상치 못한 오류가 발생할 수 있다.

### Valibot

하지만 Valibot은 다르다.
```ts
import { number, parse, strictObject } from 'valibot';

// Valibot를 사용한 검증
const a = parse(
  strictObject({ num: number() }), // 엄격한 객체 스키마 정의
  { num: 123, str: 'hello' }, // 런타임에 들어온 데이터
);
```
이 코드에서 `parse` 함수는 데이터의 모든 속성을 철저히 검사한다. `num` 속성은 `number()` 스키마를 통과하지만, `strictObject`는 스키마에 정의되지 않은 `str` 속성이 포함된 것을 감지하고 즉시 오류를 발생시킨다. 이처럼 **Valibot는 런타임에 데이터의 '불순물'을 엄격하게 걸러내어, TypeScript가 보지 못하는 위험까지 막아준다.**

---

## 결론: TypeScript와 런타임 유효성 검사는 상호 보완 관계

**TypeScript**가 컴파일 타임에 코드의 **정확성**을 높여준다면, **Valibot**는 런타임에 **데이터의 신뢰성**을 보장해준다. 이 둘을 함께 사용해야만 외부로부터 오는 데이터를 안전하게 처리하고, 견고하며 예측 가능한 애플리케이션을 만들 수 있다.

**TypeScript가 제공하는 안전망**은 훌륭하지만, 그 울타리를 넘어오는 데이터까지 막아주지는 못한다는 점을 기억하자. 런타임 검증은 그 울타리 밖에서 들어오는 모든 위험을 막아주는 또 하나의 중요한 방패다.


## Reference
[Typescript - Type Assertion](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#type-assertions)
[Valibot](https://valibot.dev/)
