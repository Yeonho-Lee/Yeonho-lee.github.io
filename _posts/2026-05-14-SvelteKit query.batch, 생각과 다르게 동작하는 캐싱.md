---
title: SvelteKit query.batch, 생각과 다르게 동작하는 캐싱
categories: [Web, SvelteKit]
tags: [sveltekit, svelte, remote-function, query-batch, tanstack-query]
date created: 2026-05-14
---

SvelteKit의 Remote Functions는 클라이언트-서버 통신을 타입 안전하게 처리해주는 실험적 기능이다. 그 중 `query.batch`는 N+1 문제를 해결해주는 유용한 도구다. 예를 들어 수업 목록을 렌더링하면서 수업마다 선생님 정보를 가져올 때, 개별 요청을 하나의 배치 요청으로 묶어준다.

서버 쪽(`data.remote.ts`)에서는 여러 인자를 한꺼번에 받아 처리하는 배치 함수를 정의하고:

```ts
// data.remote.ts
export const getTeacher = query.batch(getTeacherSchema, async (args) => {
  const teachers = await db.query.userTable.findMany({
    where: { id: { in: args.map((a) => a.teacherId) } },
  });
  const map = new Map(teachers.map((t) => [t.id, t]));
  return (arg) => map.get(arg.teacherId);
});
```

Svelte 컴포넌트에서는 각 row마다 **개별 함수처럼** 호출한다:

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import { getLessons, getTeacher } from './data.remote';

  const lessons = await getLessons();
</script>

{#each lessons as lesson}
  <tr>
    <td>{lesson.title}</td>
    <td>{(await getTeacher({ teacherId: lesson.teacherId }))?.name}</td>
  </tr>
{/each}
```

`getTeacher`를 row마다 따로 호출하고 있지만, SvelteKit이 같은 렌더링 사이클 안의 호출들을 감지해 **하나의 HTTP 요청으로 묶어서** 서버로 보낸다. 호출하는 쪽은 일반 함수와 똑같이 쓰면 된다.

---

## 기대했던 동작

레슨 일정 페이지에서 날짜를 변경하며 탐색할 때, 이렇게 기대했다.

> 2026-05-12를 조회하면 선생님·수업 정보가 캐싱된다. 다른 날짜로 갔다가 다시 2026-05-12로 돌아오면 → 캐시 hit.

## 실제 동작

직접 확인해보니 달랐다. 날짜를 바꿨다가 돌아오면 query.batch로 이루어진 함수들이 전부 재요청된다.

### 재현 예제 (MRE)

단순화를 위해 쇼핑몰 카테고리 예제를 만들었다.

```ts
// data.remote.ts
const PRODUCTS: Record<string, string[]> = {
  desktop: ["desktop-1", "desktop-2", "desktop-3"],
  laptop: ["laptop-1", "laptop-2", "2in1-1"],
  tablet: ["tablet-1", "2in1-1"],
};

export const getProducts = query(
  picklist(["desktop", "laptop", "tablet"]),
  async (category) => {
    return PRODUCTS[category];
  },
);

export const getProduct = query.batch(string(), async () => {
  const calledAt = Date.now();
  return (id) => ({ id, calledAt });
});
```

```svelte
<!-- desktop / laptop / tablet 버튼으로 category 상태를 바꿀 수 있다 -->
{#each await getProducts(category) as id}
  <tr>
    <td>{id}</td>
    {#await getProduct(id) then product}
      <td>{product.calledAt}</td>
    {/await}
  </tr>
{/each}
```

**재현 순서**: desktop 선택 → tablet 선택 → desktop 다시 선택

| 시점              | calledAt            |
| ----------------- | ------------------- |
| desktop 최초 로드 | `...817383`         |
| desktop 재방문    | `...834500` ← 새 값 |

`calledAt`이 바뀌었다. 캐시 miss다.

반면 laptop ↔ tablet 전환은 `2in1-1`의 `calledAt`이 유지된다 — 캐시 hit.

---

## 왜 이렇게 동작하는가

SvelteKit 공식 문서에는 이렇게 명시되어 있다.

> _"this instance is kept cached only as long as it is actively used on the page in a reactive context"_

핵심은 **reactive context**다. `{#each await getProducts(category) as id}` 블록에서 `category`가 바뀌면:

1. `getProducts('desktop')` 인스턴스는 더 이상 reactive context에서 참조되지 않는다
2. GC 대상이 된다 — `query.batch`로 만들어진 `getProduct('desktop-1..3')` 인스턴스도 마찬가지
3. `category`가 다시 `'desktop'`으로 바뀌어도 캐시는 이미 비워진 상태

tablet처럼 `desktop`과 겹치는 상품이 없으면 전환 즉시 모든 캐시가 증발한다. laptop ↔ tablet이 `2in1-1` 캐시를 유지할 수 있는 이유는, 두 카테고리 모두 항목이 있어서 `{#each}` 블록이 항상 `2in1-1`을 reactive context 안에 두기 때문이다.

결국 `query.batch`의 캐시 유효 범위는 "지금 이 순간 화면에 렌더링 중인 것"에 한정된다. 페이지에서 했던 모든 행동의 누적이 아니다.

---

## 비교: TanStack Query

TanStack Query는 `gcTime`(기본 5분)으로 이 문제를 해결한다. 컴포넌트가 사라지거나 reactive context를 벗어나도 일정 시간 동안 캐시를 유지한다.

Svelte용 TanStack Query(`@tanstack/svelte-query`)는 `QueryClientProvider` 컴포넌트 대신 `setQueryClientContext`로 클라이언트를 주입한다.

```svelte
<!-- +layout.svelte -->
<script lang="ts">
  import { onDestroy, onMount } from 'svelte';
  import { QueryClient, setQueryClientContext } from '@tanstack/svelte-query';

  let { children } = $props();

  const client = new QueryClient({
    defaultOptions: {
      queries: { staleTime: Infinity, gcTime: 5 * 60 * 1000 },
    },
  });

  setQueryClientContext(client);
  onMount(() => client.mount());
  onDestroy(() => client.unmount());
</script>

{@render children()}
```

`query.batch`처럼 아이템 단위 캐시를 만들려면 `queryClient.fetchQuery`를 활용한다. `createQueries`(여러 쿼리를 동시에 관리하는 API)는 Svelte 5의 `$state` 배열 프록시와 호환 문제가 있어서, 카테고리 쿼리의 `queryFn` 안에서 직접 아이템별 캐시를 채우는 방식을 쓴다.

```svelte
<!-- +page.svelte -->
<script lang="ts">
  import { createQuery, useQueryClient } from '@tanstack/svelte-query';

  const PRODUCTS: Record<string, string[]> = {
    desktop: ['desktop-1', 'desktop-2', 'desktop-3'],
    laptop:  ['laptop-1', 'laptop-2', '2in1-1'],
    tablet:  ['tablet-1', '2in1-1'],
  };

  async function fetchProduct(id: string) {
    await new Promise((r) => setTimeout(r, 50));
    return { id, calledAt: Date.now() };
  }

  let category = $state<'desktop' | 'laptop' | 'tablet'>('desktop');

  const queryClient = useQueryClient();

  // queryKey: ['product', id] 단위로 캐싱 → 2in1-1은 laptop/tablet이 공유
  const productsQuery = createQuery(() => ({
    queryKey: ['productList', category],
    queryFn: () =>
      Promise.all(
        PRODUCTS[category].map((id) =>
          queryClient.fetchQuery({
            queryKey: ['product', id],
            queryFn: () => fetchProduct(id),
            staleTime: Infinity,
          }),
        ),
      ),
  }));
</script>
```

`queryClient.fetchQuery`는 `staleTime` 안에 캐시가 있으면 `queryFn`을 다시 호출하지 않는다. 따라서 laptop → tablet 전환 시 `2in1-1`은 캐시 hit이고, desktop → tablet → desktop 재방문 시에도 각 아이템의 `calledAt`이 그대로 유지된다.

Remote Functions가 더 나은 부분도 있다 — 타입 안전성, 스키마 검증, `query.batch`의 자동 배칭은 TanStack Query에 없거나 직접 구현해야 한다. 캐시 수명 제어는 TanStack Query가 더 유연하다.

---

## 해결 방법

### 1. 더 넓은 범위를 한 번에 fetch하고 클라이언트에서 필터링

`days`가 바뀔 때 `getLiveLessons` 재요청을 막는다. `startDate`별로 캐시 키가 고정된다.

```ts
const allLessons = $derived(await getLiveLessons([startDate, 7]));
const visibleLessons = $derived.by(() => {
  const end = new Date(`${startDate}T00:00:00+09:00`);
  end.setUTCDate(end.getUTCDate() + days);
  return allLessons.filter((l) => new Date(l.startsAt) < end);
});
```

### 2. 누적 ID로 캐시 앵커 유지

조회했던 모든 teacher/class ID를 컴포넌트 수명 동안 유지하고, 해당 query 인스턴스를 reactive context에 붙잡아 둔다.

```svelte
<script lang="ts">
  let seenTeacherIds = $state<string[]>([]);
  let seenClassIds = $state<string[]>([]);

  $effect(() => {
    const newT = visibleLessons.map((l) => l.teacherId).filter((id) => !seenTeacherIds.includes(id));
    const newC = visibleLessons.map((l) => l.classId).filter((id) => !seenClassIds.includes(id));
    if (newT.length) seenTeacherIds = [...seenTeacherIds, ...newT];
    if (newC.length) seenClassIds = [...seenClassIds, ...newC];
  });

  // 이 derived들이 캐시를 살아있게 만드는 앵커 역할
  const _teachers = $derived(seenTeacherIds.map((id) => getTeacher({ teacherId: id })));
  const _classes  = $derived(seenClassIds.map((id) => getLiveLessonClass({ classId: id })));
  const _students = $derived(seenClassIds.map((id) => getLiveLessonStudentCenter({ classId: id })));
</script>
```

템플릿에서는 여전히 `getTeacher({ teacherId: lesson.teacherId })`를 호출하지만, `_teachers`가 해당 인스턴스를 살려두므로 항상 캐시 hit이 된다.

---

## 정리

| 동작                        | Remote Functions               | TanStack Query             |
| --------------------------- | ------------------------------ | -------------------------- |
| `category` 변경 후 재방문   | miss (reactive context 소멸)   | hit (gcTime으로 캐시 유지) |
| 두 카테고리에 공유된 아이템 | hit (context를 벗어나지 않음)  | hit (queryKey 단위로 공유) |
| 캐시 수명 제어              | 불가 (reactive context에 종속) | `gcTime`으로 조정 가능     |
| 자동 배칭                   | `query.batch`로 기본 제공      | 직접 구현 필요             |

`query.batch`의 캐싱은 "현재 렌더링 중인 것"에만 유효하다는 걸 이해하고 나면, 어디에 캐시 앵커를 놓아야 하는지가 보인다.

## Reference

Special Thanks to [@HyunbinSeo](https://github.com/hyunbinseo)
