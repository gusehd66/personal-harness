# WC 공통 컴포넌트 인덱스

## 목적
이 문서는 `wc-*` 공통 컴포넌트를 모두 1파일씩 쪼개지 않기 위한 인덱스다.

기본 원칙:
- 처음에는 역할별 묶음 문서에 기록한다.
- 특정 컴포넌트가 길어지거나 변경 위험이 커지면 그때 별도 문서로 승격한다.
- 화면을 새로 만들기 전에 기존 `wc-*` 컴포넌트와 사용처를 먼저 확인한다.

## 묶음 문서
| 묶음 | 문서 | 포함 대상 |
| --- | --- | --- |
| Data Display | `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/wc-data-display.md` | `wc-table`, `wc-treetable`, `wc-treeview`, row action, renderer, selection, pagination |
| Form Controls | `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/wc-form-controls.md` | `wc-form`, `wc-form-row`, `wc-form-label`, `wc-form-value`, validation, combobox, date/time, input 계열 |
| Layout | `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/wc-layout.md` | `wc-card`, `wc-toolbar`, `wc-split-panel`, `wc-scroll-panel`, `wc-dialog`, `wc-window`, `wc-tab` |
| NMS Topology | `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/topology-canvas.md` | topology canvas, shelfview, 장비/링크 시각화 |

실제 사용처를 찾을 때는 `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/wc-usage-map.md`를 함께 확인한다.

## 별도 유지 문서
NMS 도메인 의미가 강하거나 화면 구조 이해에 직접 필요한 문서는 별도 파일로 유지한다.

- `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/side-menu.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/group-tree.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/topology-canvas.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/detail-panel.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/components/command-toolbar.md`

## 분리 승격 기준
다음 중 하나에 해당하면 묶음 문서에서 별도 문서로 분리한다.

- 한 컴포넌트 설명이 너무 길어져 다른 컴포넌트 탐색을 방해한다.
- action, renderer, validation, lifecycle 같은 위험 지점이 많다.
- 여러 화면에서 반복적으로 버그가 발생한다.
- 매뉴얼 캡처나 테스트 절차가 해당 컴포넌트에 강하게 의존한다.
- 신규 개발자가 자주 실수하는 anti-pattern이 많다.

## 변경 전 확인
- `src/app/temp/wccontrol/` 아래 sample이 있으면 먼저 확인한다.
- 실제 product 화면 사용처를 `rg`로 확인한다.
- site-specific replacement가 있는 경우 `sites/<site>/` 아래 사용처도 확인한다.
- 공통 컴포넌트 변경은 여러 product 화면에 영향을 줄 수 있으므로 사용처와 위험 범위를 결과 보고에 남긴다.
