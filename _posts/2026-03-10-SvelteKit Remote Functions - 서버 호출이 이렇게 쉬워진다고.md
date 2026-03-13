---
title: SvelteKit Remote Functions - 서버 호출이 이렇게 쉬워진다고?
categories: [Web, SvelteKit]
tags: [sveltekit, svelte, remote-function, typescript]
date created: 2026-03-11-00:49
date modified: 2026-03-14-00:08
---

SvelteKit을 쓰다 보면 서버에서 데이터를 읽거나, 폼을 제출하거나, 버튼 클릭으로 서버 작업을 실행하는 일이 자주 있다.

Remote Function은 이런 작업들을 **로컬 함수를 호출하듯이** 쓸 수 있게 해주는 SvelteKit의 기능이다.

---

## Remote Function이란?

`.remote` 확장자를 가진 파일에서 `query`, `form`, `command`로 감싼 함수들을 말한다.

```
src/routes/blog/data.remote.ts
```

이 파일 안의 함수들은 **서버에서만 실행**되지만, 클라이언트에서 **그냥 import해서 호출**할 수 있다.

원래 서버 코드는 클라이언트에서 직접적으로 호출할 수 없지만, remote function을 사용하면 SvelteKit이 fetch 요청으로 변환해주기 때문에, 개발자 입장에선 그냥 로컬 함수를 호출하는 것처럼 쓸 수 있다.

---

## 세 가지 Remote Function

| 함수      | 용도                          |
| --------- | ----------------------------- |
| `query`   | 데이터 읽기 (SELECT)          |
| `form`    | 폼 제출 (HTML form 연동)      |
| `command` | 서버 작업 실행 (버튼 클릭 등) |

---

## 1. query — 데이터 읽기

> `query` 함수는 서버에서 동적 데이터를 읽을 수 있게 해준다. (정적 데이터는 `prerender`를 고려하자.)

### 기존 방식

`+page.server.ts`에서 `load()`를 만들고, 컴포넌트에서 `$props()`로 받아야 했다.

```ts
// +page.server.ts
export async function load({ fetch }) {
  const response = await fetch(
    "https://jsonplaceholder.typicode.com/posts?_limit=5",
  );
  const posts = await response.json();
  return { posts }; // 반드시 return 해야 함
}
```

```svelte
<!-- +page.svelte -->
<script>
	let { data } = $props(); // load()의 return 값을 여기서 받음
</script>

{#each data.posts as post}
	<p>{post.title}</p>
{/each}
```

서버에서 데이터를 가져오려면 `load()` → `return` → `$props()`라는 세 단계를 반드시 거쳐야 한다. 데이터를 만드는 곳(`+page.server.ts`)과 쓰는 곳(`+page.svelte`)이 분리되어 있어서, 필드 하나 추가할 때도 양쪽을 오가며 수정해야 한다.

### Remote Function 방식

```ts
// data.remote.ts
import { query } from "$app/server";

export const getPosts = query(async () => {
  const response = await fetch(
    "https://jsonplaceholder.typicode.com/posts?_limit=5",
  );
  return await response.json();
});
```

```svelte
<!-- +page.svelte -->
<script>
	import { getPosts } from './data.remote';

	const posts = await getPosts(); // 그냥 바로 호출!
</script>

{#each posts as post}
	<p>{post.title}</p>
{/each}
```

`load()`, `return`, `$props()` 없이 **바로 호출**할 수 있다. 데이터를 정의하고 사용하는 흐름이 일반 함수 호출과 같아서, 파일을 오갈 필요가 없다.

---

## 2. command — 버튼 클릭으로 서버 작업 실행

> `command` 함수는 `form`과 마찬가지로 서버에 데이터를 쓸 수 있게 해준다. `form`과 다른 점은 특정 엘리먼트에 종속되지 않아서 어디서든 호출할 수 있다는 것이다.

폼 제출이 아니라, 버튼 클릭 같은 이벤트로 서버 작업을 실행할 때 쓴다.

### 기존 방식

버튼 클릭 하나에도 `<form>`으로 감싸고 form action을 정의해야 했다.

```ts
// +page.server.ts
import { fail } from "@sveltejs/kit";

export function load() {
  return { likes: 0 };
}

export const actions = {
  like: async ({ fetch }) => {
    const response = await fetch(
      "https://jsonplaceholder.typicode.com/posts/1",
      {
        method: "PATCH",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ likes: 1 }),
      },
    );

    if (!response.ok) {
      return fail(500, { message: "Failed to add like" });
    }

    return { success: true };
  },
};
```

```svelte
<!-- +page.svelte -->
<script>
	import { enhance } from '$app/forms';

	let { data } = $props();
	let likes = $derived(data.likes);
	let localLikes = $state(0);
</script>

<p>Likes: {likes + localLikes}</p>

<form method="POST" action="?/like" use:enhance={() => {
	return async ({ result }) => {
		if (result.type === 'success') {
			localLikes += 1;
		}
	};
}}>
	<button>Like</button>
</form>
```

좋아요 버튼 하나 만드는데 `<form method="POST">`, `action="?/like"`, `use:enhance`, `result.type` 분기까지 필요하다. 버튼인데 폼으로 감싸야 하는 것 자체가 어색하다.

### Remote Function 방식

```ts
// actions.remote.ts
import { command } from "$app/server";

export const addLike = command(async () => {
  const response = await fetch("https://jsonplaceholder.typicode.com/posts/1", {
    method: "PATCH",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ likes: 1 }),
  });

  if (!response.ok) {
    throw new Error("Failed to add like");
  }
});
```

```svelte
<!-- +page.svelte -->
<script>
	import { addLike } from './actions.remote';

	let likes = $state(0);
</script>

<p>Likes: {likes}</p>

<button onclick={async () => {
	await addLike();
	likes += 1;
}}>Like</button>
```

`<form>` 없이 `onclick`에서 바로 `await addLike()`를 호출한다. form action, `use:enhance`, `result.type` 분기가 전부 사라지고, 일반적인 이벤트 핸들러 패턴 그대로 서버 작업을 실행할 수 있다.

---

## 3. form — 폼 제출

> `form` 함수는 서버에 데이터를 쓰기 쉽게 해준다. 제출된 `FormData`에서 만들어진 `data`를 받는 콜백을 인자로 받고, `<form>` 엘리먼트에 spread할 수 있는 객체를 반환한다.

### 기존 방식

`+page.server.ts`에 `actions`를 정의하고, `formData`를 수동으로 파싱하고, 유효성 검사도 직접 작성해야 했다. 클라이언트 쪽에서는 HTML `required` 속성을 따로 추가해야 하고, 서버 응답을 받아 처리하는 코드도 별도로 필요하다.

```ts
// +page.server.ts
import { fail } from "@sveltejs/kit";

export const actions = {
  create: async ({ request, fetch }) => {
    const formData = await request.formData();
    const title = formData.get("title")?.toString();
    const body = formData.get("body")?.toString();

    if (!title || !body) {
      return fail(400, { message: "Title and body are required" });
    }

    const response = await fetch("https://jsonplaceholder.typicode.com/posts", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ title, body, userId: 1 }),
    });

    if (!response.ok) {
      return fail(500, { message: "Failed to create post" });
    }

    const post: { title: string; id: number } = await response.json();
    return { post };
  },
};
```

```svelte
<!-- +page.svelte -->
<script lang="ts">
	import { enhance } from '$app/forms';
	import { page } from '$app/state';

	interface Post {
		title: string;
		id: number;
	}

	let result: Post | null = $state(null);

	const form = page.form as { post: Post } | null;
	$effect(() => {
		if (form?.post) {
			result = form.post;
		}
	});
</script>

<h2>Create Post</h2>

<form method="POST" action="?/create" use:enhance>
	<label>
		Title
		<input name="title" type="text" required />
	</label>
	<label>
		Body
		<textarea name="body" required></textarea>
	</label>
	<button>Create</button>
</form>

{#if result}
	<p>Created: {result.title} (id: {result.id})</p>
{/if}
```

서버에서는 `formData.get()`으로 하나하나 꺼내서 `?.toString()`으로 변환하고, `if (!title || !body)` 같은 검사를 직접 작성해야 한다. 클라이언트에서는 HTML `required`로 같은 검증을 **또** 작성하고, 서버 응답을 받으려면 `page.form`에서 `$effect`로 꺼내야 한다.

### Remote Function 방식

```ts
// data.remote.ts
import * as v from "valibot";
import { form } from "$app/server";

export const createPost = form(
  v.object({
    title: v.pipe(v.string(), v.nonEmpty()),
    body: v.pipe(v.string(), v.nonEmpty()),
  }),
  async ({ title, body }) => {
    const response = await fetch("https://jsonplaceholder.typicode.com/posts", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ title, body, userId: 1 }),
    });

    if (!response.ok) {
      throw new Error("Failed to create post");
    }

    return await response.json();
  },
);
```

```svelte
<!-- +page.svelte -->
<script>
	import { createPost } from './data.remote';
	import * as v from 'valibot';

	const schema = v.object({
		title: v.pipe(v.string(), v.nonEmpty('Title is required')),
		body: v.pipe(v.string(), v.nonEmpty('Body is required'))
	});
</script>

<h2>Create Post</h2>

<form {...createPost.preflight(schema)}>
	<label>
		Title
		<input {...createPost.fields.title.as('text')} />
		{#each createPost.fields.title.issues() as issue}
			<p style="color: red">{issue.message}</p>
		{/each}
	</label>
	<label>
		Body
		<textarea {...createPost.fields.body.as('text')}></textarea>
		{#each createPost.fields.body.issues() as issue}
			<p style="color: red">{issue.message}</p>
		{/each}
	</label>
	<button>Create</button>
</form>
```

가장 큰 차이는 **validation 보일러플레이트가 사라진다**는 점이다.

기존 방식에서는 클라이언트와 서버의 유효성 검사를 따로따로 작성해야 했다:

|                     | Client-side                                       | Server-side                                              |
| ------------------- | ------------------------------------------------- | -------------------------------------------------------- |
| **기존 방식**       | HTML `required` 직접 추가                         | `formData` 수동 파싱 + `fail()`                          |
| **Remote `form()`** | `preflight(schema)` — 제출 전 클라이언트에서 검증 | `form(schema, ...)` — 같은 스키마로 서버에서도 자동 검증 |

remote `form()`은 **valibot 스키마 하나로 양쪽을 모두 커버**한다:

- `form(schema, handler)`: 서버에서 스키마 기반 자동 검증. `formData` 파싱, 타입 변환, 에러 처리가 한 번에 해결된다.
- `preflight(schema)`: 제출 전에 클라이언트에서 먼저 검증한다. 실패하면 서버 요청 자체를 보내지 않는다.
- `fields.title.issues()`: 필드별 에러 메시지를 바로 꺼내 쓸 수 있다.

---

## 정리

Remote Function은 서버 로직을 `.remote` 파일에 모아두고, 클라이언트에서 **그냥 import해서 쓸 수 있게** 해준다.

|                | 기존 방식                        | Remote Function         |
| -------------- | -------------------------------- | ----------------------- |
| 데이터 전달    | `load()` → `return` → `$props()` | `await fn()` 바로 호출  |
| mutation       | `<form>` + action 필수           | `onclick`에서 바로 호출 |
| validation     | 서버/클라이언트 따로 작성        | 스키마 하나로 양쪽 커버 |
| 서버 응답 처리 | `page.form`, `result.type` 분기  | 함수 반환값 그대로 사용 |

## Reference

[Remote Functions - SvelteKit](https://svelte.dev/docs/kit/remote-functions#form)
