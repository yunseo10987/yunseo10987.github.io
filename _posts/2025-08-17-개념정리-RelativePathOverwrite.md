---
title: "[2025-08-17] Relative Path Overwrite(RPO)"
excerpt: "Relative Path Overwrite 개념, 공격 방법(JS, CSS Injection과의 연계) 및 대응 방법"

categories:
  - Justice
tags:
  - [Relative Path Overwrite, XSS, CSS Injection]

permalink: /Justice/[2025-08-17] Relative Path Overwrite(RPO)/

toc: true
toc_sticky: true

date: 2025-08-17
last_modified_at: 2025-08-17
---

## 🦥 본문

### 정의 및 개념

웹 애플리케이션이 상대 경로(relative path)를 사용해 리소스를 불러올 때 발생하는 취약점. 공격자가 경로를 조작하여 원래 의도한 파일 대신 다른 리소스를 불러오게 만드는 기법. 

### URL rewrite

웹 서버가 요청된 URL을 내부적으로 다른 경로로 변환하는 기술. 

- 목적
    - 사용자 친화적 URL 제공.
    - SEO 개선
    - 특정 요청을 공통 처리 스크립트로 라우팅
- 예시) 스크립트명 이하의 경로를 별도로 지정해도 같은 페이지가 조회되는 경우
    - `/index.php/path` 같이 새로운 경로를 지정해도 서버가 `/index.php`를 참조하도록 구현
- 문제점 : 상대 경로와 절대 경로의 차이
    
    ```jsx
    <script src="/app/main.js"></script>
    <script src="app/main.js"></script>
    ```
    
    - `/` 의 차이로 첫 번째는 절대 경로(사이트 최상위 기준)를 탐색하고 로드. 두 번째는 상대 경로(현재 페이지 기준)로 탐색하고 로드
    - `index.php` 에서는 두 번째 스크립트가 `index.php/app/main.js` 로 실행됨 → 브라우저가 JS로 인식하고 불러 → `index.php`의 페이지 내용을 자바스크립트의 내용으로 사용 가능

### 공격 방법

- JS와 연계 : RPO 취약점을 로드하는 JS 코드의 앞 부분을 공격자가 조작하는 경우 XSS 연계 가능.
    - EX)
        1. `/static/script.js` 로 로드되어야 할 파일을 `/USER_INPUT/static/script.js` 의 형태로 로드할 수 있는 경우
        2. `USER_INPUT` 부분을 조작하여 `index.php/alert(1);` 코드를 넣음
        3. 최종 경로는 `/index.php/alert(1);//static/script.js`로 처리
        4. 브라우저는 JS로 인식하고 `/index.php` 를 실행 → HTML 반환
        5. alert(1); 부분은 JS 코드로 실행. 뒤 `//static/script.js` 는 주석 처리되어 무시
- CSS와 연계 : CSS의 특징인 올바르지 않은 문법을 만나면 무시하고 다음 문법으로 넘어간다는 점을 이용. RPO 취약점을 이용해 올바르지 않은 스타일시트를 임포트하고 유효한 CSS 문법을 삽입하여 CSS Injection과 연계
    - EX)
        1. 아래와 같은 페이지의 코드가 있다고 가정
            
            ```jsx
            <link rel="stylesheet" href="style.css">
            ```
            
        2. 공격자가 URL 조작 (RPO)
            
            ```jsx
            http://victim.com/page.php/evil.css
            ```
            
            - 서버에서는 `page.php`를 실행하지만 브라우저는 css로 판단하여 실행
            - 이 때 악성 CSS 코드를 삽입하여 CSS Injection 실행
        - 악성 CSS 코드를 삽입하는 방법 : page.php가 html 처럼 실행됨. page.php에서 파라미터로 받아서 html 페이지 내에서 사용한다면 body에 실어서 보냄
            - 예시 1 : 페이지가 쿼리나 경로 일부를 화면에 출력하는 경우
                
                ```jsx
                [서버 코드]
                <!-- page.php (예시) -->
                검색어: <?= htmlspecialchars($_GET['q']) ?>
                
                [공격자 URL]
                http://victim.com/page.php/evil.css?q=body{background:url(//attacker/x)}
                ```
                
            - 예시 2 : 404에서 해당 요청 문자열을 페이지에 출력하는 경우
                
                ```jsx
                http://victim.com/page.php/evil.css/*{background:url(//att/x)}
                ```
                
- base-uri 미지정 : base 태그를 미지정하는 경우 현재 페이지 URL 기준으로 상대 경로 해석. 기준점을 원하는 곳으로 바꿀 수 있기 때문에 원하는 리소스 유도 가능

### 대응 방법

1. 절대 경로 사용 
2. MIME 타입•헤더 강화 
    - Content-Type 지정 : 
    CSS → `text/css`, JS → `application/javascript`, HTML → `text/html`
    - X-Content-Type-Options: nosniff 지정
    - CSP 적용
3. URL Rewrite / PathInfo 제한 
    - Apache: `AcceptPathInfo Off`
    - Nginx: 정규식 매칭 강화해서 잘못된 경로 거부.
4. 존재하지 않는 파일 요청 시, 404 HTML 대신 명확한 에러 응답 반환
5. <base> 태그 관리