# 📂 Case Analysis: command-injection-1

## 1. 문제 정보 (Challenge Info)
- **Description**: ping 테스트 기능 등에서 발생하는 OS Command Injection 취약점을 이용해 서버의 시스템 명령어를 실행하고 플래그를 탈취하는 문제입니다.
- **Target**: 드림핵 워게임 서버 (command-injection-1)
- **Flag Format**: `DH{...}`

## 2. 분석 개요 (Overview)
- **Objective**: 서버가 시스템 명령어(예: `ping`)를 호출할 때, 명령어 구분자(Command Separator)를 삽입하여 공격자가 원하는 임의의 명령어를 이어서 실행하게 만듦.
- **Key Concept**: **OS Command Injection**, Command Separator (`;`, `&&`, `||`), Shell Meta-characters

## 3. 분석 및 해결 단계 (Steps)

### Step 1: 소스 코드 및 취약점 분석 (Root Cause)
웹 서비스에서 사용자가 입력한 IP 주소를 받아 핑(ping)을 보내는 기능이 구현되어 있습니다.

> **서버 내부 동작**:
> os.system(f"ping -c 3 {host}")

- **취약점**: 사용자 입력값(`host`)에 대한 검증이나 필터링이 클라이언트 단에서 일어납니다. 따라서 개발자도구를 이용해 `pattern` 속성을 제거하여 정규표현식을 우회하여 원하는 명령어를 실행할 수 있습니다.

### Step 2: 페이로드 구성 (Payload Construction)
플래그가 담긴 파일(`flag.py`)의 내용을 읽어오기 위해 다음과 같이 페이로드를 구성합니다.

> **Payload**: `8.8.8.8"; ls #`
> **Payload**: `8.8.8.8"; cat flag.py #`

**[동작 원리]**
서버에서 완성되는 최종 명령어는 다음과 같이 해석됩니다.

> ping -c 3 "8.8.8.8"; ls #"
> ping -c 3 "8.8.8.8"; cat flag.py #"

1. 앞의 `ping` 명령어가 먼저 실행됩니다.
2. 세미콜론(`;`)에 의해 앞 명령어가 끝난 것으로 간주됩니다.
3. 뒤이어 `ls #`, `cat flag.py #` 명령어가 실행되면서 플래그 파일의 내용이 화면에 출력됩니다.

### Step 3: 공격 실행 (Exploit)
1. 문제 페이지의 대상 IP 입력란에 구성한 페이로드(`8.8.8.8"; ls #`)를 입력 후 실행합니다.
2. `ls` 명령어의 결과를 통해 flag 값이 어디에 위치해 있는지 파악합니다.
3. 문제 페이지의 대상 IP 입력란에 구성한 페이로드(`8.8.8.8"; cat flag.py`)를 입력 후 실행합니다.
4. `cat` 명령어에 의해 출력된 플래그 값을 확인합니다.

## 4. 결과 (Result)
명령어 주입을 통해 시스템 내부에 존재하는 플래그 파일을 성공적으로 읽어왔습니다.

- **Flag**: `DH{pingpingppppppppping!!}`

## 5. 보안 인사이트 (Retrospective)
- **Root Cause**: 외부 사용자 입력값을 셸 명령어의 인자로 직접 전달하고, 셸 메타문자에 대한 검증 로직이 부재함.
- **Countermeasures**:
  - **입력값 검증**: 서버 단에서 정규표현식 등을 사용하여 입력값이 정확한 IP 주소 형식인지(예: 숫자와 온점만 포함) 철저히 검증해야 함.
  - **안전한 API 사용**: `os.system()`처럼 셸을 직접 호출하는 함수 대신, 셸 기능을 거치지 않는 라이브러리(예: Python의 `subprocess` 모듈에서 `shell=False` 설정)를 사용하여 명령어 주입을 원천 차단해야 함.
