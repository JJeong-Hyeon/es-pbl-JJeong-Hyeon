# Day 4 개인 Dashboard 설계

## 1. 사용자와 목적

- 내 주제: kpop-songs (케이팝 곡 카탈로그)
- 이 Dashboard를 볼 사람: 매주 플레이리스트를 만드는 나(청취자)
- Dashboard를 보고 결정하거나 행동할 것: 이번 주에 어떤 장르·템포대·연도의 곡을 우선 들을지 정하고, Discover에서 그 조건으로 다시 필터링해 실제 곡을 고른다
- 사용할 index / Data View: kpop-songs

## 2. 데이터 준비 경로

- [x] A: 개인 데이터로 제작
- [ ] B: 공통 products로 제작하며 개인 데이터 보강 규칙 작성
- [ ] C: 공통 Dashboard를 완성하고 개인 청사진에 집중

선택 이유: kpop-songs가 이미 1,000건이며 Q1~Q4에 필요한 field(genre, bpm, release_date)가 모두 keyword/integer/date로 존재해 보강 없이 4개 패널을 바로 만들 수 있음

## 3. 질문-데이터-차트 청사진

| 번호 | 분석 질문 | 필요한 field | 현재 존재? | mapping type | 계산·그룹 방식 | 차트 | filter/control | 확인 기준 |
|---|---|---|---|---|---|---|---|---|
| Q1 전체 규모 | 등록된 곡은 총 몇 곡인가? | (없음) | - | - | Count of records | Metric | 없음 | 1,000 |
| Q2 그룹 비교 | 장르별 곡 수는 어떻게 다른가? | genre | 존재 | keyword | Top values(6), Count | Bar | genre Control(선택) | 6개 장르, 239~269 |
| Q3 분포/정확한 값 | BPM은 어느 구간에 몰려 있는가? | bpm | 존재 | integer | Intervals(20 단위 custom ranges) | Bar | 없음 | 6구간, 합계 1,000 |
| Q4 상태/시간 | 연도별 발매곡 수는 어떻게 달라지는가? | release_date | 존재 | date | Date histogram(1y) | Line | 없음 | 2015~2026, 연도별 54~97 |

## 4. 데이터 부족 분석

- 현재 데이터로 답할 수 없는 질문: 무드(mood)별 평균 인기도는 어떻게 다른가?
- 부족한 field: mood_keywords (현재 순수 text, keyword 서브필드 없음)
- 필요한 mapping type: text + `fields: { keyword: { type: keyword } }`
- 필요한 값의 범위·범주·비율: moods 후보 8개(청량한/몽환적인/잔잔한 등), 문서당 1~3개
- 날짜가 필요하다면 기간과 단위: 해당 없음
- 한 문서가 의미할 사건 또는 대상: 해당 없음(기존 문서 재사용)
- 생성 또는 수집 방법: 새 문서 생성이 아니라 기존 1,000건의 mapping만 재정의(keyword 서브필드 추가)한 뒤 재색인
- 데이터 수가 충분하다고 판단할 기준: 재색인 후 `mood_keywords.keyword`로 Terms aggregation을 실행했을 때 8개 무드 값이 모두 정상 집계되면 충분

## 5. 제작 순서

1. Metric(전체 곡 수) 패널 생성 — Records/Count
2. Bar(genre Top values) 패널 생성
3. Bar(bpm custom ranges) 패널 생성
4. Line(release_date, 1y interval) 패널 생성 후 전체 화면 캡처

## 6. 완료 예상 화면

- Dashboard 제목: D4 개인 미션 - kpop-songs
- 필수 패널 수: 4개 (Metric, genre Bar, bpm Bar, release_date Line)
- 사용할 control/filter: 없음 (선택 사항, 필수 기준은 패널 4개 이상 + filter/control 1개 이상)
- 저장할 캡처 파일명: evidence/day-04-practice/p07-personal-dashboard.jpg
