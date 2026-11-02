# Six-Gen React Preference System Info Parity

## 활성 Angular 계약

- Route: `/preference/system-info`
- Provider: `PreferenceContentProviderModule`
- Component: `src/app/content/contents/preference/content/system-info/system-info.component.ts`
- Product source: `sites/six-gen-flexible/product.ts`
- Product: `6G Flexible EMS`
- Version: local `1.0.0`
- Logo: `assets/image/common/my5g_logo.png`, natural size `83x26`
- `isShowExpireTime=false`이므로 화면의 license card는 숨긴다.
- 활성 six-gen에서는 화면 자체 server-version 또는 license REST 요청을 보내지 않는다.

로그인 직후 발생하는 `get-license-expire-time`은 System Info 화면 요청이 아니라 공통 인증 startup 요청이다.

## 데스크톱 UI 계약

기준 viewport는 `1440x960`이다. 모바일은 현재 이식 범위 밖이다.

- 콘텐츠 시작점: `x=337`, `y=95`
- 카드: `1083x84`, 흰색, `1px #d9dadb`, radius `4px`
- Angular `wc-card`는 title이 비어 있어도 `showTitle=true` 기본값을 유지한다.
- 빈 title 영역은 `16px`, body는 `10px` padding이다.
- app 영역은 왼쪽 `10px` padding과 flex layout을 사용한다.
- 제품명은 `24px/700`, 버전은 `13px`, line-height는 `18.2px`이다.
- font stack은 Angular의 Segoe UI/Malgun Gothic 계열을 사용한다.

빈 title 영역을 제거하거나 React 기본 Lato를 사용하면 카드 높이와 제품명 폭이 Angular와 달라진다.

## React 구현 위치

- 화면: `src/screens/preference/PreferenceSystemInfoScreen.tsx`
- 스타일: `src/screens/preference/systemInfo.css`
- 화면 데이터: `src/screens/preference/systemInfoModel.ts`
- token: `src/styles.css`, `DESIGN.md`

## 검증

- fixture: `SIXGEN_REACT_URL=http://127.0.0.1:<port>/unms node scripts/qa-sixgen-preference-system-info.mjs`
- live parity: `npm run qa:six-gen-preference-system-info-live`
- evidence: `.codex/visual-qa/preference-system-info-live/`
- state, startup 순서, normalized license request, computed style, geometry, browser error를 비교한다.
- fresh content-area pixel ratio above RGB difference 16: `0.001269`
- fresh full-frame ratio above RGB difference 16: `0.008992`
