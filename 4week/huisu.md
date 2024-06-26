# 인덱스 튜닝

## 인덱스 스캔 효율화

### BETWEEN을 IN-List로 전환

- BETWEEN을 IN 조건으로 바꾸면 효과를 보는 경우가 있음
- IN-List 개수만큼 UNION ALL 브랜치가 생성되고 각 브랜치마다 모든 칼럼을 ‘=’ 조건으로 검색
- BEFORE

    ```sql
    select 해당층, 펑당가, 입력일, 해당동, 매물구분, 연사용일수, 중개업소코드
    from 매물아파트매매
    where 아파트시세코드 = 'A01011350900056'
    and 평형 = '59'
    and 평형타입 = 'A'
    and **인터넷매물 between '1' and '3'**
    order by 입력일 desc;
    ```

- AFTER

    ```sql
    select 해당층, 평당가, 입력일, 해당동, 매물구분, 연사용일수, 중개업소코드
    from 매물아파트매매
    where **인터넷매물 in ('1' , '2' , '3')**
    and 아파트시세코드 = 'A01011350900056 '
    and 평형 = '59'
    and 평형타입 = 'A'
    order by 입력일 desc
    ```


**BETWEEN 조건을 IN-List로 전환할 때 주의 사항**

- IN-List 의 개수가 많지 않아야 함
    - 많을 경우 수직적 탐색이 많이 발생할 수 있음
- BETWEEN의 조건 범위가 좁은 경우 효과가 미비하거나 IN-List 전환 시 수직적 탐색 비용이 더 많이 나올 수 있음

### IN 조건은 ‘=’인가

- IN 조건은 ‘=’과 다름
- 데이터가 멀리 분포되어 있다면 IN-List Iterator로 바꾸어 수직적 탐색을 하는 것이 편리할 수 있으나, IN 값들이 하나의 블록 내부에 속할 경우 필터로 적용하는 것이 더욱 효율적

**NUM_INDEX_KEYS 활용**

- 여러 칼럼을 기준으로 삼은 인덱스가 있을 때 어디까지 액세스 조건으로 사용할지 지정

### 다양한 옵션 조건 처리 방식의 장단점 비교

**OR 조건 활용**

- 인덱스 선두 컬럼에 대한 옵션 조건에 OR 조건을 사용해선 안 됨
- OR 조건을 활용한 옵션 조건 처리 요약
    - 인덱스 액세스 조건으로 사용 불가
    - 인덱스 필터 조건으로 사용 불가
    - 테이블 필터 조건으로만 사용 가능
    - 단, 인덱스 구성 컬럼 중 하나 이상이 Not Null 컬럼이면, 18c부터 인덱스 필터 조건으로 사용 가능
- OR 조건을 이용한 옵션 조건 처리는 가급적 사용하지 않아야 함

**LIKE/BETWEEN 조건 활용**

- 변별력이 좋은 필수 조건이 있는 상황에서 옵션 조건 처리를 사용이 효율적일 수 있음
- 필수 조건의 변별력이 좋지 않을 때는 성능에 문제가 생김
- LIKE/BETWEEN 패턴 사용 시 아래 네 가지 경우에 속하는지 점검
    - 인덱스 선두 컬럼
        - 사용자가 인덱스 선두 컬럼 값을 입력하지 않으면 모든 데이터를 스캔하면서 필터링하는 불상사 발생
    - NULL 허용 컬럼
        - NULL 허용 컬럼이라면 결과 집합에서 누락될 수 있음
    - 숫자형 컬럼
        - LIKE 조건에 대한 자동 형 변환 문제 야기
    - 가변 길이 컬럼
        - 길이가 가변적이라면 LIKE 패턴에서 검색한 문자열이 포함된 모든 데이터가 함께 조회됨

**UNION ALL 활용**

- 옵션 조건에 값에 입력되었는지 아닌지에 따라 다른 SQL 실행
- 변수에 값이 입력되든 아니든 인덱스를 가장 최적으로 사용 가능
- SQL 코딩 량이 길어짐

### 함수 호출 부하 해소를 위한 인덱스 구성

**PS/SQL 함수의 성능적 특성**

- 일반적으로 매우 느림
    - 가상머신 상에서 실행되는 인터프리터 언어
    - 호출 시마다 컨텍스트 스위칭 발생
    - 내장 SQL에 대한 Recursive Call 발생
- 일반적으로 Recursive 부하가 가장 큼
- 함수 호출 횟수를 줄이는 액세스 조건을 고려한 인덱스 구성

**효과적인 인덱스 구성을 통한 함수 호출 최소화**

- 테이블을 Full Scan으로 읽으면 함수는 테이블 건수만큼 수행됨
- 조건절을 만족하는 만큼만 수행하도록 호출 최소화

## 인덱스 설계

### 인덱스 설계가 어려운 이유

- 인덱스가 많으면 생기는 문제점
    - DML 성능 저하 (TPS 저하)
    - 데이터베이스 사이즈 증가 (디스크 공간 낭비)
    - 데이터베이스 관리 및 운영 비용 상승

### 가장 중요한 두 가지 선택 기준

- 인덱스 스캔 방식에 있어 가장 정상적이고 일반적인 방식은 Index Range Scan
- 이를 위해서는 인덱스 선두 컬럼을 조건절에 반드시 사용
- 결합 인덱스 구성
    - 조건절에 항상 사용하거나 자주 사용하는 컬럼을 선정
    - 그렇게 선정한 컬럼 중 ‘=’ 조건으로 자주 조회되는 컬럼을 앞쪽에 배치

### 스캔 효율성 이외의 판단 기준

- 이외의 고려해야 할 효율성 판단 기준
    - 수행 빈도
        - 자주 수행하는 SQL은 최적의 인덱스 구성이 효율적
    - 업무상 중요도
    - 클러스터링 팩터
    - 데이터량
        - 적은 데이터는 굳이 인덱스를 만들 이유 없음
    - DML 부하 (기존 인덱스 개수, 초당 DML 발생량, 자주 갱신하는 컬럼 포함 여부)
    - 저장 공간
    - 인덱스 관리 비용

### 공식을 초월한 전략적 설계

- 여러 개 중 최적을 달성해야 할 가장 핵심적인 액세스 경로 한두 개를 전략적으로 선택해서 최적 인덱스 설계
- 나머지 액세스 경로는 약간의 비효율이 있더라도 목표 성능을 만족하는 수준으로 인덱스 구성
- 단순한 공식에 의한 결정이 아니라 업무 상황을 이해하고 나름의 판단 기준으로 결정

### 소트 연산을 생략하기 위한 컬럼 추가

- 조건절에 사용하지 않는 컬럼이라도 소트 연산을 생략할 목적으로 인덱스 구성에 포함
- I/O를 최소화하면서도 소트 연산을 생략하는 방법
    - ‘=’ 연산자로 사용한 조건절 컬럼 선정
    - ORDER BY 절에 기술한 컬럼 추가
    - ‘=’ 연산자가 아닌 조건절 컬럼은 데이터 분초를 고려해 추가 여부 결정
- IN 조건은 ‘=’이 아니기에 소트 연산 생략을 위해 대체할 수 없음

### 결합 인덱스 선택도

- 선택도: 전체 레코드 중에서 조건절에 의해 선택되는 레코드 비율
- 인덱스 선택도: ‘=’ 로 조회할 때 평균적으로 선택되는 비율
- 선택도가 높은 인덱스는 생성해 봤자 높은 효용 가치를 가지지 않음
- 테이블 액세스가 많이 발생하기 때문
- 컬럼 간 순서를 결정할 때는 각 컬럼의 선택도보다 필수 조건 여부, 연산자 형태가 더 중요한 판단 기준