# NMS/EMS 용어집

## 핵심 용어
- NE: EMS/NMS가 관리하는 Network Element.
- Site: NE와 관련 topology를 묶는 논리적 또는 물리적 위치 단위.
- Shelf: NE 내부의 hardware 또는 logical equipment container.
- Port: 장비의 물리적 또는 논리적 endpoint.
- Alarm: 운영자가 확인해야 하는 fault state 또는 event.
- PM: monitoring과 reporting을 위해 수집하는 Performance Management 데이터.
- Sync: EMS/NMS 상태를 network, device, backend 상태와 맞추는 과정.
- Session: auth token과 WebSocket mapping을 포함한 user 또는 system interaction context. REST(`access_token` 헤더)와 WebSocket(URL path의 session_id)이 같은 session_id를 공유한다. 연결 흐름은 `api/fe-be-connection.md` 참고.

## 작업 원칙
이 용어와 관련된 동작을 바꿀 때는 수정 전에 실제 read path, write path, active runtime branch, user-visible impact를 추적한다.
