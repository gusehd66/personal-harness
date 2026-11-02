---
name: manual-update
description: 명시적으로 요청된 NMS/EMS 운영 매뉴얼 Word 문서와 화면 캡처의 안전한 갱신을 수행할 때 사용한다.
---

# 적용 범위

기존 DOCX의 구조와 스타일을 보존하면서 변경된 기능 절과 필요한 화면 캡처만 갱신한다. 일반 문서 편집에는 사용하지 않는다.

## 먼저 읽을 문서

- `<HARNESS_ROOT>/core/playbooks/harness-routing.md`
- `<HARNESS_ROOT>/packs/nms-ems/PACK.md`
- `<HARNESS_ROOT>/packs/nms-ems/knowledge/manual/my5g-docx-structure.md`
- 변경 화면에 해당하는 Pack UI·component knowledge

## 절차

1. 변경 diff와 실제 UI flow를 확인해 영향을 받은 매뉴얼 절과 캡처 필요 여부를 판단한다.
2. 프로젝트 `.ai/manual/` mapping이 있으면 우선 사용하고, 없으면 DOCX heading과 Pack manual knowledge로 대상 절을 찾는다.
3. 원본을 보존한 복사본에서 대상 설명, step, 표, 이미지, caption만 최소 수정한다.
4. 화면 조사는 조회 범위로 수행하며 저장·적용·장비 명령·discovery·upgrade는 실행하지 않는다.
5. 가능하면 변경 페이지와 전후 페이지를 렌더링해 이미지, 표, page break, glyph를 확인한다.
6. `harness-verify`로 수정 절, 캡처, 렌더 결과, 미검증을 분리해 보고한다.

## 규칙

- 운영 credential, 운영 DB 실데이터, 실제 장비 상태를 캡처나 실행 기록에 남기지 않는다.
- 기존 heading, 표, caption, 이미지 배치를 우선 보존한다.
- 원본 DOCX 덮어쓰기, 임시 캡처 삭제, 매핑 파일 추가는 사용자 요청과 프로젝트 규칙을 따른다.
