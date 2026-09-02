# Day 3 검색 구현·품질 검증 산출물

> 공통 쇼핑몰 답을 복사하지 않고 자신의 PBL index와 실제 결과를 기록합니다. 실행하지 않은 결과는 완료로 표시하지 않습니다.

## 1. 실행 기준

- 개인 index: kpop-songs
- 수업 시작 시 실제 `_count`: 1000 (`GET /kpop-songs/_count` → `{"count":1000}`)
- 개인 요청 파일: `requests.http` (`V1-T17-P`~`V1-T21-P` 구간, 아직 작성 전)
- 검색 품질 주 문서: `docs/quality-test.md` (아직 작성 전)
- 실행 환경·시각: 2026-09-02, 로컬 Docker(Elasticsearch/Kibana 9.5.0), Kibana Dev Tools. 1~4교시(Search API 기본, term/match, 전문 검색 확장, filter/range) 범위만 진행

## 2. 검색 질문과 요구사항

| 요청 ID | 사용자 질문 | 검색 field·검색어 | 정확 조건·범위 | 정렬 | 표시·highlight |
|---|---|---|---|---|---|
| Q01 전문 검색 | 우주 느낌 나는 드라이브용 노래 찾아줘 | theme_keywords, situation_keywords / "우주 드라이브" (multi_match) | 없음 | 없음 | 없음 |
| Q02 정확 조건 | Dance 장르 곡만 보여줘 | genre (term) / "Dance" | genre=Dance (keyword 정확 일치) | 없음 | 없음 |
| Q03 bool/filter | Dance 장르이면서 bpm 108인 곡 찾아줘 | genre + bpm (bool filter) | genre=Dance AND bpm=108 | 없음 | 없음 |

## 3. 실행 전 기대 기준

| 요청 ID | 기대 문서 ID·이유 | 제외 문서 ID·이유 | 의도한 0건 조건 | 경계 포함·제외 기준 |
|---|---|---|---|---|
| Q01 | SONG-00065/00069/00100 — theme_keywords에 "우주" 포함 | situation_keywords에 "드라이브"가 없는 곡(예: SONG-00069, 상황이 "여행") | 없음 | 해당 없음 |
| Q02 | SONG-00001 — genre 배열에 "Dance" 포함 | genre 배열에 "Dance"가 전혀 없는 곡 | 없음 | 해당 없음 (keyword 정확값) |
| Q03 | SONG-00001 — genre=Dance, bpm=108 둘 다 만족 | SONG-00002 — genre는 Dance지만 bpm이 108이 아님 | 없음 | 해당 없음 (bpm term 정확값) |

## 4. 실제 결과와 판정

| 요청 ID | `hits.total.value` | 상위 3개 ID | 조건·경계 통과 | 관련/보류/무관과 근거 | 판정 |
|---|---:|---|---|---|---|
| Q01 | 325 | SONG-00065, SONG-00069, SONG-00100 | 통과 | SONG-00065·00100은 우주+드라이브 둘 다 실제 포함(관련), SONG-00069는 우주만 있고 상황이 "여행"이라 보류 | 통과 |
| Q02 | 269 | SONG-00001, SONG-00002, SONG-00008 | 통과 | 3개 모두 `_source.genre`에 "Dance" 실제 포함 확인 | 통과 |
| Q03 | 3 | SONG-00001, SONG-00039, SONG-00606 | 통과 | 3개 모두 genre에 Dance 포함 + bpm=108 실제 확인. 기대했던 SONG-00002는 결과에서 제외됨(예상과 일치) | 통과 |

## 5. 조건 제거·변형 실험

| 기준 요청 | 바꾼 한 요소 | 변경 전 total·대표 ID | 변경 후 total·새로 들어온/빠진 ID | 관찰한 역할 |
|---|---|---|---|---|
| popularity_score 범위 검색(80~90) | `gte/lte` → `gt/lt` | 102건, 대표 SONG-00002(popularity_score=85) | 86건, popularity_score가 정확히 80 또는 90인 16건(값 80: 9건, 값 90: 7건) 빠짐 | `gte/lte`는 경계값 포함, `gt/lt`는 경계값 제외한다는 것을 실제 데이터(102-86=16=9+7)로 검증함 |

## 6. 실패 원인 진단

- 문제: 아직 없음 (1~4교시 범위에서는 실패 사례 없이 모두 통과)
- 1차 원인 분류: 해당 없음
- 확인한 실제 근거: 해당 없음
- 다음 확인 또는 변경: 5~8교시(정렬/highlight, bool 심화, 품질 검증, 통합) 진행하며 계속 기록 예정

## 7. 개선 전후

| 문제 | 추정 원인 | 변경한 한 요소 | 같은 조건으로 재실행한 결과 | 개선 판정과 근거 |
|---|---|---|---|---|
| 아직 없음 | - | - | - | 5~8교시 진행 후 기록 예정 |

## 8. 완료 체크

- [x] 전문 검색 요청 1개 (Q01)
- [x] 정확 조건 요청 1개 (Q02)
- [x] bool/filter 요청 1개 (Q03)
- [x] filter 2개 이상 (Q03: genre + bpm)
- [ ] sort 2개
- [ ] highlight 1개
- [ ] 의도한 0건 요청 1개
- [x] 상위 3건 사람 평가 (Q01)
- [ ] 개선 1건과 전후 결과
- [ ] README의 기능 목록·실행 경로 동기화
- [ ] 최종 commit SHA: (5~8교시 완료 후 업데이트 예정)
