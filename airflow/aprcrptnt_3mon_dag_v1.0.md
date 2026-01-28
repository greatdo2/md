# 데이터베이스 마이그레이션 소스코드 초보자 가이드

## 📚 목차
1. [전체 구조 이해하기](#1-전체-구조-이해하기)
2. [DAG 파일 상세 설명](#2-dag-파일-상세-설명)
3. [DBConnector 클래스 상세 설명](#3-dbconnector-클래스-상세-설명)
4. [핵심 개념 정리](#4-핵심-개념-정리)
5. [주요 수정사항 요약](#5-주요-수정사항-요약)

---

## 1. 전체 구조 이해하기

### 🎯 이 프로그램이 하는 일
```
┌─────────────┐         ┌──────────────┐
│   Tibero    │  복사   │  PostgreSQL  │
│  (원본 DB)  │  ────>  │   (대상 DB)  │
└─────────────┘         └──────────────┘
     환자 진료 데이터 마이그레이션
```

**쉽게 말하면:**
- Tibero 데이터베이스에 저장된 환자 진료 데이터를 읽어서
- PostgreSQL 데이터베이스로 복사하는 프로그램입니다
- 이 작업을 Airflow(에어플로우)라는 도구로 자동화합니다

---

## 2. DAG 파일 상세 설명

### 📋 파일명: `tibero_aprcrptnt_mig_3month_fixed.py`

### 2.1 전체 흐름도
```
시작 → 테이블 생성 → 데이터 복사 → 검증 → 종료
 ↓         ↓            ↓         ↓      ↓
start → create → migrate → verify → end
```

### 2.2 주요 함수들 설명

#### 🔧 함수 1: `create_table_if_not_exists()`
**목적:** PostgreSQL에 데이터를 받을 테이블을 만듭니다

```python
def create_table_if_not_exists():
```

**단계별 설명:**

```python
# 1단계: 연결 정보 준비
target_db_sid = MigTaskConfig.target_phis_mig_db_sid
# 👆 설정 파일에서 데이터베이스 연결 정보를 가져옵니다
```

```python
# 2단계: 테이블 구조 정의
create_table_query = """
    CREATE TABLE IF NOT EXISTS aprcrptnt_mig (
        mdrp_no VARCHAR(50),      -- 진료예약번호
        ptno VARCHAR(50),          -- 환자번호
        ptnt_nm VARCHAR(100),      -- 환자이름
        mdcr_ymd VARCHAR(8),       -- 진료일자 (YYYYMMDD)
        ...
    )
"""
# 👆 어떤 컬럼(열)들이 필요한지 정의합니다
```

```python
# 3단계: try-except-finally 패턴 (안전한 실행)
try:
    # 데이터베이스에 연결
    target_db = DBConnector(...)
    # 테이블 생성 실행
    target_db.execute(create_table_query)
    
except Exception as e:
    # 에러가 발생하면 로그에 기록
    log.error(f"Error: {e}")
    
finally:
    # 성공하든 실패하든 연결은 꼭 닫습니다
    target_db.close()
```

**왜 이렇게 하나요?**
- 테이블이 이미 있으면 다시 만들지 않습니다 (`IF NOT EXISTS`)
- 연결은 반드시 닫아야 합니다 (메모리 누수 방지)

---

#### 🚚 함수 2: `migrate_aprcrptnt_data()`
**목적:** 실제로 데이터를 복사합니다

**핵심 수정사항: 날짜 데이터 처리**

```python
# 문제가 있던 원본 코드
sanitized_row = tuple(
    str(val) if isinstance(val, datetime) else val 
    for val in row
)
# ❌ 모든 datetime을 문자열로 바꿔버림
# 결과: '2026-02-11 00:00:00' → 19자 → VARCHAR(8) 에러!
```

```python
# 수정된 코드
for idx, val in enumerate(row):
    # mdcr_ymd 컬럼 (index 3)만 특별 처리
    if idx == 3:  
        if isinstance(val, datetime):
            sanitized_row.append(val.strftime('%Y%m%d'))
            # ✅ '20260211' 형식으로 변환 → 8자
        elif isinstance(val, str):
            sanitized_row.append(val.replace('-', '')[:8])
            # ✅ '2026-02-11' → '20260211'
    # 나머지 datetime은 문자열로
    elif isinstance(val, datetime):
        sanitized_row.append(str(val))
```

**시각적 비교:**
```
원본 데이터:   2026-02-11 00:00:00
               ↓
잘못된 변환:   '2026-02-11 00:00:00'  (19자) ❌
               ↓
올바른 변환:   '20260211'             (8자)  ✅
```

**전체 흐름:**
```
1. Tibero 연결
   ↓
2. 데이터 조회 (SELECT)
   ↓
3. 데이터가 있나? 
   없으면 → 종료
   있으면 → 계속
   ↓
4. 데이터 전처리
   - 날짜 형식 변환
   - NULL 처리
   ↓
5. PostgreSQL 연결
   ↓
6. 데이터 삽입 (INSERT)
   - 1000개씩 배치로 처리
   ↓
7. 연결 종료
```

---

#### ✅ 함수 3: `verify_migration()`
**목적:** 데이터가 잘 들어갔는지 확인합니다

```python
verify_query = """
    SELECT COUNT(*) as total_count
    FROM aprcrptnt_mig
"""
# 👆 PostgreSQL에 몇 개의 레코드가 있는지 세어봅니다
```

**로직:**
```
PostgreSQL 연결
  ↓
레코드 개수 조회
  ↓
로그에 기록: "Total records: 1234"
  ↓
연결 종료
```

---

### 2.3 DAG 구조 설명

```python
with DAG(
    "tibero_APRCRPTNT_mig_3month",  # DAG 이름
    default_args={
        "retries": 3,                # 실패하면 3번 재시도
        "retry_delay": timedelta(minutes=5),  # 5분 후 재시도
    },
    schedule_interval=None,          # 수동 실행 (자동 실행 안 함)
    catchup=False,                   # 과거 데이터 소급 적용 안 함
) as dag:
```

**Task 실행 순서:**
```python
start_task >> create_table_task >> migrate_task >> verify_task >> end_task
# 👆 ">>" 기호는 "다음에 실행"을 의미합니다
```

**시각적 표현:**
```
[start_task]
     ↓
[create_table_task]  ← 테이블 생성
     ↓
[migrate_task]       ← 데이터 복사
     ↓
[verify_task]        ← 검증
     ↓
[end_task]
```

---

## 3. DBConnector 클래스 상세 설명

### 📋 파일명: `db_connector_improved.py`

### 3.1 클래스 개요

```python
class DBConnector:
    """데이터베이스에 연결하고 쿼리를 실행하는 도구"""
```

**역할:**
- 데이터베이스 연결 관리
- SQL 쿼리 실행
- 에러 처리
- 리소스 관리

---

### 3.2 주요 개선사항

#### 🔍 개선 1: 더 나은 에러 메시지

**기존 코드:**
```python
except Exception as e:
    self.log.warning(f"Connection error: {e}")
```

**개선된 코드:**
```python
except Exception as e:
    self.log.error(f"Connection failed for {self.db_type}: {e}")
    import traceback
    self.log.error(f"Full traceback:\n{traceback.format_exc()}")
    # 👆 에러가 어디서 발생했는지 자세히 보여줍니다
```

**실제 에러 로그 예시:**
```
[ERROR] Connection failed for postgresql: Connection refused
[ERROR] Full traceback:
  File "db_connector.py", line 150, in connect
    self.con = jdbc.connect(...)
  jdbc.Error: FATAL: password authentication failed
  
👆 어디서 문제가 생겼는지 한눈에 파악 가능!
```

---

#### 🛡️ 개선 2: Context Manager 지원

**기존 방식:**
```python
db = DBConnector(...)
try:
    db.execute(query)
finally:
    db.close()  # ← 깜빡하면 문제 발생!
```

**개선된 방식:**
```python
with DBConnector(...) as db:
    db.execute(query)
# ← 자동으로 close() 호출됨!
```

**왜 좋나요?**
- 연결을 닫는 것을 깜빡할 수 없습니다
- 에러가 나도 자동으로 정리됩니다
- Python의 `with` 문법 사용

---

#### 📦 개선 3: 배치 처리 개선

**기존 코드:**
```python
cursor.executemany(sql_text, rows)
self.con.commit()
# ❌ 10,000개 레코드를 한번에 처리 → 메모리 부족 가능
```

**개선된 코드:**
```python
for i in range(0, len(rows), batch_size):
    batch = rows[i:i + batch_size]  # 1000개씩 나눔
    cursor.executemany(sql_text, batch)
    self.con.commit()  # 배치마다 저장
    log.info(f"Batch {i//1000 + 1} committed")
```

**시각적 비교:**
```
기존: [10,000개 한번에] ❌ 메모리 부족!

개선: [1,000개] → 저장
     [1,000개] → 저장
     [1,000개] → 저장
     ...
     ✅ 안전하고 빠름!
```

---

#### 🔄 개선 4: 트랜잭션 관리

```python
try:
    cursor.executemany(sql_text, batch)
    self.con.commit()  # 성공하면 저장
except:
    self.con.rollback()  # 실패하면 되돌림
    raise
```

**트랜잭션이란?**
```
은행 거래 예시:
1. A 계좌에서 -10,000원  ← 성공
2. B 계좌에 +10,000원     ← 실패!

트랜잭션 없으면: A의 돈만 없어짐 😱
트랜잭션 있으면: 둘 다 취소됨 ✅
```

---

#### 🔌 개선 5: 연결 상태 확인

```python
def is_connected(self):
    """데이터베이스가 아직 연결되어 있나요?"""
    try:
        cursor = self.con.cursor()
        cursor.execute("SELECT 1")  # 간단한 쿼리
        cursor.close()
        return True  # 연결됨!
    except:
        return False  # 연결 끊김!
```

**사용 예:**
```python
db = DBConnector(...)
if db.is_connected():
    print("연결 정상")
else:
    print("연결 끊김 - 재연결 필요")
```

---

### 3.3 주요 메서드 설명

#### 메서드 1: `connect()`
**목적:** 데이터베이스 연결

```python
def connect(self):
    # 1. JAR 파일 확인 (JDBC 드라이버)
    if not os.path.exists(self.jdbc_driver):
        raise FileNotFoundError("드라이버 없음!")
    
    # 2. JVM이 처음 시작되면 모든 JAR 로드
    if DBConnector._first_connection:
        all_jars = [tibero_jar, postgresql_jar]
        jars_to_load = all_jars
    
    # 3. 연결 시도
    self.con = jdbc.connect(
        self.driver,      # 드라이버 이름
        self.url,         # 접속 주소
        [user, password], # 인증 정보
        jars=jars_to_load # JAR 파일
    )
    
    # 4. Auto-commit 끄기 (수동 관리)
    self.con.jconn.setAutoCommit(False)
```

**JAR 파일이란?**
- Java로 작성된 데이터베이스 드라이버
- 각 DB마다 다른 JAR 필요 (Tibero, PostgreSQL)
- 파이썬이 Java DB와 통신하려면 필요

---

#### 메서드 2: `select_fetchall()`
**목적:** 데이터 조회

```python
def select_fetchall(self, sql_text, bind_params=None):
    cursor = self.con.cursor()
    
    # 파라미터가 있으면 바인딩
    if bind_params:
        cursor.execute(sql_text, bind_params)
    else:
        cursor.execute(sql_text)
    
    # 결과 가져오기
    rows = cursor.fetchall()
    column_names = [desc[0] for desc in cursor.description]
    
    return rows, column_names
```

**실제 사용 예:**
```python
query = "SELECT * FROM users WHERE age > ?"
rows, columns = db.select_fetchall(query, [20])

# 결과:
# columns = ['id', 'name', 'age']
# rows = [(1, 'Alice', 25), (2, 'Bob', 30)]
```

---

#### 메서드 3: `execute_many()`
**목적:** 대량 데이터 삽입

```python
def execute_many(self, sql_text, rows, batch_size=1000):
    total_rows = len(rows)
    
    # 1000개씩 나눠서 처리
    for i in range(0, total_rows, batch_size):
        batch = rows[i:i + batch_size]
        
        # 배치 번호 계산
        batch_num = i // batch_size + 1
        total_batches = (total_rows + batch_size - 1) // batch_size
        
        log.info(f"Processing batch {batch_num}/{total_batches}")
        
        try:
            cursor.executemany(sql_text, batch)
            self.con.commit()
            log.info(f"Batch {batch_num} committed ✅")
        except:
            self.con.rollback()  # 실패하면 되돌리기
            raise
```

**배치 처리 흐름:**
```
전체 데이터: 3,500개

Batch 1/4: [0 ~ 999]    → 저장 ✅
Batch 2/4: [1000 ~ 1999] → 저장 ✅
Batch 3/4: [2000 ~ 2999] → 저장 ✅
Batch 4/4: [3000 ~ 3499] → 저장 ✅

완료! 총 3,500개 저장됨
```

---

## 4. 핵심 개념 정리

### 🎓 개념 1: try-except-finally

```python
try:
    # 시도할 코드
    db = DBConnector(...)
    db.execute(query)
    
except Exception as e:
    # 에러 발생 시 실행
    log.error(f"Error: {e}")
    
finally:
    # 항상 실행 (성공/실패 무관)
    db.close()
```

**실생활 비유:**
```
try:
    문을 연다
    방에 들어간다
    일을 한다
except:
    문제 생기면 경비에게 알린다
finally:
    무조건 문을 닫는다  ← 중요!
```

---

### 🎓 개념 2: Context Manager (with 문)

```python
# 수동 관리 (번거로움)
file = open('data.txt')
try:
    data = file.read()
finally:
    file.close()

# Context Manager (편리함)
with open('data.txt') as file:
    data = file.read()
# 자동으로 닫힘!
```

---

### 🎓 개념 3: enumerate()

```python
items = ['A', 'B', 'C']

# 기본 for문
for item in items:
    print(item)
# 출력: A, B, C

# enumerate 사용
for idx, item in enumerate(items):
    print(f"{idx}: {item}")
# 출력: 0: A, 1: B, 2: C
```

**마이그레이션 코드에서 사용:**
```python
for idx, val in enumerate(row):
    if idx == 3:  # 4번째 컬럼 (mdcr_ymd)
        # 특별 처리
```

---

### 🎓 개념 4: isinstance()

```python
val = datetime(2026, 2, 11)

if isinstance(val, datetime):
    print("날짜입니다!")
    # ✅ 실행됨

if isinstance(val, str):
    print("문자열입니다!")
    # ❌ 실행 안 됨
```

---

### 🎓 개념 5: 리스트 슬라이싱

```python
numbers = [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]

# 처음부터 3번째까지
numbers[:3]    # [0, 1, 2]

# 3번째부터 6번째까지
numbers[3:6]   # [3, 4, 5]

# 처음 8글자만
text = "2026-02-11"
text[:8]       # "2026-02-"
```

---

## 5. 주요 수정사항 요약

### ❌ 문제: VARCHAR(8) 길이 초과 에러

**에러 메시지:**
```
ERROR: value too long for type character varying(8)
```

**원인:**
```
데이터베이스 정의: mdcr_ymd VARCHAR(8)  ← 8자만 허용
실제 저장 시도:   '2026-02-11 00:00:00'  ← 19자
```

---

### ✅ 해결 방법

**1단계: 데이터 타입 확인**
```python
# mdcr_ymd 컬럼이 datetime 객체로 오는지 확인
if isinstance(val, datetime):
    # datetime → 문자열 변환
```

**2단계: 올바른 형식으로 변환**
```python
# YYYYMMDD 형식으로 변환 (8자리)
formatted = val.strftime('%Y%m%d')
# 2026-02-11 → '20260211'
```

**3단계: 예외 처리**
```python
# 이미 문자열인 경우 대비
if isinstance(val, str):
    formatted = val.replace('-', '')[:8]
    # '2026-02-11' → '20260211'
```

---

### 📊 수정 전후 비교

| 구분 | 수정 전 | 수정 후 |
|------|---------|---------|
| 데이터 형식 | `'2026-02-11 00:00:00'` | `'20260211'` |
| 길이 | 19자 ❌ | 8자 ✅ |
| 결과 | 에러 발생 | 정상 작동 |

---

## 6. 실전 활용 팁

### 💡 팁 1: 로그 확인하기

```python
# 로그 레벨
log.debug("디버그 정보")    # 개발 중
log.info("일반 정보")       # 진행 상황
log.warning("경고")         # 주의 필요
log.error("에러")           # 문제 발생
```

**Airflow 로그 위치:**
```
Airflow UI → DAGs → 해당 DAG 클릭 → Logs
```

---

### 💡 팁 2: 데이터 검증

```python
# 마이그레이션 전
SELECT COUNT(*) FROM APRCRPTNT;  -- Tibero
→ 결과: 10,000건

# 마이그레이션 후
SELECT COUNT(*) FROM aprcrptnt_mig;  -- PostgreSQL
→ 결과: 10,000건  ✅ 일치!
```

---

### 💡 팁 3: 테스트 방법

```python
# 1. 소량 테스트
WHERE a.MDCR_YMD >= TRUNC(SYSDATE) - 1  -- 어제 데이터만
# 데이터 적으니 빠르게 테스트 가능

# 2. 본격 실행
WHERE a.MDCR_YMD >= ADD_MONTHS(TRUNC(SYSDATE), -3)  -- 3개월
```

---

### 💡 팁 4: 에러 대처법

**에러가 나면:**
1. 로그 확인 (`log.error` 메시지 찾기)
2. 에러 메시지의 키워드 파악
3. Traceback으로 어느 줄에서 에러났는지 확인
4. 해당 부분 디버깅

**예시:**
```
[ERROR] Type error during batch execution: 
        invalid literal for int() with base 10: '20260211'
        
→ 해결: 숫자가 아닌 문자열을 숫자로 변환하려 했음
       데이터 타입 확인 필요
```

---

## 7. 마무리

### ✅ 체크리스트

배포 전 확인사항:
- [ ] 테이블이 올바르게 생성되는가?
- [ ] 데이터가 정상적으로 복사되는가?
- [ ] 날짜 형식이 올바른가? (VARCHAR(8))
- [ ] 에러 처리가 되어 있는가?
- [ ] 연결이 제대로 닫히는가?
- [ ] 로그가 충분히 남는가?

---

### 🎯 핵심 요약

1. **문제:** 날짜 데이터가 19자로 저장되어 VARCHAR(8) 초과
2. **해결:** datetime을 'YYYYMMDD' 형식(8자)으로 변환
3. **개선:** 에러 처리, 배치 처리, 리소스 관리 강화

---

### 📚 더 공부하면 좋은 것들

- Python Context Manager 심화
- JDBC와 jaydebeapi 이해
- Airflow DAG 작성 기초
- SQL 트랜잭션 관리
- 에러 핸들링 베스트 프랙티스

---

## 부록: 용어 사전

| 용어 | 설명 | 예시 |
|------|------|------|
| DAG | 작업 흐름을 정의한 그래프 | A → B → C |
| Migration | 데이터를 다른 DB로 이동 | Tibero → PostgreSQL |
| Batch | 데이터를 묶음으로 처리 | 1000개씩 |
| Transaction | 하나의 작업 단위 | 전체 성공 or 전체 실패 |
| Context Manager | 자동으로 정리해주는 구조 | `with` 문 |
| JDBC | Java가 DB와 통신하는 방법 | tibero6-jdbc.jar |
| Cursor | DB 쿼리를 실행하는 도구 | cursor.execute() |
| Rollback | 실패 시 이전 상태로 복구 | 되돌리기 |
| Commit | 변경사항 저장 | 저장하기 |

---

**작성일:** 2026-01-28  
**버전:** 1.0  
**대상:** 초보 개발자
