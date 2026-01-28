# COM_LV2 프로젝트 상세 분석서

## 목차
1. [프로젝트 개요](#프로젝트-개요)
2. [프로젝트 구조](#프로젝트-구조)
3. [SMC 프레임워크](#smc-프레임워크)
4. [SEHRData 데이터 레이어](#sehrdata-데이터-레이어)
5. [SMCExpress UI 컴포넌트](#smcexpress-ui-컴포넌트)
6. [핵심 기능 상세 분석](#핵심-기능-상세-분석)
7. [주요 클래스 및 함수](#주요-클래스-및-함수)

---

## 프로젝트 개요

**COM_LV2**는 Nexmed EHR 시스템의 핵심 프레임워크 레이어로, 애플리케이션의 기본 인프라를 제공합니다. 이 프로젝트는 다음과 같은 주요 역할을 담당합니다:

- **SMC 프레임워크**: 폼 관리, 스킨 시스템, 도킹 관리
- **SEHRData**: REST API 통신 및 데이터셋 관리
- **SMCExpress**: UI 컴포넌트 라이브러리
- **SMCService**: 서비스 레이어 및 메시지 처리

---

## 프로젝트 구조

```
COM_LV2/
├── SMC/                    # SMC 프레임워크 핵심
│   ├── SMCForm.pas         # 폼 베이스 클래스 및 스킨 시스템
│   ├── SMCFormManager.pas  # BPL 폼 로딩 및 관리
│   ├── SMCMainCommon.pas   # 공통 유틸리티 함수
│   ├── SMCMessageDialog.pas # 메시지 다이얼로그
│   ├── SMCMainDockControl.pas # 도킹 시스템
│   └── SMCInformation.pas  # 전역 변수 및 정보
├── DataSet/                # 데이터 레이어
│   ├── SEHR.Dataset.pas    # 메모리 데이터셋
│   ├── SEHR.RestClient.pas # REST API 클라이언트
│   └── SEHR.ServiceClasses.pas # 서비스 메시지 클래스
├── SMCExpress/             # UI 컴포넌트
│   ├── SMCGrids/           # 그리드 컴포넌트
│   ├── SMCInputs/          # 입력 컴포넌트
│   ├── SMCLayouts/         # 레이아웃 컴포넌트
│   └── SMCViews/           # 뷰 컴포넌트
├── Core/                   # 코어 유틸리티
│   ├── SEHR.JsonDataObjects.pas # JSON 처리
│   └── SEHR.Global.pas      # 전역 유틸리티
└── SMCService/             # 서비스 레이어
```

---

## SMC 프레임워크

### 1. SMCForm.pas - 폼 베이스 클래스

**역할**: 모든 애플리케이션 폼의 베이스 클래스로, 스킨 시스템, 도킹, 메시지 처리 등을 제공합니다.

#### 주요 클래스

##### TSMCForm
```pascal
TSMCForm = class(TForm)
```
- 모든 애플리케이션 폼의 기본 클래스
- 스킨 적용, 도킹 지원, 링크 데이터 처리

**주요 속성**:
- `PackageHandle`: 로드된 BPL 패키지 핸들
- `PackageName`: 패키지 이름
- `FormClassName`: 폼 클래스 이름
- `WorkspaceNo`: 워크스페이스 번호
- `IsDockingForm`: 도킹 가능 여부
- `ShowStatusBar`: 상태바 표시 여부

**주요 메서드**:
- `SMCFormCreate`: 폼 생성 시 초기화
- `SendLinkData`: 다른 폼으로 데이터 전달
- `ReceiveLinkData`: 다른 폼에서 데이터 수신
- `ShowPatInfo`, `ShowPatList`: 환자 정보/목록 표시

##### TSMCSkinCustomFormController
```pascal
TSMCSkinCustomFormController = class(TSMCSkinWinController)
```
- 폼의 스킨 렌더링을 담당하는 컨트롤러
- Non-Client 영역(캡션, 테두리, 스크롤바) 그리기

**주요 기능**:
- `CalculateViewInfo`: 뷰 정보 계산
- `DrawWindowBorder`: 윈도우 테두리 그리기
- `DrawWindowCaption`: 캡션 영역 그리기
- `DrawWindowScrollArea`: 스크롤바 영역 그리기

##### TSMCFormNonClientAreaInfo
```pascal
TSMCFormNonClientAreaInfo = class(TObject)
```
- 폼의 Non-Client 영역 정보를 관리
- 캡션, 테두리, 아이콘, 스크롤바 정보 포함

**주요 속성**:
- `CaptionBounds`: 캡션 영역 좌표
- `BorderBounds`: 테두리 영역 좌표
- `IconInfoList`: 시스템 아이콘 정보 (최소화, 최대화, 닫기)
- `UserIconInfoList`: 사용자 정의 아이콘 정보
- `BizIconInfoList`: 비즈니스 아이콘 정보

#### 주요 메시지 처리

SMC 프레임워크는 Windows 메시지를 확장하여 폼 간 통신을 지원합니다:

```pascal
const
  UM_SMCBASE            = WM_USER * 2;
  UM_LINKDATA           = UM_SMCBASE +  1;  // 링크 데이터 전달
  UM_NEWSCREEN          = UM_SMCBASE + 10;  // 새 화면 생성
  UM_SHOWPATIENTLIST    = UM_SMCBASE + 16;  // 환자 목록 표시
  UM_SHOWPATIENTINFO    = UM_SMCBASE + 17;  // 환자 정보 표시
  UM_GETPATIENTINFO     = UM_SMCBASE + 22;  // 환자 정보 조회
  UM_FORMSHOWCOMPLETE   = UM_SMCBASE + 23;  // 폼 표시 완료
  UM_SMCFORMCREATE      = UM_SMCBASE + 24;  // 폼 생성
  UM_SMCFORMDESTROY     = UM_SMCBASE + 25;  // 폼 소멸
  UM_MENUCREATEFORM     = UM_SMCBASE + 26;  // 메뉴에서 폼 생성
  UM_PAGECHANGE         = UM_SMCBASE + 27;  // 페이지 변경
```

### 2. SMCFormManager.pas - BPL 폼 관리

**역할**: BPL(Borland Package Library) 파일에서 동적으로 폼을 로드하고 관리합니다.

#### 주요 함수

##### LoadBplForm
```pascal
function LoadBplForm(AOwner: TComponent; const AScreenName, APackageName, 
  AMenuGroupId, AMenuId: string; ALinkData: TLinkData; 
  const AClassName: string = ''): TSMCForm;
```

**작동 방식**:
1. 패키지 파일 경로 생성: `BPL 경로 + 패키지명.bpl`
2. 버전 체크: `UM_VERSIONCHECK` 메시지로 서버와 버전 비교
3. 패키지 로드: `LoadPackage` 함수로 BPL 파일 로드
4. 클래스 조회: `GetClass('T' + ProgId)`로 폼 클래스 찾기
5. 폼 생성: 클래스 인스턴스 생성
6. 초기화: 패키지 핸들, 링크 데이터 설정

**예제**:
```pascal
// 패키지명: "PAT001L1", 클래스명: "PAT001F1"
LSMCForm := LoadBplForm(Application, '환자조회', 'PAT001L1', 
  'M0000001', 'M0000002', nil, 'PAT001F1');
```

##### CreateBplForm
```pascal
function CreateBplForm(AOwner: TComponent; ADockSite: TCustomTabControl; 
  const AScreenName, APackageName, AMenuGroupId, AMenuId: string; 
  ALinkData: TLinkData; AExecStyle: TExecStyle; 
  const AClassName: string = ''): TSMCForm;
```

**실행 스타일 (TExecStyle)**:
- `esDockSite`: 도킹 사이트에 도킹
- `esFloat`: 독립 플로팅 윈도우
- `esFloatSubMon`: 서브 모니터에 플로팅
- `esSingle`: 단일 독립 윈도우
- `esModal`: 모달 다이얼로그
- `esContainer`: 컨테이너 내부에 삽입

**작동 흐름**:
```pascal
case AExecStyle of
  esDockSite:
    begin
      // 1. 폼 로드
      LSMCForm := LoadBplForm(...);
      // 2. 도킹 패널 생성
      LDockPanel := TSMCBplDockPanel.Create(LSMCForm);
      // 3. 도킹 사이트에 도킹
      LSMCForm.ManualDock(ADockSite);
      // 4. 링크 데이터 전달
      LSMCForm.SendLinkData := ALinkData;
      // 5. 표시
      LSMCForm.Show;
    end;
  esFloat:
    begin
      // 플로팅 윈도우 생성
      LDockPanel := TSMCBplDockPanel.Create(LSMCForm);
      LDockPanel.DockFormToFloatPanel(...);
      LDockPanel.FloatForm.Show;
    end;
end;
```

##### TSMCContainer
```pascal
TSMCContainer = class(TSMCScrollBox)
```
- 디자인 타임에 패키지를 로드할 수 있는 컨테이너
- 폼 내부에 다른 폼을 삽입할 때 사용

**주요 메서드**:
- `LoadPackage`: 패키지 로드
- `FreePackage`: 패키지 해제

### 3. SMCMainCommon.pas - 공통 유틸리티

**역할**: 애플리케이션 전반에서 사용되는 공통 함수들을 제공합니다.

#### 주요 함수

##### CreateForm
```pascal
function CreateForm(const APackageName: string; 
  const AClassName: string = ''): TSMCForm;
function CreateForm(AOwner: TComponent; ADockSite: TCustomTabControl; 
  const AScreenName, APackageName: string; ALinkData: TLinkData; 
  const AExecStyle: TExecStyle; const AClassName: string = ''): TSMCForm;
```
- 폼 생성을 위한 편의 함수
- 내부적으로 `CreateBplForm` 호출

##### 메시지 다이얼로그 함수
```pascal
procedure SMCMainShowMessage(AMsg: string);
procedure SMCMainWarningMessage(const AMsg: string);
procedure SMCMainErrorMessage(const AMsg: string);
function SMCMainConfirmMessage(const AMsg: string): Integer;
function SMCMainEmergencyMessage(const AMsg: string; Title: string = ''; 
  Buttons: TMsgBoxButtons = [mbbOK]): Integer;
```
- 표준화된 메시지 다이얼로그
- 메시지 코드를 사용하여 다국어 지원

##### PopupMenuToBarMenu
```pascal
procedure PopupMenuToBarMenu(AMenu: TPopupMenu; X, Y: Integer; 
  APopWidth: Integer = 0);
```
- `TPopupMenu`를 DevExpress `TdxBarPopupMenu`로 변환
- 메뉴 구조를 유지하면서 팝업 메뉴 표시

##### 폼 배치 관리
```pascal
procedure SaveFormPlacement(AForm: TCustomForm; IsUserSep: Boolean = True);
procedure LoadFormPlacement(AForm: TCustomForm; IsUserSep: Boolean = True);
```
- 폼의 위치와 크기를 INI 파일에 저장/로드
- 사용자별로 분리 저장 가능

##### MenuTableToTreeList
```pascal
procedure MenuTableToTreeList(AMenuTable: TDataSet; ATreeView: TcxTreeList);
```
- 메뉴 데이터셋을 트리뷰로 변환
- 계층 구조 메뉴 생성

### 4. SMCMainDockControl.pas - 도킹 시스템

**역할**: 폼의 도킹 및 탭 관리를 담당합니다.

#### 주요 클래스

##### TSMCMainPageControl
```pascal
TSMCMainPageControl = class(TCustomTabControl)
```
- 탭 기반 도킹 컨테이너
- 여러 폼을 탭으로 관리

**주요 기능**:
- 탭 추가/삭제
- 탭 드래그 앤 드롭
- 환자 정보/목록 버튼
- 스크롤 가능한 탭

##### TSMCMainTabSheet
```pascal
TSMCMainTabSheet = class(TWinControl)
```
- 개별 탭 시트
- 도킹된 폼을 포함

**주요 속성**:
- `DockForm`: 도킹된 폼
- `TabCaption`: 탭 캡션
- `WorkspaceNo`: 워크스페이스 번호

##### TSMCMainFloatForm
```pascal
TSMCMainFloatForm = class(TSMCForm)
```
- 플로팅 윈도우
- 독립적인 페이지 컨트롤 포함

### 5. SMCMessageDialog.pas - 메시지 다이얼로그

**역할**: 표준화된 메시지 다이얼로그를 제공합니다.

#### 주요 함수

##### SMCGetMessage
```pascal
function SMCGetMessage(const MsgId: string): string;
function SMCGetMessage(const MsgId: string; 
  const Args: array of const): string;
```
- 메시지 코드로 메시지 텍스트 조회
- 다국어 지원
- 포맷 문자열 지원

**작동 방식**:
1. `SMCMessageDataSet`에서 `mesgCd`로 검색
2. `mesgEtcCtn` 필드에서 메시지 텍스트 가져오기
3. 포맷 인자가 있으면 `Format` 함수 적용

##### SMCMessageBox
```pascal
function SMCMessageBox(const MsgId: string; 
  DlgType: TMsgBoxType = mbtInformation; 
  Buttons: TMsgBoxButtons = [mbbOK]): Integer;
```
- 표준 메시지 박스
- 메시지 코드 사용

**다이얼로그 타입**:
- `mbtWarning`: 경고
- `mbtError`: 오류
- `mbtInformation`: 정보
- `mbtConfirmation`: 확인

### 6. SMCInformation.pas - 전역 변수

**역할**: 애플리케이션 전역 변수를 관리합니다.

#### TSMCGlobalVar
```pascal
TSMCGlobalVar = class(TPersistent)
```

**주요 속성**:

**사용자 정보 (TSMCUserInfo)**:
- `lginId`: 로그인 ID
- `userId`: 사원코드
- `userNm`: 사원명
- `dprtCd`, `dprtNm`: 부서코드/명
- `ocfmCd`, `ocfmNm`: 직종코드/명
- `occlCd`, `occlNm`: 직급코드/명
- 등등...

**접속 정보 (TSMCConnectInfo)**:
- `cnctIp`: 접속 IP
- `sesnId`: 세션 ID
- `lginDt`: 로그인 일자
- `systEvnrDvsnCd`: 시스템 환경 구분코드 (D/T/P/R)
- `bplAddr`, `bplPort`: BPL 서버 주소/포트
- `hieType`, `hieIp`, `hiePort`: HIE 연동 정보

**사업장 정보 (TSMCCompanyInfo)**:
- `hsptCd`, `hsptNm`: 병원코드/명
- `cntrCd`, `cntrCdNm`: 센터코드/명
- `bsplCd`, `bsplNm`: 사업장코드/명

**경로 정보 (TSMCPath)**:
- `Root`: 루트 경로
- `Bpl`: BPL 파일 경로
- `Dll`: DLL 파일 경로
- `Ini`: 설정 파일 경로
- `Log`: 로그 파일 경로
- `Temp`: 임시 파일 경로

---

## SEHRData 데이터 레이어

### 1. SEHR.Dataset.pas - 메모리 데이터셋

**역할**: REST API와 연동하는 메모리 데이터셋을 제공합니다.

#### 주요 클래스

##### TSHISDataSet
```pascal
TSHISDataSet = class(TCustomSHISDataSet)
```

**기본 클래스**: `TCustomSHISDataSet` (상속: `TkbmCustomMemTable`)

**주요 속성**:
- `Provider`: `TSHISDataProvider` - 데이터 프로바이더
- `ServiceIDs`: `TServiceIDs` - 서비스 ID (Execute, List)
- `FieldName`: 서비스 응답에서 가져올 필드명
- `Schema`: `TSMCSCustomTableSchema` - 테이블 스키마
- `Kind`: `TSMCSTableKind` - 테이블 처리 방식
  - `tkRefresh`: 새로고침 (기존 데이터 삭제 후 추가)
  - `tkAppend`: 추가 (기존 데이터에 추가)
  - `tkFix`: 고정 (변경 없음)
- `Validations`: `TSHISValidateItems` - 유효성 검증 규칙
- `IncludeGlobalValue`: 전역 변수 포함 여부

**주요 메서드**:

##### Open
```pascal
procedure Open;
```
- 데이터셋을 열고 서비스를 호출하여 데이터를 가져옵니다.

**작동 흐름**:
1. `Provider`가 설정되어 있는지 확인
2. `ServiceIDs.List` 또는 `ServiceIDs.Execute` 서비스 호출
3. 서비스 응답을 JSON으로 파싱
4. `FieldName`에 해당하는 데이터 추출
5. `Kind`에 따라 데이터 처리:
   - `tkRefresh`: 기존 데이터 삭제 후 추가
   - `tkAppend`: 기존 데이터에 추가
6. `DoDone` 이벤트 발생

##### Post
```pascal
procedure Post; override;
```
- 레코드 변경사항을 저장합니다.
- 유효성 검증 수행
- `DoAfterPost` 이벤트 발생

##### Validation
```pascal
function Validation: Boolean;
```
- 모든 필드에 대한 유효성 검증 수행
- `Validations` 컬렉션의 규칙 적용

**유효성 검증 규칙 (TSHISValidateItem)**:
- `Required`: 필수 여부
- `Min`, `Max`: 최소/최대값
- `Pattern`: 정규식 패턴

##### ToJSON / FromJSON
```pascal
function ToJSON: TJSONObject;
procedure FromJSON(AValue: TJSONObject);
```
- 현재 레코드를 JSON으로 변환
- JSON에서 레코드 생성

##### CopyFrom / CopyTo
```pascal
procedure CopyFrom(ADataSet: TCustomSHISDataSet; const AppendMode: Boolean = False);
procedure CopyTo(ADataSet: TCustomSHISDataSet; const AppendMode: Boolean = False);
```
- 다른 데이터셋과 데이터 복사

##### TSHISDataProvider
```pascal
TSHISDataProvider = class(TComponent)
```

**역할**: REST API 서비스를 호출하고 응답을 처리합니다.

**주요 속성**:
- `RestClient`: `TSHISRestClient` - REST 클라이언트
- `ServiceID`: 서비스 ID
- `ServiceIDs`: `TServiceIDs` - 서비스 ID 컬렉션

**주요 메서드**:

##### Execute
```pascal
function Execute(AMessage: TSMCSCustomMessage; 
  ADataSet: TCustomSHISDataSet = nil): IRestResponse;
```
- 서비스를 동기적으로 호출
- `ServiceIDs.Execute` 사용

##### ExecuteList
```pascal
function ExecuteList(AMessage: TSMCSCustomMessage; 
  ADataSet: TCustomSHISDataSet = nil): IRestResponse;
```
- 목록 조회 서비스를 호출
- `ServiceIDs.List` 사용

##### ExecuteAsync
```pascal
procedure ExecuteAsync(AMessage: TSMCSCustomMessage; 
  ADataSet: TCustomSHISDataSet; AOnDone: TNotifyEvent = nil);
```
- 서비스를 비동기적으로 호출
- 백그라운드 스레드에서 실행

### 2. SEHR.RestClient.pas - REST API 클라이언트

**역할**: HTTP/HTTPS 통신을 통해 REST API를 호출합니다.

#### 주요 클래스

##### TSHISRestClient
```pascal
TSHISRestClient = class(TComponent)
```

**주요 속성**:
- `ServerName`: 서버 주소
- `ServerPort`: 서버 포트
- `Protocol`: 프로토콜 (http/https)
- `SessionID`: 세션 ID
- `ContentType`: 컨텐트 타입
- `Accept`: Accept 헤더
- `UseCompress`: 압축 사용 여부
- `ConnectionTimeout`: 연결 타임아웃
- `ReadTimeout`: 읽기 타임아웃
- `HttpLibType`: HTTP 라이브러리 타입 (0: Indy, 1: CURL)
- `Logging`: 로깅 여부

**주요 메서드**:

##### Post
```pascal
function Post(AResource: string; ABodyString: string; 
  ATimeOut: Integer = _TIMEOUT): IRestResponse;
```
- POST 요청 전송
- JSON 본문 포함

**작동 흐름**:
1. URL 구성: `Protocol://ServerName:ServerPort/AResource`
2. 헤더 설정: Content-Type, Accept, Session ID 등
3. 요청 본문 설정: `ABodyString` (JSON)
4. HTTP 요청 전송
5. 응답 수신 및 `IRestResponse` 반환

##### GetFile
```pascal
function GetFile(AResource: string; AStream: TStream): Integer;
function GetFile(AResource, AFileName: string): Integer;
function GetFile(AtchFileId: String; AtchFileSno: Integer; 
  AFileName: string): Integer;
```
- 파일 다운로드
- 스트림 또는 파일로 저장

##### PostFile
```pascal
function PostFile(AStream: TIdMultiPartFormDataStream; 
  var FileInfoList: TList<TEHRFileInfo>): Integer;
function PostFile(AFieldName, AFileName: string; 
  var FileInfoList: TList<TEHRFileInfo>): Integer;
```
- 멀티파트 폼 데이터로 파일 업로드
- 파일 정보 리스트 반환

##### Asynch
```pascal
procedure Asynch(AProc: TProc<IRestResponse>; 
  AProcErr: TProc<Exception> = nil; 
  AProcAlways: TProc = nil; 
  ASynchronized: Boolean = false);
```
- 비동기 요청
- 완료/오류/항상 실행 콜백 지원

##### IRestResponse
```pascal
IRestResponse = interface
  function Body: TStringStream;
  function BodyAsJSON: TJSONObject;
  function BodyAsString: string;
  function ResponseCode: Word;
  function ResponseText: string;
  function ErrorCode: Integer;
  function ErrorMessage: string;
  function Headers: TStringlist;
  function GetHeaderValue(const Name: string): string;
end;
```

**주요 메서드**:
- `BodyAsJSON`: 응답 본문을 JSON 객체로 변환
- `BodyAsString`: 응답 본문을 문자열로 반환
- `ResponseCode`: HTTP 응답 코드
- `ErrorCode`: 오류 코드
- `ErrorMessage`: 오류 메시지

### 3. SEHR.ServiceClasses.pas - 서비스 메시지 클래스

**역할**: 서버와의 통신에 사용되는 메시지 구조를 정의합니다.

#### 주요 클래스

##### TSMCSCustomMessage
```pascal
TSMCSCustomMessage = class(TPersistent)
```

**역할**: 서비스 요청/응답 메시지의 루트 클래스

**주요 속성**:
- `Header`: `TSMCSCustomMessageHeader` - 메시지 헤더
- `Schema`: `TSMCSCustomMessageSchema` - 메시지 스키마
- `Data`: `TSMCSCustomMessageData` - 메시지 데이터
- `Kind`: `TSMCSMessageKind` - 메시지 종류
  - `mkCore`: 코어
  - `mkSystem`: 시스템
  - `mkBusiness`: 업무
  - `mkFile`: 파일

**주요 메서드**:
- `ToJSON`: JSON으로 변환
- `FromJSON`: JSON에서 생성
- `ToStream`: 스트림으로 변환
- `FromStream`: 스트림에서 생성

##### TSMCSCustomMessageHeader
```pascal
TSMCSCustomMessageHeader = class(TPersistent)
```

**주요 속성**:
- `ServiceID`: 서비스 ID
- `TransactionID`: 트랜잭션 ID
- `UserID`: 사용자 ID
- `CompanyCode`: 회사 코드
- `RequestTime`: 요청 시간
- `ResponseTime`: 응답 시간

##### TSMCSCustomMessageSchema
```pascal
TSMCSCustomMessageSchema = class(TOwnedCollection)
```

**역할**: 메시지의 필드 구조를 정의

**주요 메서드**:
- `FieldByName`: 필드명으로 필드 찾기
- `FieldByPath`: 경로로 필드 찾기
- `CreateData`: 데이터 객체 생성

##### TSMCSCustomMessageSchemaField
```pascal
TSMCSCustomMessageSchemaField = class(TCollectionItem)
```

**주요 속성**:
- `Name`: 필드명
- `Type1`: 필드 타입 (Integer, Float, String, DateTime, Mass, Record)
- `Size`: 크기
- `Scale`: 소수점 자릿수
- `Count`: 배열 크기
- `Options`: 옵션 (Read, Write, Encode, Compress)
- `Reference`: 참조 필드명

##### TSMCSCustomMessageData
```pascal
TSMCSCustomMessageData = class(TPersistent)
```

**역할**: 실제 데이터를 담는 클래스

**주요 메서드**:
- `FieldByName`: 필드명으로 필드 찾기
- `FieldByPath`: 경로로 필드 찾기
- `ToJSON`: JSON으로 변환
- `FromJSON`: JSON에서 생성

##### TSMCSCustomMessageDataField
```pascal
TSMCSCustomMessageDataField = class(TPersistent)
```

**주요 속성**:
- `AsInteger`: 정수값
- `AsFloat`: 실수값
- `AsString`: 문자열값
- `AsDateTime`: 날짜/시간값
- `AsMass`: 바이트 배열값
- `Value`: Variant 값

---

## SMCExpress UI 컴포넌트

SMCExpress는 DevExpress 컴포넌트를 기반으로 한 확장 UI 컴포넌트 라이브러리입니다.

### 주요 컴포넌트 그룹

#### 1. SMCGrids - 그리드 컴포넌트
- `TSMCGrid`: 확장 그리드
- `TSMCWebBrowserGrid`: 웹 브라우저 그리드
- `TSMCFrameGrid`: 프레임 그리드

#### 2. SMCInputs - 입력 컴포넌트
- `TSMCEdit`: 확장 에디트
- `TSMCComboView`: 콤보 뷰
- `TSMCSearchBox`: 검색 박스
- `TSMCProcessCheckbox`: 프로세스 체크박스

#### 3. SMCLayouts - 레이아웃 컴포넌트
- `TSMCPanel`: 확장 패널
- `TSMCSplitter`: 스플리터
- `TSMCScrollBox`: 스크롤 박스

#### 4. SMCViews - 뷰 컴포넌트
- 다양한 뷰 컴포넌트 제공

---

## 핵심 기능 상세 분석

### 1. BPL 동적 로딩 시스템

#### 작동 원리

1. **패키지 파일 경로 생성**
   ```pascal
   sFileName := LGlobalVar.Path.Bpl + sPackageName + '.bpl';
   ```

2. **버전 체크**
   ```pascal
   if not Boolean(SendMessage(Application.MainFormHandle, 
     UM_VERSIONCHECK, Integer(AOwner), Integer(sFormInfo))) then
     Exit;
   ```
   - 메인 폼에 버전 체크 메시지 전송
   - 서버와 로컬 버전 비교

3. **패키지 로드**
   ```pascal
   hPackage := LoadPackage(sFileName, nil);
   ```
   - Windows API `LoadLibrary` 사용
   - 패키지 핸들 반환

4. **클래스 조회**
   ```pascal
   sClassName := 'T' + sProgId;
   LFormClass := GetClass(sClassName);
   ```
   - `GetClass`로 등록된 클래스 찾기
   - 클래스명 규칙: `T` + `ProgId`

5. **폼 생성**
   ```pascal
   Result := TComponentClass(LFormClass).Create(AOwner) as TSMCForm;
   ```

6. **초기화**
   ```pascal
   Result.PackageHandle := hPackage;
   Result.PackageName := sPackageName;
   Result.FormClassName := Copy(sClassName, 2, Length(sClassName));
   Result.SendLinkData := ALinkData;
   ```

#### 패키지 해제

폼이 닫힐 때 패키지를 해제합니다:
```pascal
if (FPackageForm.PackageHandle <> 0) then
  UnloadPackage(FPackageForm.PackageHandle);
```

### 2. 스킨 시스템

#### 스킨 적용 과정

1. **스킨 컨트롤러 생성**
   ```pascal
   FController := TSMCSkinCustomFormController.CreateEx(Self);
   ```

2. **윈도우 프로시저 후킹**
   ```pascal
   procedure TSMCSkinCustomFormController.HookWndProc;
   begin
     FWindowProcObject := cxWindowProcController.Add(Handle, WndProc);
   end;
   ```

3. **Non-Client 영역 그리기**
   - `WM_NCPAINT`: 캡션, 테두리 그리기
   - `WM_NCCALCSIZE`: 클라이언트 영역 크기 계산
   - `WM_NCHITTEST`: 마우스 히트 테스트

4. **스킨 요소 그리기**
   ```pascal
   procedure TSMCSkinCustomFormController.DrawWindowBorder(DC: HDC = 0);
   procedure TSMCSkinCustomFormController.DrawWindowCaption(...);
   procedure TSMCSkinCustomFormController.DrawWindowScrollArea(DC: HDC = 0);
   ```

### 3. 도킹 시스템

#### 도킹 프로세스

1. **도킹 패널 생성**
   ```pascal
   LDockPanel := TSMCBplDockPanel.Create(LSMCForm);
   ```

2. **도킹 사이트에 도킹**
   ```pascal
   LSMCForm.DragKind := dkDock;
   LSMCForm.ManualDock(ADockSite);
   ```

3. **탭 추가**
   ```pascal
   LSMCPageControl.ActivePage := GetParentTabSheet(LSMCForm);
   ```

#### 플로팅 윈도우

1. **플로팅 폼 생성**
   ```pascal
   FFloatForm := TSMCMainFloatForm.Create(Application);
   ```

2. **페이지 컨트롤 설정**
   ```pascal
   FFloatForm.PageDivision.PageControl1.Align := alClient;
   ```

3. **폼 도킹**
   ```pascal
   LSMCForm.ManualDock(LSMCPageControl);
   ```

### 4. 데이터 바인딩

#### 데이터셋과 그리드 연동

1. **데이터셋 설정**
   ```pascal
   DataSet.Provider := DataProvider;
   DataSet.ServiceIDs.List := 'PatientListService';
   DataSet.FieldName := 'patientList';
   ```

2. **그리드 연결**
   ```pascal
   Grid.DataController.DataSource := DataSource;
   DataSource.DataSet := DataSet;
   ```

3. **데이터 로드**
   ```pascal
   DataSet.Open;
   ```

4. **자동 바인딩**
   - 스키마에서 필드 정의 자동 생성
   - 그리드 컬럼 자동 생성

### 5. 링크 데이터 시스템

#### 링크 데이터 전달

```pascal
type
  TLinkData = class
    SenderObject: TObject;
    SenderForm: TSMCForm;
    Data: Variant;
    FormPosition: TLinkDataFormPosition;
    ShowForm: Boolean;
    ActivateForm: Boolean;
    StayOnTop: Boolean;
    OnAfterFormCreate: TLinkDataFormCreateEvent;
    OnAfterSendLink: TLinkDataSendLinkEvent;
  end;
```

**사용 예제**:
```pascal
var
  LinkData: TLinkData;
begin
  LinkData := TLinkData.Create;
  LinkData.SenderForm := Self;
  LinkData.Data := PatientNo;
  LinkData.FormPosition := ldfpParentFormCenter;
  
  CreateForm('PAT001L1', 'PAT001F1', LinkData, esModal);
end;
```

**수신 처리**:
```pascal
procedure TForm.ReceiveLinkData(ALinkData: TLinkData);
begin
  if ALinkData <> nil then
  begin
    PatientNo := ALinkData.Data;
    // 데이터 처리
  end;
end;
```

---

## 주요 클래스 및 함수 요약

### SMC 프레임워크

| 클래스/함수 | 파일 | 설명 |
|------------|------|------|
| `TSMCForm` | SMCForm.pas | 폼 베이스 클래스 |
| `TSMCSkinCustomFormController` | SMCForm.pas | 스킨 컨트롤러 |
| `LoadBplForm` | SMCFormManager.pas | BPL 폼 로드 |
| `CreateBplForm` | SMCFormManager.pas | BPL 폼 생성 |
| `TSMCContainer` | SMCFormManager.pas | 컨테이너 컴포넌트 |
| `TSMCMainPageControl` | SMCMainDockControl.pas | 페이지 컨트롤 |
| `TSMCMainTabSheet` | SMCMainDockControl.pas | 탭 시트 |
| `CreateForm` | SMCMainCommon.pas | 폼 생성 편의 함수 |
| `SMCMainMessageBox` | SMCMainCommon.pas | 메시지 박스 |
| `TSMCGlobalVar` | SMCInformation.pas | 전역 변수 |

### SEHRData 레이어

| 클래스/함수 | 파일 | 설명 |
|------------|------|------|
| `TSHISDataSet` | SEHR.Dataset.pas | 메모리 데이터셋 |
| `TSHISDataProvider` | SEHR.Dataset.pas | 데이터 프로바이더 |
| `TSHISRestClient` | SEHR.RestClient.pas | REST 클라이언트 |
| `IRestResponse` | SEHR.RestClient.pas | REST 응답 인터페이스 |
| `TSMCSCustomMessage` | SEHR.ServiceClasses.pas | 메시지 클래스 |
| `TSMCSCustomMessageSchema` | SEHR.ServiceClasses.pas | 메시지 스키마 |
| `TSMCSCustomMessageData` | SEHR.ServiceClasses.pas | 메시지 데이터 |

---

## 결론

COM_LV2 프로젝트는 Nexmed EHR 시스템의 핵심 인프라를 제공하는 프레임워크 레이어입니다. 주요 특징:

1. **모듈화**: BPL 패키지 기반의 동적 로딩으로 모듈화된 개발 지원
2. **확장성**: 스킨 시스템과 도킹 시스템으로 UI 확장 용이
3. **표준화**: 표준화된 데이터 통신 및 메시지 처리
4. **재사용성**: 공통 컴포넌트와 유틸리티 함수 제공

이 프레임워크를 통해 개발자는 비즈니스 로직에 집중할 수 있으며, 일관된 UI/UX와 안정적인 데이터 통신을 보장받을 수 있습니다.

