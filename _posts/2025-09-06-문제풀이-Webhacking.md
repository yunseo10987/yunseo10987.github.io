---
title: "[2025-09-06] WebHacking.kr 1일차"
excerpt: "Challenge 1, Challenge 15, Challenge 26,Challenge 16"

categories:
  - Problem
tags:
  - [XSS]

permalink: /Problem/[2025-09-06] WebHacking.kr 1일차/

toc: true
toc_sticky: true

date: 2025-09-06
last_modified_at: 2025-09-06
---

## 🦥 본문

## Challenge 1

```jsx
<?php
  include "../../config.php";
  if($_GET['view-source'] == 1){ view_source(); }
  if(!$_COOKIE['user_lv']){
    SetCookie("user_lv","1",time()+86400*30,"/challenge/web-01/");
    echo("<meta http-equiv=refresh content=0>");
  }
?>
<html>
<head>
<title>Challenge 1</title>
</head>
<body bgcolor=black>
<center>
<br><br><br><br><br>
<font color=white>
---------------------<br>
<?php
  if(!is_numeric($_COOKIE['user_lv'])) $_COOKIE['user_lv']=1;
  if($_COOKIE['user_lv']>=4) $_COOKIE['user_lv']=1;
  if($_COOKIE['user_lv']>3) solve(1);
  echo "<br>level : {$_COOKIE['user_lv']}";
?>
<br>
<a href=./?view-source=1>view-source</a>
</body>
</html>
```

`SetCookie()` : `user_lv` 값을 1 로 지정. 만료 기간은 한달. /challenge/web-01 경로로 온 것만 셋팅

### 풀이 과정

`user_lv` 의 값이 3보다 크고 4보다 작으면 solve. 쿠키에서 해당 값을 3.5로 지정.

## Challenge 15

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2025-09-06%20Webhacking/image.png)

`alert()` 이후에는 webhacking.kr로 리디렉션 된다. 코드를 보기 위해서는 해당 자바스크립트를 실행 못하게 막아야 한다.

### 풀이 과정

크롬에서 설정 > 개인 정보 보호 > 자바스크립트 탭에서 해당 사이트의 자바스크립트 권한을 해제하면 빈 페이지에 다음과 같은 코드가 나온다

```jsx
<script>
  alert("Access_Denied");
  location.href='/';
  document.write("<a href=?getFlag>[Get Flag]</a>");
</script>
```

파라미터로 getFlag를 넣어주면 해결된다. 

## Challenge 26

```jsx
<?php
  include "../../config.php";
  if($_GET['view_source']) view_source();
?><html>
<head>
<title>Challenge 26</title>
<style type="text/css">
body { background:black; color:white; font-size:10pt; }    
a { color:lightgreen; }
</style>
</head>
<body>
<?php
  if(preg_match("/admin/",$_GET['id'])) { echo"no!"; exit(); }
  $_GET['id'] = urldecode($_GET['id']);
  if($_GET['id'] == "admin"){
    solve(26);
  }
?>
<br><br>
<a href=?view_source=1>view-source</a>
</body>
</html>
```

해당 코드에서 id 값에 admin을 입력하면 성공이지만 필터링이 있다.

### 풀이 과정

해당 코드에서 디코딩을 한 번 더 하기 때문에, `admin`을 한 번 더 인코딩을 하여 URL에 보내 주면 된다.

```jsx
id=%2561%2564%256D%2569%256E
```

## Challenge 16

```jsx
<html>
<head>
<title>Challenge 16</title>
<body bgcolor=black onload=kk(1,1) onkeypress=mv(event.keyCode)>
<font color=silver id=c></font>
<font color=yellow size=100 style=position:relative id=star>*</font>
<script> 
document.body.innerHTML+="<font color=yellow id=aa style=position:relative;left:0;top:0>*</font>";
function mv(cd){
  kk(star.style.left-50,star.style.top-50);
  if(cd==100) star.style.left=parseInt(star.style.left+0,10)+50+"px";
  if(cd==97) star.style.left=parseInt(star.style.left+0,10)-50+"px";
  if(cd==119) star.style.top=parseInt(star.style.top+0,10)-50+"px";
  if(cd==115) star.style.top=parseInt(star.style.top+0,10)+50+"px";
  if(cd==124) location.href=String.fromCharCode(cd)+".php"; // do it!
}
function kk(x,y){
  rndc=Math.floor(Math.random()*9000000);
  document.body.innerHTML+="<font color=#"+rndc+" id=aa style=position:relative;left:"+x+";top:"+y+" onmouseover=this.innerHTML=''>*</font>";
}
</script>
</body>
</html>
```

해당 코드는 evnet.keyCode의 값에 따라 변하는 페이지를 볼 수 있다

### 풀이과정

124에 해당하는 문자열은 `|` 이고 `|.php` 를 접속하면 해결된다.