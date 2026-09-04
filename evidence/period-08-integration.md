# 8교시 실습 — 통합·개선·제출

## (공통) 문제 1 — 제공 코드로 통합 검색 검증

```http
GET /products/_search
{
  "size": 10,
  "_source": ["product_id", "name", "description", "category", "price", "rating", "in_stock"],
  "query": {
    "bool": {
      "must": [{
        "multi_match": {
          "query": "무선 이어폰",
          "fields": ["name^3", "description"]
        }
      }],
      "filter": [
        { "term": { "category": "전자기기" } },
        { "term": { "in_stock": true } },
        { "range": { "price": { "gte": 50000, "lte": 200000 } } }
      ]
    }
  },
  "sort": [{ "rating": "desc" }, { "price": "asc" }],
  "highlight": { "fields": { "name": {}, "description": {} } }
}
```

### 결과 입력

- `hits.total.value`: 74
- 상위 3개 ID: P-08761(rating 5.0/107200원), P-06457(rating 5.0/184900원), P-09025(rating 4.9/51300원)
- 세 filter 통과 여부: 통과. 상위 10개 모두 category="전자기기", in_stock=true, price가 50,000~200,000 범위 안에 있음
- 1·2차 정렬 통과 여부: 통과. rating이 5.0 → 5.0 → 4.9 → 4.9 → 4.9 → 4.9 → 4.9 → 4.8 → 4.8 → 4.7로 내림차순이며, rating 동률인 5.0 두 건은 price 107200 → 184900으로 오름차순
- highlight 확인 결과: 상위 10개 전부 `name` field에 "무선"·"이어폰" 강조 태그가 생성됨(예: "MobiCore 컴팩트 `<em>`무선`</em>` `<em>`이어폰`</em>`"). `description` highlight는 생성되지 않음(공통 문구라 검색어 미포함)
- 관련/보류/무관 판정: 관련. 상위 3개 모두 name에 "무선 이어폰"이 그대로 포함되고 세 filter·정렬 조건을 모두 만족함

## (공통) 문제 2 — boost 개선 전후 직접 구현

`name`, `description`에서 `무선 이어폰`을 검색하는 boost 없는 요청과 `name^3` 요청을 각각 작성하세요. 다른 조건은 동일하게 유지하세요.

### 개선 전 API

```http
GET /products/_search
{
  "size": 5,
  "_source": ["product_id", "name"],
  "query": {
    "multi_match": {
      "query": "무선 이어폰",
      "fields": ["name", "description"]
    }
  }
}
```

### 개선 후 API

```http
GET /products/_search
{
  "size": 5,
  "_source": ["product_id", "name"],
  "query": {
    "multi_match": {
      "query": "무선 이어폰",
      "fields": ["name^3", "description"]
    }
  }
}
```

### 비교 결과

- 전/후 상위 3개 ID: P-00241, P-00305, P-00529 (전/후 동일)
- 순위가 달라진 문서: 없음. score만 6.76 → 20.28로 커졌을 뿐 상위 5개 순서·구성 모두 동일
- 개선/보류/악화: 보류. 상위권만 보면 차이가 드러나지 않음
- 사용자 의도 근거: 상위권 문서들은 boost 이전에도 이미 name에서 강하게 매치돼 1~5위였기 때문에 이번 데이터로는 boost 효과가 상위 3~5개에서 관측되지 않음. name에는 검색어가 없고 description에만 있는 문서를 상대적으로 밀어내는 방향이라, 하위권까지 비교해야 boost가 실제로 순위를 바꾸는지 확인 가능

## (공통) 문제 3 — 요구사항으로 최종 API 직접 구현

다음 요구사항만 보고 실행 가능한 Search API 전체를 작성하세요.

- index: `products`
- 검색어: `무선 이어폰`
- 검색 field: `name`, `description`; name을 더 중요하게 처리
- category: `전자기기`
- 재고 있는 상품만 포함
- 가격: 50,000원 이상 200,000원 이하
- 평점 높은 순, 가격 낮은 순
- 최대 10건
- 결과 카드 field와 검색어 highlight 포함

### API 전체 입력

```http
GET /products/_search
{
  "size": 10,
  "_source": ["product_id", "name", "price", "category", "rating", "in_stock"],
  "query": {
    "bool": {
      "must": [{
        "multi_match": {
          "query": "무선 이어폰",
          "fields": ["name^3", "description"]
        }
      }],
      "filter": [
        { "term": { "category": "전자기기" } },
        { "term": { "in_stock": true } },
        { "range": { "price": { "gte": 50000, "lte": 200000 } } }
      ]
    }
  },
  "sort": [{ "rating": "desc" }, { "price": "asc" }],
  "highlight": { "fields": { "name": {}, "description": {} } }
}
```

### 검증 결과

- 문제 1과 기능적으로 같은 조건인가: 예
- 다른 부분이 있다면 이유: `_source`에서 `description`을 뺀 것만 차이. 카드에 실제로 표시할 field(`product_id`, `name`, `price`, `category`, `rating`, `in_stock`)만 남기고 본문 설명은 목록 화면에서는 불필요해 제외함. query·filter·sort·highlight 로직은 문제 1과 완전히 동일
- 실제 실행 성공 여부: 성공 (200, 오류 없음)
- 상위 결과 검증: `hits.total.value` 74로 문제 1과 동일, 상위 10개 ID·순서·highlight 결과까지 완전히 일치(P-08761 → P-06457 → P-09025 순)

## (개인) 문제 4 — 자기 검색 한 요소 개선

7교시에서 진단한 개인 검색 문제 하나를 선택해 query, field, boost, filter, sort, 검색어 중 한 요소만 변경하고 다시 실행하세요.

### 역할·검증 기준

- 같은 index·데이터·검색어·size를 유지합니다. 검색어를 바꾸는 실험이라면 나머지 요소를 유지합니다.
- 변경 전후 요청을 모두 보존합니다.
- hit 수가 아니라 사용자 의도와 조건 통과로 개선을 판정합니다.

### API와 결과 입력

```http
GET /kpop-songs/_search
{
  "size": 5,
  "_source": ["song_id", "title", "theme_keywords", "situation_keywords"],
  "query": {
    "multi_match": {
      "query": "우주 드라이브",
      "fields": ["theme_keywords", "situation_keywords"],
      "type": "most_fields"
    }
  }
}
```

- 문제 / 추정 원인: (7교시 문제 5에서 진단) V1-T17-P 전문 검색에서 "우주"만 매치된 문서와 "우주"+"드라이브" 둘 다 매치된 문서의 score가 동일(2.1201499)해 관련도 순위가 제대로 서지 않음. 원인은 `multi_match` 기본 type `best_fields`가 field 하나의 최고 점수만 쓰고 다른 field의 추가 매치를 반영하지 않기 때문
- 변경한 한 요소: `multi_match`의 `type`을 `best_fields`(기본값, 생략)에서 `most_fields`로 변경(다른 조건은 index·검색어·field·size 모두 동일하게 유지)
- 전/후 상위 3개: 변경 전 SONG-00065(청춘/우주, 드라이브/공부 — 우주만 확실히 매치), SONG-00069(우주만 매치), SONG-00100(우주+드라이브 매치) → score 모두 동일(2.1201499)로 뒤섞임. 변경 후 SONG-00100(theme:우주, situation:드라이브 — 3.98), SONG-00134(theme:우주, situation:드라이브 — 3.69), SONG-00140(theme:우주, situation:드라이브 — 3.69)로 theme+situation 양쪽에서 매치된 문서가 확실히 상위로 재배치됨
- 개선/보류/악화와 근거: 개선. `most_fields`로 바꾸자 "우주"와 "드라이브"를 모두 포함하는 문서(SONG-00100 등)가 score 3.98~3.69로 명확히 상위에 오고, 한 field에서만 매치된 문서와 점수 차이가 생겨 사용자 의도(우주 느낌 + 드라이브 상황을 모두 원함)에 더 맞는 순위가 됨

## (개인) 문제 5 — 최종 재현·산출물 완성

자기 전문 검색·정확 조건·bool/filter 요청을 새 Console에서 다시 실행하고 다른 사람이 commit만으로 재현할 수 있게 정리하세요.

### 역할·검증 기준

- 루트 `requests.http`에 `V1-T17-P`~`V1-T21-P`를 정리합니다.
- `docs/quality-test.md`에 질문별 기대·실제·개선 근거를 작성합니다.
- `evidence/day-03-search.md`에 핵심 결과와 commit SHA를 기록합니다.

### 최종 입력

- 새 Console 재현 성공 여부: 성공. 전문 검색·정확 조건·bool/filter 세 요청을 다시 실행해 모두 HTTP 200으로 재현됨(같은 index·field·조건으로 동일한 total 재확인: 325 / 269 / 26)
- 전문 검색 요청 ID: V1-T17-P (theme_keywords·situation_keywords multi_match "우주 드라이브")
- 정확 조건 요청 ID: V1-T18-P (genre term "Dance")
- bool/filter 요청 ID: V1-T19-P (mood_keywords match + genre·popularity_score filter)
- 품질표 경로: `day-03/practice/period-07-quality.md`의 문제 4 표(이 course repo 안에 작성됨). 개인 PBL 저장소의 `docs/quality-test.md`는 별도 파일이라 이번 세션에서는 아직 만들지 않음
- evidence 경로: `day-03/evidence/day-03-search.md` — 아직 템플릿 상태로 비어 있어 다음 단계에서 채워야 함
- 최종 commit SHA: 아직 커밋 전(현재 로컬 작업 트리 변경 상태, HEAD는 df221fb). 오늘 작업을 커밋한 뒤 이 항목에 실제 SHA를 기록해야 함
- 미완료 또는 재현 실패 항목: 재현 실패는 없음. 다만 (1) `evidence/day-03-search.md` 최종 요약 미작성, (2) 개인 PBL 저장소의 `requests.http`(V1-T17-P~T21-P)와 `docs/quality-test.md`는 이 course repo 범위 밖이라 별도로 개인 저장소에 옮겨 작성해야 함
