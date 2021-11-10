---
title: "DB Lock을 예방하는 7가지 팁"
layout: single
categories:
  - Database
tags:
  - Database, PG정복기
date: '2021-11-09 22:42:14'
last_modified_at: '2021-11-09 22:42:14 +0800'
toc: true
toc_sticky: true
toc_label: Table of Contents
---

> Lock은 트랜잭션 처리의 순차성을 보장해주는 기능을 제공합니다. 트랜잭션은 DB가 처리하는 가장 작은 단위의 처리 단위입니다. Lock은 하나의 트랜잭션이 완벽하게 끝날 때까지 다른 요청을 막아줍니다.

## **1: Never add a column with a default value**

postgresql에는 골든 룰이 하나 있습니다.

테이블에컬럼을 추가할 때는 ***절대 <span style="color:red">default</span> 값을 설정하지 마라.***

컬럼을 추가하는 행위 자체가 락을 유발하는 매우 공격적인 행위입니다. 더군다나 default 값을 같이 설정해 버리면 Postgresql은 전체 테이블에 write 작업을 수행하게 됩니다.

 이 경우 데이터베이스는 해당 테이블을 Lock 하며 조회 및 수정이 불가능 하게 됩니다. (+ 큰 테이블의 경우 어마어마한 시간이 소요된다!)

 <br>

 ```sql
 -- Never Do This!
ALTER TABLE items ADD COLUMN last_update timestamptz DEFAULT now();

-- Do instead!
ALTER TABLE items ADD COLUMN last_update timestamptz;
UPDATE items SET last_update = now();
 ```
 따라서 위와 같이 기본값 설정이 없는 컬럼을 먼저 추가한 후 UPDATE 쿼리를 날리는 것이 좋습니다.

 혹은,

 ```sql
 do {
  numRowsUpdated = executeUpdate(
    "UPDATE items SET last_update = ? " +
    "WHERE ctid IN (SELECT ctid FROM items WHERE last_update IS NULL LIMIT 5000)",
    now);
} while (numRowsUpdate > 0);
 ```
위와 같이 업데이트 할 컬럼 수를 쪼개서 하는 것이 좋습니다.
<br><br>

## **2: Beware of lock queues, use lock timeouts**

Postgresql의 모든 Lock은 queue를 가지고 있습니다.

만약 Transaction A가 Confilicting lock level로 점유하고 있는 lock에 Transaction B가 접근하고자 하면 Transaction B는 lock queue에서 Transaction A가 Commit 되거나 Rollback을 통해 점유가 풀리기까지 대기합니다.

이후 또 다른 Transaction C가 lock을 얻어야 하는 경우 이 또한 queue에서 대기하게 됩니다.

하지만 이때 Transaction C는 Transaction A의 lock level과 충돌 나지 않더라도 대기해야 된다는 점이 간단한 쿼리임에도 불구하고 실행의 시간을 늘리는 원인이 됩니다.

```sql
-- Never Do This
ALTER TABLE items ADD COLUMN last_update timestamptz;

-- Do Instead
SET lock_timeout TO '2s'
ALTER TABLE items ADD COLUMN last_update timestamptz;
```

따라서 위와 같이 lock_timeout을 걸어 줌으로서 lock 점유 시간을 강제하고 대기 큐를 실행 할 수 있도록 해주는 것이 좋다.
<br><br>

## **3: Create indexes CONCURRENTLY**
인덱스를 추가할 때는 CONCURRENTLY 옵션을 통해 추가하는 것이 좋습니다.

특히 큰 테이블에 인덱스를 추가할 경우에는 시간이 어마어마하게 소요됩니다. 
길게는 몇일 소요되는 경우도 있습니다.

또한 이 기간동안 모든 Write 실행을 막게됩니다.

```sql
-- Block all write
CREATE INDEX items_value_idx ON items USING GIN (value jsonb_path_ops);

- only block DDL
CREATE INDEX CONCURRENTLY items_value_idx ON items USING GIN (value jsonb_path_ops);
```
하지만 CONCURRENTLY 옵션을 넣어 인덱스를 생성하게 되면 DDL(CREAT, ALTER, DROP, TRUNCAT)만 막게 됩니다.
<br><br>

## **4: Take aggressive locks as late as possible**
만약 lock이 많이 걸릴 것 같은 커멘드는 최대한 transaction의 마지막에 실행 하는 것이 좋습니다.

```sql
-- Don't do this
BEGIN;
-- reads and writes blocked from here:
TRUNCATE items;
-- long-running operation:
\COPY items FROM 'newdata.csv' WITH CSV 
COMMIT;

-- Do instead
BEGIN;
CREATE TABLE items_new (LIKE items INCLUDING ALL);
-- long-running operation:
\COPY items_new FROM 'newdata.csv' WITH CSV
-- reads and writes blocked from here:
DROP TABLE items;
ALTER TABLE items_new RENAME TO items;
COMMIT;
```
위와 같이 테이블의 모든 데이터를 지우고 덤프를 하는 방식 보다는

새 테이블을 만들어 복사 후 이름을 변경해 주는 것이 좋습니다.
<br>
<br>

## **5: Adding a primary key with minimal locking**
Postgesql의 경우 `ALTER` 명령을 통해 쉽게 PK를 추가 할 수 있습니다.

하지만 크기가 큰 테이블에 PK를 추가하는 경우 시간이 많이 소요될 수 있습니다. 뿐만 아니라 명령을 실행하는 동안 모든 쿼리가 block이 됩니다.
<br>

```sql
-- blocks queries for a long time
ALTER TABLE items ADD PRIMARY KEY (id);

CREATE UNIQUE INDEX CONCURRENTLY items_pk ON items (id); -- takes a long time, but doesn’t block queries
ALTER TABLE items ADD CONSTRAINT items_pk PRIMARY KEY USING INDEX items_pk;  -- blocks queries, but only very briefly
```
<br><br>

## **6: Never VACUUM FULL**
~~절대~~ vacuum full을 쓰면 안됩니다.

(실제로는 써야 하는 경우도 생긴다.)

vacuum full은 디스크에 전체 테이블을 다시 쓰는 작업을 하기 때문에 많게는 몇일이 걸릴 수가 있습니다. 이 동안 모든 쿼리가 블락되게 됩니다.

필요하다면 vacuum을 쓰되 vacumm full은 쓰지 말자!
<br><br>

## **7: Avoid deadlocks by ordering commands**
Postgresql을 사용하다 보면 다음과 같은 에러를 본 적이 있을 수도 있으실 겁니다.

```sql
ERROR:  deadlock detected
DETAIL:  Process 13661 waits for ShareLock on transaction 45942; blocked by process 13483.
Process 13483 waits for ShareLock on transaction 45937; blocked by process 13661.
```
여러 트랜색션이 동일한 lock을 걸려고 할 때 생기는 문제입니다.

이를 피하기 위해서는 deadlock을 발생 시키지 않게끔 순서를 조정해서 실행해야 하는 것 이 중요합니다.

<br><br>

:bulb:
본 글은<br> [When Postgres blocks: 7 tips for dealing with locks by Marco Slot](https://www.citusdata.com/blog/2018/02/22/seven-tips-for-dealing-with-postgres-locks/) <br> article을 보고 정리한 글입니다.
{: .notice--info}