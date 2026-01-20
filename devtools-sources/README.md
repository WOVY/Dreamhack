# 📂 Case Analysis: devtools-sources

## 1. 문제 정보 (Challenge Info)
- **Description**: 개발자 도구의 Sources 탭 기능을 활용해 플래그를 식별하는 문제
- **Target**: Problem Source Files (Local Static Analysis)
- **Flag Format**: `DH{...}`

## 2. 분석 개요 (Overview)
- **Objective**: 클라이언트 사이드 리소스 내 하드코딩된 민감 정보 식별 및 개발자 도구 활용 능력 배양
- **Key Concept**: Client-side Information Exposure, Static Analysis

## 3. 분석 환경 (Environment)
- **OS**: Windows 11
- **Browser**: Google Chrome (DevTools)
- **Tools**: Chrome DevTools - Sources Tab

## 4. 분석 및 해결 단계 (Steps)

### Step 1: 소스 코드 확보 및 로컬 실행
문제에서 제공된 압축 파일을 다운로드하여 로컬 환경에 해제한 뒤, 브라우저로 `index.html`을 실행하여 분석 환경을 구축함.

### Step 2: Sources 탭 분석
`F12`를 통해 개발자 도구를 실행하고 `Sources` 탭으로 이동하여 로드된 정적 리소스 파일(HTML, CSS, JS) 구조를 확인함.

### Step 3: 키워드 검색 및 플래그 식별
전체 소스 코드 내에서 플래그 형식인 `DH{...}` 키워드를 찾기 위해 전체 검색 기능(`Ctrl + Shift + F`)을 수행함.
- 분석 결과, `main.scss` 파일 하단에 주석 처리된 상태로 노출된 플래그 데이터를 식별함.

## 5. 결과 (Result)

### Flag 획득 화면
![Flag Success Screenshot](./devtools_sources_flag.png) 

- **Flag**: `DH{2ed07940b6fd9b0731ef698a5f0c065be9398f7fa00f03ed9da586c3ed1d54d5}`

## 6. 보안 인사이트 (Retrospective)
- **Root Cause**: 중요 데이터(Flag)가 클라이언트에게 그대로 전송되는 정적 파일 내에 주석 형태로 포함됨.
- **Countermeasures**: 민감한 정보는 절대 클라이언트 사이드 코드에 포함해서는 안 되며, 보안 검증이 필요한 데이터는 서버 측(Server-side)에서 엄격히 관리해야 함.
