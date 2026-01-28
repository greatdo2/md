# CCOM 프로젝트 상세 분석서

## 목차
1. [프로젝트 개요](#프로젝트-개요)
2. [프로젝트 구조](#프로젝트-구조)
3. [CPCOM - 엑셀 출력 및 로깅](#cpcom---엑셀-출력-및-로깅)
4. [CMCommon - 공통 유틸리티](#cmcommon---공통-유틸리티)
5. [CMC001U1 - 공통코드 컴포넌트](#cmc001u1---공통코드-컴포넌트)
6. [GridToExcelEngine - 그리드 엑셀 엔진](#gridtoexcelengine---그리드-엑셀-엔진)
7. [CMSaveGrid - 그리드 레이아웃 관리](#cmsavegrid---그리드-레이아웃-관리)
8. [CMVersionCheck - 버전 체크](#cmversioncheck---버전-체크)
9. [CallHrmEnc - HRM 암호화](#callhrmenc---hrm-암호화)
10. [chrome_cefvcl - Chromium 브라우저](#chrome_cefvcl---chromium-브라우저)
11. [SMCLogData - 로그 데이터 관리](#smclogdata---로그-데이터-관리)
12. [CMCCodeData - 공통 상수 데이터](#cmccodedata---공통-상수-데이터)
13. [핵심 기능 상세 분석](#핵심-기능-상세-분석)
14. [주요 클래스 및 함수](#주요-클래스-및-함수)

---

## 프로젝트 개요

**CCOM (Common Component)**는 Nexmed EHR 시스템의 공통 컴포넌트 패키지로, 애플리케이션 전반에서 사용되는 공통 기능들을 제공합니다. 이 프로젝트는 다음과 같은 주요 역할을 담당합니다:

- **엑셀 출력**: 그리드 데이터를 엑셀로 내보내기
- **로깅**: 출력물 및 사용자 작업 로깅
- **공통코드 관리**: 공통코드 조회 및 관리 컴포넌트
- **그리드 관리**: 그리드 레이아웃 저장/로드
- **버전 체크**: BPL 패키지 버전 확인
- **HRM 연동**: HRM 시스템 연동 및 암호화
- **웹 브라우저**: Chromium 기반 웹 브라우저 컴포넌트
- **공통 유틸리티**: 메모리 관리, 폼 활성화 체크 등

---

## 프로젝트 구조

```
CCOM/
├── CCOMPACK.dpk              # 메인 패키지 파일
├── CPCOM.pas                 # 엑셀 출력 및 로깅 메인 유닛
├── CMCommon.pas              # 공통 유틸리티
├── CMC001M1.pas              # 공통 데이터 모듈
├── CMC001U1.pas              # 공통코드 컴포넌트
├── CMC001U2.pas              # 공통코드 조회 폼
├── CMC001U4.pas              # 사용자 조회 폼
├── GridToExcelEngine.pas     # 그리드 엑셀 엔진
├── CMSaveGrid.pas            # 그리드 레이아웃 저장/로드
├── CMVersionCheck.pas        # 버전 체크
├── CallHrmEnc.pas            # HRM 암호화
├── chrome_cefvcl.pas         # Chromium 브라우저 컴포넌트
├── SMCLogData.pas            # 로그 데이터 관리
├── CMCCodeData.pas           # 공통 상수 데이터
└── ...
```

---

## CPCOM - 엑셀 출력 및 로깅

### 개요
**CPCOM**은 그리드 데이터를 엑셀로 내보내고, 출력물에 대한 로깅을 수행하는 핵심 유닛입니다.

### 주요 함수

#### `GridToExcel`
DevExpress 그리드를 엑셀로 내보내는 함수입니다. 여러 오버로드 버전이 있습니다.

**함수 시그니처:**
```pascal
// 기본 버전
procedure GridToExcel(AGrid: TcxControl; APgmId, AFileName, ATitle: string; 
  ExcelConditionArray: TStringList; AExpand: Boolean; 
  APageOrientation, AConfidential: Integer);

// 확장 버전 (열 자동 맞춤, 저장 위치 지정)
procedure GridToExcel(AGrid: TcxControl; APgmId, AFileName, ATitle: string; 
  ExcelConditionArray: TStringList; AExpand: Boolean; 
  APageOrientation, AConfidential: Integer; 
  AColumnAutoFit: Boolean; AUserFilePath: string);

// 최대 확장 버전 (모든 옵션)
procedure GridToExcel(AGrid: TcxControl; APgmId, AFileName, ATitle: string; 
  ExcelConditionArray: TStringList; AExpand: Boolean; 
  APageOrientation, AConfidential: Integer; 
  AColumnAutoFit: Boolean; AUserFilePath: string; 
  AOpenFile: Boolean; ANativeFormat: Boolean; 
  AFreezePane: Boolean; AFreezeColumn, AFreezeRow: Integer; 
  APreview: Boolean; ARowAutoFit: Boolean; APassWord: String);
```

**매개변수:**
- `AGrid`: 엑셀로 내보낼 그리드 컨트롤
- `APgmId`: 프로그램 ID (로깅용)
- `AFileName`: 저장할 파일명
- `ATitle`: 엑셀 제목
- `ExcelConditionArray`: 검색 조건 배열
- `AExpand`: 확장된 데이터 포함 여부
- `APageOrientation`: 페이지 방향 (1: 세로, 2: 가로)
- `AConfidential`: 기밀 등급
- `AColumnAutoFit`: 열 자동 맞춤 여부
- `AUserFilePath`: 사용자 지정 저장 경로
- `AOpenFile`: 파일 열기 여부
- `ANativeFormat`: 수식 컬럼 적용 여부
- `AFreezePane`: 틀 고정 여부
- `AFreezeColumn`, `AFreezeRow`: 틀 고정 위치
- `APreview`: 미리보기 여부
- `ARowAutoFit`: 행 자동 간격 맞춤
- `APassWord`: 파일 비밀번호

**동작 방식:**
1. 비밀번호 확인 (`GetPasswordCheck`)
2. 로깅 ID 생성 (`GetLoggingID`)
3. 엑셀 폴더 생성 (`IsExistExcelFolder`)
4. 파일명 정리 (`ExcelFileName` - 특수문자 제거)
5. 그리드 타입에 따라 적절한 내보내기 함수 호출
   - `TcxGrid`: `ExportSMCGridToExcelGridControl`
   - `TcxVirtualVerticalGrid`: `ExportSMCVerticalGridToExcel`
   - `TcxTreeList`: `TreeListToExcel`

#### `AdvGridToExcel`
TMS AdvStringGrid를 엑셀로 내보내는 함수입니다.

**함수 시그니처:**
```pascal
procedure AdvGridToExcel(AGrid: TAdvStringGrid; APgmId, AFileName, ATitle: string; 
  ExcelConditionArray: TStringList; AExpand: Boolean; 
  APageOrientation, AConfidential: Integer; 
  ASheetName: String; AMulti: Boolean; Visible: Boolean; 
  ACellText: Boolean; AColumnAutoFit: Boolean);
```

#### `TreeListToExcel`
DevExpress TreeList를 엑셀로 내보내는 함수입니다.

**함수 시그니처:**
```pascal
procedure TreeListToExcel(ATreeList: TcxCustomTreeList; APgmId, AFileName, ATitle: string; 
  ExcelConditionArray: TStringList; AExpand: Boolean; 
  APageOrientation, AConfidential: Integer; 
  AColumnAutoFit: Boolean; AUserFilePath: string);
```

#### `GetLoggingID`
출력물 로깅 ID를 생성하는 함수입니다.

**함수 시그니처:**
```pascal
function GetLoggingID(APrivateGubun, APgmId, APrintGubun, AUserId: string; 
  QueryCondition: String; workCct: integer): String;
```

**동작 방식:**
1. 개인정보 여부 확인 (`GetPersonalInfoYN`)
2. 개인정보 사유 입력 (`ShowPersonalInfo`)
3. 로깅 ID 생성 (타임스탬프 + 사용자ID)
4. 서버에 로깅 정보 전송 (`dp_IniflmtI01`)
5. 로깅 ID 반환

**반환값:**
- 성공: `타임스탬프_사용자ID` 형식의 문자열
- 실패: `'LogFailed'` 또는 `'PersonalCancel'`

#### `GetPrintLoggingID`
인쇄물 로깅 ID를 생성하는 함수입니다.

**함수 시그니처:**
```pascal
function GetPrintLoggingID(APrivateGubun, APgmId, APrintGubun, AUserId: string; 
  QueryCondition: String; PersonalCnte: string; workCct: integer): String;
```

**차이점:**
- `GetLoggingID`: 엑셀 다운로드용 (개인정보 사유 입력)
- `GetPrintLoggingID`: 인쇄물용 (개인정보 사유 미리 입력)

#### `ShowPersonalInfo`
개인정보 다운로드 사유를 입력받는 함수입니다.

**함수 시그니처:**
```pascal
function ShowPersonalInfo(APgmId: string; AForm: TForm; var VarCnte: String): integer;
```

**반환값:**
- `0`: 개인정보 체크 불필요
- `1`: 개인정보 사유 입력 완료
- `2`: 사용자 취소

#### `GetPasswordCheck`
엑셀 파일 다운로드 비밀번호 확인 함수입니다.

**함수 시그니처:**
```pascal
function GetPasswordCheck(APgmId: string; AForm: TForm): boolean;
```

**동작 방식:**
1. 서버에서 비밀번호 설정 여부 확인 (`GetPasswordCheckYN`)
2. 비밀번호가 설정되어 있으면 입력 다이얼로그 표시 (`Cmc001uf_ShowPassword`)
3. 비밀번호 일치 여부 반환

#### `BeginExcelForGrid` / `EndExcelForGrid`
엑셀 출력 시작/종료를 표시하는 함수입니다.

**함수 시그니처:**
```pascal
procedure BeginExcelForGrid(APrivateGubun, APgmId, APrintGubun, AUserId: string; 
  ExcelConditionArray: TStringList);
procedure EndExcelForGrid;
```

**용도:**
- 엑셀 출력 전 로깅 시작
- 엑셀 출력 후 정리 작업

#### `ExcelFileName`
파일명에서 특수문자를 제거하는 함수입니다.

**함수 시그니처:**
```pascal
function ExcelFileName(AFileName: string): String;
```

**제거되는 문자:**
`/`, `\`, `&`, `` ` ``, `@`, `#`, `$`, `%`, `^`, `*`, `|`, `?`, `,`, `'`, `"`

#### `GetQueryCondition`
조건 배열을 문자열로 변환하는 함수입니다.

**함수 시그니처:**
```pascal
function GetQueryCondition(const Condition: TStringList): string;
```

**동작:**
- TStringList의 각 항목을 줄바꿈(`#13#10`)으로 연결

---

## CMCommon - 공통 유틸리티

### 개요
**CMCommon**은 애플리케이션 전반에서 사용되는 공통 유틸리티 함수들을 제공합니다.

### 주요 함수

#### `MemoryClear`
PC 메모리 정리 함수입니다.

**함수 시그니처:**
```pascal
procedure MemoryClear;
```

**동작:**
- Windows API `SetProcessWorkingSetSize`를 사용하여 Working Set 정리
- 메모리 사용량 최적화

#### `IsActiveForm`
폼이 현재 활성화되어 있는지 확인하는 함수입니다.

**함수 시그니처:**
```pascal
function IsActiveForm(Sender: TForm): Boolean;
```

**동작:**
1. 부모가 `TSMCMainTabSheet`가 아닌 경우 → 항상 `True`
2. PageControl 내에서 현재 활성 페이지인지 확인
3. 활성 페이지의 DockForm과 일치하면 `True`

#### `ShowChrome`
Chromium 브라우저로 URL을 표시하는 함수입니다.

**함수 시그니처:**
```pascal
procedure ShowChrome(Url: string; bShowModal: Boolean = true);
```

**동작:**
1. `TCMC001FW` 폼 생성
2. URL 설정
3. 모달 또는 모달리스로 표시

#### `fn_hrm_create_connect_file`
HRM 시스템 연결 파일을 생성하는 함수입니다.

**함수 시그니처:**
```pascal
procedure fn_hrm_create_connect_file(isFlag, isAppdiv, isEmpno: String; 
  isGetPost: String = 'post'; bShowModal: Boolean = true);
```

**동작:**
1. HRM 연결 파일 생성 (`fn_hrm_create_connect_file_kumc`)
2. Chromium 브라우저로 표시 (`ShowChrome`)

#### `ShowMedicalCheckup`
건강검진 프로그램을 실행하는 함수입니다.

**함수 시그니처:**
```pascal
procedure ShowMedicalCheckup(LginId, Param: String);
```

**동작:**
1. 로그인 ID를 SHA256 해시로 암호화
2. `C:\Nurion\PDn.exe` 실행

#### `SetDelimitedValue`
구분자로 분리된 문자열을 TStrings로 변환하는 함수입니다.

**함수 시그니처:**
```pascal
procedure SetDelimitedValue(ADelimiter: char; Value: String; AList: Tstrings);
```

**특징:**
- 연속된 구분자 처리 (빈 문자열 추가)
- 예외 처리 포함

---

## CMC001U1 - 공통코드 컴포넌트

### 개요
**CMC001U1**은 공통코드를 조회하고 표시하는 컴포넌트들을 제공합니다.

### 주요 클래스

#### `TSMCCCustomLookupComboBox`
공통코드를 표시하는 LookupComboBox의 베이스 클래스입니다.

**주요 속성:**
```pascal
property CDCOMNDTLS: TSMCMemTable;  // 공통코드 상세 테이블
property CDCOMNItem: TSMCMemTable;  // 공통코드 항목 테이블
property DSDTLS: TDataSource;       // 데이터 소스
property GetFocusData: TStrings;    // 선택된 데이터
```

**주요 메서드:**
```pascal
// 공통코드 목록 가져오기
function GetCodeList(GRP_CD: String; DTLS_CD: String = ''; 
  LANGUAGE_CD: String = ''; AddNONERow: Boolean = True; 
  defaultTxt: String = ''): Boolean;

// 옵션 코드 목록 가져오기
function GetOptCodeList(GRP_CD: String; ITEM_CD: String; 
  LANGUAGE_CD: String = ''; AddNONERow: Boolean = True; 
  defaultTxt: String = ''): Boolean;

// 부서 목록 가져오기
function GetDeptList(MDCR_TISS_DVSN_CD: String; 
  DEPT_CORD_ND_NAME: String = ''; ABRV_CODE: Boolean = False; 
  AddNONERow: Boolean = True; defaultTxt: String = ''): Boolean;

// 열 너비 자동 조정
procedure ActWidthFix;
```

**사용 예시:**
```pascal
var
  ComboBox: TSMCCLookupComboBox;
begin
  ComboBox := TSMCCLookupComboBox.Create(Self);
  ComboBox.GetCodeList('GENDER', '', '', True, '전체');
  ComboBox.ActWidthFix;
end;
```

#### `TSMCCLookupComboBox`
`TSMCCCustomLookupComboBox`를 상속받은 실제 사용 클래스입니다.

**추가 속성:**
```pascal
property HsptCd: string;  // 병원 코드 (다중 병원 지원)
```

#### `TSMCCRadioGroup`
공통코드를 라디오 버튼으로 표시하는 컴포넌트입니다.

**주요 메서드:**
```pascal
function GetCodeList(GRP_CD: String; DTLS_CD: String = ''; 
  LANGUAGE_CD: String = ''; defaultTxt: String = ''): Boolean;
procedure SetRadioGroupItems(KeyField, KeyValue, Caption: String; 
  DataTable: TSHISDataSet);
```

#### `TSMCCCheckGroup`
공통코드를 체크박스로 표시하는 컴포넌트입니다.

**주요 메서드:**
```pascal
function GetCodeList(GRP_CD: String; DTLS_CD: String = ''; 
  LANGUAGE_CD: String = ''; defaultTxt: String = ''): Boolean;
function GetSelectedValue: String;  // 선택된 값 반환
```

### 전역 함수

#### `fOpenServerServiceCOMCode`
서버에서 공통코드를 조회하는 함수입니다.

**함수 시그니처:**
```pascal
function fOpenServerServiceCOMCode(GRP_CD, DTLS_CD, LANGUAGE_CD: String): Boolean;
```

#### `fOpenServerServiceDept`
서버에서 부서 정보를 조회하는 함수입니다.

**함수 시그니처:**
```pascal
function fOpenServerServiceDept(MDCR_DVSN_CD, DEPT_CORD_ND_NAME: String): Boolean;
```

#### `CheckMultiComCodeDataSet`
다중 병원 공통코드 데이터셋을 확인하고 생성하는 함수입니다.

**함수 시그니처:**
```pascal
function CheckMultiComCodeDataSet(HsptCd: string): Boolean;
```

**동작:**
1. `SMCMultiComCodeDataSet`이 없으면 생성
2. 해당 병원 코드의 데이터가 없으면 서버에서 조회하여 추가
3. 필터 적용

---

## GridToExcelEngine - 그리드 엑셀 엔진

### 개요
**GridToExcelEngine**은 그리드 데이터를 엑셀로 내보내는 엔진 클래스를 제공합니다.

### 주요 클래스

#### `TGridToExcelTest`
그리드를 엑셀로 내보내는 메인 클래스입니다.

**주요 속성:**
```pascal
property ColumnAutoFit: Boolean;     // 열 자동 맞춤
property RowAutoFit: Boolean;        // 행 자동 맞춤
property PageSetup: TRecPageSetup;   // 페이지 설정
```

**주요 메서드:**
```pascal
constructor Create(AOwner: TComponent); override;
destructor Destroy; override;

procedure ExportGrid(ExcelDocument: TExcelDocument);
```

**동작 방식:**
1. 로깅 ID 생성
2. 페이지 설정 구성
3. Excel COM 객체 생성 (`DoCreateExcelObject`)
4. 문서 추가 및 내보내기
5. 표시 옵션 적용 (`DoDisplayOption`)

#### `TExcelDocument`
엑셀 문서를 나타내는 클래스입니다.

**주요 속성:**
```pascal
property Title: String;              // 제목
property PrintDate: string;          // 인쇄 날짜
property Condition: TStringList;     // 조건
property PgmId: string;              // 프로그램 ID
property GridTableView: TcxGridTableView;  // 그리드 뷰
```

**주요 메서드:**
```pascal
constructor Create(AOwner: TComponent);
destructor Destroy; override;

procedure DoExport;  // 내보내기 실행
```

**내부 구조:**
- `TExcelTitle`: 제목 영역
- `TExcelPrintDate`: 인쇄 날짜 영역
- `TExcelCondition`: 조건 영역
- `TExcelHeaderColumn`: 헤더 컬럼 영역
- `TExcelData`: 데이터 영역

---

## CMSaveGrid - 그리드 레이아웃 관리

### 개요
**CMSaveGrid**는 그리드의 컬럼 레이아웃을 저장하고 로드하는 기능을 제공합니다.

### 주요 함수

#### `SaveGridViewLayout`
그리드 뷰 레이아웃을 저장하는 함수입니다.

**함수 시그니처:**
```pascal
procedure SaveGridViewLayout(AView: TcxCustomGridView); overload;
procedure SaveGridViewLayout(AView: TcxCustomGridView; Temp: Boolean); overload;
```

**저장 위치:**
- 일반: `GlobalVar.Path.User + 'GridLayouts\' + FormName + '\' + ViewName`
- 임시: 위 경로 + `'_Temp'`

**동작:**
1. 보이는 컬럼이 최소 1개 이상인지 확인
2. 디렉토리 생성
3. `StoreToIniFile`로 레이아웃 저장

#### `LoadGridViewLayout`
그리드 뷰 레이아웃을 로드하는 함수입니다.

**함수 시그니처:**
```pascal
procedure LoadGridViewLayout(AView: TcxCustomGridView); overload;
procedure LoadGridViewLayout(AView: TcxCustomGridView; Temp: Boolean); overload;
```

**동작:**
1. 레이아웃 파일 존재 확인
2. `RestoreFromIniFile`로 레이아웃 복원

#### `InitGridViewLayout`
그리드 뷰 레이아웃을 초기화하는 함수입니다.

**함수 시그니처:**
```pascal
procedure InitGridViewLayout(AView: TcxCustomGridView); overload;
procedure InitGridViewLayout(AView: TcxCustomGridView; Temp: Boolean); overload;
```

**동작:**
- 레이아웃 파일 삭제

#### `upcSCFSetGridColumns`
그리드 컬럼 레이아웃을 저장하거나 로드하는 함수입니다.

**함수 시그니처:**
```pascal
procedure upcSCFSetGridColumns(smcTableView: TSMCGridDBTableView; blSave: Boolean);
```

**매개변수:**
- `blSave = False`: 레이아웃 로드
- `blSave = True`: 레이아웃 저장

#### `upcSCFSetAllGridColumns`
폼 내의 모든 그리드 컬럼 레이아웃을 저장하거나 로드하는 함수입니다.

**함수 시그니처:**
```pascal
procedure upcSCFSetAllGridColumns(frm: TSMCForm; blSave: Boolean);
```

#### `LoadGridColumns` / `SaveGridColumns`
버전별 그리드 컬럼 레이아웃을 로드/저장하는 함수입니다.

**함수 시그니처:**
```pascal
procedure LoadGridColumns(Sender: TObject; sVer: String);
procedure SaveGridColumns(Sender: TObject; sVer: String);
```

**저장 위치:**
- `GlobalVar.Path.Ini + GlobalVar.lginId + '\GridLayouts\' + FormName + '\' + ViewName + '.' + sVer + '.vsn'`

**동작:**
1. 버전 파일 존재 확인
2. 존재하면 레이아웃 로드
3. 존재하지 않으면 기본 레이아웃 저장 후 로드

---

## CMVersionCheck - 버전 체크

### 개요
**CMVersionCheck**는 BPL 패키지의 버전을 확인하는 기능을 제공합니다.

### 주요 함수

#### `VersionCheck`
BPL 파일의 버전을 확인하는 함수입니다.

**함수 시그니처:**
```pascal
function VersionCheck(AForm: TSMCForm): Boolean; overload;
function VersionCheck(FileName: String): Boolean; overload;
```

**동작 방식:**
1. 폼 이름에서 BPL 파일명 생성 (`FormName + 'L1.bpl'`)
2. 서버에서 최신 버전 정보 조회 (`dp_PrgmfmtS04`)
3. 로컬 파일의 타임스탬프 확인
4. 서버 버전과 비교
5. 일치하면 `True`, 불일치하면 `False` 반환

**사용 예시:**
```pascal
if not VersionCheck(Self) then
begin
  ShowMessage('최신 버전이 아닙니다.');
end;
```

---

## CallHrmEnc - HRM 암호화

### 개요
**CallHrmEnc**는 HRM 시스템과의 연동을 위한 암호화 기능을 제공합니다.

### 주요 함수

#### `fn_hrm_MakeKey`
암호화에 사용할 키를 생성하는 함수입니다.

**함수 시그니처:**
```pascal
function fn_hrm_MakeKey(): string;
```

**동작:**
- 16자리 랜덤 문자열 생성 (A-Z, a-z, 0-9)

#### `fn_hrm_Encrypt`
사용자 ID를 암호화하는 함수입니다.

**함수 시그니처:**
```pascal
function fn_hrm_Encrypt(ssrc, skey: string): string;
```

**암호화 방식:**
1. 각 문자를 키와 XOR 연산
2. 결과값을 `_hrm_mask` 배열을 사용하여 3자리 문자로 변환
3. 원본 길이 × 3 길이의 암호화 문자열 생성

#### `fn_hrm_Decrypt`
암호화된 사용자 ID를 복호화하는 함수입니다.

**함수 시그니처:**
```pascal
function fn_hrm_Decrypt(seData, skey: string): string;
```

**복호화 방식:**
1. 3자리씩 묶어서 `_hrm_mask` 배열에서 원래 값 찾기
2. 키와 XOR 연산하여 원본 문자 복원

#### `fn_hrm_get_enc_data`
사번을 암호화한 결과와 키를 반환하는 함수입니다.

**함수 시그니처:**
```pascal
function fn_hrm_get_enc_data(inEmpno: string; var outEncData: string; 
  var outKey: string): boolean;
```

#### `fn_hrm_create_connect_file_kumc`
HRM 연결 HTML 파일을 생성하는 함수입니다.

**함수 시그니처:**
```pascal
function fn_hrm_create_connect_file_kumc(isFlag, isAppdiv, isEmpno: string; 
  isGetPost: string = 'post'; IniDir: string = ''): string;
```

**동작:**
1. 사번 암호화
2. INI 파일에서 페이지 정보 읽기 (`CHROMSENDLIST.INI`)
3. POST 또는 GET 방식에 따라 HTML 파일 생성
4. 임시 파일 경로 반환

**생성되는 HTML 구조:**
```html
<form name="sso" method="post">
  <input type="hidden" name="key" value="...">
  <input type="hidden" name="encdata" value="...">
  <input type="hidden" name="appdiv" value="...">
</form>
<script>
  // 페이지 이동 스크립트
  document.sso.submit();
</script>
```

---

## chrome_cefvcl - Chromium 브라우저

### 개요
**chrome_cefvcl**은 Chromium Embedded Framework를 Delphi에서 사용할 수 있도록 래핑한 컴포넌트입니다.

### 주요 클래스

#### `TCustomChromium`
Chromium 브라우저 컴포넌트의 베이스 클래스입니다.

**주요 속성:**
```pascal
property DefaultUrl: ustring;                    // 기본 URL
property Options: TChromiumOptions;               // 브라우저 옵션
property DefaultEncoding: ustring;                // 기본 인코딩
property FontOptions: TChromiumFontOptions;       // 폰트 옵션
```

**주요 이벤트:**
```pascal
property OnLoadStart: TOnLoadStart;              // 페이지 로드 시작
property OnLoadEnd: TOnLoadEnd;                   // 페이지 로드 완료
property OnLoadError: TOnLoadError;               // 페이지 로드 오류
property OnBeforeBrowse: TOnBeforeBrowse;         // 페이지 이동 전
property OnAddressChange: TOnAddressChange;       // 주소 변경
property OnTitleChange: TOnTitleChange;           // 제목 변경
property OnBeforePopup: TOnBeforePopup;           // 팝업 열기 전
property OnAfterCreated: TOnAfterCreated;         // 브라우저 생성 후
property OnBeforeClose: TOnBeforeClose;           // 브라우저 닫기 전
```

**주요 메서드:**
```pascal
procedure Load(const url: ustring);               // URL 로드
procedure Reload;                                 // 새로고침
procedure Stop;                                   // 중지
procedure GoBack;                                 // 뒤로 가기
procedure GoForward;                              // 앞으로 가기
procedure Print;                                  // 인쇄
```

**사용 예시:**
```pascal
var
  Chromium: TCustomChromium;
begin
  Chromium := TCustomChromium.Create(Self);
  Chromium.Parent := Self;
  Chromium.Align := alClient;
  Chromium.DefaultUrl := 'https://example.com';
  Chromium.Load(Chromium.DefaultUrl);
end;
```

---

## SMCLogData - 로그 데이터 관리

### 개요
**SMCLogData**는 사용자 작업 로그를 관리하는 클래스를 제공합니다.

### 주요 클래스

#### `TLogData`
로그 데이터를 관리하는 싱글톤 클래스입니다.

**주요 메서드:**
```pascal
// 로그 시작
function BeginLog(FForm: TCustomForm; EventName: String; 
  ServiceID: String = ''; InptData: String = ''): TLogUID;

// 로그 종료
procedure EndLog(FForm: TCustomForm; EventName: String); overload;
procedure EndLog(LogUUID: TLogUID); overload;

// 로그 데이터 추가
procedure AddLogData(FForm: TCustomForm; DataLog: String);

// 이벤트 이름 설정
procedure SetEventName(LogUUID: TLogUID; EventName: String); overload;
procedure SetEventName(FForm: TCustomForm; EventName: String); overload;

// 서비스 ID 설정
procedure SetServiceID(LogUUID: TLogUID; ServiceID: String); overload;
procedure SetServiceID(FForm: TCustomForm; ServiceID: String); overload;

// 로그 초기화
procedure Clear;
```

**로그 데이터 구조:**
```pascal
TLogValue = class
  work_uuid: String;          // 작업 UUID
  user_id: String;            // 사용자 ID
  user_ip: String;            // 사용자 IP
  evnt_vl: String;            // 이벤트 값
  scrn_id: String;            // 화면 ID
  srvc_id: String;            // 서비스 ID
  inpt_data_vl: TStrings;     // 입력 데이터 값
  srvc_strt_dt: TDateTime;    // 서비스 시작 일시
  srvc_fnsh_dt: TDateTime;    // 서비스 종료 일시
end;
```

**사용 예시:**
```pascal
var
  LogUUID: TLogUID;
begin
  // 로그 시작
  LogUUID := LogData.BeginLog(Self, '조회', 'cco_useramv_l01');
  
  try
    // 작업 수행
    // ...
  finally
    // 로그 종료
    LogData.EndLog(LogUUID);
  end;
end;
```

#### `UserLog`
사용자 로그를 기록하는 전역 함수입니다.

**함수 시그니처:**
```pascal
procedure UserLog(FForm: TControl; LogData: string; TypeLevel: String = 'CL');
```

**매개변수:**
- `FForm`: 폼 또는 컨트롤
- `LogData`: 로그 데이터
- `TypeLevel`: 로그 타입 레벨 (기본: 'CL')

**동작:**
1. UUID 생성
2. 폼 정보 추출
3. 서버에 로그 전송 (`dp_CcoInsertUserPrcmLogDataSID`)

---

## CMCCodeData - 공통 상수 데이터

### 개요
**CMCCodeData**는 공통 상수 데이터를 관리하는 클래스를 제공합니다.

### 주요 클래스

#### `TComConstItem`
공통 상수 항목을 나타내는 클래스입니다.

**주요 속성:**
```pascal
property CnstId: string;      // 상수 ID
property CnstVal: string;     // 상수 값
```

#### `TComConstList`
공통 상수 목록을 관리하는 컬렉션 클래스입니다.

**주요 메서드:**
```pascal
function Add: TComConstItem;
procedure AddData(const ACnstId: string; const ACnstVal: string);
procedure DeleteData(const ACnstId: string);
function IndexOf(const ACnstId: string): integer;
```

### 전역 함수

#### `IsExistComConst`
공통 상수가 존재하는지 확인하는 함수입니다.

**함수 시그니처:**
```pascal
function IsExistComConst(CnstGrpId, CnstId: String; 
  CnstVal: String = ''): Boolean; overload;
function IsExistComConst(CnstGrpId, CnstId: String; 
  CnstVal: String; HsptCd: String): Boolean; overload;
```

#### `GetComConstValue`
공통 상수 값을 가져오는 함수입니다.

**함수 시그니처:**
```pascal
function GetComConstValue(CnstGrpId, CnstId: String): String; overload;
function GetComConstValue(CnstGrpId, CnstId, HsptCd: String): String; overload;
```

#### `GetComConstList`
공통 상수 목록을 가져오는 함수입니다.

**함수 시그니처:**
```pascal
function GetComConstList(CnstGrpId: String): TComConstList; overload;
function GetComConstList(CnstGrpId, HsptCd: String): TComConstList; overload;
```

#### `CheckMultiComConstDataSet`
다중 병원 공통 상수 데이터셋을 확인하고 생성하는 함수입니다.

**함수 시그니처:**
```pascal
function CheckMultiComConstDataSet(HsptCd: string): Boolean;
```

**동작:**
- `CheckMultiComCodeDataSet`과 유사한 방식으로 동작

---

## 핵심 기능 상세 분석

### 1. 엑셀 출력 프로세스

**전체 흐름:**
```
사용자 요청
    ↓
비밀번호 확인 (GetPasswordCheck)
    ↓
개인정보 사유 입력 (ShowPersonalInfo) - 필요시
    ↓
로깅 ID 생성 (GetLoggingID)
    ↓
엑셀 폴더 생성 (IsExistExcelFolder)
    ↓
파일명 정리 (ExcelFileName)
    ↓
그리드 타입 확인
    ↓
적절한 내보내기 함수 호출
    ↓
Excel COM 객체 생성
    ↓
데이터 내보내기
    ↓
서식 적용
    ↓
파일 저장
    ↓
파일 열기 (옵션)
```

### 2. 로깅 프로세스

**출력물 로깅:**
```
출력 요청
    ↓
개인정보 여부 확인
    ↓
개인정보 사유 입력 (필요시)
    ↓
로깅 ID 생성 (타임스탬프 + 사용자ID)
    ↓
서버에 로깅 정보 전송
    ↓
로깅 ID 반환
```

**작업 로깅:**
```
작업 시작
    ↓
BeginLog 호출
    ↓
UUID 생성
    ↓
서버에 로그 삽입
    ↓
작업 수행
    ↓
EndLog 호출
    ↓
서버에 로그 업데이트
```

### 3. 공통코드 조회 프로세스

**단일 병원:**
```
GetCodeList 호출
    ↓
서버에서 공통코드 조회
    ↓
메모리 테이블에 저장
    ↓
컴포넌트에 바인딩
```

**다중 병원:**
```
GetCodeList 호출 (HsptCd 지정)
    ↓
다중 병원 데이터셋 확인
    ↓
해당 병원 데이터 없으면 서버에서 조회
    ↓
필터 적용
    ↓
컴포넌트에 바인딩
```

---

## 주요 클래스 및 함수

### CPCOM

#### 전역 함수
```pascal
// 엑셀 출력
procedure GridToExcel(...);
procedure AdvGridToExcel(...);
procedure TreeListToExcel(...);

// 로깅
function GetLoggingID(...): String;
function GetPrintLoggingID(...): String;
function ShowPersonalInfo(...): integer;
function GetPasswordCheck(...): boolean;

// 유틸리티
function GetQueryCondition(...): string;
function ExcelFileName(...): String;
procedure BeginExcelForGrid(...);
procedure EndExcelForGrid;
```

### CMCommon

#### 전역 함수
```pascal
procedure MemoryClear;
function IsActiveForm(Sender: TForm): Boolean;
procedure ShowChrome(Url: string; bShowModal: Boolean = true);
procedure fn_hrm_create_connect_file(...);
procedure ShowMedicalCheckup(LginId, Param: String);
procedure SetDelimitedValue(ADelimiter: char; Value: String; AList: Tstrings);
```

### CMC001U1

#### 주요 클래스
```pascal
TSMCCCustomLookupComboBox
TSMCCLookupComboBox
TSMCCRadioGroup
TSMCCCheckGroup
TSMCCDBLookupComboBox
```

#### 전역 함수
```pascal
function fOpenServerServiceCOMCode(...): Boolean;
function fOpenServerServiceDept(...): Boolean;
function CheckMultiComCodeDataSet(HsptCd: string): Boolean;
```

### GridToExcelEngine

#### 주요 클래스
```pascal
TGridToExcelTest
TExcelDocument
TExcelItem
TExcelTitle
TExcelPrintDate
TExcelCondition
TExcelHeaderColumn
TExcelData
```

### CMSaveGrid

#### 전역 함수
```pascal
procedure SaveGridViewLayout(AView: TcxCustomGridView);
procedure LoadGridViewLayout(AView: TcxCustomGridView);
procedure InitGridViewLayout(AView: TcxCustomGridView);
procedure upcSCFSetGridColumns(...);
procedure upcSCFSetAllGridColumns(...);
procedure LoadGridColumns(Sender: TObject; sVer: String);
procedure SaveGridColumns(Sender: TObject; sVer: String);
```

### CMVersionCheck

#### 전역 함수
```pascal
function VersionCheck(AForm: TSMCForm): Boolean; overload;
function VersionCheck(FileName: String): Boolean; overload;
```

### CallHrmEnc

#### 전역 함수
```pascal
function fn_hrm_MakeKey(): string;
function fn_hrm_Encrypt(ssrc, skey: string): string;
function fn_hrm_Decrypt(seData, skey: string): string;
function fn_hrm_get_enc_data(...): boolean;
function fn_hrm_create_connect_file_kumc(...): string;
```

### SMCLogData

#### 전역 변수
```pascal
var LogData: TLogData;
```

#### 전역 함수
```pascal
procedure UserLog(FForm: TControl; LogData: string; TypeLevel: String = 'CL');
function GetIdentification: string;
```

### CMCCodeData

#### 전역 함수
```pascal
function IsExistComConst(...): Boolean;
function GetComConstValue(...): String;
function GetComConstList(...): TComConstList;
function CheckMultiComConstDataSet(HsptCd: string): Boolean;
```

---

## 프로젝트 간 의존성

```
COM_LV0 (기본 인프라)
    ↓
COM_LV1 (써드파티 라이브러리)
    ↓
COM_LV2 (프레임워크 레이어)
    ↓
CCOM (공통 컴포넌트) ← 현재 분석
    ↓
MAIN (애플리케이션)
```

**CCOM의 역할:**
- COM_LV2 프레임워크 위에 구축되는 공통 컴포넌트 패키지
- 애플리케이션 전반에서 사용되는 공통 기능 제공
- 엑셀 출력, 로깅, 공통코드 관리 등 핵심 기능 모듈

---

## 요약

CCOM 프로젝트는 Nexmed EHR 시스템의 공통 컴포넌트 패키지로, 다음과 같은 핵심 기능을 제공합니다:

1. **CPCOM**: 엑셀 출력 및 출력물 로깅
2. **CMCommon**: 공통 유틸리티 (메모리 관리, 브라우저 표시 등)
3. **CMC001U1**: 공통코드 조회 및 표시 컴포넌트
4. **GridToExcelEngine**: 그리드 엑셀 내보내기 엔진
5. **CMSaveGrid**: 그리드 레이아웃 저장/로드
6. **CMVersionCheck**: BPL 패키지 버전 확인
7. **CallHrmEnc**: HRM 시스템 연동 및 암호화
8. **chrome_cefvcl**: Chromium 브라우저 컴포넌트
9. **SMCLogData**: 사용자 작업 로그 관리
10. **CMCCodeData**: 공통 상수 데이터 관리

이러한 모듈들은 MAIN 애플리케이션 및 다른 프로젝트에서 광범위하게 사용되어 EHR 시스템의 공통 기능을 구현합니다.

