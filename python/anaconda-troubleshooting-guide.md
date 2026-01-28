# Anaconda 트러블슈팅 가이드 🐍

## 목차
1. [Anaconda 설치 및 초기 설정](#1-anaconda-설치-및-초기-설정)
2. [가상환경 (Conda Environment) 문제](#2-가상환경-conda-environment-문제)
3. [패키지 관리 문제](#3-패키지-관리-문제)
4. [Jupyter 통합 문제](#4-jupyter-통합-문제)
5. [Conda vs Pip 충돌](#5-conda-vs-pip-충돌)
6. [성능 및 디스크 공간 문제](#6-성능-및-디스크-공간-문제)
7. [채널 (Channel) 관리](#7-채널-channel-관리)
8. [실전 예제 시나리오](#8-실전-예제-시나리오)

---

## 1. Anaconda 설치 및 초기 설정

### 1.1 설치 문제

#### 문제: Anaconda 설치 후 conda 명령어를 찾을 수 없음

```bash
# 에러:
conda --version
# bash: conda: command not found
```

**해결 방법:**

```bash
# 1. PATH 확인
echo $PATH

# 2. Anaconda 경로를 PATH에 추가
# ~/.bashrc 또는 ~/.zshrc에 추가
export PATH="/home/username/anaconda3/bin:$PATH"

# 또는 conda init 실행
/home/username/anaconda3/bin/conda init bash
# 터미널 재시작

# 3. 설정 즉시 적용
source ~/.bashrc

# 4. 확인
conda --version
```

**Windows 환경:**

```powershell
# Anaconda Prompt 사용 또는
# 시스템 환경 변수에 추가
# C:\Users\username\anaconda3
# C:\Users\username\anaconda3\Scripts
# C:\Users\username\anaconda3\Library\bin
```

### 1.2 Base 환경 자동 활성화 문제

#### 문제: 터미널 시작 시 (base) 환경이 자동으로 활성화됨

```bash
# 터미널 시작 시
(base) user@machine:~$
```

**해결 방법:**

```bash
# 자동 활성화 비활성화
conda config --set auto_activate_base false

# 다시 활성화하려면
conda config --set auto_activate_base true

# 또는 매번 수동으로 활성화
conda activate base

# 비활성화
conda deactivate
```

### 1.3 초기 설정 최적화

```bash
# conda 설정 확인
conda config --show

# 자주 사용하는 설정
conda config --set channel_priority strict  # 채널 우선순위 엄격하게
conda config --set always_yes true          # 자동으로 yes 응답
conda config --add channels conda-forge     # conda-forge 채널 추가

# 설정 파일 위치
# ~/.condarc (Linux/macOS)
# C:\Users\username\.condarc (Windows)

# .condarc 예제
cat > ~/.condarc <<EOF
channels:
  - conda-forge
  - defaults
channel_priority: strict
auto_activate_base: false
show_channel_urls: true
EOF
```

---

## 2. 가상환경 (Conda Environment) 문제

### 2.1 환경 생성 및 관리

#### 기본 환경 관리

```bash
# 환경 생성
conda create -n myenv python=3.11

# Python 버전 지정하여 생성
conda create -n py39 python=3.9
conda create -n py311 python=3.11

# 특정 패키지와 함께 생성
conda create -n ml-env python=3.10 numpy pandas scikit-learn

# 환경 목록 확인
conda env list
# 또는
conda info --envs

# 환경 활성화
conda activate myenv

# 환경 비활성화
conda deactivate

# 환경 삭제
conda remove -n myenv --all

# 환경 복제
conda create -n myenv-copy --clone myenv
```

### 2.2 환경 활성화 실패

#### 문제: conda activate가 작동하지 않음

```bash
# 에러:
conda activate myenv
# CommandNotFoundError: Your shell has not been properly configured to use 'conda activate'
```

**해결 방법:**

```bash
# 1. conda init 실행
conda init bash  # 또는 zsh, fish, powershell 등

# 2. 터미널 재시작 또는
source ~/.bashrc

# 3. 그래도 안 되면 수동으로 활성화
source /path/to/anaconda3/bin/activate myenv

# 4. Windows에서
# Anaconda Prompt 사용 또는
conda init powershell
# PowerShell 재시작
```

### 2.3 환경 Export 및 Import

#### 환경 공유하기

```bash
# 1. environment.yml로 export (권장)
conda env export > environment.yml

# OS 독립적인 버전 (플랫폼 간 공유)
conda env export --no-builds > environment.yml

# 더 간단한 버전 (pip 패키지 제외)
conda env export --from-history > environment.yml

# 2. environment.yml에서 환경 생성
conda env create -f environment.yml

# 3. 기존 환경 업데이트
conda env update -f environment.yml --prune
```

**environment.yml 예제:**

```yaml
name: ml-project
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.10
  - numpy=1.24
  - pandas=2.0
  - scikit-learn=1.3
  - matplotlib=3.7
  - jupyter
  - pip
  - pip:
    - torch==2.0.1
    - transformers==4.30.0
```

#### 문제: environment.yml에서 환경 생성 실패

```bash
# 에러:
# ResolvePackageNotFound: 
#   - torch==2.0.1+cu118
```

**해결 방법:**

```yaml
# 1. 플랫폼 특정 빌드 제거
# environment.yml 수정:
dependencies:
  - torch=2.0.1  # +cu118 제거

# 2. 또는 핵심 패키지만 포함
name: ml-project
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.10
  - numpy
  - pandas
  - scikit-learn
  - pip:
    - torch
    - transformers
```

### 2.4 환경 변수 관리

```bash
# 환경별 환경 변수 설정
# 1. 환경 디렉토리 확인
conda info --envs

# 2. activate.d 디렉토리 생성
mkdir -p $CONDA_PREFIX/etc/conda/activate.d
mkdir -p $CONDA_PREFIX/etc/conda/deactivate.d

# 3. 환경 변수 스크립트 생성
cat > $CONDA_PREFIX/etc/conda/activate.d/env_vars.sh <<'EOF'
#!/bin/sh
export MY_KEY='my_value'
export DATA_PATH='/path/to/data'
EOF

cat > $CONDA_PREFIX/etc/conda/deactivate.d/env_vars.sh <<'EOF'
#!/bin/sh
unset MY_KEY
unset DATA_PATH
EOF

# 4. 실행 권한 부여
chmod +x $CONDA_PREFIX/etc/conda/activate.d/env_vars.sh
chmod +x $CONDA_PREFIX/etc/conda/deactivate.d/env_vars.sh

# 5. 테스트
conda activate myenv
echo $MY_KEY  # 출력: my_value
conda deactivate
echo $MY_KEY  # 출력: (빈 값)
```

---

## 3. 패키지 관리 문제

### 3.1 패키지 설치 문제

#### 기본 패키지 관리

```bash
# 패키지 설치
conda install numpy

# 특정 버전 설치
conda install numpy=1.24.3

# 여러 패키지 동시 설치
conda install numpy pandas matplotlib

# 특정 채널에서 설치
conda install -c conda-forge package-name

# 패키지 업데이트
conda update numpy

# 모든 패키지 업데이트
conda update --all

# 패키지 제거
conda remove numpy

# 설치된 패키지 목록
conda list

# 특정 패키지 검색
conda search numpy
```

#### 문제: 패키지 설치 시 의존성 충돌

```bash
# 에러:
conda install tensorflow
# Collecting package metadata: done
# Solving environment: failed
# 
# UnsatisfiableError: The following specifications were found to be incompatible:
#   - numpy==1.24
#   - tensorflow -> numpy[version='>=1.16,<1.19']
```

**해결 방법:**

```bash
# 1. 새로운 환경에서 설치 (권장)
conda create -n tf-env python=3.9
conda activate tf-env
conda install tensorflow

# 2. 버전 제약 완화
conda install tensorflow numpy

# 3. conda-forge 채널 사용
conda install -c conda-forge tensorflow

# 4. mamba 사용 (더 빠른 solver)
conda install -c conda-forge mamba
mamba install tensorflow

# 5. pip로 설치 (최후의 수단)
pip install tensorflow
```

### 3.2 패키지 버전 고정

```bash
# 특정 버전으로 고정
conda install numpy=1.24.3 --freeze-installed

# 또는 environment.yml에 명시
# environment.yml
dependencies:
  - numpy=1.24.3
  - pandas=2.0.1
  - scikit-learn>=1.3,<1.4

# 설치된 패키지 버전 확인
conda list | grep numpy

# 패키지 정보 상세 확인
conda info numpy
```

### 3.3 캐시 및 인덱스 문제

#### 문제: 패키지 다운로드 실패 또는 느림

```bash
# 캐시 위치 확인
conda info

# 캐시 정리
conda clean --all

# 또는 단계별로
conda clean --packages      # 사용하지 않는 패키지 제거
conda clean --tarballs      # 압축 파일 제거
conda clean --index-cache   # 인덱스 캐시 제거

# 인덱스 업데이트
conda update conda

# 채널 캐시 강제 새로고침
conda search --override-channels -c conda-forge numpy
```

---

## 4. Jupyter 통합 문제

### 4.1 Jupyter에서 Conda 환경 사용

#### 문제: Jupyter에서 생성한 환경이 보이지 않음

```bash
# Jupyter Notebook을 실행했지만 커널 목록에 내 환경이 없음
```

**해결 방법:**

```bash
# 1. 환경에 ipykernel 설치
conda activate myenv
conda install ipykernel

# 2. Jupyter에 커널 등록
python -m ipykernel install --user --name=myenv --display-name "Python (myenv)"

# 3. 확인
jupyter kernelspec list

# 4. Jupyter 재시작
jupyter notebook

# 커널 제거
jupyter kernelspec uninstall myenv
```

#### 여러 환경을 Jupyter에 등록

```bash
# ML 환경
conda create -n ml-env python=3.10 numpy pandas scikit-learn
conda activate ml-env
conda install ipykernel
python -m ipykernel install --user --name=ml-env --display-name "Python (ML)"

# Deep Learning 환경
conda create -n dl-env python=3.9 numpy pandas
conda activate dl-env
conda install ipykernel
pip install torch torchvision
python -m ipykernel install --user --name=dl-env --display-name "Python (DL)"

# NLP 환경
conda create -n nlp-env python=3.10 numpy pandas
conda activate nlp-env
conda install ipykernel
pip install transformers datasets
python -m ipykernel install --user --name=nlp-env --display-name "Python (NLP)"

# 모든 커널 확인
jupyter kernelspec list
```

### 4.2 JupyterLab Extension 문제

```bash
# JupyterLab 설치
conda install -c conda-forge jupyterlab

# Extension 목록 확인
jupyter labextension list

# Extension 설치 예제
conda install -c conda-forge jupyterlab-git  # Git extension
conda install -c conda-forge jupyterlab_code_formatter  # Code formatter

# Node.js 관련 문제 해결
conda install -c conda-forge nodejs

# Extension 캐시 정리
jupyter lab clean

# JupyterLab 재빌드
jupyter lab build
```

---

## 5. Conda vs Pip 충돌

### 5.1 Conda와 Pip 혼용 시 문제

#### 문제: Conda와 Pip를 함께 사용하면 의존성이 꼬임

```bash
# 나쁜 예:
conda install numpy
pip install pandas  # ❌ numpy를 다시 설치할 수 있음
conda install scikit-learn  # ❌ 의존성 충돌 가능
```

**Best Practice:**

```bash
# 1. 가능하면 conda만 사용
conda install numpy pandas scikit-learn

# 2. conda에 없는 패키지만 pip 사용
conda install numpy pandas
pip install some-special-package

# 3. environment.yml에 명시
# environment.yml
name: myenv
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.10
  - numpy
  - pandas
  - pip
  - pip:
    - some-special-package
    - another-pip-only-package

# 4. pip 패키지는 마지막에 설치
conda install numpy pandas matplotlib
pip install -r requirements.txt
```

### 5.2 의존성 추적 문제

```bash
# Conda와 Pip 패키지 구분하기
conda list

# 출력 예시:
# numpy     1.24.3    py310_0    conda-forge  # conda로 설치
# pandas    2.0.1     pypi_0     pypi          # pip로 설치

# pip만 표시
conda list | grep pypi

# conda만 표시
conda list | grep -v pypi

# pip freeze (pip로 설치된 것만)
pip freeze
```

### 5.3 환경 재생성으로 정리

```bash
# 문제가 생긴 환경 export
conda env export > broken-env.yml

# 환경 삭제
conda deactivate
conda remove -n myenv --all

# environment.yml 정리 (conda 패키지만)
cat > clean-env.yml <<EOF
name: myenv
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.10
  - numpy
  - pandas
  - scikit-learn
  - pip
  - pip:
    - special-package
EOF

# 새 환경 생성
conda env create -f clean-env.yml
```

---

## 6. 성능 및 디스크 공간 문제

### 6.1 디스크 공간 부족

#### 문제: Anaconda가 너무 많은 공간을 차지함

```bash
# 디스크 사용량 확인
conda info

# envs 디렉토리 크기
du -sh ~/anaconda3/envs/*

# pkgs 디렉토리 크기
du -sh ~/anaconda3/pkgs

# 출력 예시:
# 5.2G    /home/user/anaconda3/envs/base
# 3.1G    /home/user/anaconda3/envs/ml-env
# 8.7G    /home/user/anaconda3/pkgs
```

**해결 방법:**

```bash
# 1. 사용하지 않는 패키지 정리
conda clean --all

# 2. 사용하지 않는 환경 삭제
conda env list
conda remove -n old-env --all

# 3. 캐시만 정리
conda clean --packages --tarballs

# 4. 특정 패키지의 오래된 버전 삭제
conda clean --packages

# 5. 디스크 사용량 확인
conda clean --all --dry-run  # 실제 삭제 전 확인

# 6. Miniconda 사용 고려
# Anaconda (3GB+) 대신 Miniconda (400MB)
# 필요한 패키지만 설치
```

### 6.2 느린 환경 해결 (Solving Environment)

#### 문제: conda install이 매우 느림

```bash
# 증상:
conda install numpy
# Solving environment: |  (수십 분 소요)
```

**해결 방법:**

**방법 1: Mamba 사용 (권장)**

```bash
# mamba 설치
conda install -c conda-forge mamba

# mamba로 패키지 설치 (훨씬 빠름)
mamba install numpy pandas scikit-learn

# mamba로 환경 생성
mamba create -n myenv python=3.10 numpy pandas

# 기존 conda 명령어를 mamba로 대체
alias conda=mamba  # ~/.bashrc에 추가
```

**방법 2: 채널 최적화**

```bash
# 채널 우선순위 설정
conda config --set channel_priority strict

# 불필요한 채널 제거
conda config --remove channels channel-name

# 권장 채널 설정
conda config --add channels conda-forge
conda config --remove channels defaults  # 필요시
```

**방법 3: Conda 버전 업데이트**

```bash
# conda 업데이트
conda update -n base conda

# libmamba solver 사용 (conda 4.11+)
conda install -n base conda-libmamba-solver
conda config --set solver libmamba
```

---

## 7. 채널 (Channel) 관리

### 7.1 채널 이해 및 설정

```bash
# 현재 채널 확인
conda config --show channels

# 채널 추가
conda config --add channels conda-forge
conda config --add channels bioconda  # 생물정보학
conda config --add channels pytorch    # PyTorch

# 채널 제거
conda config --remove channels channel-name

# 채널 우선순위 확인
conda config --get channels

# 채널 우선순위 설정
conda config --set channel_priority strict  # 엄격 (권장)
conda config --set channel_priority flexible  # 유연
```

### 7.2 특정 채널에서만 설치

```bash
# conda-forge에서만 설치
conda install -c conda-forge numpy

# 여러 채널 지정
conda install -c conda-forge -c defaults numpy

# 특정 채널만 사용 (다른 채널 무시)
conda install --override-channels -c conda-forge numpy

# 채널별 패키지 검색
conda search -c conda-forge numpy
conda search -c defaults numpy
```

### 7.3 채널 충돌 문제

#### 문제: 여러 채널의 패키지가 충돌함

```bash
# 에러:
# PackagesNotFoundError: The following packages are not available from current channels
```

**해결 방법:**

```bash
# 1. 채널 우선순위를 strict로 설정
conda config --set channel_priority strict

# 2. .condarc에서 채널 순서 확인
cat ~/.condarc
# channels:
#   - conda-forge
#   - defaults

# 3. 특정 채널 명시
conda install -c conda-forge package-name

# 4. 환경별 채널 설정
# environment.yml
name: myenv
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.10
  - numpy
```

---

## 8. 실전 예제 시나리오

### 시나리오 1: 데이터 과학 프로젝트 환경 구축

**목표:** NumPy, Pandas, Scikit-learn, Jupyter를 포함한 완벽한 환경

```bash
# 1. 환경 생성
conda create -n ds-project python=3.10
conda activate ds-project

# 2. 기본 패키지 설치
conda install -c conda-forge numpy pandas matplotlib seaborn scikit-learn

# 3. Jupyter 설치 및 설정
conda install -c conda-forge jupyterlab ipykernel
python -m ipykernel install --user --name=ds-project --display-name "Python (DS Project)"

# 4. 추가 패키지
conda install -c conda-forge plotly statsmodels scipy

# 5. environment.yml 생성
conda env export --no-builds > environment.yml

# 6. Git 저장소에 추가
git add environment.yml
git commit -m "Add conda environment"
```

**environment.yml 예제:**

```yaml
name: ds-project
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.10
  - numpy=1.24
  - pandas=2.0
  - matplotlib=3.7
  - seaborn=0.12
  - scikit-learn=1.3
  - jupyterlab=4.0
  - ipykernel
  - plotly=5.14
  - statsmodels=0.14
  - scipy=1.11
```

### 시나리오 2: 딥러닝 환경 구축 (GPU 지원)

**CUDA 환경 설정:**

```bash
# 1. CUDA 버전 확인
nvidia-smi

# 출력 예시:
# CUDA Version: 11.8

# 2. PyTorch 환경 생성
conda create -n pytorch-gpu python=3.10
conda activate pytorch-gpu

# 3. PyTorch 설치 (CUDA 11.8)
# https://pytorch.org/get-started/locally/ 참조
conda install pytorch torchvision torchaudio pytorch-cuda=11.8 -c pytorch -c nvidia

# 4. 추가 라이브러리
conda install -c conda-forge numpy pandas matplotlib jupyterlab

# 5. 설치 확인
python -c "import torch; print(torch.cuda.is_available())"  # True여야 함
python -c "import torch; print(torch.cuda.device_count())"   # GPU 개수

# 6. environment.yml
cat > environment.yml <<EOF
name: pytorch-gpu
channels:
  - pytorch
  - nvidia
  - conda-forge
  - defaults
dependencies:
  - python=3.10
  - pytorch
  - torchvision
  - torchaudio
  - pytorch-cuda=11.8
  - numpy
  - pandas
  - matplotlib
  - jupyterlab
  - pip
  - pip:
    - transformers
    - datasets
EOF
```

**문제: CUDA 버전 불일치**

```bash
# 에러:
# RuntimeError: CUDA error: no kernel image is available for execution on the device

# 해결: 올바른 CUDA 버전 설치
# 1. CUDA 버전 확인
nvidia-smi

# 2. 호환되는 PyTorch 버전 설치
# CUDA 11.8
conda install pytorch==2.0.1 torchvision==0.15.2 pytorch-cuda=11.8 -c pytorch -c nvidia

# CUDA 12.1
conda install pytorch==2.0.1 torchvision==0.15.2 pytorch-cuda=12.1 -c pytorch -c nvidia

# CPU 버전 (GPU 없을 때)
conda install pytorch torchvision torchaudio cpuonly -c pytorch
```

### 시나리오 3: 여러 프로젝트 환경 관리

```bash
# 프로젝트별 환경 구조
# ~/projects/
#   ├── project-a/  (Python 3.9, TensorFlow)
#   ├── project-b/  (Python 3.10, PyTorch)
#   └── project-c/  (Python 3.11, Scikit-learn)

# Project A - TensorFlow
conda create -n project-a python=3.9
conda activate project-a
conda install tensorflow numpy pandas jupyterlab
python -m ipykernel install --user --name=project-a

# Project B - PyTorch
conda create -n project-b python=3.10
conda activate project-b
conda install pytorch torchvision -c pytorch
conda install numpy pandas jupyterlab
python -m ipykernel install --user --name=project-b

# Project C - Traditional ML
conda create -n project-c python=3.11
conda activate project-c
conda install scikit-learn xgboost lightgbm numpy pandas jupyterlab
python -m ipykernel install --user --name=project-c

# 환경 목록 확인
conda env list

# Jupyter에서 모든 커널 사용 가능
jupyter lab
```

### 시나리오 4: 환경 이식 및 공유

**개발 환경 → 프로덕션 환경**

```bash
# 개발 머신에서
conda activate myenv

# 1. 상세한 환경 export
conda env export > environment-dev.yml

# 2. 크로스 플랫폼 환경 (플랫폼 독립적)
conda env export --from-history > environment.yml

# 3. pip 패키지도 포함
conda env export > environment-full.yml

# Git에 커밋
git add environment.yml
git commit -m "Add conda environment specification"
git push
```

```bash
# 프로덕션 머신에서
git clone <repository>
cd <repository>

# 환경 생성
conda env create -f environment.yml

# 활성화 및 테스트
conda activate myenv
python -m pytest  # 테스트 실행
```

**Docker와 통합:**

```dockerfile
# Dockerfile
FROM continuumio/miniconda3

# 작업 디렉토리
WORKDIR /app

# environment.yml 복사
COPY environment.yml .

# 환경 생성
RUN conda env create -f environment.yml

# 환경 활성화를 위한 쉘 설정
SHELL ["conda", "run", "-n", "myenv", "/bin/bash", "-c"]

# 소스 코드 복사
COPY . .

# 애플리케이션 실행
CMD ["conda", "run", "-n", "myenv", "python", "app.py"]
```

### 시나리오 5: 대규모 팀 환경 표준화

**팀 표준 환경 관리:**

```bash
# 1. 기본 환경 템플릿 생성
# base-environment.yml
name: team-base
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.10
  - numpy
  - pandas
  - matplotlib
  - jupyterlab
  - pytest
  - black
  - flake8
  - mypy

# 2. 프로젝트별 추가 패키지
# project-ml.yml
name: team-ml
channels:
  - conda-forge
  - defaults
dependencies:
  - python=3.10
  - numpy
  - pandas
  - matplotlib
  - jupyterlab
  - scikit-learn
  - xgboost
  - optuna

# 3. 환경 생성 스크립트
cat > setup-env.sh <<'EOF'
#!/bin/bash
ENV_NAME=$1
ENV_FILE=$2

if [ -z "$ENV_NAME" ] || [ -z "$ENV_FILE" ]; then
    echo "Usage: ./setup-env.sh <env-name> <env-file>"
    exit 1
fi

# 환경 생성
conda env create -f "$ENV_FILE" -n "$ENV_NAME"

# Jupyter 커널 등록
conda activate "$ENV_NAME"
python -m ipykernel install --user --name="$ENV_NAME"

echo "Environment $ENV_NAME created successfully!"
EOF

chmod +x setup-env.sh

# 4. 사용
./setup-env.sh my-ml-project project-ml.yml
```

---

## Anaconda 명령어 치트시트

### 환경 관리
```bash
conda create -n myenv python=3.10        # 환경 생성
conda activate myenv                     # 환경 활성화
conda deactivate                         # 환경 비활성화
conda env list                          # 환경 목록
conda remove -n myenv --all             # 환경 삭제
conda env export > environment.yml      # 환경 export
conda env create -f environment.yml     # 환경 생성
```

### 패키지 관리
```bash
conda install numpy                     # 패키지 설치
conda install numpy=1.24.3              # 특정 버전 설치
conda install -c conda-forge numpy      # 특정 채널에서 설치
conda update numpy                      # 패키지 업데이트
conda remove numpy                      # 패키지 제거
conda list                             # 설치된 패키지 목록
conda search numpy                      # 패키지 검색
```

### 정보 확인
```bash
conda info                             # Conda 정보
conda info --envs                      # 환경 정보
conda list                             # 패키지 목록
conda config --show                    # 설정 확인
```

### 유지보수
```bash
conda clean --all                      # 모든 캐시 정리
conda update conda                     # Conda 업데이트
conda update --all                     # 모든 패키지 업데이트
```

---

## Anaconda 트러블슈팅 체크리스트 ✅

### 환경 문제
- [ ] conda 명령어가 작동하는가?
- [ ] 올바른 환경이 활성화되었는가?
- [ ] environment.yml이 올바른가?
- [ ] 채널 우선순위가 설정되었는가?

### 패키지 문제
- [ ] 패키지 버전이 호환되는가?
- [ ] Conda와 Pip를 적절히 사용했는가?
- [ ] 캐시를 정리했는가?
- [ ] Mamba를 사용해봤는가?

### Jupyter 문제
- [ ] ipykernel이 설치되었는가?
- [ ] Jupyter 커널이 등록되었는가?
- [ ] 올바른 커널을 선택했는가?

### 디스크 공간
- [ ] 불필요한 환경을 삭제했는가?
- [ ] 캐시를 정리했는가?
- [ ] Miniconda 사용을 고려했는가?

---

## 추가 팁 💡

### 1. Mamba 사용 (필수!)

```bash
# Mamba 설치
conda install -c conda-forge mamba

# 모든 conda 명령어를 mamba로 대체
# 10-100배 빠른 속도!
mamba install numpy pandas matplotlib
mamba create -n myenv python=3.10 numpy
```

### 2. .condarc 최적화

```yaml
# ~/.condarc
channels:
  - conda-forge
  - defaults
channel_priority: strict
auto_activate_base: false
show_channel_urls: true
always_yes: false
```

### 3. 환경 이름 규칙

```bash
# 프로젝트명-목적-버전
project1-dev-py310
project1-prod-py310
ml-research-v1
dl-gpu-pytorch2
```

### 4. 정기 유지보수

```bash
# 매월 실행
conda update conda
conda update --all
conda clean --all

# 사용하지 않는 환경 정리
conda env list
conda remove -n old-env --all
```

---

## 결론

Anaconda 트러블슈팅의 핵심:

1. **Mamba 사용**: 패키지 설치 속도 대폭 향상
2. **환경 분리**: 프로젝트별 독립적인 환경 유지
3. **Conda 우선**: Conda로 설치 가능하면 Pip 사용 자제
4. **정기 정리**: 디스크 공간 관리를 위한 캐시 정리
5. **환경 공유**: environment.yml로 재현 가능한 환경

Happy Conda! 🐍
