---
title: "[2025-07-08] Template Engine"
excerpt: "Template Engine 개념 정리"

categories:
  - Justice
tags:
  - [Template]

permalink: /Justice/[2025-07-08] Template Engine/

toc: true
toc_sticky: true

date: 2025-07-08
last_modified_at: 2025-07-08
---

## 🦥 본문

지정된 템플릿 양식과 데이터가 합쳐져 HTML 문서를 출력하는 소프트웨어 

- 템플릿 파일 : {% raw %}{{name}}{% endraw %} 같은 변수 자리 표시자가 있는 HTML 형태의 파일
- 데이터 : 동적인 값
- 렌더링 : 템플릿 엔진이 두 가지를 결합하여 최종 HTML을 생성

### Sever Side Template Engine

서버 쪽에서 템플릿 구성을 하는 방식. 서버에서 DB 혹은 API에서 가져온 데이터를 미리 정의된 템플릿에 넣어 HTML을 그려 클라이언트에 전달하는 역할. 완성된 페이지를 받을 수 있기 때문에 SEO에 유리

동작 과정

1. 클라이언트 요청을 서버로 보낸 후 서버가 데이터 처리를 함
2. 미리 정의된 Template에 필요한 데이터를 넣음
3. 서버에서 데이터가 들어간 HTML을 그린 후 클라이언트에게 전달

### Client side Template Engine

HTML 형태로 코드를 작성할 수 있으며 동적으로 DOM을 그리게 해주는 역할. 서버의 부하가 줄어들고 사용자의 상호작용에 따라 페이지의 일부만을 빠르게 업데이트하여 반응성이 좋고 효율적임. 초기 로딩 시간이 길어질 수가 있음

동작 과정

1. 클라이언트에서 공통 프레임을 Template로 만듦
2. 데이터를 서버로 받으면 Template에 넣고 DOM 객체에 동적으로 그려줌