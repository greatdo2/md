# Python 트러블슈팅 가이드 🐍

## 목차
1. [Python 환경 문제](#1-python-환경-문제)
2. [패키지 설치 및 의존성 문제](#2-패키지-설치-및-의존성-문제)
3. [Import 오류](#3-import-오류)
4. [인코딩 문제](#4-인코딩-문제)
5. [메모리 및 성능 문제](#5-메모리-및-성능-문제)
6. [멀티프로세싱 및 멀티쓰레딩](#6-멀티프로세싱-및-멀티쓰레딩)
7. [디버깅 기법](#7-디버깅-기법)
8. [실전 예제 시나리오](#8-실전-예제-시나리오)

---

## 1. Python 환경 문제

### 1.1 여러 Python 버전 충돌

#### 문제: 시스템에 여러 Python 버전이 설치되어 혼란

```bash
# 현재 사용 중인 Python 확인
which python
which python3

# Python 버전 확인
python --version
python3 --version

# 설치된 모든 Python 찾기 (macOS/Linux)
ls -la /usr/bin/python*
ls -la /usr/local/bin/python*

# Windows
where python
```

**출력 예시:**
```
/usr/bin/python -> python2.7
/usr/bin/python3 -> python3.8
/usr/local/bin/python3 -> python3.11
```

**해결 방법:**

```bash
# 1. alias 설정으로 명확하게 지정
# ~/.bashrc 또는 ~/.zshrc
alias python=python3.11
alias pip=pip3

# 2. pyenv로 버전 관리 (권장)
# pyenv 설치
curl https://pyenv.run | bash

# 특정 버전 설치
pyenv install 3.11.5

# 전역 Python 버전 설정
pyenv global 3.11.5

# 프로젝트별 버전 설정
cd my-project/
pyenv local 3.11.5

# 확인
python --version
```

### 1.2 pip vs pip3 문제

#### 문제: pip와 pip3가 다른 Python 버전을 가리킴

```bash
# pip가 어떤 Python을 사용하는지 확인
pip --version
# pip 20.0.2 from /usr/lib/python2.7/dist-packages/pip (python 2.7)

pip3 --version
# pip 23.2.1 from /usr/lib/python3.11/site-packages/pip (python 3.11)
```

**해결 방법:**

```bash
# Python 모듈로 pip 실행 (권장)
python3.11 -m pip install package-name

# 또는 pip를 업그레이드
python3 -m pip install --upgrade pip

# 특정 버전의 pip 사용
python3.11 -m pip list
```

### 1.3 PATH 환경 변수 문제

#### 문제: 설치한 패키지의 실행 파일을 찾을 수 없음

```bash
# 에러 예시:
# command not found: jupyter
# command not found: black

# PATH 확인
echo $PATH

# Python 패키지 설치 경로 확인
python3 -m site --user-site
```

**해결 방법:**

```bash
# 1. User bin 디렉토리를 PATH에 추가
# ~/.bashrc 또는 ~/.zshrc
export PATH="$HOME/.local/bin:$PATH"

# macOS의 경우
export PATH="$HOME/Library/Python/3.11/bin:$PATH"

# 2. 가상환경 사용 (권장)
python3 -m venv myenv
source myenv/bin/activate  # Windows: myenv\Scripts\activate

# 3. 패키지 재설치
pip install --user jupyter
# 또는 가상환경 내에서
pip install jupyter
```

---

## 2. 패키지 설치 및 의존성 문제

### 2.1 패키지 설치 실패

#### 문제: pip install 실패

```bash
# 에러 예시:
pip install numpy

# ERROR: Could not build wheels for numpy, which is required to install 
# pyproject.toml-based projects
```

**원인 및 해결:**

**1. 권한 문제**

```bash
# 에러:
# ERROR: Could not install packages due to an OSError: 
# [Errno 13] Permission denied

# 해결 방법 1: --user 플래그 사용
pip install --user numpy

# 해결 방법 2: 가상환경 사용 (권장)
python3 -m venv myenv
source myenv/bin/activate
pip install numpy

# 해결 방법 3: sudo 사용 (비권장 - 시스템 패키지 오염 가능)
sudo pip install numpy  # 권장하지 않음!
```

**2. 컴파일러 누락**

```bash
# macOS
xcode-select --install

# Ubuntu/Debian
sudo apt-get update
sudo apt-get install build-essential python3-dev

# CentOS/RHEL
sudo yum groupinstall "Development Tools"
sudo yum install python3-devel
```

**3. 특정 패키지 버전 충돌**

```bash
# 에러:
# ERROR: pip's dependency resolver does not currently take into account 
# all the packages that are installed

# 진단
pip check

# 해결: requirements.txt에 버전 명시
cat > requirements.txt <<EOF
numpy==1.24.3
pandas==2.0.3
scipy==1.11.1
EOF

pip install -r requirements.txt

# 또는 호환 가능한 버전 찾기
pip install numpy pandas scipy --upgrade
```

### 2.2 의존성 지옥 (Dependency Hell)

#### 예제: 패키지 A는 numpy>=1.20을 요구, 패키지 B는 numpy<1.20을 요구

```bash
# 문제 상황
pip install package-a  # requires numpy>=1.20
pip install package-b  # requires numpy<1.20

# 에러:
# ERROR: Cannot install package-a and package-b because these package 
# versions have conflicting dependencies.
```

**해결 방법:**

```bash
# 1. 의존성 트리 확인
pip install pipdeptree
pipdeptree

# 출력 예시:
# package-a==1.0.0
#   - numpy [required: >=1.20, installed: 1.24.3]
# package-b==2.0.0
#   - numpy [required: <1.20, installed: 1.24.3]

# 2. 가상환경으로 분리
python3 -m venv env_a
source env_a/bin/activate
pip install package-a

deactivate

python3 -m venv env_b
source env_b/bin/activate
pip install package-b

# 3. Docker 사용 (완전한 격리)
# Dockerfile
FROM python:3.11-slim
RUN pip install package-a numpy==1.24.3
```

### 2.3 캐시 문제

#### 문제: 이전 설치 실패로 인한 캐시 손상

```bash
# 증상: 패키지 재설치 시 계속 실패

# 해결: pip 캐시 삭제
pip cache purge

# 또는 캐시 없이 설치
pip install --no-cache-dir numpy

# 캐시 디렉토리 확인
pip cache dir
```

---

## 3. Import 오류

### 3.1 ModuleNotFoundError

#### 문제: 설치한 패키지를 import할 수 없음

```python
# 에러 예시
import numpy
# ModuleNotFoundError: No module named 'numpy'
```

**진단 및 해결:**

```python
# 1. Python이 패키지를 찾는 경로 확인
import sys
print(sys.path)

# 2. 패키지가 실제로 설치되었는지 확인
import subprocess
result = subprocess.run(['pip', 'list'], capture_output=True, text=True)
print(result.stdout)

# 또는 커맨드라인에서
pip list | grep numpy
```

**일반적인 원인:**

**원인 1: 다른 Python 환경에 설치**

```bash
# 해결: 현재 Python에 직접 설치
python -m pip install numpy

# Jupyter에서 사용하는 경우
python -m pip install ipykernel
python -m ipykernel install --user --name=myenv
```

**원인 2: 잘못된 가상환경**

```bash
# 현재 가상환경 확인
which python
echo $VIRTUAL_ENV

# 올바른 가상환경 활성화
source /path/to/correct/venv/bin/activate
```

**원인 3: PYTHONPATH 문제**

```python
# 임시 해결 (권장하지 않음)
import sys
sys.path.append('/path/to/package')

# 영구 해결: 패키지를 editable 모드로 설치
pip install -e /path/to/package
```

### 3.2 순환 Import (Circular Import)

#### 문제: 모듈 간 순환 참조

```python
# module_a.py
from module_b import function_b

def function_a():
    return function_b()

# module_b.py
from module_a import function_a  # ❌ 순환 import!

def function_b():
    return function_a()

# main.py
import module_a  # ImportError: cannot import name 'function_a' from partially initialized module
```

**해결 방법:**

**방법 1: Import 위치 변경**

```python
# module_b.py
def function_b():
    from module_a import function_a  # ✅ 함수 내부에서 import
    return function_a()
```

**방법 2: 구조 재설계**

```python
# common.py - 공통 기능 분리
def shared_function():
    pass

# module_a.py
from common import shared_function

def function_a():
    return shared_function()

# module_b.py
from common import shared_function

def function_b():
    return shared_function()
```

**방법 3: Import 타입 힌팅만 사용**

```python
# module_a.py
from typing import TYPE_CHECKING

if TYPE_CHECKING:
    from module_b import ClassB  # 런타임에는 실행되지 않음

def function_a(obj: 'ClassB'):  # 문자열로 타입 힌팅
    pass
```

### 3.3 상대 Import 문제

#### 문제: 상대 import가 작동하지 않음

```python
# 프로젝트 구조:
# myproject/
#   __init__.py
#   module_a.py
#   subpackage/
#     __init__.py
#     module_b.py

# subpackage/module_b.py
from ..module_a import function  # ImportError: attempted relative import with no known parent package
```

**해결 방법:**

```bash
# 방법 1: 패키지로 실행
python -m myproject.subpackage.module_b

# 방법 2: PYTHONPATH 설정
export PYTHONPATH="${PYTHONPATH}:/path/to/myproject"
python subpackage/module_b.py

# 방법 3: 절대 import 사용 (권장)
# subpackage/module_b.py
from myproject.module_a import function
```

---

## 4. 인코딩 문제

### 4.1 파일 읽기/쓰기 인코딩 오류

#### 문제: 한글 파일 처리 시 오류

```python
# 에러 예시
with open('data.txt', 'r') as f:
    content = f.read()
# UnicodeDecodeError: 'utf-8' codec can't decode byte 0xc7 in position 0
```

**해결 방법:**

```python
# 1. 인코딩 명시
with open('data.txt', 'r', encoding='utf-8') as f:
    content = f.read()

# 2. 인코딩 자동 감지
import chardet

with open('data.txt', 'rb') as f:
    raw_data = f.read()
    result = chardet.detect(raw_data)
    encoding = result['encoding']
    print(f"Detected encoding: {encoding}")

with open('data.txt', 'r', encoding=encoding) as f:
    content = f.read()

# 3. 에러 무시 (데이터 손실 가능)
with open('data.txt', 'r', encoding='utf-8', errors='ignore') as f:
    content = f.read()

# 4. 에러를 다른 문자로 대체
with open('data.txt', 'r', encoding='utf-8', errors='replace') as f:
    content = f.read()
```

**일반적인 인코딩 종류:**

```python
# 인코딩 감지 및 변환 함수
def read_file_with_encoding(filename):
    """다양한 인코딩으로 파일 읽기 시도"""
    encodings = ['utf-8', 'cp949', 'euc-kr', 'latin-1', 'ascii']
    
    for encoding in encodings:
        try:
            with open(filename, 'r', encoding=encoding) as f:
                content = f.read()
            print(f"Successfully read with {encoding}")
            return content
        except UnicodeDecodeError:
            continue
    
    raise ValueError(f"Could not decode {filename} with any encoding")

# 사용
content = read_file_with_encoding('data.txt')
```

### 4.2 CSV 파일 인코딩 문제

```python
import pandas as pd

# 문제: Excel에서 저장한 한글 CSV
try:
    df = pd.read_csv('data.csv')
except UnicodeDecodeError as e:
    print(f"Error: {e}")

# 해결 방법 1: 인코딩 지정
df = pd.read_csv('data.csv', encoding='cp949')  # Windows 한글
df = pd.read_csv('data.csv', encoding='euc-kr')  # 리눅스 한글

# 해결 방법 2: 자동 감지
import chardet

with open('data.csv', 'rb') as f:
    result = chardet.detect(f.read(10000))  # 처음 10KB만 확인
    encoding = result['encoding']

df = pd.read_csv('data.csv', encoding=encoding)

# 해결 방법 3: 에러 핸들링
df = pd.read_csv('data.csv', encoding='utf-8', encoding_errors='ignore')
```

---

## 5. 메모리 및 성능 문제

### 5.1 메모리 부족 (MemoryError)

#### 문제: 대용량 데이터 처리 시 메모리 부족

```python
# 문제 상황
import pandas as pd

# 10GB CSV 파일 읽기
df = pd.read_csv('huge_file.csv')  # MemoryError!
```

**해결 방법:**

**방법 1: 청크로 읽기**

```python
import pandas as pd

# 청크로 나눠서 읽기
chunk_size = 10000
chunks = []

for chunk in pd.read_csv('huge_file.csv', chunksize=chunk_size):
    # 각 청크 처리
    processed = chunk[chunk['value'] > 0]  # 필터링 예시
    chunks.append(processed)

# 필요시 결합
df = pd.concat(chunks, ignore_index=True)

# 또는 청크별로 처리 후 저장
for i, chunk in enumerate(pd.read_csv('huge_file.csv', chunksize=chunk_size)):
    processed = process_chunk(chunk)
    processed.to_csv(f'output_{i}.csv', index=False)
```

**방법 2: 데이터 타입 최적화**

```python
import pandas as pd
import numpy as np

# 메모리 사용량 확인
df = pd.read_csv('data.csv')
print(df.memory_usage(deep=True))
print(f"Total memory: {df.memory_usage(deep=True).sum() / 1024**2:.2f} MB")

# 데이터 타입 최적화
def optimize_dtypes(df):
    """데이터프레임의 메모리 사용량 최적화"""
    
    # 정수형 최적화
    for col in df.select_dtypes(include=['int']).columns:
        df[col] = pd.to_numeric(df[col], downcast='integer')
    
    # 실수형 최적화
    for col in df.select_dtypes(include=['float']).columns:
        df[col] = pd.to_numeric(df[col], downcast='float')
    
    # 객체형을 카테고리로 변환 (반복되는 값이 많은 경우)
    for col in df.select_dtypes(include=['object']).columns:
        num_unique = df[col].nunique()
        num_total = len(df[col])
        if num_unique / num_total < 0.5:  # 50% 이하면 카테고리로
            df[col] = df[col].astype('category')
    
    return df

df_optimized = optimize_dtypes(df)
print(f"Optimized memory: {df_optimized.memory_usage(deep=True).sum() / 1024**2:.2f} MB")
```

**방법 3: 필요한 컬럼만 읽기**

```python
# 특정 컬럼만 선택
df = pd.read_csv('huge_file.csv', usecols=['col1', 'col2', 'col3'])

# 특정 행만 읽기
df = pd.read_csv('huge_file.csv', nrows=100000)

# 조건에 맞는 행만 읽기 (불가능 - 대신 청크 사용)
chunks = []
for chunk in pd.read_csv('huge_file.csv', chunksize=10000):
    filtered = chunk[chunk['date'] > '2023-01-01']
    chunks.append(filtered)
df = pd.concat(chunks)
```

**방법 4: Dask 사용 (병렬 처리)**

```python
import dask.dataframe as dd

# Dask로 대용량 파일 처리
df = dd.read_csv('huge_file.csv')

# Pandas와 유사한 API
result = df[df['value'] > 0].groupby('category').mean()

# 계산 실행
result = result.compute()  # 이 시점에 실제 계산 수행
```

### 5.2 느린 코드 최적화

#### 문제: 반복문이 너무 느림

```python
import time

# 느린 코드
start = time.time()
result = []
for i in range(1000000):
    result.append(i ** 2)
print(f"Time: {time.time() - start:.2f}s")  # ~0.15s
```

**최적화 방법:**

**방법 1: List Comprehension**

```python
# 더 빠른 코드
start = time.time()
result = [i ** 2 for i in range(1000000)]
print(f"Time: {time.time() - start:.2f}s")  # ~0.09s
```

**방법 2: NumPy 벡터화**

```python
import numpy as np

# 가장 빠른 코드
start = time.time()
result = np.arange(1000000) ** 2
print(f"Time: {time.time() - start:.2f}s")  # ~0.003s
```

**방법 3: Numba JIT 컴파일**

```python
from numba import jit
import numpy as np

@jit(nopython=True)
def compute_squares(n):
    result = np.empty(n, dtype=np.int64)
    for i in range(n):
        result[i] = i ** 2
    return result

start = time.time()
result = compute_squares(1000000)
print(f"Time: {time.time() - start:.2f}s")  # ~0.002s (첫 실행 후)
```

**성능 프로파일링:**

```python
# cProfile로 병목 지점 찾기
import cProfile
import pstats

def slow_function():
    total = 0
    for i in range(1000000):
        total += i ** 2
    return total

# 프로파일링 실행
profiler = cProfile.Profile()
profiler.enable()
result = slow_function()
profiler.disable()

# 결과 출력
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(10)  # 상위 10개 함수

# 또는 간단하게
cProfile.run('slow_function()', sort='cumulative')
```

**line_profiler로 라인별 분석:**

```bash
# 설치
pip install line_profiler

# 코드에 @profile 데코레이터 추가
# script.py
@profile
def slow_function():
    result = []
    for i in range(100000):
        result.append(i ** 2)
    return result

# 실행
kernprof -l -v script.py
```

---

## 6. 멀티프로세싱 및 멀티쓰레딩

### 6.1 GIL (Global Interpreter Lock) 문제

#### 문제: 멀티쓰레딩이 예상보다 느림

```python
import threading
import time

def cpu_bound_task(n):
    """CPU 집약적 작업"""
    total = 0
    for i in range(n):
        total += i ** 2
    return total

# 단일 쓰레드
start = time.time()
result1 = cpu_bound_task(10000000)
result2 = cpu_bound_task(10000000)
print(f"Sequential: {time.time() - start:.2f}s")  # ~2.0s

# 멀티쓰레딩 (GIL 때문에 느림!)
start = time.time()
thread1 = threading.Thread(target=cpu_bound_task, args=(10000000,))
thread2 = threading.Thread(target=cpu_bound_task, args=(10000000,))
thread1.start()
thread2.start()
thread1.join()
thread2.join()
print(f"Threading: {time.time() - start:.2f}s")  # ~2.1s (더 느릴 수도!)
```

**해결 방법: 멀티프로세싱 사용**

```python
from multiprocessing import Pool
import time

def cpu_bound_task(n):
    total = 0
    for i in range(n):
        total += i ** 2
    return total

if __name__ == '__main__':
    # 멀티프로세싱
    start = time.time()
    with Pool(processes=2) as pool:
        results = pool.map(cpu_bound_task, [10000000, 10000000])
    print(f"Multiprocessing: {time.time() - start:.2f}s")  # ~1.1s (실제로 빠름!)
```

### 6.2 멀티프로세싱 오류

#### 문제: pickle 오류

```python
from multiprocessing import Pool

class MyClass:
    def __init__(self, value):
        self.value = value
    
    def process(self, x):
        return x * self.value

# 에러 발생!
obj = MyClass(10)
with Pool() as pool:
    result = pool.map(obj.process, range(10))
# AttributeError: Can't pickle local object
```

**해결 방법:**

```python
# 방법 1: 전역 함수 사용
def process_global(args):
    x, value = args
    return x * value

with Pool() as pool:
    result = pool.starmap(process_global, [(i, 10) for i in range(10)])

# 방법 2: __call__ 메서드 사용
class MyClass:
    def __init__(self, value):
        self.value = value
    
    def __call__(self, x):
        return x * self.value

# 방법 3: pathos 라이브러리 사용
from pathos.multiprocessing import ProcessPool

class MyClass:
    def __init__(self, value):
        self.value = value
    
    def process(self, x):
        return x * self.value

obj = MyClass(10)
with ProcessPool() as pool:
    result = pool.map(obj.process, range(10))
```

### 6.3 공유 메모리 문제

```python
from multiprocessing import Pool, Manager

# 문제: 각 프로세스가 독립적인 메모리 사용
counter = 0

def increment(x):
    global counter
    counter += 1  # 이 변경사항은 메인 프로세스에 반영되지 않음!
    return x ** 2

with Pool(4) as pool:
    results = pool.map(increment, range(100))

print(f"Counter: {counter}")  # 0 (변경되지 않음!)
```

**해결 방법:**

```python
from multiprocessing import Pool, Manager, Value, Lock

# 방법 1: Manager 사용
def increment_with_manager(args):
    x, shared_dict = args
    shared_dict['counter'] += 1
    return x ** 2

if __name__ == '__main__':
    with Manager() as manager:
        shared_dict = manager.dict({'counter': 0})
        args = [(i, shared_dict) for i in range(100)]
        
        with Pool(4) as pool:
            results = pool.map(increment_with_manager, args)
        
        print(f"Counter: {shared_dict['counter']}")  # 100

# 방법 2: Value와 Lock 사용 (더 빠름)
def increment_with_value(args):
    x, counter, lock = args
    with lock:
        counter.value += 1
    return x ** 2

if __name__ == '__main__':
    counter = Value('i', 0)  # 정수 타입 공유 변수
    lock = Lock()
    
    args = [(i, counter, lock) for i in range(100)]
    
    with Pool(4) as pool:
        results = pool.map(increment_with_value, args)
    
    print(f"Counter: {counter.value}")  # 100
```

---

## 7. 디버깅 기법

### 7.1 기본 디버깅

```python
# 1. print() 디버깅
def complex_function(data):
    print(f"[DEBUG] Input: {data}")
    result = process_data(data)
    print(f"[DEBUG] Result: {result}")
    return result

# 2. logging 사용 (권장)
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

logger = logging.getLogger(__name__)

def complex_function(data):
    logger.debug(f"Input: {data}")
    result = process_data(data)
    logger.info(f"Result: {result}")
    return result

# 3. assert 사용
def divide(a, b):
    assert b != 0, "Divisor cannot be zero"
    assert isinstance(a, (int, float)), "a must be numeric"
    assert isinstance(b, (int, float)), "b must be numeric"
    return a / b
```

### 7.2 pdb 디버거 사용

```python
import pdb

def buggy_function(data):
    result = []
    for item in data:
        # 디버거 시작점
        pdb.set_trace()  # 또는 breakpoint() (Python 3.7+)
        
        processed = item * 2
        result.append(processed)
    return result

# 실행 시 대화형 디버거 실행
# 주요 명령어:
# n (next): 다음 줄
# s (step): 함수 내부로 들어가기
# c (continue): 다음 breakpoint까지 계속
# l (list): 현재 코드 보기
# p variable: 변수 출력
# q (quit): 종료
```

### 7.3 예외 처리 및 트레이스백

```python
import traceback
import sys

def risky_function():
    try:
        result = 1 / 0
    except ZeroDivisionError as e:
        # 방법 1: 기본 에러 메시지
        print(f"Error: {e}")
        
        # 방법 2: 전체 트레이스백 출력
        traceback.print_exc()
        
        # 방법 3: 트레이스백을 문자열로 저장
        tb_str = traceback.format_exc()
        print(tb_str)
        
        # 방법 4: 상세 정보 추출
        exc_type, exc_value, exc_traceback = sys.exc_info()
        print(f"Type: {exc_type}")
        print(f"Value: {exc_value}")
        
        # 방법 5: 로그에 기록
        import logging
        logging.exception("Exception occurred")

# 커스텀 예외
class DataValidationError(Exception):
    """데이터 검증 실패 시 발생하는 예외"""
    def __init__(self, field, value, message="Invalid data"):
        self.field = field
        self.value = value
        self.message = f"{message}: {field}={value}"
        super().__init__(self.message)

def validate_data(data):
    if 'age' not in data:
        raise DataValidationError('age', None, 'Missing required field')
    if data['age'] < 0:
        raise DataValidationError('age', data['age'], 'Age cannot be negative')
```

---

## 8. 실전 예제 시나리오

### 시나리오 1: 대용량 CSV 파일 처리

**문제:** 10GB CSV 파일을 처리하면 메모리 부족

```python
import pandas as pd
import numpy as np
from pathlib import Path
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class LargeCSVProcessor:
    """대용량 CSV 파일을 효율적으로 처리하는 클래스"""
    
    def __init__(self, input_file, output_file, chunk_size=100000):
        self.input_file = input_file
        self.output_file = output_file
        self.chunk_size = chunk_size
    
    def process_chunk(self, chunk):
        """개별 청크 처리 로직"""
        # 예: 특정 조건 필터링
        chunk = chunk[chunk['value'] > 0]
        
        # 데이터 타입 최적화
        chunk = self.optimize_dtypes(chunk)
        
        # 새로운 컬럼 추가
        chunk['processed_value'] = chunk['value'] * 2
        
        return chunk
    
    def optimize_dtypes(self, df):
        """메모리 사용량 최적화"""
        for col in df.select_dtypes(include=['int']).columns:
            df[col] = pd.to_numeric(df[col], downcast='integer')
        
        for col in df.select_dtypes(include=['float']).columns:
            df[col] = pd.to_numeric(df[col], downcast='float')
        
        return df
    
    def process(self):
        """전체 파일 처리"""
        logger.info(f"Processing {self.input_file}")
        
        # 첫 번째 청크로 헤더 생성
        first_chunk = True
        total_rows = 0
        
        try:
            for i, chunk in enumerate(pd.read_csv(
                self.input_file, 
                chunksize=self.chunk_size,
                low_memory=False
            )):
                logger.info(f"Processing chunk {i+1} ({len(chunk)} rows)")
                
                # 청크 처리
                processed = self.process_chunk(chunk)
                
                # 결과 저장
                processed.to_csv(
                    self.output_file,
                    mode='w' if first_chunk else 'a',
                    header=first_chunk,
                    index=False
                )
                
                first_chunk = False
                total_rows += len(processed)
                
                logger.info(f"Total processed: {total_rows} rows")
        
        except Exception as e:
            logger.error(f"Error processing file: {e}")
            raise
        
        logger.info(f"Processing complete. Total rows: {total_rows}")
        return total_rows

# 사용 예제
if __name__ == '__main__':
    processor = LargeCSVProcessor(
        input_file='huge_data.csv',
        output_file='processed_data.csv',
        chunk_size=100000
    )
    
    total = processor.process()
    print(f"Processed {total} rows successfully")
```

### 시나리오 2: 병렬 처리로 성능 향상

**문제:** 수백만 개의 이미지를 처리해야 하는데 너무 느림

```python
from multiprocessing import Pool, cpu_count
from pathlib import Path
from PIL import Image
import time
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class ImageProcessor:
    """이미지 병렬 처리 클래스"""
    
    def __init__(self, input_dir, output_dir, target_size=(224, 224)):
        self.input_dir = Path(input_dir)
        self.output_dir = Path(output_dir)
        self.target_size = target_size
        self.output_dir.mkdir(parents=True, exist_ok=True)
    
    @staticmethod
    def process_single_image(args):
        """단일 이미지 처리 (정적 메서드 - pickle 가능)"""
        input_path, output_path, target_size = args
        
        try:
            # 이미지 로드
            img = Image.open(input_path)
            
            # 리사이즈
            img_resized = img.resize(target_size, Image.LANCZOS)
            
            # 저장
            img_resized.save(output_path, quality=95)
            
            return True, input_path.name
        
        except Exception as e:
            logger.error(f"Error processing {input_path}: {e}")
            return False, input_path.name
    
    def get_image_files(self):
        """처리할 이미지 파일 목록 가져오기"""
        extensions = {'.jpg', '.jpeg', '.png', '.bmp'}
        return [
            f for f in self.input_dir.iterdir() 
            if f.suffix.lower() in extensions
        ]
    
    def process_sequential(self):
        """순차 처리 (비교용)"""
        logger.info("Starting sequential processing...")
        start_time = time.time()
        
        files = self.get_image_files()
        results = []
        
        for input_path in files:
            output_path = self.output_dir / input_path.name
            result = self.process_single_image(
                (input_path, output_path, self.target_size)
            )
            results.append(result)
        
        elapsed = time.time() - start_time
        success_count = sum(1 for r, _ in results if r)
        
        logger.info(f"Sequential: {success_count}/{len(files)} images in {elapsed:.2f}s")
        return results
    
    def process_parallel(self, num_workers=None):
        """병렬 처리"""
        if num_workers is None:
            num_workers = cpu_count()
        
        logger.info(f"Starting parallel processing with {num_workers} workers...")
        start_time = time.time()
        
        # 작업 목록 생성
        files = self.get_image_files()
        tasks = [
            (f, self.output_dir / f.name, self.target_size)
            for f in files
        ]
        
        # 병렬 처리
        with Pool(processes=num_workers) as pool:
            results = pool.map(self.process_single_image, tasks)
        
        elapsed = time.time() - start_time
        success_count = sum(1 for r, _ in results if r)
        
        logger.info(f"Parallel: {success_count}/{len(files)} images in {elapsed:.2f}s")
        logger.info(f"Speedup: {len(files)/elapsed:.2f} images/sec")
        
        return results

# 사용 예제
if __name__ == '__main__':
    processor = ImageProcessor(
        input_dir='./input_images',
        output_dir='./output_images',
        target_size=(224, 224)
    )
    
    # 순차 처리
    processor.process_sequential()
    
    # 병렬 처리
    processor.process_parallel(num_workers=4)
```

### 시나리오 3: 복잡한 데이터 파이프라인 디버깅

**문제:** 여러 단계를 거치는 데이터 처리 파이프라인에서 어디서 오류가 발생하는지 찾기 어려움

```python
import pandas as pd
import logging
from functools import wraps
import time
from typing import Callable, Any

# 로깅 설정
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('pipeline.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

def log_step(func: Callable) -> Callable:
    """파이프라인 단계를 로깅하는 데코레이터"""
    @wraps(func)
    def wrapper(*args, **kwargs):
        step_name = func.__name__
        logger.info(f"Starting step: {step_name}")
        start_time = time.time()
        
        try:
            result = func(*args, **kwargs)
            elapsed = time.time() - start_time
            
            # 결과 정보 로깅
            if isinstance(result, pd.DataFrame):
                logger.info(f"Step {step_name} completed in {elapsed:.2f}s")
                logger.info(f"  Output shape: {result.shape}")
                logger.info(f"  Memory usage: {result.memory_usage(deep=True).sum() / 1024**2:.2f} MB")
            else:
                logger.info(f"Step {step_name} completed in {elapsed:.2f}s")
            
            return result
        
        except Exception as e:
            logger.error(f"Step {step_name} failed: {str(e)}")
            logger.exception("Full traceback:")
            raise
    
    return wrapper

class DataPipeline:
    """데이터 처리 파이프라인"""
    
    def __init__(self, data):
        self.data = data
        self.history = []
    
    @log_step
    def load_data(self, filepath):
        """데이터 로드"""
        logger.info(f"Loading data from {filepath}")
        self.data = pd.read_csv(filepath)
        self.history.append(('load', self.data.shape))
        return self
    
    @log_step
    def clean_data(self):
        """데이터 정제"""
        logger.info("Cleaning data...")
        
        # 결측치 확인
        missing = self.data.isnull().sum()
        if missing.any():
            logger.warning(f"Missing values found:\n{missing[missing > 0]}")
        
        # 결측치 처리
        self.data = self.data.dropna()
        
        # 중복 제거
        duplicates = self.data.duplicated().sum()
        if duplicates > 0:
            logger.warning(f"Found {duplicates} duplicate rows")
            self.data = self.data.drop_duplicates()
        
        self.history.append(('clean', self.data.shape))
        return self
    
    @log_step
    def validate_data(self):
        """데이터 검증"""
        logger.info("Validating data...")
        
        # 데이터 타입 검증
        expected_types = {
            'id': 'int64',
            'name': 'object',
            'value': 'float64'
        }
        
        for col, expected_type in expected_types.items():
            if col in self.data.columns:
                actual_type = str(self.data[col].dtype)
                if actual_type != expected_type:
                    logger.warning(
                        f"Column {col} has type {actual_type}, expected {expected_type}"
                    )
        
        # 범위 검증
        if 'value' in self.data.columns:
            invalid = self.data[self.data['value'] < 0]
            if not invalid.empty:
                logger.error(f"Found {len(invalid)} rows with negative values")
                raise ValueError("Negative values not allowed in 'value' column")
        
        self.history.append(('validate', self.data.shape))
        return self
    
    @log_step
    def transform_data(self):
        """데이터 변환"""
        logger.info("Transforming data...")
        
        # 새로운 컬럼 추가
        if 'value' in self.data.columns:
            self.data['value_squared'] = self.data['value'] ** 2
            self.data['value_log'] = self.data['value'].apply(
                lambda x: pd.np.log1p(x) if x > 0 else 0
            )
        
        self.history.append(('transform', self.data.shape))
        return self
    
    @log_step
    def aggregate_data(self, group_by):
        """데이터 집계"""
        logger.info(f"Aggregating data by {group_by}...")
        
        if group_by not in self.data.columns:
            raise ValueError(f"Column {group_by} not found in data")
        
        self.data = self.data.groupby(group_by).agg({
            'value': ['mean', 'sum', 'count'],
            'value_squared': 'mean'
        }).reset_index()
        
        self.history.append(('aggregate', self.data.shape))
        return self
    
    @log_step
    def save_data(self, filepath):
        """데이터 저장"""
        logger.info(f"Saving data to {filepath}")
        self.data.to_csv(filepath, index=False)
        self.history.append(('save', self.data.shape))
        return self
    
    def print_history(self):
        """파이프라인 히스토리 출력"""
        logger.info("Pipeline execution history:")
        for step, shape in self.history:
            logger.info(f"  {step}: {shape}")

# 사용 예제
if __name__ == '__main__':
    try:
        # 파이프라인 실행
        pipeline = DataPipeline(data=None)
        pipeline \
            .load_data('input.csv') \
            .clean_data() \
            .validate_data() \
            .transform_data() \
            .aggregate_data('category') \
            .save_data('output.csv')
        
        # 히스토리 출력
        pipeline.print_history()
        
        logger.info("Pipeline completed successfully!")
    
    except Exception as e:
        logger.error(f"Pipeline failed: {e}")
        raise
```

### 시나리오 4: 메모리 프로파일링

**문제:** 프로그램이 메모리를 너무 많이 사용함

```python
from memory_profiler import profile
import numpy as np

@profile
def memory_intensive_function():
    """메모리 사용량이 많은 함수"""
    # 큰 배열 생성
    large_array = np.random.rand(10000, 10000)  # ~800MB
    
    # 처리
    result = large_array ** 2
    
    # 메모리 해제를 위해 del 사용
    del large_array
    
    return result

# 실행
# python -m memory_profiler script.py

# 결과:
# Line #    Mem usage    Increment  Occurrences   Line Contents
# ============================================================
#      5   45.7 MiB   45.7 MiB           1   @profile
#      6                                     def memory_intensive_function():
#      8  845.7 MiB  800.0 MiB           1       large_array = np.random.rand(10000, 10000)
#     11 1645.7 MiB  800.0 MiB           1       result = large_array ** 2
#     14  845.7 MiB -800.0 MiB           1       del large_array
#     16  845.7 MiB    0.0 MiB           1       return result
```

**메모리 최적화 예제:**

```python
import numpy as np
from memory_profiler import profile

@profile
def optimized_function():
    """메모리 효율적인 버전"""
    # 청크로 나눠서 처리
    chunk_size = 1000
    total_size = 10000
    
    results = []
    for i in range(0, total_size, chunk_size):
        chunk = np.random.rand(chunk_size, total_size)
        chunk_result = chunk ** 2
        results.append(chunk_result)
        del chunk  # 즉시 메모리 해제
    
    # 결과 합치기
    final_result = np.vstack(results)
    return final_result

# 또는 제너레이터 사용
def generate_chunks(size, chunk_size):
    """청크를 생성하는 제너레이터"""
    for i in range(0, size, chunk_size):
        chunk = np.random.rand(chunk_size, size)
        yield chunk ** 2

# 메모리 사용량을 더욱 줄임
result_iterator = generate_chunks(10000, 1000)
```

---

## 유용한 Python 도구 모음

### 디버깅 도구

```bash
# pdb - 기본 디버거
python -m pdb script.py

# ipdb - IPython 통합 디버거
pip install ipdb
python -m ipdb script.py

# pudb - 시각적 디버거
pip install pudb
python -m pudb script.py
```

### 프로파일링 도구

```bash
# cProfile - CPU 프로파일링
python -m cProfile -s cumulative script.py

# line_profiler - 라인별 프로파일링
pip install line_profiler
kernprof -l -v script.py

# memory_profiler - 메모리 프로파일링
pip install memory_profiler
python -m memory_profiler script.py

# py-spy - 실시간 프로파일링
pip install py-spy
py-spy top -- python script.py
```

### 코드 품질 도구

```bash
# pylint - 정적 분석
pip install pylint
pylint script.py

# flake8 - 스타일 검사
pip install flake8
flake8 script.py

# black - 자동 포매팅
pip install black
black script.py

# mypy - 타입 체킹
pip install mypy
mypy script.py
```

---

## Python 트러블슈팅 체크리스트 ✅

### 환경 문제
- [ ] Python 버전이 올바른가?
- [ ] 가상환경이 활성화되었는가?
- [ ] PATH가 올바르게 설정되었는가?
- [ ] pip가 올바른 Python을 가리키는가?

### 패키지 문제
- [ ] 패키지가 설치되었는가?
- [ ] 버전 충돌이 없는가?
- [ ] requirements.txt가 최신인가?
- [ ] 가상환경을 사용하는가?

### Import 문제
- [ ] 모듈 이름이 정확한가?
- [ ] PYTHONPATH가 올바른가?
- [ ] 순환 import가 없는가?
- [ ] __init__.py가 있는가?

### 성능 문제
- [ ] 불필요한 반복문이 있는가?
- [ ] NumPy 벡터화를 사용할 수 있는가?
- [ ] 메모리 사용량이 적절한가?
- [ ] 병렬 처리가 필요한가?

---

## 추가 팁 💡

### 1. 가상환경 관리 Best Practices

```bash
# 프로젝트별 가상환경 생성
python3 -m venv venv

# 활성화
source venv/bin/activate  # Linux/macOS
venv\Scripts\activate     # Windows

# requirements.txt 생성
pip freeze > requirements.txt

# requirements.txt에서 설치
pip install -r requirements.txt

# 가상환경 비활성화
deactivate
```

### 2. 디버깅 팁

```python
# 1. print 대신 pprint 사용
from pprint import pprint
pprint(complex_dict)

# 2. breakpoint() 활용 (Python 3.7+)
def function():
    x = 10
    breakpoint()  # 여기서 디버거 시작
    return x * 2

# 3. assert로 조건 검증
assert len(data) > 0, "Data is empty!"

# 4. try-except로 예외 처리
try:
    result = risky_operation()
except Exception as e:
    print(f"Error: {e}")
    import traceback
    traceback.print_exc()
```

### 3. 성능 최적화 팁

```python
# 1. List comprehension 사용
# 느림
result = []
for i in range(1000):
    result.append(i ** 2)

# 빠름
result = [i ** 2 for i in range(1000)]

# 2. Generator 사용 (메모리 절약)
def generate_squares(n):
    for i in range(n):
        yield i ** 2

# 3. 내장 함수 활용
# 느림
result = []
for item in data:
    result.append(item * 2)

# 빠름
result = list(map(lambda x: x * 2, data))

# 4. set을 활용한 조회
# 느림 - O(n)
if item in my_list:
    pass

# 빠름 - O(1)
if item in my_set:
    pass
```

---

## 결론

Python 트러블슈팅의 핵심:

1. **가상환경 사용**: 프로젝트별로 독립적인 환경 유지
2. **로깅 활용**: print 대신 logging 모듈 사용
3. **프로파일링**: 성능 문제는 추측하지 말고 측정
4. **예외 처리**: 적절한 try-except로 에러 핸들링
5. **코드 리뷰**: pylint, flake8 등으로 코드 품질 유지

Happy Python Coding! 🐍
