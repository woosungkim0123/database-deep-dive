# 복합 인덱스

## 개념

상당 수의 쿼리 패턴은 두개 이상의 키를 기반으로 인덱스를 작성해야 합니다.

복합 인덱스는 2개 이상의 필드로 구성된 인덱스로 쿼리에서 정렬이나 검색 조건이 여러 개 있을 때 유용 합니다.

## 성능 비교

### 복합 인덱스 적용 전 성능 체크

```shell
db.users.createIndex({"username" : 1})
```

```shell
db.users.find().sort({"age" : 1, "username" : 1}).explain("executionStats") 
# executionTimeMillis: 4459
```

"age"로 정렬한 후 "username"으로 정렬하는데 기존 인덱스는 별로 도움이 되지 않습니다.

해당 쿼리를 최적화하려면 "age"와 "username"을 기반으로 `복합 인덱스`를 생성해야 합니다.

### 복합 인덱스 적용 후 성능 체크

```shell
db.users.createIndex({"age" : 1, "username" : 1}) 
```

```shell
db.users.find().sort({"age" : 1, "username" : 1}).explain("executionStats")
# executionTimeMillis: 1772
```

복합 인덱스를 사용하여 쿼리 시간이 4459초에서 1772초로 줄어든 것을 확인할 수 있습니다.

### 중간 질문

**Question**

`{ "age" : 1, "username" : 1 }` 과 `{ "gender" : 1, "username" : 1}` 복합 인덱스가 존재할 때 정렬이 "age" : 1, "gender" : 1 인 경우 어떻게 처리합니까?

<details>
<summary>Answer</summary>

MongoDB의 인덱스는 왼쪽에서 오른쪽으로 방향성을 가집니다.

따라서 첫번째 정렬 조건과 일치하는 `{ "age" : 1, "username" : 1}` 인덱스를 사용할 것입니다. 그러나 gender에 대해서는 추가적인 필터링 작업을 수행해야합니다.

최적의 성능을 위해서는 `{ "age" : 1, "gender" : 1 }` 인덱스를 생성하는 것이 좋을 수 있습니다.

</details>

<br>

## 복합 인덱스 형태

컬렉션에 `{"age" : 1, "username" : 1}`로 인덱스를 만들면 다음과 같은 형태로 표현됩니다.

```
[0, "user100020"] -> 8623513776
[0, "user1002"] -> 8599246768
[0, "user100388"] -> 862350880
...
[1, "user100113"] -> 8623513208
[1, "user100280"] -> 8599246405
[1, "user100388"] -> 8623581744
...
[30, "user1002"] -> 85992467200
[30, "user100388"] -> 862350140
[30, "user100626"] -> 8623513745
```

각 인덱스 항목은 나이와 사용자명을 포함하고 레코드 식별자를 가리킵니다.

레코드 식별자는 내부에서 스토리지 엔진에 의해 사용되며 문서 데이터를 찾습니다.

<br>

## 쿼리 종류

MongoDB가 실행하는 쿼리 종류에따라 인덱스를 사용하는 방법이 다릅니다.

### 동등 쿼리

```shell
db.users.find({"age" : 21})
```

동등 쿼리는 특정 필드가 주어진 값과 정확히 일치하는 문서를 찾는 것을 의미합니다.

예시는 users 컬렉션에서 age 값이 정확히 21인 모든 문서를 찾는 동등 쿼리입니다.


```shell
db.users.find({"age" : 21}).sort({"username" : -1})
```

```
[21, "user100154"] -> 8623513723
[21, "user100266"] -> 8599246414
```

`{"age" : 21}`과 일치하는 마지막 항목부터 순서대로 인덱스를 탐색합니다. MongoDB는 인덱스를 어느 방향으로도 쉽게 탐색하므로 정렬 방향은 문제가 되지 않습니다.

### 범위 쿼리

범위 쿼리는 주어진 범위 내의 값과 일치하는 문서를 찾는 쿼리를 의미합니다.

```shell
db.users.find({"age" : {"$gte" : 21, "$lte" : 30}})
```

인덱스에 있는 첫번째 키인 "age"를 사용해 범위 쿼리를 수행합니다.

`{"age" : 1, "username" : 1}` 복합 인덱스는 age 필드와 username 필드를 포함하므로, age 필드를 사용하는 쿼리는 해당 인덱스를 사용할 수 있습니다.

```
[21, "user100154"] -> 8623513723
[21, "user100266"] -> 8599246414
...
[30, "user100154"] -> 8623513723
[30, "user100266"] -> 8599246414
```

### 다중 값 쿼리와 정렬

범위 쿼리와 정렬을 함께 사용하는 쿼리를 의미합니다.

```shell
db.users.find({"age" : {"$gte" : 21, "$lte" : 30}}).sort({"username" : 1})
```

인덱스 `{"age" : 1, "username" : 1}`를 사용하면, age 조건에 맞는 문서를 효율적으로 찾을 수 있습니다.

그러나 사용자명(username)에 대해서는 추가적인 `인 메모리 정렬`이 필요할 수 있습니다.

```
[22, "user100101"] -> 8623513723
[22, "user100202"] -> 8599246414
[22, "user100303"] -> 8599246414
...
[30, "user100100"] -> 8623513723
[30, "user100200"]
```

반대 순서인 인덱스 `{"username" : 1, "age" : 1}`를 사용하면, "username"을 기준으로 이미 정렬된 상태에서 나이 조건을 만족하는 문서를 효율적으로 필터링 할 수 있습니다.

원하는 순서로 정렬된 결과를 바로 반환할 수 있으며, `인 메모리 정렬`을 피할 수 있지만 전체 인덱스를 탐색 해야합니다.

```shell
[user0, 4]
[user1, 67]
# ...
[user10000, 21] -> 8623513723
# ...
[user100001, 52]
[user100002, 21] -> 8623513723
[user100003, 27] -> 8599246414
```

대부분의 어플리케이션에서는 데이터셋의 크기가 크고 계속 증가하는 경우가 많습니다.

범위 쿼리와 정렬을 함께 사용하는 경우, 는, 정렬 키를 복합 인덱스의 첫 번째에 놓는 것이 성능 최적화에 도움이 될 수 있습니다.

<br>

## MongoDB가 인덱스를 선택하는 방법

5개의 인덱스가 있다고 가정하겠습니다.

1. 쿼리가 들어오면 MongoDB는 **쿼리 모양**을 확인하게 됩니다.

   - 쿼리 모양은 검색할 필드, 정렬 여부 등 여러 정보와 관련이 있습니다.

2. 해당 정보를 기반으로 인덱스 후보 집합을 식별합니다. (3개 인덱스가 후보 집합이 되었다고 가정하겠습니다.)


3. MongoDB는 인덱스 후보 마다 쿼리 플랜을 만들고(3개) 병렬 스레드(3개)에서 실행 합니다.

    - 어떤 스레드가 가장 빨리 결과를 반환하는지 확인하기 위해 경쟁을 합니다.


4. 가장 먼저 목표 상태에 도달하는 쿼리 플랜이 선택됩니다.

    - 병렬 스레드에서 실행하여 반환할 떄까지 기간을 `시범 기간`이라고 합니다.


5. 승리한 플랜은 다음번 모양이 같은 쿼리에서 사용하기 위해 캐시에 저장됩니다.

    - 앞으로 동일한 모양을 가진 쿼리는 캐시된 플랜을 사용하여 더 빠르게 실행됩니다.

### 쿼리 플랜 캐시 제거

컬렉션과 인덱스가 변경되면 쿼리 플랜이 캐시에서 제거되고 다시 쿼리 플랜을 실험해 가장 적합한 플랜을 찾습니다.

또한 인덱스를 추가, 삭제, 변경하면 쿼리 플랜이 캐시에서 제거됩니다. 

쿼리 플랜 캐시는 명시적으로 지울 수 있으며 mongod 프로세스를 다시 시작할 때도 삭제됩니다.

<br>

## 복합 인덱스 설계 모범 사례

인덱스를 올바르게 설계하려면 인덱스를 테스트하고 조정해야 합니다. 그러나 그전에 몇가지 모범 사례를 적용해볼 수 있습니다.

> MongoDB 7버전과 책의 내용이 달라 MongoDB 7버전 실험 결과는 마지막에 서술하겠습니다.

### 임시 데이터 생성

100만개의 가상 데이터를 생성 하겠습니다.

```javascript
const batchSize = 1000;
let batch = [];
const scoreTypes = ["exam", "quiz", "homework", "homework"]; 

for (let i = 0; i < 1000000; i++) {
  let scores = scoreTypes.map(type => ({
    type: type,
    score: parseFloat((Math.random() * 100).toFixed(14)) 
  }));
  
  const document = {
    student_id: i,
    class_id: Math.floor(Math.random() * 501), 
    scores: scores
  };

  batch.push(document);

  if (batch.length === batchSize) {
    db.students.insertMany(batch);
    batch = [];
  }
}

if (batch.length > 0) {
  db.students.insertMany(batch);
}
```
```
[
  {
    _id: ObjectId('65e473ca2ca178def2c85297'),
    student_id: 19,
    class_id: 371,
    scores: [
      { type: 'exam', score: 51.80624107950489 },
      { type: 'quiz', score: 64.44871547516604 },
      { type: 'homework', score: 46.75760678103447 },
      { type: 'homework', score: 44.85046424076016 }
    ]
  }
  ...
]
```

### 필터링 중 하나를 기준으로 정렬하는 복합 인덱스

```shell
db.students.find({"student_id" : { "$gt" : 500000}, "class_id" : 54}).sort({"student_id" : 1})
```

필터링 대상인 `student_id`와 `class_id` 중 하나인 `student_id`를 기준으로 정렬하는 쿼리를 예시로 들겠습니다.

```shell
db.students.createIndex({"class_id" : 1})
db.students.createIndex({"student_id" : 1, "class_id" : 1})
```

```shell
db.students.find({"student_id" : { "$gt" : 500000}, "class_id" : 54}).sort({"student_id" : 1}).explain("executionStats")
```

```javascript
executionStats = {
    nReturned: 9903, // 반환 받은 결과의 개수
    executionTimeMillis: 4325, // 쿼리하는데 걸린 시간 (4.3초)
    totalKeysExamined: 850477, // 인덱스를 탐색한 횟수
}
```

`toalKeysExamined`를 `nReturned`와 비교하면 얼마나 많은 인덱스를 통과했는지 알 수 있습니다.

예시에서는 9903개의 문서를 찾기위해 850477개의 인덱스를 탐색했습니다. 이는 사용된 인덱스가 선택성이 높지 않음을 의미합니다.

데이터 필터링이 잘못되었다는 것을 의심할 수 있고 시간도 4.3초나 걸린 것을 확인할 수 있습니다.

> `선택성`이란 인덱스에서 얼마나 많은 데이터를 효과적으로 필터링할 수 있는지를 나타냅니다.

```javascript
winningPlan = { // 선정된 쿼리 플랜
    stage : "FETCH",
    inputStage : {
        stage : "IXSCAN", // 인덱스 스캔
        keyPattern : {"student_id" : 1, "class_id" : 1},
    }
}
```

선정된 쿼리 플랜을 살펴보면 student_id와 class_id를 기반으로 복합 인덱스를 사용했음을 알 수 있습니다.

또한, 입력 단계인 인덱스 스캔이 쿼리와 일치하는 문서의 레코드 ID를 상위 단계인 FETCH에 제공한 것을 확인할 수 있습니다. 
(FETCH 단계에서는 문서를 검색하고 클라이언트가 요청하면 이를 일괄적으로 반환합니다.)

> explain 출력은 쿼리 플랜의 단계를 트리로 표시하기 때문에 각 단게에는 하위 단계 개수에 따라 하나 이상의 입력 단계(inputStage)가 있을 수 있습니다.

> MongoDB 7 버전 이상에서는 예시와 다르게 기본적으로 { "class_id" : 1 } 인덱스를 사용하였습니다.

```javascript
rejectedPlans = [ // 거부된 쿼리 플랜
    {
        stage : "SORT",
        sortPattern : {"student_id" : 1},
    }
]
```

실패한 쿼리 플랜에 SORT 단계가 표시된다면 정렬할 때 인덱스를 사용하지 못했고 `인 메모리 정렬`을 수행했음을 뜻합니다.

**다중값 조건과 동등 조건**

다중 값 조건(`"$gt": 500000`)은 student_id 필드에 대해 넓은 범위의 레코드를 탐색합니다. 
이 조건은 더 많은 레코드를 검사해야 하므로 처리 시간이 길어질 수 있습니다.

동등 조건(`"class_id": 54`)은 필드의 값이 정확히 일치하는 레코드를 찾는 좁은 범위의 검색입니다.
이 조건은 검색해야 할 데이터의 양이 적어져 처리 속도가 향상될 수 있습니다.

복합 인덱스에서 동등 조건을 다중 값 조건보다 먼저 사용하도록 쿼리를 조정하면, 데이터베이스는 먼저 좁은 범위의 레코드를 필터링하여 검색해야 할 데이터의 양을 줄일 수 있습니다.
이로 인해 처리속도가 향상될 수 있습니다.

예시에서 동등 조건을 먼저 사용하도록 인덱스를 조정해보겠습니다.

```shell
db.students.find({"student_id" : { "$gt" : 500000}, "class_id" : 54}).sort({"student_id" : 1}).hint({"class_id" : 1}).explain("executionStats")
```

```javascript
executionStats = {
    nReturned: 9903,
    executionTimeMillis: 272,
    totalKeysExamined: 20076,
}
rejectedPlans: []
```

결과를 보면 850477개에서 20076개로 인덱스를 탐색한 횟수가 줄었고, 실행 시간도 4.3초에서 0.27초로 줄었습니다.

**성능 더 개선하기**

여기서 더 성능을 개선하려면 검색과 정렬에 적합한 새로운 복합 인덱스를 생성하는 방법을 사용할 수 있습니다.

```shell
db.students.createIndex({"class_id" : 1, "student_id" : 1})
```

```shell
db.students.find({"student_id" : { "$gt" : 500000}, "class_id" : 54}).sort({"student_id" : 1}).explain("executionStats")
```

```javascript
executionStats =  {
    nReturned: 9903,
    executionTimeMillis: 37,
    totalKeysExamined: 9903,
}
```

`nReturned`와 `totalKeysExamined`이 일치하고 hint 함수를 사용할 필요가 없어졌습니다.

### 필터링이 아닌 다른 필드를 기준으로 정렬하는 복합 인덱스

````shell
db.students.find({"student_id" : { "$gt" : 500000}, "class_id" : 54}).sort({"final_grade" : 1})
````

필터링 대상인 `student_id`와 `class_id`이 아닌 다른 필드인 `final_grade`를 기준으로 정렬하는 쿼리를 예시로 들겠습니다.

````shell
db.students.find({"student_id" : { "$gt" : 500000}, "class_id" : 54}).sort({"final_grade" : 1}).explain("executionStats")
````

```javascript
executionStats =  {
  executionStages : {
    nReturned: 9903,
    executionTimeMillis: 136,
    totalDocsExamined: 9903,
     
    stage : "SORT",
    sortPattern : {"final_grade" : 1},
  }
}
```

SORT 단계가 포함되며 `인 메모리 정렬`을 수행하는 것을 확인할 수 있습니다.

인덱스를 더 잘 설계하면 `인 메모리 정렬`을 피할 수 있습니다. 이를 통해 데이터셋 크기와 시스템 부하와 관련해 보다 쉽게 확장할 수 있습니다.

대신 `인 메모리 정렬`을 피할려면 반환하는 도큐먼트 개수보다 더 많은 키를 검사해야합니다. (트레이드 오프)

**인메모리 정렬 피하기**

인덱스를 사용해 정렬하기 위해서는 복합 인덱스 키 사이에 정렬을 위한 필드가 있어야 합니다.

```shell
db.students.createIndex({"class_id" : 1, "final_grade" : 1, "student_id" : 1})
```

정렬 구성 요소는 동등 필터 바로 뒤, 다중값 필터 바로 앞에 위치해야 합니다.

`student_id`가 중간에 있으면 결국 인메모리 정렬을 해야합니다.

| class_id | student_id | final_grade |
| --- |------------|------------|
| 53 | 444404     | 90         |
| 54 | 500001     | 80         |
| 54 | 500002     | 70         |
| 54 | 500003     | 71         |
| 54 | 500004     | 60         |

**정리**

복합 인덱스를 설계할 때는 사용할 공통 쿼리 패턴의 동등 필터, 다중값 필터, 정렬 구성 요소를 고려해야 합니다.

동등 필터에 대한 키를 맨 앞, 정렬에 사용되는 키는 다중값 필드 앞, 다중값 필터에 대한 키는 맨 뒤에 표시하는 것이 좋습니다.

### 키 방향 선택하기

age는 오름차순 username은 내림차순으로 정렬하고 싶다면 `{"age" : 1, "username" : -1}` 인덱스를 사용하면 됩니다.

참고로 역방향 인덱스는 서로 동등합니다.

`{"age" : 1, "username" : -1}`와 `{"age" : -1, "username" : 1}`는 동일한 인덱스입니다. 따라서 두 인덱스 중 하나만 생성하면 됩니다.

### 커버드 쿼리 (Covered Query)

인덱스에 포함된 필드만을 찾는 중이라면 도큐먼트를 가져올 필요가 없습니다. 인덱스가 쿼리가 요구하는 값을 모두 포함하면 쿼리가 커버드 된다고 합니다.

커버드 쿼리를 사용할 수 있다면 사용하는 것이 성능을 크게 향상 시킬수 있습니다.

```shell
db.students.find(
    {"student_id": { "$gt": 500000}, "class_id": 54},
    {"_id": 0, "student_id": 1, "class_id": 1, "final_grade": 1}
).sort({"final_grade": 1}).explain("executionStats")
```

```javascript
totalDocsExamined: 0
executionStages: {
    stage: 'PROJECTION_COVERED'
}
```

`totalDocsExamined`가 0이라는 것은 도큐먼트를 검사하지 않았음을 의미합니다. 그리고 `PROJECTION_COVERED`는 커버드 쿼리임을 의미합니다.

### 암시적 인덱스

`{ "age" : 1, "username" : 1 }`로 인덱스를 가지면 "age" 필드는 이미 정렬되어 있으므로 `{ "age" : 1 }` 인덱스를 추가할 필요가 없습니다.

즉, 복합인덱스 자체적으로 단일 필드 인덱스를 포함하고 있습니다.  복합 인덱스의 키들의 앞부분은 공짜 인덱스로 볼 수 있습니다.
