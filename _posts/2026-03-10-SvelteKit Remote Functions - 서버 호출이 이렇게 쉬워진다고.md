---
title: SvelteKit Remote Functions - 서버 호출이 이렇게 쉬워진다고?
categories: [Web, SvelteKit]
tags: [sveltekit, svelte, remote-function, typescript]
date created: 2026-03-11-00:49
date modified: 2026-03-12-00:59
---

SvelteKit로 간단한 어드민 페이지를 만들다가 생각보다 파일이 너무 많아진다는 걸 느꼈다. 폼 하나 만드는데 스키마 파일, `+page.server.js`, `+page.svelte` 세 파일을 왔다 갔다 하면서 같은 검증 로직을 두 번 연결하고 있었다. 분명히 더 나은 방법이 있을 것 같아서 찾아보다가 Remote Function을 알게 됐다.

Remote Function은 서버에서 데이터를 읽거나, 폼을 제출하거나, 버튼 클릭으로 서버 작업을 실행하는 일들을 **로컬 함수를 호출하듯이** 쓸 수 있게 해주는 SvelteKit의 기능이다.

---

## Remote Function이란?

`.remote` 확장자를 가진 파일에서 `query`, `form`, `command`로 감싼 함수들을 말한다.

```
src/routes/blog/data.remote.ts
```

이 파일 안의 함수들은 **서버에서만 실행**되지만, 클라이언트에서 **그냥 import해서 호출**할 수 있다.

원래 서버 코드는 클라이언트에서 직접적으로 호출할 수 없지만, remote function을 사용하면 SvelteKit이 fetch 요청으로 변환해주기 때문에, 개발자 입장에선 그냥 로컬 함수를 호출하는 것처럼 쓸 수 있다.

이 개념 자체는 **RPC(Remote Procedure Call)**라는 이름으로 오래전부터 존재해왔다. RPC는 네트워크 너머에 있는 함수를 마치 로컬 함수인 것처럼 호출할 수 있게 해주는 패턴이다. 클라이언트가 직접 HTTP 요청을 구성하거나 URL을 신경 쓸 필요 없이, 함수 이름과 인자만으로 서버 로직을 실행할 수 있다는 게 핵심이다.

tRPC나 gRPC 같은 라이브러리가 이 패턴을 구현한 대표적인 예고, SvelteKit Remote Function도 같은 맥락에 있다. 다만 별도 라이브러리 없이 프레임워크 수준에서 지원한다는 점이 다르다.

---

## 세 가지 Remote Function

|함수|용도|
|---|---|
|`query`|데이터 읽기 (SELECT)|
|`form`|폼 제출 (HTML form 연동)|
|`command`|서버 작업 실행 (버튼 클릭 등)|

---

## 1. query — 데이터 읽기

### 기존 방식

`+page.server.js`에서 `load()`를 만들고, 컴포넌트에서 `$props()`로 받아야 했다.


```js
// src/routes/blog/+page.server.js
import * as db from '$lib/server/database';

export async function load() {
  const posts = await db.getPosts();
  return { posts }; // 반드시 return 해야 함
}
```
```svelte
<!-- src/routes/blog/+page.svelte -->
<script>
  let { data } = $props(); // load()의 return 값을 여기서 받음
</script>

{#each data.posts as post}
  <p>{post.title}</p>
{/each}
```

### Remote Function 방식


```ts
// src/routes/blog/data.remote.ts
import { query } from '$app/server';
import * as db from '$lib/server/database';

export const getPosts = query(async () => {
  return await db.getPosts();
});
```

> [!WARNING]
> `<script>` 블록에서의 top-level `await`는 현재 SvelteKit의 **실험적 기능**이다. 프로덕션 사용 시 주의가 필요하다.

```svelte
<!-- src/routes/blog/+page.svelte -->
<script>
  import { getPosts } from './data.remote';

  const posts = await getPosts();
</script>

{#each posts as post}
  <p>{post.title}</p>
{/each}
```

> [!NOTE]
> **실행 흐름 차이**
>
> 기존 방식에서는 `+page.server.js`의 `load()`가 먼저 완료된 뒤에 페이지 렌더링이 시작된다.
> Remote Function 방식에서는 컴포넌트 렌더링 도중 `await`를 만나면 그 지점에서 렌더링이 일시 중단되고, 비동기 작업이 끝나면 재개된다.
> 두 방식 모두 데이터를 받아오기 전까지 렌더링이 블로킹된다는 점은 동일하다.

`load()`, `return`, `$props()` 없이 **바로 호출**할 수 있다.

---

## 2. form — 폼 제출

### 기존 방식

`+page.server.js`에 `actions`를 따로 정의하고, 폼에서 `action` 속성으로 연결해야 했다.

유효성 검사도 서버와 클라이언트에 **각각 따로** 작성해야 했다.

```ts
// src/routes/admin/users/schema.ts
import * as v from 'valibot';

export const AddUserSchema = v.object({
  contact: v.pipe(v.string(), v.minLength(1, '연락처를 입력해주세요')),
  type: v.picklist(['email', 'phone'], '올바른 타입을 선택해주세요'),
});
```
```js
// src/routes/admin/users/+page.server.js
import * as v from 'valibot';
import { AddUserSchema } from './schema';

export const actions = {
  addUser: async ({ request, locals }) => {
    // 인증 체크를 매번 직접 작성해야 함
    if (!locals.session || locals.session.role !== 'admin') {
      redirect(303, '/login');
    }

    const formData = await request.formData();

    // Object.fromEntries(formData)의 타입은 Record<string, FormDataEntryValue>로,
    // FormDataEntryValue는 string | File의 유니온 타입이다.
    // 반면 AddUserSchema는 v.object()로 정의된 순수 객체 형태를 기대하므로,
    // 타입이 엄밀히 호환되지 않는다.
    // v.safeParse가 런타임에서 string을 처리해주긴 하지만,
    // TypeScript 타입 레벨에서는 불일치가 발생한다.
    const raw = Object.fromEntries(formData);

    // 서버에서 한 번 더 검사
    const result = v.safeParse(AddUserSchema, raw);
    if (!result.success) {
      return fail(400, { errors: v.flatten(result.issues) });
    }

    await db.insert(userTable).values(result.output);
  }
};
```
```svelte
<!-- src/routes/admin/users/+page.svelte -->
<script>
  let errors = {};

  // 클라이언트에서 또 한 번 검사
  // (서버와 동일한 스키마를 쓰더라도 검사 코드를 별도로 연결해야 한다)
  function handleSubmit(e) {
    const raw = Object.fromEntries(new FormData(e.target));
    const result = v.safeParse(AddUserSchema, raw);
    if (!result.success) {
      errors = v.flatten(result.issues);
      e.preventDefault();
    }
  }
</script>

<form method="POST" action="?/addUser" onsubmit={handleSubmit}>
  <input name="contact" />
  {#if errors.contact}<span>{errors.contact}</span>{/if}

  <input name="type" />
  {#if errors.type}<span>{errors.type}</span>{/if}

  <button type="submit">추가</button>
</form>
```

스키마를 정의해도 서버/클라이언트 양쪽에 검사 코드를 **따로 연결**해야 해서 번거롭다. 스키마를 공유하면 끝날 것 같은데, 실제로는 연결하는 코드가 계속 붙는다. 결국 파일 세 개를 동시에 열어두고 작업하게 된다.

### Remote Function 방식

`form(Schema, handler)` 형태로 스키마를 넘기면, **클라이언트 사이드 validation이 자동으로 적용**된다.
```ts
// src/routes/admin/users/schema.ts
import * as v from 'valibot';

export const AddUserSchema = v.object({
  contact: v.pipe(v.string(), v.minLength(1, '연락처를 입력해주세요')),
  type: v.picklist(['email', 'phone'], '올바른 타입을 선택해주세요'),
});
```
```ts
// src/routes/admin/users/data.remote.ts
import { form } from '$app/server';
import { AddUserSchema } from './schema';

export const addUser = form(AddUserSchema, async (data, issue) => {
  // 인증 체크를 한 줄로
  requireSession('admin');

  // data는 이미 스키마 검증이 끝난 상태
  // (FormData → 스키마 파싱 → 타입 안전한 객체로 변환까지 SvelteKit이 처리)
  const existing = await db.query.userTable.findFirst({
    where: { contact: data.contact, type: data.type },
  });

  if (existing) {
    invalid(issue.contact('이미 등록된 사용자입니다'));
  }

  const [user] = await db
    .insert(userTable)
    .values({ contact: data.contact, type: data.type })
    .returning({ id: userTable.id });

  return { id: user!.id };
});
```
```svelte
<!-- src/routes/admin/users/+page.svelte -->
<script>
  import { addUser } from './data.remote';
</script>

<!-- use:addUser 하나로 클라이언트 validation + 서버 제출 모두 처리 -->
<form use:addUser>
  <input name="contact" />
  <input name="type" />
  <button type="submit">추가</button>
</form>
```

> [!IMPORTANT]
> **클라이언트/서버 validation이 모두 실행되는 건 동일하다**
>
> Remote Function을 쓴다고 해서 validation을 한 번만 실행하는 것이 아니다.
> 제출 시 클라이언트에서 스키마 검사가 먼저 실행되고, 통과하면 서버에서 다시 검증된다.
> 달라지는 건 그 동작을 **직접 코드로 연결하지 않아도 된다**는 점이다.
> 스키마를 `form()`에 한 번만 넘기면, SvelteKit이 양쪽 검사를 알아서 연결해준다.

처음 봤을 때 `use:addUser` 한 줄짜리 폼이 실제로 동작한다는 게 믿기지 않았다. 에러 표시는 어디서 하나 싶었는데, 그것도 알아서 처리해준다.

---

## 3. command — 버튼 클릭으로 서버 작업 실행

폼 제출이 아니라, 버튼 클릭 같은 이벤트로 서버 작업을 실행할 때 쓴다.

### 기존 방식

API 엔드포인트를 따로 만들고, `fetch`로 직접 호출해야 했다.
```js
// src/routes/api/post/delete/+server.js
export async function POST({ request }) {
  const { id } = await request.json();
  await db.deletePost(id);
  return new Response('ok');
}
```
```svelte
<!-- src/routes/blog/+page.svelte -->
<script>
  async function handleDelete(id) {
    await fetch('/api/post/delete', {
      method: 'POST',
      body: JSON.stringify({ id }),
      headers: { 'Content-Type': 'application/json' }
    });
  }
</script>

<button onclick={() => handleDelete(post.id)}>삭제</button>
```

### Remote Function 방식

```ts
// src/routes/blog/data.remote.ts
import { command } from '$app/server';
import * as db from '$lib/server/database';

export const deletePost = command(async ({ id }: { id: number }) => {
  await db.deletePost(id);
});
```
```svelte
<!-- src/routes/blog/+page.svelte -->
<script>
  import { deletePost } from './data.remote';
</script>

<button onclick={() => deletePost({ id: post.id })}>삭제</button>
```

API 파일도 없고, `fetch`도 없고, URL도 없다.

---

## 정리

기존 방식은 서버 코드와 클라이언트 코드가 **여러 파일에 나뉘어져** 있어서 관리가 번거롭다.

Remote Function은 서버 로직을 `.remote` 파일 하나에 모아두고, 클라이언트에서 **그냥 import해서 쓸 수 있게** 해준다.

아직 experimental 딱지가 붙어 있어서 프로덕션에 바로 쓰기엔 조심스럽지만, 방향 자체는 맞다고 생각한다. 서버/클라이언트 경계를 신경 쓰는 게 아니라 기능 단위로 코드를 모아두는 게 훨씬 자연스럽다. stable이 되면 적극적으로 쓸 것 같다.

|           | 기존 방식     | Remote Function |
| --------- | ------------- | --------------- |
| 파일 수   | 여러 개       | `.remote` 하나  |
| URL       | 직접 입력     | 필요 없음       |
| JSON 변환 | 직접 해야 함  | 자동            |
| 코드 위치 | 여기저기 분산 | 한 곳에 모임    |

## Reference
[Remote Functions - SvelteKit](https://svelte.dev/docs/kit/remote-functions#form)