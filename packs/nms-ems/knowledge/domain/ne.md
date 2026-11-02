# NE 도메인 노트

## 사용 시점
Network Element 표시, 선택, 설정, topology membership, base data loading 변경 시 이 문서를 사용한다.

## 읽기 경로
- 새 fetch path를 추가하기 전에 기존 FE base data service와 cache를 확인한다.
- 화면이 generic module을 사용한다고 가정하기 전에 active site configuration과 provider wiring을 확인한다.

## 쓰기 경로
- 변경이 UI state만 갱신하는지, backend API를 호출하는지 식별한다.
- backend write라면 request payload, response handling, refresh timing을 확인한다.

## 위험
- 잘못된 NE mapping은 topology, equipment, alarm, PM, sync 화면에 영향을 줄 수 있다.
- multi-site filtering 실수는 잘못된 NE set을 노출하거나 숨길 수 있다.

## 변경 전 확인
- active route와 provider를 확인한다.
- 같은 NE object가 다른 화면에서 재사용되는지 확인한다.
- 기존 null/empty handling과 user authority filtering을 유지한다.
