# my5G/uNMS 운영 가이드 DOCX 구조 참고

## 기준 문서
- 파일: `<MANUAL_DOCX>`
- 크기: 약 32MB
- 이미지: `word/media/`에 약 399개 포함
- 성격: 스크린샷 중심 NMS 운영 가이드

## 제목 스타일
기존 문서는 Word 기본 `Heading 1/2/3` 스타일을 사용하지 않는다. 절 탐색 시 아래 커스텀 스타일을 우선 사용한다.

- `UEMS1`: 대제목
- `UEMS11`: 중제목
- `uEMS3`: 소제목
- `uEMS4`: 세부 제목

## 주요 본문/표 스타일
- `HFR`: HFR 본문
- `HFR0`: HFR 국문 본문
- `HFR2`: 본문HFR
- `affff0`: 본문내용
- `afffd`: 표본문_수정
- `afffe`: 표본문_타이틀
- `aff8`: 문서제목
- `13`, `23`, `31`, `42`: TOC 계열 스타일

## 절 탐색 기준
1. `word/document.xml`에서 paragraph style이 `UEMS1`, `UEMS11`, `uEMS3`, `uEMS4`인 문단을 추출한다.
2. heading text를 기능명, 메뉴명, route, component명과 매칭한다.
3. 같은 heading text가 여러 번 나오면 상위 heading chain을 함께 확인한다.
4. 대상 절은 해당 heading부터 다음 같은 레벨 또는 상위 레벨 heading 직전까지로 본다.
5. 이미지 교체는 relationship id나 `word/media/imageN.png` 번호보다 절 내부의 이미지 위치와 주변 caption/context를 기준으로 판단한다.

## 확인된 heading 예시
- `uNMS 개요`
- `Login`
- `Monitoring`
- `Main Dashboard`
- `Topology`
- `Active Alarm`
- `NE Dashboard (NE 별 Monitoring)`
- `Performance`
- `구성 관리`
- `그룹 및 NE 관리`
- `NE 관리`
- `NE 제어 관리`
- `NE Configuration 관리`
- `Configuration (O-CU/DU)`
- `Configuration (FHM: FrontHaulMux)`

## DOCX 갱신 주의점
- 기존 매뉴얼은 이미지가 많으므로 전체 재작성보다 대상 절의 local edit가 안전하다.
- 목차는 Word field일 수 있으므로 OOXML 수정만으로 페이지 번호가 즉시 갱신되지 않을 수 있다.
- 이미지가 바뀌면 캡션, 본문 step 번호, 전후 문단 spacing을 함께 확인한다.
- 렌더 검증 시 변경 페이지뿐 아니라 앞뒤 페이지도 확인한다.
