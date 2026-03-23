# 📂 Case Analysis: Mango

## 1. 문제 정보 (Challenge Info)
- **Description**: MongoDB 필터링 로직을 우회하여 관리자 계정의 비밀번호를 탈취하는 NoSQL Injection 문제입니다.
- **Target**: 드림핵 워게임 서버 (Mango)
- **Flag Format**: `DH{32alphanumeric}`

## 2. 분석 개요 (Overview)
- **Objective**: 특정 키워드(`admin`, `dh`)를 차단하는 필터링 기능을 정규표현식으로 우회하고, Blind NoSQL Injection으로 플래그를 추출함.
- **Key Concept**: **NoSQL Injection**, Filter Bypass (Regex), Blind Exploitation

## 3. 분석 및 해결 단계 (Steps)

### Step 1: 소스 코드 및 필터링 로직 분석
서버 코드에는 다음과 같은 취약한 필터링 로직이 존재합니다.

**BAN List**: `['admin', 'dh', 'admi']`

**Filter Function**: 
`const dump = JSON.stringify(data).toLowerCase();`, `if(dump.indexOf(word)!=-1) flag = true;`

- **취약점**: 입력값(`req.query`) 전체를 문자열로 변환하여 금지어가 포함되었는지 검사하지만, MongoDB의 정규표현식(`$regex`) 연산자를 사용하면 금지어를 쓰지 않고 필터링 규칙을 우회할 수 있습니다.

### Step 2: 필터 우회 전략 (Filter Bypass)
금지어에 걸리지 않으면서 관리자(`admin`)와 플래그 시작점(`DH{`)을 찾기 위한 정규표현식을 구성합니다.

- **uid (admin) 우회**: `admin`, `admi`이 금지어이므로, 와일드카드나 범위를 활용합니다.
> `uid[$regex]=^ad`
- **upw (DH{) 우회**: `dh`가 금지어이므로, 대소문자 구분이나 와일드카드를 활용합니다.
> `upw[$regex]=^D.{`

### Step 3: Blind NoSQL Injection 공격 (Exploit)
필터를 우회하는 동시에 비밀번호를 한 글자씩 알아내기 위한 자동화 로직을 설계합니다.

> **최종 페이로드 구조**: 
> `?uid[$regex]=^ad&upw[$regex]=^D.{32alphanumeric}`

1. **길이 확인**: 문제에서 32자리 alphanumeric임이 명시되어 있습니다. 만약 명시되지 않았을 경우 `....`을 이용해 길이를 파악합니다.
2. **문자 대조**: STRING_SET`(string.digits + string.ascii_letters)`을 순회하며 필터를 우회하는 정규표현식으로 요청을 보냅니다.
3. **조건부 응답**: 관리자 계정의 `uid`인 "admin"이 응답으로 돌아오면 해당 문자가 일치하는 것으로 판단합니다.

### Step 4: 공격 실행
Python 스크립트를 사용하여 `dh`를 직접 포함하지 않는 방식으로 `upw`의 나머지 32자리를 추출합니다.

> **스크립트**:
> ```python
>  for i in range(32):
>      for ch in STRING_SET:
>          response = requests.get(f'{HOST}/login?uid[$regex]=^ad&upw[$regex]=^D.{{{flag}{ch}')
>    # 응답에 "admin"이 있으면 flag 변수에 char 추가

## 4. 결과 (Result)
필터링을 우회하여 관리자 비밀번호 32자리를 모두 알아내고 로그인에 성공하였습니다.

- **Flag**: `FLAG: DH{89e50fa6fafe2604e33c0ba05843d3df}`

## 5. 보안 인사이트 (Retrospective)
- **Root Cause**: 블랙리스트 방식의 필터링은 정규표현식과 같은 특수 연산자를 통한 우회에 매우 취약함.
- **Countermeasures**:
  - **Type Checking**: `req.query` 값이 객체가 아닌 문자열인지 반드시 검사하여 `$regex` 등의 연산자 주입을 차단해야 함.
  - **White-list**: 금지어를 설정하기보다 허용된 패턴만 통과시키는 화이트리스트 방식 도입.
