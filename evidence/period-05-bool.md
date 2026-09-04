# 5교시 실습 — bool 검색

## (공통) 문제 1 — 제공 코드로 must·filter 확인

```http
GET /products/_search
{
  "size": 10,
  "query": {
    "bool": {
      "must": [{ "match": { "name": "무선" } }],
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

- `hits.total.value`: 74
- 상위 3개 ID·name: P-00025(MobiCore 컴팩트 무선 이어폰), P-00129(Auralis 스마트 무선 이어폰), P-00369(SoundLab 데일리 무선 이어폰)
- 세 filter의 실제 값: category="전자기기", in_stock=true, price는 50,000~200,000 범위 안(예: 59400, 53800, 162800)
- must와 filter의 역할 차이: `must`(match)는 관련도 점수를 계산해 결과 순위에 반영하고, `filter`(term/range)는 점수 계산 없이 조건을 만족하는지 여부만 판단해 결과 집합을 좁힘. 그래서 세 filter 값이 모두 같아도 score는 오직 `name` match(무선) 점수 하나(3.02)로만 결정됨

## (공통) 문제 2 — 조건 제거 실험 직접 구현

문제 1의 요청에서 `in_stock` filter만 제거한 API를 작성하세요. 다른 조건은 바꾸지 마세요.

### API 전체 입력

```http
GET /products/_search
{
  "size": 10,
  "query": {
    "bool": {
      "must": [{ "match": { "name": "무선" } }],
      "filter": [
        { "term": { "category": "전자기기" } },
        { "range": { "price": { "gte": 50000, "lte": 200000 } } }
      ]
    }
  }
}
```

### 비교 결과

- 변경 전 total / 변경 후 total: 74 / 83
- 새로 포함된 문서 ID·in_stock: P-00457(in_stock=false), P-00521(in_stock=false) 등 9건 추가. 모두 in_stock=false
- 변화가 없다면 데이터 근거: 해당 없음(변화 있음)
- 제거한 조건의 역할: `in_stock=true` filter는 품절 상품을 결과에서 제외하는 역할이었음. 제거하니 무선+전자기기+가격 조건은 만족하지만 품절인 상품 9건이 결과에 새로 포함됨

## (공통) 문제 3 — should 조건 직접 구현

category가 `전자기기`인 문서 중 `name`에 `무선`이 있거나 `in_stock=true`인 조건을 최소 하나 만족하도록 bool API를 작성하세요. `minimum_should_match`를 명시하세요.

### API 전체 입력

```http
GET /products/_search
{
  "size": 10,
  "query": {
    "bool": {
      "filter": [{ "term": { "category": "전자기기" } }],
      "should": [
        { "match": { "name": "무선" } },
        { "term": { "in_stock": true } }
      ],
      "minimum_should_match": 1
    }
  }
}
```

### 결과 입력

- `hits.total.value`: 1097
- 무선이지만 품절인 문서 존재 여부: 있음(32건, 확인 예: P-00457). name에 "무선"이 매치되면 in_stock 조건과 무관하게 should 하나를 만족해 결과에 포함됨
- 무선이 아니지만 재고가 있는 문서 존재 여부: 있음(848건, 확인 예: P-00009 "NeoTech 데일리 기계식 키보드", score 0.0). in_stock=true만 만족해도 should 조건 충족
- should 조건 판정: `minimum_should_match:1`이라 두 조건 중 하나만 만족해도 포함되므로, 결과 집합이 "무선 AND 재고" 교집합(둘 다 만족)보다 훨씬 넓은 합집합 형태로 나타남(1097건, 문제 1의 74건보다 훨씬 큼)

## (개인) 문제 4 — 자기 bool 검색

자기 사용자 질문 하나를 검색 의도와 정확 조건으로 분해해 bool 요청을 구현하세요.

### 역할·검증 기준

- must 0~1개, filter 2개 이상을 사용합니다.
- 각 field와 query 선택 이유를 mapping type으로 설명합니다.
- 반환 문서 3개 이상을 실제 값으로 검증합니다.

### API와 결과 입력

```http
GET /kpop-songs/_search
{
  "size": 10,
  "_source": ["song_id", "title", "genre", "mood_keywords", "popularity_score"],
  "query": {
    "bool": {
      "must": [{ "match": { "mood_keywords": "신나는" } }],
      "filter": [
        { "term": { "genre": "Dance" } },
        { "range": { "popularity_score": { "gte": 70 } } }
      ]
    }
  }
}
```

- 사용자 질문: "인기 있는 댄스 장르 중에서 신나는 분위기 노래 추천해줘"
- must와 이유: `match: mood_keywords "신나는"`. text field라 검색 의도를 형태소로 분석해 관련도 점수로 순위를 매겨야 하는 부분이라 must로 사용
- filter 2개와 이유: `term: genre="Dance"`(keyword, 정확히 Dance 장르인지), `range: popularity_score>=70`(정확한 인기도 기준선 통과 여부). 둘 다 점수 없이 예/아니오만 필요해 filter로 처리
- 실제 검증 결과: `hits.total.value` 26건. 상위 확인: SONG-00853(genre:[Dance], mood:[신나는], popularity:84), SONG-00002(genre:[Dance], mood:[청량한,신나는], popularity:85), SONG-00031(genre:[Dance], mood:[신나는,청량한], popularity:70) — 3개 모두 genre에 Dance 포함, mood_keywords에 신나는 포함, popularity_score 70 이상 확인됨

## (개인) 문제 5 — 조건 역할 검증

개인 문제 4에서 filter 하나를 제거하고 전후 결과를 비교하세요. 추가로 원래 조건에서 제외되어야 하는 문서 1개를 독립 요청으로 확인하세요.

### 역할·검증 기준

- 한 번에 filter 하나만 제거합니다.
- 새로 포함된 문서의 실제 값을 확인합니다.
- 제외 문서는 원래 bool 결과에 포함되지 않아야 합니다.

### API와 결과 입력

```http
GET /kpop-songs/_search
{
  "size": 10,
  "_source": ["song_id", "title", "genre", "mood_keywords", "popularity_score"],
  "query": {
    "bool": {
      "must": [{ "match": { "mood_keywords": "신나는" } }],
      "filter": [{ "range": { "popularity_score": { "gte": 70 } } }]
    }
  }
}
```

- 제거한 filter: `term: genre="Dance"`
- 전/후 total: 26 / 83
- 새로 포함된 ID와 값: SONG-00059(genre:[Hip-hop], popularity:89), SONG-00242(genre:[R&B], popularity:75), SONG-00325(genre:[Ballad], popularity:85) 등 Dance가 아닌 genre 문서 57건이 새로 포함됨
- 제외 확인 ID와 근거: 독립 요청(`genre=Ballad` + `mood_keywords=신나는` + `popularity_score>=70`)으로 확인한 SONG-00325(Tidal, genre:[Ballad], popularity:85)는 원래 문제 4의 bool 결과(genre=Dance 필수)에는 포함되지 않아야 하는데, 실제로 문제 4의 26건 목록에 없음을 확인 → filter가 정확히 조건대로 문서를 배제하고 있음이 검증됨
