---
title: "[2025-07-14] Command Injection"
excerpt: "Command Injection 개념 정리"

categories:
  - Justice
tags:
  - [Command Injection]

permalink: /Justice/[2025-07-14] Command Injection/

toc: true
toc_sticky: true

date: 2025-07-14
last_modified_at: 2025-07-14
---

## 🦥 본문

### 정의

사용자 입력값을 시스템 명령어로 실행하게 하는 취약점. 명령어를 실행하는 함수에 이용자가 임의의 인자를 전달할 수 있을 때 발생.

### 메타 문자

shell이 명령을 해석하거나 조작하는 데 사용되는 제어 문자

| 구문 | 의미 | 설명 | 예시 코드 | 결과 (요약) |
| --- | --- | --- | --- | --- |
| `` `` | 명령어 치환 (구식) | `` ``안의 명령어 결과를 실행 | `echo `echo theori`` | `theori` 출력 |
| `$()` | 명령어 치환 (신식) | `$(명령어)` 형식으로 중첩 사용 가능 | `echo $(echo theori)` | `theori` 출력 |
| `&&` | 명령어 연속 실행 (AND) | 앞 명령어 **성공 시** 뒤 명령어 실행 | `echo hello && echo theori` | `hello`, `theori` 출력 |
| `||` | 명령어 연속 실행 (OR) | 앞 명령어 **실패 시** 뒤 명령어 실행 | `$ cat / || echo theori`
 | `cat: /: Is a directory
theori`   |
| `;` | 명령어 구분자 | 앞 명령어 성공/실패 여부와 **무관하게** 뒤 명령어 실행 | `echo hello ; echo theori` | `hello`, `theori`  |
| `|` | 파이프 (Pipe) | 앞 명령어 **출력 결과를 뒤 명령어 입력으로 전달** | `echo id | /bin/sh` | `uid=1001(theori) gid=1001(theori) groups=1001(theori)` |

### 예시

아래와 같은 코드를 작동시키는 API가 있다고 하자.

```python
@app.route('/ping')
def ping():
    ip = request.args.get('ip')
    os.system("ping -c 1 " + ip)
```

```python
http://example.com/ping?ip=8.8.8.8; cat /etc/passwd
```

위의 코드를 삽입하는 경우에는 /etc/passwd 파일 내용이 외부에 노출될 수 있다. 

### 대응 방법

1. 사용자 입력 검증 + 화이트 리스트 사용
2. 사용자 입력값을 os.system() 등에 직접 넣지 않기
3. 서브프로세스 사용
    - subprocess.run() 방식을 사용하면 쉘을 사용하지 않아 인젝션 위험에 안전하다
    - 서브프로세스를 통해 프로그램을 실행하고 사용자의 입력은 단순 인자로 전달하게 된다. command Injection을 해도 인자가 문자열로 인식되기 때문에 해당 문자열을 시도하고 오류가 발생하기만 한다.
4. 메타 문자 등을 이스케이프 처리 
5. 권한 최소화