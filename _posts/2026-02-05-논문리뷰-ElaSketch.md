---
title: "[2026-02-05] Elastic Sketch Adaptive and Fast Network-wide Measurements "
excerpt: "Sketch"

categories:
  - Reviews
tags:
  - [NetWork]

permalink: /Reviews/[2026-02-05] Elastic Sketch Adaptive and Fast Network-wide Measurements /

toc: true
toc_sticky: true

date: 2026-02-05
last_modified_at: 2026-02-05
---

## 🦥 본문

## Background and Motivation

기존 연구에서는 트래픽 특성 (가용 대역폭, 플로우 크기 분포, 패킷 속도 포함)에 따른 동적 네트워크 측정에 대해 다루지 않음 

트래픽 특성

- 가용 대역폭 : 네트워크 혼잡 시 능동적으로 압축
- 패킷 속도 : 네트워크 공격 시 패킷은 작아지지만 속도는 빨라져서 가용 대역폭은 여유가 있지만 처리 속도가 못 따라가서 중요한 정보를 기록하지 못할 수 있음. 중요하지 않은 정보를 능동적으로 폐기하여 처리 속도를 가속화.
- 플로우 크기 분포 : 마우스 플로우와 엘리펀트 플로우를 분리하여 저장하기 위해 다른 데이터 구조를 사용. 플로우 크기 분포는 변하기 때문에 메모리 크기를 동적으로 할당

요구 사항 

- 범용성 : 노드는 여러 작업을 수행해야 하기 때문에 모든 작업을 위한 하나의 범용 데이터 구조.
- 신속성 : 패킷의 처리 시간이 짧고 일정
- 정확성 : 제한된 리소스로 오류율을 낮춤

### Challenge

1. 압축하면서도 높은 정확도 
    - 기존 연구 : 고정된 메모리 크기
    - 해당 연구 : 네트워크 가용성이 낮을 때, 메모리 크기를 압축하지만 큰 메모리였을 때의 이점을 살려서 처음부터 작았던 고정된 메모리 크기보다 높은 정확도를 달성하는 것이 목표
2. 패킷 속도에 맞는 처리 속도 
    - 기존 연구 : 일정한 처리 속도. 패킷 하나를 처리하는 데 여러 번의 메모리 접근
    - 해당 연구 : 패킷 속도가 낮을 때에는 2회, 패킷 속도가 높을 때에는 1회의 메모리 접근 수행
3. 플로우 분포에 따른 메모리 할당
    - 대부분은 마우스 플로우이고 극소수가 엘리펀트 플로우이지만, 엘리펀트 플로우가 중요한 경우가 많으므로, 적절한 메모리 크기 할당

### Contribution

1. 새로운 스케치인 Elastic Sketch를 제안
    - 트래픽 특성에 적응하는 능력
    - 범용적이고 빠르며 정확
    - 엘리펀트 플로우와 마우스 플로우를 분리하는 기술
    - 스케치 압축을 위한 기술
2. 다양한 플랫폼에서 작동

### Generic Method for Measurements

- 플로우 크기 추정
    - 플로우 ID는 5-튜플(5-tuple)의 조합이거나 프로토콜 단독
    - 플로우의 패킷 수를 플로우 크기로 간주.
        - EX) 최소 패킷을 64바이트라고 가정할 때 120바이트의 패킷이 들어오면 이를 2개의 패킷으로 간주
- Heavy Heater 탐지 : 미리 정의된 임계값보다 큰 플로우 탐지
- Heavy Change 탐지 : 인접한 두 time window 사이에서 플로우 크기가 임계값 이상으로 증가하거나 감소한 플로우 탐지
- 플로우 크기 분포 추정/엔트로피 추정/카디널리티 추정

## Elastic Sketch

**데이터 구조** 

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-02-05%20ElaSketch/image.png)

- Heavy part : 엘리펀트 플로우를 저장
    - Heavy part $H$는 해시 함수 $h(.)$와 연결된 해시 테이블로 구성
    - 각 버킷은 플로우 ID, positive votes($vote^{+}$), negative votes($vote^{-}$), flag 정보를 저장
        - positive votes는 이 플로우에 속한 패킷의 수(플로우 크기)를 기록
        - negative votes는 다른 플로우에 속한 패킷의 수를 기록
        - flag는 해당 플로우에 대한 패킷이 light part에 포함되어 있을 가능성이 있는 지를 나타냄
    - 패킷이 해당 버킷에 들어왔을 때, 버킷에 있는 플로우와 동일하면 positive votes를 증가하고 그렇지 않으면 negative votes를 증가시킴.  만약 $\frac{\text{부정 투표수}}{\text{긍정 투표수}} \ge \lambda$ (사전에 정의된 임계값)라면, 해당 flow를 evict 시키고 그 자리에 해당 플로우 삽입
- Light part : 마우스 플로우를 저장. CM 스케치
    - d개의 배열로 구성되어 있고 각 배열은 하나의 해시 함수와 연결되어 있음. w개의 카운터로 구성.
    - 패킷이 들어오면, 플로우 ID를 추출하여 d개의 해시 함수를 계산하여 배열마다 하나씩 카운터 위치를 찾아낸 뒤, 그 d개의 카운터를 1씩 증가
    - 조회 : d 개의 해시 카운터 값을 얻은 후, 그 중 최솟값을 통해 조회

**동작 흐름** 

- 삽입
    1. 플로우 ID가 $f$인 패킷이 들어오면, 헤비 파트의 버킷 수인 $B$를 이용하여 $H[h(f)\%B]$ 버킷으로 해싱
    2. 해당 버킷에 $f_{1}, vote^{+}, flag_{1}, vote^{-}$이 있다고 가정할 때, $f$와 에 $f_{1}$가 같으면 $vote^{+}$를 증가. 아닌 경우 $vote^{-}$를 증가시키고 $f_{1}$을 방출할 지 결정
        - 버킷이 비어 있는 경우.  버킷에 ($f, 1, F, 0$)을 삽입합니다. 여기서 $F$는 버킷에서 방출이 일어난 적이 없음을 의미. 삽입 종료
    3. 다음과 같은 경우가 있음
        - $vote^{-}$를 증가시킨 후에도 $\frac{vote^{-}}{vote^{+}} < \lambda$인 경우 ($f, 1$)을 CM 스케치에 삽입. 1은 해당 플로우의 패킷의 수를 의미함.
        - $vote^{-}$를 증가시킨 후에도 $\frac{vote^{-}}{vote^{+}} > \lambda$인 경우. 버킷을 ($f, 1, T, 1$)로 설정. 기존 플로우 ($f_{1}, vote^{+}$)을 CM 스케치로 삽입. 매핑된 카운터들은 $vote^{+}$만큼 증가
- 조회 : Heavy part 조회 후 없는 플로우에 대해서는 Light part에서 조회
    - Heavy part에 있는 경우
        1. 플래그가 false인 경우 : 플로우 크기는 오차 없는 $vote^{+}$ 값
        2. 플래그가 true인 경우 : $vote^{+}$ 값과 CM 스케치 조회 결과를 더해야 함 

**정확도 분석** 

- Theorem
    - 벡터 $\mathbf{f} = (f_{1}, f_{2}, ..., f_{n})$를 스트림의 크기 벡터. $f_{i}$는 i번째 플로우의 크기
    - 두 매개변수 $\epsilon$과 $\delta$가 주어졌을 때,  $w = \lceil \frac{e}{\epsilon} \rceil$ (e는 오일러 수)이고 $d = \lceil \ln \frac{1}{\delta} \rceil$
    - $d$(카운터 배열의 수)와 $w$(각 배열의 카운터 수)를 갖는 엘라스틱 스케치
    - $*\mathbf{f}_{L}*$은 라이트 파트에 기록된 서브 스트림의 크기 벡터
    - 엘라스틱 스케치에 의해 추정된 플로우 i의 크기 $\hat{f}{i}$*는* 최소 *$1 - \delta$*의 확률로 다음과 같은 범위를 가짐
        
        ![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-02-05%20ElaSketch/image-1.png)
        
    - 추정 오차는 Count-Min의 $||\mathbf{f}||_{1}$ 대신 $||\mathbf{f}_{L}||_{1}$에 의해 제한. 실제로는 스트림의 대부분의 패킷이 헤비 파트에 기록되므로, $||\mathbf{f}_{L}||_{1}$은 보통 $||\mathbf{f}||_{1}$보다 현저히 작음.
    - 매개변수 d와 w가 같을 때 CM보다 더 촘촘한 오차 범위를 가짐
- Heavy part의 flag가 false인 플로우의 경우 오차가 없음
- Heavy part의 flag가 true인 플로우의 경우 Light part에 기록된 플로우의 일부분만 오차가 있음
- Light part에서는 플로우 크기만 기록되므로 많은 수의 작은 카운터 (8비트 카운터)를 사용할 수 있음.
    - 기존 CM은 엘리펀트 플로우를 수용하기 위해 32비트 카운터를 사용
- 엘리펀트 충돌 : 정확도가 최악인 경우. 둘 이상의 엘리펀트 플로우가 동일한 버킷에 매핑될 때 일부 엘리펀트 플로우가 라이트 파트로 방출되어 일부 마우스 플로우가 크게 과대 추정되는 경우
    - 엘리펀트 충돌률 ($P_{hc}$) : 하나 이상의 엘리펀트 플로우가 매핑된 버킷의 수를 전체 버킷 수로 나눈 값으로 정의. 각 버킷에 매핑된 엘리펀트 플로우의 수는 이항 분포를 따름
    - 여러 개의 서브 테이블을 사용하거나 하나의 버킷에 여러 개의 키-값 쌍을 사용하여 해결

### Adaptivity to Available Bandwidth

**스케치 압축** 

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-02-05%20ElaSketch/image-2.png)

카운터들을 그룹화한 다음 동일한 그룹 내의 카운터들을 하나의 카운터로 압축

크기가  $zw' \times d$인 스케치 A. (d는 배열의 개수. z는 그룹화 개수. w’는 그룹 내 카운터 개수 

1. 그룹화 
    1. A를 z개의 동일한 구역으로 나눔. 각 구역의 크기는 $w' \times d$
2. 크기는 $w' \times d$인 스케치 B 생성
3. 압축 연산자에 의해 압축
    1. 합산 압축(Sum Compression, SC) : 각 그룹의 카운터들을 합산.  $B_{i}[j] = \sum_{k=1}^{z}{A_{i}^{k}[j]}$
        
        동일한 정확도를 갖지만, 큰 메모리에 기록된 정보를 충분히 활용하지 못함
        
    2. 최대 압축(Maximum Compression, MC) : 각 그룹의 카운터들 중 가장 큰 값으로 압축. $B$$_{i}[j] = \max{A_{i}^{1}[j], A_{i}^{2}[j], ..., A_{i}^{z}[j]}$
    - 원본의 정보를 더 많이 활용하여 더 나은 정확도를 갖춤. 합산 압축 후에는 CM 스케치의 오차 범위가 변하지 않는 반면, 최대 압축 후에는 오차 범위가 더 촘촘해짐
    - MC를 사용하면 과대 추정 오차는 발생하지만, 과소 추정 오차는 발생하지 않음
    - 압축 속도가 빠르며, 압축 해제가 필요하지 않고 추가 데이터 구조가 필요하지 않음

**압축 스케치 조회** : 해시 함수 $h_{i}(.) \% w$를 $h_{i}(.) \% w \% w'$로 변경

- 임의의 정수 i와 두 정수 w, w'가 주어졌을 때, w가 w'로 나누어 떨어지면 (i % w) % w' = i % w'가 성립

**스케치 병합**  

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-02-05%20ElaSketch/image-3.png)

각각 서버는 측정 노드들로부터 많은 스케치를 수신하여 이를 병합한 후 수집기로 전송. 모든 스케치에 대해 동일한 해시 함수를 사용. 

- 병합 방법
    1. 합산 병합 (Sum merging) : 동일한 크기를 가진 두 CM 스케치를 대응하는 모든 카운터 쌍을 더하여 두 스케치를 합침. 빠르지만 정확하지 않음 
    2. 최대 병합(Maximum Merging, MM) : 두 스케치에 대응하는 모든 카운터 쌍 중 최댓값을 통해 새로운 스케치 생성. 과소 추정 오차가 발생하지 않으며, 서로 다른 너비를 가진 두 스케치를 병합할 수 있음. 
        
        ![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-02-05%20ElaSketch/image-4.png)
        

### Adaptivity to Packet Rate

패킷 속도가 높으면 입력 큐가 빠르게 채워져 모든 패킷의 정보를 기록하기 어려움. 

**기존 연구**

SketchVisor : fast path라는 전용 컴포넌트를 사용. 분할 상환 O(1) 업데이트 복잡도를 가짐에도 불구하고 최악의 경우에는 전체 데이터 구조를 탐색해야 함 →  상당한 메모리 접근을 유발하고 성능을 저해 

**동작 흐름**

1. 입력 큐의 패킷 수가 미리 정의된 임계값보다 커지면, 들어오는 패킷이 heavy part에만 접근하도록 하여 엘리펀트 플로우의 정보만 기록하고 마우스 플로우는 폐기
    - light part를 폐기하는 것이 아니라 업데이트를 안하는 것.
    - light part에 기록되어야 할 정보만 손실
2. 일반적인 경우 Heavy part 삽입 과정과 같음 
3. 버킷에 있는 플로우 $f$가 다른 플로우 $f'$로 교체될 때, $f'$의 플로우 크기는 $f$의 크기로 설정
    - 기존 알고리즘에서 지우고 새 ID를 쓰고 값을 1로 초기화하는 것은 여러 연산이 필요하지만 카운트 값을 그대로 덮어 씌워 메모리 접근과 연산을 최소화

각 삽입 작업은 헤비 파트 내 버킷에 대한 단 한 번의 probe(메모리 접근)만을 필요. 정확도가 저하되는 대신 높은 처리 속도를 달성. 

패킷 속도가 내려가면 다시 이전 알고리즘을 사용. 

### Adaptivity to Flow Size Distribution

플로우 크기 분포는 동적이므로 Heavy part의 크기도 동적으로 결정 

**동작 흐름** 

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-02-05%20ElaSketch/image-5.png)

1. Heavy part에 작은 메모리 크기를 할당하고 임계값 $T_{1}$을 정의
2. 특정 버킷 안에 있는 모든 플로우의 크기가 $T_{2}$보다 크면 **가득 참**으로 간주
3. 가득 찬 버킷의 수가 임계값 $T_{1}$을 초과하면 Heavy part가 가득 찬 것으로 간주 
4. Heavy part를 복사하고 원본과 결합 
5. 해시 함수는 $h(.)\%w$에서 $h(.)\%(2w)$로 변경
6. 복사 작업 후 버킷 내 플로우의 절반을 점진적으로 제거. 
    1. 새로운 패킷 삽입 시 해당 버킷의 플로우를 확인하여 다른 칸에 있어야 할 플로우이면 제거
    2. 제거가 안되더라도 알고리즘에 부정적인 영향을 미치지 않음 

**오버 헤드** 

- 복사 비용 : Heavy part가 작기 때문에 복사하는 시간은 무시할 정도
- 압축 : 최대 압축과 유사한 방식으로 압축. 카운터가 아닌 ($key, vote^{+}, flag, vote^{-}$)을 병합. 두 키에 대해 빈도를 조회하여 더 큰 것을 유지하고 다른 하나는 Light part로 evict

## Optimization

**Light part** 

d=1 CM 스케치 사용 : 정확도보다 구현 가능성과 속도를 중시하며, 충분히 정확함 

### Hardware Version of Elastic Sketches

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-02-05%20ElaSketch/image-6.png)

엘리펀트 충돌을 줄이기 위해 여러 개의 서브 테이블을 사용. Heavy part와 동일하지만 서로 다른 해시 함수를 사용. 각 서브 테이블이 동일한 연산을 수행하므로 하드웨어 플랫폼에 적합 

**동작 흐름** 

1.  $f_{8}$을 삽입할 때, 첫 번째 서브 테이블에서 $vote^{-}$가 1 증가하고, $f_{8}$은 다음 서브 테이블로 삽입
2. 첫 번째 서브 테이블에 $f_{9}$를 삽입할 때, 플로우 크기가 7인 $f_{4}$가 방출되어 다음 단계로 삽입. 두 번째 서브 테이블에서 $f_{4}$는 이미 $f_{4}$가 있는 버킷에 매핑. 값을 2에서 9로 증가
3. 플로우를 조회할 때는 여러 Heavy part의 모든 값을 더함

### Software Version of Elastic Sketches

![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-02-05%20ElaSketch/image-7.png)

엘리펀트 충돌을 줄이기 위해 각 버킷에 여러 개의 플로우 저장. 버킷 크기가 Word 단위보다 커질 수 있어서, Heavy packet에 접근하는 과정이 병목 지점

CPU 플랫폼에서 SIMD를 사용하여 가속화할 수 있어서 소프트웨어 플랫폼에 적합 

**동작 흐름** 

1. 각 버킷의 모든 플로우가 하나의 $vote^{-}$ 필드를 공유
2. 플로우 $f_{8}$인 패킷이 들어왔을 때, $vote^{-}$$를 증가. $11 \le 11 * \lambda = 11 * 8$이므로, $f_{8}$을 Light part애 삽입 
3. 플로우 $f_{9}$인 패킷이 들어왔을 때, $vote^{-}$$를 증가. $56 \ge 7 * \lambda = 7 * 8$이므로, 플로우 크기가 가장 작은 $f_{4}$을 evict 

## Application

**플로우 크기 추정**

플래그가 false인 플로우들에 대해서는 추정 오차가 없음. 실험 결과에 따르면, 250만 개의 패킷에 대해 600KB의 메모리를 사용할 때 헤비 파트(heavy part)에 있는 플로우의 56.6% 이상이 오차가 없음

**Heavy Hitter 탐지**

Heavy part에 플로우 ID를 직접 기록하기 때문에 높은 정확도로 Heavy Heater 탐지. Light part에서 교체된 아주 적은 부분의 플로우들만이 오차를 가짐

**Heavy Change 탐지**  

각 시간 창에서 크기가 임계값 이상인 모든 플로우를 확인하는 방식으로 탐지. 두 시간창의 Heavy part를 조회한 후 크기 차이가 임계값보다 큰 경우 탐지

**플로우 크기 분포, 엔트로피, 카디널리티 추정**

1. 플로우 크기 분포 추정 : MRAC 알고리즘을 사용하여 Light part의 분포를 추정한 후, Heavy part의 분포와 합산 
2. 엔트로피 추정 : 플로우 크기 분포를 바탕으로 $-\sum(i * \frac{n_{i}}{m} \log \frac{n_{i}}{m})$ 방식으로 계산. 
3. 카디널리티 추정 : Heavy part에 있는 고유 플로우 개수를 세고 Linear counting 방법을 사용하여 Light part에 있는 고유한 플로우의 개수와 합산 

## Implementation

### Hardware Version Implementations

**P4**

- data plane의 레지스터를 사용하여 Heavy part와 Light part 구현
- Stateful ALU를 활용하여 레지스터 배열의 항목을 조회하고 업데이트
- Stateful ALU 리소스 제한
    - 3개의 필드($vote_{all}, key, vote^{+}$)만 사용
    - $\frac{vote_{all}}{vote^{+}} \ge \lambda'$인 경우에 eviction. $\lambda' = 32$를 권장
    - 한 플로우($f, vote^{+}$)가 다른 플로우$(f_{1}, vote^{+}{1}$*)*에 의해 방출될 때, 버킷을 *($f{1}, vote^{+} + vote^{+}_{1}$*)로 설정

스위치 ASIC의 전체 리소스 사용량을 6% 미만으로 유지하면서 Line-rate 처리가 가능함을 입증

**FPGA**

Altera Stratix V 모델에 구현되었으며, 온칩 RAM의 4%와 전체 핀의 4%만을 사용하여 162.6 Mpps의 처리 속도를 달성

**GPU**

NVIDIA GTX 1080에서 CUDA를 사용하여 구현했으며, 배치 처리(Batch processing)와 멀티 스트리밍 기술로 삽입 속도를 가속화

### Software Version Implementations

**CPU 및 멀티코어**

- 한 버킷에 여러 플로우를 저장하는 소프트웨어 버전을 적용
- CPU의 **SIMD** 명령어를 활용하여 큰 버킷 데이터를 병렬로 처리함으로써 메모리 접근 병목 현상을 해결

**OVS (Open vSwitch)**

가상 스위치 환경에 통합되었으며, 8개 스레드를 사용할 때 OVS 전체 처리량에 거의 영향을 주지 않는 고성능

## Experimental results

- Setup
    - Trace : CAIDA에서 수집한 Equinix-Chicago 모니터의 1시간짜리 공공 트래픽 트레이스 4개를 사용.
    - 지표 : 플로우 크기 추정에는 ARE(평균 상대 오차), Heavy Heater 및 Change 탐지에는 F1 스코어, 그리고 속도 측정에는 처리량(Mpps)을 사용

**Accuracy** 

- 플로우 크기 추정: 600KB의 메모리를 사용할 때, 엘라스틱 스케치의 ARE는 CM, CU, Count 스케치보다 각각 약 3.8배, 2.5배, 7.5배 더 낮습니다.
- Heavy Heater 탐지 : 200KB 미만의 메모리에서도 엘라스틱 스케치는 100%의 정밀도와 재현율을 달성
- Heavy Change 탐지 : 99.5% 이상의 F1 score 달성. 다른 알고리즘들의 최고 기록은 97%

**Memory and Bandwidth Usage**

Heavy Change 탐지에서 99%의 정밀도와 재현율을 달성하기 위해 단 150KB의 대역폭만을 사용

**Elasticity**

- 대역폭 적응성 : 최대 압축 방식이 합산 압축보다 1.24배에서 2.38배 더 정확
- 패킷 속도 적응성 : 패킷 손실 없이 약 50Mpps의 패킷 속도를 견딜 수 있으며, Light part가 없는 quick mode에서는 약 70Mpps까지 가능
- 트래픽 분포 적응성
    
    ![image.png](https://yunseo10987.github.io/assets/images/posts_img/2026-02-05%20ElaSketch/image-8.png)
    

## Conclusion & My Opinion

엘라스틱 스케치가 트래픽 특성이 변할 때 잘 작동하며, 속도와 정확도 모든 면에서 최신 기술들을 능가함

개인적인 생각으로는 복사 후 점진적 청소 과정과 패킷 속도가 빠르게 오는 경우는 어떻게 할 지…. 오래걸릴 거 같은데…