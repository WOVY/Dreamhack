# 📂 Case Analysis: BypassIF

## 1. 문제 정보 (Challenge Info)
- **Description**: Flask 기반 웹 서비스에서 명령어 필터(if 조건)를 우회해 Admin KEY를 탈취하고 플래그를 획득하는 문제입니다.
- **Target**: 드림핵 워게임 서버 (BypassIF)
- **Flag Format**: `DH{...}`

## 2. 분석 개요 (Overview)
- **Objective**: `filter_cmd`의 반환값 구조와 subprocess 타임아웃 예외를 이용해 Admin KEY를 노출시킨 뒤, 2단계 요청으로 플래그 획득.
- **Key Concept**: **OS Command Injection / 예외 핸들러 정보 노출 (Information Disclosure via Exception Handler)**

## 3. 분석 및 해결 단계 (Steps)

### Step 1: 소스코드 분석

서버의 핵심 로직입니다.

```python
KEY = hashlib.md5(FLAG.encode()).hexdigest()
guest_key = hashlib.md5(b"guest").hexdigest()

def filter_cmd(cmd):
    command_list = ['flag','cat','chmod','head','tail','less','awk','more','grep']
    alphabet = list(string.ascii_lowercase) + [' '] + list('0123456789')

    for c in command_list:
        if c in cmd:
            return True          # 위험 → 차단
    for c in cmd:
        if c not in alphabet:
            return True          # 허용 안된 문자 → 차단
    # 안전한 경우 return 없음 → None (falsy)

@app.route('/flag', methods=['POST'])
def flag():
    key = request.form.get('key', '')
    cmd = request.form.get('cmd_input', '')

    if cmd == '' and key == KEY:
        return render_template('flag.html', txt=FLAG)   # 플래그 반환
    elif cmd == '' and key == guest_key:
        return render_template('guest.html', ...)

    if cmd != '' or key == KEY:                         # cmd가 있으면 무조건 진입
        if not filter_cmd(cmd):                         # filter가 None이면 실행
            try:
                output = subprocess.check_output(['/bin/sh', '-c', cmd], timeout=5)
                return render_template('flag.html', txt=output.decode('utf-8'))
            except subprocess.TimeoutExpired:
                return render_template('flag.html', txt=f'Timeout! Your key: {KEY}')
```

### Step 2: Root Cause 파악

두 결함이 결합되어 있습니다.

| # | 결함 | 설명 |
|---|---|---|
| 1 | **불완전한 블랙리스트 필터** | `sleep`은 차단 목록에 없고 소문자+공백+숫자만으로 구성되어 필터를 통과함 (`filter_cmd` → `None`) |
| 2 | **예외 핸들러 정보 노출** | `TimeoutExpired` 발생 시 `KEY`를 응답에 직접 포함 |

**필터 로직의 핵심**: `filter_cmd`는 위험할 때만 `True`를 반환하고, 안전하면 아무것도 반환하지 않아 `None`이 됩니다. 서버는 `if not filter_cmd(cmd)` — 즉 필터가 falsy일 때만 명령을 실행합니다. `sleep 10`은 이 조건을 통과합니다.

**접근 조건의 허점**: `cmd != ''`만 충족하면 key 검증 없이 명령 실행 경로에 진입할 수 있습니다.

### Step 3: 익스플로잇 1단계 — 타임아웃으로 KEY 획득

초기 페이지(`/`)에는 key 입력란만 존재합니다. 임의의 값 `HI`를 입력하고 Burp Suite로 패킷을 가로챈 뒤, 요청 본문에 `cmd_input` 파라미터를 직접 추가합니다.

```
POST /flag
key=HI&cmd_input=sleep 10
```

- `cmd != ''` → if 진입
- `filter_cmd("sleep 10")` → `None` (sleep은 차단 목록 없음, 허용 문자만 사용) → `if not None` = True → 명령 실행
- 5초 후 `TimeoutExpired` 발생 → 응답:

```
Timeout! Your key: 409ac0d96943d3da52f176ae9ff2b974
```

### Step 4: 익스플로잇 2단계 — KEY로 플래그 획득

KEY 획득 후 표시되는 페이지에는 cmd 입력란만 존재합니다. 마찬가지로 Burp Suite로 패킷을 가로채 `key` 파라미터를 직접 추가합니다.

```
POST /flag
key=409ac0d96943d3da52f176ae9ff2b974&cmd_input=
```

- `cmd == '' and key == KEY` → True → FLAG 반환

## 4. 결과 (Result)

- **Flag**: `DH{9c73a5122e06f1f7a245dbe628ba96ba68a8317de36dba45f1373a3b9a631b92}`

## 5. 보안 인사이트 (Retrospective)
- **Root Cause**: 블랙리스트 기반 명령어 필터는 `sleep`처럼 직접 위험하지 않은 명령을 막지 못하며, `TimeoutExpired` 예외 핸들러가 내부 민감 값(`KEY`)을 응답에 포함시켜 정보가 노출됨. 또한 `cmd != ''` 조건만으로 명령 실행 경로에 진입 가능해 KEY 검증이 무의미해짐.
- **Countermeasures**:
  - **화이트리스트 방식 적용**: 허용할 명령어를 명시적으로 정의하고 그 외는 전부 차단. 블랙리스트는 우회 경로가 항상 존재함.
  - **예외 핸들러 정보 노출 제거**: 예외 발생 시 민감 값을 절대 응답에 포함하지 않고 일반적인 오류 메시지만 반환.
  - **접근 제어 로직 통합**: `cmd != ''` 단독으로 실행 경로에 진입하는 구조 대신, key 검증과 cmd 검증을 함께 요구하도록 조건문 재설계.
  - **subprocess 사용 자제**: 사용자 입력을 셸 명령으로 실행하는 구조 자체를 피하고 필요한 기능은 전용 라이브러리로 구현.
