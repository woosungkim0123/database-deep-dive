# CUD (Create, Update, Delete)

## 문서(Document) 삽입

### InsertOne

`_id`키가 추가되고 도큐먼트가 몽고DB에 저장됩니다.

```shell
db.users.insertOne({ "name": "John", "age": 20 });
```

**결과**

```shell
{
  acknowledged: true, # 성공적으로 수행되었음을 나타내는 표시
  insertedId: ObjectId('65e1aa28e155ec69a55dc759')
}
```

### InsertMany

여러 도큐먼트를 컬렉션에 삽입할 수 있습니다. 

대량 삽입(batch insert)을 하므로 효율적 입니다.

```shell
db.users.insertMany([
  { "name": "John", "age": 20 },
  { "name": "Ann", "age": 21 },
  { "name": "David", "age": 22 }
])
```

한 문서의 최대 데이터 양은 16MB 이고, 한번에 처리할 수 있는 최대 데이터 양은 100,000개 입니다. (`maxWriteBatchSize`의 기본값이 100,000개)

만약 20만개의 문서를 요청하면 여러 배치로 나누어 처리하게 됩니다.

**대량 삽입 중 오류 발생하는 경우** 정렬 연산을 선택했는지 안했는지에 따라 발생하는 상황이 달라집니다.

insertMany의 두번째 매개변수인 옵션 도큐먼트에 `ordered` 키에 true(기본값)를 지정하면 제공된 순서대로 삽입되고 false면 성능 개선을 위해 삽입을 재배열 할 수도 있습니다.

**정렬이 된 경우** 오류가 발생하면 그 시점에서 삽입을 중단하고 오류를 반환합니다. (기본적으로 이미 삽입된 데이터는 롤백되지 않음)



```shell
# 세번째 도큐먼트 id를 중복시켜 오류를 발생하도록 함 (ordered: true) 
db.users.insertMany([
  { "_id": 0, "name": "John", "age": 20 },
  { "_id": 1, "name": "Ann", "age": 21 },
  { "_id": 1, "name": "David", "age": 22 }, 
  { "_id": 2, "name": "Mike", "age": 23 }
])
```

**결과**

```shell
# db.users.find()
[ { _id: 0, name: 'John', age: 20 }, { _id: 1, name: 'Ann', age: 21 } ]
```

**정렬이 되지 않은 경우** 한 문서의 삽입이 실패해도 나머지 문서의 삽입을 계속 시도합니다.

```shell
# 세번째 도큐먼트 id가 중복되어 오류 발생 (ordered: false)
db.users.insertMany([
  { "_id": 0, "name": "John", "age": 20 },
  { "_id": 1, "name": "Ann", "age": 21 },
  { "_id": 1, "name": "David", "age": 22 }, 
  { "_id": 2, "name": "Mike", "age": 23 }
], { ordered: false })
```

**결과**

```shell
# db.users.find()
[
  { _id: 0, name: 'John', age: 20 },
  { _id: 1, name: 'Ann', age: 21 },
  { _id: 2, name: 'Mike', age: 23 }
]
```

> 몽고DB는 insertMany 외에 다양한 대량 쓰기(Bulk Write) API를 지원합니다.

### 삽입 유효성 검사

MongoDB는 삽입된 데이터에 `_id` 필드가 존재하지 않으면 새로 추가하고 도큐먼트가 16MB보다 작은지 확인하는 최소한의 검사를 수행합니다.

<br>

## 도큐먼트 삭제

데이터를 한번 제거하면 백업된 데이터를 복원하는 방법 외에는 복구 방법이 없습니다.

### deleteOne()

필터와 일치하는 첫번째 도큐먼트를 삭제합니다.

```shell
[
  { _id: 0, name: 'John', age: 20 },
  { _id: 1, name: 'Ann', age: 20 },
  { _id: 2, name: 'David', age: 21 },
  { _id: 3, name: 'Mike', age: 20 }
]
```

```shell
db.users.deleteOne({name: "John"})
```

**결과**

```shell
# db.users.find()
[
  { _id: 1, name: 'Ann', age: 20 },
  { _id: 2, name: 'David', age: 21 },
  { _id: 3, name: 'Mike', age: 20 }
]
```

### deleteMany()

필터와 일치하는 모든 도큐먼트를 삭제합니다.

```shell 
db.users.find()
[
  { _id: 0, name: 'John', age: 20 },
  { _id: 1, name: 'Ann', age: 20 },
  { _id: 2, name: 'David', age: 21 },
  { _id: 3, name: 'Mike', age: 20 }
]

db.users.deleteMany({age: 20})
# 결과
[ { _id: 2, name: 'David', age: 21 } ]
```

### drop

전체 컬렉션을 삭제하려면 drop을 사용하는 것이 좋습니다.

```shell
db.users.drop()
```

<br>

## 도큐먼트 갱신

기본적으로 갱신은 원자적으로 이루어지며 요청 두개가 동시에 발생하면 서버에 먼저 도착한 요청이 적용된 후 다음 요청이 적용됩니다.

### replaceOne()

도큐먼트를 새로운 것으로 완전히 치환합니다.

```shell
# age와 gender를 attribute라는 서브 도큐먼트로 변경
db.users.find()
[ { _id: 0, name: 'John', age: 20, gender: 'male' } ]

var john = db.users.findOne({"name" : "John"})
john.attribute =  { "age": john.age, "gender" : john.gender }
john.username = john.name

delete john.age
delete john.gender
delete john.name

db.users.replaceOne({"name" : "John"}, john)
# 결과
[
  { 
    _id: 0, 
    attribute: { age: 20, gender: 'male' }, 
    username: 'John' 
  }
]
```

**조심할 만한 예제**

```shell
db.users.find()
[
  { _id: 0, name: 'John', age: 20 },
  { _id: 1, name: 'John', age: 21 },
  { _id: 2, name: 'John', age: 22 },
  { _id: 3, name: 'John', age: 23 }
]

john = db.users.findOne({"name" : "John", "age" : 21 })
john.age = 30
db.users.replaceOne({"name" : "John"}, john)

# 에러
After applying the update, the (immutable) field '_id' was found to have been altered to _id: 1

[
  { _id: 1, name: 'John', age: 30 },
  { _id: 1, name: 'John', age: 21 }, # _id 중복 => 에러
  { _id: 2, name: 'John', age: 22 },
  { _id: 3, name: 'John', age: 23 }
]
```

이런 상황을 피하기 위해서는 고유한 도큐먼트를 갱신 대상으로 지정하는 것이 좋습니다.

```shell
db.users.replaceOne({"_id": 1}, john)
# 결과
[
  { _id: 0, name: 'John', age: 20 },
  { _id: 1, name: 'John', age: 30 },
  { _id: 2, name: 'John', age: 22 },
  { _id: 3, name: 'John', age: 23 }
]
```

### updateOne()

도큐먼트의 특정 부분만 갱신하는 경우가 많은데 이때 갱신 연산자를 사용합니다.

연산자를 사용할 때 `_id` 값은 변경할 수 없습니다.(변경하려면 도큐먼트 전체 치환)

그 외 다른 키 값은 모두(고유하게 인덱싱된 키를 포함해) 변경할 수 있습니다.

**$inc**

이미 존재하는 키의 값을 변경하거나 새 키를 생성하는 데 사용됩니다.

int, long, double, decimal 타입 값에만 사용할 수 있습니다.

```shell
db.users.findOne({"name" :"John"})
{ _id: 0, name: 'John', age: 20 }

db.users.updateOne({"name" : "John"}, {$inc: {age: 3}})
# 결과
{ _id: 0, name: 'John', age: 23 }
```

**$set**

필드 값을 설정합니다. 필드가 존재하지 않으면 새로 생성합니다.

스키마를 갱신하거나 사용자 정의 키를 추가할 때 편리합니다.

```shell
db.users.findOne({"name" :"John"})
{ _id: 0, name: 'John', age: 23 }

db.users.updateOne({"_id" : 0}, {"$set" : { "favoriteColor" : "blue" }})
# 결과
{ _id: 0, name: 'John', age: 23, favoriteColor: 'blue' }

db.users.updateOne({"_id" : 0}, {"$set" : { "favoriteColor" : "red" }})
# 결과
{ _id: 0, name: 'John', age: 23, favoriteColor: 'red' }

db.users.updateOne({"_id" : 0}, {"$set" : { "favoriteColor" : ["blue", "red"] }}) # 데이터형 변경도 가능
# 결과
{ _id: 0, name: 'John', age: 23, favoriteColor: [ 'blue', 'red' ]}
```

```shell
# 내장 도큐먼트 내부 데이터 변경
db.users.findOne({"name" :"John"})
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  attribute: { gender: 'male', favorite: 'bear' }
}

db.users.updateOne({"attribute.gender" : "male"}, {"$set" : { "attribute.favorite" : "panda" }})
# 결과
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  attribute: { gender: 'male', favorite: 'panda' }
}
```

키를 추가, 변경, 삭제할 때는 항상 `$` 제한자를 사용해야 합니다. 

값을 갱신하다가 키 값을 다른 값으로 바꾸는 경우가 너무 많이 발생하다 보니 `$` 제한자를 사용하지 않으면 오류가 나도록 변경되었습니다.

```shell
db.users.updateOne({"attribute.gender" : "male"}, { "attribute.gender" : "panda" })
# 오류
MongoInvalidArgumentError: Update document requires atomic operators
```

**$unset**

키와 값을 제거합니다.

```shell
db.users.findOne({"name" :"John"})
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  attribute: { gender: 'male', favorite: 'panda' }
}

db.users.updateOne({"name" : "John"}, {"$unset" : { "attribute.favorite" : "" }}) 
# 결과
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  attribute: { gender: 'male' }
}
```

**배열 연산자** 

배열 연산자는 배열을 값으로 갖는 키에만 사용할 수 있습니다.

**$push**

배열이 이미 존재하면 배열 끝에 요소를 추가하고 배열이 존재하지 않으면 새 배열을 생성합니다.

```shell
db.users.findOne({"name" :"John"})
{ _id: ObjectId('65e1bef9e155ec69a55dc75d'), name: 'John', age: 20 }

db.users.updateOne({"name" : "John"}, {"$push" : { "likes" : { "id" : 1, "type" : "board" }}})
# 결과
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [ { id: 1, type: 'board' } ]
}
```

**$each** 

`$push`에 `$each`를 사용하면 여러 요소를 배열에 추가할 수 있습니다.

```shell
db.users.findOne({"name" :"John"})
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [ { id: 1, type: 'board' } ]
}

db.users.updateOne({"name" : "John"}, {"$push" : { "likes" : { "$each" : [{ "id" : 2, "type" : "board" }, { "id" : 3, "type" : "board" }] }}})
# 결과
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 1, type: 'board' },
    { id: 2, type: 'board' },
    { id: 3, type: 'board' }
  ]
}
```

**slice**

배열을 특정 길이로 늘이려면 `$slice`와 `$push`를 결합해서 사용하면 됩니다.

음수를 넣게 되면 배열의 끄텡서부터 시작하여 해당 숫자만큼의 항목을 유지하라는 의미입니다.

양수를 넣게 되면 배열의 앞에서부터 시작하여 해당 숫자만큼의 항목을 유지하라는 의미입니다. 

`$slice`에 3을 설정했지만 배열에 항목이 2개밖에 없다면, $slice 연산은 배열의 현재 상태를 변경하지 않습니다.

```shell
# 배열이 특정 크기 이상으로 늘어나지 않도록 제한
db.users.findOne({"name" :"John"})
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 1, type: 'board' },
    { id: 2, type: 'board' },
    { id: 3, type: 'board' }
  ]
}

db.users.updateOne({"name" : "John"}, {"$push" : { "likes" : { "$each" : [{ "id" : 4, "type" : "board" }], "$slice" : -2 }}}) # 2개로 제한
# 결과
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [ { id: 3, type: 'board' }, { id: 4, type: 'board' } ]
}
```

**sort**

트리밍(배열의 크기를 조정하거나 조건에 맞지 않는 데이터를 제거하는 작업) 하기전 `$sort` 제한자를 `$push`에 사용하면 배열을 정렬할 수 있습니다.

이때 추가되는 값 뿐만 아니라 기존에 있는 값들도 정렬 기준에 따라 재정렬 됩니다. 그리고 이 변경된 순서는 저장되어 다음 조회 시에도 유지됩니다.

정렬 대상에는 숫자 뿐만 아니라 문자열, 날짜 등 다양한 데이터 타입에도 적용될 수 있습니다. (-1은 내림차순, 1은 오름차순을 의미합니다.

```shell
db.users.findOne({"name" :"John"})
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [ { id: 3, type: 'board' }, { id: 4, type: 'board' } ]
}

db.users.updateOne({"name" : "John"}, {"$push" : { "likes" : { "$each" : [{ "id" : 4, "type" : "board" }], "$slice" : 10, "$sort" : { "id" : -1 } }}})
# 결과
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 4, type: 'board' },
    { id: 4, type: 'board' },
    { id: 3, type: 'board' }
  ]
}
```

> `$slice`, `$sort`를 배열상에서 `$push`와 함께 사용하려면 반드시 `$each`와 함께 사용해야 합니다.

**addToSet**

배열을 집합처럼 처리할 수 있습니다. 중복된 값을 추가하지 않습니다.

```shell
db.users.findOne({"name" :"John"})
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 4, type: 'board' },
    { id: 4, type: 'board' },
    { id: 3, type: 'board' }
  ]
}

db.users.updateOne({"name" : "John"}, {"$addToSet" : { "likes" : { "id" : 4, "type" : "board" }}})
# 결과 (변화없음)
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 4, type: 'board' },
    { id: 4, type: 'board' },
    { id: 3, type: 'board' }
  ]
}

db.users.updateOne({"name" : "John"}, {"$addToSet" : { "likes" : { "id" : 5, "type" : "board" }}})
# 결과
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 4, type: 'board' },
    { id: 4, type: 'board' },
    { id: 3, type: 'board' },
    { id: 5, type: 'board' }
  ]
}
```

고유한 값을 여러개 추가하고 싶으면 `$addToSet`에 `$each`를 사용하면 됩니다.

```shell
db.users.findOne({"name" :"John"})
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 4, type: 'board' },
    { id: 4, type: 'board' },
    { id: 3, type: 'board' },
    { id: 5, type: 'board' }
  ]
}

db.users.updateOne({"name" : "John"}, {"$addToSet" : { "likes" : {"$each" : [{ "id" : 5, "type" : "board" }, { "id" : 6, "type" : "board" }, { "id" : 7, "type" : "board"}, ] }}})
# 결과 - 중복된 값은 추가 안되고 중복되지 않은 값만 추가
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 4, type: 'board' },
    { id: 4, type: 'board' },
    { id: 3, type: 'board' },
    { id: 5, type: 'board' },
    { id: 6, type: 'board' },
    { id: 7, type: 'board' }
  ]
}
```

**$pop**

배열의 양쪽 끝에서 요소를 제거할 수 있습니다. 1은 배열의 끝에서 요소를 제거하고 -1은 배열의 앞에서 요소를 제거합니다.

```shell
db.users.findOne({"name" :"John"})
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 4, type: 'board' },
    { id: 4, type: 'board' },
    { id: 3, type: 'board' },
    { id: 5, type: 'board' },
    { id: 6, type: 'board' },
    { id: 7, type: 'board' }
  ]
}

db.users.updateOne({"name" : "John"}, {"$pop" : { "likes" : 1 }})
# 결과
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 4, type: 'board' },
    { id: 4, type: 'board' },
    { id: 3, type: 'board' },
    { id: 5, type: 'board' },
    { id: 6, type: 'board' }
  ]
}

db.users.updateOne({"name" : "John"}, {"$pop" : { "likes" : -1 }})
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 4, type: 'board' },
    { id: 3, type: 'board' },
    { id: 5, type: 'board' },
    { id: 6, type: 'board' }
  ]
}
```

**$pull**

배열에서 조건에 일치하는 요소를 모두 제거합니다.

```shell
db.users.findOne({"name" :"John"})
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 4, type: 'board' },
    { id: 3, type: 'board' },
    { id: 5, type: 'board' },
    { id: 6, type: 'board' }
  ]
}

db.users.updateOne({"name" : "John"}, {"$pull" : { "likes" : { "type" : "board" }}})
# 결과
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: []
}
```

**배열 위치 기반 변경**

쿼리 도큐먼트와 일치하는 배열 요소 및 요소의 위치를 알아내서 갱신하는 위치 연산자 `$`를 사용할 수 있습니다.

```shell
db.users.findOne({"name" :"John"})
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 5, type: 'board' },
    { id: 6, type: 'board' },
    { id: 7, type: 'board' }
  ]
}

db.users.updateOne({"name" : "John", "likes.id" : 6}, {"$set" : { "likes.$.type" : "notice" }})
# 결과
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 5, type: 'board' },
    { id: 6, type: 'notice' },
    { id: 7, type: 'board' }
  ]
}
```

**arrayFilters**

`arrayFilters`를 사용하면 특정 조건에 맞는 배열 요소를 갱신할 수 있습니다.

```shell
db.users.findOne({"name" :"John"})
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 5, type: 'board' },
    { id: 6, type: 'notice' },
    { id: 7, type: 'board' }
  ]
}

db.users.updateOne({"name" : "John"}, {"$set" : { "likes.$[element].type" : "notice" }}, { "arrayFilters" : [ { "element.id" : 5 } ] })
# 결과
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: [
    { id: 5, type: 'notice' },
    { id: 6, type: 'notice' },
    { id: 7, type: 'board' }
  ]
}
```

**갱신 입력**

항목이 있으면 업데이트하고 없으면 새로 만드는 코드를 짜게 되면 여러 프로세스에서 실행하면 경쟁 상태(race condition)가 발생할 수 있습니다.

갱신 입력을 사용하면 이런 경쟁 상태 문제를 해결하는데 어느정도 도움이 될 수 있습니다.

```shell
db.users.findOne({"name" :"John"})
{ _id: ObjectId('65e1bef9e155ec69a55dc75d'), name: 'John', age: 20 }

db.users.updateOne({"name" : "John"}, {"$inc" : { "likes" : 1 }}, { "upsert" : true })
# 결과
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: 1
}

db.users.updateOne({"name" : "John"}, {"$inc" : { "likes" : 1 }}, { "upsert" : true })
# 결과
{
  _id: ObjectId('65e1bef9e155ec69a55dc75d'),
  name: 'John',
  age: 20,
  likes: 2
}

db.users.drop()
db.users.updateOne({"like" : 25}, {"$inc" : { "like" : 1 }}, { "upsert" : true })
# 결과 - like 키가 없으므로 새로 생성하고 1을 더함, 만약 갱신 입력을 지정하지 않으면 조건이 일치하는 도큐먼트가 없으므로 아무것도 하지 않음
{
  _id: ObjectId('65e1e6f4a0c48f462ada421e'),
  like: 26
}
```

**$setOnInsert**

도큐먼트가 삽입될 때만 도큐먼트에 값을 설정합니다. 만약 값이 이미 존재하는 경우 변경되지 않습니다.

```shell
db.users.drop()
db.users.updateOne({}, {"$setOnInsert" : { "createdAt" : new Date() }}, { "upsert" : true })
{
  _id: ObjectId('65e1e8e6a0c48f462ada4294'),
  createdAt: ISODate('2024-03-01T14:40:38.213Z')
}

db.users.updateOne({}, {"$setOnInsert" : { "createdAt" : new Date() }}, { "upsert" : true })
{
  _id: ObjectId('65e1e8e6a0c48f462ada4294'),
  createdAt: ISODate('2024-03-01T14:40:38.213Z')
}
```

### updateMany()

조건에 맞는 도큐먼트를 모두 수정합니다.

스키마를 변경한다거나 특정 사용자에게 새로운 정볼르 추가할 때 사용하기 좋습니다.

e.g. 생일인 사용자에게 쿠폰 추가

```shell
db.users.find()
[
  { _id: ObjectId('65e266654842563d1a223a85'), name: 'John', age: 20 },
  { _id: ObjectId('65e266654842563d1a223a86'), name: 'Ann', age: 21 },
  { _id: ObjectId('65e266654842563d1a223a87'), name: 'David', age: 22 }
]

db.users.updateMany({ "age" : { "$gt" : 20 } }, { "$set" : { "coupon" : "20% off" } })
# 결과
[
  { _id: ObjectId('65e266654842563d1a223a85'), name: 'John', age: 20 },
  {
    _id: ObjectId('65e266654842563d1a223a86'),
    name: 'Ann',
    age: 21,
    coupon: '20% off'
  },
  {
    _id: ObjectId('65e266654842563d1a223a87'),
    name: 'David',
    age: 22,
    coupon: '20% off'
  }
]
```

### 갱신한 도큐먼트 반환

`findOneAndUpdate`, `findOneAndReplace`, `findOneAndDelete`를 사용하면 갱신한 도큐먼트를 반환할 수 있습니다.

`updateOne`과 주요 차이점은 해당 연산들은 트랜잭션의 원자성(Atomicity) 원칙을 따르며, 시작부터 끝까지 (즉, 찾고 업데이트하는 과정 전체를 포함하여) '분할 불가능한' 하나의 단위로 처리됩니다. 

몽고DB 4.2부터는 갱신을 위한 집계 파이프라인을 수용하도록 확장하여 `addFields($set)`, `project($unset)`, `replaceRoot($replaceWith)`로 구성될 수 있습니다.

**경쟁 상태 문제 예시**

`준비 상태`인 프로세스 중 우선순위가 가장 높은 프로세스 하나를 찾아서 `동작중`인 상태로 변경하고 특정 작업을 수행 후 `완료` 상태로 변경하는 알고리즘을 짠다고 가정하겠습니다.

```shell
var cursor = db.process.find({ "status" : "READY" }).sort({ "priority" : -1 }).limit(1)

while ((ps = cursor.next()) != null) {
    var result = db.process.updateOne({ "_id" : ps._id, "status" : "READY" }, { "$set" : { "status" : "RUNNING" } })
    
    if (result.modifiedCount > 0) {
        # 특정 작업 수행
        db.process.updateOne({ "_id" : ps._id, "status" : "RUNNING" }, { "$set" : { "status" : "COMPLETE" } })
        break;
    }
    cursor = db.process.find({ "status" : "READY" }).sort({ "priority" : -1 }).limit(1)
}
```

위 처럼 알고리즘을 짜게되면 여러 스레드가 동시에 접속했을 때 경쟁 상태가 발생할 수 있습니다.

두개 스레드가 동일한 가장 우선순위가 높은 프로세스를 찾아서 변경하려고 하면 하나는 성공하고 다른 하나는 실패하게 됩니다. 이러다보면 하나의 스레드만 계속 동작할 가능성이 발생합니다. 

이런 상황에 `findOneAndUpdate`를 사용하면 경쟁 상태 문제를 해결할 수 있습니다.

```shell
db.process.insertMany([
{ "name": "ProcessA", "priority": 20, status: "READY" },
{ "name": "ProcessB", "priority": 5, status: "READY" },
{ "name": "ProcessC", "priority": 3, status: "READY" }
])
[
  {
    _id: ObjectId('65e272f04842563d1a223a88'),
    name: 'ProcessA',
    priority: 20,
    status: 'READY'
  },
  {
    _id: ObjectId('65e272f04842563d1a223a89'),
    name: 'ProcessB',
    priority: 5,
    status: 'READY'
  },
  {
    _id: ObjectId('65e272f04842563d1a223a8a'),
    name: 'ProcessC',
    priority: 3,
    status: 'READY'
  }
]
# returnNewDocument를 사용하면 갱신된 도큐먼트를 반환합니다. (false면 갱신 전 도큐먼트 반환)
var ps = db.process.findOneAndUpdate({ "status" : "READY" }, 
  { "$set" : { "status" : "RUNNING" } }, 
  { "sort" : { "priority" : -1 }, 
  "returnNewDocument" : true })

# 로직 작업처리~

db.process.updateOne({ "_id" : ps._id}, { "$set" : { "status" : "DONE" } })

# 결과
[
  {
    _id: ObjectId('65e272f04842563d1a223a88'),
    name: 'ProcessA',
    priority: 20,
    status: 'DONE'
  },
  {
    _id: ObjectId('65e272f04842563d1a223a89'),
    name: 'ProcessB',
    priority: 5,
    status: 'READY'
  },
  {
    _id: ObjectId('65e272f04842563d1a223a8a'),
    name: 'ProcessC',
    priority: 3,
    status: 'READY'
  }
]
```
