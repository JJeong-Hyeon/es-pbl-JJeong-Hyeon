# Day 2 데이터 준비 결과

> 예시 문장을 복사하지 말고 자신의 실제 실행 결과를 작성합니다.
> 실행하지 않은 항목은 완료로 표시하지 않습니다.

## 1. Index와 문서

- Index 이름: kpop-songs
- 문서 한 건의 의미: 사용자가 입력한 주제·분위기·상황 조건에 가장 잘 맞는 K-POP 노래 1곡
- 실제 색인 건수: 1000 (`GET /kpop-songs/_count` → `{"count":1000}`)
- Mapping의 `dynamic` 설정: strict (미정의 필드로 색인 시도 시 `strict_dynamic_mapping_exception` 발생 확인)

## 2. 최종 Field

| Field | Type | 검색에서 사용할 목적 |
|---|---|---|
| song_id | keyword | 문서 고유 식별자, CRUD 조회/수정/삭제 |
| title | text | 곡 제목 전문 검색 |
| artist | text + keyword | 아티스트 검색(text) 및 아티스트별 집계(keyword) |
| release_date | date | 발매 시기 범위 조건, 최신순 정렬 |
| genre | keyword | 장르 필터 및 장르별 집계 |
| theme_keywords | text | 우주/여름/이별 등 가사 주제 연관 검색 |
| mood_keywords | text | 청량한/몽환적인 등 분위기 검색 |
| situation_keywords | text | 새벽/드라이브 등 상황 검색 |
| bpm | integer | 빠르기 범위 조건, 정렬 |
| popularity_score | integer | 인기도 범위 조건, 정렬 |

## 3. 대량 데이터 생성·색인 결과

- 생성 건수: 1000 (Seed=20260901, SampleCount=30, `day-02/data/pbl-data-template/my-data-settings.ps1`)
- 로컬 검증 결과: `generate-data.ps1` 실행 시 `Assert-DocumentMapping` 통과. `load-data.ps1` 내부에서 호출된 `validate-data.ps1`도 `LOCAL CHECK PASS: 1000 documents, unique IDs, target index and NDJSON verified.` 통과.
- Bulk 색인 결과: `PASS: Bulk item errors=false. Actual count=1001, generated=1000.` → 이전 curl 테스트로 남아있던 문서(`_id:1`) 중복 발견, `DELETE /kpop-songs/_doc/1`로 정리.
- ES 실제 `_count`: 정리 후 1000 (생성 건수와 일치)
- 분류·숫자·boolean 분포 확인 결과:
  - genre terms: Dance 269 / Rock 257 / Hip-hop 252 / R&B 245 / Electronic 243 / Ballad 239 (문서당 1~2개라 합이 1000 초과, 6개 장르 고르게 분포)
  - bpm stats: min 60 / max 170 / avg 114.9 (설정 범위 60~170과 일치)
  - popularity_score stats: min 0 / max 100 / avg 49.3 (설정 범위 0~100과 일치)
  - boolean 필드 없음 (mapping에 boolean 필드 미정의)
  - **발견한 문제**: `mood_keywords`, `theme_keywords`, `situation_keywords`는 `text` 타입이라 `terms` aggregation 시 `Fielddata is disabled` 오류 발생. Day 4 Dashboard에서 이 필드들로 분포 차트를 만들려면 `keyword` 하위 field 추가가 필요함 (Day 2 T11 mapping을 다시 검토할지, Day 4 전에 mapping을 보완할지 결정 필요).

## 4. Day 3 연결

- 검색 질문 기준: `day-01/pbl-start-card.md` 및 `day-01/data-model-template.md`의 사용자 질문 3개(2020년 이후 우주 느낌 몽환적인 노래 / 여름 바다 청량하고 신나는 노래 / 새벽에 혼자 들을 잔잔하고 몽환적인 노래)
- 각 질문에 대응하는 고정 문서를 데이터에 미리 배정함: SONG-00001(Orbit, Q1), SONG-00002(Neon Wave, Q2), SONG-00003(Afterglow, Q3)

## 5. 결과 파일 위치

- Mapping: `day-02/elasticsearch/index-create.json`
- 실행 요청: 아직 없음 (requests.http 미작성 — 다음 작업)
- 대표 문서: `day-01/data-model-template.md`(섹션 2), `day-02/data/sample-documents.json`
- 데이터 생성 설정: `day-02/data/pbl-data-template/my-data-settings.ps1`
- 생성 표본: `day-02/data/pbl-data-template/generated/kpop-songs-sample-30.ndjson`
- 생성 요약: `day-02/data/pbl-data-template/generated/generation-summary.json`

## 6. Pipeline 적용 판단

- 적용 / 미적용 / 보류: 보류
- 판단 이유: T16 판단 절차(공통 simulate 관찰 포함)를 아직 진행하지 않음. 현재 데이터 생성기에서 값 정규화(예: 대소문자, 공백)가 이미 처리되고 있어 ingest pipeline 없이도 문제가 없는지 다음 단계에서 확인 후 결정 예정.

## 7. 미완료·오류

- 현재 상태: T13(analyzer 관찰), T14(정식 CRUD 절차), T16(pipeline 판단) 미완료. `mood_keywords` 등 text 필드의 aggregation 제약 발견(위 3번 참고).
- 다음에 할 작업: analyzer 비교 실행, CRUD 5단계(생성→조회→수정→재조회→삭제→재조회) 기록, pipeline 적용 여부 결정, text 필드 aggregation 문제에 대한 mapping 보완 여부 결정, `elasticsearch/requests.http` 작성.
