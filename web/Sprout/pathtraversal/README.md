# 📂 Case Analysis: pathtraversal

## 1. 문제 정보 (Challenge Info)
- **Description**: 사용자 정보를 조회하는 기능에서 발생하는 API Path Traversal 취약점을 이용하여, 외부에서 직접 접근이 차단된 내부 전용 플래그 API 엔드포인트를 우회 호출해 플래그를 탈취하는 문제입니다.
- **Target**: 드림핵 워게임 서버 (pathtraversal)
- **Flag Format**: `DH{...}`

## 2. 분석 개요 (Overview)
- **Objective**: `/get_info` 엔드포인트의 `userid` POST 파라미터에 `../flag`를 주입하여, 서버가 내부적으로 구성하는 API 요청 경로를 `/api/user/../flag` → `/api/flag`로 정규화시켜 플래그를 응답받음.
- **Key Concept**: **API Path Traversal** (URL Path Injection), Internal API Access via SSRF-like Proxy

## 3. 분석 및 해결 단계 (Steps)

### Step 1: 취약점 분석 (Root Cause)
Flask 앱의 `/get_info` 라우트는 사용자로부터 `userid`를 POST로 입력받아, 서버 내부에서 아래와 같이 직접 URL에 삽입하여 내부 API 서버(포트 8000)로 재요청합니다.

```python
# 취약한 코드
userid = request.form.get('userid', '')
info = requests.get(f'{API_HOST}/api/user/{userid}').text
```

내부 API 서버에는 두 가지 핵심 엔드포인트가 존재합니다.

```
GET /api/user/<uid>   →  users 딕셔너리에서 해당 uid의 정보 반환
GET /api/flag         →  서버에 저장된 FLAG 값 반환
```

두 엔드포인트 모두 `@internal_api` 데코레이터로 보호되어 `127.0.0.1`에서의 요청만 허용합니다. 그런데 `/get_info`가 서버 자신(`127.0.0.1`)을 대신해 내부 API를 호출하므로, 이 내부망 접근 제한은 실질적으로 우회됩니다.

- **취약점**: `userid` 파라미터를 URL 경로에 그대로 삽입하면서 `../` 시퀀스에 대한 검증이 전혀 없음. 사용자가 `../flag`를 입력하면 서버는 `/api/user/../flag`를 요청하고, 이는 HTTP 경로 정규화에 의해 `/api/flag`로 해석됨.

### Step 2: 페이로드 구성 (Payload Construction)
클라이언트는 `userid` 값을 POST body에 담아 Flask 서버로 전송합니다. Flask는 이 값을 꺼내 **서버 내부에서** 내부 API로 보내는 GET 요청의 URL 경로에 직접 삽입합니다.

**정상 요청 흐름**:
```
클라이언트 → Flask (POST /get_info, body: userid=0)
Flask      → 내부 API (GET http://127.0.0.1:8000/api/user/0)  →  guest 정보 반환
```

**페이로드 주입 시 흐름**:
```
클라이언트 → Flask (POST /get_info, body: userid=../flag)
Flask      → 내부 API (GET http://127.0.0.1:8000/api/user/../flag)
                                           ↓ (경로 정규화)
                       GET http://127.0.0.1:8000/api/flag      →  FLAG 반환
```

### Step 3: 공격 실행 (Exploit)
1. Burp Suite로 `/get_info` 페이지의 POST 요청을 가로챔.
2. 요청 바디의 `userid` 값을 아래 페이로드로 교체하여 전송.

**Payload**:
```
userid=../flag
```

3. 서버가 내부적으로 `/api/flag` 엔드포인트를 호출하여 응답에 플래그를 반환함.

## 4. 결과 (Result)
`userid` 파라미터가 URL 경로에 무검증으로 삽입되는 구조를 파악하고, Burp Suite로 요청을 가로채 페이로드를 변조하여 내부 전용 플래그 API 호출에 성공하였습니다.

- **Flag**: `DH{8a33bb6fe0a37522bdc8adb65116b2d4}`

## 5. 보안 인사이트 (Retrospective)
- **Root Cause**: 사용자 입력값(`userid`)을 서버 내부 API 호출 URL의 경로에 직접 삽입하면서, 경로 탐색 메타문자(`../`)에 대한 필터링 및 검증 로직이 누락됨. 서버가 내부 API 프록시 역할을 수행하는 구조상, 내부 네트워크 보호 장치(IP 기반 접근 제어)도 함께 우회됨.
- **Countermeasures**:
  - **화이트리스트 기반 입력 검증**: `userid`는 숫자 등 사전에 정의된 형식만 허용하고, `../`, `/`, `.` 등 URL 경로 조작에 사용되는 문자를 원천 차단.
  - **파라미터와 경로의 분리**: 사용자 입력을 URL 경로에 직접 삽입하는 방식 대신, 서버 내부에서 입력값을 키로 하여 허용된 리소스 목록을 조회하는 간접 참조(Indirect Reference) 방식으로 설계.
  - **내부 API 인증 강화**: IP 주소 기반의 단순 접근 제어 외에, API 키나 서명 토큰 등 추가적인 인증 수단을 도입하여 프록시 구조에서의 우회를 차단.
