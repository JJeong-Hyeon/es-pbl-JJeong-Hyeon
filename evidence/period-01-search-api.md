# 1교시 실습 — Search API 기본

## (공통) 문제 1 — 제공 코드 실행·응답 읽기

다음 요청을 실행하세요.

```http
GET /products/_search
{
  "size": 5,
  "query": { "match_all": {} }
}
```

### 결과 입력

- HTTP 성공 여부: 성공 (200, `timed_out: false`)
- `hits.total.value`: 10000
- `hits.hits`에 반환된 문서 수: 5
- 첫 번째 문서의 `_id`: P-00003
- 첫 번째 문서의 `_source` field 3개: `name`("Morrow 실속형 오버핏 후드"), `category`("패션"), `price`(27700)
- `hits.total.value`와 반환 문서 수가 다를 수 있는 이유: `size`가 5로 제한되어 실제 반환은 5건뿐이지만, `hits.total`은 `size`와 무관하게 query 조건(여기선 match_all이라 전체)에 맞는 문서 총수를 별도로 계산해 보여주기 때문

## (공통) 문제 2 — 반환 개수와 field 직접 구현

`products` index의 전체 문서 중 최대 3건만 반환하고, `_source`에는 `product_id`, `name`, `price`, `in_stock`만 포함하는 Search API를 작성하고 실행하세요.

### API 전체 입력

```http
GET /products/_search
{
  "size": 3,
  "_source": ["product_id", "name", "price", "in_stock"],
  "query": { "match_all": {} }
}
```

### 결과 입력

- 반환 문서 수: 3
- `_source`에 요구하지 않은 field가 포함됐는가: 아니오. `product_id`, `name`, `price`, `in_stock` 4개만 반환됨(description·category·brand 등 제외 확인)
- 검증한 문서 ID: P-00003, P-00004, P-00008

## (공통) 문제 3 — 정렬이 포함된 전체 조회 구현

`products` index의 전체 문서 중 최대 10건을 `price`가 낮은 순서로 반환하세요. `_source`에는 `product_id`, `name`, `price`만 포함하세요.

### API 전체 입력

```http
GET /products/_search
{
  "size": 10,
  "_source": ["product_id", "name", "price"],
  "query": { "match_all": {} },
  "sort": [{ "price": "asc" }]
}
```

### 결과 입력

- 첫 3개 문서의 ID와 price: P-00431(5900), P-06599(5900), P-06479(5900)
- 오름차순 여부: 예. 반환된 10건의 price가 5900 → 5900 → 5900 → 5900 → 6100 → 6100 → 6100 → 6100 → 6200 → 6200 순으로 감소하지 않음
- 두 문서의 price가 같을 때 순서가 고정된다고 말할 수 있는가? 근거: 아니오. price=5900인 문서가 4건(P-00431, P-06599, P-06479, P-08895) 있는데 정렬 기준이 price 하나뿐이라 동점 문서 간 순서는 tie-breaker가 없어 매 실행마다 바뀔 수 있음. 순서를 고정하려면 `_id` 등 고유 field를 sort에 추가해야 함

## (개인) 문제 4 — 자기 index의 첫 Search API

자기 index의 전체 문서 중 최대 5건을 반환하는 Search API를 작성하세요.

### 역할·검증 기준

- 실제 자기 index 이름을 사용합니다.
- `_count`와 `hits.total.value`를 비교합니다.
- `size`와 전체 일치 문서 수를 구분해 설명합니다.

### API와 결과 입력

```http
GET /kpop-songs/_search
{
  "size": 5,
  "query": { "match_all": {} }
}
```

- 자기 index: `kpop-songs`
- `_count`: 1000
- `hits.total.value`: 1000
- 반환 문서 수: 5
- 판정과 근거: `_count`와 `hits.total.value`가 1000으로 동일 → match_all이라 index 전체 문서 수와 query 매치 수가 같음. 다만 실제로 화면에 돌아온 `hits.hits` 배열 길이는 `size:5`로 제한돼 5건뿐이라, "전체 몇 건과 매치되는가(total)"와 "이번 응답에 몇 건이 실려오는가(size)"는 서로 다른 개념임

## (개인) 문제 5 — 결과 카드 field 설계

자기 서비스에서 검색 결과 카드 한 개를 보여 준다고 가정하세요. 사용자가 클릭 여부를 결정하는 데 필요한 field 3~5개만 반환하는 Search API를 작성하세요.

### 역할·검증 기준

- 선택한 field가 자기 mapping과 실제 문서에 존재해야 합니다.
- 식별자, 제목 역할, 판단용 정보가 포함되어야 합니다.
- 불필요한 field를 하나 이상 제외하고 이유를 설명합니다.

### API와 결과 입력

```http
GET /kpop-songs/_search
{
  "size": 3,
  "_source": ["song_id", "title", "artist", "genre", "popularity_score"],
  "query": { "match_all": {} }
}
```

- 포함한 field와 이유: `song_id`(식별자), `title`(제목 역할), `artist`(누가 불렀는지 클릭 판단에 필요), `genre`(어떤 장르인지 취향 판단), `popularity_score`(인기도로 신뢰도 판단)
- 제외한 field와 이유: `bpm`, `release_date`, `mood_keywords`, `theme_keywords`, `situation_keywords`는 카드 미리보기에는 정보가 과해서 제외. 상세 페이지 진입 후 확인할 정보이지 클릭 여부 결정에 필수는 아님
- 실제 반환 문서 ID: SONG-00001(Orbit, LUMINA, [Dance, Electronic], 78), SONG-00002(Neon Wave, TIDE9, [Dance], 85), SONG-00003(Afterglow, 이서윤, [Ballad, R&B], 91)
- 완료 판정: 통과. 요구한 5개 field만 정확히 반환됨
