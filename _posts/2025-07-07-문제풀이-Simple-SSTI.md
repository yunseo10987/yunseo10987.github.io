---
title: "[2025-07-07] Simple-SSTI"
excerpt: "simple-ssti"

categories:
  - Problem
tags:
  - [SSTI]

permalink: /Problem/[2025-07-07] Simple-SSTI/

toc: true
toc_sticky: true

date: 2025-07-07
last_modified_at: 2025-07-07
---

## 🦥 본문

![image.png](../assets/images/posts_img/[2025-07-07]%20Simple-SSTI/image.png)
![image.png](../assets/images/posts_img/[2025-07-07]%20Simple-SSTI/image1.png)

```jsx
#!/usr/bin/python3
from flask import Flask, request, render_template, render_template_string, make_response, redirect, url_for
import socket

app = Flask(__name__)

try:
    FLAG = open('./flag.txt', 'r').read()
except:
    FLAG = '[**FLAG**]'

app.secret_key = FLAG

@app.route('/')
def index():
    return render_template('index.html')

@app.errorhandler(404)
def Error404(e):
    template = '''
    <div class="center">
        <h1>Page Not Found.</h1>
        <h3>%s</h3>
    </div>
''' % (request.path)
    return render_template_string(template), 404

app.run(host='0.0.0.0', port=8000)
```

404가 발생하면 해당 <div>를 출력하는 페이지이다. 처음에는 {% raw %}{{FLAG}}{% endraw %}를 넣으면 되는 거 아닌가? 라고 생각했다. 하지만

![image.png](../assets/images/posts_img/[2025-07-07]%20Simple-SSTI/image2.png)

위와 같이 아무 것도 나오지 않는다. 

정답은 {% raw %}{{config}}{% endraw %}를 통해 어플리케이션 설정 값 중 secret_key 값을 알아내는 것이다. 근데 왜 {% raw %}{{FLAG}}{% endraw %}와 {% raw %}{{config}}{% endraw %} 모두 전역 변수인데 하나는 되고 하나는 안될까?
{% raw %}{{config}}{% endraw %}는 Jinja 템플릿에게 자동으로 넘겨줄 수 있는 전역 변수이다. 하지만 {% raw %}{{FLAG}}{% endraw %}는 전역 변수이지만 Jinja에게 넘겨주는 코드가 필요하여 FLAG = FLAG라는 것이 추가되야 한다.

즉. 정리해보자면, {% raw %}{{config}}{% endraw %}는 템플릿 엔진도 이미 알고 있는 전역 변수이므로 자연스레 데이터와 HTML을 합쳐 보내주지만 {% raw %}{{FLAG}}{% endraw %}는 파이썬에 있는 전역 변수이지만 템플릿 엔진에 FLAG라는 변수가 전역 변수 FLAG임을 알려주는 코드가 있어야 한다는 것이다.