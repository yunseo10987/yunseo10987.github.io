---
title: "[2026-05-28] MITRE ATT&CK: Design and Philosophy"
excerpt: "MITRE ATT&CK"

categories:
  - Reviews
tags:
  - [NetWork]

permalink: /Reviews/[2026-05-28] MITRE ATT&CK: Design and Philosophy/

toc: true
toc_sticky: true

date: 2026-05-28
last_modified_at: 2026-05-28
---

## 🦥 본문

## Intro

**MITRE ATT&CK** : 타겟으로 알려진 플랫폼과 공격의 모든 진행 단계(attack lifecycle)를 담고 있는, 사이버 공격에 대한 백과사전이자 분석 모델.

**ATT&CK의 요소**

- **Tactics** : 공격이 진행되는 동안 적이 가지는 단기적이고 전술적인 목표
- **Techniques** : tactical goals을 달성하기 위해 사용되는 수단
    - Sub-techniques : Techniques보다 낮은 수준에서 tactical goals을 달성하기 위한 구체적인 수단
- 문서화된 적의 techniques, 과정, 메타데이터 등.

**Background & Goal**

- 전통적인 보안은 공격자가 침입을 못하는 방어 중심이었으나, MITRE 연구원들은 Assume Breach(공격자가 이미 내부에 침투)를 가정.
- Goal : telemetry sensing(원격 측정 감지)와 행동 분석을 통해 기업 네트워크에 침투한 post-compromise(위협의 침해 후) 탐지 능력을 향상시키는 것

**Motivation**

- 위협 행동의 분류 및 문서화 : MITRE 사내 연구 환경(FMX)에서 공격 에뮬레이션과 방어 훈련을 효과적으로 수행하고 그 수준을 측정하기 위해, 실제 적대자들의 공격 행동을 명확한 기준으로 분류하고 문서화
- 공격(Offense)과 방어(Defense)를 담당하는 사람들이 서로 다른 용어를 쓰지 않고, 명확하게 소통할 수 있는 공통된 기준과 분류 체계가 필요했습니다.

## **Use Case**

- 적대자 에뮬레이션 및 레드 티밍**:** 알려진 위협 그룹의 전략을 모방하여 보안 시스템이 이를 방어할 수 있는지 실제 환경에서 검증.
- 행동 분석 개발**:** 특정 악성 파일(IoC)에 의존하지 않고, 공격자의 행동 패턴 자체를 찾아내는 분석 방식 구축
- 방어 격차 평가 및 SOC 성숙도 평가**:** 조직의 보안 사각지대(탐지 불가능한 영역)를 찾아내고, 보안 운영 센터(SOC)의 대응 능력 판단.
- 사이버 위협 인텔리전스 강화**:** 위협 그룹의 행동 프로필을 문서화, 다양한 공격 그룹 간의 공통된 행동 패턴을 파악하여 방어의 우선순위 설정

**Coverage**

- 100% coverage(탐지 및 방어)는 비현실적
    - 기술들을 구현하는 데에는 다양한 절차가 있음
        - 코드를 수정하거나 다른 툴을 사용한다는 등의 여러 절차
    - 참고 사항들이지 체크리스트가 아님
- 목적에 맞게 coverage 기준을 정해야 함
    - 때로는 데이터를 기록하는 것만으로도 coverage가 가능
        - 악성 의심 행위에 대한 과도한 알림은 정상적인 행위에 대해서 가짜 알림을 만듦.
        - 가짜 알림에 지쳐 중요한 알림을 놓칠 수 있음 → 로깅만으로도 충분한 coverage
- 유연하게 적응하고 지속적인 업데이트 할 수 있는 process가 중요

## ATT&CK Model

적의 Tactic를 달성하기 위해 수행할 수 있는 행동을 나타내는 techniques와 sub-techniques의 집합 

**Example** 

지속성(Persistence) Tactic(대상 환경에 계속 남아 있으려는 적의 목표) 아래에는 실행 흐름 하이재킹(Hijack Execution Flow), OS 부팅 전(Pre-OS Boot), 예약된 작업(Scheduled Task/Job)을 포함한 일련의 technique

- 'OS 부팅 전(Pre-OS Boot)' technique은 운영 체제가 부팅되기 전에 지속성이 어떻게 달성되는지 설명하기 위해 부트킷(Bootkit), 구성 요소 펌웨어(Component Firmware), 시스템 펌웨어(System Firmware)로 구성된 세 가지 sub-techniques.

**Technology Domains** : 목표를 달성하기 위해 우회하거나 이용해야 하는 일련의 제약 조건을 제공하는, 적대자가 활동하는 생태계. Enterprise(IT 네트워크 시스템과 클라우드 환경), Mobile, ICS(산업 제어 시스템)으로 나뉨 

- Platforms : 구체적인 OS나 애플리케이션.
    - Mobile이라는 domain 내에는 iOS와 Android가 platform이 있음.

**PRE-ATT&CK** : 접근 권한을 얻기 위한 준비 단계. 요구 사항 수집, 정찰 및 무기화 과정 등이 있음. 

**Tactics** : 적의 전술적 목표이자 어떤 행동을 수행하는 이유. 

- 지속성 유지(persist), 정보 탐색(discover information), 측면 이동(move laterally), 파일 실행(execute files), 데이터 유출(exfiltrate data)
    - 하나의 Tactics 안에 technique이 속해 있는 게 아니라 tag처럼 technique에 여러 개의 목표,Tactics가 붙는 방식
- Tactics 간의 연계성 : 하나의 전술로 끝나지 않고 꼬리를 물고 이어짐.
    - 시스템에 들어오기 위해 초기 접근(Initial Access) 전술 → 악성 스크립트를 돌리기 위해 실행(Execution) ****전술을 연계 → 옆 부서의 서버로 넘어가기 위해 측면 이동(Lateral Movement) 전술을 사용
- 새로운 domain이 등장할 때, tactics도 새로 확장될 수 있음

**Techniques and Sub-Techniques**

- **Techniques** : ****적이 Tactics를 달성하기 위해 행동을 수행하는 방법
- **Sub-Techniques :** 기술이 설명하는 행동을 더 세분화하여, 목표 달성을 위해 행동이 어떻게 사용되는지 더 구체적으로 설명
    - 추상화의 수준을 맞추거나 새로운 공격 수법이 나오면 상위 기술을 건드리지 않고 하위 기술만 쉽게 추가할 수 있도록 구조를 개편
    - 규칙
        - 하나의 sub-techniques는 하나의 상위 techniques에만 속함
        - techniques가 두 가지 tactics에 쓰인다고 해서 sub-techniques도 두 tactics에 쓰일 필요는 없음
        - sub-techniques의 mitigation이나 data source가 있으면 상위 기술의 mitigation으로 인정
        - procedure는 sub-techniques나 techniques 하나에만 적음 (중복 X)
- **Procedures** : 적들이 기술이나 하위 기술을 위해 사용한 구체적인 구현
    - 공격자가 행동을 수행하는 과정에서 특정 악성코드, 스크립트, 또는 도구를 어떻게 사용하는지가 포함
    - Example : APT28 그룹이 희생자의 시스템에서 'PowerShell'을 사용하여 'lsass.exe' 프로세스에 인젝션(주입)한 뒤, LSASS 메모리를 스크래핑하여 자격 증명(비밀번호 등)을 덤프
        - 하나의 공격 절차 안에는 PowerShell 사용, 프로세스 인젝션, LSASS 메모리 접근이라는 서로 다른 고유한 해킹 행동(기술 및 하위 기술)들이 모두 포함

**Group** : 해커 조직 

- 공격자가 어떤 소프트웨어를 써서 어떤 기술을 통해 공격했는 지를 통해 해커 조직을 알아냄
    - 소프트웨어 : techniques나 sub-techniques 들이 구체적으로 실현된 것
        - tool : 정상적인 목적으로 만들어진 프로그램이지만, 해커에게 악용될 수 있는 소프트웨어
        - malware : 악의적인 목적으로 사용하기 위해 만든 소프트웨어
- 기술을 직접 사용 : 특정 해커 조직이 윈도우 기본 명령어를 직접 타이핑하여 자격 증명 탈취(Technique)를 수행
- 소프트웨어를 통해 기술을 구현 : 특정 해커 조직이 'Mimikatz'라는 악성코드(Software)를 사용하여 자격 증명 탈취(Technique)를 수행

**Mitigation :** technique이나 sub-technique이 성공적으로 실행되는 것을 방지하는 데 사용할 수 있는 보안 개념 및 기술 종류. 특정 벤더의 제품에 종속되지 않으며, 특정 솔루션이 아닌 기술의 범주나 종류만을 설명

### Relationships

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-05-28%20MITRE%20ATT&CK/image.png)

## Methodology

ATT&CK을 구축, 유지, 관리에 사용되는 방법론. 

**핵심 개념**

1. 적의 관점 유지
    - 기존의 방어 관점에 비해 맥락 속에서 행동과 잠재적 대응책을 이해하기가 쉬움
    - 행동에 대한 동기를 추적
2. 실증적인 use case를 통한 실제 활동 추적 : 현실적으로 불가능한 이론적인 공격이 아닌 실제로 발생한 사건이나 발생할 가능성이 높은 사건들만 도출 
3. 공격 행동과 방어 대응책을 연결하기 위한 추상화 수준 
    - 고수준의 목표와 저수준의 데이터를 이어주는 중간 수준의 추상화
        - 고수준 : 큰 공격 흐름 (록히드 마틴의 킬체인(Kill Chain)처럼 공격의 큰 흐름(예: 정찰 → 무기화 → 침투 → 목적 달성)만 보여줌
        - 저수준 : 특정 악성코드의 해시값, 구체적인 취약점(CVE) 코드, 또는 단일 패킷의 특정 페이로드 값. 디테일해서 코드를 한 줄만 바꾸거나 값만 바꿔서 우회 가능
    - 중간 수준 기술을 나타내는 행동 패턴 정의

**Tactics**

- 일관성 : 공격 수법은 바뀌지만 근본적인 목표는 잘 변하지 않음. 플랫폼이나 도메인 사이에서도 일관성이 있음
- 세분화 : 분석이 고도화되면서 전술이 세분화
    - 예를 들어, 데이터를 찾는 것과 밖으로 빼내는 것을 유출로만 봤지만 두 행위는 탐지 포인트와 시간이 다르기 때문에 수집과 유출로 분리
- Impact : 이전에는 기밀성의 위협에만 집중했으나 이제는 가용성 파괴, 무결성 파괴를 다루는 Impact 전술이 추가
    - Impact의 데이터 파괴 기술와 Defense Evasion의 파일 삭제 기술은 파일 삭제라는 행동이 같지만 목표가 다르므로 목표를 고려하는 것이 중요

**Techniques & Sub-Techniques**

ATT&CK의 기초이며, 적이 수행하는 개별 행동을 나타냄

- 추상화 : Techniques는 여러 플랫폼에 적용됨. Sub**-**Techniques는 특정 기술이 하나 이상의 플랫폼에 적용될 수 있는 구체적인 방법
- 기술 구분 : 목표(Objective), 행동(Actions), 사용 주체(Use), 요구 사항(Requirements), 탐지(Detection), 완화 조치(Mitigations)
- Example : process injection & SQL injection
    - process injection : 탐지가 어려움. 방어 로직이 다른 해킹 기법과 다름 → 고유한 방어 체계 구축을 위해 독립적인 기술로 분리
    - SQLi : 다른 웹 취약점 공격과 다르지 않아 상위 카테고리에 합침