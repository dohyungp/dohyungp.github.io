---
layout: default
title: BigQuery SESSION TABLE 사용하기
---

## BigQuery SESSION TABLE 사용하기

### 1. BigQuery의 쿼리 결과 캐싱방식
최근 BigQuery를 잘 쓰기 위한 방식들을 연구하고 있는데 쓸데없는데 신박한 방식이 생각나 메모용으로 공유해봅니다(전문가가 쓴 글이 아님 주의).
기본적으로 BigQuery는 SELECT절의 결과를 세션 라이프사이클동안 캐싱해서 사용가능하도록 내부적으로 temp table을 만듭니다([참고](https://cloud.google.com/bigquery/docs/writing-results?hl=ko#temporary_and_permanent_tables)).

이 테이블에 대한 정보는 콘솔에서는 `쿼리 결과 > 작업정보 탭 > 대상 테이블`에서 확인 가능합니다.

### 2. BigQuery의 TEMP TABLE 생성방식 및 접근방식
BigQuery에서는 이 temp table을 사용자도 만들 수 있도록 `CRAETE TEMP TABLE` 구문을 지원하고 있습니다. 이 TEMP TABLE 역시 세션의 라이프사이클동안 사용가능합니다. 
그리고 이 테이블은 명시적으로 `_SESSION.{TEMP_TABLE_NAME}`로 사용가능한데요. 아쉽게도 이를 API레벨에서는 사용하기가 어렵습니다. 쉽게 표현하자면 콘솔에서는 가능한 방식인데 Java나 Python등을 통한 client에서 CREATE TEMP TABLE은 지원하는 방식이 아니라는 뜻입니다.

실제로 아래와 같이 [콘솔에서는 동작하는 방식](https://cloud.google.com/bigquery/docs/reference/standard-sql/data-definition-language?hl=ko#temporary_tables)을,

```sql
-- Create a temp table
CREATE TEMP TABLE t1 (x INT64);

-- Create a temp table using the `_SESSION` qualifier
CREATE TEMP TABLE _SESSION.t2 (x INT64);

-- Select from a temporary table using the `_SESSION` qualifier
SELECT * FROM _SESSION.t1;
```

python에서 사용하게 되면 400에러를 냅니다.

```py
client.query('CREATE TEMP TABLE t1 (x INT64);')
```

따라서 이 방식을 유사하게 동작시키는 방법을 고민하게 되었습니다.

### 3. 프로그래매틱하게 BigQuery Temp table 만들기

사실 프로그래매틱하게 Temp table을 만드는 방식이 여러가지가 있습니다. 

첫번째로 SP를 만드는 방식입니다. 빅쿼리는 SP를 지원하므로 SP를 만들고 client는 SP만 조회하는 방식을 택할 수 있습니다. 하지만 BigQuery 내부적으로 SP를 실행할때 DAG로 job을 만들고 실행시키며 테스트해본 결과 DAG plan을 만드는데 꽤 많은 시간을 사용하는 것으로 보입니다.
두번째로는 Expiration이 매우 짧은 Table을 만드는 가짜 Temp table을 만드는 방식입니다. 하지만 엄밀하게 임시테이블과 다르므로 이 방식은 개인적으로 선호하는 방식이 아니었습니다.
따라서 마지막 방식으로 SELECT로 session의 임시테이블을 사용하는 것을 생각해보게 되었습니다.

[스택오버플로우](https://stackoverflow.com/questions/28147406/how-to-get-name-of-a-temporary-table-in-bigquery-using-api)글을 보다가 생각난 방식입니다.

query_job의 `_properties`를 가져와 그 안에 임시테이블 정보를 가져온 다음 그 값으로 cached_query_job을 다시 만드는 구조입니다. 
```py
sql = 'SELECT 1'
query_job = client.query(sql)
query_conf = query_job._properties['configuration']
project, dataset, table = query_conf['projectId'], query_conf['datasetId'], query_conf['tableId']
cached_query_job = client.query(f'SELECT * FROM {project}.{dataset}.{table}')
for row in cached_query_job:
    print(row)
```

### 4. 이 방식이 어떻게 쓰일 수 있을까?

위의 코드처럼 쓸것이라면 그다지 필요없는 구조이나, 만약 추가적인 가공이 필요한데 마치 sp처럼 step by step으로 진행해야 하고 프로그래매틱해야 한다면 검토 정도는 해볼 수 있는 테크닉이 되겠습니다.
