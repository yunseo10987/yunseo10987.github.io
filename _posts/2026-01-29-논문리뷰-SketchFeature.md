---
title: "[2026-01-29] SketchFeature"
excerpt: "High-Quality Per-Flow Feature Extractor Towards Security-Aware Data Plane
High-Quality Per-Flow Feature Extractor Towards Security-Aware Data Plane"

categories:
  - Reviews
tags:
  - [NetWork]

permalink: /Reviews/[2026-01-29] SketchFeature/

toc: true
toc_sticky: true

date: 2026-01-29
last_modified_at: 2026-01-29
---

## 🦥 본문

## Motivation

1. 3rd-order feature의 필요성
    - 1st-order feature : flow-agnostic & stateless
        - EX) 프로토콜, TTL, 패킷 크기
    - 2nd-order feature :  flow-aware & stateful
        - EX) flow 크기, 평균, 분산, 최소값, 최대값
    - 3rd-order feature : 흐름당 패킷 간 지연(IPD) 분포 및 패킷 크기(PS) 분포 등의 분포 정보
        - 애플리케이션 계층 공격을 탐지하기 위한 flow 별 분포 정보 중요
        - 3차 특징은 가장 풍부한 정보를 담고 있어, 저차원 특징도 도출 가능
    
    → 양자화를 사용했어도 하드웨어 리소스 제약, 계산 복잡성 등으로 측정이 어려움 
    
2. 데이터 평면 내에서 이루어져야 하는 필요성  
    - 실시간 처리 : 공격 탐지 로직의 위치와 관계없이, 모든 흐름과 트래픽에 대한 Line-rate 조사 및 완화
        - Line-rate : 네트워크 장비가 들어오는 패킷을 지연이나 손실 없이 물리적 회선이 허용하는 최대 속도로 처리하는 능력
        - LFA 같은 공격을 신속하게 차단
    
    →  복잡한 연산 지원하지 않음, 하드웨어 리소스 제약 
    
3. 기존 우회 방식의 한계 데이터 평면에서 이루어지는 특징 추출기의 한계 (증상 탐지기) 
    - FlowLens, NetWarden
        
        control plane에서 flow 당 탐지를 수행. CPU 처리량의 한계로 의심스러운 트래픽에 대해서만 control plane으로 보낸 후 탐지
        
        - 전체 flow가 아닌 Top-K를 통해 일부 공격을 탐지하는 데, 공격자가 패턴을 일부 변경하는 순간 탐지하기 어려움
        - Control plane Flooding 공격 : data plane - control plane 통신 사이의 flooding
            - control plane rate-limiter로는 막을 수 없고 data plane rate-limiter는 리소스 부족&추가 매커니즘이 필요로 인해 막기 어려움
    - NetBeacon
        
        data plane에 특징 추출기와 ML 배치. 리소스 제한으로 1차 특징만을 사용 
        

## SketchFeature Design

### Quantization

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-29%20SketchFeature/image.png)

제한된 리소스 내에서 분포 정보를 얻기 위해 양자화를 선택. 연속적인 데이터를 L개의 칸으로 나누는 과정. 이 때 양자화의 범위가 작을 수록 해상도가 높아져 성능이 좋아지지만 리소스의 제한이 있어 L값을 최적화해야 함.

### Sketch (Count-Min)

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-29%20SketchFeature/image-8.png)

데이터의 통계적 특성을 기록하는 확률적 자료구조 

1. 해시 함수를 통해 분포 정보를 각 행마다 해싱
2. 해싱 값이 가리키는 인덱스 카운터의 값을 1씩 증가 

나중에 특정 데이터를 확인하기 위해서는 각 행에 대한 해당 데이터의 해시 인덱스 값들을 확인하여 최솟값을 반환  

- 기존 방식
    
    스케치를 물리적으로 나누는 hard partitioning
    
    - 메모리 비효율성 문제
    - 해시 충돌 현상 발생 → 오차 범위 발생
- virtual sketch
    
    단일 메모리 공간을 가상으로 공유. 각 bin마다 서로 다른 해시 함수를 사용함
    
- 실험
    
    ![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-29%20SketchFeature/image-1.png)
    

### Phantom decoding

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-29%20SketchFeature/image-2.png)

존재하지 않는 feature에 해시 충돌로 인해 0이 아닌 값을 반환하는 현상. 분포에 노이즈가 발생

- Bloom Filter 사용
    
    bin에 따라 서로 다른 해시 함수를 사용하여 가상으로 분할. flow를 인코딩할 때, 해당 bin에 대응하는 bloom filter의 비트가 1로 설정
    
- 실험
    
    ![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-29%20SketchFeature/image-3.png)
    

## SketchFeature

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-29%20SketchFeature/image-4.png)

QNT (양자화 함수), virtual sketch, BloomFilter로 구성

**동작 흐름**  

- 인코딩
    1. 양자화 함수를 통해 특징 값을 양자화하여 양자화 빈을 결정 
    2. Flow ID와 양자화 레벨을 해싱하여 해당 Bloom Filter 부분을 1로 저장
    3. 동시에, Flow ID와 양자화 레벨을 해싱하여 virtual sketch에 +1을 하여 저장
- 디코딩
    1. 분포를 복원할 때, 해당 Flow ID와 양자화 레벨을 해싱
    2. Bloom filter를 먼저 확인하여 1인지 검증
    3. Bloom filter에서 1인 경우에만 스케치에서 값을 읽어오며, 오차를 줄이기 위해 여러 칸 중 가장 작은 값을 선택 
    
    → 분포 벡터 완성 
    

## Evaluation

**baseline과의 비교** 

- 메모리 변화에 따른 정확도

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-29%20SketchFeature/image-5.png)

- workload 변화에 따른 정확도 ㅇ

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-29%20SketchFeature/image-6.png)

**다른 시스템과의 비교**

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-01-29%20SketchFeature/image-7.png)

- NetWarden과의 비교
    
    NetWarden은 CTC 탐지에서 최적에 가깝지만, 
    
    NetWarden의 사전 지식을 바탕으로 한 탐지, 무제한 리소스 등을 가정했기 때문에 현실성이 없음
    
- FlowLens&NetBeacon과의 비교
    
    SketchFeature의 모든 부분에서 성능이 뛰어남
    

## Discussions & Contribution

- Bloom Filter를 활용한 팬텀 디코딩 문제 해결
- 스케치 가상화를 통한 메모리 효율 극대화
- 스케치를 사용하여 공격에 대한 견고함
    - 해시 충돌을 악용하기 위해서 해시 함수의 seed를 알아내는 것은 어려움

 

- 공격자가 SketchFeature에 사전 지식이 있어서 mice flow를 많이 알면..?

→ 동적 양자화를 통해 마이스 플로우는 쳐내는 게..?