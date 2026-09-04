# 7교시 실습 — 검색 품질 점검

## (공통) 문제 1 — 제공 코드로 상위 결과 평가

```http
GET /products/_search
{
  "size": 10,
  "query": {
    "multi_match": {
      "query": "무선 이어폰",
      "fields": ["name^3", "description"]
    }
  }
}
```

### 결과 입력

| 순위 | 문서 ID | name | 관련/보류/무관 | 근거 |
|---:|---|---|---|---|
| 1 | P-00241 | SoundLab 프리미엄 무선 이어폰 | 관련 | name에 "무선 이어폰"이 그대로 포함, 검색 의도와 정확히 일치 |
| 2 | P-00305 | Auralis 실속형 무선 이어폰 | 관련 | name에 "무선 이어폰"이 그대로 포함 |
| 3 | P-00529 | NeoTech 스마트 무선 이어폰 | 관련 | name에 "무선 이어폰"이 그대로 포함 |

- `hits.total.value`: 505
- 결과 수가 많다는 사실만으로 품질이 좋다고 말할 수 있는가: 아니오. total 505는 "무선"이나 "이어폰" 중 하나만 들어간 문서(예: 무선 청소기, 유선 이어폰)까지 넓게 포함할 수 있는 수치다. 실제 품질은 상위 순위 문서가 사용자 의도("무선 이어폰")와 얼마나 정확히 일치하는지로 판단해야 하며, 이번 상위 3개는 실제로 관련성이 높았지만 total 숫자 자체는 품질의 근거가 아님

## (공통) 문제 2 — 정확 조건 품질 직접 구현

category가 `전자기기`이고 `in_stock=true`인 상품만 검색하는 API를 작성하세요. 최대 10건을 반환하고 모든 결과가 두 조건을 만족하는지 확인하세요.

### API 전체 입력

```http
GET /products/_search
{
  "size": 10,
  "_source": ["product_id", "name", "category", "in_stock"],
  "query": {
    "bool": {
      "filter": [
        { "term": { "category": "전자기기" } },
        { "term": { "in_stock": true } }
      ]
    }
  }
}
```

### 결과 입력

- `hits.total.value`: 1065
- 기대 문서 ID·이유: P-00009(NeoTech 데일리 기계식 키보드, category=전자기기·in_stock=true라 기대), P-00025(MobiCore 컴팩트 무선 이어폰, 동일 조건 만족)
- 제외 문서 ID·이유: P-00003(Morrow 실속형 오버핏 후드, category=패션이라 제외돼야 함) — 별도 문제 3에서 조회했을 때 실제로 category가 패션이고 in_stock=false임을 확인함
- 확인한 모든 문서가 조건을 통과했는가: 예. 상위 10개 문서(P-00009~P-00313)의 category가 모두 "전자기기", in_stock이 모두 true로 확인됨

## (공통) 문제 3 — 의도한 0건 직접 구현

실제 `products` index와 실제 `product_id` field를 사용하되, 데이터에 존재하지 않는 값 `__DAY03_INTENTIONAL_ZERO__`를 검색해 정상적인 0건 요청을 구현하세요.

### API 전체 입력

```http
GET /products/_search
{
  "size": 5,
  "query": {
    "term": { "product_id": "__DAY03_INTENTIONAL_ZERO__" }
  }
}
```

### 결과 입력

- HTTP 성공 여부: 성공 (200, `timed_out: false`, `_shards.failed: 0`)
- `hits.total.value`: 0
- index·field가 실제 존재한다는 근거: 같은 요청을 `product_id: "P-00003"`으로 바꿔 실행하면 실제 문서 1건이 정상 반환됨(`hits.total.value: 1`) — index와 field 자체는 정상 동작함
- 값만 존재하지 않는다는 근거: 요청 구조·field명은 위와 동일하고 검색값만 `__DAY03_INTENTIONAL_ZERO__`로 바꿨을 뿐인데 결과가 0건으로 나옴. mapping 오류나 field 오타였다면 애초에 `_shards`가 실패하거나 예외가 발생했을 것
- 오류 0건과 정상 0건의 차이: 오류로 인한 0건은 보통 `_shards.failed > 0`, HTTP 4xx/5xx, 또는 응답에 `error` 객체가 포함됨(index 없음, field mapping 충돌 등). 이번 요청은 `_shards.successful: 3`, `failed: 0`으로 정상 실행됐고 단지 조건에 맞는 문서가 없어서 0건인 것 — 요청 자체는 정상이고 데이터가 없을 뿐임

## (개인) 문제 4 — 자기 질문 3개 품질 점검

자기 PBL의 전문 검색, 정확 조건, bool/filter 질문을 각각 실행하고 기대 문서와 제외 문서를 판정하세요.

### 역할·검증 기준

- 질문마다 독립된 요청 ID를 사용합니다.
- 실행 전에 기대·제외 기준을 적습니다.
- 상위 3개를 실제 `_source`로 평가합니다.

### 결과 입력

```http
GET /kpop-songs/_search
{ "size": 5, "_source": ["song_id","title","theme_keywords","situation_keywords"],
  "query": { "multi_match": { "query": "우주 드라이브", "fields": ["theme_keywords","situation_keywords"] } } }

GET /kpop-songs/_search
{ "size": 5, "_source": ["song_id","title","genre"],
  "query": { "term": { "genre": "Dance" } } }

GET /kpop-songs/_search
{ "size": 5, "_source": ["song_id","title","genre","mood_keywords","popularity_score"],
  "query": { "bool": {
    "must": [{ "match": { "mood_keywords": "신나는" } }],
    "filter": [{ "term": { "genre": "Dance" } }, { "range": { "popularity_score": { "gte": 70 } } }]
  } } }
```

| 질문 | 요청 ID | 기대 ID | 제외 ID | 상위 3개 | 판정·근거 |
|---|---|---|---|---|---|
| 전문 검색 | V1-T17-P | SONG-00065, SONG-00100(우주+드라이브 둘 다 포함) | 우주·드라이브 둘 다 없는 곡 | SONG-00065(우주+드라이브 관련), SONG-00069(우주만, 보류), SONG-00100(우주+드라이브 관련) | total 325건 중 상위는 관련이지만 우주만 매치된 SONG-00069·SONG-00102 같은 보류 사례도 상위권에 섞임 → 통과(부분 보류 포함) |
| 정확 조건 | V1-T18-P | genre 배열에 "Dance"가 정확히 포함된 곡 | genre에 "Dance"가 없는 곡(Ballad 단독 등) | SONG-00001(genre:[Dance,Electronic]), SONG-00002(genre:[Dance]), SONG-00008(genre:[Dance,Rock]) | 통과. total 269건 모두 genre 배열에 Dance 정확 포함 확인 |
| bool/filter | V1-T19-P | genre=Dance이며 신나는 분위기·popularity 70 이상인 곡 | Dance가 아니거나 popularity 70 미만인 곡 | SONG-00853(Dance/신나는/84), SONG-00002(Dance/신나는/85), SONG-00031(Dance/신나는/70) | 통과. total 26건 모두 세 조건(genre, mood, popularity) 동시 만족 확인 |

## (개인) 문제 5 — 실패 원인 진단

개인 문제 4에서 실패·보류 또는 개선 여지가 있는 결과 하나를 선택해 원인을 진단하세요.

### 역할·검증 기준

- mapping / analyzer / query / filter / sort / data 중 1차 원인을 선택합니다.
- 실제 mapping·문서·응답 근거를 하나 이상 첨부합니다.
- 아직 요청을 고치지 말고 원인과 다음 실험을 분리합니다.

### 진단 입력

- 문제: V1-T17-P(전문 검색) 상위 5개의 `_score`가 2.1201499로 전부 동일함. "우주"와 "드라이브"를 모두 포함한 SONG-00065·SONG-00100과, "우주"만 포함한 SONG-00069·SONG-00102가 같은 점수로 묶여 있어 더 관련 있는 문서를 상위로 올리지 못함
- 1차 원인: query. `multi_match`의 기본 type인 `best_fields`는 여러 field 중 가장 잘 맞은 field 하나의 점수만 가져오고, 다른 field에서 추가로 매치된 부분을 점수에 반영하지 않음. 그래서 theme_keywords에서만 "우주"에 매치된 문서와, theme_keywords+situation_keywords 양쪽에서 각각 매치된 문서가 구분되지 않음
- 확인한 실제 근거: 응답 JSON에서 SONG-00065(theme:[청춘,우주], situation:[드라이브,공부] — 두 field 모두 매치 가능)와 SONG-00069(theme:[우주,사랑], situation:[여행] — situation에 드라이브 없음)의 `_score`가 둘 다 2.1201499로 동일하게 나옴
- 다음에 바꿀 한 요소: `multi_match`의 `type`을 `best_fields`에서 `most_fields` 또는 `cross_fields`로 바꾸거나, 각 field를 개별 `should` 절로 나눠 매치된 field 수만큼 점수가 누적되는지 확인
- 새 Console 재현 여부: 아직 미실행(문제 5는 원인 진단까지만 수행, 요청 변경은 8교시 문제 4에서 진행 예정)
