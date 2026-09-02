# 3교시 실습 — 전문 검색 확장

## (공통) 문제 1 — 제공 코드로 여러 field 검색

```http
GET /products/_search
{
  "size": 5,
  "query": {
    "multi_match": {
      "query": "무선 이어폰",
      "fields": ["name", "description"]
    }
  }
}
```

### 결과 입력

- `hits.total.value`: 505
- 상위 3개 ID·name: P-00241(SoundLab 프리미엄 무선 이어폰), P-00305(Auralis 실속형 무선 이어폰), P-00529(NeoTech 스마트 무선 이어폰)
- 각 문서가 name·description 중 어디에서 의도와 연결되는가: 세 문서 모두 name에 "무선 이어폰"이 그대로 포함되어 매치의 핵심은 name. description은 공통 문구("~에 잘 어울리는 전자기기 상품입니다")라 변별력이 낮음
- 상위 3개 관련/보류/무관 판정: 3개 모두 관련

## (공통) 문제 2 — field boost 직접 구현

문제 1과 같은 조건을 유지하되 `name` 일치를 `description`보다 3배 중요하게 보는 Search API를 작성하세요.

### API 전체 입력

```http
GET /products/_search
{
  "size": 5,
  "query": {
    "multi_match": {
      "query": "무선 이어폰",
      "fields": ["name^3", "description"]
    }
  }
}
```

### 비교 결과

- 변경 전 상위 3개 ID: P-00241, P-00305, P-00529
- 변경 후 상위 3개 ID: P-00241, P-00305, P-00529 (동일)
- 순위가 달라진 문서와 이유: 상위 3개 순위는 변화 없음. score는 6.76 → 20.28로 커졌지만, 이 문서들은 boost 전에도 이미 name에서 강하게 매치해 1~3위였기 때문에 재배열이 상위권에서는 드러나지 않음
- boost가 사용자 의도에 유리했는가: 상위권만 보면 차이 없음. 다만 name에는 검색어가 없고 description에만 있는 문서를 상대적으로 밀어내는 방향이라, 하위권까지 비교하면 효과가 드러날 것

## (공통) 문제 3 — 구문 검색 직접 구현

`products` index의 `name`에서 `무선 이어폰`이라는 단어 순서와 인접성을 중요하게 검색하세요. `slop`은 0, 최대 5건으로 구현하세요.

### API 전체 입력

```http
GET /products/_search
{
  "size": 5,
  "query": {
    "match_phrase": { "name": { "query": "무선 이어폰", "slop": 0 } }
  }
}
```

### 결과 입력

- `hits.total.value`: 249 (문제 1의 505보다 감소)
- 상위 문서 ID·name: P-00241(SoundLab 프리미엄 무선 이어폰), P-00305(Auralis 실속형 무선 이어폰), P-00529(NeoTech 스마트 무선 이어폰)
- 문제 1보다 결과가 같거나 줄어든 이유: match_phrase는 "무선"과 "이어폰"이 정확히 이 순서로 인접해야 매치됨. "무선 청소기"처럼 한 단어만 있거나 순서가 다른 문서는 제외되어 total이 줄어듦
- 구문 의도에 맞지 않는 문서가 있는가: 없음. 상위 5개 모두 실제로 "무선 이어폰" 문구를 그대로 포함

## (개인) 문제 4 — 여러 text field 검색

자기 프로젝트에서 같은 사용자 검색어가 적용될 수 있는 text field 2개 이상을 선택해 전문 검색을 구현하세요.

### 역할·검증 기준

- 각 field의 서비스 역할을 설명합니다.
- 상위 3개 문서를 사람이 평가합니다.
- 한 field만 필요한 도메인이라면 `match`를 선택하고 그 이유를 적어도 됩니다.

### API와 결과 입력

```http
GET /kpop-songs/_search
{
  "size": 5,
  "_source": ["song_id", "title", "theme_keywords", "situation_keywords"],
  "query": {
    "multi_match": {
      "query": "우주 드라이브",
      "fields": ["theme_keywords", "situation_keywords"]
    }
  }
}
```

- 사용자 질문·검색어: "우주 느낌 나는 드라이브용 노래 찾아줘" / "우주 드라이브"
- 선택 field와 역할: theme_keywords(곡의 주제·컨셉), situation_keywords(듣기 좋은 상황·장소·시간대) — 같은 검색어가 두 field 의도로 나뉠 수 있어 함께 검색
- 상위 3개 판정: SONG-00065(우주+드라이브 둘 다 매치, 관련), SONG-00069(우주만 매치·상황은 여행, 보류), SONG-00100(우주+드라이브 둘 다 매치, 관련)
- query 선택 근거: 사용자 의도가 theme와 situation 두 field에 걸쳐 있어 multi_match로 두 field를 동시에 검색

## (개인) 문제 5 — boost 또는 phrase 가설 검증

자기 검색에서 field boost 또는 phrase 중 하나를 선택해 기본 요청과 비교하세요.

### 역할·검증 기준

- 같은 index·데이터·검색어·size를 유지합니다.
- 한 요소만 변경합니다.
- 결과가 바뀌지 않아도 실제 결과대로 기록합니다.

### API와 결과 입력

```http
GET /kpop-songs/_search
{
  "size": 5,
  "_source": ["song_id", "title", "theme_keywords", "situation_keywords"],
  "query": {
    "multi_match": {
      "query": "우주 드라이브",
      "fields": ["theme_keywords^3", "situation_keywords"]
    }
  }
}
```

- 선택한 가설: theme_keywords가 situation_keywords보다 곡 정체성에 더 중요하므로 3배 boost하면 관련도 높은 곡이 상위로 온다
- 변경 전·후 상위 3개: SONG-00065, SONG-00069, SONG-00100 (동일). score만 2.12 → 6.36으로 상승
- 개선/보류/악화 판정: 보류
- 판정 근거: size=5 범위 안 문서가 모두 이미 theme_keywords에서 매치돼 boost가 순위에 영향을 못 줌. situation_keywords에만 매치되는 문서와 비교하거나 size를 늘려야 효과가 드러날 것
