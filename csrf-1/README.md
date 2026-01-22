# 📂 Case Analysis: csrf-1

## 1. 문제 정보 (Challenge Info)
- **Description**: 여러 기능과 입력받은 URL을 확인하는 봇이 구현된 서비스입니다. CSRF 취약점을 이용해 플래그를 획득하세요.
- **Target**: 드림핵 워게임 서버 (csrf-1)
- **Flag Format**: `DH{...}`

## 2. 분석 개요 (Overview)
- **Objective**: 관리자 봇이 공격자가 의도한 URL로 요청을 보내도록 유도하여, 관리자 전용 기능을 강제로 실행시킴.
- **Key Concept**: **CSRF (Cross-Site Request Forgery)**, HTML Injection

## 3. 분석 환경 (Environment)
- **Tools**: Chrome DevTools, Python (Source Code Analysis)

## 4. 분석 및 해결 단계 (Steps)

### Step 1: 서버 소스 코드 분석 (White-box)
제공된 `app.py`를 분석하여 공격 대상을 식별하였습니다.
- **Target Endpoint**: `/admin/notice_flag`
  - `userid` 파라미터가 'admin'인 경우, 현재 메모(`memo_text`)에 플래그를 추가하는 로직이 존재함.
  - **접근 제어**: 로컬호스트(127.0.0.1)에서의 요청만 허용되므로, 외부에서 직접 접속은 불가능하며 봇(Admin Bot)을 통해서만 실행 가능함.
- **Vulnerability**: `/vuln` 페이지 등에서 사용자 입력값에 대한 검증이 미흡하여 HTML 태그 삽입이 가능하며, 요청 시 CSRF Token과 같은 검증 수단이 부재함.

### Step 2: 페이로드(Payload) 구성
관리자가 `/admin/notice_flag` URL을 호출하도록 만드는 HTML 구문을 작성하였습니다.
- **Payload**: `<img src="/admin/notice_flag?userid=admin" />`
- **동작 원리**: 
  1. `<img>` 태그는 `src` 속성의 주소로 `GET` 요청을 보냅니다.
  2. 관리자 봇이 이 태그를 렌더링하면, 봇의 브라우저는 자동으로 `/admin/notice_flag?userid=admin`에 접속을 시도합니다.
  3. 서버는 이를 관리자의 정당한 요청으로 인식하고 메모에 플래그를 기록합니다.

### Step 3: 공격 실행 (Exploit)
1. `/flag` 페이지에 위 `<img>` 태그 페이로드를 입력하고 제출합니다.
2. 관리자 봇이 해당 페이지를 방문하면서 CSRF 공격이 수행됩니다.
3. `/memo` 페이지로 이동하여 관리자가 강제로 기록한 플래그 값을 확인합니다.


## 5. 결과 (Result)

### Flag 획득 확인
- **Flag**: `DH{11a230801ad0b80d52b996cbe203e83d}`
- **확인 경로**: `/memo` 페이지 하단

## 6. 보안 인사이트 (Retrospective)
- **Root Cause**: 상태를 변경하는 중요 요청(플래그 조회 등)에 대해 요청자가 사용자가 의도한 것인지 검증하는 절차가 없음.
- **Countermeasures**: 
  - **CSRF Token 사용**: 임의의 난수(Token)를 세션과 폼에 부여하여, 요청 시 이 값이 일치하는지 검증해야 함.
  - **SameSite Cookie 설정**: 쿠키 속성을 `SameSite=Strict` 또는 `Lax`로 설정하여 타 사이트에서의 요청 시 쿠키 전송을 제한해야 함.
  - **Referer 검증**: 요청이 발생한 이전 페이지의 도메인을 확인하여 신뢰할 수 있는 경로인지 검사해야 함.
