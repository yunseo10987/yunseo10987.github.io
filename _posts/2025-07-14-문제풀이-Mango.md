---
title: "[2025-07-13] CSRF-advanced"
excerpt: "CSRF-advanced"

categories:
  - Problem
tags:
  - [CSRF]

permalink: /Problem/[2025-07-13] CSRF-advanced/

toc: true
toc_sticky: true

date: 2025-07-13
last_modified_at: 2025-07-13
---

## 🦥 본문

```c
#!/usr/bin/python3
from flask import Flask, request, render_template, make_response, redirect, url_for
from selenium.webdriver.common.by import By
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from hashlib import md5
import urllib
import os

app = Flask(__name__)
app.secret_key = os.urandom(32)

try:
    FLAG = open("./flag.txt", "r").read()
except:
    FLAG = "[**FLAG**]"

users = {
    'guest': 'guest',
    'admin': FLAG
}

session_storage = {}
token_storage = {}

def read_url(url, cookie={"name": "name", "value": "value"}):
    cookie.update({"domain": "127.0.0.1"})
    service = Service(executable_path="/chromedriver")
    options = webdriver.ChromeOptions()
    try:
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
        driver.get("http://127.0.0.1:8000/login")
        driver.add_cookie(cookie)
        driver.find_element(by=By.NAME, value="username").send_keys("admin")
        driver.find_element(by=By.NAME, value="password").send_keys(users["admin"])
        driver.find_element(by=By.NAME, value="submit").click()
        driver.get(url)
    except Exception as e:
        driver.quit()
        # return str(e)
        return False
    driver.quit()
    return True

def check_csrf(param, cookie={"name": "name", "value": "value"}):
    url = f"http://127.0.0.1:8000/vuln?param={urllib.parse.quote(param)}"
    return read_url(url, cookie)

@app.route("/")
def index():
    session_id = request.cookies.get('sessionid', None)
    try:
        username = session_storage[session_id]
    except KeyError:
        return render_template('index.html', text='please login')

    return render_template('index.html', text=f'Hello {username}, {"flag is " + FLAG if username == "admin" else "you are not an admin"}')

@app.route("/vuln")
def vuln():
    param = request.args.get("param", "").lower()
    xss_filter = ["frame", "script", "on"]
    for _ in xss_filter:
        param = param.replace(_, "*")
    return param

@app.route("/flag", methods=["GET", "POST"])
def flag():
    if request.method == "GET":
        return render_template("flag.html")
    elif request.method == "POST":
        param = request.form.get("param", "")
        if not check_csrf(param):
            return '<script>alert("wrong??");history.go(-1);</script>'

        return '<script>alert("good");history.go(-1);</script>'

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('login.html')
    elif request.method == 'POST':
        username = request.form.get('username')
        password = request.form.get('password')
        try:
            pw = users[username]
        except:
            return '<script>alert("user not found");history.go(-1);</script>'
        if pw == password:
            resp = make_response(redirect(url_for('index')) )
            session_id = os.urandom(8).hex()
            session_storage[session_id] = username
            token_storage[session_id] = md5((username + request.remote_addr).encode()).hexdigest()
            resp.set_cookie('sessionid', session_id)
            return resp 
        return '<script>alert("wrong password");history.go(-1);</script>'

@app.route("/change_password")
def change_password():
    session_id = request.cookies.get('sessionid', None)
    try:
        username = session_storage[session_id]
        csrf_token = token_storage[session_id]
    except KeyError:
        return render_template('index.html', text='please login')
    pw = request.args.get("pw", None)
    if pw == None:
        return render_template('change_password.html', csrf_token=csrf_token)
    else:
        if csrf_token != request.args.get("csrftoken", ""):
            return '<script>alert("wrong csrf token");history.go(-1);</script>'
        users[username] = pw
        return '<script>alert("Done");history.go(-1);</script>'

app.run(host="0.0.0.0", port=8000)
```

- make_response는 응답 객체를 생성하는 메소드
- md5는 해쉬값을 생성하는 메소드
- Login API
    
    랜덤 숫자 생성 → 세션 스토리지 { 랜덤 : 아이디 } → 토큰 스토리지 {랜덤 : (아이디+주소) 의 해쉬값 } → 쿠키 { sessionId : 랜덤 } → index로 돌아감
    
- 비밀번호 변경 API
    
    쿠키에서 랜덤값 받아옴 → 랜덤값을 통해 세션/토큰 스토리지에서 아이디와 해쉬값 받아옴 → 변경할 비밀번호 받아옴 → 비밀번호 없으면 csrf_token 주면서 change_password.html로 이동
    
    → 있을 때는 get으로 받아온 csrf 토큰과 비교함
    
- read_url
    
    ```c
    driver.find_element(by=By.NAME, value="username").send_keys("admin")
    driver.find_element(by=By.NAME, value="password").send_keys(users["admin"])
    driver.find_element(by=By.NAME, value="submit").click()
    ```
    
    - Selenium 브라우저에서 해당 요소들을 찾아서 admin과 비밀번호를 입력하고 로그인함
- change_password.html
    
    ```c
    <h2>Change Password</h2><br/>
    <form method="GET">
      <div class="form-group">
        <label for="InputName">new password</label>
        <input type="password" class="form-control" placeholder="new password" name="pw" required>
      </div>
      <input type="text" value="{{ csrf_token }}" name="csrftoken" hidden>
      <button type="submit" class="btn btn-default">Change Password</button>
    </form>
    ```
    
    - csrf_token이 숨겨져 있음 흠….

### 풀이

Selenium 브라우저에서 admin으로 로그인해서 비밀번호를 변경하려 해도 csrf_token 때문에 문제가 된다. 하지만 토큰은 md5(아이디+주소)를 통해 생성되는 데 이 해쉬값은 같은 input을 넣으면 같은 값이 나온다.

그래서 md5를 이용해서 해쉬값을 알아내서 변경할 비밀번호와 csrf토큰 값을 보내어 변경해서 성공했다!!!

```c
<img src= /change_password?pw=1234&csrftoken=7505b9c72ab4aa94b1a4ed7b207b67fb>
```

- 이제 1단계들은 혼자 풀 수 있는 듯