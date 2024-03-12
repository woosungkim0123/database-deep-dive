# 인덱스

## 인덱스 개념

인덱스는 데이터베이스에서 빠르게 정보를 찾을 수 있도록 도와주는 데이터 구조로써, 책의 목차나 사전의 색인과 비슷한 역할을 합니다.

전체 내용을 살펴보는 대신 특정 내용을 가리키는 정렬된 리스트를 확인함으로써 더 빠르게 원하는 정보를 찾을 수 있습니다.

<br>

## 인덱스 적용 속도 비교

인덱스를 적용하지 않은 쿼리와 적용한 쿼리의 속도를 비교해보겠습니다.

MongoDB의 인덱스는 전형적인 관계형 데이터베이스의 인덱스와 거의 동일하게 작동합니다.

### 임시 데이터 생성

```javascript
const batchSize = 1000; // 한 번에 삽입할 문서의 수
const totalDocuments = 1000000; // 총 문서의 수
let documents = [];

for (let i = 0; i < totalDocuments; i++) {
  documents.push({
    "i": i,
    "username": "user" + i,
    "age": Math.floor(Math.random() * 120),
    "created": new Date()
  });

  // 배치 사이즈에 도달하거나 마지막 문서에 도달한 경우 데이터베이스에 삽입
  if (documents.length === batchSize || i === totalDocuments - 1) {
    db.users.insertMany(documents);
    documents = []; // 다음 배치를 위해 배열 초기화
  }
}
```

### 인덱스 적용 전 성능 체크

인덱스를 사용하지 않은 쿼리를 `컬렉션 스캔`이라고 합니다. 컬렉션 스캔은 모든 문서를 읽어야 하므로 성능이 떨어집니다.

`explain()` 함수를 사용하여 쿼리의 실행 계획을 확인해보겠습니다. 

인덱스를 사용하지 않는다면 `"COLLSCAN"`이라고 표시되고 사용한다면 `"IXSCAN"`이라고 표시됩니다.

```shell
db.users.find({"username" : "user101"}).explain("executionStats")
```

```javascript
// 필요한 부분만 추출
executionStats = {
    nReturned: 1, // 반환된 문서의 수
    executionTimeMillis: 359, // 쿼리하는데 걸린 시간
    totalDocsExamined: 1000000, // 쿼리를 실행하면서 살펴본 문서의 수
}
```

MongoDB가 컬렉션 내 모든 문서를 검색하여 일치하는 문서를 찾은 것을 확인할 수 있습니다.

### 인덱스 적용 후 성능 체크

```shell
db.users.createIndex({"username" : 1}) # 1은 오름차순
```

```shell
db.users.find({"username" : "user101"}).explain("executionStats")
```

```javascript
// 필요한 부분만 추출
executionStats =  {
    nReturned: 1,
    executionTimeMillis: 6,
    totalDocsExamined: 1,
}
```

쿼리 시간이 359초에서 6초로 줄어든 것을 확인할 수 있습니다.

### 인덱스 적용에 대한 결론

MongoDB의 쿼리 성능을 최적화하기 위해 인덱스의 사용은 필수적입니다. 그러나 애플리케이션에서 사용되는 모든 쿼리 패턴에 대해 인덱스를 생성하는 것은 매우 비효율적입니다.

자주 사용되며 성능에 중대한 영향을 미치는 쿼리 패턴에 대해서만 인덱스를 생성하는 것이 효율적일 것입니다.

> 쿼리 패턴 (Query Pattern)
> 
> 데이터베이스에 요청되는 특정 유형의 쿼리를 의미합니다. 단순한 개별 쿼리가 아닌 비슷한 유형이나 구조를 가진 쿼리의 집합을 말합니다.  
> 사용자의 이름으로 사용자 정보를 조회하는 쿼리, 특정 날짜 범위 내에서 이벤트를 검색하는 쿼리 이런 예시들은 각각 다른 쿼리 패턴을 나타냅니다.


### 인덱스 단점

인덱싱된 필드를 변경하는 쓰기(INSERT, UPDATE, DELETE) 작업이 더 오래 걸리는 단점이 있습니다.

인덱싱된 필드 데이터가 변경될 때마다 인덱스도 변경되어야 하기 때문입니다. 

> ps. 인덱싱된 필드가 아닌 다른 필드만 변경하거나 삭제하는 경우 인덱스로 인한 성능저하는 없습니다.

