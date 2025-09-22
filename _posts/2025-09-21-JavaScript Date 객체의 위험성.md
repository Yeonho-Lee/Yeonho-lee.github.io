---
title: JavaScript Date 객체의 위험성
categories: JavaScript Date
tags: javascript date utc frontend
date created: 2025-09-21-22:50
date modified: 2025-09-22-18:12
---
## **JavaScript `Date` 객체의 위험성

JavaScript에서 날짜와 시간을 다룰 때 흔히 사용하는 `Date` 객체는 직관적이지만, 때로는 예상치 못한 문제를 일으키기도 한다. 특히 **시간대(Time Zone)**와 **지역 설정(Locale)**의 영향을 크게 받기 때문에, 서버와 클라이언트 간에 일관된 시간을 다루는 작업에서는 위험할 수 있다.

---

## **`Date` 객체 메서드의 문제점 🚨
### Date.parse(): 로컬시간 vs UTC

```js
console.log(new Date(Date.parse('Sep 22, 2025')));
// Output: 2025-09-21T15:00:00.000Z
```

위와 같이 연월일을 포함한 문자열은 JavaScript 엔진이 사용자의 **로컬 시간**으로 인식한다. 따라서 한국에서 실행하면 대한민국 표준시(GMT+0900)의 자정에 맞춰 Date 객체를 생성한다. 콘솔에는 이 로컬 시간을 UTC로 변환한 시간대가 찍힌다.

```js
console.log(new Date(Date.parse('2025-09-22')));
// Output: 2025-09-22T00:00:00.000Z

```
반면에, **ISO 8601형식**(YYYY-MM-DD)의 문자열은 똑같이 시간대 정보가 없음에도 불구하고 UTC 기준으로 해석된다.

---
## 해결책
### 임시 해결책: UTC시간을 로컬 시간으로 보정

개발 환경이 한국에 국한되어 있고 서버와 클라이언트 모두 한국 시간을 기준으로 데이터를 주고받는 경우, UTC와 한국 표준시(KST)의 **9시간 차이**를 이용해 시간을 보정하는 임시 방법도 있다.

`new Date()`로 생성된 로컬 시간을 `9 * 60 * 60 * 1000` (9시간에 해당하는 밀리초) 만큼 더해 UTC로 변환하는 방식이다. 이렇게 변환된 시간은 `toISOString()`을 통해 데이터베이스에 저장하기만 하면 그대로 사용할 수 있는 시간이 된다.
```js
// 한국 표준시를 기준으로 생성된 시간
const nowInKST = Date.now();

// 9시간을 더해 UTC로 변환
const safeDateForDB = new Date(nowInKST + 9 * 60 * 60 * 1000).toISOString();

console.log(safeDateForDB);
// 2025-09-22T00:00:00.000Z
// (실제 시간과 관계없이 9시간을 더해 UTC 자정으로 맞춘 예시)

// DB에 저장하기 좋은 ISO 8601 포맷
```
이 방법은 단순하고 직관적이지만, **전 세계 사용자를 대상으로 하는 서비스에는 적합하지 않다.** 시간대가 한국으로 고정되어 있다는 전제하에는 사용하기 좋을 수 있다.


### 더 나은 해결책: UTC 기반 메서드 사용

이러한 문제를 방지하려면, **UTC(Coordinated Universal Time)** 기반의 메서드를 사용해야 한다.
```js
// 안전한 날짜 포맷팅 함수 (UTC 기반)
const getSafeMonthAndDay = (date) => {
  const month = date.getUTCMonth() + 1; // getMonth() 대신 getUTCMonth() 사용
  const day = date.getUTCDate(); // getDate() 대신 getUTCDate() 사용
  return `${month}월 ${day}일`;
};

// 한국에서 생성된 날짜
const safeDateInKorea = new Date('2025-09-22T00:00:00.000Z');
console.log(getSafeMonthAndDay(safeDateInKorea));
// 결과: "9월 22일"

// 미국에서 실행해도 동일한 결과를 보장
const safeDateInUSA = new Date('2025-09-22T00:00:00.000Z');
console.log(getSafeMonthAndDay(safeDateInUSA));
// 결과: "9월 22일"
```
`getUTCMonth()`와 `getUTCDate()` 같은 메서드는 실행되는 환경의 로컬 시간에 관계없이 항상 UTC를 기준으로 작동하므로, 서버와 클라이언트 간에 일관된 날짜를 보장할 수 있다.

## **결론: `Date` 객체, 현명하게 사용하는 법**

`Date` 객체는 직관적이지만, **로컬 타임존**이라는 함정이 있다. 로컬 시간을 기반으로 작동하는 메서드들은 전 세계 사용자를 대상으로 하는 서비스에서 예측 불가능한 버그를 일으킬 수 있다.

따라서 서버와 데이터를 주고받을 때는 `toISOString()`을 사용해 **UTC 기준 시간**을 저장하고, `getUTCMonth()`와 같은 **UTC 기반 메서드**를 사용해야 한다.

`Date` 객체는 화면에 **시간을 보여주는 용도**로만 사용하고, 데이터 처리와 저장에는 UTC를 사용하는 것이 **안전하고 견고한 애플리케이션을 만드는 가장 확실한 방법**이다.

## Reference
[Date - JavaScript | MDN](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Date)
[Java의 날짜와 시간 API](https://d2.naver.com/helloworld/645609)
