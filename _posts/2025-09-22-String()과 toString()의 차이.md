---
title: String()과 toString()의 차이
categories: Javascript TypeConversion
tags: string() tostring() type-conversion global-function
date created: 2025-09-22-00:30
date modified: 2025-09-22-18:52
---


## 1. String()
```js
console.log(String(134)); // "134"

console.log(String(3.14)); // "3.14"

console.log(String(true)); // "true"

console.log(String(false)); // "false"

console.log(String([1, 2, 3])); // "1,2,3"

console.log(String({ a: 1, b: 2 })); // "[object Object]"

console.log(String(null)); // "null"

console.log(String(undefined)); // "undefined"
```
- 값을 문자열로 변환하는 **전역 함수**
- 어떤 타입의 값이든 인자로 받아 문자열로 바꿔줌, **안정적**
- `null`이나 `undefined`와 같은 '값이 없는' 인자를 받아도 오류를 발생시키지 않음

## 2. toString()
```js
console.log((134).toString()); // "134"

console.log((3.14).toString()); // "3.14"

console.log(true.toString()); // "true"

console.log(false.toString()); // "false"

console.log([1, 2, 3].toString()); // "1,2,3"

console.log({ a: 1, b: 2 }.toString()); // "[object Object]"

console.log(null.toString()); // The value 'null' cannot be used here.

console.log(undefined.toString()); // The value 'undefined' cannot be used here."
```
- 모든 객체가 상속받는 **메서드**
- 따라서 `null`이나 `undefined`처럼 객체가 아닌 값에는 이 메서드를 사용할 수 없음

## Reference
[String() 생성자 - MDN](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/String/String)
[Object.prototype.toString()](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Object/toString)
