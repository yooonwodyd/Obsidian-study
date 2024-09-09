sql은 **선언형 질의언어이다**.
즉, **무엇을 할지**만 기술하고, **어떻게 할지**는 데이터베이스 엔진이 알아서 결정한다.

따라서 우리가 SQL문을 작성하면, DBMS 내부 엔진이 어떻게 해당 결과를 만들어낼 지 프로시저(절차)를 작성한다.

이때 해당 프로시저를 작성하는 것이 **SQL 옵티마이저**이다. 때문에 SQL 옵티마이저가 어떻게 프로시저를 작성했냐에 따라 쿼리의 성능이 달라질 수 있다. 

> sql옵티마이저에 대한 다양한 내용 중 실제로 옵티마이저가 어떤 기준으로 실행계획을 선택하는지, 이를 내가 강제할 수 있는지 Mysql 예제를 통해 알아보는 것을 목표로 한다.


# 1.예제 데이터
*처음에는 H2를 사용하려고 했지만, H2의 explain 키워드의 경우 쿼리의 실행 비용을 보여주지 않기 때문에 Mysql을 사용하기로 했다.*

먼저, 다음과 같은 테이블을 만들었다.
![[스크린샷 2024-09-05 오후 4.28.25.png]]

해당 데이터에서 아래와 같은 두개의 인덱스를 추가로 만들었다.
![[스크린샷 2024-09-05 오후 6.29.21.png]]
![[스크린샷 2024-09-05 오후 6.33.05.png]]

이때, '윤지우' 라는 학생을 찾아서 학생 정보를 조회하는 다음의 쿼리에서, 옵티마이저는 어떤 실행계획을 가지고 있을까?
```sql
EXPLAIN FORMAT=JSON SELECT * FROM student_info WHERE sname = '윤지우';
```

```json
{
  "query_block": {
    "select_id": 1,
    "cost": 0.00345856,
    "nested_loop": [
      {
        "table": {
          "table_name": "student_info",
          "access_type": "ref",
          "possible_keys": ["idx_name"],
          "key": "idx_name",
          "key_length": "152",
          "used_key_parts": ["sname"],
          "ref": ["const"],
          "loops": 1,
          "rows": 1,
          "cost": 0.00345856,
          "filtered": 100,
          "index_condition": "student_info.sname = '윤지우'"
        }
      }
    ]
  }
}
```

내용을 보니, idx_name 인덱스를 사용해서 조회를 한다는 것을 알 수 있다.

그렇다면 `CREATE INDEX idx_sname_height ON student_info(sname,height);` 다음 인덱스를 추가하면 어떻게 될까?

```json
{
  "query_block": {
    "select_id": 1,
    "cost": 0.00345856,
    "nested_loop": [
      {
        "table": {
          "table_name": "student_info",
          "access_type": "ref",
          "possible_keys": ["idx_name", "idx_sname_height"],
          "key": "idx_name",
          "key_length": "152",
          "used_key_parts": ["sname"],
          "ref": ["const"],
          "loops": 1,
          "rows": 1,
          "cost": 0.00345856,
          "filtered": 100,
          "index_condition": "student_info.sname = '윤지우'"
        }
      }
    ]
  }
}
```
사용 가능한 인덱스가 두 개로 늘었다. 
이를 통해 우리가 어떤 인덱스를 사용할 지 선택하지 않아도,
sql 엔진 내부에서 어떤 인덱스를 선택해서 조회할 지 정한다는 걸 알 수 있다.

sql 옵티마이저가 선택하는 인덱스가 비용이 가장 적게 드는 방법이라면 좋겠지만, 그렇지 않을 수 도 있다. 우리는 다음과 같은 방법으로 사용할 인덱스를 선택할 수 있다.

![[스크린샷 2024-09-05 오후 7.07.29.png]]

## 그렇다면, 옵티마이저는 어떻게 실행계획을 선택하는가?

책 *친절한 SQL 튜닝*에 나오는 최적화 단계를 요약하면 다음과 같다.

1. 사용자로부터 전달 받은 쿼리를 수행 할 수 있는 실행계획을 찾는다.
2. 해당 계획들의 예상 비용을 산정한다.(오브젝트 통계, 시스템 통계정보 등 활용)
3. 가장 낮은 비용의 실행계획을 선택한다.