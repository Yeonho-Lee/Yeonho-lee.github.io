---
title: SQLite + Drizzle 조합, 언제 유리할까
categories: [Database, SQLite]
tags: [database, sqlite, wal, drizzle, node-sqlite, sveltekit]
date: 2026-11-19 21:20:00 +0900
---

> 📚 **검색 한 번의 여정** 시리즈
> 1. [네이버에서 검색 한 번 하면 무슨 일이 일어날까]({% post_url 2026-11-19-네이버에서 검색 한 번 하면 무슨 일이 일어날까 %})
> 2. [네트워크 계층과 HTTP, 그리고 REST]({% post_url 2026-11-19-네트워크 계층과 HTTP, 그리고 REST %})
> 3. **SQLite + Drizzle 조합, 언제 유리할까** ← 지금 글

[지난 글]({% post_url 2026-11-19-네트워크 계층과 HTTP, 그리고 REST %})까지 해서 요청이 백엔드에 도착하는 과정을 봤다. 이 글에서는 요청의 마지막 목적지인 **DB**를 다루고자 한다.

지금 프로젝트에서는 **SQLite**와 **Drizzle ORM**을 쓰고 있는데, 커피챗에서 "왜 SQLite를 쓰냐"는 질문을 받았다. 그래서 이 조합이 어떤 조건에서 유리한지 정리하고, 지금 프로젝트를 예로 들어 본다.

## SQLite는 RDBMS다

SQLite는 MySQL, PostgreSQL처럼 데이터를 **테이블**에 저장하고 테이블끼리의 **관계**를 정의하는 **관계형 DB(RDBMS)**다. 회원과 주문, 게시글과 댓글처럼 데이터끼리 관계가 뚜렷하다면 관계형 DB가 자연스러운 선택이다.

> 💡 지금 프로젝트도 학생, 선생님, 수업처럼 서로 얽힌 데이터를 다룬다. RDBMS와 NoSQL의 차이는 다음 글에서 따로 다룬다.

## SQLite가 유리한 이유

### 1. 가볍다

설치하거나 설정할 게 거의 없다. 작은 서비스나 초기 프로젝트에서 부담 없이 시작할 수 있다.

### 2. 서버가 없다. 파일 자체가 DB다

MySQL이나 PostgreSQL은 DB 서버를 따로 띄우고 거기에 접속해야 한다. SQLite는 그냥 **파일 하나**가 DB다.

```
project/
├── src/
└── local.db   ← 이게 DB 전체
```

DB 서버를 따로 운영하지 않아도 되니까 배포와 관리가 훨씬 단순하다. 백업도 파일을 복사하면 끝이다.

또 SQLite는 별도 프로그램이 아니라 백엔드 안에서 돌아가는 **라이브러리**다. 서버형 DB는 쿼리 하나마다 네트워크를 거쳐 DB 서버와 주고받지만, SQLite는 같은 프로세스 안에서 파일을 바로 읽는다. 쿼리 하나하나의 왕복 비용이 작다.

> 💡 정확히는 "SQLite **자체**에는 서버가 없다"이다. SQLite를 서버로 감싸서 네트워크 너머에서 쓰게 해주는 서비스도 있다. [Cloudflare D1](https://developers.cloudflare.com/d1/)은 SQLite 기반의 서버리스 DB이고, [Turso](https://turso.tech/)는 SQLite를 포크한 libSQL에 서버와 복제 기능을 붙였다. Turso는 2026년 10월 [Supabase에 인수](https://www.prnewswire.com/news-releases/supabase-announces-150m-in-new-funding-and-turso-acquisition-302896752.html)됐다.

### 3. WAL 모드로 읽기와 쓰기를 동시에

SQLite는 기본 모드(롤백 저널)에서 DB 파일 **전체**에 잠금을 건다.

| 상황 | 잠금 | 다른 연결은 |
| --- | --- | --- |
| 읽는 중 | SHARED | 여럿이 동시에 읽을 수 있다 |
| 쓰기 준비 중 | RESERVED | 읽기는 되지만, 다른 쓰기는 안 된다 |
| 커밋하는 순간 | EXCLUSIVE | 읽기도 쓰기도 `database is locked` |

게다가 읽는 중인 연결이 남아 있으면, 쓰는 쪽은 EXCLUSIVE 잠금을 잡지 못해 커밋하지 못하고 기다린다. 결국 **읽기와 쓰기가 서로를 막는다.** 이걸 풀어주는 게 **WAL(Write-Ahead Logging)** 모드다.

```sql
PRAGMA journal_mode = WAL;
```

WAL 모드에서는 변경 내용을 DB 파일에 바로 쓰지 않고, 별도의 `-wal` 파일에 **먼저 기록**한다. 쌓인 내용은 나중에 **체크포인트** 시점에 원래 DB 파일로 옮겨진다.

실제로 WAL 모드를 켜고 데이터를 한 번 쓰면 파일이 늘어난다.

```bash
$ ls
local.db                  # WAL 켜기 전

$ ls
local.db  local.db-shm  local.db-wal   # WAL 켜고 한 번 쓴 뒤
```

`-wal`이 먼저 기록되는 로그 파일이고, `-shm`은 여러 연결이 WAL 파일을 같이 쓰기 위한 공유 메모리 인덱스다. 이 설정은 DB 파일에 저장돼서, 다음에 다른 프로그램으로 열어도 WAL 모드가 유지된다.

덕분에 한쪽에서 쓰는 동안에도 다른 쪽은 기존 데이터를 계속 읽을 수 있다. 읽는 쪽은 **읽기를 시작한 시점의 스냅샷**을 보기 때문에, 쓰는 도중의 어중간한 데이터를 볼 일도 없다. 여러 버전의 데이터를 동시에 유지해서 읽기와 쓰기가 서로 막지 않게 하는 이 방식을 **MVCC(다중 버전 동시성 제어)**라고 부른다. WAL에서는 읽기와 쓰기가 서로를 막지 않는다.

그래도 **쓰기끼리는 여전히 한 번에 하나**만 가능하다. 한쪽 연결이 쓰기 잠금을 잡고 있을 때, 다른 연결에서 쓰기를 시도해 보면 차이가 보인다.

```ts
const holder = new DatabaseSync('local.db');
holder.exec('BEGIN IMMEDIATE'); // 쓰기 잠금을 잡고 놓지 않는다

const writer = new DatabaseSync('local.db', { timeout: 1000 });
writer.exec("INSERT INTO students (name) VALUES ('대기')");
```

| `timeout` | 결과 |
| --- | --- |
| `0` (`node:sqlite` 기본값) | 즉시 `database is locked` 에러 |
| `1000` | 약 1초 기다린 뒤 `database is locked` 에러 |
| `5000` (잠금이 0.5초 뒤 풀릴 때) | 약 0.5초 기다린 뒤 **성공** |

`timeout`은 그 시간 안에 잠금이 풀리면 이어서 쓰고, 끝까지 안 풀릴 때만 에러를 낸다. 반대로 같은 상황에서 **읽기는 기다리지 않고 바로 성공**한다. 이게 앞에서 말한 WAL의 장점이다.

> 💡 SQLite 자체는 기본적으로 잠금이 풀리길 기다리지 않는다. 다만 감싸는 라이브러리마다 기본값이 다르다. `node:sqlite`는 0이라 바로 에러가 나고, `better-sqlite3`는 5초를 기다린다. 쓰는 라이브러리의 기본값을 꼭 확인하자.

> 💡 그래서 여러 프로세스가 같은 DB 파일에 쓸 수 있다면 `timeout`을 주는 게 좋다. 지금 프로젝트에서는 `timeout: 5000`을 줘서, 다른 쪽이 쓰고 있으면 바로 에러를 내지 않고 최대 5초까지 기다리게 했다.

## SQLite가 불리한 경우

- **쓰기가 많은 서비스**: 쓰기는 한 번에 하나라서, 동시에 쓰는 요청이 많으면 줄을 서느라 병목이 생긴다.
- **서버를 여러 대로 늘려야 하는 서비스**: DB가 서버 한 대의 파일이라, 여러 서버가 같은 DB를 나눠 쓰기 어렵다.

이런 한계가 문제가 되면 PostgreSQL 같은 서버형 DB나, 그걸 클라우드로 제공하는 **Supabase** 같은 SaaS를 고려하게 된다.

반대로 SQLite에 서버와 복제를 붙여 이 한계를 넘으려는 시도도 있다. 앞에서 본 Turso, Cloudflare D1 말고도, 여러 서버가 같은 SQLite를 나눠 쓰게 해주는 [rqlite](https://rqlite.io/), [LiteFS](https://fly.io/docs/litefs/) 같은 도구가 있다.

## 그래서 이런 서비스에 맞다

정리하면 아래 조건에 많이 해당할수록 SQLite가 잘 맞는다.

- [ ] 서버 한 대로 충분하다
- [ ] 읽기에 비해 쓰기가 많지 않다
- [ ] 데이터끼리 관계가 뚜렷하다
- [ ] DB 서버를 따로 운영할 사람이나 여유가 없다

이 정도 규모라면 DB 서버를 따로 운영하는 비용보다 파일 하나로 끝나는 단순함이 더 크다.

> 💡 지금 프로젝트는 서버 한 대에서 돌아가고, 사용자가 주로 내부 직원이라 쓰기 요청이 많지 않다. 네 가지 조건에 모두 해당해서 SQLite를 쓰고 있다.

### 조건을 벗어나도 생각보다 멀리 간다

위 체크리스트는 "고민 없이 맞는 경우"일 뿐이다. SQLite로 꽤 큰 트래픽을 처리하는 사례도 많다.

- **SQLite 공식 사이트**는 SQLite로 하루 40~50만 건의 요청을 처리한다(2015년 기준). [공식 문서](https://www.sqlite.org/whentouse.html)는 "하루 10만 히트 이하라면 문제없다"고 하면서도, 이건 보수적으로 잡은 수치라고 말한다.
- **Expensify**는 SQLite를 고쳐 만든 자체 DB(Bedrock)로, 서버 한 대에서 읽기 위주 벤치마크 기준 [초당 400만 쿼리](https://use.expensify.com/blog/scaling-sqlite-to-4m-qps-on-a-single-server)를 처리한 결과를 공개했다.
- **Rails 8**은 SQLite를 개발용이 아니라 프로덕션에서도 쓸 수 있는 DB로 공식 지원하기 시작했다.

이런 사례는 보통 몇 가지 최적화를 같이 쓴다.

- WAL 모드 + `PRAGMA synchronous = NORMAL`
- 여러 쓰기를 **하나의 트랜잭션으로 묶기**
- 쓰기를 한 곳(큐)으로 모아 순서대로 처리하기
- [Litestream](https://litestream.io/) 같은 도구로 실시간 백업과 복제

## Drizzle을 같이 쓰면 좋은 이유

Drizzle은 **ORM**이다. SQL을 직접 문자열로 쓰는 대신 TypeScript 코드로 테이블을 정의하고 쿼리를 짤 수 있게 해준다. SQLite와 같이 쓸 때 좋은 점을 세 가지로 정리해 본다.

### 1. Type-safe하다

```ts
// schema.ts
import { sqliteTable, integer, text } from 'drizzle-orm/sqlite-core';

export const students = sqliteTable('students', {
	id: integer('id').primaryKey({ autoIncrement: true }),
	name: text('name').notNull()
});
```

```ts
// 쿼리
const list = await db.select().from(students).where(eq(students.name, '홍길동'));
```

테이블 구조가 타입으로 잡히니까 컬럼 이름을 잘못 쓰면 바로 에러가 난다.

```ts
db.select().from(students).where(eq(students.nmae, '홍길동'));
//                                          ~~~~
// Property 'nmae' does not exist on type 'SQLiteTableWithColumns<{ name: "students"; ... }>'
```

> 💡 Drizzle에서는 테이블, 컬럼 이름을 한글로 지정할 수도 있다. [이전 글]({% post_url 2026-03-16-한글로 DB테이블,컬럼명을 지정할 수 있다고 %})에서 다뤘다.

### 2. SQL에 가깝고 얇다

Drizzle의 쿼리 문법은 `select`, `from`, `where`처럼 SQL과 거의 같은 모양이라, SQL을 알면 따로 배울 게 적다. 그리고 SQL을 감추지 않는다. Drizzle은 TypeScript 코드를 **SQL로 바꿔서** 드라이버에 넘겨주는 얇은 층일 뿐이다. `.toSQL()`을 붙이면 실제로 어떤 SQL이 나가는지 볼 수 있다.

```ts
db.select().from(students).where(like(students.name, '%홍%')).toSQL();
// {
//   sql: 'select "id", "name" from "students" where "students"."name" like ?',
//   params: ['%홍%']
// }
```

[첫 번째 글]({% post_url 2026-11-19-네이버에서 검색 한 번 하면 무슨 일이 일어날까 %})에서 본 `SELECT ... WHERE ... LIKE` 쿼리가 그대로 나온다. 검색어가 SQL 문자열에 직접 들어가지 않고 `?` 자리에 **파라미터**로 따로 넘어가는 것도 눈여겨볼 점이다. 덕분에 SQL 인젝션 걱정을 덜 수 있다.

### 3. SQLite 드라이버를 골라 쓸 수 있다

Drizzle은 드라이버를 **포함하지 않는다**. 대신 여러 SQLite 드라이버를 지원해서, 실행 환경에 맞는 걸 골라 감싸 쓴다.

| 실행 환경 | 드라이버 |
| --- | --- |
| Node.js | `better-sqlite3`, `node:sqlite` (Node.js 내장) |
| Bun | `bun:sqlite` (Bun 내장) |
| 원격 SQLite (Turso 등) | `@libsql/client` |

특히 Node.js에는 22.5부터 `node:sqlite` 모듈이 **내장**돼 있어서, 이걸 쓰면 드라이버 패키지를 따로 설치할 필요도 없다.

```ts
// db/client.ts
import { DatabaseSync } from 'node:sqlite'; // Node.js 내장 드라이버
import { drizzle } from 'drizzle-orm/node-sqlite';

const client = new DatabaseSync(DATABASE_URL, { timeout: 5000 });
client.exec('PRAGMA journal_mode = WAL');

export const db = drizzle({ client });
```

> 💡 커피챗에서 "SQLite 드라이버를 따로 설치한 적이 없는데, Drizzle이 대신 해주는 건가?"라는 이야기가 나왔다. 확인해 보니 반은 맞고 반은 틀렸다. 지금 프로젝트는 Node.js 24에서 `node:sqlite`를 쓰고 있어서, 드라이버는 Node.js가 제공하고 Drizzle은 그 위에서 쿼리를 만들어 줄 뿐이었다. `package.json`에 드라이버 패키지가 없는 이유다.

> 💡 `drizzle-orm/node-sqlite`는 **Drizzle 1.0(베타/RC)부터** 들어 있다. 글 쓰는 시점의 안정 버전(0.45.x)에는 없어서, 0.45를 쓴다면 `better-sqlite3`나 `@libsql/client` 같은 드라이버를 따로 설치해야 한다. 지금 프로젝트는 `1.0.0-rc.4`를 쓴다. 또 `node:sqlite`는 Node.js 22.5에서 처음 들어왔지만 그때는 `--experimental-sqlite` 플래그가 있어야 했고, 22.13부터 플래그 없이 쓸 수 있다.

## 마치며

SQLite + Drizzle은 **서버 한 대, 쓰기가 많지 않은, 관계형 데이터**를 다루는 서비스에서 특히 잘 맞는 조합이다. DB 서버 없이 파일 하나로 시작하고, 쿼리는 타입 안전하게 짤 수 있다.

나중에 서비스가 커져서 PostgreSQL로 옮기게 되더라도 Drizzle 쿼리 코드는 거의 비슷하게 유지된다. 다만 스키마 정의는 `sqliteTable`에서 `pgTable`로 바꿔야 한다.

이걸로 커피챗에서 나온 "검색 한 번의 여정"은 일단 마무리다.
