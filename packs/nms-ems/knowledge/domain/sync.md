# Sync 도메인 노트

## 사용 시점
sync state, scheduler, queue, refresh, polling, retry, device/backend reconciliation 변경 시 이 문서를 사용한다.

## 읽기 경로
- 현재 sync state의 출처를 식별한다: backend API, base data loader, queue status, UI cache.
- 여러 site 또는 NE가 동시에 load되는지 확인한다.

## 쓰기 경로
- 무엇이 sync를 시작하고, 무엇이 completion을 표시하며, 무엇이 failure를 처리하는지 식별한다.
- retry logic에서는 timeout과 duplicate execution 동작을 식별한다.

## 위험
- duplicate sync는 device 또는 backend service에 과부하를 줄 수 있다.
- failed sync handling이 잘못되면 stale UI state가 남을 수 있다.
- queue backpressure는 alarm, PM, configuration visibility를 지연시킬 수 있다.

## 변경 전 확인
- flow가 one-shot, scheduled, user-triggered 중 무엇인지 확인한다.
- cancellation, timeout, empty result, retry behavior를 확인한다.
- 결과 보고에 data loss, duplicate execution, backpressure risk를 설명한다.
