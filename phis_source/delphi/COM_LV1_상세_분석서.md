# COM_LV1 프로젝트 상세 분석서

## 목차
1. [프로젝트 개요](#프로젝트-개요)
2. [프로젝트 구조](#프로젝트-구조)
3. [kbmMemTable - 메모리 테이블](#kbmmemtable---메모리-테이블)
4. [MQTT - 통신 프로토콜](#mqtt---통신-프로토콜)
5. [SynEdit - 코드 에디터](#synedit---코드-에디터)
6. [TMS - UI 컴포넌트](#tms---ui-컴포넌트)
7. [DevExpress - UI 프레임워크](#devexpress---ui-프레임워크)
8. [SWBC_DLL - 암호화 라이브러리](#swbc_dll---암호화-라이브러리)
9. [핵심 기능 상세 분석](#핵심-기능-상세-분석)
10. [주요 클래스 및 함수](#주요-클래스-및-함수)

---

## 프로젝트 개요

**COM_LV1**는 Nexmed EHR 시스템의 써드파티 라이브러리 레이어로, COM_LV0 위에 구축되는 외부 컴포넌트 및 라이브러리 패키지들을 포함합니다. 이 프로젝트는 다음과 같은 주요 역할을 담당합니다:

- **메모리 데이터셋**: kbmMemTable을 통한 인메모리 데이터 관리
- **통신 프로토콜**: MQTT를 통한 실시간 메시징
- **UI 컴포넌트**: DevExpress, TMS를 통한 고급 UI 구성 요소
- **코드 에디터**: SynEdit를 통한 텍스트 편집 기능
- **암호화**: SWBC_DLL을 통한 데이터 암호화

---

## 프로젝트 구조

```
COM_LV1/
├── kbmMemTable/              # 메모리 테이블 패키지
│   ├── Source/               # 소스 파일
│   │   ├── kbmMemTable.pas   # 메인 메모리 테이블 클래스
│   │   ├── kbmMemSQL.pas     # SQL 지원
│   │   └── ...
│   ├── Design/               # 디자인 타임 컴포넌트
│   └── Package/              # 패키지 파일
├── MQTT/                     # MQTT 통신 패키지
│   ├── Source/               # 소스 파일
│   │   ├── TMS.MQTT.Client.pas        # MQTT 클라이언트
│   │   ├── TMS.MQTT.Connection.pas    # 네트워크 연결
│   │   ├── TMS.MQTT.Packets.*.pas     # MQTT 패킷 처리
│   │   └── ...
│   └── Package/              # 패키지 파일
├── SynEdit/                  # 코드 에디터 패키지
│   ├── Source/               # 소스 파일
│   │   ├── SynEdit.pas       # 메인 에디터 컴포넌트
│   │   ├── SynHighlighter*.pas # 문법 하이라이터
│   │   └── ...
│   └── Package/              # 패키지 파일
├── TMS/                      # TMS 컴포넌트 라이브러리
│   ├── *.pas                 # 다양한 TMS 컴포넌트
│   └── TMSD.dpk              # 패키지 파일
├── DevExpress/               # DevExpress UI 프레임워크
│   ├── Express*/             # 다양한 Express 컴포넌트
│   └── ...
└── SWBC_DLL/                 # 암호화 DLL
    ├── samsung_ehr_swbc.cpp  # 암호화 구현
    ├── SWBC_Samsung_EHR.h    # 암호화 키 정의
    └── ...
```

---

## kbmMemTable - 메모리 테이블

### 개요
**kbmMemTable**은 TDataSet을 상속받은 인메모리 데이터셋 컴포넌트로, 데이터베이스 연결 없이 메모리에서 데이터를 관리할 수 있게 해줍니다.

### 주요 파일
- **`kbmMemTable.pas`**: 메인 메모리 테이블 클래스 (`TkbmCustomMemTable`, `TkbmMemTable`)
- **`kbmMemSQL.pas`**: SQL 쿼리 지원
- **`kbmMemTypes.pas`**: 타입 정의

### 주요 클래스

#### `TkbmCustomMemTable`
모든 메모리 테이블의 베이스 클래스입니다.

**주요 속성:**
```pascal
property Active: Boolean;                    // 테이블 활성화 여부
property FieldDefs: TFieldDefs;             // 필드 정의
property IndexDefs: TIndexDefs;             // 인덱스 정의
property IndexName: string;                  // 현재 인덱스 이름
property IndexFieldNames: string;            // 인덱스 필드 이름
property SortFields: string;                 // 정렬 필드
property Filter: string;                     // 필터 조건
property Filtered: Boolean;                  // 필터 활성화 여부
property MasterFields: string;               // 마스터 필드 (Master/Detail)
property MasterSource: TDataSource;          // 마스터 데이터 소스
property Persistent: Boolean;                // 영속성 활성화
property PersistentFile: TFileName;         // 영속성 파일 경로
```

**주요 메서드:**
```pascal
// 데이터 로드/저장
procedure LoadFromStream(Stream: TStream; Format: TkbmMemTableFormat = mtfCSV);
procedure SaveToStream(Stream: TStream; Format: TkbmMemTableFormat = mtfCSV);
procedure LoadFromFile(const FileName: string; Format: TkbmMemTableFormat = mtfCSV);
procedure SaveToFile(const FileName: string; Format: TkbmMemTableFormat = mtfCSV);
procedure LoadFromDataSet(Source: TDataSet; CopyOptions: TkbmMemTableCopyTableOptions);
procedure SaveToDataSet(Dest: TDataSet; SaveOptions: TkbmMemTableSaveFlags);

// 테이블 구조
procedure CreateTable;
procedure CreateTableAs(Source: TDataSet; CopyOptions: TkbmMemTableCopyTableOptions);
procedure EmptyTable;
procedure DeleteTable;

// 인덱스 관리
function AddIndex(const Name, Fields: string; Options: TIndexOptions): TkbmIndex;
procedure DeleteIndex(const Name: string);
procedure UpdateIndexes;

// 정렬
procedure Sort(Options: TkbmMemTableCompareOptions; const SortFields: string);
procedure SortOn(const SortFields: string; Options: TkbmMemTableCompareOptions = []);

// 검색
function Locate(const KeyFields: string; const KeyValues: Variant; Options: TLocateOptions): Boolean;
function Lookup(const KeyFields: string; const KeyValues: Variant; const ResultFields: string): Variant;

// 범위 설정
procedure SetRangeStart;
procedure SetRangeEnd;
procedure ApplyRange;
procedure CancelRange;

// 키 검색
procedure SetKey;
procedure EditKey;
function GotoKey: Boolean;
function FindKey(const KeyValues: array of const): Boolean;
function FindNearest(const KeyValues: array of const): Boolean;
```

**주요 기능:**

1. **인메모리 데이터 관리**
   - 데이터베이스 연결 없이 메모리에서 데이터 관리
   - 빠른 데이터 접근 및 조작

2. **인덱스 지원**
   - 다중 인덱스 생성 및 관리
   - 인덱스를 통한 빠른 검색 및 정렬

3. **Master/Detail 관계**
   - 다른 데이터셋과의 마스터/디테일 관계 지원
   - 자동 필터링 및 동기화

4. **데이터 영속성**
   - CSV, 바이너리 형식으로 파일 저장/로드
   - 자동 저장/로드 기능

5. **필터링 및 정렬**
   - 복잡한 필터 조건 지원
   - 동적 정렬 기능

6. **버전 관리**
   - 레코드 버전 관리 (EnableVersioning)
   - 변경 이력 추적

7. **트랜잭션 지원**
   - 로컬 트랜잭션 처리
   - StartTransaction, Commit, Rollback

### 사용 예시

```pascal
// 테이블 생성 및 데이터 로드
var
  MemTable: TkbmMemTable;
begin
  MemTable := TkbmMemTable.Create(nil);
  try
    // 필드 정의
    MemTable.FieldDefs.Add('ID', ftInteger);
    MemTable.FieldDefs.Add('Name', ftString, 50);
    MemTable.FieldDefs.Add('Age', ftInteger);
    MemTable.CreateTable;
    
    // 데이터 추가
    MemTable.Append;
    MemTable.FieldByName('ID').AsInteger := 1;
    MemTable.FieldByName('Name').AsString := '홍길동';
    MemTable.FieldByName('Age').AsInteger := 30;
    MemTable.Post;
    
    // 정렬
    MemTable.SortOn('Name', [mtcoCaseInsensitive]);
    
    // 검색
    if MemTable.Locate('ID', 1, []) then
      ShowMessage(MemTable.FieldByName('Name').AsString);
      
    // 파일 저장
    MemTable.SaveToFile('data.csv', mtfCSV);
  finally
    MemTable.Free;
  end;
end;
```

---

## MQTT - 통신 프로토콜

### 개요
**MQTT (Message Queuing Telemetry Transport)**는 경량 메시징 프로토콜로, IoT 및 실시간 통신에 적합합니다. TMS MQTT 클라이언트를 사용하여 MQTT 브로커와 통신합니다.

### 주요 파일
- **`TMS.MQTT.Client.pas`**: MQTT 클라이언트 메인 클래스
- **`TMS.MQTT.Connection.pas`**: 네트워크 연결 관리
- **`TMS.MQTT.Packets.*.pas`**: MQTT 패킷 처리 (CONNECT, PUBLISH, SUBSCRIBE 등)
- **`TMS.MQTT.Client.Session.pas`**: 세션 관리
- **`TMS.MQTT.Client.Threading.pas`**: 스레드 처리

### 주요 클래스

#### `TTMSMQTTClient`
MQTT 클라이언트의 메인 클래스입니다.

**주요 속성:**
```pascal
property ClientID: string;                    // 클라이언트 ID
property BrokerHostName: string;              // 브로커 호스트명
property BrokerPort: integer;                // 브로커 포트 (기본: 1883)
property UseSSL: Boolean;                    // SSL 사용 여부
property IPVersion: TIdIPVersion;             // IP 버전 (IPv4/IPv6)
property ProtocolVersion: TTMSMQTTProtocolVersion; // 프로토콜 버전
property KeepAliveSettings: TTMSMQTTKeepAliveSettings; // Keep-Alive 설정
property Credentials: TTMSMQTTCredentials;   // 인증 정보
property LastWillSettings: TTMSMQTTLastWillSettings; // Last Will 설정
property ConnectionStatus: TTMSMQTTConnectionStatus; // 연결 상태
```

**주요 메서드:**
```pascal
// 연결 관리
procedure Connect(ACleanSession: Boolean = True);
procedure Disconnect;
function IsConnected: Boolean;

// 메시지 발행
function Publish(ATopic: string; APayload: string; AQos: TTMSMQTTQoS = qosAtMostOnce; ARetain: Boolean = False): Word; overload;
function Publish(ATopic: string; APayload: TBytes; AQos: TTMSMQTTQoS = qosAtMostOnce; ARetain: Boolean = False): Word; overload;

// 구독
function Subscribe(ATopic: string; ATopicQosLevel: TTMSMQTTQoS = qosAtMostOnce): Word; overload;
function Subscribe(ATopics: TTMSStringArray; ATopicQosLevels: TTMSQosArray): Word; overload;

// 구독 해제
function Unsubscribe(ATopic: string): Word; overload;
function Unsubscribe(ATopics: TTMSStringArray): Word; overload;

// Keep-Alive
procedure Ping;
```

**주요 이벤트:**
```pascal
property OnConnectedStatusChanged: TTMSMQTTOnConnectedStatusChangedEvent; // 연결 상태 변경
property OnPublishReceived: TTMSMQTTOnPublishReceivedEvent; // 메시지 수신
property OnSubscriptionAcknowledged: TTMSMQTTOnSubscriptionAcknowledgedEvent; // 구독 확인
property OnPacketReceived: TTMSMQTTPacketEvent; // 패킷 수신
property OnPacketSent: TTMSMQTTPacketEvent; // 패킷 전송
property OnPingRequest: TNotifyEvent; // Ping 요청
property OnPingResponse: TNotifyEvent; // Ping 응답
```

**QoS (Quality of Service) 레벨:**
- `qosAtMostOnce` (0): 최대 한 번 전달 (fire and forget)
- `qosAtLeastOnce` (1): 최소 한 번 전달 (확인 필요)
- `qosExactlyOnce` (2): 정확히 한 번 전달 (중복 방지)

### 사용 예시

```pascal
var
  MQTTClient: TTMSMQTTClient;
begin
  MQTTClient := TTMSMQTTClient.Create(nil);
  try
    // 연결 설정
    MQTTClient.ClientID := 'NexmedEHR_Client';
    MQTTClient.BrokerHostName := 'mqtt.example.com';
    MQTTClient.BrokerPort := 1883;
    MQTTClient.UseSSL := False;
    
    // Keep-Alive 설정
    MQTTClient.KeepAliveSettings.KeepConnectionAlive := True;
    MQTTClient.KeepAliveSettings.KeepAliveInterval := 60;
    MQTTClient.KeepAliveSettings.AutoReconnect := True;
    MQTTClient.KeepAliveSettings.AutoReconnectInterval := 30;
    
    // 인증 정보
    MQTTClient.Credentials.Username := 'user';
    MQTTClient.Credentials.Password := 'password';
    
    // 이벤트 핸들러
    MQTTClient.OnConnectedStatusChanged := procedure(Sender: TObject; const AConnected: Boolean; AStatus: TTMSMQTTConnectionStatus)
    begin
      if AConnected then
        ShowMessage('MQTT 연결 성공')
      else
        ShowMessage('MQTT 연결 끊김');
    end;
    
    MQTTClient.OnPublishReceived := procedure(Sender: TObject; APacketID: Word; ATopic: string; APayload: TBytes)
    var
      PayloadStr: string;
    begin
      PayloadStr := TEncoding.UTF8.GetString(APayload);
      ShowMessage(Format('토픽: %s, 메시지: %s', [ATopic, PayloadStr]));
    end;
    
    // 연결
    MQTTClient.Connect(True);
    
    // 구독
    MQTTClient.Subscribe('patient/update', qosAtLeastOnce);
    MQTTClient.Subscribe('alarm/notification', qosAtLeastOnce);
    
    // 메시지 발행
    MQTTClient.Publish('patient/update', '{"patientId": 123, "status": "updated"}', qosAtLeastOnce);
    
    // 연결 유지...
    // ...
    
    // 연결 해제
    MQTTClient.Disconnect;
  finally
    MQTTClient.Free;
  end;
end;
```

### COM_LV2에서의 사용
COM_LV2의 `SEHR.Dataset.pas`에서 MQTT 클라이언트를 사용하여 실시간 데이터 업데이트를 처리합니다:

```pascal
procedure TSHISDataProvider.ConnectMQTTClient;
begin
  FMQTTClient := TTMSMQTTClient.Create(Self);
  
  FMQTTClient.ClientID := MQTTClientClientId + '_' + MQTTLocalIp;
  FMQTTClient.BrokerHostName := MQTTClientIp;
  FMQTTClient.BrokerPort := StrToInt(MQTTClientPort);
  FMQTTClient.Credentials.Username := MQTTClientClientId;
  FMQTTClient.Credentials.Password := MQTTClientClientId;
  FMQTTClient.KeepAliveSettings.AutoReconnect := True;
  FMQTTClient.KeepAliveSettings.AutoReconnectInterval := 30;
  
  FMQTTClient.OnConnectedStatusChanged := ClientOnConnectedStatusChanged;
  FMQTTClient.OnPublishReceived := PublishReceived;
end;
```

---

## SynEdit - 코드 에디터

### 개요
**SynEdit**는 고급 텍스트 편집 컴포넌트로, 코드 편집기, 로그 뷰어, 스크립트 편집기 등에 사용됩니다.

### 주요 파일
- **`SynEdit.pas`**: 메인 에디터 컴포넌트 (`TCustomSynEdit`, `TSynEdit`)
- **`SynHighlighter*.pas`**: 다양한 언어의 문법 하이라이터
- **`SynEditSearch.pas`**: 검색 기능
- **`SynEditPrint.pas`**: 인쇄 기능

### 주요 클래스

#### `TSynEdit`
메인 에디터 컴포넌트입니다.

**주요 속성:**
```pascal
property Lines: TStrings;                     // 텍스트 라인
property Text: string;                        // 전체 텍스트
property CaretX: Integer;                     // 커서 X 위치
property CaretY: Integer;                     // 커서 Y 위치
property SelText: string;                     // 선택된 텍스트
property SelStart: Integer;                   // 선택 시작 위치
property SelLength: Integer;                  // 선택 길이
property ReadOnly: Boolean;                   // 읽기 전용 여부
property Options: TSynEditorOptions;          // 에디터 옵션
property Highlighter: TSynCustomHighlighter; // 문법 하이라이터
property Font: TFont;                         // 폰트
property Color: TColor;                       // 배경색
property Gutter: TSynGutter;                 // 거터 (라인 번호 영역)
```

**주요 메서드:**
```pascal
// 텍스트 조작
procedure Clear;
procedure ClearSelection;
procedure SelectAll;
procedure CopyToClipboard;
procedure PasteFromClipboard;
procedure CutToClipboard;

// 검색 및 교체
function SearchReplace(const ASearch, AReplace: string; AOptions: TSynSearchOptions): Integer;
function SearchNext: Boolean;
function SearchPrevious: Boolean;

// 커서 이동
procedure GotoLineAndCenter(ALine: Integer);
procedure GotoXY(X, Y: Integer);

// 인쇄
procedure Print;
procedure PrintPreview;
```

**주요 이벤트:**
```pascal
property OnChange: TNotifyEvent;             // 텍스트 변경
property OnStatusChange: TStatusChangeEvent; // 상태 변경
property OnPaint: TPaintEvent;               // 그리기
property OnReplaceText: TReplaceTextEvent;   // 텍스트 교체
```

### 사용 예시

```pascal
var
  SynEdit: TSynEdit;
  Highlighter: TSynPasSyn;
begin
  SynEdit := TSynEdit.Create(nil);
  try
    // 기본 설정
    SynEdit.Parent := Self;
    SynEdit.Align := alClient;
    SynEdit.Font.Name := 'Courier New';
    SynEdit.Font.Size := 10;
    SynEdit.Color := clWhite;
    
    // 옵션 설정
    SynEdit.Options := [eoAutoIndent, eoDragDropEditing, eoEnhanceEndKey,
                       eoScrollPastEol, eoShowScrollHint, eoSmartTabs,
                       eoTabsToSpaces, eoSmartTabDelete, eoGroupUndo];
    
    // 문법 하이라이터 설정 (Pascal)
    Highlighter := TSynPasSyn.Create(nil);
    SynEdit.Highlighter := Highlighter;
    
    // 거터 설정 (라인 번호)
    SynEdit.Gutter.ShowLineNumbers := True;
    SynEdit.Gutter.AutoSize := True;
    
    // 텍스트 로드
    SynEdit.Lines.LoadFromFile('script.pas');
    
    // 이벤트 핸들러
    SynEdit.OnChange := procedure(Sender: TObject)
    begin
      // 텍스트 변경 시 처리
    end;
  finally
    Highlighter.Free;
    SynEdit.Free;
  end;
end;
```

---

## TMS - UI 컴포넌트

### 개요
**TMS (TMS Software)**는 다양한 UI 컴포넌트를 제공하는 라이브러리입니다.

### 주요 컴포넌트
- **AdvGrid**: 고급 그리드 컴포넌트
- **AdvCombo**: 고급 콤보박스
- **AdvDateTimePicker**: 날짜/시간 선택기
- **MoneyEdit**: 금액 입력 컴포넌트
- **PictureContainer**: 이미지 컨테이너

### 주요 파일
- **`AdvGrid.pas`**: 그리드 컴포넌트
- **`AdvCombo.pas`**: 콤보박스 컴포넌트
- **`AdvDateTimePicker.pas`**: 날짜/시간 선택기
- **`MoneyEdit.pas`**: 금액 입력 컴포넌트

---

## DevExpress - UI 프레임워크

### 개요
**DevExpress**는 강력한 UI 컴포넌트 라이브러리로, 현대적인 사용자 인터페이스를 구축하는 데 사용됩니다.

### 주요 컴포넌트
- **ExpressQuantumGrid**: 고급 그리드 컴포넌트
- **ExpressBars**: 메뉴바, 툴바
- **ExpressEditors**: 다양한 에디터 컴포넌트
- **ExpressSkins**: 스킨 시스템
- **ExpressDocking**: 도킹 시스템
- **ExpressPrinting**: 인쇄 시스템
- **ExpressScheduler**: 스케줄러
- **ExpressPivotGrid**: 피벗 그리드

### 주요 디렉토리
- **ExpressQuantumGrid/**: 그리드 컴포넌트
- **ExpressBars/**: 메뉴바/툴바
- **ExpressEditors Library/**: 에디터 컴포넌트
- **ExpressSkins Library/**: 스킨 파일
- **ExpressDocking Library/**: 도킹 시스템
- **ExpressPrinting System/**: 인쇄 시스템

### 사용 예시
COM_LV2의 SMC 프레임워크에서 DevExpress 컴포넌트를 광범위하게 사용합니다:
- 스킨 시스템: `dxSkinsManager`
- 그리드: `TcxGrid`
- 에디터: `TcxTextEdit`, `TcxDateEdit` 등
- 도킹: `TdxDockPanel`

---

## SWBC_DLL - 암호화 라이브러리

### 개요
**SWBC_DLL**은 LEA128-SWBC 암호화 알고리즘을 구현한 DLL로, 삼성 EHR 시스템의 데이터 암호화에 사용됩니다.

### 주요 파일
- **`samsung_ehr_swbc.cpp`**: 암호화/복호화 함수 구현
- **`SWBC_Samsung_EHR.h`**: 암호화 키 정의
- **`lea128-swbc-layer1-ctr.*`**: LEA128-SWBC 레이어1 CTR 모드 구현

### 주요 함수

#### `encrypt`
데이터를 암호화합니다.

```cpp
extern "C" int __declspec(dllexport) __stdcall encrypt(
    const char* iv,           // 초기화 벡터 (16 bytes)
    const int iv_len,         // IV 길이
    const char *in,           // 입력 데이터
    const int in_len,         // 입력 데이터 길이
    char *out,                // 출력 버퍼 (IV + 암호화된 데이터)
    int *out_len              // 출력 길이
);
```

**동작 방식:**
1. IV 길이 검증 (16 bytes)
2. LEA128-SWBC 컨텍스트 초기화
3. 암호화 키 로드 (SWBC_Samsung_EHR)
4. CTR 모드로 암호화 수행
5. 출력: IV (16 bytes) + 암호화된 데이터

#### `decrypt`
데이터를 복호화합니다.

```cpp
extern "C" int __declspec(dllexport) __stdcall decrypt(
    const char *in,           // 입력 데이터 (IV + 암호화된 데이터)
    const int in_len,         // 입력 데이터 길이
    char *out,                // 출력 버퍼 (복호화된 데이터)
    int *out_len              // 출력 길이
);
```

**동작 방식:**
1. 입력 데이터에서 IV 추출 (처음 16 bytes)
2. LEA128-SWBC 컨텍스트 초기화
3. 암호화 키 로드 (SWBC_Samsung_EHR)
4. CTR 모드로 복호화 수행
5. 출력: 복호화된 데이터

### 암호화 키
`SWBC_Samsung_EHR.h`에 정의된 암호화 키:
- 16x256 바이트의 키 테이블
- 16 바이트의 태그

### 사용 예시

```pascal
// Delphi에서 DLL 사용
type
  TEncryptFunc = function(const iv: PAnsiChar; iv_len: Integer;
                         const in_data: PAnsiChar; in_len: Integer;
                         out_data: PAnsiChar; out var out_len: Integer): Integer; stdcall;
  TDecryptFunc = function(const in_data: PAnsiChar; in_len: Integer;
                         out_data: PAnsiChar; out var out_len: Integer): Integer; stdcall;

var
  DLLHandle: THandle;
  EncryptFunc: TEncryptFunc;
  DecryptFunc: TDecryptFunc;
  IV: array[0..15] of Byte;
  PlainText, EncryptedData, DecryptedData: TBytes;
  OutLen: Integer;
begin
  // DLL 로드
  DLLHandle := LoadLibrary('SWBC_DLL.dll');
  if DLLHandle = 0 then
    raise Exception.Create('DLL 로드 실패');
    
  try
    // 함수 주소 가져오기
    EncryptFunc := GetProcAddress(DLLHandle, 'encrypt');
    DecryptFunc := GetProcAddress(DLLHandle, 'decrypt');
    
    if not Assigned(EncryptFunc) or not Assigned(DecryptFunc) then
      raise Exception.Create('함수 주소 가져오기 실패');
    
    // IV 생성 (랜덤)
    Randomize;
    for I := 0 to 15 do
      IV[I] := Random(256);
    
    // 평문 데이터
    PlainText := TEncoding.UTF8.GetBytes('암호화할 데이터');
    
    // 암호화
    SetLength(EncryptedData, Length(PlainText) + 16);
    OutLen := Length(EncryptedData);
    if EncryptFunc(@IV[0], 16, @PlainText[0], Length(PlainText),
                   @EncryptedData[0], OutLen) <> 0 then
      raise Exception.Create('암호화 실패');
    SetLength(EncryptedData, OutLen);
    
    // 복호화
    SetLength(DecryptedData, Length(EncryptedData) - 16);
    OutLen := Length(DecryptedData);
    if DecryptFunc(@EncryptedData[0], Length(EncryptedData),
                   @DecryptedData[0], OutLen) <> 0 then
      raise Exception.Create('복호화 실패');
    SetLength(DecryptedData, OutLen);
    
    // 결과 확인
    ShowMessage(TEncoding.UTF8.GetString(DecryptedData));
  finally
    FreeLibrary(DLLHandle);
  end;
end;
```

---

## 핵심 기능 상세 분석

### 1. 메모리 데이터셋 관리 (kbmMemTable)

**역할:**
- REST API로부터 받은 데이터를 메모리에 저장
- 빠른 검색, 정렬, 필터링
- 다른 데이터셋과의 데이터 교환

**사용 시나리오:**
```pascal
// COM_LV2의 SEHR.Dataset.pas에서 사용
var
  DataSet: TSHISDataSet;
begin
  DataSet := TSHISDataSet.Create(nil);
  try
    // REST API에서 데이터 로드
    DataSet.Open;
    
    // 메모리 테이블로 변환
    MemTable.LoadFromDataSet(DataSet, [mtcpoStructure, mtcpoProperties]);
    
    // 빠른 검색 및 정렬
    MemTable.SortOn('PatientName', []);
    if MemTable.Locate('PatientID', 123, []) then
      // 데이터 처리
  finally
    DataSet.Free;
  end;
end;
```

### 2. 실시간 통신 (MQTT)

**역할:**
- 서버로부터 실시간 알림 수신
- 다른 클라이언트와의 실시간 데이터 동기화
- 이벤트 기반 아키텍처 지원

**사용 시나리오:**
```pascal
// 환자 정보 업데이트 알림 수신
MQTTClient.OnPublishReceived := procedure(Sender: TObject; APacketID: Word; ATopic: string; APayload: TBytes)
var
  JSON: TJSONObject;
  PatientID: Integer;
begin
  if ATopic = 'patient/update' then
  begin
    JSON := TJSONObject.ParseJSONValue(TEncoding.UTF8.GetString(APayload)) as TJSONObject;
    try
      PatientID := JSON.GetValue<Integer>('patientId');
      // 환자 정보 새로고침
      RefreshPatientInfo(PatientID);
    finally
      JSON.Free;
    end;
  end;
end;
```

### 3. 코드 편집 (SynEdit)

**역할:**
- 스크립트 편집기
- 로그 뷰어
- 설정 파일 편집기

**사용 시나리오:**
```pascal
// SQL 쿼리 편집기
var
  SQLEditor: TSynEdit;
  SQLHighlighter: TSynSQLSyn;
begin
  SQLEditor := TSynEdit.Create(nil);
  SQLHighlighter := TSynSQLSyn.Create(nil);
  try
    SQLEditor.Highlighter := SQLHighlighter;
    SQLEditor.Lines.Text := 'SELECT * FROM Patients WHERE Age > 30';
    // 사용자가 SQL 편집
    // 실행 버튼 클릭 시
    ExecuteQuery(SQLEditor.Lines.Text);
  finally
    SQLHighlighter.Free;
    SQLEditor.Free;
  end;
end;
```

### 4. 데이터 암호화 (SWBC_DLL)

**역할:**
- 민감한 데이터 암호화
- 네트워크 전송 시 데이터 보호
- 저장 시 데이터 보호

**사용 시나리오:**
```pascal
// 환자 정보 암호화 후 전송
var
  PatientData: TBytes;
  EncryptedData: TBytes;
begin
  // JSON 데이터를 바이트로 변환
  PatientData := TEncoding.UTF8.GetBytes(PatientJSON);
  
  // 암호화
  EncryptedData := EncryptData(PatientData);
  
  // 서버로 전송
  SendToServer(EncryptedData);
end;
```

---

## 주요 클래스 및 함수

### kbmMemTable

#### `TkbmMemTable`
```pascal
// 생성자
constructor Create(AOwner: TComponent); override;

// 데이터 로드/저장
procedure LoadFromStream(Stream: TStream; Format: TkbmMemTableFormat = mtfCSV);
procedure SaveToStream(Stream: TStream; Format: TkbmMemTableFormat = mtfCSV);
procedure LoadFromFile(const FileName: string; Format: TkbmMemTableFormat = mtfCSV);
procedure SaveToFile(const FileName: string; Format: TkbmMemTableFormat = mtfCSV);

// 검색
function Locate(const KeyFields: string; const KeyValues: Variant; Options: TLocateOptions): Boolean;
function Lookup(const KeyFields: string; const KeyValues: Variant; const ResultFields: string): Variant;

// 정렬
procedure Sort(Options: TkbmMemTableCompareOptions; const SortFields: string);
procedure SortOn(const SortFields: string; Options: TkbmMemTableCompareOptions = []);
```

### MQTT

#### `TTMSMQTTClient`
```pascal
// 연결 관리
procedure Connect(ACleanSession: Boolean = True);
procedure Disconnect;
function IsConnected: Boolean;

// 메시지 발행
function Publish(ATopic: string; APayload: string; AQos: TTMSMQTTQoS = qosAtMostOnce; ARetain: Boolean = False): Word;

// 구독
function Subscribe(ATopic: string; ATopicQosLevel: TTMSMQTTQoS = qosAtMostOnce): Word;
function Unsubscribe(ATopic: string): Word;

// Keep-Alive
procedure Ping;
```

### SynEdit

#### `TSynEdit`
```pascal
// 텍스트 조작
procedure Clear;
procedure SelectAll;
procedure CopyToClipboard;
procedure PasteFromClipboard;

// 검색
function SearchReplace(const ASearch, AReplace: string; AOptions: TSynSearchOptions): Integer;
function SearchNext: Boolean;

// 커서 이동
procedure GotoLineAndCenter(ALine: Integer);
procedure GotoXY(X, Y: Integer);
```

### SWBC_DLL

#### 암호화 함수
```cpp
// 암호화
int encrypt(const char* iv, const int iv_len,
            const char *in, const int in_len,
            char *out, int *out_len);

// 복호화
int decrypt(const char *in, const int in_len,
            char *out, int *out_len);
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
COM_LV3 (비즈니스 로직)
    ↓
MAIN (애플리케이션)
```

**COM_LV1의 역할:**
- COM_LV0 위에 구축되는 써드파티 라이브러리 레이어
- COM_LV2가 사용하는 고급 컴포넌트 제공
- 데이터 관리, 통신, UI, 암호화 기능 제공

---

## 요약

COM_LV1 프로젝트는 Nexmed EHR 시스템의 써드파티 라이브러리 레이어로, 다음과 같은 핵심 기능을 제공합니다:

1. **kbmMemTable**: 인메모리 데이터셋 관리
2. **MQTT**: 실시간 메시징 통신
3. **SynEdit**: 고급 텍스트 편집
4. **TMS/DevExpress**: 현대적인 UI 컴포넌트
5. **SWBC_DLL**: 데이터 암호화

이러한 라이브러리들은 COM_LV2 프레임워크 레이어에서 활용되어 강력하고 현대적인 EHR 시스템을 구축하는 기반을 제공합니다.

