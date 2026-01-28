# 머신러닝 개발 트러블슈팅 가이드 🤖

## 목차
1. [데이터 전처리 문제](#1-데이터-전처리-문제)
2. [모델 학습 문제](#2-모델-학습-문제)
3. [메모리 및 성능 최적화](#3-메모리-및-성능-최적화)
4. [GPU 활용 문제](#4-gpu-활용-문제)
5. [과적합 및 과소적합](#5-과적합-및-과소적합)
6. [모델 배포 문제](#6-모델-배포-문제)
7. [딥러닝 프레임워크별 문제](#7-딥러닝-프레임워크별-문제)
8. [실전 예제 시나리오](#8-실전-예제-시나리오)

---

## 1. 데이터 전처리 문제

### 1.1 결측치 처리

#### 문제: 결측치로 인한 학습 실패

```python
import pandas as pd
import numpy as np
from sklearn.model_selection import train_test_split

# 데이터 로드
df = pd.read_csv('data.csv')

# 에러: ValueError: Input contains NaN
from sklearn.linear_model import LogisticRegression
model = LogisticRegression()
model.fit(X_train, y_train)  # ❌ NaN 값 때문에 실패
```

**해결 방법:**

```python
# 1. 결측치 확인
print(df.isnull().sum())
print(f"Total missing: {df.isnull().sum().sum()}")

# 결측치 비율 확인
missing_ratio = df.isnull().sum() / len(df) * 100
print(missing_ratio[missing_ratio > 0])

# 2. 결측치 시각화
import matplotlib.pyplot as plt
import seaborn as sns

plt.figure(figsize=(12, 6))
sns.heatmap(df.isnull(), yticklabels=False, cbar=True, cmap='viridis')
plt.title('Missing Values Heatmap')
plt.show()

# 3. 결측치 처리 전략

# 방법 1: 행 삭제 (결측치가 적을 때)
df_dropped = df.dropna()

# 방법 2: 열 삭제 (특정 열에 결측치가 많을 때)
threshold = 0.5  # 50% 이상 결측치면 삭제
df_dropped_cols = df.dropna(thresh=len(df) * threshold, axis=1)

# 방법 3: 평균/중앙값으로 대체
from sklearn.impute import SimpleImputer

# 수치형 데이터
imputer_num = SimpleImputer(strategy='mean')  # 또는 'median'
df[numeric_cols] = imputer_num.fit_transform(df[numeric_cols])

# 범주형 데이터
imputer_cat = SimpleImputer(strategy='most_frequent')
df[categorical_cols] = imputer_cat.fit_transform(df[categorical_cols])

# 방법 4: 고급 대체 (KNN, Iterative)
from sklearn.impute import KNNImputer, IterativeImputer

# KNN Imputer
knn_imputer = KNNImputer(n_neighbors=5)
df_imputed = pd.DataFrame(
    knn_imputer.fit_transform(df),
    columns=df.columns
)

# Iterative Imputer (MICE)
iter_imputer = IterativeImputer(random_state=42, max_iter=10)
df_imputed = pd.DataFrame(
    iter_imputer.fit_transform(df),
    columns=df.columns
)

# 방법 5: 결측치 표시 변수 추가
for col in df.columns:
    if df[col].isnull().any():
        df[f'{col}_missing'] = df[col].isnull().astype(int)
```

### 1.2 불균형 데이터 처리

#### 문제: 클래스 불균형으로 모델 성능 저하

```python
import pandas as pd
from sklearn.datasets import make_classification

# 불균형 데이터 생성 (예시)
X, y = make_classification(
    n_samples=1000,
    n_features=20,
    n_classes=2,
    weights=[0.95, 0.05],  # 95:5 비율
    random_state=42
)

# 클래스 분포 확인
import numpy as np
unique, counts = np.unique(y, return_counts=True)
print(dict(zip(unique, counts)))
# {0: 950, 1: 50}  # 심한 불균형!
```

**해결 방법:**

```python
from imblearn.over_sampling import SMOTE, ADASYN, RandomOverSampler
from imblearn.under_sampling import RandomUnderSampler, TomekLinks
from imblearn.combine import SMOTETomek
from collections import Counter

# 1. Over-sampling

# Random Over-sampling
ros = RandomOverSampler(random_state=42)
X_resampled, y_resampled = ros.fit_resample(X, y)
print(f"After ROS: {Counter(y_resampled)}")

# SMOTE (Synthetic Minority Over-sampling Technique)
smote = SMOTE(random_state=42)
X_resampled, y_resampled = smote.fit_resample(X, y)
print(f"After SMOTE: {Counter(y_resampled)}")

# ADASYN (Adaptive Synthetic)
adasyn = ADASYN(random_state=42)
X_resampled, y_resampled = adasyn.fit_resample(X, y)
print(f"After ADASYN: {Counter(y_resampled)}")

# 2. Under-sampling

# Random Under-sampling
rus = RandomUnderSampler(random_state=42)
X_resampled, y_resampled = rus.fit_resample(X, y)
print(f"After RUS: {Counter(y_resampled)}")

# 3. Combination (권장)
smt = SMOTETomek(random_state=42)
X_resampled, y_resampled = smt.fit_resample(X, y)
print(f"After SMOTETomek: {Counter(y_resampled)}")

# 4. Class Weight 조정 (모델 학습 시)
from sklearn.ensemble import RandomForestClassifier

# 자동 가중치
model = RandomForestClassifier(class_weight='balanced', random_state=42)
model.fit(X, y)

# 수동 가중치
class_weights = {0: 1, 1: 19}  # 클래스 1에 19배 가중치
model = RandomForestClassifier(class_weight=class_weights, random_state=42)
model.fit(X, y)

# 5. 평가 지표 변경
from sklearn.metrics import classification_report, roc_auc_score, f1_score

# Accuracy 대신 다른 지표 사용
predictions = model.predict(X_test)
print(classification_report(y_test, predictions))
print(f"ROC-AUC: {roc_auc_score(y_test, predictions)}")
print(f"F1-Score: {f1_score(y_test, predictions)}")
```

### 1.3 범주형 변수 인코딩

```python
import pandas as pd
from sklearn.preprocessing import LabelEncoder, OneHotEncoder
from category_encoders import TargetEncoder, BinaryEncoder

# 예시 데이터
df = pd.DataFrame({
    'color': ['red', 'blue', 'green', 'red', 'blue'],
    'size': ['S', 'M', 'L', 'M', 'S'],
    'city': ['Seoul', 'Busan', 'Seoul', 'Incheon', 'Busan'],
    'target': [1, 0, 1, 0, 1]
})

# 1. Label Encoding (순서가 있는 경우)
le = LabelEncoder()
df['size_encoded'] = le.fit_transform(df['size'])
print(df[['size', 'size_encoded']])

# 2. One-Hot Encoding (순서가 없는 경우)
# Pandas
df_onehot = pd.get_dummies(df, columns=['color'], prefix='color')

# Scikit-learn
from sklearn.preprocessing import OneHotEncoder
ohe = OneHotEncoder(sparse=False, handle_unknown='ignore')
color_encoded = ohe.fit_transform(df[['color']])
color_df = pd.DataFrame(
    color_encoded,
    columns=ohe.get_feature_names_out(['color'])
)

# 3. Target Encoding (카디널리티가 높을 때)
te = TargetEncoder()
df['city_encoded'] = te.fit_transform(df['city'], df['target'])

# 4. Binary Encoding (카디널리티가 매우 높을 때)
be = BinaryEncoder(cols=['city'])
df_binary = be.fit_transform(df)

# 5. Frequency Encoding
freq_encoding = df['city'].value_counts().to_dict()
df['city_freq'] = df['city'].map(freq_encoding)

# 6. 커스텀 순서 인코딩
size_order = {'S': 0, 'M': 1, 'L': 2}
df['size_ordered'] = df['size'].map(size_order)
```

---

## 2. 모델 학습 문제

### 2.1 학습이 진행되지 않음

#### 문제: Loss가 감소하지 않음

```python
import torch
import torch.nn as nn
import torch.optim as optim

# 간단한 신경망
model = nn.Sequential(
    nn.Linear(10, 64),
    nn.ReLU(),
    nn.Linear(64, 1)
)

criterion = nn.MSELoss()
optimizer = optim.SGD(model.parameters(), lr=0.01)

# 학습
for epoch in range(100):
    outputs = model(X_train)
    loss = criterion(outputs, y_train)
    
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
    
    if epoch % 10 == 0:
        print(f"Epoch {epoch}, Loss: {loss.item()}")
# 출력: Loss가 감소하지 않음! 🚨
```

**원인 및 해결:**

**원인 1: 학습률 문제**

```python
# 학습률이 너무 큼 → Loss 발산
optimizer = optim.SGD(model.parameters(), lr=10.0)  # ❌ 너무 큼

# 학습률이 너무 작음 → 학습 안 됨
optimizer = optim.SGD(model.parameters(), lr=1e-10)  # ❌ 너무 작음

# 해결: 적절한 학습률 찾기
# Learning Rate Finder
learning_rates = [1e-5, 1e-4, 1e-3, 1e-2, 1e-1]
losses = []

for lr in learning_rates:
    model = create_model()
    optimizer = optim.SGD(model.parameters(), lr=lr)
    
    # 몇 번 학습
    for _ in range(10):
        loss = train_one_epoch(model, optimizer, train_loader)
    
    losses.append(loss)

# 최적 학습률 선택
import matplotlib.pyplot as plt
plt.plot(learning_rates, losses)
plt.xscale('log')
plt.xlabel('Learning Rate')
plt.ylabel('Loss')
plt.title('Learning Rate Finder')
plt.show()

# 또는 Learning Rate Scheduler 사용
from torch.optim.lr_scheduler import ReduceLROnPlateau, CosineAnnealingLR

# 성능이 향상되지 않으면 학습률 감소
scheduler = ReduceLROnPlateau(optimizer, mode='min', factor=0.5, patience=5)

for epoch in range(100):
    train_loss = train(model, train_loader, optimizer)
    val_loss = validate(model, val_loader)
    
    # Scheduler 업데이트
    scheduler.step(val_loss)
```

**원인 2: Gradient Vanishing/Exploding**

```python
# Gradient 확인
def check_gradients(model):
    for name, param in model.named_parameters():
        if param.grad is not None:
            grad_norm = param.grad.norm().item()
            print(f"{name}: {grad_norm}")

# Gradient Clipping
torch.nn.utils.clip_grad_norm_(model.parameters(), max_norm=1.0)

# 또는
torch.nn.utils.clip_grad_value_(model.parameters(), clip_value=0.5)

# Batch Normalization 추가
model = nn.Sequential(
    nn.Linear(10, 64),
    nn.BatchNorm1d(64),  # ✅ 추가
    nn.ReLU(),
    nn.Linear(64, 32),
    nn.BatchNorm1d(32),  # ✅ 추가
    nn.ReLU(),
    nn.Linear(32, 1)
)

# 활성화 함수 변경
# Sigmoid/Tanh 대신 ReLU, LeakyReLU, ELU 사용
model = nn.Sequential(
    nn.Linear(10, 64),
    nn.LeakyReLU(0.01),  # ✅ LeakyReLU
    nn.Linear(64, 1)
)
```

**원인 3: 데이터 정규화 안 됨**

```python
from sklearn.preprocessing import StandardScaler, MinMaxScaler

# 정규화 전
X_train_raw = train_data

# StandardScaler (평균 0, 표준편차 1)
scaler = StandardScaler()
X_train_scaled = scaler.fit_transform(X_train_raw)
X_test_scaled = scaler.transform(X_test_raw)

# MinMaxScaler (0~1 범위)
scaler = MinMaxScaler()
X_train_scaled = scaler.fit_transform(X_train_raw)

# 주의: Test 데이터는 Train 데이터의 통계로 변환!
# ❌ 잘못된 예
scaler.fit(X_test)  # Train과 다른 통계 사용

# ✅ 올바른 예
scaler.fit(X_train)
X_test_scaled = scaler.transform(X_test)
```

### 2.2 과적합 (Overfitting)

#### 증상: Training accuracy는 높지만 Validation accuracy가 낮음

```python
# 학습 결과
# Epoch 50: Train Loss: 0.05, Train Acc: 0.98
#          Val Loss: 0.45, Val Acc: 0.75  # ❌ 큰 차이!
```

**해결 방법:**

```python
import torch.nn as nn
from torch.utils.data import DataLoader

# 1. Dropout 추가
model = nn.Sequential(
    nn.Linear(100, 256),
    nn.ReLU(),
    nn.Dropout(0.5),  # ✅ 50% 드롭아웃
    nn.Linear(256, 128),
    nn.ReLU(),
    nn.Dropout(0.3),  # ✅ 30% 드롭아웃
    nn.Linear(128, 10)
)

# 2. L2 Regularization (Weight Decay)
optimizer = optim.Adam(model.parameters(), lr=0.001, weight_decay=1e-4)  # ✅

# 3. Early Stopping
class EarlyStopping:
    def __init__(self, patience=7, min_delta=0):
        self.patience = patience
        self.min_delta = min_delta
        self.counter = 0
        self.best_loss = None
        self.early_stop = False
    
    def __call__(self, val_loss):
        if self.best_loss is None:
            self.best_loss = val_loss
        elif val_loss > self.best_loss - self.min_delta:
            self.counter += 1
            if self.counter >= self.patience:
                self.early_stop = True
        else:
            self.best_loss = val_loss
            self.counter = 0

early_stopping = EarlyStopping(patience=10)

for epoch in range(1000):
    train_loss = train(model, train_loader)
    val_loss = validate(model, val_loader)
    
    early_stopping(val_loss)
    if early_stopping.early_stop:
        print(f"Early stopping at epoch {epoch}")
        break

# 4. 데이터 증강 (Data Augmentation)
from torchvision import transforms

train_transform = transforms.Compose([
    transforms.RandomHorizontalFlip(),
    transforms.RandomRotation(10),
    transforms.ColorJitter(brightness=0.2, contrast=0.2),
    transforms.ToTensor(),
])

# 5. 모델 복잡도 줄이기
# Before (과도하게 복잡)
model_complex = nn.Sequential(
    nn.Linear(100, 1000),
    nn.ReLU(),
    nn.Linear(1000, 1000),
    nn.ReLU(),
    nn.Linear(1000, 10)
)

# After (단순화)
model_simple = nn.Sequential(
    nn.Linear(100, 128),
    nn.ReLU(),
    nn.Linear(128, 10)
)

# 6. 더 많은 데이터 수집
# 가능하면 학습 데이터 증가
```

### 2.3 학습 시간이 너무 오래 걸림

```python
import time
from tqdm import tqdm

# 문제: 학습이 너무 느림
start_time = time.time()
for epoch in range(10):
    for batch in train_loader:
        # 학습 코드
        pass
elapsed = time.time() - start_time
print(f"Training time: {elapsed:.2f}s")  # 너무 오래 걸림!
```

**해결 방법:**

```python
# 1. Batch Size 증가
train_loader = DataLoader(
    dataset,
    batch_size=128,  # 32 → 128로 증가
    shuffle=True,
    num_workers=4  # 병렬 데이터 로딩
)

# 2. Mixed Precision Training
from torch.cuda.amp import autocast, GradScaler

scaler = GradScaler()

for epoch in range(num_epochs):
    for inputs, labels in train_loader:
        optimizer.zero_grad()
        
        # Mixed precision
        with autocast():
            outputs = model(inputs)
            loss = criterion(outputs, labels)
        
        # Gradient scaling
        scaler.scale(loss).backward()
        scaler.step(optimizer)
        scaler.update()

# 3. DataLoader 최적화
train_loader = DataLoader(
    dataset,
    batch_size=64,
    shuffle=True,
    num_workers=4,      # CPU 병렬 처리
    pin_memory=True,    # GPU 전송 최적화
    persistent_workers=True  # Worker 재사용
)

# 4. Gradient Accumulation (작은 GPU 메모리)
accumulation_steps = 4
optimizer.zero_grad()

for i, (inputs, labels) in enumerate(train_loader):
    outputs = model(inputs)
    loss = criterion(outputs, labels)
    loss = loss / accumulation_steps  # 손실 정규화
    
    loss.backward()
    
    if (i + 1) % accumulation_steps == 0:
        optimizer.step()
        optimizer.zero_grad()
```

---

## 3. 메모리 및 성능 최적화

### 3.1 GPU 메모리 부족 (OOM)

#### 문제: CUDA out of memory

```python
# 에러:
# RuntimeError: CUDA out of memory. 
# Tried to allocate 2.00 GiB (GPU 0; 7.79 GiB total capacity)
```

**해결 방법:**

```python
import torch
import gc

# 1. Batch Size 줄이기
train_loader = DataLoader(dataset, batch_size=16)  # 64 → 16

# 2. 메모리 정리
def clear_memory():
    gc.collect()
    torch.cuda.empty_cache()

# 학습 중 주기적으로 실행
for epoch in range(num_epochs):
    train(model, train_loader)
    clear_memory()

# 3. Gradient Checkpointing
from torch.utils.checkpoint import checkpoint

class MyModel(nn.Module):
    def __init__(self):
        super().__init__()
        self.layer1 = nn.Linear(1000, 1000)
        self.layer2 = nn.Linear(1000, 1000)
        self.layer3 = nn.Linear(1000, 10)
    
    def forward(self, x):
        # Gradient checkpointing 사용
        x = checkpoint(self.layer1, x)
        x = checkpoint(self.layer2, x)
        x = self.layer3(x)
        return x

# 4. Mixed Precision Training
from torch.cuda.amp import autocast

with autocast():
    outputs = model(inputs)
    loss = criterion(outputs, labels)

# 5. 메모리 사용량 모니터링
def print_gpu_memory():
    if torch.cuda.is_available():
        allocated = torch.cuda.memory_allocated() / 1024**3
        reserved = torch.cuda.memory_reserved() / 1024**3
        print(f"Allocated: {allocated:.2f} GB")
        print(f"Reserved: {reserved:.2f} GB")

print_gpu_memory()

# 6. 불필요한 연산 그래프 제거
with torch.no_grad():
    # 평가 시에는 gradient 계산 안 함
    outputs = model(inputs)

# 7. In-place 연산 사용
x = torch.randn(1000, 1000)
x.relu_()  # in-place
x.add_(5)  # in-place
```

### 3.2 학습 속도 프로파일링

```python
import torch
from torch.profiler import profile, record_function, ProfilerActivity

# 코드 프로파일링
with profile(
    activities=[ProfilerActivity.CPU, ProfilerActivity.CUDA],
    record_shapes=True,
    profile_memory=True
) as prof:
    with record_function("model_training"):
        for inputs, labels in train_loader:
            outputs = model(inputs)
            loss = criterion(outputs, labels)
            loss.backward()
            optimizer.step()
            optimizer.zero_grad()

# 결과 출력
print(prof.key_averages().table(sort_by="cuda_time_total", row_limit=10))

# TensorBoard로 시각화
prof.export_chrome_trace("trace.json")
```

---

## 4. GPU 활용 문제

### 4.1 GPU가 사용되지 않음

```python
import torch

# GPU 사용 가능 확인
print(f"CUDA available: {torch.cuda.is_available()}")
print(f"CUDA version: {torch.version.cuda}")
print(f"GPU count: {torch.cuda.device_count()}")
print(f"GPU name: {torch.cuda.get_device_name(0)}")

# 모델과 데이터를 GPU로 이동
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
print(f"Using device: {device}")

model = model.to(device)

# 학습 루프
for inputs, labels in train_loader:
    inputs = inputs.to(device)
    labels = labels.to(device)
    
    outputs = model(inputs)
    loss = criterion(outputs, labels)
    
    optimizer.zero_grad()
    loss.backward()
    optimizer.step()
```

### 4.2 Multi-GPU 학습

```python
import torch
import torch.nn as nn

# 1. DataParallel (간단하지만 느림)
if torch.cuda.device_count() > 1:
    print(f"Using {torch.cuda.device_count()} GPUs")
    model = nn.DataParallel(model)

model.to(device)

# 2. DistributedDataParallel (권장)
import torch.distributed as dist
from torch.nn.parallel import DistributedDataParallel as DDP

def setup(rank, world_size):
    dist.init_process_group("nccl", rank=rank, world_size=world_size)

def cleanup():
    dist.destroy_process_group()

def train_ddp(rank, world_size):
    setup(rank, world_size)
    
    # 모델 생성
    model = MyModel().to(rank)
    ddp_model = DDP(model, device_ids=[rank])
    
    # 학습
    for epoch in range(num_epochs):
        for inputs, labels in train_loader:
            inputs = inputs.to(rank)
            labels = labels.to(rank)
            
            outputs = ddp_model(inputs)
            loss = criterion(outputs, labels)
            
            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
    
    cleanup()

# 실행
import torch.multiprocessing as mp
world_size = torch.cuda.device_count()
mp.spawn(train_ddp, args=(world_size,), nprocs=world_size, join=True)
```

### 4.3 GPU 메모리 모니터링

```bash
# 터미널에서 GPU 상태 확인
nvidia-smi

# 실시간 모니터링
watch -n 1 nvidia-smi

# Python에서 모니터링
```

```python
import torch
import GPUtil

def monitor_gpu():
    """GPU 사용량 모니터링"""
    GPUs = GPUtil.getGPUs()
    for gpu in GPUs:
        print(f"GPU {gpu.id}: {gpu.name}")
        print(f"  Temperature: {gpu.temperature}°C")
        print(f"  Load: {gpu.load * 100}%")
        print(f"  Memory Used: {gpu.memoryUsed}MB / {gpu.memoryTotal}MB")
        print(f"  Memory Util: {gpu.memoryUtil * 100}%")

monitor_gpu()

# PyTorch GPU 메모리
if torch.cuda.is_available():
    print(f"Allocated: {torch.cuda.memory_allocated() / 1024**3:.2f} GB")
    print(f"Cached: {torch.cuda.memory_reserved() / 1024**3:.2f} GB")
    print(f"Max Allocated: {torch.cuda.max_memory_allocated() / 1024**3:.2f} GB")
```

---

## 5. 과적합 및 과소적합

### 5.1 과소적합 (Underfitting)

#### 증상: Train과 Validation 성능 모두 낮음

```python
# 결과:
# Train Acc: 0.65, Val Acc: 0.63  # 둘 다 낮음!
```

**해결 방법:**

```python
# 1. 모델 복잡도 증가
# Before
model = nn.Sequential(
    nn.Linear(100, 10)  # 너무 단순
)

# After
model = nn.Sequential(
    nn.Linear(100, 256),
    nn.ReLU(),
    nn.Linear(256, 128),
    nn.ReLU(),
    nn.Linear(128, 10)
)

# 2. 학습 시간 증가
epochs = 100  # 10 → 100

# 3. 학습률 증가
optimizer = optim.Adam(model.parameters(), lr=0.001)  # 0.0001 → 0.001

# 4. Feature Engineering
# 더 많은 feature 추가
# 다항식 feature
from sklearn.preprocessing import PolynomialFeatures
poly = PolynomialFeatures(degree=2)
X_poly = poly.fit_transform(X)

# 5. 정규화 감소
# Weight decay 줄이기
optimizer = optim.Adam(model.parameters(), lr=0.001, weight_decay=1e-6)  # 1e-4 → 1e-6

# Dropout 줄이기
nn.Dropout(0.2)  # 0.5 → 0.2
```

### 5.2 Learning Curve 분석

```python
import matplotlib.pyplot as plt
import numpy as np

def plot_learning_curves(train_losses, val_losses, train_accs, val_accs):
    """학습 곡선 시각화"""
    fig, (ax1, ax2) = plt.subplots(1, 2, figsize=(15, 5))
    
    # Loss 곡선
    ax1.plot(train_losses, label='Train Loss')
    ax1.plot(val_losses, label='Val Loss')
    ax1.set_xlabel('Epoch')
    ax1.set_ylabel('Loss')
    ax1.set_title('Loss Curves')
    ax1.legend()
    ax1.grid(True)
    
    # Accuracy 곡선
    ax2.plot(train_accs, label='Train Acc')
    ax2.plot(val_accs, label='Val Acc')
    ax2.set_xlabel('Epoch')
    ax2.set_ylabel('Accuracy')
    ax2.set_title('Accuracy Curves')
    ax2.legend()
    ax2.grid(True)
    
    plt.tight_layout()
    plt.savefig('learning_curves.png', dpi=300)
    plt.show()
    
    # 분석
    print("=== Learning Curve Analysis ===")
    
    # 과적합 체크
    final_train_val_gap = train_accs[-1] - val_accs[-1]
    if final_train_val_gap > 0.1:
        print(f"⚠️  Overfitting detected! Gap: {final_train_val_gap:.3f}")
        print("   → Try: Dropout, L2 regularization, Early stopping")
    
    # 과소적합 체크
    if train_accs[-1] < 0.8 and val_accs[-1] < 0.8:
        print("⚠️  Underfitting detected!")
        print("   → Try: Increase model complexity, More epochs")
    
    # 수렴 확인
    if len(val_losses) > 10:
        recent_improvement = np.mean(val_losses[-10:-5]) - np.mean(val_losses[-5:])
        if recent_improvement < 0.001:
            print("✓ Model has converged")

# 사용 예제
train_losses, val_losses = [], []
train_accs, val_accs = [], []

for epoch in range(num_epochs):
    train_loss, train_acc = train(model, train_loader)
    val_loss, val_acc = validate(model, val_loader)
    
    train_losses.append(train_loss)
    val_losses.append(val_loss)
    train_accs.append(train_acc)
    val_accs.append(val_acc)

plot_learning_curves(train_losses, val_losses, train_accs, val_accs)
```

---

## 6. 모델 배포 문제

### 6.1 모델 저장 및 로드

```python
import torch

# 1. 전체 모델 저장 (비권장)
torch.save(model, 'model.pth')
loaded_model = torch.load('model.pth')

# 2. State Dict 저장 (권장)
torch.save(model.state_dict(), 'model_weights.pth')

# 로드
model = MyModel()
model.load_state_dict(torch.load('model_weights.pth'))
model.eval()

# 3. Checkpoint 저장 (학습 재개용)
checkpoint = {
    'epoch': epoch,
    'model_state_dict': model.state_dict(),
    'optimizer_state_dict': optimizer.state_dict(),
    'loss': loss,
    'train_losses': train_losses,
    'val_losses': val_losses
}
torch.save(checkpoint, 'checkpoint.pth')

# Checkpoint 로드
checkpoint = torch.load('checkpoint.pth')
model.load_state_dict(checkpoint['model_state_dict'])
optimizer.load_state_dict(checkpoint['optimizer_state_dict'])
start_epoch = checkpoint['epoch']
loss = checkpoint['loss']

# 4. ONNX로 변환 (다른 프레임워크 사용)
dummy_input = torch.randn(1, 3, 224, 224)
torch.onnx.export(
    model,
    dummy_input,
    "model.onnx",
    export_params=True,
    opset_version=11,
    input_names=['input'],
    output_names=['output']
)

# 5. TorchScript로 변환 (최적화된 배포)
scripted_model = torch.jit.script(model)
scripted_model.save('model_scripted.pt')

# 로드
loaded_scripted = torch.jit.load('model_scripted.pt')
```

### 6.2 추론 최적화

```python
import torch

# 1. 추론 모드
model.eval()

with torch.no_grad():
    outputs = model(inputs)

# 2. TorchScript (JIT 컴파일)
scripted_model = torch.jit.trace(model, example_input)
scripted_model = torch.jit.script(model)

# 3. 양자화 (Quantization)
# Dynamic Quantization
quantized_model = torch.quantization.quantize_dynamic(
    model,
    {torch.nn.Linear},  # 양자화할 레이어
    dtype=torch.qint8
)

# Static Quantization
model.qconfig = torch.quantization.get_default_qconfig('fbgemm')
torch.quantization.prepare(model, inplace=True)
# Calibration
torch.quantization.convert(model, inplace=True)

# 4. 모델 크기 비교
def get_model_size(model):
    torch.save(model.state_dict(), "temp.p")
    size = os.path.getsize("temp.p") / 1e6  # MB
    os.remove("temp.p")
    return size

print(f"Original model: {get_model_size(model):.2f} MB")
print(f"Quantized model: {get_model_size(quantized_model):.2f} MB")

# 5. 배치 추론
def batch_predict(model, data_loader, device):
    """배치로 효율적인 추론"""
    model.eval()
    predictions = []
    
    with torch.no_grad():
        for batch in data_loader:
            batch = batch.to(device)
            output = model(batch)
            predictions.extend(output.cpu().numpy())
    
    return np.array(predictions)
```

### 6.3 모델 서빙 (FastAPI 예제)

```python
from fastapi import FastAPI, File, UploadFile
from pydantic import BaseModel
import torch
import numpy as np
from PIL import Image
import io

app = FastAPI()

# 모델 로드 (서버 시작 시 한 번만)
model = torch.load('model.pth')
model.eval()
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = model.to(device)

class PredictionInput(BaseModel):
    features: list

@app.post("/predict")
async def predict(input_data: PredictionInput):
    """REST API 예측"""
    # 입력 처리
    features = torch.tensor([input_data.features], dtype=torch.float32).to(device)
    
    # 추론
    with torch.no_grad():
        output = model(features)
        prediction = output.argmax(dim=1).item()
        confidence = torch.softmax(output, dim=1)[0][prediction].item()
    
    return {
        "prediction": int(prediction),
        "confidence": float(confidence)
    }

@app.post("/predict_image")
async def predict_image(file: UploadFile = File(...)):
    """이미지 예측"""
    # 이미지 읽기
    image_data = await file.read()
    image = Image.open(io.BytesIO(image_data))
    
    # 전처리
    transform = transforms.Compose([
        transforms.Resize((224, 224)),
        transforms.ToTensor(),
        transforms.Normalize(mean=[0.485, 0.456, 0.406], 
                           std=[0.229, 0.224, 0.225])
    ])
    image_tensor = transform(image).unsqueeze(0).to(device)
    
    # 추론
    with torch.no_grad():
        output = model(image_tensor)
        prediction = output.argmax(dim=1).item()
        confidence = torch.softmax(output, dim=1)[0][prediction].item()
    
    return {
        "prediction": int(prediction),
        "confidence": float(confidence)
    }

# 실행: uvicorn main:app --reload
```

---

## 7. 딥러닝 프레임워크별 문제

### 7.1 PyTorch 특정 문제

```python
import torch

# 1. CUDA 오류: "RuntimeError: Expected all tensors to be on the same device"
# 원인: 텐서가 다른 디바이스에 있음
model = model.to(device)
inputs = inputs.to(device)  # ✅ 모든 텐서를 같은 디바이스로

# 2. "RuntimeError: Expected object of backend CPU but got backend CUDA"
# 원인: CPU 텐서와 CUDA 텐서 혼용
cpu_tensor = torch.randn(10)
cuda_tensor = torch.randn(10).cuda()
result = cpu_tensor + cuda_tensor  # ❌ 에러!

# 해결
result = cpu_tensor.cuda() + cuda_tensor  # ✅

# 3. Inplace 연산 에러
x = torch.randn(10, requires_grad=True)
x += 1  # ❌ inplace 연산은 gradient 계산에 문제
x = x + 1  # ✅

# 4. DataLoader num_workers 문제
# Windows에서 num_workers > 0이면 오류 발생 가능
if __name__ == '__main__':
    train_loader = DataLoader(
        dataset,
        batch_size=32,
        num_workers=4
    )
```

### 7.2 TensorFlow/Keras 특정 문제

```python
import tensorflow as tf
from tensorflow import keras

# 1. GPU 메모리 증가 설정
gpus = tf.config.experimental.list_physical_devices('GPU')
if gpus:
    try:
        for gpu in gpus:
            tf.config.experimental.set_memory_growth(gpu, True)
    except RuntimeError as e:
        print(e)

# 2. Mixed Precision
from tensorflow.keras import mixed_precision
policy = mixed_precision.Policy('mixed_float16')
mixed_precision.set_global_policy(policy)

# 3. 모델 저장 및 로드
# 전체 모델 저장
model.save('model.h5')
model = keras.models.load_model('model.h5')

# Weights만 저장
model.save_weights('weights.h5')
model.load_weights('weights.h5')

# 4. Custom Layer 직렬화
class CustomLayer(keras.layers.Layer):
    def get_config(self):
        config = super().get_config()
        config.update({"my_param": self.my_param})
        return config

# 5. 학습률 스케줄러
lr_schedule = keras.optimizers.schedules.ExponentialDecay(
    initial_learning_rate=1e-3,
    decay_steps=10000,
    decay_rate=0.9
)
optimizer = keras.optimizers.Adam(learning_rate=lr_schedule)
```

---

## 8. 실전 예제 시나리오

### 시나리오 1: 이미지 분류 모델 학습 완전 가이드

```python
import torch
import torch.nn as nn
import torch.optim as optim
from torch.utils.data import DataLoader, Dataset
from torchvision import transforms, models
from sklearn.model_selection import train_test_split
import numpy as np
from tqdm import tqdm
import matplotlib.pyplot as plt

# 1. 데이터셋 클래스 정의
class CustomImageDataset(Dataset):
    def __init__(self, image_paths, labels, transform=None):
        self.image_paths = image_paths
        self.labels = labels
        self.transform = transform
    
    def __len__(self):
        return len(self.image_paths)
    
    def __getitem__(self, idx):
        from PIL import Image
        image = Image.open(self.image_paths[idx]).convert('RGB')
        label = self.labels[idx]
        
        if self.transform:
            image = self.transform(image)
        
        return image, label

# 2. 데이터 증강 및 전처리
train_transform = transforms.Compose([
    transforms.Resize((256, 256)),
    transforms.RandomCrop(224),
    transforms.RandomHorizontalFlip(),
    transforms.RandomRotation(15),
    transforms.ColorJitter(brightness=0.2, contrast=0.2, saturation=0.2),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], 
                       std=[0.229, 0.224, 0.225])
])

val_transform = transforms.Compose([
    transforms.Resize((256, 256)),
    transforms.CenterCrop(224),
    transforms.ToTensor(),
    transforms.Normalize(mean=[0.485, 0.456, 0.406], 
                       std=[0.229, 0.224, 0.225])
])

# 3. 데이터셋 생성
# image_paths와 labels는 실제 데이터로 대체
train_paths, val_paths, train_labels, val_labels = train_test_split(
    image_paths, labels, test_size=0.2, random_state=42, stratify=labels
)

train_dataset = CustomImageDataset(train_paths, train_labels, train_transform)
val_dataset = CustomImageDataset(val_paths, val_labels, val_transform)

train_loader = DataLoader(train_dataset, batch_size=32, shuffle=True, 
                         num_workers=4, pin_memory=True)
val_loader = DataLoader(val_dataset, batch_size=32, shuffle=False, 
                       num_workers=4, pin_memory=True)

# 4. 모델 정의 (전이 학습)
model = models.resnet50(pretrained=True)

# 마지막 레이어 교체
num_classes = 10
model.fc = nn.Linear(model.fc.in_features, num_classes)

# 5. 디바이스 설정
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
model = model.to(device)

# 6. 손실 함수 및 옵티마이저
criterion = nn.CrossEntropyLoss()
optimizer = optim.Adam(model.parameters(), lr=0.001, weight_decay=1e-4)

# 학습률 스케줄러
scheduler = optim.lr_scheduler.ReduceLROnPlateau(
    optimizer, mode='min', factor=0.5, patience=5, verbose=True
)

# 7. 학습 및 검증 함수
def train_one_epoch(model, train_loader, criterion, optimizer, device):
    model.train()
    running_loss = 0.0
    correct = 0
    total = 0
    
    pbar = tqdm(train_loader, desc='Training')
    for inputs, labels in pbar:
        inputs, labels = inputs.to(device), labels.to(device)
        
        optimizer.zero_grad()
        outputs = model(inputs)
        loss = criterion(outputs, labels)
        loss.backward()
        optimizer.step()
        
        running_loss += loss.item()
        _, predicted = outputs.max(1)
        total += labels.size(0)
        correct += predicted.eq(labels).sum().item()
        
        pbar.set_postfix({
            'loss': f'{loss.item():.4f}',
            'acc': f'{100.*correct/total:.2f}%'
        })
    
    epoch_loss = running_loss / len(train_loader)
    epoch_acc = 100. * correct / total
    
    return epoch_loss, epoch_acc

def validate(model, val_loader, criterion, device):
    model.eval()
    running_loss = 0.0
    correct = 0
    total = 0
    
    with torch.no_grad():
        pbar = tqdm(val_loader, desc='Validation')
        for inputs, labels in pbar:
            inputs, labels = inputs.to(device), labels.to(device)
            
            outputs = model(inputs)
            loss = criterion(outputs, labels)
            
            running_loss += loss.item()
            _, predicted = outputs.max(1)
            total += labels.size(0)
            correct += predicted.eq(labels).sum().item()
            
            pbar.set_postfix({
                'loss': f'{loss.item():.4f}',
                'acc': f'{100.*correct/total:.2f}%'
            })
    
    epoch_loss = running_loss / len(val_loader)
    epoch_acc = 100. * correct / total
    
    return epoch_loss, epoch_acc

# 8. 학습 루프
num_epochs = 50
best_val_acc = 0
train_losses, val_losses = [], []
train_accs, val_accs = [], []

for epoch in range(num_epochs):
    print(f'\nEpoch {epoch+1}/{num_epochs}')
    print('-' * 50)
    
    train_loss, train_acc = train_one_epoch(
        model, train_loader, criterion, optimizer, device
    )
    val_loss, val_acc = validate(model, val_loader, criterion, device)
    
    # 기록
    train_losses.append(train_loss)
    val_losses.append(val_loss)
    train_accs.append(train_acc)
    val_accs.append(val_acc)
    
    print(f'Train Loss: {train_loss:.4f}, Train Acc: {train_acc:.2f}%')
    print(f'Val Loss: {val_loss:.4f}, Val Acc: {val_acc:.2f}%')
    
    # 스케줄러 업데이트
    scheduler.step(val_loss)
    
    # Best 모델 저장
    if val_acc > best_val_acc:
        best_val_acc = val_acc
        torch.save({
            'epoch': epoch,
            'model_state_dict': model.state_dict(),
            'optimizer_state_dict': optimizer.state_dict(),
            'best_val_acc': best_val_acc,
        }, 'best_model.pth')
        print(f'✓ Best model saved! (Val Acc: {val_acc:.2f}%)')

# 9. 학습 곡선 시각화
plt.figure(figsize=(15, 5))

plt.subplot(1, 2, 1)
plt.plot(train_losses, label='Train Loss')
plt.plot(val_losses, label='Val Loss')
plt.xlabel('Epoch')
plt.ylabel('Loss')
plt.title('Loss Curves')
plt.legend()
plt.grid(True)

plt.subplot(1, 2, 2)
plt.plot(train_accs, label='Train Acc')
plt.plot(val_accs, label='Val Acc')
plt.xlabel('Epoch')
plt.ylabel('Accuracy (%)')
plt.title('Accuracy Curves')
plt.legend()
plt.grid(True)

plt.tight_layout()
plt.savefig('training_history.png', dpi=300)
plt.show()

print(f'\nTraining completed!')
print(f'Best validation accuracy: {best_val_acc:.2f}%')
```

### 시나리오 2: 하이퍼파라미터 튜닝

```python
import optuna
from optuna.visualization import plot_optimization_history, plot_param_importances

def objective(trial):
    """Optuna objective 함수"""
    
    # 하이퍼파라미터 제안
    lr = trial.suggest_loguniform('lr', 1e-5, 1e-2)
    batch_size = trial.suggest_categorical('batch_size', [16, 32, 64, 128])
    weight_decay = trial.suggest_loguniform('weight_decay', 1e-6, 1e-3)
    dropout_rate = trial.suggest_uniform('dropout', 0.1, 0.5)
    hidden_size = trial.suggest_int('hidden_size', 64, 512, step=64)
    
    # 모델 생성
    model = nn.Sequential(
        nn.Linear(input_size, hidden_size),
        nn.ReLU(),
        nn.Dropout(dropout_rate),
        nn.Linear(hidden_size, hidden_size // 2),
        nn.ReLU(),
        nn.Dropout(dropout_rate),
        nn.Linear(hidden_size // 2, num_classes)
    ).to(device)
    
    # DataLoader
    train_loader = DataLoader(train_dataset, batch_size=batch_size, shuffle=True)
    val_loader = DataLoader(val_dataset, batch_size=batch_size, shuffle=False)
    
    # Optimizer
    optimizer = optim.Adam(model.parameters(), lr=lr, weight_decay=weight_decay)
    criterion = nn.CrossEntropyLoss()
    
    # 짧은 학습 (10 epochs)
    for epoch in range(10):
        model.train()
        for inputs, labels in train_loader:
            inputs, labels = inputs.to(device), labels.to(device)
            
            optimizer.zero_grad()
            outputs = model(inputs)
            loss = criterion(outputs, labels)
            loss.backward()
            optimizer.step()
    
    # 검증
    model.eval()
    correct = 0
    total = 0
    
    with torch.no_grad():
        for inputs, labels in val_loader:
            inputs, labels = inputs.to(device), labels.to(device)
            outputs = model(inputs)
            _, predicted = outputs.max(1)
            total += labels.size(0)
            correct += predicted.eq(labels).sum().item()
    
    accuracy = correct / total
    
    return accuracy

# Optuna Study 생성
study = optuna.create_study(direction='maximize')
study.optimize(objective, n_trials=50)

# 결과 출력
print('Best trial:')
trial = study.best_trial
print(f'  Value: {trial.value:.4f}')
print('  Params:')
for key, value in trial.params.items():
    print(f'    {key}: {value}')

# 시각화
plot_optimization_history(study).show()
plot_param_importances(study).show()
```

### 시나리오 3: 모델 앙상블

```python
import torch
import torch.nn as nn

class EnsembleModel:
    """여러 모델의 앙상블"""
    
    def __init__(self, models):
        self.models = models
        for model in self.models:
            model.eval()
    
    def predict_average(self, x):
        """평균 앙상블"""
        predictions = []
        with torch.no_grad():
            for model in self.models:
                pred = model(x)
                predictions.append(pred)
        
        return torch.stack(predictions).mean(dim=0)
    
    def predict_voting(self, x):
        """투표 앙상블 (분류)"""
        predictions = []
        with torch.no_grad():
            for model in self.models:
                pred = model(x)
                predictions.append(pred.argmax(dim=1))
        
        predictions = torch.stack(predictions)
        # 다수결 투표
        mode_result = torch.mode(predictions, dim=0)
        return mode_result.values
    
    def predict_weighted(self, x, weights):
        """가중 평균 앙상블"""
        predictions = []
        with torch.no_grad():
            for model, weight in zip(self.models, weights):
                pred = model(x)
                predictions.append(pred * weight)
        
        return torch.stack(predictions).sum(dim=0)

# 사용 예제
model1 = torch.load('model1.pth')
model2 = torch.load('model2.pth')
model3 = torch.load('model3.pth')

ensemble = EnsembleModel([model1, model2, model3])

# 예측
x = torch.randn(1, 3, 224, 224).to(device)
avg_pred = ensemble.predict_average(x)
voting_pred = ensemble.predict_voting(x)
weighted_pred = ensemble.predict_weighted(x, weights=[0.5, 0.3, 0.2])

print(f"Average prediction: {avg_pred.argmax()}")
print(f"Voting prediction: {voting_pred}")
print(f"Weighted prediction: {weighted_pred.argmax()}")
```

---

## 머신러닝 트러블슈팅 체크리스트 ✅

### 데이터 준비
- [ ] 결측치를 적절히 처리했는가?
- [ ] 데이터가 정규화되었는가?
- [ ] 클래스 불균형을 고려했는가?
- [ ] Train/Val/Test를 올바르게 분리했는가?
- [ ] 데이터 누수(leakage)가 없는가?

### 모델 학습
- [ ] 학습률이 적절한가?
- [ ] Loss가 감소하는가?
- [ ] Gradient vanishing/exploding이 없는가?
- [ ] 배치 크기가 적절한가?
- [ ] 충분한 epoch 동안 학습했는가?

### 성능 평가
- [ ] 과적합이 발생하는가?
- [ ] 검증 데이터로 성능을 확인했는가?
- [ ] 적절한 평가 지표를 사용하는가?
- [ ] Confusion Matrix를 확인했는가?

### GPU/메모리
- [ ] GPU가 실제로 사용되는가?
- [ ] 메모리 부족 문제가 있는가?
- [ ] Mixed precision을 사용하는가?
- [ ] 불필요한 메모리를 정리하는가?

---

## 추가 팁 💡

### 1. 재현 가능성 확보

```python
import torch
import numpy as np
import random

def set_seed(seed=42):
    """모든 랜덤 시드 고정"""
    random.seed(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)
    torch.cuda.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
    torch.backends.cudnn.deterministic = True
    torch.backends.cudnn.benchmark = False

set_seed(42)
```

### 2. 실험 추적 (Weights & Biases)

```python
import wandb

# 초기화
wandb.init(project="my-project", name="experiment-1")

# Config 저장
wandb.config.update({
    "learning_rate": 0.001,
    "epochs": 100,
    "batch_size": 32
})

# 학습 중 로깅
for epoch in range(num_epochs):
    train_loss, train_acc = train(...)
    val_loss, val_acc = validate(...)
    
    wandb.log({
        "train_loss": train_loss,
        "train_acc": train_acc,
        "val_loss": val_loss,
        "val_acc": val_acc,
        "epoch": epoch
    })

wandb.finish()
```

### 3. 디버깅 모드

```python
# 작은 데이터셋으로 빠른 테스트
debug_mode = True

if debug_mode:
    # 데이터 일부만 사용
    train_dataset = torch.utils.data.Subset(train_dataset, range(100))
    num_epochs = 2
    print("⚠️  Debug mode: Using small dataset")
```

---

## 결론

머신러닝 트러블슈팅의 핵심:

1. **데이터 우선**: 좋은 데이터가 좋은 모델을 만듦
2. **단계별 검증**: 각 단계를 검증하며 진행
3. **실험 기록**: 모든 실험을 추적하고 기록
4. **재현 가능성**: 시드 고정과 환경 명시
5. **지속적 모니터링**: 학습 중 메트릭과 리소스 모니터링

Happy Machine Learning! 🤖
