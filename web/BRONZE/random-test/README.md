# 📂 Case Analysis: random-test

## 1. 문제 정보 (Challenge Info)
- **Description**: 임의의 사물함 번호(4자리 소문자+숫자)와 자물쇠 비밀번호(100~200 사이 정수)가 서버에 생성됩니다. 두 값을 모두 맞춰 플래그를 획득하는 문제입니다.
- **Target**: 드림핵 워게임 서버 (random-test)
- **Flag Format**: `DH{...}`

## 2. 분석 개요 (Overview)
- **Objective**: 서버가 사물함 번호를 입력 길이만큼만 잘라 비교하는 취약점을 이용해 Burp Suite Intruder로 한 글자씩 열거하여 전체 번호를 복원하고, 이후 비밀번호를 브루트포스해 플래그를 획득.
- **Key Concept**: **부분 문자열 비교 취약점 (Partial String Comparison)** + Burp Suite Intruder 브루트포스

## 3. 분석 및 해결 단계 (Steps)

### Step 1: 취약점 분석 (Root Cause)
Flask 앱은 서버 시작 시 4자리 랜덤 문자열(`rand_str`)과 100~200 사이의 정수(`rand_num`)를 생성합니다. 로그인 검증 로직은 아래와 같습니다.

```python
# 취약한 코드
if rand_str[0:len(locker_num)] == locker_num:   # 입력 길이만큼만 비교
    if str(rand_num) == password:
        return FLAG
    return 'Good'   # 사물함 번호만 맞았을 때
return 'Wrong'
```

`rand_str[0:len(locker_num)]`는 입력값의 길이만큼만 앞에서 잘라 비교합니다. 즉, 1글자만 입력하면 첫 번째 글자만 비교되므로, 한 글자씩 순차적으로 시도하면 경우의 수를 줄일 수 있습니다.

### Step 2: 사물함 번호 열거 (Burp Suite Intruder)

Burp Suite Intruder를 Sniper 모드로 구성하고 `locker_num` 파라미터에 Payload Position을 설정합니다.

| 항목 | 설정 |
|---|---|
| Attack type | Sniper |
| Payload type | Simple list |
| Payload | a-z, 0-9 (36개) |
| Grep-Match | `Good` |

- **첫 번째 자리 탐색**: `locker_num` 값을 `§a§` 형태로 1글자 페이로드로 전송. 응답 길이가 다르거나 Grep-Match에 `Good`이 체크된 항목이 정답 글자.
- **이후 자리 탐색**: 앞서 찾은 글자를 접두사로 고정하고 다음 자리에 페이로드를 붙여 반복 (`§ab§` → `§abc§` → …).
- 4회 반복으로 전체 `rand_str` 복원.

> **팁**: Intruder Options → **Grep-Match**에 `Good`을 등록하면, 응답에 `Good`이 포함된 행에 표시가 생겨 길이 비교 없이 바로 정답 글자를 확인할 수 있습니다.

### Step 3: 자물쇠 비밀번호 브루트포스

사물함 번호를 확정한 뒤, `password` 파라미터에 **Numbers** 페이로드(100~200, 증감 1)를 설정해 Intruder를 재실행합니다.

| 항목 | 설정 |
|---|---|
| locker_num | Step 2에서 찾은 값 (고정) |
| Payload type | Numbers (From 100, To 200, Step 1) |
| Grep-Match | `DH{` |

응답에 `DH{`가 포함된 요청이 정답 비밀번호이며, 해당 응답 본문에 플래그가 출력됩니다.

## 4. 결과 (Result)
사물함 번호 부분 비교 취약점을 Burp Suite Intruder로 단계별 열거해 `rand_str`을 복원하고, 비밀번호를 100~200 범위에서 브루트포스하여 플래그를 획득하였습니다.

- **Flag**: `DH{2e583205a2555b8890d141b51cee41379cead9a65a957e72d4a99568c0a2f955}`

## 5. 보안 인사이트 (Retrospective)
- **Root Cause**: 사용자 입력값의 길이에 따라 비교 범위가 달라지는 `rand_str[:len(input)]` 구조로 인해, 완전한 값을 몰라도 한 글자씩 정답 여부를 확인할 수 있는 오라클(Oracle)이 형성됨.
- **Countermeasures**:
  - **전체 길이 검증**: 입력값 길이를 먼저 검사하여, 기대 길이(4자리)가 아니면 즉시 거부.
  - **실패 횟수 제한 (Rate Limiting)**: 동일 IP/세션에 대한 시도 횟수를 제한하고 실패 시 지연을 부여해 브루트포스를 실질적으로 차단.
  - **응답 균일화**: 성공/실패 응답의 크기와 구조를 동일하게 유지해 응답 길이 차이에 의한 정보 노출을 방지.
