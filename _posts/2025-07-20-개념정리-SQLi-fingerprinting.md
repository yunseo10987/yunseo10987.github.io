---
title: "[2025-07-20] SQL Injection - Fingerprinting"
excerpt: "SQL Injection - Fingerprinting 개념 정리"

categories:
  - Justice
tags:
  - [SQL Injection, Fingerprinting]

permalink: /Justice/[2025-07-20] SQL Injection - Fingerprinting/

toc: true
toc_sticky: true

date: 2025-07-20
last_modified_at: 2025-07-20
---

## 🦥 본문

## System Table

데이터베이스 시스템 내부에서 사용되는 메타 데이터를 저장하는 테이블. 설정 및 계정 정보, 테이블과 컬럼 정보, 쿼리의 정보 등을 포함한다. 

### System Table : MySQL

초기에는 `information_schema`, `mysql`, `performance_schema`, `sys` 데이터베이스 존재. 

```sql
mysql> show databases;
/*
+--------------------+
| Database           |
+--------------------+
| information_schema |
| DREAMHACK          | # 이용자 정의 데이터베이스
| mysql              |
| performance_schema |
| sys                |
+--------------------+
*/
```

- 스키마 정보와 테이블 정보 : `TABLE` 테이블을 통해 스키마의 정보와 테이블 정보를 조회할 수 있다
    
    ```sql
    //스키마 정보
    mysql> select TABLE_SCHEMA from information_schema.tables group by TABLE_SCHEMA;
    /*
    +--------------------+
    | TABLE_SCHEMA       |
    +--------------------+
    | information_schema |
    | DREAMHACK          |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+
    5 rows in set (0.01 sec)
    */
    
    //테이블 정보
    mysql> select TABLE_SCHEMA, TABLE_NAME from information_schema.TABLES;
    /*
    +--------------------+----------------+
    | TABLE_SCHEMA       | TABLE_NAME     |
    +--------------------+----------------+
    | information_schema | CHARACTER_SETS |
    ...
    | DREAMHACK          | users          |
    | mysql              | db             |
    ...
    +--------------------+----------------+
    292 rows in set (0.01 sec)
    */
    ```
    
- 컬럼 정보 : `COLUMNS`테이블을 통해 컬럼의 정보를 조회할 수 있습니다.
    
    ```sql
    mysql> select TABLE_SCHEMA, TABLE_NAME, COLUMN_NAME from information_schema.COLUMNS;
    /*
    +--------------------+----------------+--------------------+
    | TABLE_SCHEMA       | TABLE_NAME     | COLUMN_NAME        |
    +--------------------+----------------+--------------------+
    | information_schema | CHARACTER_SETS | CHARACTER_SET_NAME |
    ...
    | DREAMHACK          | users          | uid                |
    | DREAMHACK          | users          | upw                |
    ...
    | mysql              | db             | Db                 |
    | mysql              | db             | User               |
    ...
    +--------------------+----------------+--------------------+
    3132 rows in set (0.07 sec)
    */
    ```
    
- 실시간 실행 쿼리 정보 : PROCESSLIST 테이블을 통해 실시간으로 실행되는 쿼리를 조회할 수 있다. sys 데이터베이스의 SESSION 테이블을 통해 실행 중인 계정과 함께 조회하는 방법도 있다.
    
    ```sql
    // PROCESSLIST 테이블을 통한 실시간 실행 쿼리 조회
    mysql> select * from information_schema.PROCESSLIST;
    /*
    +-------------------------------------------------+
    | info                                            |
    +-------------------------------------------------+
    | select info from information_schema.PROCESSLIST |
    +-------------------------------------------------+
    1 row in set (0.00 sec)
    */
    
    // SYS 데이터베이스의 SESSION 테이블을 통한 계정 및 실행 쿼리 조회
    mysql> select user,current_statement from sys.session;
    /*
    +----------------+------------------------------------------------+
    | user           | current_statement                              |
    +----------------+------------------------------------------------+
    | root@localhost | select user,current_statement from sys.session |
    +----------------+------------------------------------------------+
    1 row in set (0.05 sec)
    */
    ```
    
- DBMS 계정 정보  : `USER_PRIVILEGES` 테이블을 통한 계정 정보 조회. `MYSQL` 데이터베이스의 `USER` 테이블을 통해 조회하는 방법도 있다.
    
    ```sql
    // USER_PRIVILEGES 테이블을 통한 계정 정보 조회
    mysql> select GRANTEE,PRIVILEGE_TYPE,IS_GRANTABLE from information_schema.USER_PRIVILEGES;
    /*
    PRIVILEGE_TYPE: 특정 사용자가 가진 권한의 종류를 나타냅니다.
    IS_GRANTABLE: 특정 사용자가 자신의 권한을 다른 유저에게 줄 수 있는지 YES 또는 NO로 표시됩니다.
    +-------------------------+-------------------------+--------------+
    | GRANTEE                 | PRIVILEGE_TYPE          | IS_GRANTABLE |
    +-------------------------+-------------------------+--------------+
    | 'root'@'localhost'      | SELECT                  | YES          |
    ...
    | 'root'@'localhost'      | SUPER                   | YES          |
    ...
    | 'user_test'@'localhost' | USAGE                   | NO           |
    +-------------------------+-------------------------+--------------+
    58 rows in set (0.00 sec)
    */
    
    // MYSQL 데이터베이스의 USER 테이블을 통한 계정 정보 조회
    mysql> select User, authentication_string from mysql.user;
    /*
    authentication_string: 사용자의 비밀번호를 해싱한 값 입니다.
    +------------------+-------------------------------------------+
    | User             | authentication_string                     |
    +------------------+-------------------------------------------+
    | root             | *...                                      |
    | mysql.sys        | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE |
    | mysql.session    | *THISISNOTAVALIDPASSWORDTHATCANBEUSEDHERE |
    | user_test        | *...                                      |
    +------------------+-------------------------------------------+
    4 rows in set (0.00 sec)
    */  
    ```
    

### System Table : PostgreSQL

초기에는 `postgres`, `template1`, `template0` 데이터베이스가 존재한다. 

```sql
postgres=$ select datname from pg_database;
/*
  datname  
-----------
 postgres
 template1
 template0
(3 rows)
*/
```

- 스키마 정보 : `pg_catalog`, `information_schema` 를 통해 스키마 정보를 조회
    
    ```sql
    postgres=$ select nspname from pg_catalog.pg_namespace;
    /*
          nspname       
    --------------------
     pg_toast
     pg_temp_1
     pg_toast_temp_1
     pg_catalog
     public
     information_schema
    (6 rows)
    */
    ```
    
- 테이블 정보 : `information_schema` 를 통해 테이블의 정보를 조회
    
    ```sql
    postgres=$ select table_name from information_schema.tables where table_schema='pg_catalog';
    /*
               table_name
    ---------------------------------
    pg_shadow
    pg_settings
    pg_database
    pg_stat_activity
    ...
    */
    
    postgres=# select table_name from information_schema.tables where table_schema='information_schema';
    /*
                  table_name
    ---------------------------------------
    schemata
    tables
    columns
    ...
    */
    
    // table_schema를 통해 스키마 및 테이블 정보 조회
    postgres=$ select table_schema, table_name from information_schema.tables;
    /*
        table_schema    |              table_name
    --------------------+---------------------------------------
     pg_catalog         | pg_statistic
    ...
     information_schem  | information_schema_catalog_name
    ...
    */
    ```
    
- 컬럼 정보 : `information_schema.columns` 를 통해 컬럼 정보 조회
    
    ```sql
    postgres=$ select table_schema, table_name, column_name from information_schema.columns;
    /*
        table_schema    |      table_name         |    column_name
    --------------------+-------------------------+------------------
     pg_catalog         | pg_stat_user_indexes    | relid
    ...
     information_schema | view_routine_usage      | specific_name
    ...
    */
    ```
    
- DBMS 계정 정보 : `pg_catalog.pgshadow` 테이블을 통해 계정 정보 조회
    
    ```sql
    postgres=$ select usename, passwd from pg_catalog.pg_shadow;
    /*
     usename  |               passwd
    ----------+-------------------------------------
     postgres | md5df6802cb10f4000bf81de27261c1155f
    (1 row)
    */
    ```
    
- DBMS 설정 정보 : `pg_catalog.pg_settings` 테이블을 통해 설정정보 조회
    
    ```sql
    postgres=$ select name, setting from pg_catalog.pg_settings;
    /*
                      name                  |                 setting
    ----------------------------------------+------------------------------------------
     allow_system_table_mods                | off
     application_name                       | psql
     ...
    */
    ```
    
- 실시간 실행 쿼리 확인 : `pg_catalog.pg_stat_activity` 테이블을 통해 실시간 실행 쿼리 조회
    
    ```sql
    postgres=$ select usename, query from pg_catalog.pg_stat_activity;
    /*
     usename  |                          query                          
    ----------+---------------------------------------------------------
     postgres | select usename, query from pg_catalog.pg_stat_activity;
    (1 row)
    */
    ```
    

### System Table : Oracle

- 데이터베이스 정보 : `all_tables` 는 현재 사용자가 접근 가능한 테이블 집합
    
    ```sql
    SELECT DISTINCT owner FROM all_tables
    SELECT owner, table_name FROM all_tables
    ```
    
- 컬럼 정보 : `all_tab_columns` 를 통해 컬럼 정보를 확인
    
    ```sql
     SELECT column_name FROM all_tab_columns WHERE table_name = 'users'
    ```
    
- DBMS 계정 정보 : `all_users` 테이블을 통해 DBMS 계정 정보 조회
    
    ```sql
    	SELECT * FROM all_users
    ```
    

## Fingerprinting

### 정의

대상 시스템의 특징과 정보를 수집하는 과정. 공격 전에 각 시스템이 가지고 있는 고유한 특징을 수집하는 “정찰”의 단계. 

### DBMS Fingerprinting

가장 먼저 알아야 할 정보는 DBMS 종류와 버전

- 쿼리 실행 결과 출력 방법 : 환경 변수나 메소드 사용
    
    ```sql
    SELECT @@version
    SELECT version()
    ```
    
- 에러 메시지 출력 방법 : 잘못된 쿼리를 삽입하여 DBMS 정보 확인
    
    ```sql
    select 1 union select 1, 2;
    # MySQL => ERROR 1222 (21000): The used SELECT statements have a different number of columns
    (select * from not_exists_table)
    # SQLite => Error: no such table: not_exists_table
    ```
    
- 참 또는 거짓 출력 : 쿼리 실행 결과가 참 거짓 여부만을 출력하는 경우 Blind SQLi를 통해 DBMS 정보 확인 가능
    
    ```sql
    mid(@@version, 1, 1)='5';
    substr(version(), 1, 1)='P';
    ```
    
    - 참 또는 거짓 여부를 출력하지 않는다면 time based SQLi를 통해 Blind SQLi 를 한다.

### Fingerprinting : MySQL

- 쿼리 실행 결과 출력 : `version` 환경 변수 또는 `version` 함수 사용
    
    ```sql
    mysql> select @@version; # select version();
    +-------------------------+
    | @@version               |
    +-------------------------+
    | 5.7.29-0ubuntu0.16.04.1 |
    +-------------------------+
    1 row in set (0.00 sec)
    ```
    
- 에러 메시지 출력 : 에러 코드를 통해 유추 가능. 1222는 MySQL에서 명시한 코드
    
    ```sql
    mysql> select 1 union select 1, 2;
    ERROR 1222 (21000): The used SELECT statements have a different number of columns
    ```
    
- 참 또는 거짓 출력
    
    ```sql
    # @@version => '5.7.29-0ubuntu0.16.04.', mid(@@version, 1, 1) => '5'
    mysql> select mid(@@version, 1, 1)='5';
    +------------------------+
    | mid(@@version,1,1)='5' |
    +------------------------+
    |                      1 |
    +------------------------+
    1 row in set (0.00 sec)
    mysql> select mid(@@version, 1, 1)='6';
    +------------------------+
    | mid(@@version,1,1)='6' |
    +------------------------+
    |                      0 |
    +------------------------+
    1 row in set (0.00 sec)
    ```
    
    - time based SQLi : `sleep` 메소드 사용
        
        ```sql
        mysql> select mid(@@version, 1, 1)='6' and sleep(2);
        +---------------------------------------+
        | mid(@@version, 1, 1)='6' and sleep(2) |
        +---------------------------------------+
        |                                     0 |
        +---------------------------------------+
        1 row in set (0.00 sec)
        
        mysql> select mid(@@version, 1, 1)='5' and sleep(2);
        +---------------------------------------+
        | mid(@@version, 1, 1)='5' and sleep(2) |
        +---------------------------------------+
        |                                     0 |
        +---------------------------------------+
        1 row in set (2.00 sec)
        ```
        

### Fingerprinting : PostgreSQL

- 쿼리 실행 결과 출력 : `version` 함수 사용
    
    ```sql
    postgres=$ select version();
    version
    --------
     PostgreSQL 12.2 (Debian 12.2-2.pgdg100+1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 8.3.0-6) 8.3.0, 64-bit
    (1 row)
    ```
    
- 에러 메시지 출력
    
    ```sql
    postgres=$ select 1 union select 1, 2;
    ERROR:  each UNION query must have the same number of columns
    LINE 1: select 1 union select 1, 2;
    ```
    
- 참 또는 거짓 출력
    
    ```sql
     /* version() => 'PostgreSQL ...', substr(version(), 1, 1) => 'P' */
    postgres=$ select substr(version(), 1, 1)='P';
     ?column?
    ----------
     t
    (1 row)
    postgres=# select substr(version(), 1, 1)='Q';
     ?column?
    ----------
     f
    (1 row)	
    ```