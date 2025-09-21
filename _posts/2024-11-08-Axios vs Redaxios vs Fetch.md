---
title: Axios vs Redaxios vs Fetch
categories: Backend Node.js
tags: axios redaxios fetch     # TAG names should always be lowercase
date created: 2024-10-26-09:04
date modified: 2024-11-11-18:05
---

# HTTP 요청 라이브러리 비교: Axios, Redaxios, Fetch

프로젝트에서 Node.js 환경에서의 크롤링을 위해 HTTP 요청 라이브러리를 사용하게 되었다.
특히 Node.js 18부터 Fetch API가 기본으로 내장되면서, 기존에 많이 쓰이던 Axios와 Redaxios, 그리고 Fetch의 차이점을 비교해봤다.

## **Axios**

- Node.js 18 이전에 주로 사용된 서드파티 라이브러리
- Promise 기반으로, `fetch`보다 간단한 API와 추가 기능(예: JSON 자동 변환, 요청/응답 인터셉터)을 제공함
- 한계: 사실상 fetch가 기본으로 내장된 Node.js 18이상에서는 필수적이지 않음

## **Redaxios**
    
- `fetch`가 Node.js에 내장되면서 등장한 `fetch` 기반 라이브러리
- `axios`와 비슷한 인터페이스를 제공하며, `fetch`와 호환됨
- 한계: 기능이 간소화되어 Axios의 고급기능(예: JSON 자동 변환, 인터셉터)은 제공하지 않음

## **Fetch**

- Node.js 18부터 기본적으로 지원
- 브라우저 환경에서도 사용되기 때문에 클라이언트-서버 코드 통합에 유리
- 한계: JSON 변환은 따로 해줘야 하며, 인터셉터 등의 고급 기능이 없음

## 요약

- Node.js 18 이상에서는 **fetch**가 내장되어 있으므로 **Redaxios**나 **fetch**를 추천
- **하지만** 복잡한 요청을 처리하거나 JSON 자동 변환이 필요하다면 **Axios**가 여전히 유용할 수 있음

[fetch 와 axios의 차이점 - 블로그](https://junghyeonsu.com/posts/fetch-vs-axios/)
