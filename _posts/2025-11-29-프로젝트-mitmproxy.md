---
title: "[2025-11-27] MITMPROXY 사용법"
excerpt: "설명충 프로젝트"

categories:
  - Project
tags:
  - [NetWork]

permalink: /Project/[2025-11-27] MITMPROXY 사용법/

toc: true
toc_sticky: true

date: 2025-11-27
last_modified_at: 2025-11-27
---

## 🦥 본문

1. 인수 필요 없이 `mitmproxy`, `mitmdump`, `mitmweb` 실행. 기본 포트 8080
2. 네트워크 설정 필요 
    - 윈도우에서 설정
        1. 프록시 서버 설정
        2. 프록시 서버의 IP와 포트 입력 
3. 해당 프록시에 접속 후 [`mitm.it`](http://mitm.it)에 접속하여 인증서 다운 후 인증서 마법사 실행 
    - 리눅스의 경우 `mitmproxy` 실행 시 `home/사용자/.mitmproxy`에 인증서들이 발급
        1. 인증서 복사 : 신뢰할 수 있는 인증서를 저장하는 표준 디렉터리에 저장
            
            ```python
            sudo cp ~/.mitmproxy/mitmproxy-ca-cert.pem /usr/local/share/ca-certificates/mitmproxy.crt
            ```
            
        2. 시스템에 인증서 등록
            
            ```python
            sudo update-ca-certificates
            ```
            
4. `mitmdump -s dlp_proxy.py` 로 실행