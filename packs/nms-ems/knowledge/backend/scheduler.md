# Scheduler 지식

## 확인 대상
- `ScheduleJobIf`
- `ScheduleAdapterRat`
- `*SchedulerJob`
- Quartz `Scheduler`
- PM, alarm, sync, discovery, upgrade, backup/restore job

## 분석 순서
1. scheduler job class를 찾는다.
2. `execute(...)` 진입점과 fire time 사용 방식을 확인한다.
3. 대상 NE/site/group/PM category/alarm range를 확인한다.
4. 중복 실행 방지, lock, running flag, queue/backpressure 처리 여부를 확인한다.
5. DB/Mongo insert/update path를 추적한다.
6. 실패 시 retry, skip, partial failure, stale data 가능성을 기록한다.

## 하네스 기준
scheduler fixture는 아래를 포함한다.

- job class
- fire time
- 대상 NE/site/group
- 입력 PM/alarm/sync 상태
- lock/running state
- mock worker/process 결과
- expected DB/Mongo write
- failure/retry/skip case

## 위험 기준
- 중복 scheduler 실행
- PM/alarm 데이터 손실
- stale display
- queue/backpressure
- sync overlap
- multi-site 동시성

기본 하네스는 scheduler를 실제로 trigger하지 않고, job 입력과 worker/process 호출을 mock으로 검증한다.

## Performance/PM scheduler 전용 기준
Performance Service의 PM 수집 scheduler는 두 경로를 분리해서 분석한다.

- 실시간 장비 조회 수집: `PMStatisticsSchedulerJob`
- FTP 업로드 파일 파싱 수집: `NeStatisticsScheduleJob`

필수 확인 항목:

- scheduler fire time과 interval
- NE별 중복 실행 방지 상태
- queue와 thread pool backpressure
- product별 PM function/parser 등록 상태
- Mongo `insertMany(...)` 또는 MyBatis `loadPerformance`, `syncListNeStatistics` 저장 경로
- 파일 success/fail/raw/parsed 이동 경로
- PM memory state와 15분 집계 key

상세 기준은 `<HARNESS_ROOT>/packs/nms-ems/knowledge/backend/services/performance.md`를 따른다.
