---
title: "PostgreSQL Autovacuum에 대하여"
layout: single
categories:
  - Database
tags:
  - Database, PG정복기
date: '2021-11-09 22:42:14'
last_modified_at: '2021-11-10 22:42:14 +0800'
toc: true
toc_sticky: true
toc_label: Table of Contents
---

서비스 운영을 전담하게 된 이후 이런 메시지를 자주 받게 되었습니다.
<br>

![title](/assets/images/posts/post_autovacuum_1.png)

크게 바쁜 시간대도 아니였기에 의문점을 가지고 모니터링을 했습니다.

해당 시간 cpu 사용을 분석해 보니 autovacuum에 의한 것으로 예상되었습니다.

타 RDBMS에서는 접하지 못했던 용어라 조사를 시작하였습니다.
<br><br>

## Autovacuum 이란

💡 [Autovacuum 공식 문서](https://www.postgresql.org/docs/current/routine-vacuuming.html)
{: .notice--success}

PostgreSQL에만 존재하는 기능으로 **autovacuum**이 동작하는 근거는 다음과 같다.

1. <u>update나 deleted 쿼리에 의해 점령당한 디스크 스페이스</u>를 다시 사용 가능한 상태로 돌리기 위해서.
2. PostgreSQL 쿼리 실행 계획기가 사용할 자료 통계 정보를 갱신할 필요가 있다.
3. [index-전용 조회](https://postgresql.kr/docs/12/indexes-index-only-scans.html) 속도를 높이기 위해, 실자료 지도를 갱신할 필요가 있다.
4. 트랜잭션 **ID **겹침이나, 다중 트랜잭션 ID **겹침 상황으로 오래된 자료가 손실 될 가능성을 방지해야할 필요가 있다.

(여기서 3, 4번은 해당사항이 없기에 1, 2번 위주로 서술하겠습니다.)

<br>
여기서 <u>update나 deleted 쿼리에 의해 점령당한 디스크 스페이스</u> 는 무엇일까?

PostgreSQL은 update나 delete 작업 대상이 된 해당 자료의 옛 버전을 바로 버리지 않습니다. 

이를 dead tuple이라 합니다. 
<br>


> PostgreSQL에서 데이터는 tuple이라는 형태로 저장이 된다.
 tuple은 live tuple, 트랜잭션이 진행되는 중이라 아직 사용자들이 볼 수 있는 데이터,
혹은 dead tuple, 트랜잭션이 완료되어 사용자들에게 보이지 않도록 마킹 해놓은 데이터
로 구분된다.
> 

이는 postgresql의 [다중 버전 동시성 제어](https://www.notion.so/MVCC-bff62e6a666f4c76a442be5c2ec3bcfd)(multiversion concurrency control, MVCC)에 있기 때문이라고 합니다. 예를 들어 만약 삭제된 자료를 다른 트랜잭션에서 사용하고 있다면, 그 자료가 삭제되면 안되기 때문에 이러한 개념을 도입했다고 합니다.

하지만 이런 상태로 지속게되면, 디스크에는 쓸모 없는 자료들(dead tuple)이 **많은 공간을 차지하게 됩니다**. 따라서 이런 쓸모 없는 자료들을 정리하여 빈 공간으로 바꾸는 작업을 `vacuum` 명령을 통해 처리할 수 있습니다.

`vacuum`은 표준 vacuum과 vacuum full, 두 가지 종류가 있습니다. vacuum full 작업은 물리적인 디스크 여유 공간을 확보할 수 있으나 그 작업 속도가 매우 느리고 Database를 Lock 후 진행하기 때문에 현재 사용중인 Database에는 적합하지 않는 방식입니다.

따라서 일반적인 관리자 같은 경우는 표준 vacuum을 주기적으로 작업하여 꾸준히 빈 공간을 확보하는 것이 중요합니다. 이를 전략적으로 가능학 하는 것이 `autovacuum` 입니다.
<br><br>

## Autovaccum 작업 방식

autovacuum은 주어진 전략을 바탕으로 실행됩니다. 이는 `autovacumm_` 으로 시작되는 환경 변수를 통해 조작 가능합니다.

<br>

### Autovacuum 환경변수


|변수명|설명|
|------|---|
|autovacuum|autovacuum 기능을 활성화 할 것인지 결정|
|autovacuum_analyze_scale_factor|Number of tuple inserts, updates, or deletes prior to analyze as a fraction of reltuples.|
|autovacuum_analyze_threshold|Minimum number of tuple inserts, updates, or deletes prior to analyze.|
|autovacuum_freeze_max_age|트랙잭션 ID 겹침 방지 작업을 시작하는 테이블 나이|
|autovacuum_max_workers|동시에 실행 될 수 있는 autovacuum 작업 >프로세스 최대 개수|
|autovacuum_multixact_freeze_max_age|Multixact age at which to autovacuum a table to prevent multixact wraparound.|
|autovacuum_naptime|Time to sleep between autovacuum runs.|
|autovacuum_vacuum_cost_delay|Vacuum cost delay in milliseconds, for autovacuum.|
|autovacuum_vacuum_cost_limit|Vacuum cost amount available before napping, for autovacuum.|
|autovacuum_vacuum_scale_factor|autovacuum 작업 대상 선정 기준이 되는 테이블 변화량, 백분율|
|autovacuum_vacuum_threshold|Minimum number of tuple updates or deletes prior to vacuum.|
|autovacuum_work_mem|Sets the maximum memory to be used by each autovacuum worker process.|


<br>

이를 바탕으로 autovacuum 작업 프로레스를 설명하자면,

1. 1분(autovacuum_naptime)마다 'autovacuum launcher process' 백그라운드 프로세스가 전체 데이터베이스를 뒤지면서 vacuum 작업 대상을 찾는다.
2. 선정 대상 기준은 
    1. age(pg_class.relfrozenxid) 값이 autocacuum_freeze_max_age 값보다 큰 모든 테이블(자료가 변경되었건 되지 않았건 무조건 모든 테이블)을 대상으로 트랜잭션 겹침 방지 작업을 위한 vacuum freeze 작업을 진행하고, 
    2. pg_stat_all_tables.n_mod_since_analyze 값과 autovacuum_vacuum_scale_factor 값을 비교해서 찾는다. (내부적으로 좀 더 복잡한 계산식이 사용되지만, 단순하게, 전체 자료 대비 변경,삭제된 자료가 20%(autovacuum_vacuum_scale_factor 기본값) 넘으면 그 대상이 된다.)
3. autovacuum launcher 프로세스가 autovacuum worker 프로세스를 만들어 해당 테이블의 vacuum 작업을 진행한다. autovacuum worker 는 autovacuum_vacuum_cost_delay 시간(기본값 20ms)만큼 작업 도중 틈틈히 쉬어가면서 vacuum 작업을 진행한다. (그래서 사용자가 직접 실행하는 vacuum 작업보다 오래걸린다. 심하면 10배 이상의 시간이 걸리기도 한다.)
4. 이런 작업을 autovacuum_max_workers 수 만큼 실행해서 처리한다.
<br><br>

## Autovacuum의 한계
<br>
만약 큰 데이블의 autovacuum 작업이 진행되고 있고, 이 작업이 autovacuum_max_workers에 설정한 세팅값 만큼 이용하고 있다면, 다른 테이블에 대한 dead tuple vacuum 작업은 지연 되게 됩니다. 이것이 가장 큰 문제점이라고 합니다.

이를 관리자가 확인하는 방법은 

`pg_stat_all_tables.n_dead_tup` 값과, `pg_stat_all_tables.last_autovacuum` 값을 통해 모니터링 하면서

`n_dead_tuple` 값이 계속 커지는데 `last_autovacuum` 값은 바뀌지 않는다면, 현재 `autovacuum_max_worker`를 full로 활용한 작업이 진행 중 이라는 상황이므로 대처를 해주어야 합니다.

이에 대한 해결 방안은 

1. 직접 vacuum 명령 실행
2. `autovacuum_max_workers` 값 증가

가 있습니다.

1번 방법은 사용자가 직접 실행하는 만큼 시스템의 자원을 100% 사용하여 처리 속도가 빠르다는 점이 있지만, 현재 서비스가 사용 중이라면 서비스에 영향을 크게 줄 수 있으니 여유 자원을 확인 후 진행 하는 것이 좋습니다.

2번 방법 또한 시스템 자원에 영향을 받습니다. 통상 CPU 코어 수의 1~2배 범위 안에서 지정하는 것이 좋다고 합니다.
<br><br>

## Autovacuum 관리

그렇다면 관리자는 autovacuum을 어떻게 모니터링 하고 전략을 수립하는 것이 좋을까?
<br>

### 1. 모니터링
<br>
먼저 dead tuple로 인한 데이터 부풀림을 조사하여야 합니다.

하지만 현재 상용되는 PostgreSQL 버전에서는 이를 명확하게 확인할 수 있는 방안이 없다고합니다.

가장 확실한 방법은 원본 테이블의 자료만 뽑아서 새로운 복제 테이블을 만들고 그 크기를 확인하는 것인데 이는 서비스를 수행 중인 데이터베이스에서는 거의 불가능한 방법이므로 불가능 합니다.

이에대한 해결책으로 `analyze` 명령을 활용하는 방법이 있습니다.

(autovacuum에서는 테이블 변경량이 10%가 넘으면 자동으로 통계를 수집한다고 한다.)

`pg_statistic` 테이블에는 `analyze` 명령을 통해 수집된 통계 정보들이 보관되는데 `pg_statistic.stawidth` 컬럼을 활용하여 대략적인 값을 알 수 있다.

```bash
postgres=# select sum(stawidth) from pg_statistic where starelid = 't_rent_route'::regclass;
 sum 
-----
 2567
(1 row)

postgres=# explain select * from pg_class;
                         QUERY PLAN                          
-------------------------------------------------------------
Seq Scan on t_rent_route  (cost=0.00..465177.57 rows=1352357 width=2567)
(1 row)
```

위 예제를 보면, t_rent_route 테이블의 로우는 2567바이트로 예상되는 것 을 확인 할 수 있습니다.

즉, t_rent_route 테이블 전체 로우수와 이 수를 곱하면 테이블의 실제 크기를 어느정도 예측이 가능합니다.

물론 이는 예상이므로 정확하지 않습니다. autovacuum의 설정값 중 default_statistics_target은 1%의 통계만 수집하게 되어있기 때문입니다. 
하지만 이를 통해 어느정도 인사이트를 가질 수 있다고 합니다.

다른 모니터링 기법으로는,

- autovacuum 작업으로 해당 테이블의 통계 정보가 잘 수집되고 있는지(pg_stat_all_tables.last_autoanalyze)
- 현재 세션들 가운데, 트랜잭션 상황에서 아주 오래 동안 유지하고 있는 세션이 있는지 (pg_stat_activity.backend_xmin)
- 테이블의 dead tuple 수가 계속 증가하고 있는지 (pg_stat_all_tables.n_dead_tuples)
- 데이터베이스가 나이를 계속 먹어가고 있는지 (pg_database.datfrozenxid)

등을 통해 관리자가 데이터 부풀림을 확인 가능합니다.

### 2. 최적화
<br>
현재 자사의 서비스는 구조적으로 update 및 delete가 수초에 한번씩 몇 천번의 요청이 들어오는 상황입니다. 이에 따라 dead tuple이 많이 생길 수 밖에 없는데 이는 종국에 CPU 사용량 증가와 성능 저하로 이어질 위험이 있습니다.

이를 예방하기 위해서 사전에 autovacuum 최적화가 필요하다고 생각하였습니다.

#### **2.1. autovacuum_vacuum_scale_factor 를 0 으로 설정하기**
<br>
가장 간단한 방법이자 효과를 확실히 볼 수 있는 방법 이라고 합니다.

`autovacuum_vacuum_threshold` 에 지정된 숫자만큼의 dead tuple 에 따라 Autovacuum 이 동작하게 되므로 훨씬 일관성 있는 성능을 확보할 수 있습니다. 하지만 이 설정을 postgresql 전역에 설정하게 될 시 업데이트가 잦지 않은 테이블들에도 영향을 줄 수 있기 때문에 각 테이블 별로 적용하는 것이 안전하다고 합니다.

```sql
ALTER TABLE t_share_vehicle SET (autovacuum_vacuum_scale_factor = 0.0);
ALTER TABLE t_share_vehicle SET (autovacuum_vacuum_threshold = 100000);
```

위와 같이 설정하게 되면 dead tuple이 `autovacuum_vacuum_threshold` 값인 100000에 도달 할 때 마다 autovacuum이 동작하게 됩니다. 

#### **2.2. autovacuum_vacuum_cost_limit 을 증가시키기**
<br>

💡 "자동 VACUUM 명령에 사용되는 비용 제한 값을 지정한다. -1을 지정하면(기본값)일반 [vacuum_cost_limit](https://postgresql.kr/docs/9.5/runtime-config-resource.html#GUC-VACUUM-COST-LIMIT) 값이 사용된다. 실행 중인 autovacuum workers가 하나 이상 있을 경우 각 worker 제한의 합계가 이 변수의 제한값을 초과하지 않도록 값이 비례 분배된다. 이 매개변수는 postgresql.conf 파일 또는 서버 커맨드 라인에서만 설정 가능하다. 이 설 정은 저장소 매개변수를 변경함으로써 개별 테이블에 오버라이드할 수 있다. " \- PostgreSQL 공식 문서
{: .notice-success}

<br><br>

:bulb:
본 글은<br> 
현재 진행 중인 글입니다.
{: .notice--info}