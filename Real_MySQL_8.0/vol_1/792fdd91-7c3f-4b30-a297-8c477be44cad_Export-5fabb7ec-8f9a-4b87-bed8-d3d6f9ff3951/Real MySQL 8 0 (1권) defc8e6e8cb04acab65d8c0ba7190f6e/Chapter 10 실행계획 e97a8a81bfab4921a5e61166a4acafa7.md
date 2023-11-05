# Chapter 10 실행계획

# 10.1 통계 정보

대부분의 DBMS는 많은 데이터를 안전하게 저장 및 관리하고 사용자가 원하는 데이터를 빠르게 조회할 수 있게 처리하는 것이 주목적

옵티마이저가 사용자의 쿼리를 최적으로 처리될 수 있게 하는 쿼리의 실행 계획을 수립할 수 있어야 함

하지만 관리자나 사용자의 개입 없이 항상 좋은 실행 계획을 만들어낼 수 있는 것은 아님

이러한 문제점을 관리자나 사용자가 보완할 수 있도록 **EXPLAIN** 명령으로 옵티마이저가 수립한 실행 계획을 확인할 수 있게 해줌

## 10.1.1 테이블 및 인덱스 통계 정보

비용 기반 최적화에서 가장 중요한 것은 통계 정보

통계 정보가 정확하지 않다면 전혀 엉뚱한 방향으로 쿼리를 실행함

MySQL은 다른 DBMS보다 통계 정보의 정확도가 높지 않고 휘발성이 강함

쿼리의 실행 계획을 수립할 때 실제 테이블의 데이터를 일부 분석해서 통계 정보를 보완해서 사용했음

5.6 버전부터는 통계 정보의 정확성을 높일 수 있는 방법이 제공되기 시작

### 10.1.1.1 MySQL 서버의 통계 정보

### **~ MySQL 5.5**

각 테이블의 통계 정보가 메모리에만 관리됨

SHOW INDEX 명령으로만 테이블의 인덱스 컬럼의 분포도를 볼 수 있었음

통계 정보가 메모리에 관리될 경우 MySQL 서버가 재시작되면 지금까지 수집된 통계 정보가 모두 사라짐

**통계 정보가 갱신되는 순간**

1. 테이블이 새로 오픈되는 경우
2. 테이블의 레코드가 대량으로 변경되는 경우(전체 레코드 중 1/16)
3. **ANALYZE TABLE** 명령 실행
4. **SHOW TABLE STATUS** or **SHOW INDEX FROM** 명령 실행
5. InnoDB 모니터 활성화
6. **innodb_stats_on_metadata** 시스템 설정이 ON인 상태에서 **SHOW TABLE STATUS** 명령 실행

자주 테이블의 통계 정보가 갱신되면 응용 프로그램의 쿼리를 인덱스 레인지 스캔으로 잘 처리하던 것을 풀 테이블 스캔으로 실행되는 상황이 발생

⇒ 영구적인 통계 정보 도입으로 개선됨

### MySQL **5.6 ~**

InnoDB 스토리지 엔진을 사용하는 테이블에 대한 통계 정보를 영구적으로 관리할 수 있게 개선

각 테이블의 정보를 mysql 데이터베이스의 innodb_index_stats 테이블과 innodb_table_stats 테이블로 관리할 수 있게됨

통계 정보를 테이블로 관리함으로써 MySQL 서버가 재시작돼도 기존의 통계 정보 유지 가능

**테이블 생성시 STATS_PERSISTENT 옵션 설정 가능**

설정값에 따라 테이블 단위로 영구적인 통계 정보를 보관할지 말지를 결정

```sql
CREATE TABLE tab_table (fb1 INT, fb2 VARCHAR(20), PRIMARY KEY(fb1))
ENIGNE = InnoDB
STATS_PERSISTENT = { DEFAULT | 0 | 1 } ;

-- 0은 5.5 버전 이전의 방식대로 관리
-- 1은 통계 정보를 보관
-- DEFAULT는 1로 설정되어 있음
-- ALTER 명령어로 변경 가능
```

**innodb_index_stats**

- n_diff_pfx%
    - 인덱스가 가진 유니크한 값의 개수
- n_leaf_pages
    - 인덱스의 리프 노드 페이지 개수
- size
    - 인덱스 트리의 전체 페이지 개수

**innodb_table_stats**

- n_rows
    - 테이블의 전체 레코드 건수
- clustered_index_size
    - 프라이머리 키의 크기
- sum_of_other_index_sizes
    - 프라이머리 키를 제외한 인덱스의 크기

**테이블의 통계 정보를 수집할 때 몇 개의 InnoDB 테이블 블록을 샘플링할지 결정하는 시스템 변수**

- innodb_stats_transient_sample_pages
    - 기본값 8
    - 자동으로 통계 정보 수집이 실행될 때 8개 페이지만 임의로 샘플링해서 분석하고 그 결과를 통계 정보로 활용함을 의미
- innodb_stats_persistent_sample_pages
    - 기본값 20
    - ANALYZE TABLE 명령이 실행되면 임의로 20개 페이지만 샘플링해서 분석하고 그 결과를 영구적인 통계 정보 테이블에 저장하고 활용함을 의미
- ~ 5.5 버전
    - innodb_stats_sample_pages라는 시스템 설정 변수 제공
    - 5.6 버전부터 없어짐

## 10.1.2 히스토그램

### ~ MySQL 5.7

통계 정보는 단순히 인덱스된 칼럼의 유니크한 값의 개수 정도만 가지고 있었음

최적의 실행 계획을 수집하기에는 많이 부족

실행 계획을 수집할 때 실제 인덱스의 일부 페이지를 랜덤으로 가져와 참조

### MySQL 8.0 ~

칼럼의 데이터 분포도를 참조할 수 있는 히스토그램 정보 활용 가능

### 10.1.2.1 히스토그램 정보 수집 및 삭제

히스토그램 정보는 칼럼 단위로 관리

자동으로 수집되지 않고 **ANALYZE TABLE … UPDATE HISTOGRAM** 명령을 실행해 수동으로 수집 및 관리됨

수집된 히스토그램은 시스템 딕셔너리와 함께 저장

MySQL 서버가 시작될 때 딕셔너리의 히스토그램 정보를 **information_schema** 데이터베이스의 **column_statistics** 테이블로 로드

```sql
ANALYZE TABLE employees.employees
UPDATE HISTOGRAM ON gender ;

SELECT *
FROM COLUMN_STATISTICS
WHERE SCHEMA_NAME = 'employees'
	AND TABLE_NAME = 'employees' ;
```

![analyzetable.jpeg](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/analyzetable.jpeg)

- sampling-rate
    - 히스토그램 정보를 수집하기 위해 스캔한 페이지의 비율
    - 샘플링 비율이 높아질수록 정확한 히스토그램이 되겠지만 테이블 전부를 스캔하는 것은 부하가 높으면 시스템의 자원을 많이 소모함
    - histogram_generation_max_mem_size 시스템 변수에 설정된 메모리 크기에 맞게 적절히 샘플링(기본값 20MB)
- histogram-type
    - 히스토그램의 종류
- number-of-buckets-specified
    - 히스토그램을 생성할 때 설정했던 버킷의 개수
    - 기본값은 100이며 최대값은 1024개지만 일반적으로 100개의 버킷이면 충분

히스토그램은 버킷 단위로 구분

레코드 건수나 칼럼값의 범위가 관리됨

**히스토그램 타입**

![히스토그램.jpeg](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2592%25E1%2585%25B5%25E1%2584%2589%25E1%2585%25B3%25E1%2584%2590%25E1%2585%25A9%25E1%2584%2580%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25A2%25E1%2586%25B7.jpeg)

- Singleton
    - 싱글톤 히스토그램
    - 칼럼값 개별로 레코드 건수를 관리하는 히스토그램
    - Value-Based 히스토그램 / 도수 분포
    - 칼럼이 가지는 값별로 버킷이 할당
    - 각 버킷이 칼럼의 값과 발생 빈도의 비율 값을 가짐
    - 주로 코드 값과 같이 유니크한 값의 개수가 상대적으로 적은 경우 사용됨
- Equi_Height
    - 높이 균형 히스토그램
    - 칼럼값의 범위를 균등한 개수로 구분해서 관리하는 히스토그램
    - Height-Balanced 히스토그램
    - 개수가 균등한 칼럼값의 범위별로 버킷이 할당
    - 각 버킷이 범위 시작 값과 마지막 값, 발생 빈도율, 각 버킷에 포함된 유니크한 값의 개수를 가짐
    - 칼럼값의 각 범위에 대해 레코드 건수 비율이 누적으로 표시됨

**히스토그램 삭제**

```sql
ANALYZE TABLE employees.employees
DROP HISTOGRAM ON gender ;
```

**히스토그램 사용 off**

```sql
-- 모든 쿼리 off
SET GLOBAL optimizer_switch = 'condition_fanout_filter=off' ;

-- 현재 커넥션에서 실행되는 쿼리만 off
SET SESSION optimizer_switch = 'condition_fanout_filter=off' ;

-- 현재 쿼리만 off
SELECT SET_VAR(optimizer_switch = 'condition_fanout_filter=off')
FROM ...
```

### 10.1.2.2 히스토그램의 용도

기존 MySQL 서버가 가지고 있던 통계 정보

테이블의 전체 레코드 건수와 인덱스된 칼럼이 가지는 유니크한 값의 개수

실제 응용 프로그램의 데이터는 항상 균등한 분포도를 가지지 않음

기존 통계 정보는 이런 부분을 고려하지 못함

이러한 단점을 보완하기 위해 히스토그램 도입

칼럼이 가지는 모든 값에 대한 분포도 정보를 가지지는 않지만 각 범위(버킷)별로 레코드의 건수와 유니크한 값의 개수 정보를 가지기 때문에 훨씬 정환한 예측 가능

히스토그램 정보가 없을 경우 ⇒ 데이터가 균등하게 분포돼 있을 것으로 예측

히스토그램 정보가 있을 경우 ⇒ 특정 범위의 데이터가 많고 적음 식별 가능

쿼리의 상당한 영향을 미칠 수 있음

어떤 테이블의 먼저 읽는지에 따라 쿼리 성능은 10배 정도 차이남

### 10.1.2.3 히스토그램과 인덱스

히스토그램과 인덱스는 완전히 다른 객체

부족한 통계 정보를 수집하기 위해 사용된다는 공통점

쿼리의 실행 계획을 수립할 때 인덱스들로부터 조건절에 일치하는 레코드 건수를 예측하기 위해 옵티마이저는 실제 인텍스의 B-Tree를 샘플링해서 사용 ⇒ **인덱스 다이브**

쿼리의 검색 조건으로 많이 사용되는 칼럼에 대해서는 일반적으로 인덱스를 생성

인덱스된 칼럼을 검색 조건으로 사용하는 경우 그 칼럼의 히스토그램을 사용하지 않고 실제 인덱스 다이브를 통해 직접 수집한 정보를 활용

실제 검색 조건의 대상 값에 대한 샘플링을 실행하는 것이므로 항상 히스토그램보다 정확한 결과를 기대할 수 있음

**히스토그램은 주로 인덱스되지 않은 칼럼에 대한 데이터 분포도를 참조하는 용도로 사용**

## 10.1.3 코스트 모델

**MySQL 서버가 쿼리를 처리할 때 필요한 작업**

1. 디스크로부터 데이터 페이지 읽기
2. 메모리(InnoDB 버퍼 풀)로부터 데이터 페이지 읽기
3. 인덱스 키 비교
4. 레코드 평가
5. 메모리 임시 테이블 작업
6. 디스크 임시 테이블 작업

사용자의 쿼리에 대해 다양한 작업이 얼마나 필요한지 예측하고 전체 작업 비용을 계산한 결과를 바탕으로 최적의 실행 계획을 찾음

전체 쿼리의 비용을 계산하는 데 필요한 단위 작업들의 비용 ⇒ **코스트 모델**

### **~ MySQL 5.7**

MySQL 서버 소스 코드에 상수화해서 사용

MySQL 서버가 사용하는 하드웨어에 따라 달라질 수 있음

고정된 비용을 일률적으로 적용하는 것은 최적의 실행 계획 수립에 있어 방해 요소가 됨

### **MySQL 5.7 ~**

DBMS 관리자가 조정할 수 있도록 개선

인덱스되지 않은 칼럼의 데이터 분포(히스토그램)나 메모리에 상주 중인 페이지의 비율 등 비용 계산과 연관된 부분의 정보가 부족한 상태

### **MySQL 8.0 ~**

칼럼의 데이터 분포를 위한 히스토그램과 각 인덱스별 메모리에 적재된 페이지의 비율이 관리되고 옵티마이저의 실행 계획 수립에 사용되기 시작

코스트 모델은 MySQL DB에 존재하는 2개의 테이블에 저장되어 있는 설정값을 사용

- **server_cost**
    - 인덱스를 찾고 레코드를 비교하고 임시 테이블 처리에 대한 비용 관리
    - cost_name
        - 코스트 모델의 각 단위 작업
    - default_value
        - 각 단위 작업의 비용
    - cost_value
        - DBMS 관리자가 설정한 값
    - last_updated
        - 단위 작업의 비용이 변경된 시점
    - comment
        - 비용에 대한 추가 설명
- **engine_cost**
    - 레코드를 가진 데이터 페이지를 가져오는 데 필요한 비용 관리
    - engine_name
        - 비용이 적용된 스토리지 엔진
    - device_type
        - 디스크 타입
        - MySQL 8.0에서는 아직 칼럼의 값을 활용하지 않음

**코스트 모델에서 지원하는 단위 작업**

![costmodel.jpeg](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/costmodel.jpeg)

- row_evaluate_cost
    - 스토리지 엔진이 반환한 레코드가 쿼리의 조건에 일치하는지를 평가하는 단위 작업
    - 증가할수록 풀 테이블 스캔과 같이 많은 레코드를 처리하는 쿼리의 비용이 높아짐
    - 비용을 높이면 풀 스캔을 실행하는 쿼리들의 비용이 높아지고 MySQL 서버 옵티마이저가 가능하면 인덱스 레인지 스캔을 사용하는 실행 계획을 선택할 가능성이 높아짐
- key_compare_cost
    - 키 값의 비교 작업에 필요한 비용
    - 증가할수록 레코드 정렬과 같이 키 값 비교가 처리가 많은 경우 쿼리의 비용이 높아짐
    - 비용을 높이면 MySQL 서버 옵티마이저가 가능하면 정렬을 수행하지 않는 방향의 실행 계획을 선택할 가능성이 높아짐
- disk_temptable_create_cost / disk_temptable_row_cost
    - 비용을 높이면 MySQL 서버 옵티마이저는 디스크에 임시 테이블을 만들지 않는 방향의 실행 계획을 선택할 가능성이 높아짐
- memory_temptable_create_cost / memory_temptable_row_cost
    - 비용을 높이면 MySQL 서버 옵티마이저는 임시 테이블을 만들지 않는 방향의 실행 계획을 선택할 가능성이 높아짐
- io_block_read_cost
    - 비용이 높아지면 MySQL 서버 옵티마이저는 가능하면 InnoDB 버퍼 풀에 데이터 페이지가 많이 적재돼 있는 인덱스를 사용하는 실행 계획을 선택할 가능성이 높아짐
- memory_block_read_cost
    - 비용이 높아지면 MySQL 서버 옵티마이저는 가능하면 InnoDB 버퍼 풀에 적재된 데이터 페이지가 상대적으로 적다고 하더라도 그 인덱스를 사용할 가능성이 높아짐

**실행 계획에 계산된 비용 확인하는 법**

```sql
EXPLAIN FORMAT = JSON | TREE
SELECT *
FROM test_table WHERE a = 'A' ;
```

# 10.2 실행 계획 확인

MySQL 서버의 실행 계획은 **DESC** 또는 **EXPLAIN** 명령으로 확인 가능

## 10.2.1 실행 계획 및 출력 포맷

MySQL 8.0 버전부터는 FORMAT 옵션을 사용해 실행 계획의 표시 방법을  JSON이나 TREE, 단순 테이블 형태로 선택 가능

개인의 선호도와 표시되는 정보의 차이가 있지만 MySQL 옵티마이저가 수립한 실행 계획의 큰 흐름을 보여주는 데는 큰 차이가 없음

```sql
EXPLAIN
SELECT *
FROM test_table t1
	INNER JOIN temp_table t2 ON t1.a = t2.a
WHERE t1.a = 'A';

EXPLAIN FORMAT = JSON | TREE
SELECT *
FROM test_table t1
	INNER JOIN temp_table t2 ON t1.a = t2.a
WHERE t1.a = 'A';
```

**단순 테이블**

![table.jpeg](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/table.jpeg)

**TREE**

![tree.jpeg](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/tree.jpeg)

**JSON**

![json.jpeg](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/json.jpeg)

## 10.2.2 쿼리의 실행 시간 확인

MySQL 8.0.18 버전부터는 쿼리의 실행 계획과 단계별 소요된 시간 정보를 확인할 수 있는 **EXPLAIN ANALYZE** 기능이 추가됨

```sql
EXPLAIN ANALYZE
SELECT t1.b, avg(t2.c)
FROM test_table t1
	INNER JOIN temp_table t2 ON t1.a = t2.a
WHERE t1.b = 'B'
GROUP BY t1.b;

-- 항상 TREE 포맷으로 보여짐
```

**실행 순서 읽는 방법**

1. 들여쓰기가 같은 레벨에서는 상단에 위치한 라인이 먼저 실행
2. 들여쓰기가 다른 레벨에서는 가장 안쪽에 위치한 라인이 먼저 실행

![timewithquery.jpeg](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/timewithquery.jpeg)

- 순서
    - D → F → E → C → B → A
- actual_time0.007..0.009
    - employees 테이블에서 읽은 emp_no 값을 기준으로 salaries 테이블에서 일치하는 레코드를 검색하는 데 걸린 시간(밀리초)
    - 첫 번째 숫자 값은 첫 번째 레코드를 가져오는 데 걸린 평균 시간(밀리초)
    - 두 번째 숫자 값은 마지막 레코드를 가져오는 데 걸린 평균 시간(밀리초)
- rows=10
    - employees 테이블에서 읽은 emp_no에 일치하는 salaries 테이블의 평균 레코드 건수
- loops=233
    - employees 테이블에서 읽은 emp_no를 이용해 salaries 테이블의 레코드를 찾는 작업이 반복된 횟수
    - employees 테이블에서 읽은 emp_no의 개수를 의미

실행 계획만 추출하는 것이 아니라 실제 쿼리를 실행하고 사용된 실행 계획과 소요된 시간을 보여주는 것이기 때문에  **EXPLAIN ANALYZE** 명령을 사용하면 쿼리가 완료돼야 결과를 확인할 수 있음

실행 계획이 아주 나쁜 경우라면 **EXPLAIN** 명령으로 먼저 실행 계획만 확인해서 어느 정도 튜닝한 후 해당 명령을 실행하는 것이 좋음

# 10.3 실행 계획 분석

실행 계획에 표시되는 각 칼럼이 어떤 것을 의미하는지

각 칼럼에 어떤 값들이 출력될 수 있는지

## 10.3.1 id 칼럼

하나의 **SELECT** 문장은 다시 1개 이상의 하위 **SELECT** 문장을 포함할 수 있음

```sql
SELECT ...
FROM (SELECT ... FROM tb_test1) tb1, tb_test2 tb2
WHERE tb1.id = tb2.id ;

-- 단위 쿼리
SELECT ... FROM tb_test1 ;
SELECT ... FROM tb1, tb_test2 tb2 WHERE tb1.id = tb2.id ;
```

**단위 쿼리**

**SELECT** 키워드 단위로 구분한 것

**실행 계획에서 가장 왼쪽에 표시된 id 칼럼**

![table.jpeg](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/table.jpeg)

SELECT 쿼리별로 부여되는 식별자 값

**SELECT** 문장은 하나인데, 여러 개의 테이블이 조인되는 경우

```sql
EXPLAIN
SELECT e.emp_no, e.first_name, s.from_date, s.salary
FROM employees e, salaries s
WHERE e.emp_no = s.emp_no LIMIT 10 ;
```

![KakaoTalk_Photo_2023-09-18-20-01-23 001.jpeg](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/KakaoTalk_Photo_2023-09-18-20-01-23_001.jpeg)

id 값이 증가하지 않고 같은 id 값이 부여

쿼리 문장이 3개의 단위 **SELECT** 쿼리로 구성된 경우

```sql
EXPLAIN
SELECT
( (SELECT COUNT(*) FROM employees)
+ (SELECT COUNT(*) FROM deparatments) )
AS total_count ;
```

![KakaoTalk_Photo_2023-09-18-20-01-23 002.jpeg](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/KakaoTalk_Photo_2023-09-18-20-01-23_002.jpeg)

각 레코드가 각기 다른 id 값을 가짐

**주의해야 할 점**

실행 계획의 id 칼럼이 테이블의 접근 순서를 의미하지 않음

## 10.3.2 select_type 칼럼

**select_type 칼럼**

각 단위 **SELECT** 쿼리가 어떤 타입의 쿼리인지 표시되는 칼럼

### SIMPLE

**UNION**이나 서브쿼리를 사용하지 않는 단순한 **SELECT** 쿼리

쿼리에 조인이 포함된 경우도 마찬가지

쿼리 문장이 아무리 복잡하더라도 단위 쿼리는 하나만 존재

일반적으로 제일 바깥 **SELECT** 쿼리의 타입임

### PRIMARY

**UNION**이나 서브쿼리를 가지는 **SELECT** 쿼리의 실행 계획에서 가장 바깥쪽에 있는 단위 쿼리

**SIMPLE**과 마찬가지로 하나만 존재

쿼리의 제일 바깥쪽에 있는 **SELECT** 단위 쿼리

### UNION

**UNION**으로 결합하는 단위 **SELECT** 쿼리 가운데 첫 번째를 제외한 두 번째 이후 단위 **SELECT** 쿼리

첫 번째 단위 **SELECT**는 임시 테이블(**DERIVED**) select_type으로 표시됨

```sql
EXPLAIN
SELECT * FROM (
	(SELECT emp_no FROM employees e1 LIMIT 10) UNION ALL -- **DERIVED**
	(SELECT emp_no FROM employees e2 LIMIT 10) UNION ALL -- **UNION**
	(SELECT emp_no FROM employees e1 LIMIT 10) -- **UNION**
) tb ;
```

### DEPENDENT UNION

**UNION**이나 **UNION ALL**로 집합을 결합하는 쿼리

**DEPENDENT**

**UNION**이나 **UNION ALL**로 결합된 단위 쿼리가 외부 쿼리에 의해 영향을 받는다는 것을 의미

```sql
EXPLAIN
SELECT *
FROM employees e1 WHERE e1.emp_no IN (
	SELECT e2.emp_no FROM employees e2 WHERE e2.first_name = 'Matt' 
	UNION
	SELECT e3.emp_no FROM employees e3 WHERE e3.last_name = 'Matt' -- DEPENDENT UNION
) ;

-- 외부에서 정의된 emp_no 칼럼이 서브쿼리에 사용됨 => 외부 영향
```

### UNION RESULT

**UNION** 결과를 담아두는 테이블

실제 쿼리에서 단위 쿼리가 아니기 때문에 별도의 id 값은 부여되지 않음

```sql
EXPLAIN
SELECT emp_no FROM salaries WHERE salary > 100000
UNION DISTINCT
SELECT emp_no FROM dept_emp WHERE from_date > '2001-01-01' ;
```

![KakaoTalk_Photo_2023-09-18-20-25-16.jpeg](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/KakaoTalk_Photo_2023-09-18-20-25-16.jpeg)

table 칼럼 “**<union1,2>**”

id 값이 1인 단위 쿼리의 조회 결과와 id 값이 2인 단위 쿼리의 조회 결과를 **UNION** 했다는 의미

**UNION DISTINCT** → **UNION ALL**

**UNION RESULT** 라인이 사라짐

**UNION ALL**을 사용하면 MySQL 서버는 임시 테이블에 버퍼링하지 않기 때문

### SUBQUERY

**FROM** 절 이외에서 사용되는 서브쿼리

```sql
EXPLAIN
SELECT e.first_name,
			(SELECT COUNT(*)
				FROM dept_emp de, dept_manager dm
				WHERE dm.dept_no = de.dept_no) AS cnt
FROM employees e WHERE e.emp_no = 10001 ;
```

MySQL 서버의 실행 계획에서 **FROM** 절에 사용된 서브쿼리는 **DERIVED**로 표시됨

### DEPENDENT SUBQUERY

서브쿼리가 바깥쪽 **SELECT** 쿼리에서 정의된 칼럼을 사용하는 경우

```sql
EXPLAIN
SELECT e.first_name,
				(SELECT COUNT(*) -- DEPENDENT SUBQUERY
					FROM dept_emp de, dept_manager dm
					WHERE dm.dept_no = de.dept_no AND de.emp_no = e.emp_no) AS cnt
FROM employees e
WHERE e.first_name = 'Matt' ;
```

### DERIVED

단위 **SELECT** 쿼리의 실행 결과로 메모리나 디스크에 임시 테이블(파생 테이블)을 생성하는 것

옵티마이져 옵션에 따라 쿼리의 특성에 맞게 임시 테이블에도 인덱스 추가 가능

 

```sql
EXPLAIN
SELECT *
FROM (SELECT de.emp_no FROM dept_emp de GROUP BY de.emp_no) tb, -- DERIVED
	employees e
WHERE e.emp_no = tb.emp_no ;
```

MySQL 서버 버전이 업그레이드되면서 조인 쿼리에 대한 최적화가 많이 성숙된 상태

파생 테이블에 대한 최적화가 부족한 버전(~5.6)에서는 조인 쿼리로 해결할 수 있도록 개선하는 것이 좋음

### DEPENDENT DERIVED

**FROM** 절의 서브쿼리가 외부 칼럼을 참조하는 경우

MySQL 8.0 버전부터 **LATERAL JOIN**을 사용해서 **FROM** 절의 서브쿼리에서도 외부 칼럼을 참조할 수 있게 됨

```sql
EXPLAIN
SELECT *
FROM employees e
LEFT JOIN LATERAL
	(SELECT * -- DEPENDENT DERIVED
		FROM salaries s
		WHERE s.emp_no = e.emp_no
		ORDER BY s.from_date DESC LIMIT 2) AS s2 ON s2.emp_no = e.emp_no ;
```

### UNCACHEABLE SUBQUERY

하나의 쿼리 문장에 서브쿼리가 하나만 있더라고 실제 그 서브쿼리가 한 번만 실행되는 건 아님

조건이 똑같은 서브쿼리가 실행될 때는 다시 실행하지 않고 이전의 실행 결과를 그대로 사용할 수 있게 서브쿼리의 결과를 내부적인 캐시 공간에 담아둠

캐시 사용 가능 여부에 따라 **SUBQUERY**인지 **UNCACHEABLE SUBQUERY**인지 구분됨

서브쿼리에 포함된 요소에 의해 캐시 자체가 불가능한 경우 **UNCACHEABLE SUBQUERY**

1. 사용자 변수가 서브쿼리에 사용된 경우
2. **NOT-DETERMINISTIC** 속성의 스토어드 루틴이 서브쿼리 내에 사용된 경우
3. **UUID()**나 **RAND()**와 같이 결괏값이 호출할 때마다 달라지는 함수가 서브쿼리에 사용된 경우

### UNCACHEABLE UNION

**UNION**과 **UNCACHEABLE**이 협쳐진 것

### MATERIALIZED

**FROM** 절이나 **IN** 형태의 쿼리에 사용된 서브쿼리의 최적화를 위해 사용됨

```sql
EXPLAIN
SELECT *
FROM employees e
WHERE e.emp_no IN 
(SELECT emp_no FROM salaries WHERE salary BETEEN 10 AND 1000) ; -- MATERIALIZED
```

서브쿼리의 내용을 임시 테이블로 구체화한 후, 임시 테이블과 employees 테이블을 조인하는 형태로 최적화되어 처리

## 10.3.3 table 칼럼

Mysql 서버의 실행 계획은 단위 SELECT 쿼리 기준이 아닌 테이블 기준으로 표시됨

```sql

mysql > EXPLAIN SELECT now()
mysql > EXPLAIN SELECT now() FROM DUAL;
-- (DUAL 이라는 테이블은 없는 테이블임, 테이블은 없지만 오류를 발생시키지 않음)
```

예제와 같이 별도의 테이블을 사용하지 않는 SELECT 쿼리인 경우 table 칼럼에 NULL 이 표시됨

![스크린샷 2023-09-25 오전 7.41.38.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-25_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_7.41.38.png)

table 칼럼에 `<derived N>` 또는 `union <M,N>` 과 같이 `< >` 로 둘러싸인 이름이 명시되는 경우가 많음

이 테이블은 `임시테이블`을 의미함, `< >` 안에 표시되는 숫자의 단위는 `SELEC 쿼리의 id 값`을 지칭함

![스크린샷 2023-09-25 오전 7.53.36.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-25_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_7.53.36.png)

`<derived2>` - `SELECT 쿼리 id 값이 2`인 실행계획으로 부터 만들어진 파생테이블을 가리킴

![스크린샷 2023-09-25 오전 7.54.10.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-25_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_7.54.10.png)

위를 분석해보면

1. <derived2> 라는 것을 보아 id 값이 2인 라인이 먼저 실행되고 경과가 파생테이블로 준비되어야함
2. id 2 에서 select_type 칼럼이 DERIVED 로 표시되어 있음, 즉, 이 라인은 table 칼럼에 표시된 dept_emp 테이블을 읽어서 파생 테이블을 생성하는 것임
3. 첫번째와 두번째가 동일한 id 를 가지고 있음, 2개의 테이블 (<derived2> 와 e )이 조인 된다는 사실을 알 수 있음.
e테이블보다 <derived 2> 가 먼저 표시됐기 때문에 <derived2>가 드라이빙 테이블, e가 드리븐 테이블이 됨

위의 실제 쿼리는 다음과 같음

```sql
mysql > SELECT *
			FROM
				(SELECT de.emp_no FROM dept_emp de GROUP BY de.emp_no) tb,
				employees e
			WHERE e.emp_no=tb.emp_no;
```

(mysql 8.0 버전에서는 서브쿼리에 대한 최적화가 많이 보완됨)

`select_type` 에서 `MATERIALIZED` 인 실행계획에서 `<subquery N>` 과 같은 값이 table에 표시됨
이는 `서브쿼리의 결과`를 구체화 해서 `임시테이블`로 만들어졌다는 의미임
실제로는 <derived N> 과 같은 방법으로 해석하면 됨

## 10.3.4 partitions 칼럼

employees_2 테이블은 hire_date 칼럼값 기준으로 5년단위로 나누어진 파티션을 가짐

파티션 생성시 제약사항(파티션 키로 사용되는 칼럼은 프라이머리 키를 포함한 모든 유니크 인덱스의 일부여야함) 으로 인해 프라이머리 키에 emp_no 칼럼과 함께 hire_date 칼럼을 추가해 테이블을 생성함

```sql
mysql > CREATE TABLE employees_2 (
			emp_no int NOT NULL,
			...
			...
			hire_date DATE NOT NULL,
			PRIMARY KEY (emp_no, hire_date)
) PARTITION BY RANGE COLUMNS(hire_date)
(PARTITION p1986_1990 VALUES LESS THEN ('1990-01-01'),
PARTITION p1991_1995 VALUES LESS THEN ('1996-01-01'),
PARTITION p1996_2000 VALUES LESS THEN ('2000-01-01'),
PARTITION p2001_2005 VALUES LESS THEN ('2006-01-01'));	

mysql > INSERT INTO employees_2 SELECT * FROM employees;
```

(1999-11-15 ~ 2001-01-15 사이 레코드 쿼리 하는 예제)

```sql
mysql > EXPLAIN
			SELECT *
			FROM employees_2
			WHERE hire_date BETWEEN '1999-11-15' AND '2000-01-15';	
```

파티션이 여러 개인 테이블에서 불필요한 파티션을 빼고 쿼리를 수행하기 위해 접근해야 할 것으로 판단되는 테이블만 골라내는 과정을 `파티션 프루닝` 이라고 함.

위의 쿼리에서 `p1996_2000` 과 `p2001_2005` 파티션만 접근했다는 것을 다음 실행 계획에서 알 수 있음

![스크린샷 2023-09-26 오전 12.34.25.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_12.34.25.png)

type 칼럼은 ALL 임 
⇒ 풀 테이블 스캔으로 쿼리가 처리된다는 뜻임!?!!?

⇒ 파티션은 물리적으로 개별 테이블 처럼 별도 저장 공간을 가지기 때문임

`employees_2` 테이블의 모든 파이션이 아니라 `p1996_2000, p2001_2005` 파티션만 풀 스캔을 실행하게 됨

## 10.3.5 type 칼럼

type 이후의 칼럼은 MySQL 서버가 각 테이블의 레코드를 어떤 방식으로 읽었는지를 나타냄

(인덱스를 사용해 레코드를 읽었는지, 풀스캔으로 레코드를 읽었는지 등)

12 가지 방법

- system, const, eq_ref, ref, fulltext, ref_or_null, unique_subquery, index_subquery, range, index_merge, index, ALL
    - 12 개이며, `ALL`을 제외한 `나머지는 모두 인덱스를 사용하는 접근` 방법임
    - index_merge 를 제외한 나머지 접근방법은 하나의 인덱스만 사용함

### 10.3.5.1 system

레코드가 1건만 존재하는 테이블 또는 한건도 존재하지 않는 테이블을 참조하는 접근형태

- MyISAM 나 MEMORY 테이블에서만 접근되는 방법
- innoDB 는 ALL 또는 index 로 표시될 가능성이 큼

실제 애플리케이션에서 사용되는 쿼리에서는 거의 보이지 않는 실행계획임

### 10.3.5.2 const

레코드 건수와 관계없이 쿼리가 프라이머리 키나 유니크 키 칼럼을 이용하는 WHERE 절 조건절

반드시 1건을 반환하는 쿼리 처리방식을 const 라고 함

(다른 DBMS 에서는 유니크 인덱스 스캔 이라고도 표현함)

```sql
mysql > EXPLAIN 
				SELECT * FROM employees WHERE emp_no=10001;
```

![스크린샷 2023-09-26 오전 1.35.27.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_1.35.27.png)

다중 칼럼으로 구성된 프라이머리키는 일부 칼럼만 조건으로 할때는 const 가 아님

```sql
mysql > EXPLAIN
			SELECT * FROM dept_emp WHERE dept_no='d005' AND emp_no=10001;
-- (dept_no, emp_no)의 복합키로 한번에 해야 const 로 나옴
```

### 10.3.5.3 eq_ref

여러 테이블이 조인되는 쿼리의 실행계획에서만 표시됨

조인에서 `처음 읽은 테이블의 칼럼값`을, `그 다음 읽어야할 테이블`의 `프라이머리 키`나 `유니크 키` 칼럼의 검색조건에서 사용될때 `eq_ref` 라고 함

유니크 키로 검색할때, 그 유니크 인덱스는 NOT NULL 이어야함.

즉, 조인에서 두번째 이후에 읽은 테이블에서 반드시 1건 만 존재한다는 보장이 있어야 함.

```sql
mysql > EXPLAIN
			select * from dept_emp de, employees e
			where e.emp_no=de.emp_no and de.dept_no='d0005';
```

![스크린샷 2023-09-26 오전 1.44.40.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_1.44.40.png)

### 10.3.5.4 ref

조인의 순서와 관계없이 사용됨
인덱스의 종류와 관계없이 동등(Equal) 조건으로 검색할 때는 ref 접근 방법이 사용됨

ref 타입은 반환되는 레코드가 반드시 1건 이라는 보장이 없으므로 const나 eq_ref 보다는 빠르지 않음

하지만 동등 조건으로만 비교되므로 매우 빠른 레코드 조회 방법의 하나임

```sql
mysql > EXPLAIN
			SELECT * FROM dep_emp WHERE dept_no='d005';
```

type 은 ref 임 ⇒ 동등조건 

ref 는 const 임 ⇒ ‘d005’

![스크린샷 2023-09-26 오전 1.55.43.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_1.55.43.png)

- const, eq_ref, ref
    - 모두 `=` 연산자 임
    - 세가지 모두 매우 좋은 접근 방법임
    - 인덱스의 분포도가 나쁘지 않다면 성능상 문제를 일으키지 않는 접근 방법임
    - 쿼리 튜닝할 때, 이 세가지 접근 방법에 대서는 크게 신경쓰지 않고 넘어가도 무방함

### 10.3.5.5 fulltext

MySQL 서버에서 전문 검색 조건은 우선순위가 상당히 높음.

쿼리에서 전문 인덱스를 사용하는 조건과 그 이외의 일반 인덱스를 사용하는 조건을 함께 사용하면
일반 인덱스의 접근방법이 `const나 eq_ref, ref 아니면`, MySQL 의 `전문 인덱스`를 사용하는 조건을 선택해서 처리함

`MATCH(..) AGAINST(…)` 구문을 사용하여 쿼리함

전문 인덱스를 사용하기위해 테이블에 정의되어있어야 함.

(전문 인덱스 사용하기 위한 전문검색 인덱스)

```sql
mysql > CREATE TABLE employees_name (
			emp_no int NOT NULL,
			first_name varchar(14) NOT NULL,
			last_name varchar(14) NOT NULL,
			PRIMARY KEY (emp_no),
			FULLTEXT KEY fx_name (first_name, last_name) WITH PARSER ngram
			) ENGINE=InnoDB;
```

예제)

```sql
mysql > EXPLAIN
	SELECT *
	FROM employee_name
	WHERE emp_no=10001
		AND emp_no BETWEEN 10001 AND 10005
		AND MATCH(first_name, last_name) AGAINST('Facello' IN BOOLEAN MODE);
```

const 로 나옴 emp_no 로 했기 때문

![스크린샷 2023-09-26 오전 2.23.37.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_2.23.37.png)

예제 2) emp_no 조건 삭제

```sql
mysql > EXPLAIN
	SELECT *
	FROM employee_name
	WHERE emp_no BETWEEN 10001 AND 10005
		AND MATCH(first_name, last_name) AGAINST('Facello' IN BOOLEAN MODE);
```

fulltext 로 나옴

![스크린샷 2023-09-26 오전 2.24.10.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_2.24.10.png)

경험적으로 fulltext 보다 일반 인덱스를 사용하는 range 접근 방법이 빨리 처리되는 경우가 더 많았음.
전문 검색 쿼리를 사용할 때는 조건별로 성능을 확인해 보는 편이 좋음.

### 10.3.5.6 ref_or_null

ref 접근 방법과 같은데 NULL 비교가 추가된 형태임

```sql
mysql > EXPLAIN
			SELECT * FROM titles
			WHERE to_date='1985-03-01' OR to_date IS NULL;
```

![스크린샷 2023-09-26 오전 2.43.29.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_2.43.29.png)

### 10.3.5.7 unique_subquery

WHERE 조건절에서 사용될 수 있는 IN(subquery)의 형태의 쿼리를 위한 접근 방법임
`unique_subquery` 의 의미 그대로 `서브쿼리에서 중복되지 않는 유니크한 값`만 반환할 때 이 접근 방법을 사용함

```sql
mysql > EXPLAIN
	SELECT * FROM departments
	WHERE dept_no IN (SELECT dept_no FROM dep_emp WHERE emp_no=10001);		
```

![스크린샷 2023-09-26 오전 2.43.48.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_2.43.48.png)

### 10.3.5.8 index_subquery

IN (subquery) 형태의 조건에서 subquery의 반환값에 중복된 값이 있을 수 는 있지만
인덱스를 이용해 중복된 값을 제거할 수 있는 경우

### 10.3.5.9 range

range 는 인덱스 레인지 스캔 형태의 접근 방법임

인덱스를 하나의 값이 아니라 범위로 검색하는 경우를 의미함, 

- `<` , `>` , `IS NULL` , `BETWEEN` , `IN` , `LIKE`  등의 연산자를 이용함

range 접근 방법도 상당히 빠름

```sql
mysql > EXPLAIN
		select * from employees where emp_no between 10002 and 10004;
```

![스크린샷 2023-09-26 오전 5.31.48.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_5.31.48.png)

### 10.3.5.10 index_merge

index_merge 접근 방법은 `2개 이상`의 `인덱스`를 이용해 각각의 `검색 결과를 만들어낸 후`, 
그 결과를 `병합`해서 `처리하는 방식`임

- 여러 인덱스를 읽어야 하므로 일반적으로 range 접근 방법보다 효율성이 떨어짐
- 전문 검색 인덱스를 사용하는 쿼리에서는 index_merge 가 적용되지 않음
- index_merge 접근 방법으로 처리된 결과는 항상 2개 이상의 집합이 되므로, 그 두 집합의 교집합이나 합집합 또는 중복 제거와 같은 `부가적인 작업이 더 필요함`

```sql
mysql > EXPLAIN
	select * from employees
	where emp_no between 10001 and 11000
		or first_name='Smith';
```

![스크린샷 2023-09-26 오전 5.37.47.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_5.37.47.png)

`emp_no between 10001 and 1100` 조건은 employees 테이블의 프라이머리 키를 이용해 조회하고, `first_name='Smith'` 조건은 ix_firstname 인덱스를 이용해 `조회한 후 두 결과를 병합`하여 처리하는 실행계획을 만들어냄

### 10.3.5.11 index

사람들이 자주 오해하는 접근 방법임

접근 방법이 index 라서 `"효율적으로 인덱스를 사용하는 구나"` 라고 생각하게 만듬

하지만, index 접근 방법은 `인덱스 풀 스캔`을 의미함

range 접근 방법과 같이 효율적으로 인덱스의 필요한 부분만 읽는 것을 의미하는 것이 아님

- range 나 const, ref 같은 접근 방법으로 인덱스를 사용하지 못하는 경우
- 인덱스에 포함된 칼럼만으로 처리할 수 있는 쿼리인 경우(즉, 데이터 파일을 읽지 않아도 되는 경우)
- 인덱스를 이용해 정렬이나 그루핑 작업이 가능한 경우(즉, 별도의 정렬 작업을 피할 수 있는 경우)

```sql
mysql > EXPLAIN
	select * from departments order by dept_name desc limit 10;
```

![스크린샷 2023-09-26 오전 6.00.25.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_6.00.25.png)

정렬할려는 칼럼이(ux_deptname) 인덱스가 있으므로 index 접근 방법을 사용함.

### 10.3.5.12 ALL

풀 테이블 스캔을 의미하는 접근 방법임

가장 비효율적인 방법임

## 10.3.6 possible_keys칼럼

옵티마이저가 최적의 실행 계획을 만들기 위해 후보로 선정 했던 접근 방법에서 사용되는 인덱스의 목록일 뿐임

즉, `“사용될법했던 인덱스의 목록”` 임

possible_keys 칼럼은 특별한 경우를 제외하고는 무시해도 됨

## 10.3.7 key 칼럼

(최종으로 선택된) 실행계획에서 사용하는 인덱스를 의미함.

`쿼리를 튜닝`할 때, `key 칼럼에 의도했던 인덱스가 표시`되는지 확인하는 것이 중요함

`index_merge` 실행계획일 경우 여러개의 인덱스가 `,`로 구분되어 표시됨

실행계획의 `type`이 `ALL` 일때와 같이 인덱스를 전혀 사용하지 못한다면 `key` 칼럼은 `NULL` 로 표시됨

## 10.3.8 key_len 칼럼

key_len 칼럼은 사용자가 쉽게 무시하는 정보이지만, `매우 중요한 정보`중 하나임

key_len 칼럼의 값은 쿼리를 처리하기 위해 `다중 칼럼`으로 구성된 `인덱스`에서 `몇개의 칼럼까지 사용했는지` 우리에게 알려줌

(더 정확하게는..) 인덱스의 각 레코드에서 몇 바이트까지 사용했는지 알려주는 값임

```sql
mysql > EXPLAIN
		select * from dept_emp where dept_no='d005';
```

![스크린샷 2023-09-26 오전 6.39.13.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_6.39.13.png)

(utf8mb4 는 1~4 바이트까지 가변적임, 하지만 메모리를 공간을 할당할 때는 고정적으로 4바이트로 계산함)

dept_no 칼럼이 CHAR(4) 이기 때문에 16바이트만 유효하게 사용했다는뜻임

예제)

```sql
mysql > EXPLAIN
	SELECT * FROM dept_emp WHERE dept_no='d005' AND emp_no=10001;
```

![스크린샷 2023-09-26 오전 6.45.56.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_6.45.56.png)

`emp_no` 는 `INTERGER 4바이트` 임, (20 = 16 + 4)

위의 쿼리에서 `dept_no` 칼럼뿐만 아니라 `emp_no` 까지 사용할 수 있게 적절히 조건이 제공됨

예제)

```sql
mysql > CREATE TABLE titles (
				emp_no int NOT NULL,
				...
				to_date date DEFAULT NULL,
				PRIMARY KEY (emp_no, from_date, title),
				KEY ix_todate (to_date)
			);

mysql > EXPLAIN
			select * from titles where to_date<='1985-10-10';
```

![스크린샷 2023-09-26 오전 6.52.45.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_6.52.45.png)

DATE 타입의 칼럼은 3바이트 인데 4바이트로 나온 이유는!?

⇒ NULLABLE 칼럼이므로 1바이트를 추가로 더 사용함 

## 10.3.9 ref 칼럼

접근 방법이 ref 면 참조 조건(Equal 비교조건)으로 어떤 값이 제공됐는지 보여줌

- 상숫값이라면 const 로 보여줌
- 다른 테이블의 칼럼값이면 그 테이블명 과 칼럼명이 표시됨
- ref 칼럼값이 func 이라 표시될때가 있음
    - 참조용으로 사용되는 값을 그대로 사용한 것이 아니라
    콜레이션 변환이나 값 자체의 연산을 거쳐서 참조했다는 의미임

예제)

```sql
mysql > EXPLAIN
	select *
	from employees e, dept_emp de
	where e.emp_no=de.emp_no;
```

![스크린샷 2023-09-26 오전 8.27.25.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_8.27.25.png)

예제2)

```sql
mysql > EXPLAIN
	select *
	from employees e, dept_emp de 
	where e.emp_no=(de.emp_no-1);
```

![스크린샷 2023-09-26 오전 8.29.57.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_8.29.57.png)

사용자가 명시적으로 값을 변환할때 뿐만 아니라, MySQL 서버가 내부적으로 값을 변환할때도 ref 칼럼에는 func가 출력됨

숫자 타입의 칼럼과 문자열 타입의 칼럼으로 조인할때 등

## 10.3.10 rows 칼럼

rows 칼럼값은 실행 계획의 효율성을 판단을 위해 예측했던 레코드 건수를 보여줌

스토리지 엔진별로 가지고있는 통계 정보를 참조해 MySQL 옵티마이저가 산출해 낸 예상값이라서 정확하지 않음.

rows 칼럼에 표시되는 값은 `반환하는 레코드의 예측치가 아니라`, 쿼리를 `처리하기 위해 얼마나 많은 레코드를 읽고 처리해야하는지`를 의미함

예제)

```sql
mysql > EXPLAIN
	select * from dept_emp where from_date >= '1985-01-01';
```

![스크린샷 2023-09-26 오전 8.43.15.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_8.43.15.png)

331143 건을 읽어야할것이라고 예측

풀 테이블 스캔을 진행함

예제2)

```sql
mysql > EXPLAIN 
	select * from dept_emp where from_date >= '2002-07-01';
```

![스크린샷 2023-09-26 오전 8.43.42.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_8.43.42.png)

292 건 예측 (8.8 %)

range 로 인덱스 레인지 스캔을 사용

key_len 에는 3바이트로 표시됨

## 10.3.11 filtered 칼럼

filtered 칼럼의 값은 필터링 되고 남은 레코드의 비율을 의미함

예제)

```sql
mysql > EXPLAIN 
	select *
	from employees e,
			salaries s
	where e.first_name='Matt'
			and e.fire_date between '1990-01-01' and '1991-01-01'
			and s.emp_no=e.emp_no
			and s.from_date between '1990-01-01' and '1991-01-01'
			and s.salary between 50000 and 60000;
```

![스크린샷 2023-09-26 오전 9.00.36.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_9.00.36.png)

대략 233 건이며, 이중 16.03% 만 인덱스를 사용하지 못하는

`e.fire_date between '1990-01-01' and '1991-01-01'` 조건에 일치한다는 것임

따라서, 37건(233 * 0.1603) 이 employees 테이블에서 salaries 테이블로 조인되었음

예제2)

```sql
mysql > EXPLAIN 
	select  /*+ JOIN_ORDER(s,e) */
  *
	from employees e,
			salaries s
	where e.first_name='Matt'
			and e.fire_date between '1990-01-01' and '1991-01-01'
			and s.emp_no=e.emp_no
			and s.from_date between '1990-01-01' and '1991-01-01'
			and s.salary between 50000 and 60000;
```

join_order : `가능하다면, 나열된 join 순서로 조인할것을 권고한다`

- [http://minsql.com/mysql8/B-2-D-optimizerHint/](http://minsql.com/mysql8/B-2-D-optimizerHint/)

![스크린샷 2023-09-26 오전 9.00.53.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-26_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_9.00.53.png)

368건(3314*0.1111)이 salaries 테이블에서 employees 테이블로 조인이 수행되었음.

## 10.3.12 Extra 칼럼

Extra 칼럼에는 주로 내부적인 처리 알고리즘에 대해 조금 더 깊이 있는 내용을 보여주는 경우가 많음

### 10.3.12.1 const row not found

const 접근 방법으로 테이블을 읽었지만 실제로 해당 테이블에 레코드가 1건도 존재하지 않는경우

### 10.3.12.2 Deleting all rows

where 조건절이 없는 delete 문장의 실행 계획에 자주 표시됨.

이 문구는 테이블의 `모든 레코드를 삭제하는 핸들러 기능(API)을 호출`함으로써 처리됐다는 것을 의미함.

### 10.3.1.12.3 Distinct

Extra 칼럼에 Distinct 키워드가 표시되는 예제

```sql
mysql > EXPLAIN 
		select distinct d.dept_no
		from departments d, dept_emp de where de.dept_no=d.dept_no;
```

조회하려는 값은 dept_no 인데, departments 테이블과 dept_emp 테이블에 모두 존재하는 dept_no 만 중복없이 유니크하게 가져오는 쿼리 임

![스크린샷 2023-09-27 오전 12.17.06.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_12.17.06.png)

distinct 를 처리하기 위해 조인하지 않아도 되는 항목은 무시하고 꼭 필요한 것만 조인함, dept_emp 테이블 에서 꼭 필요한 레코드만 읽었음.

![스크린샷 2023-09-27 오전 12.17.31.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_12.17.31.png)

### 10.3.12.4 FirstMatch

세미 조인의 여러 최적화중 FirstMatch 전략이 사용되면 FirstMatch(table_name) 메세지를 출력함
(세미조인: [http://wiki.gurubee.net/pages/viewpage.action?pageId=1966761](http://wiki.gurubee.net/pages/viewpage.action?pageId=1966761))

(firstmatch - 서브쿼리가 아니라 조인으로 풀어서 실행하는것)

```sql
mysql > EXPLAIN select *
			from employees e
			where e.first_name='Matt'
				and e.emp_no in (
					select t.emp_no from titles t
					where t.from_date between '1995-01-01' and '1995-01-30'
				);
```

![스크린샷 2023-09-27 오전 12.35.12.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_12.35.12.png)

employees 테이블 기준으로 titles 테이블의 첫번째로 일치하는 한 건만 검색한다는 의미임.

### 10.3.12.5 Full scan on NULL Key

`col1 IN (select col2 from ...)` 과 같은 쿼리에서 자주 발생함.

만약 col1 의 값이 NULL 이라면.. `NULL IN (select co2 from…)` 로 바뀜

- 서브쿼리가 1건이라도 결과 레코드를 가진다면 최종 비교 결과는 NULL
- 서브쿼리가 1건도 결과 레코드를 가지지 않는다면 최종 비교 결과는 FALSE

Full scan on NULL Key 는 MySQL 서버가 쿼리를 실행중 col1 이 NULL 을 만나면 차선책으로 서브쿼리 테이블에 대해서 풀 테이블 스캔을 사용할 것이라는 사실을 알려주는 키워드임

```sql
mysql > EXPLAIN
	select d.depo_no,
				NULL IN (select id.dept_name from departments d2)
	from departments d1;
```

![스크린샷 2023-09-27 오전 1.36.12.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_1.36.12.png)

NULL 비교 규칙을 무시해도 된다면 `col1 is not null` 을 추가하면됨

### 10.3.12.6 Impossible Having

쿼리에 사용된 having 절의 만족하는 레코드가 없을때 `Impossible Having` 키워드가 표시됨

```sql
mysql > EXPLAIN
	select e.emp_no, count(*) as cnt
	...
	group by e.emp_no
	having e.emp_no is NULL;
```

`e.emp_no is null` 은 조건에 만족할 가능성이 없으므로

Extra 칼럼에 `Impossible HAVING` 이라는 키워드를 표시함

⇒ 해당 키워드의 경우 쿼리가 제대로 작성되지 못한 경우가 대부분임, 쿼리 내용을 다시 점검하는 것이 좋음

### 10.3.12.7 Impossible WHERE

WHERE 조건이 항상 FALSE 가 될수밖에 없는 경우임

```sql
mysql > EXPLAIN
	select * from employees where emp_no is null;
```

Extra 칼럼에 `Impossible WHERE` 이 노출됨

### 10.3.12.8 LooseScan

LooseScan 최적화 전략이 사용되면 표시됨

(서브쿼리 부분과 아웃터 쿼리가 조인하여 찾는 방법)

(서브쿼리 부분이 루스 인덱스 스캔을 사용 할 수 있는 형태여야함.)

```sql
mysql > EXPLAIN
		select * from departments d where d.dept_no in (
				select de.dept_no from dept_emp de);
```

### 10.3.12.9 No matching min/max row

MIN() 이나 MAX()와 같은 집합 함수가 있는 쿼리에서 한건도 없을때 노출됨

```sql
mysql > EXPLAIN
	select MIN(dept_no), MAX(dept_no)
	from dept_emp where dept_no ='';
```

### 10.3.12.10 no matching row in const table

const 방법으로 접근할때 일치하는 레코드가 없는 경우

```sql
mysql > EXPLAIN
	select *
	from dept_emp de,
		(select emp_no from employees where emp_no=0) tb1
	where tb1.emp_no=de.emp_no and de.dept_no='d005';
```

### 10.3.12.11 No matching rows after partition pruning

파티션된 테이블에 대한 update 또는 delete 할 대상 레코드가 없을때 표시됨

```sql
mysql > CREATE TABLE employees_2 (
			emp_no int NOT NULL,
			...
			...
			hire_date DATE NOT NULL,
			PRIMARY KEY (emp_no, hire_date)
) PARTITION BY RANGE COLUMNS(hire_date)
(PARTITION p1986_1990 VALUES LESS THEN ('1990-01-01'),
PARTITION p1991_1995 VALUES LESS THEN ('1996-01-01'),
PARTITION p1996_2000 VALUES LESS THEN ('2000-01-01'),
PARTITION p2001_2005 VALUES LESS THEN ('2006-01-01'));	

mysql > EXPLAIN
	delete from employees_parted where hire_date >= '2020-01-01';
```

### 10.3.12.12 No tables used

from 절이 없거나 from dual 의 쿼리 실행일 경우

다른 DBMS 와 달리 MySQL 서버는 from 절이 없는 쿼리도 허용 됨

### 10.3.12.13 Not exists

A 테이블에는 존재하는데 B 에는 없는 값을 조회하는 경우 `안티 조인` 이라고함
`NOT IN(subquery)` 나 `NOT EXISTS` 연산자를 주로 사용함.

일반적으로 `not in`, `not exists` 로 처리해야하지만, 레코드 건수가 많을때 아우터 조인을 이용하면 빠른 성능을 낼 수 있음 

ex) dept_emp 있지만 departments 에 없는 dept_no 조회

```sql
mysql > EXPLAIN
	select *
	from dept_emp de
		left join departments d on de.dept_no=d.dept_no
	where d.ept_no is null;
```

`안티 조인` 을 수행하는 쿼리에서 `Not exists` 메세지가 표시됨

![스크린샷 2023-09-27 오전 2.42.51.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_2.42.51.png)

### 10.3.12.14 Plan isn’t ready yet

다른 커넥션에서 실행 중인 쿼리의 실행 계획을 볼 수 있음

![스크린샷 2023-09-27 오전 2.47.09.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_2.47.09.png)

![스크린샷 2023-09-27 오전 2.48.23.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_2.48.23.png)

`EXPLAIN FOR CONNECTION` 명령을 했을때, 커넥션에서 아직 쿼리의 실행계획을 수행하지 못하였다면, `Plan isn’t ready yet` 메시지가 표시됨

### 10.3.12.15 Range checked for each record(index map:N)

```sql
mysql > EXPLAIN
	select *
	from employees e1, employees e2
	where e2.emp_no >= e1.emp_no;
```

1번부터 1억번 까지 사번이 있다고 할때 

e1.emp_no=1 인 경우, e2 테이블의 1억건을 읽어야함

e1.emp_no=1억 인 경우, e2테이블을 1건만 읽으면됨

레코드 마다, 인덱스 레인지 스캔을 체크해야함. (Range checked for each record)

![스크린샷 2023-09-27 오전 3.00.46.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_3.00.46.png)

![스크린샷 2023-09-27 오전 8.20.29.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_8.20.29.png)

Extra 에 `Range checked for each record(index map: 0x1)` 이라고 표시됨

0x1 은 16진수로 , 이진수로 바꿔 해석해야함

이진수로 바꾸면 1 이고, e2  `테이블의 첫번째 인덱스`를 사용할지 아니면 `테이블을 풀 스캔`할지 매 레코드 단위로 결정하면서 처리됨

첫번째 인덱스란 `show create table employees` 명령어로 조회했을때 제일 먼저 출력되는 인덱스임

`type 에 ALL 로 표시되어 풀스캔으로 해석하기 쉽지만`, `Range checked for each record` 가 표시되면 `index map` 에 표시된 `인덱스를 사용할지 검토`해서 도움되지 않으면 최종적으로 풀스캔(ALL) 을 하겠다는 의미임.

---

index map

![스크린샷 2023-09-27 오전 8.26.56.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_8.26.56.png)

0x19인경우 이진 → 11001 

### 10.3.12.16 Recursive

8.0 부터 CTE(Common Table Expression) 을 이용해 재귀 쿼리를 작성할 수 있게 됨

재귀쿼리는 with 구문을 이용해 CTE 를 사용하면됨

```sql
mysql > with recursive cte (n) as
			(
				select 1 
				union all 
				select n+1 from cte where n<5
			)
			select * from cte;
```

1. n 이라는 칼럼 하나를 cte 라는 이름의 내부 임시 테이블을 생성
2. n 칼럼의 값이 1부터 5씩 증가해서 레코드 5건을 만들어서 cte 내부 임시테이블에 저장

### 10.3.12.17 Rematerialize

8.0 부터 래터럴 조인 기능이 추가되었음

`래터럴 조인`(realmysql 2권에 나오는듯) 로 조인되는 테이블은 `선행 테이블의 레코드별로 서브쿼리`를 해서 `그 결과를 임시테이블`로 저장함

```sql
mysql > EXPLAIN
	select * from employees e
		left join lateral 
				(select * from salaries s
					where s.emp_no=e.emp_no
					order by s.form_date desc limit 2) s2 on s2.emp_no=e.emp_no
	where e.first_name='Matt';
```

### 10.3.12.18 Select tables optimized away

`MIN() 또는 MAX() 만 Select 절`에 사용되거나

`group by` 로 `MIN(), MAX()` 를 조회하는 쿼리가 `인덱스`를 오름차순 또는 내림차순으로 

`1건만 읽는 형태의 최적화`가 적용된다면 

```sql
mysql > EXPLAIN
		select max(emp_no), min(emp_no) from employees;

mysql > EXPLAIN
		select max(from_date), min(from_date) 
			from salaries 
			where emp_no=10002;
```

![스크린샷 2023-09-27 오전 8.42.52.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_8.42.52.png)

인덱스의 첫번째와 마지막만 읽어 최적화가 가능함

![스크린샷 2023-09-27 오전 8.42.29.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_8.42.29.png)

(emp_no, from_date) 가 인덱스로 생성되어 있어 emp_no=10002 중에 오름차순, 내림차순 하나만 조회하면 됨, 따라서 최적화가 가능함

### 10.3.12.19 Start temporary, End temporary

Duplicate Weed-out 최적화 전략은 불필요한 중복건을 제거하기 위해서 내부 임시테이블을 사용하는것임

<aside>
💡 ([https://velog.io/@code-10/세미-조인-서브.-쿼리-Duplicate-Weedout-최적화](https://velog.io/@code-10/%EC%84%B8%EB%AF%B8-%EC%A1%B0%EC%9D%B8-%EC%84%9C%EB%B8%8C.-%EC%BF%BC%EB%A6%AC-Duplicate-Weedout-%EC%B5%9C%EC%A0%81%ED%99%94))

Weedout 최적화 알고리즘 처리과정은 다음과 같다.

1. salaries 테이블의 ix_salary 인덱스를 스캔해서 salary 가 150000 보다 큰 사원을 검색하여 employees 테이블의 조인을 실행
2. 조인된 결과를 임시 테이블에 저장
3. 임시 테이블에 저장된 결과에서 emp_no 기준으로 중복 제거
4. 중복을 제거하고 남은 레코드를 최종적으로 반환
</aside>

이때 조인되어 내부 임시테이블에 저장되는 테이블을 식별할 수 있게 조인의 첫번째 테이블에 `Start temporary` 문구를 보여줌,

조인이 끝나는 부분에 `End temporary` 문구를 표시해줌

```sql
mysql > EXPLAIN
	select * from employees e
	where e.emp_no 
			in (select s.emp_no from salaries s where s.salary > 150000);
```

![스크린샷 2023-09-27 오전 8.49.03.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_8.49.03.png)

이 예제는 salaries 테이블 부터 시작해서 employees 테이블까지의 내용을 임시 테이블에 저장한다는 의미임

### 10.3.12.20 unique row not found

두개의 테이블이 각각 유니크 칼럼으로 아우터 조인을 수행하는 쿼리에서 아우터 테이블에 일치하는 레코드가 존재하지 않을때

### 10.3.12.21 Using filesort

Order by 처리가 인덱스를 사용하지 못할때임.

 Mysql 옵티마이저는 레코드를 읽어서 `소트버퍼` 에 복사하고, 정렬해서 그 결과를 클라이언트에 보냄

`Using filesort` 가 출력되는 쿼리는 많은 부하를 이르키므로 가능하다면 쿼리를 튜닝하거나 인덱스를 생성하는 것이 좋음

### 10.3.12.22 Using index (커버링 인덱스)

데이터 파일을 전혀 읽지 않고 인덱스만 읽어서 쿼리를 모두 처리할 수 있을때

인덱스를 이용해 처리하는 쿼리에서 가장 큰 부하를 차지하는 부분은 인덱스 검색에서 일치하는 키값들의 레코드를 읽기 위해 `데이터 파일을 검색하는 작업`임

```sql
mysql > EXPLAIN
	select first_name, birth_date
	from employees
	where first_name between 'Babette' and 'Gad';
```

(Using where) - `birth_date` 칼럼값을 가져오기위해 데이터 페이지를 읽어야함

```sql
mysql > EXPLAIN
	select first_name
	from employees
	where first_name between 'Babette' and 'Gad';
```

(Using where, Using index)

```sql
mysql > EXPLAIN
	select emp_no, first_name
	from empoyees
	where first_name between 'Babette' and 'Gad';
```

(Using where, Using index)

### 10.3.12.23 Using index condition

인덱스 컨디션 푸시 다운 최적화를 사용하면 표시됨

(인덱스 컨기션 푸시 다운: [https://neverfadeaway.tistory.com/76](https://neverfadeaway.tistory.com/76))

```sql
mysql > select * from employees 
where last_name='Acton' and first_name like '%sal';
```

### 10.3.1.24 Using index for group-by

Group by 가 처리가 인덱스를 이용할때 나타남

### 13.3.12.24.1 타이트 인덱스 스캔(인덱스 스캔)을 통한 GROUP BY 처리

AVG(), SUM(), COUNT() 를 하면 듬성듬성 읽을 수 없음

따라서 Using index for group-by 메세지가 나오지 않음

### 10.3.12.24.2 루스 인덱스 스캔을 통한 GROUP BY 처리

단일 칼럼으로 구성된 인덱스에서 `그루핑 칼럼` 말고는 아무것도 조회하지 않는 쿼리에서 루스 인덱스 스캔을 쓸 수 있음

MIN(), MAX() 같이 조회하는 값이 인덱스의 첫번째만 읽어도 된다면 `루스 인텍스 스캔`이 될 수 있음

```sql
mysql > EXPLAIN
		select emp_no, min(from_date) as first_changed_date,
					max(from_date) as last_changed_date
		from salaries
		group by emp_no;
```

### 10.3.12.25 Using index for skip scan

인덱스 스킵 스캔 최적화를 사용하면 표시됨

```sql
mysql > alter table employees
		add index ix_gender_birthdate (gender, birth_date);

mysql > explain
			select gender, birth_date
			from employees
			where birth_date >= '1965-02-01';
```

### 10.3.12.26 Using join buffer(Block Nested Loop , Batched Key Access, hash join)

조인이 수행될때 드리븐 테이블에 검색을 위한 적절한 인덱스가 없다면 MySQL `블록 네스티드 루프 조인` 이나 `해시 조인`을 사용함

이때 MySQL 서버는 `조인 버퍼`를 사용함

이때 `Using join buffer` 라는 메세지가 표시됨

### 10.3.12.27 Using MRR

MySQL 엔진은 실행 계획을 수립하고 그 실행계획에 맞게 `스토리지 엔진의 API 를 호출`해서 쿼리를 처리함

즉, 실제 매번 읽어서 반환하는 레코드가 동일 페이지에 있다고 하더라도 레코드 단위로 API 호출이 필요한 것임

MRR - multi range read

MySQL 엔진이 `여러개의 키`값을 한번에 `스토리지 엔진`으로 전달함, 스토리지 엔진은 넘겨받은 키값들을 정렬해서 `최소한의 페이지만 접근`만 으로 `필요한 레코드`를 읽을 수 있게 최적화함

```sql
mysql > EXPLAIN 
	select /*+ JOIN_ORDER(s, e) */ *
	from employees e,
			salaries s
	where e.first_name='Matt'
			and e.fire_date between '1990-01-01' and '1991-01-01'
			and s.emp_no=e.emp_no
			and s.from_date between '1990-01-01' and '1991-01-01'
			and s.salary between 50000 and 60000;
```

![스크린샷 2023-09-27 오전 9.57.04.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_9.57.04.png)

### 10.3.12.28 Using sort_union(..), Using union(…), Using intersect(..)

index_merge 접근 방법 으로 실행되는 경우 , 2개 이상의 인덱스가 동시에 사용 될 수 있음

어떻게 병합했는지 아래 3개중 하나의 메세지를 선택함

- Using intersect(..) : and 로 연결된경우,처리결과에서 교집합을 추출해 내는 작업을 수행했음
- Using union(..): or 로 연결된 경우, 합집합으로 추출해 내는 작업을 수행했음
- Using sort_union(..):  Using union 이지만 처리될 수 없는 경우 (or 로 연결된 상태적으로 대량의 range 조건들)
    
    
    Using union 과 Using sort_union 의 차이점은 Using sort_union 은 프라이머리 키만 먼저 읽어서 정렬하고 병합 한 후, 레코드를 읽어서 반환할 수 있다는 것임
    

### 10.3.12.29 Using temporary

MySQL 서버에서 쿼리를 처리하는 동안 중간 결과를 담아 두기 위해 임시 테이블을 사용함

임시 테이블은 메모리상에 생성될 수도 있고, 디스크상에 생성될 수도 있음.

이때 사용된 임시 테이블이 메모리에 생성됐는지, 디스크에 생성됐는지는 실행 계획만으로 판단 할 수 없음

### 10.3.12.30 Using where

Mysql 엔진 레이어에서 별도의 가공을 해서 필터링 작업을 처리한 경우 표시됨

필터링이나 가공이 없이 그 데이터를 그대로 전달하면 Using where 가 표시되지 않음

![스크린샷 2023-09-27 오전 10.07.08.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_10.07.08.png)

```sql
mysql > EXPLAIN
	select *
	from employees 
	where emp_no between 10001 and 10100
		and gender='F';
```

(레코드가 37건 나옴)

100 건(emp_no 10001~10100) 을 가져와 63건은 필터링해서 버렸다는 의미임

### 10.3.12.31 Zero limit

MySQL 서버에서 데이터 값이 아닌 쿼리 결과값의 메타데이터만 필요한 경우도 있음

즉, 쿼리의 결과가 몇개 칼럼을 가지고, 각 칼럼의 타입은 무엇인지 등의 정보만 필요한 경우

⇒ 마지막에 LIMIT 0 을 사용하면 됨

MySQL 옵티마이저는 사용자의 의도를 알아채고, 레코드는 전혀 읽지 않고 메타 정보만 반환함

```sql
mysql > EXPLAIN select * from employees LIMIT 0;
```

![스크린샷 2023-09-27 오전 10.14.54.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_10.14.54.png)

---

`select * from stl_cm_banner limit 0;` 아무결과 안나옴 

아래는 explain 추가

![스크린샷 2023-09-27 오전 10.16.22.png](Chapter%2010%20%E1%84%89%E1%85%B5%E1%86%AF%E1%84%92%E1%85%A2%E1%86%BC%E1%84%80%E1%85%A8%E1%84%92%E1%85%AC%E1%86%A8%20e97a8a81bfab4921a5e61166a4acafa7/%25E1%2584%2589%25E1%2585%25B3%25E1%2584%258F%25E1%2585%25B3%25E1%2584%2585%25E1%2585%25B5%25E1%2586%25AB%25E1%2584%2589%25E1%2585%25A3%25E1%2586%25BA_2023-09-27_%25E1%2584%258B%25E1%2585%25A9%25E1%2584%258C%25E1%2585%25A5%25E1%2586%25AB_10.16.22.png)