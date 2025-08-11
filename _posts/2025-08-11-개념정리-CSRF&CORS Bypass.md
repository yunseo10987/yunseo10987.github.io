---
title: "[2025-08-11] CSRF&CORS Bypass"
excerpt: "CSRF&CORS Bypass"

categories:
  - Justice
tags:
  - [CSRF, CORS]

permalink: /Justice/[2025-08-11] CSRF&CORS Bypass/

toc: true
toc_sticky: true

date: 2025-08-11
last_modified_at: 2025-08-11
---

## 🦥 본문

## CSRF Token

### 동작 흐름

1. 같은 오리진에서만 접근 가능한 형태로 특정 Token을 세션에 저장
2. 서버가 HTML form 태그의 `hidden` 속성이나 메타 태그로 전달
3. 요청 시 들어온 토큰과 세션의 토큰을 비교

### 예시

```html
<?php
if (!isset($_SESSION["csrftoken"])) {
    $csrftoken = bin2hex(random_bytes(32));
    $_SESSION["csrftoken"] = $csrftoken;
} else {
    $csrftoken = $_SESSION["csrftoken"];
}
$method = $_SERVER["HTTP_METHOD"];
if ($method !== "GET" && $method !== "HEAD") {
    if (!isset($_POST["csrftoken"]) ||
        !hash_equals($csrftoken, $_POST["csrftoken"]) {
        header("HTTP/1.1 403 Forbidden");
        die("CSRF detected");
    }
    echo "Input value: ";
    echo htmlentities($_POST["query"], ENT_QUOTES|ENT_HTML5, 'utf-8');
}
?>
<form action="" method="POST">
    <input name="csrftoken" type="hidden" value="<?=htmlentities($csrftoken, ENT_QUOTES|ENT_HTML5, 'utf-8'); ?>">
    <input name="query" type="text" />
    <input type="submit" />
</form>
```

### 장점

- 사용자의 추가적인 상호작용 불필요 : CAPTCHA, OTP, 이메일 확인 등과 같은 보안 대책 같은 경우에는 추가 인증, 입력, 클릭 등이 필요하지만 CSRF Token은 자동으로 토큰 전송
- 위조 요청 차단 : 공격자는 토큰값을 알 수 없으므로 위조 불가
- 브라우저 의존도 낮음 : 토큰 생성 및 검증 로직을 서버가 전담하므로, 브라우저/클라이언트 환경 변화와 상관없이 동작. SameSite 쿠키 정책 같 브라우저 기능과 무관하게 작동

### 주의 사항

- 짧은 CSRF Token : Brute-force attack으로 획득할 수 없도록 길어야 함
- 예측 가능한 CSRF Token : PRNG 같은 의사 난수 생성기를 사용하는 경우 예측 가능. CSPRNG 같은 난수 생성기 사용
- CSRF Token 유출 : CSRF Token이 URL 쿼리 파라미터로 넘겨지는 경우, 이후 다른 링크를 방문하였을 때 `Referer` 헤더에 Token 노출
- 긴 유효시간을 가진 CSRF Token : 사용자가 로그아웃 하고 난 뒤 새로운 세션이 생성되도 토큰이 유지되어 공격자가 사용 가능

## CORS Vulnerability

교차 출처 리소스 공유를 위해 `postMessage`, JSONP, CORS 정책 기술 등이 도입

- 취약점의 종류
    - 현재 사이트에서 다른 사이트로 정보 유출 (기밀성) : 다른 사이트로부터 CORS 요청을 받을 때 그 Origin에 대한 검사가 진행되지 않고 응답하거나 Origin에 제약이 없는 경우
    - 다른 사이트에서 현재 사이트 변조 (무결성) : 출처에 대한 신뢰성에 대한 제한과 XSS 필터 등 신뢰하지 않는 입력에 대한 방어가 병행

### `postMessage` Vulnerability

- 정의 : 서로 다른 Origin 간에 메시지를 주고 받을 수 있는 API 및 메소드.
    - 메시지를 전송할 때에는 대상 윈도우의 postMessage를 호출. 수신하는 윈도우는 `message` 전역 이벤트를 통해 청취하여 메시지를 받을 수 있음
    - 문자열뿐만 아니라 객체를 주고 받을 수 있음.
        - 함수, DOM 노드 객체, 프로토타입 및 get/set 속성 정보는 보낼수 없음
        - 객체는 복사 방식으로 전달하므로 송신 후 변경 사항은 반영 X
- **targetWindow.postMessage(message, targetOrigin, [transfer])**
    
    
    | 변수 | 설명 |
    | --- | --- |
    | **targetWindow** | 메시지를 보낼 대상 Window |
    | **message** | 메시지 객체 (함수, DOM 객체 등은 보낼 수 없음) |
    | **targetOrigin** | 메시지 송신 시점에 `targetWindow`의 Origin이 `targetOrigin`과 일치해야 함. `targetOrigin`에 `"*"`를 지정하면 Origin 검사가 이루어지지 않음 |
    | **transfer** | (선택사항) `ArrayBuffer`나 canvas context 등 소유권을 전이할 객체의 배열을 지정 |
- **MessageEvent**
    
    
    | 고유 속성 | 설명 |
    | --- | --- |
    | **origin** | 메시지를 송신한 Origin 반환 |
    | **source** | 메시지를 송신한 Window 객체 반환 |
    | **data** | 복사된 메시지 객체 또는 값 반환 |
    
    ```html
    // 메시지 송신
    targetWindow.postMessage(message, targetOrigin);
    // 메시지 수신
    window.onmessage = function (e) {
        if (e.origin === 'https://dreamhack.io') {
            console.log(e.data);
            e.source.postMessage('Hello, world!', e.origin);
        }
    }
    ```
    
- Origin 전환 경합 조건 : 메시지를 보내는 대상은 출처가 고정된 **웹 문서**가 아닌 **창(윈도우)** 창의 경우에는 사용자가 하이퍼링크를 방문하거나 리다이렉트 시 Origin이 변경 → 의도하지 않은 Origin으로 메시지 전송
    - 공격 시나리오
        1. 부모 윈도우에서 자식 윈도우 생성
        2. 자식 윈도우가 부모 윈도우로 `postMessage`로 메시지 및 비밀값 전송 
        3. 부모 윈도우가 공격자의 다른 웹 사이트로 리다이렉트
        4. 자식 윈도우가 계속 보내는 메시지가 공격자의 사이트로 수신 

### JSONP Vulnerability

- Origin 검사 부재 : JSONP에 한정된 취약점은 아님. 하지만 JSONP 특성상 CSRF 공격에 더 취약함
    - CSRF Token 사용하여 방어
- 콜백 함수명 검증 부재로 인한 제공자 XSS : 콜백명에 HTML 코드 등을 삽입하여 HTML로 인신 → XSS 취약점 발생
    - 필터 적용
        - HTTP `Accept` 헤더에 `text/javascript` MIME 타입이 포함되어 있는 지 검사
        - `Content-Type: text/javascript` 설정 및 `X-Content-Type-Options: nosniff` 헤더로 자바스크립트가 아닌 다른 콘텐츠로 인식되는 경우 방지
            - MIME Type sniffing : 서버에서 받은 Content-Type을 신뢰하지 않고 리소스 데이터를 분석하여 적절한 형식으로 해석하는 것. `nosniff` 로 설정하여 서버에서 전달 받은 Content-Type 만을 믿을 수 있도록 해야 함
- JSONP API 침해 사고 발생 시 이용자 XSS : JSONP API가 침해 사고를 당해 악의적인 응답이 오는 공격 → JSONP가 아닌 CORS 정책 헤더를 사용하는 이유