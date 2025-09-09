---
title: "[2025-09-08] WebHacking.kr 2일차"
excerpt: "Challenge 24"

categories:
  - Problem
tags:
  - [XSS]

permalink: /Problem/[2025-09-08] WebHacking.kr 2일차/

toc: true
toc_sticky: true

date: 2025-09-08
last_modified_at: 2025-09-08
---

## 🦥 본문

## Challenge 24

```php
<?php
  include "../../config.php";
  if($_GET['view_source']) view_source();
?><html>
<head>
<title>Challenge 24</title>
</head>
<body>
<p>
<?php
  extract($_SERVER);
  extract($_COOKIE);
  $ip = $REMOTE_ADDR;
  $agent = $HTTP_USER_AGENT;
  if($REMOTE_ADDR){
    $ip = htmlspecialchars($REMOTE_ADDR);
    $ip = str_replace("..",".",$ip);
    $ip = str_replace("12","",$ip);
    $ip = str_replace("7.","",$ip);
    $ip = str_replace("0.","",$ip);
  }
  if($HTTP_USER_AGENT){
    $agent=htmlspecialchars($HTTP_USER_AGENT);
  }
  echo "<table border=1><tr><td>client ip</td><td>{$ip}</td></tr><tr><td>agent</td><td>{$agent}</td></tr></table>";
  if($ip=="127.0.0.1"){
    solve(24);
    exit();
  }
  else{
    echo "<hr><center>Wrong IP!</center>";
  }
?><hr>
<a href=?view_source=1>view-source</a>
</body>
</html>
```

`extract($_SERVER)` : `$_SERVER` 배열의 키를 변수명으로, value를 변수의 값으로 가져옴

`$HTTP_USER_AGENT` : `User-Agent` 헤더 값을 담음

맨 처음에는 $agent값에 스크립트를 삽입하여 `location.href = 127.0.0.1` 을 실행시키려고 했다. 

`htmlspecialchars` : 문자열 안에 있는 HTML 문자를 HTML 엔티티로 변환해주는 함수

위와 같은 함수 때문에 인코딩되어 스크립트를 실행시킬 수 없었다

### 풀이 과정

1. `extract($_COOKIE)` 를 이용하여 `$REMOTE_ADDR` 값을 덮어 씌운다.
    1. 쿠키값에 `$REMOTE_ADDR` 키 값을 추가하고 `127.0.0.1` 을 입력한다
2. 필터링이 있기 때문에 다음과 `$REMOTE_ADDR` 의 value에 다음과 같이 입력한다. 
    
    REMOTE_ADDR : 112277...00...00...1
