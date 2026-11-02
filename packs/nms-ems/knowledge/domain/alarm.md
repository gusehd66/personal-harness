# Alarm 도메인 노트

## 사용 시점
current alarm, alarm history, severity, fault display, alarm filtering, alarm count 변경 시 이 문서를 사용한다.

## 읽기 경로
- 화면이 active alarm, history alarm, summarized count, cached display state 중 무엇을 읽는지 식별한다.
- color 또는 label을 바꾸기 전에 severity mapping과 user preference configuration을 확인한다.

## 쓰기 경로
- alarm acknowledgement, clear, inhibit, preference 변경은 backend API와 refresh path를 식별해야 한다.
- active/history alarm semantics를 조용히 바꾸지 않는다.

## 위험
- 잘못된 severity display는 운영자를 오도할 수 있다.
- active/history 혼동은 live fault를 숨기거나 old fault를 중복 표시할 수 있다.
- filter 변경은 backend data가 있어도 alarm이 없는 것처럼 보이게 만들 수 있다.

## 변경 전 확인
- 현재 severity enum/string mapping을 확인한다.
- table/filter state와 route parameter를 확인한다.
- 변경이 active alarm, history alarm, display-only rendering 중 무엇에 영향을 주는지 보고한다.
