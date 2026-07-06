# Re:balance — 웹캠 자세 교정 서비스 백엔드

로그인 전에 측정한 데이터를 버리지 않기 위한 세션 설계

## 개요

Re:balance는 웹캠으로 목·어깨 자세를 인식해 점수를 매기고, 스트레칭과 AI 리포트를 제공하는 서비스다. 나는 Express 백엔드를 맡아 데이터 모델, 세션, 인증, API를 설계·구현했다.

| 항목 | 내용 |
| --- | --- |
| 기간 | 2026.04 ~ 2026.06 (커밋 기준) |
| 인원 | 5명 |
| 역할 | 백엔드 전체 — 데이터 모델(5 테이블), 세션 설계, 인증, API |
| 기술 스택 | Node.js, Express 5.2.1, Sequelize 6.37.8, MySQL2 3.20.0, bcrypt 6.0.0, jsonwebtoken 9.0.3, Passport (Google·Kakao OAuth), Google Generative AI (Gemini 1.5 flash) |
| 레포 | https://github.com/2026-SMHRD-KDT-HealthCare-4/balance |

자세 인식(MediaPipe 포즈 추정, 각도 계산)은 프론트엔드에서 다른 팀원이 담당했다. 나는 그 결과로 넘어온 각도·점수를 받아 저장·집계·서빙하는 백엔드를 맡았다.

---

## 로그인 전에 측정한 데이터를 어디에 둘 것인가

### 문제

Re:balance의 핵심 기능은 웹캠 자세 측정이다. 사용자는 회원가입 전에 바로 측정을 해볼 수 있어야 이탈이 줄어든다. 그런데 측정 데이터(각도·점수)는 세션에 묶여 DB에 쌓인다. 여기서 문제가 생긴다. 아직 로그인하지 않은 사용자의 측정 데이터를 어떤 주인에 매달아 저장할 것인가. 그리고 그 사용자가 나중에 가입하면, 가입 전에 측정한 데이터는 버릴 것인가 이어 붙일 것인가.

### 판단

두 방향을 놓고 비교했다.

**A. 측정하려면 먼저 로그인하게 한다**
- 얻는 것: 모든 세션이 처음부터 주인(user_id)을 가진다. 데이터 모델이 단순하고, 주인 없는 데이터를 정리할 일도 없다.
- 잃는 것: 핵심 기능을 써보기도 전에 가입을 강요한다. "일단 한번 재보고 싶다"는 사용자를 문 앞에서 돌려보내게 된다.

**B. 비회원도 측정하게 하고, 가입 시점에 데이터를 회원에게 연결한다**
- 얻는 것: 가입 없이 바로 측정을 써볼 수 있다. 측정 후 가입하면 방금 잰 데이터가 그대로 계정에 남는다.
- 잃는 것: 주인 없는 세션(user_id = null)을 허용해야 한다. 그러면 끝내 가입하지 않은 사용자의 세션이 DB에 계속 쌓이므로, 이걸 정리하는 장치가 따로 필요하다.

이탈을 막는 게 서비스의 핵심 기능과 직결된다고 봤다. B를 택하되, B의 단점인 "주인 없는 데이터가 쌓이는 문제"를 세 가지 장치로 막기로 했다.

첫 번째, 세션 소유를 선택적으로 뒀다. `Session` 테이블의 `user_id`를 nullable로 두고, 비회원 측정은 `user_id = null` 세션에 저장한다. 유저가 삭제돼도 세션이 통째로 날아가지 않도록 외래키를 `onDelete: SET NULL`로 걸었다.

두 번째, 가입 시점에 데이터를 넘겨받게 했다. 비회원이 가입하면 `linkSessionToUser`가 그 임시 세션(`user_id = null`)을 새 회원 id로 갱신한다. 가입 전 측정 데이터가 유실 없이 계정에 붙는다. 이 호출은 `auth.controller`의 가입 처리에 연결돼 있다.

세 번째, 끝내 가입하지 않은 데이터는 주기적으로 지운다. `deleteAbandonedData`가 `user_id = null`이면서 생성된 지 1시간이 지난 세션을 삭제한다. 서버 기동 1분 뒤 첫 실행, 이후 1시간 간격으로 돈다. 방치된 비회원 데이터가 DB에 무한정 쌓이지 않는다.

### 결과

비회원이 가입 없이 측정하고, 가입하면 그 데이터가 계정에 이어 붙고, 끝내 가입하지 않으면 1시간 뒤 정리되는 흐름이 백엔드에서 닫혔다. 관련 로직은 `services/posture.service.js`(`save`·`linkSessionToUser`·`deleteAbandonedData`)와 `app.js`의 스케줄러에 있다.

배운 점: 사용자 편의(가입 전 측정)를 위해 데이터 모델에 예외(주인 없는 세션)를 들이면, 그 예외를 되돌리는 장치(연결)와 정리하는 장치(GC)까지 함께 설계해야 구조가 무너지지 않는다.

---

## 그 밖의 백엔드 작업

- 인증: bcrypt 비밀번호 해싱과 JWT 발급에 더해, Passport로 Google·Kakao 소셜 로그인을 붙였다. (`config/passport.js`, `services/auth.service.js`)
- AI 리포트: 저장된 각도·점수를 Gemini(gemini-1.5-flash)에 넘겨 자세 조언 텍스트를 생성한다. `AiReport` 테이블의 `posture_id`를 nullable로 둬, 특정 시점 리포트와 기간(daily·weekly) 리포트를 한 테이블로 처리한다.
- API: 6개 라우트 그룹(auth·sessions·posture·stretching·stats·admin)을 컨트롤러·서비스·모델 계층으로 나눠 구성했다.
- 데이터 모델: User·Session·PostureData·StretchingLog·AiReport 5개 테이블과 연관관계를 Sequelize로 정의했다.

---

## 내 담당 범위

레포: https://github.com/2026-SMHRD-KDT-HealthCare-4/balance (브랜치: `final`, `jae-ryang`)

**내가 담당:** Express 백엔드 전체 — 데이터 모델 5개, 세션 설계(비회원 세션·연결·GC), 인증(로컬 + Google·Kakao 소셜 로그인·JWT), AI 리포트 연동, 6개 API 라우트. 백엔드 소스 전 파일에서 내가 주 커밋 저자다.

**팀원 담당:** 프론트엔드 자세 인식(MediaPipe 포즈 추정·각도 계산), 백엔드 일부 보조 커밋(Bogyeong-Kim).
