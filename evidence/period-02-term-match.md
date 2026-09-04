# 2교시 실습 — term과 match

## (공통) 문제 1 — 제공 코드로 정확 조건 확인

```http
GET /products/_search
{
  "size": 5,
  "query": { "term": { "category": "전자기기" } }
}
```

### 결과 입력

- `hits.total.value`: 1250
- 상위 3개 문서 ID: P-00009, P-00025, P-00081
- 상위 3개 문서의 category: 셋 다 "전자기기"
- 모든 확인 문서가 정확 조건을 만족하는가: 예
- `term`을 선택한 mapping 근거: `category`는 `keyword` type이라 analyzer 없이 입력값 전체가 하나의 token으로 저장됨. 따라서 부분 매치가 아니라 "전자기기"라는 문자열 전체와 정확히 같은 문서만 찾는 `term`이 적합함

## (공통) 문제 2 — text 전문 검색 직접 구현

`products` index에서 상품명 `name`에 `무선`이라는 검색 의도가 있는 문서를 찾으세요. text 전문 검색에 적합한 query를 선택해 최대 5건을 반환하세요.

### API 전체 입력

```http
GET /products/_search
{
  "size": 5,
  "query": { "match": { "name": "무선" } }
}
```

### 결과 입력

- 선택한 query와 이유: `match`. `name`은 `korean_search` analyzer가 적용된 `text` field라 검색어도 같은 방식으로 분석해 형태소 단위로 매치시키는 `match`가 적합함
- `hits.total.value`: 505
- 상위 3개 ID·name: P-00025(MobiCore 컴팩트 무선 이어폰), P-00042(CleanMate 실속형 무선 청소기), P-00129(Auralis 스마트 무선 이어폰)

## (공통) 문제 3 — 부적절한 조합 비교

같은 `name` field와 `무선` 검색어에 `term` query를 사용한 API를 직접 작성하세요. 문제 2와 결과를 비교하고, 차이를 mapping 또는 분석된 token 관점에서 설명하세요.

### API 전체 입력

```http
GET /products/_search
{
  "size": 5,
  "query": { "term": { "name": "무선" } }
}
```

### 비교 결과

- 문제 2 total / 문제 3 total: 505 / 505 (동일)
- 공통으로 나온 문서 ID: P-00025, P-00042, P-00129, P-00153, P-00209 (상위 5개 전부 동일)
- 달라진 이유: 이번 경우엔 달라지지 않음. `name`의 `korean_search`(nori) analyzer가 "무선 이어폰" 같은 문자열을 형태소 단위로 쪼개면서 "무선"이라는 token을 그대로 만들어내기 때문에, 색인된 token 목록에 "무선"이 정확히 존재해 `term`도 우연히 매치됨
- `term`은 text에서 항상 0건인가? 실제 근거: 아니오. `term`은 analyzer를 타지 않고 입력값을 색인된 token과 문자 그대로 비교할 뿐이라, 검색어가 analyzer 처리 후 실제로 생성되는 token과 우연히 같으면 매치된다. 여기서는 nori가 "무선"을 독립 형태소로 분리해 token화하기 때문에 결과가 같았지만, "무선이어폰"처럼 붙여서 검색하면 색인된 token(무선/이어폰 분리)과 문자 그대로 일치하지 않아 0건이 될 수 있음

## (개인) 문제 4 — 자기 정확 조건 검색

자기 mapping에서 값 전체가 정확히 일치해야 하는 `keyword` 또는 `boolean` field 하나를 선택해 정확 조건 검색을 구현하세요.

### 역할·검증 기준

- 실제 존재하는 field와 값을 사용합니다.
- 반환 문서의 `_source`에서 조건을 직접 확인합니다.
- 왜 전문 검색이 아니라 정확 비교인지 설명합니다.

### API와 결과 입력

```http
GET /kpop-songs/_search
{
  "size": 10,
  "_source": ["song_id", "title", "genre"],
  "query": { "term": { "genre": "Dance" } }
}
```

- field / type / 값: `genre`(keyword) = "Dance"
- 사용자 질문: "댄스 장르 노래만 보여줘"
- 상위 3개 ID와 실제 값: SONG-00001(genre: [Dance, Electronic]), SONG-00002(genre: [Dance]), SONG-00008(genre: [Dance, Rock]) — 셋 다 genre 배열에 "Dance" 포함
- 통과/실패와 근거: 통과. `hits.total.value` 269건 모두 genre 배열에 "Dance"가 정확히 포함된 문서만 반환됨(keyword라 부분 매치나 형태소 분석 없이 정확한 값 비교)

## (개인) 문제 5 — 자기 전문 검색

자기 mapping의 `text` field 하나와 사용자가 입력할 검색어를 정해 전문 검색 API를 구현하세요.

### 역할·검증 기준

- field가 실제 `text`인지 mapping으로 확인합니다.
- 상위 3개 결과를 관련/보류/무관으로 판정합니다.
- 정확 조건 문제와 query 선택 이유가 달라야 합니다.

### API와 결과 입력

```http
GET /kpop-songs/_search
{
  "size": 5,
  "_source": ["song_id", "title", "mood_keywords"],
  "query": { "match": { "mood_keywords": "신나는" } }
}
```

- field / type / 검색어: `mood_keywords`(text) / 검색어 "신나는"
- 상위 3개 ID: SONG-00016(Echo, mood_keywords: [신나는]), SONG-00059(Glassy, mood_keywords: [신나는]), SONG-00159(Blackout City, mood_keywords: [신나는])
- 관련/보류/무관과 이유: 3개 모두 관련. mood_keywords 배열에 "신나는"이 그대로 들어있어 사용자가 원하는 신나는 분위기 곡 추천 의도와 일치함
- 완료 판정: 통과. 정확 조건 문제(문제 4, genre keyword)와 달리 이번엔 text field를 형태소 분석해 매치하는 전문 검색이라 query 선택 근거가 다름
