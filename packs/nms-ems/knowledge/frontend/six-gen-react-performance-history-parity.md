# Six-Gen React Performance History Parity

## 범위

- Angular 기준: `History > Performance`
- React URL: `/history/performance`
- 기준 viewport: 데스크톱 `1440x960`; 모바일은 현재 범위 밖이다.
- route 렌더링만으로 완료 처리하지 않고 setup, 조회, paging, graph, compare,
  download, profile 저장 계약을 함께 검증한다.

## 요청 계약

- setup 순서는 `Code -> GetIndex -> Search`다.
- `Code`와 `GetIndex`는 `performance/v1/performance`, `Search`는
  `performance/v1/<PerformanceCode.requestType>`를 사용한다.
- `Code` 요청 data는 선택한 NE 종류에 따라 COT `10100`, RT `10101` 숫자를 보낸다.
- `GetIndex`와 `Search`는 선택한 Angular `PerformanceCode` wire object를 그대로
  전달한다. unknown, nested, null, empty 필드를 보존하고 공통 serializer에서
  `__hashcode__`만 제거한다.
- `Search`는 선택 NE, index filter, start/end time, `limit_count`, `skip_count`를
  직렬화한다.
- 보호 요청의 `access_token` header와 body `client_session_id`는 활성 통신
  `session_id`와 같다.
- Code/GetIndex/Search/Profile 응답의 `status`가 없거나 `1`이면 정상 처리하고,
  그 외 값은 `error_message`를 화면 오류로 전달한다. app 오류나 HTTP 실패를
  빈 결과 또는 기본 row count로 바꾸지 않는다.
- 날짜는 Angular와 같이 millisecond와 timezone offset을 포함한다. 시작은
  `.000`, 종료는 `.999`를 유지한다.
- paging, graph, compare, download는 화면의 미제출 draft가 아니라 마지막으로
  성공한 Search 조건을 재사용한다.

## 화면 및 데이터 계약

- Network는 active six-gen COT/RT만 표시하고 첫 eligible COT를 기본 선택한다.
- group은 펼칠 수 있지만 실제 Search 대상은 NE만 선택할 수 있다.
- Code/Category/Item과 dependent index option은 backend source order를 보존한다.
- 초기 no-data DOM은 Angular와 같이 `Collect Time`, `Host Name`, `gNB ID`,
  `Cell ID` 네 header를 가진다. Search 이후 Six-Gen 결과는 `Collect Time`,
  `Host Name` 뒤에 선택 metric column을 붙인다. `gNB ID`, `Cell ID`는 이후
  graph series identity 계산에 사용한다.
- 결과가 없을 때 pager를 표시하지 않고 Angular와 같은 no-data 영역을 표시한다.
- 화면 CSS의 색상과 치수는 root `--performance-history-*` token을 사용하고,
  Recharts 색상과 수치 계약은 `performanceHistoryChartTokens`를 사용한다.
- table/graph 전환, compare panel, shared series visibility를 유지한다.
- `gNB ID`와 `Cell ID`는 Angular `cell_id` bit 계약으로 계산한다.
- graph series는 metric 하나당 하나로 합치지 않고 Angular와 같이
  `metric x gNB/Cell identity`로 분리한다.
- CSV는 화면용 변환 row가 아니라 backend raw export field를 사용한다.
- main/compare panel은 각각 최신 요청만 결과를 반영한다. 새 search, paging,
  rows 변경, download가 시작되면 같은 panel의 이전 요청을 abort하고 늦게
  도착한 응답은 loading/result 상태를 변경하지 않는다.

## Paging 및 Profile 계약

- row option은 `10/20/30/50/100`이다.
- profile API는 `system/v1/user-profile`, type `RowPerPage`, target `Common`이다.
- Performance key는 `HistoryPerformance`다.
- row count 변경은 기존 profile의 id와 다른 화면 값을 보존한 채 Add 또는
  Modify한다.

## CSV 계약

- download Search는 마지막 성공 조건에 `limit_count=0`, `skip_count=0`을 적용한다.
- UTF-8 BOM, Angular column order, row별 trailing comma와 LF newline을 유지한다.
- comma가 포함된 값만 quote한다.
- 파일명은 `PerformanceHistory_<NE>_<start>_<end>.csv`다.
- Angular와 같이 total count가 20,000 이하일 때만 전체 download를 요청한다.

## React 구현 위치

- 화면: `src/screens/performance/PerformanceHistoryScreen.tsx`
- setup: `src/screens/performance/usePerformanceHistorySetup.ts`
- 요청/응답: `src/screens/performance/performanceHistoryModel.ts`
- response 오류: `src/screens/performance/performanceHistoryResponse.ts`
- index cascade: `src/screens/performance/performanceHistoryIndexModel.ts`
- graph series: `src/screens/performance/performanceHistoryChartModel.ts`
- 최신 요청 제어: `src/screens/performance/performanceHistoryRequestCoordinator.ts`
- profile: `src/screens/performance/performanceHistoryProfile.ts`
- CSV: `src/screens/performance/performanceHistoryCsv.ts`
- browser QA: `scripts/qa-sixgen-performance-history.mjs`
- live parity QA: `scripts/qa-sixgen-performance-history-live.mjs`

## 검증

- `npm run typecheck:react`
- `npm run test:react`
- `npm run build:six-gen-flexible`
- `SIXGEN_REACT_URL=http://127.0.0.1:4317/unms node scripts/qa-sixgen-performance-history.mjs`
- `npm run qa:six-gen-performance-history-live`
- browser QA는 COT/RT model body, Reset, GetIndex/Search session contract, page 2,
  row profile Modify, compare/graph, 두 cell의 독립 series, 실제 download
  파일명/내용, 초기 4-header에서 Search 후 2-header로의 전이, 한국어 resource와
  `1440x960` 캡처를 검증한다.
- live QA는 실제 Angular/React 로그인, startup, backend Code 응답을 사용한다.
  당시 backend GetIndex는 application status `0`을 반환했고 Search는 비교 시간
  안에 완료되지 않아, GetIndex/Search 성공 응답만 양쪽에 동일하게 제어하여 전체
  request body, session field, control, toolbar, empty state와 geometry를 비교했다.
- 따라서 실제 backend Search 성공 응답과 실제 performance row는 아직 미검증이다.
  이 제한을 해소하기 전까지 전체 이식 완료 근거로 사용하지 않는다.
