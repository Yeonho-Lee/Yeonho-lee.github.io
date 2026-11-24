---
title: DB JSON 데이터 버전 관리 - 객체와 Discriminated Union으로 타입 좁히기
categories: [Web, TypeScript]
tags: [typescript, type-narrowing, discriminated-union, drizzle]
date created: 2026-11-20
---

이전에 프로젝트 기획을 하면서 저장 형식이 정책에 따라 바뀔 수 있는 데이터가 있었는데, 이걸 버전으로 관리하기로 결정했다. 이때 버전을 DB에 저장은 해야 하니까 별도 column으로 둬버리고 잊고 있었다. 심지어 `data`는 `text({ mode: 'json' })`, type은 `string[]`으로 저장했다.

실제 데이터 대신 회고 기록을 예로 들어보겠다. 처음에는 **KPT** 방식으로 회고를 쓰다가, 나중에 **4L** 방식으로 바꿨다고 해보자.

- **KPT (v1):** Keep(좋았던 점), Problem(문제점), Try(다음에 시도할 것)
- **4L (v2):** Liked(좋았던 점), Learned(배운 점), Lacked(부족했던 점), Longed for(바랐던 점)

처음 만든 스키마는 이런 식이었다.

```ts
// schema.ts
type RetroVersion = 1 | 2;

export const retrospective = sqliteTable('retrospective', {
	id: integer().primaryKey({ autoIncrement: true }),
	version: integer().$type<RetroVersion>().notNull(),
	// v1 (KPT): [0] Keep, [1] Problem, [2] Try
	// v2 (4L):  [0] Liked, [1] Learned, [2] Lacked, [3] Longed for
	data: text({ mode: 'json' }).notNull().$type<string[]>()
});
```

| id  | version | data                                                                               |
| --- | ------- | ---------------------------------------------------------------------------------- |
| 1   | 1       | `["PR 리뷰를 하루 안에 끝냈다", "배포 전에 테스트를 못 돌렸다", "배포 체크리스트 만들기"]`                         |
| 2   | 2       | `["페어 프로그래밍이 재밌었다", "Drizzle 마이그레이션 흐름", "회의가 길어서 개발 시간이 부족했다", "회의는 30분 안에 끝내기"]` |

그런데 이번에 리팩토링을 하면서 두 가지 문제가 드러났다.

## 문제 1: `string[]`은 각 자리에 뭐가 들어 있는지 모른다

```ts
const problem = row.data[1];
// v1이면 Problem, v2면 Learned. 둘 다 string이라 컴파일러는 모른다
```

각 인덱스의 뜻은 주석이나 코드를 짠 사람의 머릿속에만 있고, TypeScript는 알 수 없다. 그래서 v2 데이터를 KPT로 읽어도 컴파일 에러 없이 Problem 자리에 Learned가 그대로 표시된다.

### 해결: `string[]` 대신 객체로 바꾼다

```ts
// schema.ts
type KptData = { keep: string; problem: string; try: string };
type FourLData = { liked: string; learned: string; lacked: string; longedFor: string };

export const retrospective = sqliteTable('retrospective', {
	// ...
	data: text({ mode: 'json' }).notNull().$type<KptData | FourLData>()
});
```

이렇게 바꾸면 내용을 이제 `data.problem`처럼 이름으로 꺼내기 때문에 코드에서 어떤 걸 사용하는지가 명시적이고, v2 데이터를 KPT로 읽으려 하면 컴파일 에러가 난다.

> **cf.** 이름 붙은 튜플(Labeled Tuple) `[keep: string, problem: string, try: string]`도 있지만, 이름은 type에만 붙고 접근은 여전히 `data[1]`이라 코드에 드러나지 않는다.

## 문제 2: `version`이 별개의 column이라 type이 좁혀지지 않는다

문제 1을 고쳐서 버전마다 type이 달라지자 보인 문제다. `version`과 `data`가 따로따로 추론돼서, 둘이 연결되어 있다는 정보가 type에 없다.

```ts
// 조회한 row type
// { version: RetroVersion; data: KptData | FourLData }

if (row.version === 1) {
	row.data.problem; // ❌ 에러: FourLData에는 problem이 없다
	const data = row.data as KptData; // 결국 as로 단언
	data.problem; // ✅
}
```

따라서 `data`의 type을 좁혀서 써주려면 위와 같이 `as`로 단언해서 써야 한다.

하지만 이 코드에는 문제가 있다. `as`는 검사를 건너뛰는 거라 잘못 단언해도 컴파일러가 잡지 못한다. 문제는 런타임에 가서야 드러난다.
예를 들어 `KptData | FourLData`를 `KptData`로 단언하면, 실제로는 `FourLData`여도 컴파일 에러가 안 난다. 둘 중 하나를 고르는 단언이라 TS는 그냥 믿어준다. 그러면 `data.problem`은 `undefined`가 되고, `data.problem.trim()`처럼 메서드를 부르는 순간 `TypeError`가 난다.

### 해결: `data` 안에 `version`을 넣는다 (Discriminated Union)

```ts
// schema.ts
type Retro =
	| { version: 1; keep: string; problem: string; try: string }
	| { version: 2; liked: string; learned: string; lacked: string; longedFor: string };

export const retrospective = sqliteTable('retrospective', {
	id: integer().primaryKey({ autoIncrement: true }),
	data: text({ mode: 'json' }).notNull().$type<Retro>()
});
```

```ts
const data = row.data; // Retro

if (data.version === 1) {
	data.problem; // ✅ as 없이 KPT로 좁혀진다
} else {
	data.lacked; // ✅ 여기선 4L
}
```

> 💡 위와 같이 공통 필드(`version`)의 리터럴 값으로 유니온을 구분하는 걸 **Discriminated Union**이라고 한다.

이렇게 하면 `version`으로 TypeScript가 `data` type을 좁힐 수 있다.

처음 스키마에서는 `row.version`과 `row.data`가 서로 다른 속성이라, `version`을 검사해도 `data`의 type은 그대로였다. 이제는 `version`이 `data` 객체 안에 있다. 그래서 `data.version === 1`을 검사하면 TypeScript가 `version`이 2인 경우를 지우고, `data`를 KPT 객체로 좁힌다. 둘이 같은 객체 안에 들어가면서 연결되어 있다는 정보가 type에 생긴 것이다.

잘못 쓴 코드도 이제는 컴파일러가 잡아준다.

```ts
if (data.version === 1) {
	data.lacked; // ❌ 에러: version 1에는 lacked가 없다
}
```

원래는 `as`로 잘못 단언해도 컴파일러가 몰랐지만, 이제는 버전에 맞지 않는 필드를 쓰는 순간 에러가 난다.

## 마치며

이번에 이렇게 바꾸면서 DB 구조를 정할 때 코드에서 type이 어떻게 추론될지까지 생각해야 된다는 걸 깨달았다. 다음에 DB를 설계할 때는 edge case나 고려해야 할 것들을 더 생각해야겠다.

> 버전이 있는 데이터라면, 버전은 데이터 안에 넣자.
