# SQL 파싱과 최적화

이 장에서는 간단히 SQL을 어떻게 처리하는지, 서버 프로세스는 데이터를 어떻게 읽고 저장하는지에 대한 내용을 짚고 넘어간다. 


### 1. 구조적, 집합적, 선언적 질의 언어
두 테이블을 조인한다고 가정하자.

    select E.empno,E.ename, E.job,D.dname,D.loc
    from EMP E, DEPT D
    where E.deptno = D.deptno
    order by E.ename
해당 sql은 DB SQL을 배운 사람이라면 간단하게 짤 수 있을 것이다. 하지만 DB가 없던 시절 파일시스템으로 데이터를 관리할 떄는 해당 내용을 어렵게 프로그래밍했었다. 

SQL은 **Structured Query Lanhuage**이다. 즉, "구조적"인 언어라는 뜻이다. 
프로그래밍을 지원하는 확장언어도 존재하지만, 기본적으로 **구조적**이고 **집합적**이고 **선언적**인 질의 언어이다.

원하는 결과집합을 구조적, 집합적으로 선언한다. 하지만 그 결과를 만들어내는 과정은 절차적이다.
즉, 프로시저가 필요하다. 프로시저를 만드는 엔진이 **SQL 옵티마이저**이다. 
옵티마이저가 프로그래밍을 대신해준다고 생각하면된다.
프로시저작성 후 컴파일해서 실행가능한 상태로 만든 과정을 "SQL 최적화"라고한다. 
### 2.  SQL 최적화
아래 내용은 최적화 과정의 세분화.
1. SQL 파싱
SQL을 전달받으면 SQL Parser가 Parsing을 진행한다. SQL Parsing을 요약하면 아래와 같다.
* 파싱트리 생성 : SQL문에 개별 구성요소를 분석해서 파싱트리를 만든다.
* Syntax 체크 : SQL 안에 문법적 오류, 사용할 수 없는 키워드, 순서 등을 확인
* Semantic 체크 : 의미상 오류체크(Ex. 존재하지않는 테이블 및 컬럼), 권한 체크

2. SQL 최적화
옵티마이저가 최적화를 맡는다. 수집한 시스템 및 오브젝트 통계정보를 바탕으로 실행경로를 생성한다. 다양한 경로를 비교하여 가장 효율적인 하나를 선택한다. 성능을 결정하는 핵심적인 엔진이다.

3. 로우 소스 생성
로우 소수 생성기(Row Source Generator)가 한다. 옵티마이저가 선택한 실행경로를 코드 또는 프로시저로 포맷팅한다.

### 3. SQL 옵티마이저
SQL 옵티마이저는 최적의 데이터 엑세스 경로를 선택해주는 DBMS의 핵심 엔진이다.
최적화 단계는 다음과 같다.
1. 쿼리를 수행하는 데 후보군이 될만한 실행계획들을 찾아낸다.
2. 데이터 딕셔너리에 미리 수집해 둔 오브젝트 통계 및 시스템 통계정보를 이용해 각 실행계획의 예상비용을 산정한다.
3. 최저 비용을 나타내는 실행계획을 선택한다.

### 4  실행계획과 비용
옵티마이저가 선택한 실행경로가 마음에 들지 않으면 검색모드를 변경, 경유지 추가를 통해 원하는 경
로로 유도할 수 있다.
DBMS에는 SQL 실행경로 미리보기 기능이 있다. 이것을 **실행계획**이라 한다. 실행을 처리하는 순서대로 보기 좋게 표현하여 사용자가 확인 할 수 있게 아래와 같이 트리 구조로 표현한다.
 

	Execution Plan
	----------------------------------------------------------------
	0		SELECT STATEMENT Optimizer=ALL_ROWS (Cost=209 Card=5 Bytes=175)
	1	0		TABLE ACCESS (BY INDEX ROWID) OF 'EMP' (Cost=2 Card=5 Bytes=85)
	2	1			NESTED LOOPS (Cost=209 Card=5 Bytes=175)
	3	2				TABLE ACCESS (BY INDEX ROWID) OF 'DEPT' (Cost=207 Card=1 Bytes=18)
	4	3					INDEX (RANGE SCAN) OF 'DEPT_LOC_IDX' (NON-UNIQUE) (Cost=7 Card=1)
	5	2			INDEX (RANGE SCAN) OF 'EMP_DEPTNO_IDX' (NON-UNIQUE) (Cost=1 Card=5)
\+ 실행계획은 앞으로 적지는 않을 것. 또한 앞으로 실습에 관련된 내용은 개념적인 내용만 서술할 예정입니다.
이러한 실행계획을 통해 테이블을 스캔하는지 인덱스를 스캔하는지, 어떤 인덱스인지 확인할 수 있다.
실행계획을 보고 의도와 다른 실행계획을 가지고 실행을 할 경우 실행경로를 바꿀 수있다.


어떠한 select문을 실행했을 때 T_X01,T_X02,f테이블 full 스캔등 여러가지가 있는데 어떠한 인덱스를 선택하여 실행계획을 수립한 근거는 무엇일까?
만약 T_X01을 선택하여 실행경로가 생성되었을 경우, 힌트를 사용해 T_X02와 Full Scan을 강제적으로 사용하여 실행계획을 보면 Cost의 값이 T_X01보다 높은 것을 볼 수 있다.
  
* Cost는 [실행계획](#4--실행계획과-비용)을 보면 Cost= 이라고 표시가 되어있다.

Cost는 예상하는I/O횟수 또는 예상 소요시간이다.

하지만 **예상**인 것이 중요하다. Cost는 언제나 예상치다. 여러 통계정보를 활용해 계산해 낸 값이다.
~~Cost에 대해 더 정확한 의미는 Ch7.1에서 다룬다. 추후 링크를 달겠다.~~

### 5. 옵티마이저 힌트
옵티마이저가 선택한 그 경로는 보통은 좋은 경로이지만 항상 최선의 결과는 아니다. SQL이 복잡할수록 실수할 가능성도 크다. ~~옵티마이저가 한계를 보이는 이유는 Ch7.2.4에서 설명한다. 추후 링크를 달겠다.~~
통계정보로는 알수 없는 데이터 또는 업무 특성을 활용해 개발자가 직접 더 효율적인 경로를 선택할 수 있다. 바로 **힌트**이다. 힌트를 이용하면 엑세스 경로를 바꿀 수 있다.
사용법은 아래와 같다.

	select /*+ INDEX(A 고객_PK)*/
		고객명,연락처, 주소, 가입일시
	from 고객 A
	where 고객ID = '0001'

	
/*+ INDEX(A 고객_PK)\*/ 이 부분이 바로 힌트를 사용한 것이다.


#### 힌트 사용 시 주의사항
힌트 안에 인자를 나열할 땐 콤마(,)를 사용할 순있지만, 힌트와 힌트 사이에 사용하면 안된다.

테이블을 지정할 때 스키마 명까지 지정하면 안된다.

FROM절에 ALIAS명을 지정했다면 힌트에도 무조건 ALLIAS명을 써줘야한다.

#### 자율이냐 강제냐, 그것이 문제
힌트를 사용한다면 빈틈없이 기술해라.
중대한 시스템일 수록 힌트를 빠짐없이 기입하여 옵티마이저의 개입을 최소화하자

#### 자주 사용하는 힌트 목록
  
  

|분류|힌트|설명|
|--|--|--|
|최적화 목표|ALL_ROWS|전체 처리속도 최적화|
||FIRST_ROWS (N)|최초 N건 응답속도 최적화|
|액세스 방식|FULL|Table Full Scan으로 유도|
||INDEX|Index Scan으로 유도|
||INDEX DESC|Index를 역순으로 스캔하도록 유도|
||INDEX_FFS|Index Fast Full Scan으로 유도|
||INDEX_ SS|Index Skip Scan으로 유도|
|조인순서|ORDERED|FROM 절에 나열된 순서대로 조인|
||LEADING|LEADING 힌트 괄호에 기술한 순서대로 조인(예) LEADING(T1 T2)|
||SWAP_JOIN_INPUTS|해시 조인 시, BUILD INPUT을 명시적으로 선택(예) SWAP_JOIN_INPUTS (T1)|
||USE_NL|NL 조인으로 유도|
|조인방식|USE_MERGE|소트 머지 조인으로 유도|
||USE_HASH|해시 조인으로 유도|
||NL_SJ|NL 세미조인으로 유도|
||MERGE_SJ|소트 머지 세미조인으로 유도|
||HASH_Sj|해시 세미조인으로 유도|
|서브쿼리 팩토링|MATERIALIZE|WITH 문으로 정의한 집합을 물리적으로 생성하도록 유도 <br>(예) WITH /*+ MATERIALIZE */ T AS ( SELECT ... )|
||INLINE|WITH 문으로 정의한 집합을 물리적으로 생성하 지 않고 INLINE 처리하도록 유도<br>(예) WITH /*+ INLINE */ T AS ( SELECT ... )|
|쿼리 변환|MERGE|뷰 머징 유도|
||NO MERGE|뷰 머징 방지|
||UNNEST|서브쿼리 Unnesting 유도|
||NO_UNNEST|서브쿼리 Unnesting 방지|
||PUSH_ PRED|조인조건 Pushdown 유도|
||NO_PUSH_PRED|조인조건 Pushdown 방지|
||USE_CONCAT|OR 또는 IN-List 조건을 0R-Expansion으로 유도|
||NO_EXPAND|OR 또는 IN-List 조건에 대한 OR-Expansion 방지|
|병렬 처리|PARALLEL|테이블 스캔 또는 DML을 병렬방식으로 처리하도록 유도<br>(예) PARALLEL(T1 2) PARALLEL (T2 2)|
||PARALLEL_INDEX|인덱스 스캔을 병렬방식으로 처리하도록 유도|
||PO DISTRIBUTE|병렬 수행 시 데이터 분배 방식 결정<br>(예) PO_DISTRIBUTE(T1 HASH HASH)|
|기타|APPEND|Direct-Path Insert 로 유도|
||DRIVING_SITE|DB Link Remote 쿼리에 대한 최적화 및 실행 주 체 지정(Local 또는 Remote)|
||PUSH_ SUBQ|서브쿼리를 가급적 빨리 필터링하도록 유도|
||NO_PUSH_ SUBQ|서브쿼리를 가급적 늦게 필터링하도록 유도|

\+ 사진을 OCR로 옮겨적어서 틀릴 수 있습니다.