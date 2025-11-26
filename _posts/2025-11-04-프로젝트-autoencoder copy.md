---
title: "[2025-11-26] AutoEncoder 코드 분석"
excerpt: "설명충 프로젝트"

categories:
  - Project
tags:
  - [NetWork]

permalink: /Project/[2025-11-26] AutoEncoder 코드 분석/

toc: true
toc_sticky: true

date: 2025-11-26
last_modified_at: 2025-11-26
---

## ExplainableBug

- 네트워크 데이터셋

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, Dataset
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
from sklearn.preprocessing import StandardScaler
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report, confusion_matrix
from sklearn.manifold import TSNE
import argparse
import os
from sklearn.ensemble import IsolationForest

# KMP_DUPLICATE_LIT_OK 설정 (matplotlib/intel 충돌 방지)
os.environ["KMP_DUPLICATE_LIT_OK"]="TRUE"

class NetworkTrafficDataset(Dataset):
    """CSV 데이터를 위한 PyTorch 커스텀 데이터셋"""
    def __init__(self, X, y):
        self.X = torch.tensor(X, dtype=torch.float32)
        self.y = torch.tensor(y, dtype=torch.int64)
    def __len__(self):
        return len(self.y)
    def __getitem__(self, idx):
        return self.X[idx], self.y[idx] 
```

`__init__` : 데이터 x와 레이블 y를 인수로 받아 Tensor로 변환

- 데이터
    
    
    | **샘플 ID** | **Feature 1: Duration (sec)** | **Feature 2: Byte_Count** | **Feature 3: Packets** |
    | --- | --- | --- | --- |
    | **0** | 0.5 | 1200 | 10 |
    | **1** | 1.2 | 980 | 8 |
    | **2** | 20.1 | 5500 | 50 |
    
    ```python
    X_numpy = [
        [0.5, 1200, 10],   # 샘플 0
        [1.2, 980, 8],     # 샘플 1
        [20.1, 5500, 50]   # 샘플 2
    ]
    ```
    
- 레이블
    
    
    | **샘플 ID** | **레이블 (Y)** | **의미** |
    | --- | --- | --- |
    | **0** | 0 | 정상 트래픽 |
    | **1** | 0 | 정상 트래픽 |
    | **2** | 0 | 정상 트래픽 |
    
    ```python
    y_numpy = [0, 0, 0]
    ```
    

`__len__` : 데이터셋 전체 크기. X와 y의 개수가 같으므로 self.X로 해도 상관 없음.

`__getitem__` : 특정 idx에 해당하는 샘플을 하나 호출.

- AutoEncoder 모델

```python
class Autoencoder(nn.Module):
    def __init__(self, input_dim, latent_dim):
        super(Autoencoder, self).__init__()
        self.encoder = nn.Sequential(
            nn.Linear(input_dim, 64),
            nn.ReLU(),
            nn.Dropout(0.2),      # 🔹 추가
            nn.Linear(64, 16),
            nn.ReLU(),
            nn.Linear(16, latent_dim)
        )
        self.decoder = nn.Sequential(
            nn.Linear(latent_dim, 16),
            nn.ReLU(),
            nn.Linear(16, 64),
            nn.ReLU(),
            nn.Dropout(0.2),      # 🔹 추가
            nn.Linear(64, input_dim)
        )

    def forward(self, x):
        z = self.encoder(x)
        x_hat = self.decoder(z)
        return x_hat, z
```

`input_dim` : 입력 데이터의 차원을 64차원으로 선형 변환

`Dropout(0.2)`  : 훈련 중 20%의 뉴런을 무작위로 비활성화하여 과적합 방지

- 전처리 함수

```python
def preprocess_csv(file_path):
    """CSV 파일을 읽고 NaN/Inf를 처리한 뒤 특징(X)을 반환"""
    df = pd.read_csv(file_path)
    
    identifier_cols = ['filename', 'Src IP', 'Src Port', 'Dst IP', 'Dst Port', 'Protocol']
    
    cols_to_drop = [col for col in identifier_cols if col in df.columns]
    
    # 식별자 열을 제외한 순수 특징(feature)만 남김
    df_features = df.drop(columns=cols_to_drop)
    
    # 'filename' 열이 있다면 제거 (특징이 아님)
    if 'filename' in df_features.columns:
        df_features = df_features.drop(columns=['filename'])
        
    # 무한대(Inf) 값 및 NaN 처리
    df_features.replace([np.inf, -np.inf], np.nan, inplace=True)
    
    # NaN 값을 해당 열의 평균(mean)으로 대체
    for col in df_features.columns:
        if df_features[col].isna().any():
            mean_val = df_features[col].mean()
            df_features[col].fillna(mean_val, inplace=True)
            
    return df_features, df
```

`df = pd.read_csv()` : csv 파일 경로를 읽어서 DataFrame으로 메모리에 로드

`identifier_cols = []` : 식별자 특징 

`cols_to_drop` : DataFrame의 feature들 중 식별자 feature 필터링

`df_features` : 식별자 feature를 제거한 DataFrame 

`if` : 제거 안되면 다시 제거 

`df_features.replace` : 양/음의 무한대를 결측값 nan으로 처리

`for()` : 열을 돌면서 만약 열 중에 nan 값이 있다면 해당 열의 평균 값으로 대체

- 메인 함수

```python
def main():
    parser = argparse.ArgumentParser(description="Train Autoencoder (PyTorch) for Anomaly Detection.")
    parser.add_argument('-n', '--normal_csv', required=True, 
                        help="Path to the NORMAL traffic CSV file (e.g., all_features.csv).")
    parser.add_argument('-m', '--malicious_csv', required=True, 
                        help="Path to the MALICIOUS traffic CSV file (e.g., malware-traffic-features.csv).")
    args = parser.parse_args()

    # --- 1. 데이터 로드 ---
    print(f"Loading normal data from: {args.normal_csv}")
    X_normal_df, _ = preprocess_csv(args.normal_csv) # 정상 데이터는 원본 식별자 불필요
    
    print(f"Loading malicious data from: {args.malicious_csv}")
    X_malicious_df, X_malicious_df_orig = preprocess_csv(args.malicious_csv)

    # --- 2. 특징(컬럼) 정렬 (중요) ---
    # 두 CSV의 컬럼 순서와 개수를 동일하게 맞춤
    common_cols = list(set(X_normal_df.columns) & set(X_malicious_df.columns))
    print(f"Found {len(common_cols)} common features.")
    
    X_normal_df = X_normal_df[common_cols]
    X_malicious_df = X_malicious_df[common_cols]
    
    input_dim = len(common_cols) # Autoencoder 입력 차원

    # Pandas DataFrame을 Numpy 배열로 변환
    X_normal_raw = X_normal_df.values
    X_malicious_raw = X_malicious_df.values
    
    print(f"Normal samples: {len(X_normal_raw)}")
    print(f"Malicious samples: {len(X_malicious_raw)}")
```

`parser ~ arg` : 실행시킬 때 만드는 인자 받고 배열화

`common_cols` : 정상 데이터와 악성 데이터의 공통 열 추출

`X_normal_df = X_normal_df[common_cols]` : DataFrame 라이브러리의 문법에 따라 [] 안에 리스트가 들어가면 해당 열로만 필터링

`X_normal_raw = X_normal_df.values` : value 들을 2차원 배열로 변경

```python
 # --- 3. 데이터셋 분리 (핵심 로직) ---
    
    # 3.1. 정상 데이터를 훈련(90%) / 테스트(10%)용으로 분리
    X_train_val_normal_raw, X_test_normal_raw = train_test_split(X_normal_raw, test_size=0.1, random_state=42)
    
    # 3.2. 훈련용(90%) 데이터를 다시 훈련(90%) / 검증(10%)용으로 분리
    # (164k * 0.9) * 0.9 = ~133k (Train)
    # (164k * 0.9) * 0.1 = ~15k (Validation)
    X_train_normal_raw, X_val_normal_raw = train_test_split(X_train_val_normal_raw, test_size=0.1, random_state=42)

```

`train_test_split()` : 데이터셋을 훈련용과 테스트용으로 분리하는 함수. 

- `random_state`를 고정하면 언제 실행해도 같은 결과를 얻음.
- `shuffle` : 기본값은 True로, 데이터 분할하기 전 무작위로 섞을 지를 지정
- array를 두 개 넣어서 동시에 분할 할 수도 있음

```python
# --- 4. 스케일링 (중요) ---
    # Scaler를 오직 '정상 훈련 데이터'로만 fit
    scaler = StandardScaler()
    scaler.fit(X_train_normal_raw)
    
    # 스케일러 적용
    X_train_scaled = scaler.transform(X_train_normal_raw)
    X_val_scaled = scaler.transform(X_val_normal_raw)
    X_test_normal_scaled = scaler.transform(X_test_normal_raw)
    X_malicious_scaled = scaler.transform(X_malicious_raw)
    
    print(f"Training samples (Normal): {len(X_train_scaled)}")
    print(f"Validation samples (Normal): {len(X_val_scaled)}")
```

`StandardScaler()` : 평균을 0, 표준편차를 1로 조정하는 스케일러 

`scaler.fit()` : 정상 훈련 데이터를 사용하여 scaler를 학습. 위의 코드에서는 정상 훈련 데이터의 평균과 표준편차를 구한 후 `transform()` 할 때, 해당 값들을 사용하여 표준화  

`scaler.transform()` : 각 데이터들을 표준화함.

```python
# --- 5. PyTorch 데이터셋 및 로더 생성 ---
    
    # 5.1. 학습용 (Train/Validation) 로더 (오직 정상 데이터)
    train_dataset = NetworkTrafficDataset(X_train_scaled, np.zeros(len(X_train_scaled)))
    val_dataset = NetworkTrafficDataset(X_val_scaled, np.zeros(len(X_val_scaled)))
    
    train_loader = DataLoader(train_dataset, batch_size=32, shuffle=True)
    val_loader = DataLoader(val_dataset, batch_size=1000, shuffle=False)

    # 5.2. 최종 평가용 (Test) 로더 (정상 10% + 악성 100%)
    # 분리해둔 정상 테스트셋과 악성 테스트셋을 합침
    X_test_combined = np.concatenate((X_test_normal_scaled, X_malicious_scaled), axis=0)
    
    # 레이블 생성 (정상: 0, 악성: 1)
    y_test_normal = np.zeros(len(X_test_normal_scaled))
    y_malicious = np.ones(len(X_malicious_scaled))
    y_test_combined = np.concatenate((y_test_normal, y_malicious), axis=0)
    
    test_dataset = NetworkTrafficDataset(X_test_combined, y_test_combined)
    test_loader = DataLoader(test_dataset, batch_size=1000, shuffle=False)
    
    print(f"Test samples: {len(X_test_combined)} (Normal: {len(y_test_normal)}, Malicious: {len(y_malicious)})")
```

`train_dataset` : `NetworkTrafficDataset` 을 통해 Tensor로 변환. 레이블은 해당 데이터셋 길이 만큼 모든 샘플에 0을 부여

- 학습 시에는 사용되지 않음????

`train_loader` : 데이터를 배치 사이즈만큼 반복하여 제공 받음. shuffle을 통해 데이터의 순서를 무작위로 섞어서 일반화 성능을 높임 

```python

    # ===============================
    # --- 6. AutoEncoder 학습 진행 ---
    # ===============================

    output_dim = 4
    print(f"\n===== Training Autoencoder with {input_dim}D Input -> {output_dim}D Latent Space =====")    

    model = Autoencoder(input_dim=input_dim, latent_dim=output_dim)
    criterion = nn.MSELoss()
    optimizer = optim.Adam(model.parameters(), lr=1e-4, weight_decay=1e-5) 
    num_epochs = 45 

    train_losses = []
    val_losses = []

    for epoch in range(num_epochs):
        model.train()
        running_train_loss = 0.0
        for features, _ in train_loader:
            output, _ = model(features)
            loss = criterion(output, features)
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
            running_train_loss += loss.item()
        
        epoch_train_loss = running_train_loss / len(train_loader)
        train_losses.append(epoch_train_loss)

        # --- 검증 단계 ---
        model.eval()
        running_val_loss = 0.0
        with torch.no_grad():
            for features, _ in val_loader:
                output, _ = model(features)
                loss = criterion(output, features)
                running_val_loss += loss.item()
        
        epoch_val_loss = running_val_loss / len(val_loader)
        val_losses.append(epoch_val_loss)

        if (epoch+1) % 5 == 0: # 5 에포크마다 출력
            print(f'Epoch [{epoch+1}/{num_epochs}], Train Loss: {epoch_train_loss:.6f}, Val Loss: {epoch_val_loss:.6f}')

    print("Training Complete")

    # 학습 완료 후
    print("Training Complete... Saving model.")
    torch.save(model.state_dict(), "autoencoder_model.pth")

if __name__ == "__main__":
    main()
```