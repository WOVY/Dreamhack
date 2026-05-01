# 📂 Case Analysis: mongoboard

## 1. 문제 정보 (Challenge Info)
- **Description**: Node.js + MongoDB 기반 게시판 서비스에서 관리자의 비밀 게시글 ID를 예측하여 플래그를 탈취하는 문제입니다.
- **Target**: 드림핵 워게임 서버 (mongoboard)
- **Flag Format**: `DH{...}`

## 2. 분석 개요 (Overview)
- **Objective**: 공개 게시글들의 MongoDB ObjectID를 분석해 관리자의 숨겨진 비밀 게시글 ObjectID를 역산하고, 직접 조회하여 플래그를 획득.
- **Key Concept**: **MongoDB ObjectID 예측 (IDOR via Predictable ObjectID)**

## 3. 분석 및 해결 단계 (Steps)

### Step 1: 게시판 목록 확인

서버에 접속하면 아래와 같이 게시글 목록이 출력됩니다.

```
no                          Title   Author  publish_date
69f461f0489c51d3cb8b7c35   Hello   guest   2026-05-01T08:18:56.819Z
69f461f5489c51d3cb8b7c36   Mongo   guest   2026-05-01T08:19:01.829Z
(숨김)                      FLAG    admin   2026-05-01T08:19:06.831Z
69f461fe489c51d3cb8b7c38   Good    guest   2026-05-01T08:19:10.835Z
```

관리자(admin)가 작성한 FLAG 게시글의 `no`(ObjectID)만 숨겨져 있습니다. 그러나 나머지 세 게시글의 ObjectID는 공개되어 있습니다.

### Step 2: ObjectID 구조 분석 (Root Cause)

MongoDB ObjectID는 24자리 hex 문자열이며 세 필드로 구성됩니다.

```
[ 8 chars ] [ 10 chars ] [ 6 chars ]
  timestamp   machine+pid   counter
```

| 필드 | 크기 | 특성 |
|---|---|---|
| Timestamp | 8 hex (4 bytes) | 문서 생성 Unix 시각 (초 단위) |
| Machine+PID | 10 hex (5 bytes) | 같은 서버 프로세스 내 **항상 동일** |
| Counter | 6 hex (3 bytes) | 문서 생성 순서대로 **1씩 증가** |

공개된 세 ObjectID를 분해하면 다음과 같습니다.

| ObjectID | timestamp | machine+pid | counter |
|---|---|---|---|
| `69f461f0489c51d3cb8b7c35` | `69f461f0` | `489c51d3cb` | `8b7c35` |
| `69f461f5489c51d3cb8b7c36` | `69f461f5` | `489c51d3cb` | `8b7c36` |
| `69f461fe489c51d3cb8b7c38` | `69f461fe` | `489c51d3cb` | `8b7c38` |

- **Machine+PID** `489c51d3cb` 가 세 ID 모두 동일 → 서버 고정값 확인
- **Counter** 가 `8b7c35` → `8b7c36` → (?) → `8b7c38` 순으로 1씩 증가 → 빠진 값은 **`8b7c37`**

### Step 3: FLAG 게시글 ObjectID 역산

두 가지 필드를 결정합니다.

**① Timestamp 계산**

publish_date를 보면 Mongo(`08:19:01`)와 FLAG(`08:19:06`)는 **5초 차이**입니다.
ObjectID의 timestamp 부분에 그대로 5를 더하면 됩니다.

```
69f461f5 + 5 = 69f461fa
```

**② Counter**

앞뒤 게시글의 카운터가 `8b7c36`과 `8b7c38`이므로 그 사이 값은 **`8b7c37`**

**③ 최종 ObjectID 조합**

```
69f461fa  +  489c51d3cb  +  8b7c37
= 69f461fa489c51d3cb8b7c37
```

### Step 4: 비밀 게시글 직접 조회

역산한 ObjectID로 게시글 상세 엔드포인트를 직접 호출합니다.

```
GET /api/board/69f461fa489c51d3cb8b7c37
```

응답 본문에서 FLAG를 획득합니다.

## 4. 결과 (Result)

- **Flag**: `DH{f823bac286ef352172b1cd73c812708ea356a000}`

## 5. 보안 인사이트 (Retrospective)
- **Root Cause**: MongoDB ObjectID는 타임스탬프·머신 식별자·순차 카운터 조합으로 생성되어 예측 가능함. `_id`를 목록에서 숨기는 것만으로는 직접 접근을 막을 수 없으며, 이는 IDOR(Insecure Direct Object Reference) 취약점으로 이어짐.
- **Countermeasures**:
  - **서버 측 권한 검증**: 게시글 조회 시 `is_secret` 여부와 요청자의 권한을 반드시 서버에서 확인하여, 비밀 게시글은 작성자·관리자만 접근 가능하도록 제한.
  - **예측 불가 ID 사용**: ObjectID 대신 암호학적으로 안전한 랜덤 UUID(v4)를 사용하거나, 별도의 접근 토큰을 발급해 ID 자체를 추산 불가능하게 구성.
  - **MongoDB 최신화**: MongoDB 4.0 이상에서는 ObjectID의 카운터가 랜덤 값으로 초기화되어 순차 예측이 어려워짐.
