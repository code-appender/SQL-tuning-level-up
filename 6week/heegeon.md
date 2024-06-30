# 5. 소트 튜닝
 
## 5.1 소트 연산에 대한 이해

### @ 소트 수행 과정
소트는 기본적으로 PGA에 할당한 Sort Area에서 수행하며 PGA가 부족하면 Temp TableSpace에 저장하여 수행한다.   <br>
메모리 공간인 Sort Area가 다 차면 디스크 Temp 테이블스페이스를 활용한다.

#### 소트 과정
1. 소트할 대상 집합을 SGA 버퍼캐시를 통해 읽어들이고 일차적으로 Sort Area에서 정렬을 시도한다.
2. Sort Area에 모두 저장할 수 없을 경우 Temp TableSpace에 임시 세그먼트를 만들어 저장한다.
3. 정렬된 최종 결과집합을 얻기 위해 머지를 진행한다.

소트는 최대한 발생하지 않도록 SQL을 작성해야 한다.

### @ 소트 오퍼레이션
1. Sort Aggregate
- 전체 로우를 대상으로 집계를 수행할 때 나타난다.
```sql
-- Sort Area에 변수를 하나 만들어 데이터를 읽으면서 최대값 비교하며 최대값을 찾아 리턴 
SELECT MAX(SAL) FROM 직원;
```
- MAX, MIN, SUM, AVG 함수를 사용할 때 나타난다.
- 실제로 데이터를 정렬하지는 않고 Sort Area를 사용한다는 의미로 실행계획에 Sort Aggregate라고 표시된다.

2. Sort Order By
- 데이터를 정렬할 때 나타난다.
```sql
SELECT * FROM 직원 ORDER BY SAL DESC;
```

3. Sort Group By
- 소트 알고리즘을 사용해 그룹별 집계를 수행할 때 나타난다.
```sql
-- 부서번호를 기준으로 ORDER BY 하므로 SORT GROUP BY 오퍼레이션 사용
SELECT DEPT_NO, MAX(SAL) FROM 직원 GROUP BY DEPT_NO ORDER BY DEPT_NO;

-- ORDER BY가 없으므로 HASH GROUP BY 오퍼레이션 사용
SELECT DEPT_NO, MAX(SAL) FROM 직원 GROUP BY DEPT_NO;
```
- GROUP BY 절에 ORDER BY가 포함되어 있지 않으면 HASH GROUP BY 오퍼레이션을 사용한다.
- 정렬된 그룹핑 결과를 얻고자 한다면 실행계획에 Sort Group By라고 표시되더라도 반드시 Order By를 명시해야 한다.

4. Sort Unique
- 서브쿼리와 메인 쿼리가 조인하기 전에 중복 레코드를 제거하는 오퍼레이션
```sql
-- 서브쿼리의 중복을 제거하기 위해 발생
SELECT /*+ ordered use_nl(dept) */
	*
FROM DEPT
WHERE
	DEPT_NO IN (SELECT /*+ unnest */ DEPT_NO FROM EMP WHERE JOB = '개발자');


-- UNION 하여 중복을 제거하기 위해 발생
SELECT JOB FROM EMP WHERE DEPT_NO = 10
UNION
SELECT JOB FROM EMP WHERE DEPT_NO = 20
```
- PK/Unique 제약 또는 Unique 인덱스를 통해 Unnesting된 서브쿼리의 유일성이 보장된다면 Sort Unique는 생략된다.
- Union, Minus, Intersect, Distinct 같은 연산자에서도 중복 제거를 위해 사용된다.

5. Sort Join
- 소트 머지 조인을 수행할 떄 발생한다.

6. Window Sort
- 윈도우 함수를 수행할 떄 나타난다.

## 5.2 소트가 발생하지 않도록 튜닝
Union, Minus, Distinct 연산자는 중복 레코드를 제거하기 위해 소트 연산을 발생시키므로 꼭 필요한 경우에만 사용해야 하며 성능이 느리다면 소트 연산을 피하는 방법을 찾아야 한다.

### @ Union vs Union All
SQL에 Union을 사용하면 옵티마이저는 상단과 하단 두 집합 간 중복을 제거하기 위해 소트 작업을 수행한다.  <br>
반면, Union All은 중복을 확인하지 않고 두 집합을 단순히 결합하므로 소트 작업을 수행하지 않는다.
따라서 될 수 있으면 Union All을 사용해야 한다.

### @ Exists 활용
중복 레코드를 제거할 목적으로 Distinct 연산자를 종종 사용하는데 이 연산자를 사용하면 조건에 해당하는 데이터를 모두 읽어서 중복을 제거해야 한다.
<br>부분범위 처리가 불가능하며 모든 데이터를 읽는 과정에 많은 I/O가 발생한다.

Exist 서브쿼리는 데이터 존재여부만 확인하면 되기 떄문에 조건절을 만족하는 데이터를 모두 읽지 않는다.
<br>또한, Distinct, Minus 연산자를 사용한 쿼리는 대부분 Exists로 변환이 가능하다.

### @ 인덱스를 이용한 소트 연산 생략
인덱스는 항상 키 컬럼 순으로 정렬된 상태를 유지하기 때문에 Order By 또는 Group By 절이 있더라도 소트 연산을 생략할 수 있다.

### @ Top N 쿼리
전체 결과 집합 중 상위 N개 레코드만 선택하는 쿼리이다.
- 인덱스를 탄다면 부분 범위 처리로 딱 원하는 갯수만 만나면 멈춘다.
- 실행 계획에는 COUNT(STOPKEY)로 표시되며 이를 Top N StopKey라고 칭한다.
- 페이징 처리를 할 떄 TOP N 쿼리를 사용하는데 반드시 아래에 ROWNUM을 사용해야 한다. (페이징 처리 ANTI 패턴)
  - 이유는 ROWNUM이 TOP N StopKey 알고리즘을 작동하게 하는 열쇠이기 떄문이다.
  - ROWNUM을 사용하지 않으면 소트생략은 가능하지만 StopKey가 작동하지 않아 전체범위를 처리하게 된다.

### @ 최소값/최대값 구하기
최소값, 최대값을 구하는 SQL 실행게획을 보면 Sort Aggregate 오퍼레이션을 사용한다.
<br>인덱스를 이용하면 전체 데이터를 읽지 않고도 최소 또는 최대값을 쉽게 찾을 수 있다.

그러기 위해서는 조건절 컬럼과 MIN/MAX 함수 인자 컬럼이 모두 인덱스에 포함되어야 한다.

#### TOP N 쿼리를 이용해 최소/최대값 구하기
```sql
CREATE INDEX EMP_X1 ON EMP(DEPTNO, SAL);

SELECT *
FROM (
    SELECT  SAL
    FROM    EMP
    WHERE   DEPTNO = 10
    AND     MGR = 7698
    ORDER BY SAL DESC
     )
WHERE ROWNUM <= 1;
```
Top N 쿼리에서 작동하는 Top N StopKey 알고리즘은 모든 컬럼이 인덱스에 포함되지 않더라도 작동한다.
<br>위의 SQL에서도 MGR 컬럼이 인덱스에 없지만 가장 큰 SAL 값을 찾기 위해 DEPTNO=10조건을 만족하는 전체 레코드를 읽지 않았으며 MGR=7698을 만족하는 레코드 하나를 찾으면 바로 멈춘다.

## 5.4 Sort Area를 적게 사용하도록 SQL 작성
- 소트 데이터 줄이기
```sql
-- 모든 컬럼을 SORT AREA에 담으므로 아래보다 느리다.
SELECT * FROM 직원 ORDER BY 나이;

-- 이름만 SORT AREA에 담으므로 위보다 빠르다.
SELECT 이름 FROM 직원 OERDER BY 나이;
```

