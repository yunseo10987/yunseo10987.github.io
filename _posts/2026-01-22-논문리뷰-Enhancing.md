---
title: "[2026-01-22] Enhancing DDoS Attack Detection and Mitigation in SDN Using an Ensemble Online Machine Learning Model"
excerpt: "DDoS Detection"

categories:
  - Reviews
tags:
  - [NetWork]

permalink: /Reviews/[2026-01-22] Enhancing DDoS Attack Detection and Mitigation in SDN Using an Ensemble Online Machine Learning Model/

toc: true
toc_sticky: true

date: 2026-01-22
last_modified_at: 2026-01-22
---

## 🦥 본문

## Motivation

기존의 offline-batch learning 방식의 한계

1. 정적 학습 : 과거의 데이터로만 학습되어 있어, Zero-day 공격이나 Low-DDoS 패턴을 식별하기 어려움
2.  Concept Drift 발생 : 네트워크 트래픽의 통계적 특성이나 공격 패턴이 시간에 따라 변하여 탐지 성능 저하

## Challenge

1. 실시간 적응성 : 동적인 네트워크 트래픽과 진화하는 공격에 적응력이 필요
2. 탐지 성능 유지 : 실시간으로 적응하면서도 높은 성능을 유지
3. 리소스 제약 내의 최적화  : 제한된 하드웨어 자원 내에서 실시간 적응과 탐지/완화를 수행

## Background

**Online machine learning**  

스트리밍 데이터 스트림으로부터의 적응성과 지속적인 학습. 데이터를 한꺼번에 모아서 학습하는 것이 아닌 새로운 데이터가 들어올 때마다 점진적으로 업데이트

- vs Batch Learning
    
    ![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-22%20Online%20ML/image.png)
    
    - Batch Learning : 전체 데이터 코퍼스를 이용하여 훈련된 모델이 추가적인 학습 없이 배포가 됨. 업데이트를 하기 위해서 정적인 상태를 유지. 방대한 양의 데이터를 한 번에 읽으므로 많은 자원을 급격하게 사용
    - OML : 새로운 데이터가 나타나는 경우나 데이터 분포가 변하는 경우, 미니 배치 단위로 데이터를 학습하여 실시간으로 모델 가중치에 반영

**Ensemble ML**

단일 모델을 사용하는 대신 여러 기저 학습기를 결합하여, 개별 모델이 가진 약점을 상호 보완하고 다양하고 진화하는 DDoS 공격을 처리할 수 있는 신뢰할 수 있는 탐지 메커니즘을 제공

## ARCHITECTURAL DESIGN

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-22%20Online%20ML/image-1.png)

1. 트래픽 수집기 모듈: SDN 환경에서 네트워크 패킷을 캡처하고 기록
2. OML 기반 IDS: 머신러닝 기술을 활용하여 유입되는 트래픽을 실시간으로 처리 및 분류하고, 잠재적인 DDoS 공격을 식별
3. OML 기반 IPS : IDS로부터 받은 정보를 바탕으로 식별된 위협을 완화하기 위해 조치

동작 흐름

1. 트래픽 수집 : SDN 컨트롤러에 연결된 트래픽 수집기가 에지 스위치로부터 트래픽 복사. OpenFlow 스위치의 플로우 테이블에 주기적으로 플로우 엔트리를 요청
2. 보안 채널을 통한 전송 : 요청과 응답은 보안이 유지되는 격리된 채널을 통해 전송되어 연결된 호스트에 노출되는 것을 방지
3. 분류 : 학습된 머신러닝 모델을 사용하여 전처리된 입력 트래픽을 의심스럽거나 정상적인 것으로 분류
4. 완화 명령 전달 : IDS의 결정은 즉시 완화 모듈로 전달. IPS 전략에 따라 적절한 조치를 결정.
5. 규칙 적용 : 결정된 조치를 플로우 규칙으로 변환. 스위치에 구현.

모듈형 설계를 통해 독립적으로 최적화 가능. 특정 컨트롤러에 의존하지 않음

### Traffic collector

트래픽 수집기 모듈은 컨트롤러 내에서 작동. 플로우 트래픽 정보를 가져오고 미러링. 플로우 통계에 대한 주기적인 요청

- 알고리즘
    1. 초기화
    2. AddFlow  
        1. 우선순위(priority), 매치(match), 액션(actions) 설정
        2. flow_id 생성
        3. 매치 및 액션 파싱
        4. flows 딕셔너리에 플로우 추가
        5. flow_id를 포함한 패킷 아웃(packet out) 메시지 전송
    3. GetFlows:
        1. flows 딕셔너리 반환 
    4. UpdateFlowExpiration
        1. flow_expiration_timeout이 존재하면, 새로운 flow_expiration_timeout을 지금으로부터 30초 후로 설정
        2. 그렇지 않으면, flow_expiration_timeout을 현재 시간으로 설정
    5. CollectFlows
        1. flow_expiration_timeout이 존재하고 현재 시간이 flow_expiration_timeout보다 크거나 같으면
            1. flows 딕셔너리의 각 플로우에 대해:
            2. 현재 시간과 플로우의 ['expiration_time'] 차이가 30초보다 크거나 같으면: flows 딕셔너리에서 해당 플로우를 제거
        2. 플로우 만료 타임아웃 업데이트
    6. 연결(connection) 및 table_id 인자 설정
    7. FlowCollector 인스턴스 생성
    8. 우선순위, 매치, 액션을 포함한 플로우 엔트리 추가
    9. 30초마다 플로우 수집

컨트롤러 동작 순서 

1. 각 스위치에 대한 고유 ID를 생성하여 연결된 모든 OpenFlow 스위치를 인증
2. OpenFlow 스위치에 패킷이 도착할 때, 패킷 헤더를 스위치의 플로우 테이블에 있는 플로우 엔트리와 대조. 
    1. 일치하면 바이트 수와 패킷 수 등의 정보를 포함하여 해당 엔트리의 통계가 업데이트
    2. 일치하지 않는 경우, 해당 패킷은 컨트롤러로 전달. 
        1. 컨트롤러는 정의된 정책을 집행하기 위해 스위치의 플로우 테이블에 새로운 플로우 엔트리를 추가. 특정 OpenFlow 스위치에 연결된 모든 호스트에서 생성된 트래픽이 스위치의 플로우 테이블을 채움 

### OML-based IDS

예측력 향상을 위해 탐지 모델의 알고리즘을 다양하게 선택. 실시간 적응성을 위해 낮은 계산 오버헤드를 가진 분류기들을 사용함.

1. 데이터 수집 및 전처리 : 전처리 후 데이터베이스 및 데이터 스트림으로의 저장.
2. 특징 선택 및 엔지니어링 : 네트워크 트래픽 특성에 따라 특징 세트를 지속적으로 조정하는 동적 특징 선택 메커니즘. 실시간으로 관련 특징을 선택함으로써 모델은 계산 복잡성을 최소화하면서 탐지 효능을 최적화
3. 모델 초기화: 온라인 단계에서 실시간 예측을 시작하기 전에, 오프라인 단계 동안 초기 데이터를 사용하여 모델을 초기화 및 기저 학습기 학습.
4. 앙상블 학습: 새로운 데이터가 도착하는 대로 예측을 시작. 예측을 가중치 조합을 통해 결합하여 최종 예측을 도출 
5. 공격 탐지 : 앙상블 모델이 들어오는 데이터에 대한 예측. 의심스러운 데이터는 flagging. 
6. 모델 모니터링 및 업데이트 : 드리프트 탐지 기술로 성능을 모니터링. 새로운 데이터가 유입됨에 따라 주기적으로 업데이트

### OML-BASED IPS

기능

1. 적절한 대응책 집행: 잠재적 위협이 식별되면 IPS는 악성 트래픽 차단이나 지정된 싱크홀(sinkhole) 우회와 같은 적절한 대응
2. 지속적인 학습 및 적응: IPS는 지속적으로 진화하며 변화하는 트래픽 패턴과 새로 등장하는 공격 기술에 맞춰 조정.

프레임 워크 

1. 트래픽 프로파일링: 이 단계에서는 실시간으로 네트워크 트래픽을 분석하고 프로파일링하여 공격의 징후나 비정상적인 패턴을 정확히 찾음
2. 조치 메커니즘 설계: 위협 탐지 시 채택할 적절한 조치와 대응책을 설계
3. 조치 구현: IPS는 공격이 성공하는 것을 막기 위해 악성 트래픽 차단이나 리디렉션과 같은 지정된 조치를 실행
4. 모델 모니터링 및 업데이트: 성능을 지속적으로 모니터링하고 업데이트

## Evaluation

- Setup : 자체 생성 데이터셋과 기존의 벤치마크 데이터셋에서 모델 평가
    - LDDoS/DDoS 공격을 포함하여 자체 생성된 데이터셋을 비롯한 다양한 데이터셋에서 평가.
    - Mininet 및 Ryu SDN 컨트롤러를 사용하여 iperf, Hping3, Scapy와 같은 도구로 DDoS 공격을 시뮬레이션.
    - CICIDS2019, InSDN, slow-read-DDoS-attack-in-SDN과 같은 벤치마크 데이터셋에서 모델을 평가
    - Environment
        
        ![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-22%20Online%20ML/image-2.png)
        
    - network topology
        
        ![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-22%20Online%20ML/image-3.png)
        
- Dataset : 지속 시간, IP 프로토콜, 바이트 수, 패킷 수, SYN 플래그 카운트 등을 포함한 22개의 feature 중 Chi2 특징 선택을 통해 14개로 축소
    
    ![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-22%20Online%20ML/image-4.png)
    
    - 종류
        1. 패킷 기반 속성: 순방향 및 역방향 모두의 총 패킷 수와 같은 패킷에 대한 세부 정보
        2. 네트워크 식별자 속성: IP 주소, 포트 번호, 프로토콜 유형 등 소스 및 목적지 플로우를 정의하는 공통 정보
        3. 서브 플로우 기술자 속성: 순방향 및 역방향 모두에서의 패킷 및 바이트 수와 같이 서브 플로우와 관련된 정보
        4. 도착 간 시간 속성: 순방향 및 역방향 모두에서의 패킷 도착 간 시간에 관한 정보
        5. 바이트 기반 속성: 바이트 특정 데이터와 관련된 것으로, 순방향 및 역방향 모두에서 전송된 총 바이트 수
        6. 플로우 타이머 속성: 활성 및 비활성 기간을 포함하여 각 플로우의 지속 시간에 관한 정보
        7. 플로우 기술자 속성: 순방향 및 역방향 모두에서의 패킷 및 바이트 수와 같은 트래픽 플로우 세부 정보
        8. 플래그 속성: SYN 플래그, RST 플래그, Push 플래그 등과 같은 플래그 관련 정보
    
    ![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-22%20Online%20ML/image-5.png)
    
    Chi2 특징 선택을 통해 가장 높은 특징만을 선택하여 데이터 차원을 14개로 축소
    
- Result
    - 자체 데이터셋
        
        
        | Accuracy  | Precision | Recall | F1 Score | 오탐율 |
        | --- | --- | --- | --- | --- |
        | **0.9926** | **0.9910** | **0.9962** | **0.981** | **0.025** |
    - 벤치마크 데이터셋
        
        ![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-22%20Online%20ML/image-6.png)
        
    - 응답 시간
        
        ![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-22%20Online%20ML/image-7.png)
        

## Contribution & Discussion

1. SDN 환경에서 제로 데이, 고속 및 저속 공격을 포함하여 진화하는 DDoS 공격을 탐지/완화하기 위해 설계된 앙상블 온라인 ML 모델 
2. 지속적인 업데이트를 통해 역동적인 DDoS 지형에서 모델의 적응성과 회복력을 향상

→ 실적용 시 northbound 통신에 병목 현상 문제