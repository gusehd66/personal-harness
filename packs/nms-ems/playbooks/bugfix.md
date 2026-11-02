# 버그 수정 플레이북

## 절차
1. 증상을 먼저 적는다.
2. 실제 실패한 객체, route, process, API 경로, data field를 찾는다.
3. 현재 읽기 경로와 쓰기 경로를 추적한다.
4. 활성 실행 분기를 확인한다. 특히 site-specific file replacement와 provider를 확인한다.
5. 안전한 최소 변경만 적용한다.
6. 가장 좁고 유용한 명령 또는 수동 경로로 검증한다.
7. 코드 변경으로 `<HARNESS_ROOT>/core/` 또는 `<HARNESS_ROOT>/packs/nms-ems/knowledge/` 갱신이 필요한지 확인한다.
8. 반복 가능한 절차나 판단 로직이 Codex 스킬 후보인지 확인한다.
9. 검증하지 못한 항목을 보고한다.

## 필수 기록
- 근본 원인과 증상을 분리한다.
- 관련 없는 정리 작업으로 범위를 넓히지 않는다.
- 기존 구조와 오류 처리 방식을 유지한다.
- NMS 도메인 변경이라면 관련된 NE/site/shelf/port, alarm, PM, sync, queue, session state 영향을 포함한다.

## 출력
`<HARNESS_ROOT>/core/templates/result-report.md`를 사용한다.
