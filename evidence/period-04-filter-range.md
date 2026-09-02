# 4교시 실습 — 정확 조건과 경계

## (공통) 문제 1 — 제공 코드로 세 filter 확인

```http
GET /products/_search
{
  "size": 10,
  "query": {
    "bool": {
      "filter": [
        { "term": { "category": "전자기기" } },
        { "term": { "in_stock": true } },
        { "range": { "price": { "gte": 50000, "lte": 200000 } } }
      ]
    }
  }
}
```

### 결과 입력

- `hits.total.value`: 380
- 확인한 문서 ID 3개: P-00025, P-00129, P-00185
- 각 문서의 category / in_stock / price: 셋 다 category=전자기기, in_stock=true / P-00025 price=59400, P-00129 price=53800, P-00185 price=161600
- 조건을 위반한 문서가 있는가: 없음 (세 조건 모두 filter라 score 없이 정확히 만족하는 문서만 반환됨)

## (공통) 문제 2 — 경계 포함 범위 직접 구현

`products`에서 category가 `전자기기`이고 가격이 50,000원 이상 200,000원 이하인 상품을 검색하세요. 최대 10건을 반환하고 `product_id`, `name`, `category`, `price`만 표시하세요.

### API 전체 입력

```http
GET /products/_search
{
  "size": 10,
  "_source": ["product_id", "name", "category", "price"],
  "query": {
    "bool": {
      "filter": [
        { "term": { "category": "전자기기" } },
        { "range": { "price": { "gte": 50000, "lte": 200000 } } }
      ]
    }
  }
}
```

### 결과 입력

- `hits.total.value`: 440
- 최소·최대 price: min 50700 / max 199500 (aggs로 확인)
- 50,000 또는 200,000 경계 문서 존재 여부와 ID: 없음. `term`으로 price=50000, price=200000을 각각 확인했으나 두 값 모두 count 0

## (공통) 문제 3 — 경계 제외 범위 직접 구현

문제 2에서 다른 조건은 모두 그대로 유지하고 가격 조건만 50,000원 초과 200,000원 미만으로 바꾸세요. 한 요소만 변경해야 합니다.

### API 전체 입력

```http
GET /products/_search
{
  "size": 10,
  "_source": ["product_id", "name", "category", "price"],
  "query": {
    "bool": {
      "filter": [
        { "term": { "category": "전자기기" } },
        { "range": { "price": { "gt": 50000, "lt": 200000 } } }
      ]
    }
  }
}
```

### 비교 결과

- 문제 2 total / 문제 3 total: 440 / 440 (동일)
- 빠진 경계 문서 ID: 없음
- 경계 문서가 없어 결과가 같다면 확인한 근거: `term` query로 category=전자기기 조건에서 price=50000, price=200000인 문서를 각각 조회했더니 count가 둘 다 0이었음. 즉 실제 최소값(50700)·최대값(199500) 모두 경계값보다 안쪽에 있어 gte/lte와 gt/lt 차이가 결과에 드러나지 않음

## (개인) 문제 4 — 자기 정확 조건 2개

자기 데이터에서 정확 조건으로 사용할 field 2개를 선택해 두 조건을 모두 만족하는 검색을 구현하세요.

### 역할·검증 기준

- keyword·boolean 등 실제 mapping type에 적합해야 합니다.
- 실행 전 포함 예상 문서 1개와 제외 예상 문서 1개를 정합니다.
- 실행 후 `_source`로 판정합니다.

### API와 결과 입력

```http
GET /kpop-songs/_search
{
  "size": 10,
  "_source": ["song_id", "title", "genre", "bpm"],
  "query": {
    "bool": {
      "filter": [
        { "term": { "genre": "Dance" } },
        { "term": { "bpm": 108 } }
      ]
    }
  }
}
```

- field·type·값 2개: `genre`(keyword) = "Dance", `bpm`(integer, 정확값 매치) = 108
- 기대 ID / 제외 ID: 기대 SONG-00001(genre에 Dance 포함, bpm 108 확인됨) / 제외 SONG-00002(genre는 Dance지만 bpm이 다름)
- 실제 결과와 판정: `hits.total.value` 3건(SONG-00001, SONG-00039, SONG-00606) 모두 genre 배열에 "Dance" 포함 + bpm=108 확인됨 → 통과. SONG-00002는 결과에 없음(제외 예상과 일치)

## (개인) 문제 5 — 자기 범위와 경계 실험

자기 데이터의 numeric 또는 date field를 선택해 포함 경계와 제외 경계 요청을 각각 구현하세요.

### 역할·검증 기준

- 실제 데이터의 최소·최대 또는 의미 있는 경계값을 먼저 확인합니다.
- `gte/lte`와 `gt/lt` 외 조건은 동일하게 유지합니다.
- 경계 문서가 없으면 fixture 설계 또는 부재 근거를 기록합니다.

### API와 결과 입력

```http
GET /kpop-songs/_search
{
  "size": 10,
  "_source": ["song_id", "title", "popularity_score"],
  "query": { "range": { "popularity_score": { "gte": 80, "lte": 90 } } }
}
```

```http
GET /kpop-songs/_search
{
  "size": 10,
  "_source": ["song_id", "title", "popularity_score"],
  "query": { "range": { "popularity_score": { "gt": 80, "lt": 90 } } }
}
```

- field / type / 경계값: `popularity_score`(integer) / 전체 분포 0~100 확인 후 경계 80·90 선택 (사전에 `term`으로 값=80 count 9건, 값=90 count 7건 확인)
- 포함 요청 total / 제외 요청 total: 102 / 86
- 달라진 문서 ID: popularity_score가 정확히 80 또는 90인 문서(총 16건, 예: SONG-00002가 아닌 값=80·90 문서들)가 gte/lte에는 포함되고 gt/lt에서는 제외됨
- 경계 판정: 102 - 86 = 16 = 9(값 80) + 7(값 90) 으로 정확히 일치 → gte/lte가 경계값 포함, gt/lt가 경계값 제외한다는 것이 실제 데이터로 검증됨
