# SQL 처리 과정과 I/O

## 1.1 SQL 파싱과 최적화
- 두 개의 테이블
  - EMP 테이블
      - 사원번호(EMPNO)
      - 사원명(ENAME)
      - 직업(JOB)
      - 부서번호(DEPTNO)
  - DEPT 테이블
      - 부서번호(DEPTNO)
      - 부서명(DNAME)
      - 지역(LOC)

- 두 테이블을 부서 번호로 조인하고 사원명 순으로 정렬하는 SQL
```sql
SELECT E.EMPNO, E.ENAME, E.JOB, D.DNAME, D.LOC
FROM EMP E, DEPT D
WHERE E.DEPTNO = D.DEPTNO
ORDER BY E.ENAME;
```

### 1.1.1 구조적, 집합적, 선언적 질의 언어
- SQL은 구조적 질의 언어(Structured Query Language)이다.
  - 또한 집합적이고 선언적인 질의언어이기도 하다.
  - 원하는 결과의 집합을 구조적, 집합적으로 선언하지만 그 과정은 절차적일 수 밖에 없다.


### 1.1.2 SQL 최적화
- SQL 최적화 과정의 세분화
- SQL 최적화 과정의 세분화
  - SQL 파싱
    - 파싱 트리 생성 : SQL 문장을 분석하여 파싱 트리 생성(select 등)
    - 문법 체크 : SQL 문장이 문법적으로 올바른지 체크(오타, 누락 등)
    - 시맨틱 체크 : SQL 문장이 의미적으로 올바른지 체크(컬럼, 테이블 존재 여부 등)
  - SQL 최적화
    - 최적화 트리 생성 : 파싱 트리를 분석하여 최적화 트리 생성(실행 계획)
    - 통계 정보 확인 : 최적화 트리 생성에 필요한 통계 정보를 확인(인덱스 통계, 테이블 통계 등)
    - 비용 계산 : 최적화 트리의 비용을 계산(비용이 가장 적은 실행 계획 선택)
  - 로우 소스 생성
    - 옵티마이저가 선택한 실행 계획에 따라 로우 소스 생성(실제 실행 계획)

- DBMS에도 SQL 실행 경로 미리보기가 있다.
  - Oracle : EXPLAIN PLAN
  - MySQL : EXPLAIN
  - SQL Server : SHOWPLAN
```sql
EXPLAIN PLAN
-------------------------
0 SELECT STATEMENT Optimizer=ALL_ROWS (Cost=4 Card=14 Bytes=560)
1 0 SORT (ORDER BY) (Cost=4 Card=14 Bytes=560)
2 1 NESTED LOOPS (Cost=3 Card=14 Bytes=560)
3 2 TABLE ACCESS (FULL) OF 'DEPT' (TABLE) (Cost=2 Card=4 Bytes=80)
4 2 TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (TABLE) (Cost=1 Card=3 Bytes=114)
5 4 INDEX (RANGE SCAN) OF 'EMP_DEPTNO_IDX' (INDEX) (Cost=1 Card=3)
```

- 옵티마이저가 특정 실행계획을 선정하는 근거
```sql
테스트 테이블 생성
SQL> create table t
    2  as
    3  select d.no, e.*
    4  from scott.emp e
    5     , (select rownum no from dual connect by level <= 1000) d;
    
인덱스 생성
SQL> create index t_x01 on t(deptno, no);
SQL> create index t_x02 on t(deptno, job, no);

t 테이블 통계 정보 확인
SQL> exec dbms_stats.gather_table_stats(user, 't');

```

### 1.1.5. 옵티마이저 힌트
- 옵티마이저 힌트의 사용법
  - /* 이 안에 힌트를 통해 데이터 엑세스 경로를 변경할 수 있다. */

```sql
SELECT /*+ INDEX_ASC(E EMP_DEPTNO_IDX) */ E.EMPNO, E.ENAME, E.JOB, D.DNAME, D.LOC
```

- 옵티마이저 힌트 목록
  - 최적화 목표
    - ALL_ROWS : 전체 처리속도 최적화
    - FIRST_ROWS : 최초 처리속도 최적화
  - 엑세스 방식
    - FULL : 풀 스캔 유도
    - INDEX : 인덱스 스캔 유도
    - INDEX_DESC : 인덱스 역순 스캔 유도
    - INDEX_FFS : 인덱스 풀 스캔 유도
    - INDEX_SS : 인덱스 스킵 스캔
  - 조인 순서
    - LEADING : 괄호에 기술한 순서대로 조인
    - ORDERED : FROM 절에 기술한 순서대로 조인
    - SWAP_JOIN_INPUTS : 해시 조인 시, 인풋을 명시적으로 선택
  - 조인 방식
    - USE_NL : 루프 조인 유도
    - USE_MERGE : 병합 조인 유도
    - USE_HASH : 해시 조인 유도
    - NL_SJ : NL 세미조인 유도
    - MERGE_SJ : MERGE 세미조인 유도
    - HASH_SJ : HASH 세미조인 유도
  - 서브쿼리 팩토링
    - MATERIALIZE : WITH 문으로 정의한 집합을 물리적으로 생성하도록 유도
    - INLINE : WITH 문으로 정의한 집합을 인라인 뷰로 변환하도록 유도
  - 쿼리 변환
    - ... 추가 작성 예정

## 1.2 SQL 공유 및 재사용
- 소프트 파싱과 하드 파싱의 차이점
  - SGA의 구성요소로 라이브러리 캐시가 있다.
  - 소프트 파싱
    - 라이브러리 캐시에 SQL 문장이 존재하면 곧바로 실행 단계로 진행
  - 하드 파싱
    - 라이브러리 캐시에 SQL 문장이 존재하지 않으면 최적화 단계 진행
    - 로우 소스 생성
    - 실행