# Common Table Component 노트

이 문서는 기존 참조 호환을 위해 유지한다.

현재 table/list 관련 공통 지식은 다음 문서를 우선 사용한다.

- `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/wc-components.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/wc-data-display.md`

## 요약
- 새 table을 만들기 전에 기존 `wc-table`, renderer, action column, selection, pagination 패턴을 확인한다.
- table column 변경은 매뉴얼, export/download, API response contract에 영향을 줄 수 있다.
- 조회 작업 중 `Action` column button은 직접 클릭하지 않는다.
