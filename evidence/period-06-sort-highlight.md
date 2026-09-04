# 6교시 실습 — 정렬·highlight

## (공통) 문제 1 — 제공 코드로 1·2차 정렬 확인

```http
GET /products/_search
{
  "size": 10,
  "_source": ["product_id", "name", "price", "rating", "in_stock"],
  "query": { "match": { "name": "무선" } },
  "sort": [
    { "rating": "desc" },
    { "price": "asc" }
  ]
}
```

### 결과 입력

- 상위 5개 ID / rating / price: P-03842(5.0/13900), P-08761(5.0/107200), P-07634(5.0/132300), P-05962(5.0/138300), P-06457(5.0/184900)
- 1차 정렬이 올바른가: 예. rating이 5.0으로 동일한 문서들이 먼저 오고, 이후 4.9대 문서(P-05738 등)로 내려감
- rating 동률에서 2차 정렬이 적용된 사례: rating=5.0인 문서 7개(P-03842~P-01409)가 price 오름차순(13900 → 107200 → 132300 → 138300 → 184900 → 322100 → 350900)으로 정확히 정렬됨
- 동률이 없다면 2차 정렬을 확인할 수 있는 방법: 해당 없음(이번 데이터에 동률 다수 존재). 동률이 없다면 sort 배열의 2차 field만 남기고 1차 field를 제거해 단독 정렬 결과와 비교하거나, 임의로 동일 rating 값을 가진 fixture 문서를 추가해 확인해야 함

## (공통) 문제 2 — 정렬 우선순위 교환

문제 1과 같은 검색 결과를 가격이 낮은 순서로 먼저 정렬하고, 가격이 같으면 평점이 높은 순서로 정렬하세요.

### API 전체 입력

```http
GET /products/_search
{
  "size": 10,
  "_source": ["product_id", "name", "price", "rating", "in_stock"],
  "query": { "match": { "name": "무선" } },
  "sort": [
    { "price": "asc" },
    { "rating": "desc" }
  ]
}
```

### 비교 결과

- 변경 후 상위 5개 ID / price / rating: P-01490(10900/2.9), P-05738(12900/4.9), P-05218(13300/4.9), P-08586(13700/2.3), P-03842(13900/5.0)
- 순서가 달라진 문서: 문제 1의 상위 5개(P-03842, P-08761, P-07634, P-05962, P-06457)와 완전히 다름. 문제 1의 1위였던 P-03842는 문제 2에서 5위로 밀림 — 정렬 기준이 rating 우선에서 price 우선으로 바뀌었기 때문
- 검색 hit 집합도 달라졌는가: 아니오. `hits.total.value`는 두 요청 모두 505로 동일. `query`는 그대로이고 `sort`만 바뀌어 정렬 순서만 변하고 매치되는 문서 집합 자체는 그대로임

## (공통) 문제 3 — highlight와 표시 field 구현

`name`, `description`에서 `무선 이어폰`을 검색하되 `name`에 3배 boost를 적용하세요. 최대 5건을 반환하고 결과 카드용 field만 `_source`에 포함하며 `name`, `description`에 highlight를 적용하세요.

### API 전체 입력

```http
GET /products/_search
{
  "size": 5,
  "_source": ["product_id", "name", "price", "in_stock"],
  "query": {
    "multi_match": {
      "query": "무선 이어폰",
      "fields": ["name^3", "description"]
    }
  },
  "highlight": { "fields": { "name": {}, "description": {} } }
}
```

### 결과 입력

- `_source` field 목록: `product_id`, `name`, `price`, `in_stock`
- highlight가 생성된 문서 ID와 field: P-00241, P-00305, P-00529, P-00617, P-00777 — 5개 모두 `name` field에서만 highlight 생성됨 (예: P-00241 → "SoundLab 프리미엄 `<em>`무선`</em>` `<em>`이어폰`</em>`")
- `_source`와 highlight의 차이: `_source`는 원본 field 값 그대로(예: "SoundLab 프리미엄 무선 이어폰"), highlight는 검색어와 실제로 매치된 조각에 `<em>` 태그를 씌운 조각만 별도 응답 구조(`highlight.name`)에 반환됨. 원본을 직접 수정하지 않고 화면 표시용으로 별도 제공되는 값
- highlight가 없는 hit가 있다면 이유 추정: 5개 상위 문서 모두 `description` highlight는 생성되지 않음. `description`이 "~에 잘 어울리는 전자기기 상품입니다" 같은 공통 문구라 "무선"·"이어폰" 토큰이 실제로 들어있지 않기 때문. 상위권은 `name^3` boost로 name 매치 점수만으로 이미 순위가 결정돼 description에서는 매치가 발생하지 않음

## (개인) 문제 4 — 자기 결과 정렬·카드 설계

자기 서비스에서 중요한 1차·2차 정렬 기준과 결과 카드 field 3~5개를 선택해 Search API를 구현하세요.

### 역할·검증 기준

- 정렬 가능한 mapping type을 사용합니다.
- 1차·2차 정렬의 업무적 이유를 설명합니다.
- 실제 상위 5개 값으로 순서를 검증합니다.

### API와 결과 입력

```http
GET /kpop-songs/_search
{
  "size": 5,
  "_source": ["song_id", "title", "artist", "genre", "popularity_score"],
  "query": { "match_all": {} },
  "sort": [
    { "popularity_score": "desc" },
    { "bpm": "asc" }
  ]
}
```

- 정렬 field·방향·이유: 1차 `popularity_score desc`(인기 높은 곡을 먼저 추천), 2차 `bpm asc`(popularity 동률일 때 템포가 느린 곡을 우선 — 다양한 상황에서 무난하게 들을 수 있는 곡을 앞에 두려는 서비스 판단)
- 카드 field와 이유: `song_id`(식별자), `title`(제목), `artist`(가수), `genre`(장르 파악), `popularity_score`(정렬 근거 확인용)
- 상위 5개 정렬 검증: popularity_score=100인 문서 5개(SONG-00912, SONG-00687, SONG-00106, SONG-00580, SONG-00131)가 모두 최상위에 오고, bpm이 76 → 80 → 81 → 96 → 98 순으로 오름차순 정렬됨 — 1·2차 정렬 모두 의도대로 동작

## (개인) 문제 5 — 자기 highlight 또는 표시 최적화

자기 text 검색에 highlight를 적용하세요. text 검색이 없는 프로젝트라면 `_source` 최소화 전후를 비교하세요.

### 역할·검증 기준

- 검색 field와 highlight field의 관계가 타당해야 합니다.
- 원본 데이터와 강조 조각을 구분합니다.
- 사용자 판단에 실제로 도움이 되는지 평가합니다.

### API와 결과 입력

```http
GET /kpop-songs/_search
{
  "size": 5,
  "_source": ["song_id", "title", "theme_keywords"],
  "query": { "match": { "theme_keywords": "우주" } },
  "highlight": { "fields": { "theme_keywords": {} } }
}
```

- 선택한 방식과 이유: `theme_keywords`(text) 검색어 "우주"에 highlight 적용. 검색 field와 highlight field를 동일하게 둬 사용자가 왜 이 곡이 나왔는지 바로 확인 가능하게 함
- 실제 결과: `hits.total.value` 160건. SONG-00065(theme_keywords: [청춘, `<em>`우주`</em>`]), SONG-00069(theme_keywords: [`<em>`우주`</em>`, 사랑]), SONG-00100(theme_keywords: [`<em>`우주`</em>`, 사랑]) 등 상위 5개 모두 "우주" 부분에 강조 태그 생성됨
- 사용자에게 유용한가: 예. theme_keywords가 배열 형태라 어떤 keyword가 실제로 매치됐는지 원본만 봐서는 구분이 안 되는데, highlight가 정확히 "우주" 항목만 짚어줘서 추천 이유를 바로 알 수 있음
- 개선할 점: 지금은 정확히 "우주"만 검색했지만, 사용자가 "우주 느낌"처럼 복합 표현을 입력하면 여러 field(mood_keywords, situation_keywords)에 흩어진 매치를 한 번에 보여주는 multi_match + 여러 field highlight로 확장하는 게 더 유용할 것
