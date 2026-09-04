# Day 4 — Data View·Discover 탐색 증거

- Data View: kpop-songs
- time field: release_date
- index: kpop-songs
- 확인한 field: song_id, title, artist, release_date, genre, theme_keywords, mood_keywords, situation_keywords, bpm, popularity_score
- 절대 시간 범위: 2015-01-01 00:00 ~ 2026-12-31 23:59 (데이터 생성 범위 전체 포함)
- Discover 실제 문서 수: 1,000
- 판정: 정상. `GET kpop-songs/_count` 결과(1,000)와 정확히 일치
