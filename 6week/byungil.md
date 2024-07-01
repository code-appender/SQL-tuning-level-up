# 5. 소트 튜닝

# 5.1 소트 연산에 대한 이해

- SQL 수행 도중 가공된 데이터 집합이 필요할 때, 오라클은 PGA와 TEMP 테이블스페이스를 활용

## 5.1.1 소트 수행 과정

- PGA에 할당한 Sort Area에서 이루어진다.
- 메모리 공간인 Sort Area가 다 차면, 디스크 Temp 테이블스페이스를 활용한다.
    - 조인 튜닝과 똑같은 공간을 활용한다.
- Sort Area에서 작업을 완료할 수 있는지에 따라 소트를 두 가지 유형으로 나눈다.
    - 메모리 소트(In-Memory Sort)
        - 전체 데이터의 정렬 작업을 메모리 내에서 완료하는 것을 말하며, ‘Internal Sort’라고도 한다.
    - 디스크 소트(To-Disk Sort)
        - 할당받은 Sort Area 내에서 정렬을 완료하지 못해 디스크 공간까지 사용하는 경우를 말하며, ‘External Sort’라고도 한다.

### 디스크 소트(To-Disk Sort) 과정

- 소트할 대상 집합을 SGA 버퍼캐시를 통해 읽어들이고, 일차적으로 Sort Area에서 정렬을 시도한다.
- Sort  Area 내에서 데이터 정렬을 마무리하는 것이 최적이지만, 양이 많을 때는 정렬된 중간집합을 Temp 테이블스페이스에 임시 세그먼트를 만들어 저장한다.
    - Temp 영역에 저장해 둔 중간 단계의 집합을 ‘Sort Run’이라고 부른다.
- 정렬된 최종 결과집합을 얻으려면 이를 다시 Merge 해야 한다.
- 각 Sort Run 내에서는 이미 정렬된 상태이므로 Merge 과정은 어렵지 않다.
- 오름차순 정렬이라면 각각에서 가장 작은 값부터 PGA로 읽어 들이다가 PGA가 찰 때마다 쿼리 수행 다음 단계로 전달하거나 클라이언트에게 전송하면 된다.

### 소트 연산 특징

- 소트 연산은 메모리 집약적일 뿐만 아니라 CPU 집약적이기도 하다.
- 처리할 데이터량이 많을 때는 디스크 I/O까지 발생하므로 쿼리 성능을 좌우하는 매우 중요한 요소다.
- 디스크 소트가 발생하는 순간 SQL 수행 성능은 나빠질 수밖에 없다.
- `부분범위 처리를 불가능하게 함으로써 OLTP 환경에서 애플리케이션 성능을 저하시키는 주요인`이 되기도 한다.
- 소트가 발생하지 않도록 SQL을 작성해야 하고, 소트가 불가피하다면 메모리 내에서 수행을 완료할 수 있도록 해야 한다.

## 5.1.2 소트 오퍼레이션

- 소트를 발생시키는 오퍼레이션에 어떤 것들이 있는지부터 살펴보자.

### (1) Sort Aggregate

- Sort Aggregate는 전체 로우를 대상으로 집계를 수행할 때 나타난다.
- ‘Sort’라는 표현을 사용하지만, 실제로 데이터를 정렬하진 않는다. Sort Area를 사용한다는 의미로 이해하면 된다.
- 데이터를 정렬하지 않고 SUM, MAX, MIN, AVG 값 구하는 절차
    - Sort Area에 SUM, MAX, MIN, COUNT 값을 위한 변수를 각각 하나씩 할당한다.
    - 그림 5-2처럼 EMP 테이블 첫 번째 레코드에서 읽은 SAL 값을 SUM, MAX, MIN 변수에 저장하고, COUNT 변수에는 1을 저장한다.

  ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/e9e781a0-6e9f-43d7-b0b2-1cbfa0ecd760/dbf32f88-2e42-4b75-90fd-b29ab2bedad5/Untitled.png)

    - EMP 테이블에서 레코드를 하나씩 읽어 내려가면서 SUM 변수에는 값을 누적하고, MAX 변수에는 기존보다 큰 값이 나타날 때마다 값을 대체하고, MIN 변수에는 깆노보다 작은 값이 나타날 때마다 값을 대체한다. COUNT 변수에는 SAL 값이 NULL이 아닌 레코드를 만날 때마다 1씩 증가시킨다.

  ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/e9e781a0-6e9f-43d7-b0b2-1cbfa0ecd760/1f5e7e76-8c54-4a21-96ad-76c2ec655ac8/Untitled.png)

    - EMP 레코드를 다 읽고 나면 그림 5-3처럼 SUM, MAX, MIN, COUNT 변수에 각각 14000, 5000, 1000, 5가 저장돼 있다. SUM, MAX, MIN 값은 변수에 담긴 값을 그대로 출력하고, AVG는 SUM 값을 COUNT로 나눈 2800을 출력하면 된다.

### (2) Sort Order By

- 데이터를 정렬할 때 나타난다.

### (3) Sort Group By

- Sort Group By는 소팅 알고리즘을 사용해 그룹별 집계를 수행할 때 나타난다.

```sql
select deptno, sum(sal), max(sal), min(sal), avg(sal)
from emp
group by deptno
order by deptno
```

- 부서는 네 개뿐이며, 부서코드로는 각각 10, 20, 30, 40을 사용한다.


- 사원의 급여 정보를 읽는다.
- 읽은 각 사원의 부서번호에 해당하는 메모지를 찾아 SUM, MAX, MIN, COUNT 값을 갱신한다. (Sort Aggregate이랑 같은 방식)
- 부서 개수를 미리 알 수 없다면, 새로운 부서가 나타날 때마다 새로 준비한 메모지를 정렬 순서에 맞춰 중간에 끼워 넣는 방식을 사용해야 한다.
- 부서가 많지 않다면 Sort Area가 클 필요가 전혀 없다. 집계할 대상 레코드가 아무리 많아도 Temp 테이블스페이스를 쓰지 않는다는 뜻이다.

### (4) Hash Group By


- Group By 절 뒤에 Order By 절을 명시하지 않으면 이제 대부분 Hash Group By 방식으로 처리한다.
- Sort Group By에서 메모지를 찾기 위해 소트 알고리즘을 사용했다면, Hash Group By는 해싱 알고리즘을 사용한다.
- 읽는 레코드마다 Group By 컬럼의 해시 값으로 해시 버킷을 찾아 그룹별로 집계항목을 갱신하는 방식이다.
- 부서(그룹 개수)가 많지 않다면 집계할 대상 레코드가 아무리 많아도 Temp 테이블스페이스 쓸 일이 없다.

### 그룹핑 결과의 정렬 순서

- Sort Group By의 의미는 소팅 알고리즘을 사용해 값을 집계한다 라는 뜻일 뿐이며 정렬을 의미하지는 않는다.
- 정렬된 그룹핑 결과를 얻고자 한다면 Order By를 명시해야 한다.
- Order By 절을 추가한다고 해서 성능에 차이가 생기는 것은 아니다.

### (4) Sort Unique

- 옵티마이저가 서브쿼리를 풀어 일반 조인문으로 변환하는 것을 ‘서브쿼리 Unnesting’이라고 한다.
- Unnesting된 서브쿼리가 M쪽 집합이면(1쪽 집합이더라도 조인 컬럼에 Unique 인덱스가 없으면), 메인 쿼리와 조인하기 전에 중복 레코드부터 제거해야 한다.
- 이때 Sort Unique 오퍼레이션이 나타난다.

### (5) Sort Join

- Sort Join 오퍼레이션은 소트 머지 조인을 수행할 때 나타난다.

### (6) Window Sort

- Window Sort는 윈도우 함수를 수행할 때 나타난다.

---

# 5.2  소트가 발생하지 않도록 SQL 작성

- SQL 작성할 때 불필요한 소트가 발생하지 않도록 주의해야 한다.
- Union, Minus, Distinct 연산자는 중복 레코드를 제거하기 위한 소트 연산을 발생시키므로 꼭 필요한 경우에만 사용하고, 성능이 느리다면 소트 연산을 피할 방법이 있는지 찾아봐야 한다.
- 조인 방식도 잘 선택해 줘야 한다.

## 5.2.1 Union vs Union All

- Union
    - Union을 사용하면 옵티마이저는 상단과 하단 두 집합 간 중복을 제거하려고 소트 작업을 수행한다.
- Union All
    - 중복을 확인하지 않고 두 집합을 단순히 결합하므로 소트 작업을 수행하지 않는다.
- 될 수 있으면 Union All을 사용해야 한다.
- Union을 Union All로 변경하려다 자칫 결과 집합이 달라질 수 있으므로 주의해야 한다.

### 왜 Union All을 사용해야 될까?

- Union을 사용함으로 인해 소트 연산을 발생시킨다.
    - Union All로 변경하자.
- 소트 연산이 일어나지 않도록 Union All을 사용하면서도 데이터 중복을 피하려면
    - 조건절을 추가하여 해결한다.

## 5.2.2 Exists 활용

- 중복 레코드를 제거할 목적으로 Distinct 연산자를 종종 사용하는데, 이 연산자를 사용하면 조건에 해당하는 데이터를 모두 읽어서 중복을 제거해야 한다.
- 부분범위 처리는 당연히 불가능하고, 모든 데이터를 읽는 과정에 많은 I/O가 발생한다.
- Exists 서브쿼리는 데이터 존재 여부만 확인하면 되기 때문에 조건절을 만족하는 데이터를 모두 읽지 않는다.
- Distinct, Minus 연산자를 사용한 쿼리는 대부분 Exists 서브쿼리로 변환 가능하다.

## 5.2.3 조인 방식 변경

- 인덱스를 이용하여 소트 연산을 생략할 수 있지만, 해시 조인이기 때문에 Sort Order By가 나타났다.
- NL 조인하도록 조인 방식을 변경하면 소트 연산을 생략할 수 있어 부분범위 처리가 가능한 솽황에서 큰 성능 개선 효과를 얻을 수 있다.
- 정렬 기준이 조인 키 컬럼이면 소트 머지 조인도 Sort Order By 연산을 생략할 수 있다.

---

# 5.3 인덱스를 이용한 소트 연산 생략

- 인덱스는 항상 키 컬럼 순으로 정렬된 상태를 유지한다.
- 이를 활용하면 SQL에 Order By 또는 Group By 절이 있어도 소트 연산을 생략할 수 있다.
- 여기에 Top N 쿼리 특성을 결합하면, 온라인 트랜잭션 처리 시스템에서 대량 데이터를 조회할 때 매우 빠른 응답 속도를 낼 수 있다.
- 특정 조건을 만족하는 최소값 또는 최대값도 빨리 찾을 수 있어 이력 데이터를 조회할 때 매우 유용하다.

## 5.3.1 Sort Order By 생략

- 인덱스 선두 컬럼을 (종목코드 + 거래일시) 순으로 구성하지 않으면, 아래 쿼리에서 소트 연산을 생략할 수 없다.

```sql
select 거래일시, 체결건수, 체결수량, 거래대금
from 종목거래
where 종목코드 = 'KR123456'
order by 거래일시
```

- 인덱스 선두 컬럼을 (종목코드 + 거래일시)순으로 구성하면 소트 연산을 샐략할 수 있다.
- 소트 연산을 생략함으로써 종목코드 = ‘KR123456’ 조건을 만족하는 전체 레코드를 읽지 않고도 바로 결과집합 출력을 시작할 수 있게 되었다.
- 즉, 부분범위 처리 가능한 상태가 되었다.
- 소트해야 할 대상 레코드가 무수히 많은 상황에서 극적인 성능 개선 효과를 얻을 수 있다.

### 부분범위 처리를 활용한 튜닝 기법, 아직도 유효한가?

- 부분범위 처리는 쿼리 수행 결과 중 앞쪽 일부를 우선 전송하고 멈추었다가 클라이언트가 추가 전송을 요청(그리드 스크롤 또는 ‘다음’ 버튼 클릭을 통한 Fetch Call)할 때마다 남은 데이터를 조금씩 나눠 전송하는 방식을 말한다.
- 클라이언트와 DB 서버 사이에 WAS, AP 서버 등이 존재하는 3-Tier 아키텍처는 서버 리소스를 수많은 클라이언트가 공유하는 구조이므로 클라이언트가 특정 DB 커넥션을 독점할 수 없다.
- 단위 작업을 마치면 DB 커넥션을 바로 커넥션 풀에 반환해야 하므로 그 전에 쿼리 조회 결과를 클라이언트에게 ‘모두’ 전송하고 커서를 닫아야만 한다.
- 따라서 `쿼리 결과 집합을 조금씩 나눠서 전송하는 방식을 사용할 수 없다.`
- 부분범위 처리 활용은 첫째, 결과집합 출력을 바로 시작할 수 있느냐
- 둘째, 앞쪽 일부만 출력하고 멈출 수 있느냐가 핵심이므로 3-Tier 환경에서 의미 없다고 생각할 만하다.
- 하지만, 부분범위 처리 원리는 3-Tier 환경에서도 여전히 유효하다. `비밀은 바로 Top N 쿼리에 있다.`

## 5.3.2 Top N 쿼리

- Top N 쿼리는 전체 결과집합 중 상위 N개 레코드만 선택하는 쿼리다.
- 인덱스를 이용하면, 옵티마이저는 `소트 연산을 생략`하며 인덱스를 스캔하다가 N개 레코드를 읽는 순간 바로 멈춘다.
- `Top N StopKey` 알고리즘이라고 부른다.

### 페이징 처리

- 3-Tier 환경에서 부분범위 처리를 응용할 수 있는 방법 - 페이징 처리
- `부분범위 처리` 가능하도록 SQL을 작성한다.
    - 인덱스 사용 가능하도록 조건절을 구사하고, 조인은 NL 조인 위주로 처리한다.
    - Order By 절이 있어도 소트 연산을 생략할 수 있도록 인덱스를 구성해주는 것을 의미한다.

## 5.3.3 최솟값/최댓값 구하기 (인덱스 활용)

- 최솟값 또는 최댓값을 구하는 SQL 실행계획을 보면, Sort Aggregate 오퍼레이션이 나타난다.
- Sort Aggregate를 위해 전체 데이터를 정렬하진 않지만, 전체 데이터를 읽으면서 값을 비교한다.
- 인덱스는 정렬돼 있으므로 전체 데이터를 읽지 않고도 최소 또는 최댓값을 쉽게 찾을 수 있다.

### 인덱스를 이용해 최소/최댓값 구하기 위한 조건

- 조건절 컬럼과 MIN/MAX 함수 인자 컬럼이 모두 인덱스에 포함돼 있어야 한다.
- 즉 테이블 액세스가 발생하지 않아야 한다.
- First Row Stopkey 알고리즘을 이용해야 효율적으로 구할 수 있다.

### Top N 쿼리를 이용해 최소/최댓값 구하기

- Top N 쿼리에 작동하는 ‘Top N Stopkey’ 알고리즘은 모든 컬럼이 인덱스에 포함돼 있지 않아도 잘 작동한다.
- 성능 측면에서는 MIN/MAX 쿼리 보다 낫다.

## 5.3.4 이력 조회

## 5.3.5 Sort Group By 생략

- 인덱스를 이용해 소트 연산을 생략할 수 있다.
- 그룹핑 연산에도  인덱스를 활용할 수 있다.
- region이 선두 컬럼인 인덱스를 이용하면, Sort Group By 연산을 생략할 수 있다.
- 실행계획에 ‘Sort Group By `Nosort`'라고 표시된 부분을 확인하자.

```sql
select region, avg(age), count(*)
from customer
group by region
```