# 📂 Case Analysis: web-ssrf

## 1. 문제 정보 (Challenge Info)
- **Description**: Image Viewer 서비스의 SSRF 취약점을 이용하여, 내부망(localhost) 필터링을 우회하고 랜덤 포트에서 동작 중인 내부 서버에 접근해 플래그를 탈취하는 문제입니다.
- **Target**: 드림핵 워게임 서버 (web-ssrf)
- **Flag Format**: `DH{...}`

## 2. 분석 개요 (Overview)
- **Objective**: URL 입력값에 대한 `localhost` 및 `127.0.0.1` 문자열 필터링을 우회하고, 1500~1800 사이의 랜덤 포트를 알아낸 뒤 Burp Suite를 활용해 응답 패킷에서 플래그를 추출함.
- **Key Concept**: **SSRF (Server-Side Request Forgery)**, Black-list Filter Bypass, Port Scanning, Packet Interception

## 3. 분석 및 해결 단계 (Steps)

### Step 1: 취약점 및 필터링 분석 (Root Cause)
서버는 사용자가 입력한 URL로 대신 HTTP 요청을 보내고, 그 결과를 Base64로 인코딩하여 화면의 `<img>` 태그에 렌더링해 줍니다.

- **취약점**: 서버 로직을 보면 요청 호스트에 `localhost`나 `127.0.0.1`이 포함되어 있으면 내부망 접근을 막습니다. 하지만 단순 문자열 비교만 수행하므로 대소문자 혼용 등 대체 표현법에 대한 방어가 없습니다.

### Step 2: 포트 스캐닝 및 필터 우회 (Port Scanning & Bypass)
내부 서버(1500~1800 포트)에 도달하기 위해 필터링을 우회하는 주소를 구성하고, 파이썬 스크립트 등을 통해 실제 열려있는 포트 번호를 찾아냅니다.

**Payload (필터 우회 및 포트 확인)**: 
`http://Localhost:1681/flag.txt` (`Localhost` 대소문자 우회 적용, 스캔으로 찾은 포트가 1681인 경우)

### Step 3: 패킷 분석을 통한 플래그 추출 (Exploit with Burp Suite)
포트 번호를 알아낸 후, 웹사이트의 URL 입력란에 직접 페이로드를 입력하여 요청을 보냅니다.

1. 웹 페이지에 페이로드(`http://Localhost:1681/flag.txt`)를 직접 입력 후 실행.
2. 서버가 내부의 `flag.txt` 텍스트 데이터를 읽어와 Base64로 인코딩한 뒤 `<img>` 태그의 `src` 속성에 넣어서 반환함.
3. 응답 데이터가 정상적인 이미지가 아닌 텍스트 데이터이므로 브라우저 화면에는 **'깨진 이미지'**로 출력됨.
4. **Burp Suite**를 사용하여 서버의 응답 패킷(Response)을 가로채고, HTML 소스코드 내 `<img>` 태그에 삽입된 Base64 문자열을 직접 추출.
5. 추출한 Base64 문자열을 디코딩하여 플래그 확인.

## 4. 결과 (Result)
단순 문자열 기반의 내부망 필터링을 우회하고, Burp Suite를 통해 이미지 렌더링 오류 뒤에 숨겨진 응답 데이터를 Base64로 디코딩하여 플래그를 성공적으로 탈취하였습니다.

- **Flag**: `DH{43dd2189056475a7f3bd11456a17ad71}`

## 5. 보안 인사이트 (Retrospective)
- **Root Cause**: 외부 서버가 사용자를 대신해 요청을 보내주는 기능(SSRF)에서, 목적지(Host)에 대한 검증이 허술하며 응답 데이터가 렌더링되는 과정에서 원본 데이터 노출 위험이 존재함.
- **Countermeasures**:
  - **IP 정규화 및 화이트리스트**: 입력받은 도메인이나 IP를 시스템에서 일관된 포맷으로 변환한 뒤, 로컬 루프백(127.x.x.x)이나 사설망 대역인지 엄격하게 검사하여 차단.
  - **응답 데이터 검증**: 외부로 요청을 보내 얻은 응답이 실제로 예상하는 콘텐츠 타입(예: `image/jpeg`, `image/png`)인지 철저히 검증하고, 텍스트나 알 수 없는 포맷일 경우 사용자에게의 출력을 차단.
