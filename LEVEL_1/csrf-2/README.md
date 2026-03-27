# 📂 Case Analysis: csrf-2

## 1. 문제 정보 (Challenge Info)
- **Description**: 여러 기능과 입력받은 URL을 확인하는 봇이 구현된 서비스입니다. CSRF 취약점을 이용해 플래그를 획득하세요.
- **Target**: 드림핵 워게임 서버 (csrf-2)
- **Flag Format**: `DH{...}`

## 2. 분석 개요 (Overview)
- **Objective**: 관리자가 공격자가 의도한 비밀번호 변경 URL을 방문하게 하여, 관리자 계정 권한을 획득함.
- **Key Concept**: **CSRF (Account Takeover)**, Improper Authentication

## 3. 분석 및 해결 단계 (Steps)

### Step 1: 서버 소스 코드 분석 (Root Cause)
- **Target Endpoint**: `/change_password`
  - 해당 기능은 현재 로그인한 세션의 비밀번호를 `pw` 파라미터 값으로 변경함.
  - **취약점**: 비밀번호 변경 시 **'기존 비밀번호(Old Password)'를 확인하지 않으며**, CSRF Token과 같은 검증 절차가 없음.
  - `GET` 요청으로도 상태 변경이 가능하도록 구현되어 있음.

### Step 2: 페이로드(Payload) 구성
관리자의 비밀번호를 공격자가 원하는 값으로 변경하는 구문을 작성하였습니다.
- **Payload**: `<img src="/change_password?pw=admin">`
- **동작 원리**: 
  1. 관리자 봇이 이 태그를 렌더링하면 `/change_password?pw=admin`으로 요청을 보냄.
  2. 서버는 관리자의 비밀번호를 `admin`으로 업데이트함.

### Step 3: 공격 실행 (Exploit) 및 플래그 획득
1. `/flag` 페이지(입력 취약점이 있는 곳)에 위 페이로드를 제출하여 관리자 봇의 실행을 유도함.
2. 공격 성공 시, 관리자의 비밀번호가 `admin`으로 변경됨.
3. 메인 페이지로 돌아와 `admin` / `admin`으로 **로그인**을 시도함.
4. 관리자 계정으로 로그인 성공 후, 화면에 출력된 플래그를 확인.


## 4. 결과 (Result)
- **Flag**: `DH{c57d0dc12bb9ff023faf9a0e2b49e470a77271ef}`

## 5. 보안 인사이트 (Retrospective)
- **Root Cause**: 민감한 정보(비밀번호) 변경 시 현재 비밀번호 검증 누락 및 CSRF 방어 기제 부재.
- **Countermeasures**: 
  - **재인증 절차**: 비밀번호 변경 전 기존 비밀번호를 입력받아 본인임을 재확인해야 함.
  - **CSRF Token**: 예측 불가능한 토큰을 사용하여 비정상적인 경로의 요청을 차단해야 함.
