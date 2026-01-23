# 📂 Case Analysis: simple-sqli

## 1. 문제 정보 (Challenge Info)
- **Description**: 로그인 서비스입니다. SQL INJECTION 취약점을 통해 플래그를 획득하세요. 플래그는 flag.txt, FLAG 변수에 있습니다.
- **Target**: 드림핵 워게임 서버 (simple-sqli)
- **Flag Format**: `DH{...}`

## 2. 분석 개요 (Overview)
- **Objective**: 올바른 비밀번호를 모르더라도 SQL 구문을 조작하여 관리자 계정으로 인증을 우회(Authentication Bypass)함.
- **Key Concept**: **SQL Injection (SQLi)**

## 3. 분석 및 해결 단계 (Steps)

### Step 1: 서버 로직 추론 (Backend Logic)
로그인 기능의 서버 측 SQL 쿼리문은 다음과 같은 형태를 가집니다.

> SELECT * FROM users WHERE userid = "{userid}" AND userpassword = "{userpassword}";

- 사용자가 입력한 `userid`와 `userpassword`가 쿼리문에 문자열로 합쳐져서 실행됩니다.
- 입력값에 특수문자(`'`, `"`, `--`, `#` 등)에 대한 필터링이 없다면 쿼리의 구조를 변경할 수 있습니다.

### Step 2: 페이로드 구성 (Payload Construction)
비밀번호 검증 로직을 무력화하기 위해 다음과 같은 공격 구문을 작성하였습니다.
- **Input ID**: `admin" -- `
- **Input PW**: `1234` (임의의 값)

**[동작 원리 분석]**
위 입력을 넣었을 때 완성되는 SQL 쿼리는 다음과 같습니다.

> SELECT * FROM users WHERE userid = "admin" -- " AND userpassword = "1234";

1. `"admin"`: 아이디를 admin으로 지정합니다.
2. `"`: 아이디 문자열을 닫습니다.
3. `--`: **SQL 주석(Comment)** 문자로, 이 뒤에 오는 모든 명령어(비밀번호 검증 부분)를 실행하지 않고 무시합니다.
4. 결과적으로 DB는 `userid = "admin"` 조건만 참이면 로그인을 승인하게 됩니다.

### Step 3: 공격 실행 (Exploit)
1. 로그인 페이지의 **ID** 입력란에 `admin" --`를 입력합니다.
2. **Password** 입력란에 `1234` 등 아무 값이나 입력합니다.
3. 로그인 버튼을 누르면 비밀번호 검증 없이 관리자 계정으로 로그인이 성공합니다.

## 4. 결과 (Result)
로그인 성공 후 출력된 플래그를 획득하였습니다.

- **Flag**: `DH{c1126c8d35d8deaa39c5dd6fc8855ed0}`

## 5. 보안 인사이트 (Retrospective)
- **Root Cause**: 사용자 입력값을 SQL 쿼리에 단순히 문자열 연결(Concatenation) 방식으로 포함시켜, 입력값이 쿼리의 구조를 변경할 수 있게 됨.
- **Countermeasures**:
  - **Prepared Statement 사용**: 입력값을 쿼리 문법이 아닌 단순 데이터로만 처리하도록 바인딩(Binding)하여 근본적인 주입 공격을 차단해야 함.
  - **Input Validation**: 입력값에 특수문자나 SQL 예약어가 포함되어 있는지 검증해야 함.
