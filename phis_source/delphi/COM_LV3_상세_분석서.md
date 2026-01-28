# COM_LV3 프로젝트 상세 분석서

## 목차
1. [프로젝트 개요](#프로젝트-개요)
2. [프로젝트 구조](#프로젝트-구조)
3. [NRCViewerPackage - 간호기록 인증 뷰어](#nrcviewerpackage---간호기록-인증-뷰어)
4. [ScreenSecurityPack - 화면 보안](#screensecuritypack---화면-보안)
5. [SMCExport - 엑셀/CSV 내보내기](#smcexport---엑셀csv-내보내기)
6. [SMCFax - 팩스 기능](#smcfax---팩스-기능)
7. [SMCLProxy - 리포트 뷰어](#smclproxy---리포트-뷰어)
8. [SMCFile - 파일 관리](#smcfile---파일-관리)
9. [핵심 기능 상세 분석](#핵심-기능-상세-분석)
10. [주요 클래스 및 함수](#주요-클래스-및-함수)

---

## 프로젝트 개요

**COM_LV3**는 Nexmed EHR 시스템의 비즈니스 로직 레이어로, COM_LV2 프레임워크 위에 구축되는 특화된 기능 모듈들을 포함합니다. 이 프로젝트는 다음과 같은 주요 역할을 담당합니다:

- **간호기록 인증 뷰어**: EMR 기록을 XML 기반으로 표시하고 비교
- **화면 보안**: 스크린샷 방지 및 화면 보안 관리
- **데이터 내보내기**: 그리드 데이터를 엑셀/CSV로 내보내기
- **팩스 기능**: 팩스 전송 기능
- **리포트 뷰어**: 리포트 미리보기 및 인쇄
- **파일 관리**: 원격 파일 서버와의 파일 업로드/다운로드

---

## 프로젝트 구조

```
COM_LV3/
├── NRCViewerPackage/          # 간호기록 인증 뷰어 패키지
│   ├── NRCViewer.pas          # 메인 뷰어 컴포넌트
│   ├── NRCViewerClasses.pas   # 뷰어 핵심 클래스
│   ├── NRCViewerFrame.pas     # 뷰어 프레임
│   ├── NRCViewerItemClasses.pas # 아이템 클래스
│   ├── NRCViewerTextComparator.pas # 텍스트 비교기
│   └── ...
├── ScreenSecurityPack/         # 화면 보안 패키지
│   ├── ScreenSecurityMain.pas  # 보안 메인 로직
│   ├── ScreenSecurityDialog.pas # 보안 다이얼로그
│   └── ScreenSecurityRegister.pas # 보안 등록
├── SMCExport/                 # 데이터 내보내기 패키지
│   ├── Sources/
│   │   ├── SMCExcel.pas       # 엑셀 내보내기
│   │   └── SMCCSV.pas         # CSV 내보내기
│   └── Packages/
├── SMCFax/                    # 팩스 패키지
│   ├── FAX001M1.pas           # 팩스 데이터 모듈
│   └── FAX001U1.pas           # 팩스 유틸리티
├── SMCLProxy/                 # 리포트 뷰어 프록시
│   ├── LPA001M1.pas           # 리포트 데이터 모듈
│   ├── LPA001U1.pas           # 리포트 뷰어 폼
│   └── ...
└── SMCFile/                   # 파일 관리 패키지
    └── SMCIFFile/
        └── SMCIFFileClasses.pas # 파일 업로드/다운로드
```

---

## NRCViewerPackage - 간호기록 인증 뷰어

### 개요
**NRCViewer (Nursing Record Certification Viewer)**는 간호기록을 XML 형식으로 표시하고, 두 기록을 비교하여 변경 사항을 시각적으로 보여주는 뷰어 컴포넌트입니다.

### 주요 파일
- **`NRCViewer.pas`**: 메인 뷰어 컴포넌트 (`TNRCViewer`)
- **`NRCViewerClasses.pas`**: 뷰어 핵심 클래스 및 인터페이스
- **`NRCViewerFrame.pas`**: 뷰어 프레임
- **`NRCViewerItemClasses.pas`**: 다양한 UI 컴포넌트를 위한 아이템 클래스
- **`NRCViewerTextComparator.pas`**: 텍스트 비교 로직

### 주요 클래스

#### `TNRCCustomViewer`
간호기록 뷰어의 베이스 클래스입니다.

**주요 속성:**
```pascal
property Body: TNRCViewerBody;              // 뷰어 본문 영역
property Control: TControl;                 // 생성된 컨트롤
property Datas: TNRCViewerDatas;            // 데이터 리스트 (ListView)
property Splitter: TSMCSplitter;           // 스플리터
```

**주요 메서드:**
```pascal
// 초기화 및 정리
procedure Clear;                            // 뷰어 내용 지우기

// XML 로드
procedure LoadFromXMLSource(const XMLSource: string);  // XML 문자열에서 로드
procedure LoadFromXMLFile(const FileName: string);     // XML 파일에서 로드

// 비교 기능
procedure Compare(const XMLSource1, XMLSource2: string); // 두 XML 비교
```

**동작 방식:**
1. XML 소스를 파싱하여 컨트롤 트리 생성
2. 각 컨트롤 타입에 맞는 Generator를 사용하여 실제 UI 컴포넌트 생성
3. 비교 모드에서는 두 XML을 비교하여 변경 사항을 시각적으로 표시

#### `TNRCViewerGenerator`
뷰어 아이템 생성 및 관리를 담당하는 팩토리 클래스입니다.

**주요 메서드:**
```pascal
// Generator 등록
class procedure RegisterItemGenerator(Generator: TNRCViewerItemGenerator);
class procedure UNRegisterItemGenerator(GeneratorClsss: TNRCViewerItemGeneratorClass);
class procedure RegisterItemPropertyGenerator(Generator: TNRCViewerItemPropertyGenerator);
class procedure RegisterItemElementGenerator(Generator: TNRCViewerItemElementGenerator);

// Generator 조회
class function GetItemGenerator(Control: TControl): TNRCViewerItemGenerator; overload;
class function GetItemGenerator(const ControlType: string): TNRCViewerItemGenerator; overload;
class function GetItemElementGenerator(Component: TComponent): TNRCViewerItemElementGenerator; overload;
class function GetItemElementGenerator(const ComponentType: string): TNRCViewerItemElementGenerator; overload;

// XML 변환
class function Execute1(Control: TControl; const Name: string; Datas: TStrings): string;  // 컨트롤 → XML
class function Execute2(const XMLSource: string; Viewer: TNRCCustomViewer): TControl;    // XML → 컨트롤
```

**Generator 패턴:**
- 각 UI 컴포넌트 타입(Label, Edit, Grid 등)마다 Generator가 등록됨
- Generator는 컨트롤을 XML로 변환하거나 XML에서 컨트롤을 생성하는 역할

#### `TNRCViewerItemGenerator`
특정 컨트롤 타입을 처리하는 Generator의 베이스 클래스입니다.

**주요 메서드:**
```pascal
// 컨트롤 정보
function GetControlClass: TControlClass;    // 컨트롤 클래스 반환
function GetControlType: string;           // 컨트롤 타입 문자열 반환
function GetControlKind: Integer;          // 컨트롤 종류 (데이터 유무, 자식 가능 여부 등)

// 검증
function Check(Control: TControl): Boolean; // 컨트롤이 처리 가능한지 확인

// 변환
function Execute1(Control: TControl; const Space: string): string;  // 컨트롤 → XML
function Execute2: INRCViewerItem;                                 // XML → 컨트롤 인터페이스

// 잠금/해제
procedure Lock(Control: TWinControl);       // 컨트롤 잠금
procedure Unlock(Control: TWinControl);    // 컨트롤 잠금 해제

// 자식 컨트롤
function GetControlCount(Control: TWinControl): Integer;
function GetControl(Control: TWinControl; Index: Integer): TControl;

// 텍스트 처리
function GetText(const Node: IXMLDOMNode): string;
procedure SetText(const Node: IXMLDOMNode; const Value: string; Attributes: TStrings);
```

#### `TNRCViewerComparator`
두 XML을 비교하여 변경 사항을 찾는 비교기입니다.

**주요 메서드:**
```pascal
// Comparator 등록
class procedure RegisterComparator(ItemGeneratorClass: TNRCViewerItemGeneratorClass; ComparatorClass: TComparatorClass);
class function ComparatorClassOf(ItemGeneratorClass: TNRCViewerItemGeneratorClass): TComparatorClass;

// 비교 수행
function Generate(const XMLSource1, XMLSource2: string): string;  // 비교 결과 XML 생성
```

**비교 방식:**
1. 두 XML을 파싱하여 노드 트리 생성
2. 각 노드를 이름으로 매칭
3. 매칭된 노드의 내용을 비교
4. 변경 사항을 표시하는 비교 결과 XML 생성

### 사용 예시

```pascal
var
  Viewer: TNRCViewer;
  XML1, XML2: string;
begin
  Viewer := TNRCViewer.Create(nil);
  try
    Viewer.Parent := Self;
    Viewer.Align := alClient;
    
    // 단일 XML 로드
    XML1 := '<NRCViewer><Label Name="Name" Text="홍길동"/></NRCViewer>';
    Viewer.LoadFromXMLSource(XML1);
    
    // 두 XML 비교
    XML2 := '<NRCViewer><Label Name="Name" Text="김철수"/></NRCViewer>';
    Viewer.Compare(XML1, XML2);
    
    // 파일에서 로드
    Viewer.LoadFromXMLFile('record.xml');
  finally
    Viewer.Free;
  end;
end;
```

### 지원하는 컨트롤 타입
- **Standard**: Label, Edit, Memo, Button 등 표준 VCL 컨트롤
- **TMS**: TMS 컴포넌트 (AdvGrid 등)
- **SMCExpress**: DevExpress 컴포넌트 (cxGrid 등)
- **EMR**: EMR 전용 컴포넌트

---

## ScreenSecurityPack - 화면 보안

### 개요
**ScreenSecurityPack**은 특정 화면에 대한 스크린샷 방지 및 화면 보안 기능을 제공합니다. 일정 시간 동안 사용자 입력이 없으면 경고 메시지를 표시하고, 필요시 화면을 닫습니다.

### 주요 파일
- **`ScreenSecurityMain.pas`**: 보안 메인 로직 및 훅 관리
- **`ScreenSecurityDialog.pas`**: 보안 경고 다이얼로그
- **`ScreenSecurityRegister.pas`**: 보안 설정 등록

### 주요 클래스 및 함수

#### `TSecurityChecker`
화면 보안을 체크하는 타이머 클래스입니다.

**주요 메서드:**
```pascal
constructor Create;                         // 생성자 (1초 간격 타이머)
destructor Destroy; override;               // 소멸자

procedure ActiveInput(Wnd: HWnd);          // 입력 활동 기록
procedure DoCheck(Sender: TObject);        // 타이머 이벤트 핸들러
```

**동작 방식:**
1. 1초마다 `DoCheck` 실행
2. 등록된 보안 창 목록을 확인
3. 각 창의 마지막 입력 시간을 체크
4. 설정된 시간(기본: `ScreenSecurityInterval` 분) 동안 입력이 없으면 경고 다이얼로그 표시
5. 카운트다운 후에도 입력이 없으면 화면 닫기

#### `TSecurityWindowList`
보안이 적용된 창 목록을 관리하는 클래스입니다.

**주요 메서드:**
```pascal
procedure Add(Wnd: HWnd; const AClassName, Caption: String);  // 창 추가
procedure Clear;                                               // 목록 지우기
function Count: Integer;                                       // 개수 반환
procedure Delete(Index: Integer);                             // 항목 삭제
function Find(Wnd: HWnd): Integer;                             // 창 찾기
```

#### `TSecurityWindow`
보안 창 정보를 담는 레코드입니다.

```pascal
TSecurityWindow = record
  Wnd: HWnd;                   // 창 핸들
  AClassName: String;          // 클래스 이름
  Caption: String;            // 캡션
  LastTime: Integer;          // 마지막 입력 시간
  ActiveMsg: Boolean;         // 메시지 활성화 여부
end;
```

#### 전역 함수

```pascal
// 보안 창 등록/해제
procedure RegistSecurityWindow(const ClassName, Caption: String);
procedure UnregistSecurityWindow(const ClassName: String);
```

**훅 함수:**
```pascal
// 키보드/마우스 훅
function KeyboardHookProc(Code: Integer; WParam: WPARAM; LParam: LPARAM): LRESULT; stdcall;
function MouseHookProc(Code: Integer; WParam: WPARAM; LParam: LPARAM): LRESULT; stdcall;

// 훅 시작/중지
procedure StartHooks;
procedure StopHooks;
```

**동작 방식:**
1. 키보드/마우스 훅을 설치하여 모든 입력 이벤트 감지
2. 입력이 발생하면 해당 창의 `LastTime` 업데이트
3. 타이머가 주기적으로 체크하여 비활성 시간이 경과하면 경고 표시

### 사용 예시

```pascal
// 보안 창 등록
RegistSecurityWindow('TMyForm', '환자 정보 입력');

// 보안 창 해제
UnregistSecurityWindow('TMyForm');
```

### 설정 변수
- **`ScreenSecurityInterval`**: 비활성 시간 (분 단위)
- **`ScreenSecurityCountDown`**: 경고 다이얼로그 카운트다운 시간 (초 단위)

---

## SMCExport - 엑셀/CSV 내보내기

### 개요
**SMCExport**는 DevExpress 그리드의 데이터를 엑셀 또는 CSV 파일로 내보내는 기능을 제공합니다.

### 주요 파일
- **`SMCExcel.pas`**: 엑셀 내보내기 구현
- **`SMCCSV.pas`**: CSV 내보내기 구현

### 주요 클래스

#### `TSMCExcelForGrid`
그리드를 엑셀로 내보내는 메인 클래스입니다.

**주요 속성:**
```pascal
property PageSetup: TPageSetup;            // 페이지 설정 (기밀, 사용자 정보 등)
property ProgressBar: TSMCProgressBar;    // 진행 표시줄
property ExcelVisible: boolean;            // 엑셀 표시 여부
```

**주요 메서드:**
```pascal
constructor Create(AOwner: TComponent); override;
destructor Destroy; override;

procedure ExportGrid;                     // 그리드 내보내기 실행
procedure Add(ExcelDocument: TExcelDocument); // 문서 추가
```

#### `TExcelDocument`
엑셀 문서를 나타내는 클래스입니다.

**주요 속성:**
```pascal
property Title: String;                   // 제목
property PrintDate: string;               // 인쇄 날짜
property Condition: TStringList;          // 조건 (검색 조건 등)
property GridTableView: TcxGridTableView; // 그리드 뷰
property ColumnAutoFit: Boolean;          // 열 자동 맞춤
property RowAutoFit: Boolean;             // 행 자동 맞춤
property SheetName: string;               // 시트 이름
property CellText: Boolean;                // 셀 텍스트 모드
property RangeFormatList: TRangeFormatList; // 범위 서식 목록
property RangeAlignmentList: TRangeAlignmentList; // 범위 정렬 목록
```

**주요 메서드:**
```pascal
constructor Create(AOwner: TComponent);
destructor Destroy;

procedure SetFreezePane(AFreezePane: Boolean; AFreezeColumn, AFreezeRow: Integer); // 고정 창 설정
```

#### `TExcelItem`
엑셀 문서의 각 요소를 나타내는 베이스 클래스입니다.

**하위 클래스:**
- **`TExcelTitle`**: 제목 영역
- **`TExcelPrintDate`**: 인쇄 날짜 영역
- **`TExcelCondition`**: 조건 영역
- **`TExcelHeaderColumn`**: 헤더 컬럼 영역
- **`TExcelData`**: 데이터 영역

#### `TSMCTableForCSV`
테이블을 CSV로 내보내는 클래스입니다.

**주요 메서드:**
```pascal
constructor Create;
destructor Destroy; override;

procedure SMCTableForCSV(Table: TkbmCustomMemTable; FilePath, FormName, UserId: string);
```

**특징:**
- UTF-8 BOM 추가 (Excel에서 한글 깨짐 방지)
- kbmMemTable의 CSV 스트림 포맷 사용

### 사용 예시

#### 엑셀 내보내기
```pascal
var
  Excel: TSMCExcelForGrid;
  Document: TExcelDocument;
begin
  Excel := TSMCExcelForGrid.Create(nil);
  try
    Excel.ExcelVisible := False;
    
    // 페이지 설정
    Excel.PageSetup.Confidential := 1;
    Excel.PageSetup.User_ID := 'USER001';
    Excel.PageSetup.Cnct_IP := '192.168.1.100';
    Excel.PageSetup.PgmId := 'PATIENT_LIST';
    
    // 문서 생성
    Document := TExcelDocument.Create(nil);
    try
      Document.GridTableView := cxGrid1DBTableView1;
      Document.Title := '환자 목록';
      Document.PrintDate := FormatDateTime('yyyy-mm-dd hh:nn:ss', Now);
      Document.SheetName := '환자목록';
      Document.ColumnAutoFit := True;
      
      // 조건 추가
      Document.Condition.Add('검색 조건: 전체');
      
      Excel.Add(Document);
    except
      Document.Free;
      raise;
    end;
    
    // 내보내기 실행
    Excel.ExportGrid;
  finally
    Excel.Free;
  end;
end;
```

#### CSV 내보내기
```pascal
var
  CSV: TSMCTableForCSV;
  MemTable: TkbmMemTable;
begin
  CSV := TSMCTableForCSV.Create;
  try
    MemTable := TkbmMemTable.Create(nil);
    try
      // 데이터 로드
      MemTable.LoadFromDataSet(DataSet, [mtcpoStructure, mtcpoProperties]);
      
      // CSV 내보내기
      CSV.SMCTableForCSV(MemTable, 'C:\Export\patient.csv', '환자목록', 'USER001');
    finally
      MemTable.Free;
    end;
  finally
    CSV.Free;
  end;
end;
```

---

## SMCFax - 팩스 기능

### 개요
**SMCFax**는 팩스 전송 기능을 제공하는 패키지입니다. FaxApp COM 객체를 사용하여 팩스를 전송합니다.

### 주요 파일
- **`FAX001M1.pas`**: 팩스 데이터 모듈
- **`FAX001U1.pas`**: 팩스 유틸리티

### 주요 클래스

#### `TFAX001D1`
팩스 기능을 위한 데이터 모듈입니다.

**주요 컴포넌트:**
```pascal
FaxApp: TFaxAppMain;                      // 팩스 애플리케이션 COM 객체
stb_cuo_ccenstg_l03: TSMCSTable;          // 팩스 설정 테이블
stb_cuo_enstimt_s02: TSMCSTable;          // 팩스 전송 테이블
```

**싱글톤 패턴:**
```pascal
function FAX001D1: TFAX001D1;  // 전역 함수로 싱글톤 인스턴스 반환
```

### 사용 예시

```pascal
var
  FaxModule: TFAX001D1;
begin
  FaxModule := FAX001D1;
  
  // 팩스 설정 로드
  FaxModule.stb_cuo_ccenstg_l03.Open;
  
  // 팩스 전송 정보 설정
  FaxModule.stb_cuo_enstimt_s02.Open;
  FaxModule.stb_cuo_enstimt_s02.Append;
  FaxModule.stb_cuo_enstimt_s02.FieldByName('FaxNumber').AsString := '02-1234-5678';
  FaxModule.stb_cuo_enstimt_s02.FieldByName('FileName').AsString := 'report.pdf';
  FaxModule.stb_cuo_enstimt_s02.Post;
  
  // 팩스 전송
  FaxModule.FaxApp.SendFax(...);
end;
```

---

## SMCLProxy - 리포트 뷰어

### 개요
**SMCLProxy (SMC Library Proxy)**는 리포트를 미리보기하고 인쇄하는 뷰어 폼을 제공합니다. Clipsoft Rexpert 리포트 뷰어를 사용합니다.

### 주요 파일
- **`LPA001M1.pas`**: 리포트 데이터 모듈
- **`LPA001U1.pas`**: 리포트 뷰어 폼 (`TLPA001F1`)

### 주요 클래스

#### `TLPA001F1`
리포트 뷰어 폼 클래스입니다. `ISMCReportForm` 인터페이스를 구현합니다.

**주요 속성:**
```pascal
property ViewRemorte: Boolean;           // 원격 리포트 여부
property ViewResult: Boolean;            // 결과 뷰 여부
property PersonalInfo: string;           // 개인정보 설정
property PersonalType: Boolean;          // 개인정보 타입
property ZoomOption: TSMCZoomOption;    // 줌 옵션
```

**주요 메서드:**
```pascal
// 인터페이스 구현
function Get_Instance: TForm;
function Get_Viewer: TRexViewerCtrl30;
function Get_OnPreviewBegin: TNotifyEvent;
procedure Set_OnPreviewBegin(const Value: TNotifyEvent);
function Get_OnPreviewEnd: TNotifyEvent;
procedure Set_OnPreviewEnd(const Value: TNotifyEvent);

// 리포트 처리
procedure Acquire;                       // 리소스 획득
procedure Release;                       // 리소스 해제
procedure Prepare(const Report: ISMCReport);  // 리포트 준비
procedure Preview;                       // 미리보기 표시
```

**주요 이벤트:**
```pascal
procedure ViewerButtonPrintClickBefore(Sender: TObject; const Cancel: OleVariant);
procedure ViewerButtonPrintClickAfter(Sender: TObject);
procedure ViewerFinishPrintResult(Sender: TObject; ResultCode: Integer);
```

**동작 방식:**
1. `Prepare`에서 리포트 설정 및 CSS 적용
2. `Preview`에서 리포트 뷰어 표시
3. 인쇄 버튼 클릭 시 인쇄 다이얼로그 표시 또는 직접 인쇄
4. 원격 리포트인 경우 서버로 전송

### 사용 예시

```pascal
var
  Report: TSMCReport;
  Viewer: TLPA001F1;
begin
  Report := TSMCReport.Create;
  try
    // 리포트 데이터 설정
    Report.AddWork(...);
    
    // 뷰어 생성
    Viewer := TLPA001F1.Create(nil);
    try
      Viewer.Prepare(Report);
      Viewer.Preview;
    finally
      Viewer.Free;
    end;
  finally
    Report.Free;
  end;
end;
```

---

## SMCFile - 파일 관리

### 개요
**SMCFile**은 원격 파일 서버와의 파일 업로드/다운로드를 관리하는 패키지입니다. InfUpDown.dll을 사용합니다.

### 주요 파일
- **`SMCIFFileClasses.pas`**: 파일 관리 클래스

### 주요 클래스

#### `TSMCIFFileConnector`
파일 서버 연결을 관리하는 클래스입니다.

**주요 속성:**
```pascal
property Address: string;               // 서버 주소 (기본: '127.0.0.1')
property Port: Integer;                  // 포트 (기본: 4006)
```

**주요 메서드:**
```pascal
constructor Create(Owner: TComponent); override;
procedure SetDefaultConnector;          // 기본 커넥터로 설정
function Open: Boolean;                 // 서버 연결
```

#### `TSMCFIFFileManager`
파일 업로드/다운로드를 관리하는 클래스입니다.

**주요 속성:**
```pascal
property Connector: TSMCIFFileConnector; // 커넥터
```

**주요 메서드:**
```pascal
constructor Create(Owner: TComponent); override;

function Upload(const LocalPath, RemotePath: string; Overwrite: Boolean = True): Integer;
function Download(const LocalPath, RemotePath: string; Overwrite: Boolean = True): Integer;
```

**반환값:**
- `0`: 성공
- `-100`: 연결 실패
- 기타: DLL 오류 코드

### DLL 함수

```pascal
// 서버 정보 설정
function SetServerInfo(const lpszServerIP: LPSTR; lPortNo: Integer; bConnTest: Boolean): Integer; stdcall;

// 파일 업로드
function UploadFile(const lpszLocalPath, lpszRemotePath: LPSTR; lpszResultPath: LPSTR; nResultPathLen: Integer): Integer; stdcall;
function UploadFileEx(const lpszLocalPath, lpszRemotePath: LPSTR; bOverwrite: Boolean): Integer; stdcall;

// 파일 다운로드
function DownloadFile(const lpszRemotePath, lpszLocalPath: LPSTR; bOverwrite: Boolean): Integer; stdcall;
```

### 사용 예시

```pascal
var
  Connector: TSMCIFFileConnector;
  FileManager: TSMCFIFFileManager;
  Result: Integer;
begin
  Connector := TSMCIFFileConnector.Create(nil);
  try
    Connector.Address := '192.168.1.100';
    Connector.Port := 4006;
    
    if Connector.Open then
    begin
      FileManager := TSMCFIFFileManager.Create(nil);
      try
        FileManager.Connector := Connector;
        
        // 파일 업로드
        Result := FileManager.Upload('C:\Local\file.pdf', '/Remote/file.pdf', True);
        if Result = 0 then
          ShowMessage('업로드 성공')
        else
          ShowMessage('업로드 실패: ' + IntToStr(Result));
        
        // 파일 다운로드
        Result := FileManager.Download('C:\Local\downloaded.pdf', '/Remote/file.pdf', True);
        if Result = 0 then
          ShowMessage('다운로드 성공')
        else
          ShowMessage('다운로드 실패: ' + IntToStr(Result));
      finally
        FileManager.Free;
      end;
    end
    else
      ShowMessage('서버 연결 실패');
  finally
    Connector.Free;
  end;
end;
```

---

## 핵심 기능 상세 분석

### 1. 간호기록 뷰어 (NRCViewer)

**역할:**
- EMR 기록을 XML 형식으로 표시
- 두 기록을 비교하여 변경 사항 시각화
- 다양한 UI 컴포넌트 지원

**작동 흐름:**
```
XML 소스 입력
    ↓
XML 파싱 (IXMLDOMNode)
    ↓
Generator 패턴으로 컨트롤 생성
    ↓
UI 컴포넌트 표시
```

**비교 모드:**
```
XML1 + XML2 입력
    ↓
각 노드를 이름으로 매칭
    ↓
내용 비교
    ↓
변경 사항 표시 (색상, 스타일 등)
```

### 2. 화면 보안 (ScreenSecurity)

**역할:**
- 특정 화면에 대한 스크린샷 방지
- 비활성 시간 모니터링
- 자동 화면 닫기

**작동 흐름:**
```
보안 창 등록
    ↓
키보드/마우스 훅 설치
    ↓
입력 이벤트 감지 → LastTime 업데이트
    ↓
타이머 체크 (1초 간격)
    ↓
비활성 시간 경과 → 경고 다이얼로그
    ↓
카운트다운 후 입력 없음 → 화면 닫기
```

### 3. 데이터 내보내기 (SMCExport)

**역할:**
- 그리드 데이터를 엑셀/CSV로 내보내기
- 페이지 설정 및 서식 적용
- 진행 상황 표시

**작동 흐름:**
```
그리드 뷰 선택
    ↓
ExcelDocument 생성 및 설정
    ↓
제목, 날짜, 조건 등 추가
    ↓
Excel COM 객체 생성
    ↓
데이터 내보내기
    ↓
서식 적용 (열 너비, 정렬 등)
    ↓
파일 저장
```

### 4. 리포트 뷰어 (SMCLProxy)

**역할:**
- 리포트 미리보기
- 인쇄 기능
- 원격 리포트 지원

**작동 흐름:**
```
리포트 데이터 준비
    ↓
뷰어 폼 생성
    ↓
CSS 설정 적용
    ↓
리포트 렌더링
    ↓
미리보기 표시
    ↓
인쇄 또는 저장
```

---

## 주요 클래스 및 함수

### NRCViewerPackage

#### `TNRCCustomViewer`
```pascal
procedure Clear;
procedure LoadFromXMLSource(const XMLSource: string);
procedure LoadFromXMLFile(const FileName: string);
procedure Compare(const XMLSource1, XMLSource2: string);
```

#### `TNRCViewerGenerator`
```pascal
class procedure RegisterItemGenerator(Generator: TNRCViewerItemGenerator);
class function Execute1(Control: TControl; const Name: string; Datas: TStrings): string;
class function Execute2(const XMLSource: string; Viewer: TNRCCustomViewer): TControl;
```

#### `TNRCViewerComparator`
```pascal
class procedure RegisterComparator(ItemGeneratorClass: TNRCViewerItemGeneratorClass; ComparatorClass: TComparatorClass);
function Generate(const XMLSource1, XMLSource2: string): string;
```

### ScreenSecurityPack

#### 전역 함수
```pascal
procedure RegistSecurityWindow(const ClassName, Caption: String);
procedure UnregistSecurityWindow(const ClassName: String);
```

#### `TSecurityChecker`
```pascal
procedure ActiveInput(Wnd: HWnd);
procedure DoCheck(Sender: TObject);
```

### SMCExport

#### `TSMCExcelForGrid`
```pascal
procedure ExportGrid;
procedure Add(ExcelDocument: TExcelDocument);
```

#### `TExcelDocument`
```pascal
procedure SetFreezePane(AFreezePane: Boolean; AFreezeColumn, AFreezeRow: Integer);
```

#### `TSMCTableForCSV`
```pascal
procedure SMCTableForCSV(Table: TkbmCustomMemTable; FilePath, FormName, UserId: string);
```

### SMCFax

#### `TFAX001D1`
```pascal
function FAX001D1: TFAX001D1;  // 싱글톤
```

### SMCLProxy

#### `TLPA001F1`
```pascal
procedure Prepare(const Report: ISMCReport);
procedure Preview;
```

### SMCFile

#### `TSMCIFFileConnector`
```pascal
function Open: Boolean;
```

#### `TSMCFIFFileManager`
```pascal
function Upload(const LocalPath, RemotePath: string; Overwrite: Boolean = True): Integer;
function Download(const LocalPath, RemotePath: string; Overwrite: Boolean = True): Integer;
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
COM_LV3 (비즈니스 로직) ← 현재 분석
    ↓
MAIN (애플리케이션)
```

**COM_LV3의 역할:**
- COM_LV2 프레임워크 위에 구축되는 특화 기능 모듈
- EMR 뷰어, 화면 보안, 데이터 내보내기 등 비즈니스 로직 제공
- MAIN 애플리케이션에서 직접 사용되는 기능 모듈

---

## 요약

COM_LV3 프로젝트는 Nexmed EHR 시스템의 비즈니스 로직 레이어로, 다음과 같은 핵심 기능을 제공합니다:

1. **NRCViewerPackage**: 간호기록 인증 뷰어 (XML 기반 EMR 표시 및 비교)
2. **ScreenSecurityPack**: 화면 보안 (스크린샷 방지, 비활성 시간 모니터링)
3. **SMCExport**: 데이터 내보내기 (엑셀/CSV)
4. **SMCFax**: 팩스 전송 기능
5. **SMCLProxy**: 리포트 뷰어 (미리보기 및 인쇄)
6. **SMCFile**: 파일 관리 (원격 파일 서버 업로드/다운로드)

이러한 모듈들은 MAIN 애플리케이션에서 직접 사용되어 EHR 시스템의 핵심 비즈니스 기능을 구현합니다.

