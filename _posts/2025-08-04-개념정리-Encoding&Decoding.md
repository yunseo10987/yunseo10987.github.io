---
title: "[2025-08-04] Encoding&Decoding"
excerpt: "Encoding&Decoding"

categories:
  - Justice
tags:
  - [XSS]

permalink: /Justice/[2025-08-04] Encoding&Decoding/

toc: true
toc_sticky: true

date: 2025-08-04
last_modified_at: 2025-08-04
---

## 🦥 본문

## Encoding

### 정의

어떤 데이터를 다른 형식으로 바꾸는 것. 

### 목적

주로 특수 문자나 바이너리 데이터를 안전하게 전송하거나 저장하기 위해 사용

- 안전한 이유 : 특수 문자들을 인코딩을 통해 명령어가 아닌 단순 텍스트로 인식하게 한다.

### 종류

| 인코딩 방식 | 예시 | 사용 목적 |
| --- | --- | --- |
| **URL 인코딩** | `<` → `%3C` | 브라우저가 URL로 해석 가능한 문자로 변환 |
| **HTML 엔티티 인코딩** | `<` → `&lt;` | HTML에서 태그로 해석되지 않게 |
| **Base64** | `hello` → `aGVsbG8=` | 바이너리 파일 등을 문자열로 변환 |
| **Unicode 이스케이프** | `s` → `\u0073` | JS/JSON 내부에서 안전하게 표현 |

## Decoding

### 정의

인코딩된 데이터를 원래대로 되돌리는 과정

### Encoding&Decoding 동작 흐름

1. 사용자 입력
    
    사용자가 form이나 URL을 통해 값을 입력 
    
2. 브라우저가 서버에 요청 
    
    URL 구조 깨짐을 방지하기 위해 자동 인코딩
    
3. 서버 수신
    
    서버 프레임워크가 자동으로 디코딩
    
4. 서버가 클라이언트에 응답
    
    이 때 자동으로 HTML 인코딩을 하지 않음. escape를 사용하여 HTML 인코딩을 하고 브라우저가 텍스트로 인식하게 해야 함
    
5. 브라우저 렌더링
    
    브라우저에서 자동으로 HTML 디코딩 
    

### 궁금했던 점

| 서버에서 받은 상황 | escape 여부 | 브라우저 해석 | 결과 |
| --- | --- | --- | --- |
| `<script>alert(1)</script>` | ❌ 없음 | script 태그 → 실행 | ❌ XSS |
| `<script>alert(1)</script>` | ✅ 있음 | `&lt;script&gt;alert(1)&lt;/script&gt` 로 변경. 단순 텍스트로 판단. | ✅ 안전 |
| `<scr&#x69;pt>...</scr&#x69;pt>` | ❌ 없음 | `&#x69;` → `i` → `<script>` | ❌ XSS |
| `<scr&amp;#x69;pt>` | ✅ 있음 | `&amp;` → `&`, 그대로 보임 | ✅ 텍스트만 보임 |

왜 위와 같은 상황이 발생하는 거지? 

브라우저에서 디코딩 하는 객체가 다름

`&lt` 같은 경우에는 텍스트 렌더러를 통해 텍스트로 판단하게 됨.

`&#amp` 같은 경우에는 HTML 파서를 통해 `<script>` 를 실행하게 함