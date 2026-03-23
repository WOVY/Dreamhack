# 🛡️ Dreamhack Web Hacking Study

## 📌 Project Overview
- **Description**: [Dreamhack](https://dreamhack.io/) 워게임 취약점 분석 및 익스플로잇 기록 저장소입니다.
- **Objective**: 각 분야별 취약점의 Root Cause를 파악하고, 재현 가능한 분석 보고서 작성을 목표로 합니다.

## 🛠️ Tech Stack & Tools
- **Platform**: Dreamhack Wargame
- **Tools**: Burp Suite, Chrome DevTools, Python (Requests, Pwntools)
- **Environment**: Linux (Ubuntu), Docker

## 📑 Challenge List
문제명을 클릭하면 상세 분석 리포트(README)로 이동합니다.

| Category | Challenge Name | Difficulty | Core Concept | Status |
| :--- | :--- | :---: | :--- | :---: |
| Client-side | [devtools-sources](./devtools-sources) | Beginner | Information Exposure | ✅ |
| Web (Auth) | [cookie](./cookie) | Beginner | Cookie Manipulation | ✅ |
| Web (Auth) | [session-basic](./session-basic) | LEVEL 1 | Information Exposure | ✅ |
| Web (Auth) | [xss-1](./xss-1) | LEVEL 1 | Reflected XSS | ✅ |
| Web (Auth) | [xss-2](./xss-2) | LEVEL 1 | DOM-based XSS / innerHTML Bypass | ✅ |
| Web (Auth) | [csrf-1](./csrf-1) | LEVEL 1 | CSRF (Cross-Site Request Forgery) | ✅ |
| Web (Auth) | [csrf-2](./csrf-2) | LEVEL 1 | CSRF (Account Takeover) | ✅ |
| Server-side | [simple_sqli](./simple_sqli) | LEVEL 1 | SQL Injection (Auth Bypass) | ✅ |
| Server-side | [Mango](./Mango) | LEVEL 2 | NoSQL Injection | ✅ |


## 🔗 Links
- **Dreamhack Profile**: [https://dreamhack.io/users/79375](https://dreamhack.io/users/79375)
