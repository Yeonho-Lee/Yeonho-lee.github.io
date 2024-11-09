---
title: SSR, CSR, SSG, ISR - Next.js 렌더링 방식의 이해와 구현
categories: Front-end Next.js
tags: ssr csr ssg isr
date created: 2024-10-26-03:42
date modified: 2024-11-09-17:54
---
## 개요
Next.js는 현대적인 웹 애플리케이션을 개발하기 위해 다양한 렌더링 방식을 지원한다. Next.js의 SSR(Server-Side Rendering), CSR(Client-Side Rendering), SSG(Static Site Generation), ISR(Incremental Static Regeneration) 기능을 활용하면 성능과 SEO를 고려한 최적의 웹 애플리케이션을 개발할 수 있다. 또한, Next.js 13에서 도입된 App Router는 기존 Page Router와 비교해 폴더 기반 라우팅, 서버/클라이언트 컴포넌트 구분 등을 통해 더 유연하고 효율적인 개발 환경을 제공한다.

## SSR, CSR이란?
렌더링은 어디서 실행되는지에 따라 두 가지 방식으로 나뉜다.
### SSR (Server-Side Rendering)
- 서버에서 미리 완성된 HTML을 만들어서 사용자에게 전달하는 방식
- 서버에서 렌더링 작업을 수행하므로 페이지 로딩이 빠르고, SEO(검색 엔진 최적화)에 유리

### CSR (Client-Side Rendering)
- 클라이언트(사용자 브라우저)에서 렌더링이 진행되는 방식
- 서버에서는 HTML의 기본 구조와 JavaScript를 보내고, JavaScript가 브라우저에서 실행되면서 화면이 완성됨
- React와 같은 프레임워크에서 CSR을 기본으로 사용함

요약하자면 SSR은 서버에서, CSR은 클라이언트에서 렌더링이 이루어지는 방식이다.


## 각 렌더링 방식에 대한 이해

Next.js의 13버전 이후의 **App Router**에서는 SSR, CSR, SSG, ISR과 같은 다양한 렌더링 방식을 지원한다. **Page Router** 방식과 차이는 있지만, 기본적인 렌더링 개념은 동일하다. 아래에 App Router를 기준으로 각 렌더링 방식을 설명해보려고 한다.

### 1. **SSR (Server-Side Rendering)**

**SSR**은 요청이 있을 때마다 서버에서 HTML을 생성하여 클라이언트로 보내주는 방식이다. Next.js의 App Router에서는 기본적으로 서버 컴포넌트를 사용해 SSR이 자연스럽게 적용된다.

#### SSR 적용 방법 (App Router):
**서버 컴포넌트**는 기본적으로 SSR로 처리된다. 특별히 추가 설정 없이도 `app/` 폴더 내의 컴포넌트를 정의하면, 요청 시마다 서버에서 렌더링이 이루어진다.
```js
// app/about/page.js
export default async function AboutPage() {
  const res = await fetch('https://api.example.com/data');
  const data = await res.json();

  return <div>{data.content}</div>;
}
```

이 코드에서 `AboutPage` 컴포넌트는 서버에서 렌더링되며, 클라이언트로 HTML이 반환됩니다.

### 2. **CSR (Client-Side Rendering)**

**CSR**은 클라이언트에서 JavaScript로 데이터를 패칭하고 렌더링하는 방식이다. **App Router**에서는 기본적으로 모든 컴포넌트가 **서버 컴포넌트**로 렌더링되므로, 클라이언트 측에서 동작하는 컴포넌트를 만들려면 **`use client`** 지시어를 사용하여 클라이언트 컴포넌트를 명시해야 한한다.

#### CSR 적용 방법 (App Router):
**클라이언트 컴포넌트**를 만들기 위해 `use client` 지시어를 사용하면 CSR이 적용된다.

```js
// app/about/page.js
"use client";

import { useState, useEffect } from 'react';

export default function AboutPage() {
  const [data, setData] = useState(null);

  useEffect(() => {
    async function fetchData() {
      const res = await fetch('https://api.example.com/data');
      const result = await res.json();
      setData(result);
    }
    fetchData();
  }, []);

  if (!data) return <div>Loading...</div>;

  return <div>{data.content}</div>;
}
```

위 코드에서 `use client` 지시어를 사용했기 때문에, 해당 페이지는 클라이언트에서 데이터를 가져와서 렌더링하며 CSR로 동작한다.

### 3. **SSG (Static Site Generation)**

**SSG**는 **빌드 시** HTML을 생성해 두는 방식이다. 이로 인해 요청 시 서버가 아닌 정적 파일을 서빙하므로 빠른 로딩과 높은 SEO 성능을 제공한다. App Router에서는 비동기 함수로 데이터를 패칭하면 정적 빌드를 통해 SSG로 자동 전환된다.

#### SSG 적용 방법 (App Router):
비동기 함수를 사용해 데이터를 패칭하면 빌드 시 정적 페이지가 생성된다.

```js
// app/about/page.js
export default async function AboutPage() {
  const res = await fetch('https://api.example.com/data');
  const data = await res.json();

  return <div>{data.content}</div>;
}
```

위 코드에서 `AboutPage`는 빌드 타임에 API로부터 데이터를 패칭하여 정적인 HTML 페이지로 생성된다. 이 페이지는 사용자 요청 시 이미 만들어진 HTML을 서빙하므로 성능이 매우 좋다.

### 4. **ISR (Incremental Static Regeneration)**

**ISR**은 정적으로 생성된 페이지를 일정 주기로 재생성하는 방식이다. SSG의 장점을 유지하면서도 일정 시간마다 페이지를 새로 생성하여 실시간 데이터를 제공할 수 있다. App Router에서는 `revalidate` 옵션을 사용해 설정할 수 있다.

#### ISR 적용 방법 (App Router):
`revalidate` 속성을 사용해 주기적으로 페이지를 재생성하도록 설정할 수 있다.

```js
// app/about/page.js
export default async function AboutPage() {
  const res = await fetch('https://api.example.com/data');
  const data = await res.json();

  return <div>{data.content}</div>;
}

export const revalidate = 10; // 10초마다 페이지를 다시 생성
```

위 코드에서는 `AboutPage`가 10초마다 재생성된다. 요청 시 10초가 경과했다면 최신 데이터를 기반으로 서버에서 페이지가 다시 생성되며, 이후 요청에서는 재생성된 페이지가 제공된다.

### 요약

- **SSR (Server-Side Rendering)**: 요청 시 서버에서 HTML을 동적으로 생성하여 클라이언트로 전달. 기본적으로 App Router에서 서버 컴포넌트는 SSR 방식으로 처리됨
- **CSR (Client-Side Rendering)**: 클라이언트에서 JavaScript로 데이터를 패칭하고 렌더링. `use client` 지시어로 클라이언트 컴포넌트를 명시해야 함
- **SSG (Static Site Generation)**: 빌드 시 HTML을 정적으로 생성하고, 이후 요청 시 캐싱된 정적 파일을 제공. 비동기 데이터 패칭을 사용하면 SSG로 작동
- **ISR (Incremental Static Regeneration)**: 정적 페이지(SSG Page)를 주기적으로 재생성하여 최신 상태를 유지. `revalidate` 옵션을 통해 ISR을 설정 가능

App Router에서는 이러한 렌더링 방식이 코드 작성 방식에 따라 자연스럽게 결정되며, 다양한 상황에 맞춰 최적화된 렌더링 방식을 사용할 수 있다.