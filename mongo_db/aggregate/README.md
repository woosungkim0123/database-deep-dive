# 집계 프레임워크

## 집계 파이프 라인

집계 프레임워크는 파이프라인 개념을 기반으로 합니다. 모든 단계의 입력과 출력은 문서(도큐먼트) 입니다.

![집계 파이프라인](../image/aggregate_pipeline.png)

한 번에 입력 문서 스트림을 하나씩 가져와 각 도큐먼트를 하나씩 처리하고, 출력 도큐먼트 스트림을 하나씩 생성합니다.

![집계 파이프라인 단계](../image/aggregate_pipline_course.png)

<br>

## 파이프 라인 단계

`aggregate` 메서드를 사용하여 집계 파이프라인을 실행합니다.

### 일치 (match)

`$match` 스테이지는 입력 문서를 필터링합니다. 일치하는 문서만 파이프라인으로 전달됩니다.

```shell
db.companies.aggregate([
    { $match: { founded_year: 2004 } }
])
```

### 선출 (project)

`$project` 스테이지는 입력 문서의 필드를 추가, 수정, 삭제합니다.

선출 단계는 작업을 수행하고 문서 모양으로 변경 후 출력합니다.

```shell
db.companies.aggregate([
    { $match: { founded_year: 2004 } },
    { $project: { _id: 0, name: 1, founded_year: 1 } } # name, founded_year 필드만 출력합니다.
])
```

### 제한 (limit)

`$limit` 스테이지는 입력 문서의 개수를 제한합니다. 

선출 단계 후에 제한을 적용할수도 있지만 성능상의 이유로 가능하면 먼저 제한을 적용하는 것이 좋습니다.

```shell
db.companies.aggregate([
    { $match: { founded_year: 2004 } },
    { $limit: 5 }, # 5개의 문서만 출력합니다.
    { $project: { _id: 0, name: 1, founded_year: 1 } }
])
```

> 파이프 라인을 구축할 때 한 단계에서 다른 단계로 전달해야 하는 문서 수를 줄이는 것이 중요합니다.

### 정렬 (sort)

`$sort` 스테이지는 입력 문서를 정렬합니다.

```shell
db.companies.aggregate([
    { $match: { founded_year: 2004 } },
    { $sort: { name: 1 } }, # name 필드를 오름차순으로 정렬합니다.
    { $limit: 5 },
    { $project: { _id: 0, name: 1, founded_year: 1 } }
])
```

### 건너뛰기 (skip)

`$skip` 스테이지는 입력 문서에서 처음 몇 개의 문서를 건너뛰고 나머지 문서를 출력합니다.

```shell
db.companies.aggregate([
    { $match: { founded_year: 2004 } },
    { $sort: { name: 1 } },
    { $skip: 5 }, # 처음 5개의 문서를 건너뛰고 나머지 문서를 출력합니다.
    { $limit: 5 },
    { $project: { _id: 0, name: 1, founded_year: 1 } }
])
```

<br>

## 표현식

집계 파이프라인을 구축할 때 다양한 표현식 클래스를 지원합니다.

**불리언 표현식**

AND, OR, NOT 표현식을 사용할 수 있습니다.

**집합 표현식**

2개 이상의 교집합이나 합집합을 얻을 수 있고 두 집합의 차를 이용해 여러 집합 연산을 수행할 수 있습니다.

### 집합 표현식

### 비교 표현식

### 산술 표현식

### 문자열 표현식

### 배열 표현식

### 가변식 표현식

### 누산기


<br>

## $project

서브 도큐먼트나 중첩된 구조를 가진 데이터에서 특정 필드를 상위 레벨로 이동시키는 과정을 중첩 필드의 승격이라고 합니다.

```shell
db.companies.aggregate([
    { $match: { founded_year: 2004 } },
    { $project: { _id: 0, name: 1, founded_year: 1, funder: "$funding_rounds.investments.financial_org.permalink" } }
])
```

```
{
    "name" : "Facebook",
    "founded_year" : 2004,
    "funder" : [
        [ "greylock", "founders-fund" ],
        [ "accel-partners, " "first-round-capital" ],
        [ "us-venture-partners" ]
    ]
}
```
