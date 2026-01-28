# COM_LV0 프로젝트 상세 분석서

## 목차
1. [프로젝트 개요](#프로젝트-개요)
2. [프로젝트 구조](#프로젝트-구조)
3. [CommonPack - 공통 유틸리티](#commonpack---공통-유틸리티)
4. [CPortLib - 시리얼 포트 통신](#cportlib---시리얼-포트-통신)
5. [BarcodePack - 바코드 처리](#barcodepack---바코드-처리)
6. [ExternalPack - 외부 라이브러리](#externalpack---외부-라이브러리)
7. [ZipMaster - ZIP 파일 처리](#zipmaster---zip-파일-처리)
8. [Rexpert - 리포트 생성](#rexpert---리포트-생성)
9. [RemotePrintServer - 원격 프린트 서버](#remoteprintserver---원격-프린트-서버)
10. [핵심 기능 상세 분석](#핵심-기능-상세-분석)
11. [주요 클래스 및 함수](#주요-클래스-및-함수)

---

## 프로젝트 개요

**COM_LV0**는 Nexmed EHR 시스템의 가장 기본적인 인프라 레이어로, 다른 모든 레이어(COM_LV1, COM_LV2, COM_LV3)가 의존하는 기초 라이브러리입니다. 이 프로젝트는 다음과 같은 주요 역할을 담당합니다:

- **공통 유틸리티**: 문자열 처리, 날짜 변환, 암호화, 인증서 처리
- **하드웨어 통신**: 시리얼 포트, 바코드 스캐너
- **파일 처리**: ZIP 압축/해제, XML 처리
- **외부 라이브러리 연동**: OCX/DLL 타입 정의
- **프린트 서버**: 원격 프린트 요청 처리

---

## 프로젝트 구조

```
COM_LV0/
├── CommonPack/              # 공통 유틸리티 패키지
│   ├── CommonFunc.pas       # 공통 함수 (문자열, 날짜, 암호화 등)
│   ├── DCPcrypt2.pas        # 암호화 기본 클래스
│   ├── DCPsha256.pas        # SHA256 해시
│   ├── DCPbase64.pas        # Base64 인코딩/디코딩
│   ├── XMLZip.pas           # ZIP 파일 처리
│   └── PopupTree.pas        # 팝업 트리 메뉴
├── CPortLib/                # 시리얼 포트 통신
│   ├── CPort.pas            # 시리얼 포트 컴포넌트
│   └── BarCodeScaner.pas    # 바코드 스캐너
├── BarcodePack/             # 바코드 생성
│   └── dbc2dun.pas          # 2D 바코드 생성
├── ExternalPack/            # 외부 라이브러리 타입 정의
│   ├── AXCROSSCERTLib_TLB.pas
│   ├── EDOfficeLib_TLB.pas
│   ├── HiraDiCom_TLB.pas
│   └── ... (다양한 OCX/DLL 타입 정의)
├── ZipMaster/               # ZIP 파일 처리
│   └── ZipMstr.pas          # ZIP 압축/해제
├── Rexpert/                 # 리포트 생성
│   └── Clipsoft_TLB.pas    # 리포트 뷰어 타입 정의
└── RemotePrintServer/       # 원격 프린트 서버
    ├── fSMCServer.pas       # 프린트 서버 메인
    └── TextPrintPas.pas     # 텍스트 프린터 처리
```

---

## CommonPack - 공통 유틸리티

### 1. CommonFunc.pas - 공통 함수 모음

**역할**: 애플리케이션 전반에서 사용되는 공통 유틸리티 함수들을 제공합니다.

#### 문자열 처리 함수

##### StringParseCount / StringParseString
```pascal
function StringParseCount(const s, delimiters: AnsiString): Word;
function StringParseString(const s, delimiters: String; Num: Word): String;
```
- **역할**: 구분자로 문자열을 분리하여 개수 또는 특정 위치의 문자열을 반환
- **예제**:
  ```pascal
  // "A,B,C"에서 ','로 분리하면 3개
  Count := StringParseCount('A,B,C', ',');
  // 2번째 항목: "B"
  Str := StringParseString('A,B,C', ',', 2);
  ```

##### NVL
```pascal
function NVL(Str, Replace: String): String;
```
- **역할**: NULL 값 처리 (Oracle의 NVL 함수와 동일)
- **예제**:
  ```pascal
  Result := NVL(EmptyStr, '기본값'); // '기본값' 반환
  ```

##### LPAD / RPAD
```pascal
function LPAD(Str: String; Len: Integer; FillChar: String): String;
function RPAD(Str: String; Len: Integer; FillChar: String): String;
```
- **역할**: 문자열을 지정된 길이로 채우기 (왼쪽/오른쪽)
- **예제**:
  ```pascal
  LPAD('123', 5, '0') // '00123'
  RPAD('123', 5, '0') // '12300'
  ```

##### SMCCopy / SMCLength / SMCPos
```pascal
function SMCCopy(const s: String; index, count: Integer): AnsiString;
function SMCLength(const s: String): Integer;
function SMCPos(const SubStr, str: String): Integer;
```
- **역할**: AnsiString 기반의 문자열 처리 함수
- **용도**: 레거시 코드 호환성

##### RemoveLFCR / DeleteStr
```pascal
function RemoveLFCR(Str: String): String;
function DeleteStr(OrigStr, DelStr: String): String;
```
- **역할**: 
  - `RemoveLFCR`: 줄바꿈 문자 제거
  - `DeleteStr`: 특정 문자열 삭제

#### 날짜/시간 처리 함수

##### ConvertSDate / ConvertDDate / ConvertLDate
```pascal
function ConvertSDate(DTstr: String): String;  // YYYYMMDD -> YYYY-MM-DD
function ConvertDDate(DTstr: AnsiString): AnsiString;  // YYYYMMDDHH24MI -> YYYY-MM-DD HH24:MI
function ConvertLDate(DTstr: String): String;  // YYYYMMDDHH24MISS -> YYYY-MM-DD HH24:MI:SS
```
- **역할**: 날짜 형식 변환 (구분자 추가)

##### InvertSDate / InvertDDate / InvertLDate
```pascal
function InvertSDate(DTstr: AnsiString): AnsiString;  // YYYY-MM-DD -> YYYYMMDD
function InvertDDate(DTstr: AnsiString): AnsiString;  // YYYY-MM-DD HH24:MI -> YYYYMMDDHH24MI
function InvertLDate(DTstr: AnsiString): AnsiString;  // YYYY-MM-DD HH24:MI:SS -> YYYYMMDDHH24MISS
```
- **역할**: 날짜 형식 변환 (구분자 제거)

##### ConvertDTime / InvertLTime
```pascal
function ConvertDTime(DTstr: String): String;  // HH24MI -> HH24:MI
function InvertLTime(DTstr: String): String;  // HH24:MI -> HH24MI
```
- **역할**: 시간 형식 변환

##### CheckDateTimeFormat
```pascal
function CheckDateTimeFormat(sDateStr: String): Boolean;
```
- **역할**: 날짜/시간 형식 유효성 검증
- **지원 형식**:
  - `YYYYMMDD`
  - `YYYY-MM-DD`
  - `YYYYMMDDHH24MISS`
  - `YYYY-MM-DD HH24:MI:SS`

#### 숫자 처리 함수

##### RRoundPos / RTruncPos / RCeilPos
```pascal
function RRoundPos(Amount: Real; Pos: Integer): Real;
function RTruncPos(Amount: Real; Pos: Integer): Real;
function RCeilPos(Amount: Real; Pos: Integer): Real;
```
- **역할**: 실수를 지정된 소수점 자리수로 반올림/버림/올림
- **예제**:
  ```pascal
  RRoundPos(123.456, 1) // 123.5
  RTruncPos(123.456, 1) // 123.4
  RCeilPos(123.451, 1)  // 123.5
  ```

##### IRoundPos / ITruncPos / ICeilPos
```pascal
function IRoundPos(Amount: Real; Pos: Integer): Integer;
function ITruncPos(Amount: Real; Pos: Integer): Integer;
function ICeilPos(Amount: Real; Pos: Integer): Integer;
```
- **역할**: 실수를 정수로 변환 (지정된 자리수 기준)

##### isNumeric / IsNumericEx
```pascal
function isNumeric(Str: String): Boolean;
function IsNumericEx(const Value: string): Boolean;
```
- **역할**: 숫자 문자열 검증
- **차이점**: `IsNumericEx`는 정규식을 사용하여 실수도 검증

#### 입력 검증 함수

##### InputNum / InputInt / InputReal
```pascal
function InputNum(in_Key: Char): Char;   // 숫자만 입력
function InputInt(in_Key: Char): Char;  // 정수만 입력
function InputReal(in_Key: Char): Char; // 실수만 입력
```
- **역할**: 키 입력 시 숫자만 허용
- **사용법**: `OnKeyPress` 이벤트에서 사용
- **예제**:
  ```pascal
  procedure TForm.Edit1KeyPress(Sender: TObject; var Key: Char);
  begin
    Key := InputNum(Key);
  end;
  ```

##### InputAlpha / InputAlphaNum
```pascal
function InputAlpha(in_c_key: Char): Char;      // 영문만
function InputAlphaNum(in_c_key: Char): Char;  // 영문+숫자
```
- **역할**: 영문 또는 영문+숫자만 입력 허용

#### 주민등록번호 검증

##### CitizenChk
```pascal
function CitizenChk(ssno1: AnsiString; ssno2: AnsiString): Boolean;
```
- **역할**: 주민등록번호 유효성 검증
- **검증 로직**:
  1. 생년월일 유효성 검증
  2. 체크섬 계산 (각 자리수에 가중치 곱셈)
  3. 마지막 자리수 검증

**검증 공식**:
```
체크섬 = (자리1*2 + 자리2*3 + ... + 자리12*5) mod 11
검증값 = 11 - 체크섬 (단, 10이면 0, 11이면 1)
```

##### CheckResnoFormat
```pascal
function CheckResnoFormat(resno: String): Boolean;
```
- **역할**: 주민등록번호 형식 검증 (13자리, 하이픈 포함/미포함 모두 지원)

#### 나이 계산 함수

##### CalcAge
```pascal
function CalcAge(sFromDate: TDate; sBirthDate: AnsiString): AnsiString;
```
- **역할**: 생년월일로부터 나이 계산 (한글 형식)
- **반환 형식**:
  - 3세 이상: "XX세"
  - 1~2세: "XX개월"
  - 1개월 미만: "XX일"

##### GetAge
```pascal
function GetAge(AFromDate: TDate; AToDate: TDate): Integer;
```
- **역할**: 두 날짜 사이의 나이 계산 (정수)

##### Calc_Age
```pascal
function Calc_Age(sBirthDate: AnsiString; sAgeShow: String = '00'; 
  sFromDate: TDate = 0): AnsiString;
```
- **역할**: 나이 계산 (더 상세한 옵션)

#### 암호화/복호화 함수

##### EncryptPassword / DecryptPassword
```pascal
function EncryptPassword(Password: String): String;
function DecryptPassword(Password: AnsiString): AnsiString;
```
- **역할**: 간단한 패스워드 암호화/복호화
- **알고리즘**: 각 문자에 위치 기반 오프셋 적용

**암호화 로직**:
```pascal
암호화: chr(Ord(Password[i]) - Length(Password) + i - 1)
복호화: chr(Ord(Password[i]) + Length(Password) - i + 1)
```

##### HashSHA256
```pascal
function HashSHA256(Text: String): String;
```
- **역할**: SHA256 해시 생성
- **사용**: `DCPsha256` 유닛 사용

#### 인증서 처리 함수

##### CertAPI / CertFree
```pascal
function CertAPI: Integer;
procedure CertFree;
```
- **역할**: 인증서 API DLL 로드/해제
- **DLL**: `EMRDLL.DLL` 또는 유사한 인증서 라이브러리

**주요 인증서 함수**:
- `CERT_SetMASInfo`: 인증서 서버 정보 설정
- `CERT_CheckMASRunning`: 인증서 서버 실행 확인
- `CERT_UploadCert`: 인증서 업로드
- `CERT_DownloadCert`: 인증서 다운로드
- `CERT_SignEMR`: EMR 서명
- `CERT_VerifyEMR`: EMR 검증
- `CERT_EncryptEMRWithKey`: EMR 암호화
- `CERT_DecryptEMRWithKey`: EMR 복호화
- `CERT_CheckCertExist`: 인증서 존재 확인
- `CERT_GetCertValidity`: 인증서 유효기간 조회
- `CERT_LoadCertCard`: 인증서 카드에서 로드
- `CERT_CopyToCard`: 인증서를 카드로 복사

#### 전화기 연동 함수

##### MctmLoad / MctmFree
```pascal
function MctmLoad: Integer;
procedure MctmFree;
```
- **역할**: 전화기 연동 DLL 로드/해제
- **DLL**: `MCTM.DLL` (전화기 제어 라이브러리)

**주요 전화기 함수**:
- `MCTM_Init`: 전화기 초기화
- `MCTM_LogON`: 로그인
- `MCTM_LogOFF`: 로그아웃
- `MCTM_GetExtStatus`: 내선 상태 조회
- `MCTM_Change_Agent_Status`: 상담원 상태 변경
- `MCTM_GetDept`: 부서 목록 조회
- `MCTM_GetStatus`: 사용자 상태 조회
- `MCTM_TransferCall`: 통화 전환
- `MCTM_MakeCall`: 통화 발신
- `MCTM_GetCallEventInfo`: 통화 이벤트 정보 조회

#### 스마트카드 처리 함수

##### SmartCardLoad / SmartCardFree
```pascal
function SmartCardLoad: Integer;
procedure SmartCardFree;
```
- **역할**: 스마트카드 DLL 로드/해제
- **함수**: `GetNID` - 주민등록번호 읽기

#### 압축/해제 함수

##### CompressString / DecompressString
```pascal
procedure CompressString(const XML: AnsiString; OutStream: TStream);
procedure DecompressString(InStream: TStream; var XML: AnsiString);
```
- **역할**: ZLib을 사용한 문자열 압축/해제
- **용도**: XML 데이터 압축 전송

#### ZIP 파일 처리 클래스

##### TZip
```pascal
TZip = class
  procedure CreateZip(Stream: TStream);
  procedure OpenZip(Stream: TStream);
  function AddStream(Stream: TStream): TZipEntriesEntry;
  procedure ExtractToStream(Item: TZipEntriesEntry; Stream: TStream);
  property Entries: TZipEntries read FZipHeader;
end;
```
- **역할**: ZIP 파일 생성 및 압축 해제
- **주요 메서드**:
  - `CreateZip`: 새 ZIP 파일 생성
  - `OpenZip`: 기존 ZIP 파일 열기
  - `AddStream`: 스트림을 ZIP에 추가
  - `ExtractToStream`: ZIP 항목을 스트림으로 추출

##### TZipEntriesEntry
```pascal
TZipEntriesEntry = class(TCollectionItem)
  property SizeUncompressed: Cardinal;
  property SizeCompressed: Cardinal;
  property LocalOffset: Cardinal;
end;
```
- **역할**: ZIP 파일 내 개별 항목 정보

#### UI 유틸리티 함수

##### FindNode
```pascal
function FindNode(FindText: string; TreeView: TTreeView): TTreeNode;
```
- **역할**: 트리뷰에서 특정 텍스트를 가진 노드 찾기

##### GetNodeName
```pascal
function GetNodeName(TreeNode: TTreeNode): String;
```
- **역할**: 트리 노드의 이름 추출 (공백 이전 부분)

##### GetTextWidth / MakeSpace
```pascal
function GetTextWidth(Handle: HFont; const str: string): Integer;
function MakeSpace(Handle: HFont; Width: Integer): string;
```
- **역할**: 
  - `GetTextWidth`: 폰트로 문자열 너비 계산
  - `MakeSpace`: 지정된 너비만큼 공백 문자열 생성

##### AlignedText
```pascal
function AlignedText(Handle: HFont; sText: String; iWidth: Integer; 
  taType: TAlignment): String;
```
- **역할**: 정렬된 텍스트 생성 (왼쪽/가운데/오른쪽)

##### FormClear / PanelClear2
```pascal
procedure FormClear(Form: TComponent);
procedure PanelClear2(Panel: TPanel);
```
- **역할**: 폼/패널의 모든 컨트롤 초기화

##### PanelColor / PanelEnable
```pascal
procedure PanelColor(Panel: TPanel; cColor: TColor);
procedure PanelEnable(Panel: TPanel; bTag: Boolean);
```
- **역할**: 패널의 컨트롤 색상/활성화 상태 일괄 변경

##### NextCtrl / PrevCtrl / MoveFocus
```pascal
procedure NextCtrl(in_o_winctrl: TWinControl);
procedure PrevCtrl(in_o_winctrl: TWinControl);
procedure MoveFocus(in_o_tgtctrl: TWinControl; 
  in_o_curctrl: TWinControl = nil);
```
- **역할**: 
  - `NextCtrl`: 다음 컨트롤로 포커스 이동
  - `PrevCtrl`: 이전 컨트롤로 포커스 이동
  - `MoveFocus`: 특정 컨트롤로 포커스 이동

#### 시스템 정보 함수

##### GetOSName
```pascal
function GetOSName: string;
```
- **역할**: 운영체제 이름 반환

##### GetLocalTermid
```pascal
function GetLocalTermid: AnsiString;
```
- **역할**: 로컬 터미널 ID 반환

##### IsFileInUse
```pascal
function IsFileInUse(fName: string): Boolean;
```
- **역할**: 파일 사용 중 여부 확인

##### FileCopy
```pascal
procedure FileCopy(SrcFile, DstFile: String);
```
- **역할**: 파일 복사

#### 기타 유틸리티 함수

##### GetCommaValue
```pascal
function GetCommaValue(Value: string): string;
```
- **역할**: 숫자에 천단위 구분자(쉼표) 추가

##### NumDataToEdit
```pascal
function NumDataToEdit(eData: String): String;
```
- **역할**: 숫자 데이터를 편집 가능한 형식으로 변환 (3자리마다 쉼표)

##### DateToKorStr
```pascal
function DateToKorStr(in_d_date: TDateTime): String;
```
- **역할**: 날짜를 한글 형식으로 변환 (예: "2024년 01월 15일")

##### OraDecode
```pascal
function OraDecode(const in_ca_args: array of const): Variant;
```
- **역할**: Oracle의 DECODE 함수와 유사한 기능

##### NullIf
```pascal
function NullIf(in_s_expr1, in_s_expr2: String): String;
```
- **역할**: 두 값이 같으면 빈 문자열, 다르면 첫 번째 값 반환

##### Get_Search_Name
```pascal
function Get_Search_Name(i_Key: String): String;
```
- **역할**: 검색 키를 한글/영문 변환

##### Get_QTY
```pascal
function Get_QTY(v_qty: String): String;
```
- **역할**: 수량 문자열 처리

##### IsNotPrinterReady
```pascal
function IsNotPrinterReady: Boolean;
```
- **역할**: 프린터 준비 상태 확인

##### SetHangeulMode
```pascal
procedure SetHangeulMode(SetHangeul: Boolean; Handle: THandle);
```
- **역할**: 한글 입력 모드 설정

##### NumberKeyPress
```pascal
procedure NumberKeyPress(Sender: TObject; var Key: Char; Int_YN: Boolean);
```
- **역할**: 숫자 키 입력 처리 (이벤트 핸들러)

##### kcMonthDays / kclsLeapYear
```pascal
function kcMonthDays(nMonth, nYear: Integer): Integer;
function kclsLeapYear(nYear: Integer): Boolean;
```
- **역할**: 
  - `kcMonthDays`: 해당 월의 일수 계산
  - `kclsLeapYear`: 윤년 여부 확인

### 2. DCPcrypt2.pas - 암호화 기본 클래스

**역할**: 암호화/복호화 및 해시 알고리즘의 기본 클래스를 제공합니다.

#### 주요 클래스

##### TDCP_hash
```pascal
TDCP_hash = class(TComponent)
  procedure Init;
  procedure Update(const Buffer; Size: longword);
  procedure Final(var Digest);
  procedure Burn;
end;
```
- **역할**: 해시 알고리즘의 기본 클래스
- **주요 메서드**:
  - `Init`: 해시 초기화
  - `Update`: 데이터 추가
  - `Final`: 최종 해시값 생성
  - `Burn`: 내부 데이터 삭제

##### TDCP_cipher
```pascal
TDCP_cipher = class(TComponent)
  procedure Init(const Key; Size: longword; InitVector: pointer);
  procedure Encrypt(const Indata; var Outdata; Size: longword);
  procedure Decrypt(const Indata; var Outdata; Size: longword);
end;
```
- **역할**: 암호화 알고리즘의 기본 클래스
- **주요 메서드**:
  - `Init`: 키 설정
  - `Encrypt`: 암호화
  - `Decrypt`: 복호화

##### TDCP_blockcipher
```pascal
TDCP_blockcipher = class(TDCP_cipher)
  property CipherMode: TDCP_ciphermode;
  procedure EncryptCBC(const Indata; var Outdata; Size: longword);
  procedure DecryptCBC(const Indata; var Outdata; Size: longword);
end;
```
- **역할**: 블록 암호화 알고리즘
- **암호화 모드**:
  - `cmCBC`: Cipher Block Chaining
  - `cmCFB8bit`: Cipher Feedback (8-bit)
  - `cmCFBblock`: Cipher Feedback (block)
  - `cmOFB`: Output Feedback
  - `cmCTR`: Counter

### 3. DCPsha256.pas - SHA256 해시

**역할**: SHA256 해시 알고리즘 구현

#### 주요 클래스

##### TDCP_sha256
```pascal
TDCP_sha256 = class(TDCP_hash)
  class function GetHashSize: integer; override;  // 256 bits
  procedure Init; override;
  procedure Update(const Buffer; Size: longword); override;
  procedure Final(var Digest); override;
end;
```
- **역할**: SHA256 해시 생성
- **해시 크기**: 256비트 (32바이트)

**사용 예제**:
```pascal
var
  Hash: TDCP_sha256;
  Digest: array[0..31] of Byte;
begin
  Hash := TDCP_sha256.Create(nil);
  try
    Hash.Init;
    Hash.UpdateStr('Hello World');
    Hash.Final(Digest);
    // Digest에 32바이트 해시값 저장
  finally
    Hash.Free;
  end;
end;
```

### 4. DCPbase64.pas - Base64 인코딩

**역할**: Base64 인코딩/디코딩

#### 주요 함수

##### Base64EncodeStr / Base64DecodeStr
```pascal
function Base64EncodeStr(const Value: AnsiString): AnsiString;
function Base64DecodeStr(const Value: AnsiString): AnsiString;
```
- **역할**: 문자열을 Base64로 인코딩/디코딩

##### Base64Encode / Base64Decode
```pascal
function Base64Encode(pInput: pointer; pOutput: pointer; Size: longint): longint;
function Base64Decode(pInput: pointer; pOutput: pointer; Size: longint): longint;
```
- **역할**: 바이너리 데이터를 Base64로 인코딩/디코딩

### 5. XMLZip.pas - ZIP 파일 처리

**역할**: ZIP 파일 형식의 직접 처리 (ZLib 기반)

#### 주요 클래스

##### TZip
```pascal
TZip = class
  procedure CreateZip(Stream: TStream);
  procedure OpenZip(Stream: TStream);
  function AddStream(Stream: TStream): TZipEntriesEntry;
  procedure ExtractToStream(Item: TZipEntriesEntry; Stream: TStream);
end;
```
- **역할**: ZIP 파일 생성 및 읽기
- **특징**: ZLib을 사용한 Deflate 압축

**ZIP 파일 구조**:
- `TLocalFile`: 로컬 파일 헤더
- `TCentralDirectoryFile`: 중앙 디렉토리 파일 헤더
- `TEndOfCentralDir`: 중앙 디렉토리 종료 레코드

### 6. PopupTree.pas - 팝업 트리 메뉴

**역할**: 계층 구조의 팝업 메뉴를 제공합니다.

#### 주요 클래스

##### TSMCPopupTree
```pascal
TSMCPopupTree = class(TCustomPopup)
  function AddChild(Parent: TPopupNode; const Caption: String): TPopupNode;
  property OnClick: TPopupNodeClick;
end;
```
- **역할**: 팝업 트리 메뉴 컨테이너

##### TPopupNode
```pascal
TPopupNode = class(TSpeedButton)
  property Caption: String;
  function AddChild(const Caption: String): TPopupNode;
end;
```
- **역할**: 개별 메뉴 노드
- **특징**: 마우스 오버 시 하위 메뉴 자동 표시

---

## CPortLib - 시리얼 포트 통신

### 1. CPort.pas - 시리얼 포트 컴포넌트

**역할**: 시리얼 포트 통신을 위한 컴포넌트를 제공합니다.

#### 주요 클래스

##### TComPort
```pascal
TComPort = class(TCustomComPort)
  property Connected: Boolean;
  property Port: TPort;
  property BaudRate: TBaudRate;
  property DataBits: TDataBits;
  property StopBits: TStopBits;
  property Parity: TComParity;
  property FlowControl: TComFlowControl;
  property Timeouts: TComTimeouts;
  property Buffer: TComBuffer;
  
  procedure Open;
  procedure Close;
  function Write(const Buffer; Count: Integer): Integer;
  function Read(var Buffer; Count: Integer): Integer;
  function WriteStr(const Str: AnsiString): Integer;
  function ReadStr(var Str: AnsiString; Count: Integer): Integer;
end;
```

**주요 속성**:
- `Port`: 포트 이름 (예: "COM1", "COM2")
- `BaudRate`: 전송 속도 (br110 ~ br256000)
- `DataBits`: 데이터 비트 (dbFive ~ dbEight)
- `StopBits`: 정지 비트 (sbOneStopBit, sbOne5StopBits, sbTwoStopBits)
- `Parity`: 패리티 (prNone, prOdd, prEven, prMark, prSpace)
- `FlowControl`: 흐름 제어 (fcHardware, fcSoftware, fcNone, fcCustom)

**주요 메서드**:
- `Open`: 포트 열기
- `Close`: 포트 닫기
- `Write` / `WriteStr`: 데이터 전송
- `Read` / `ReadStr`: 데이터 수신
- `ClearBuffer`: 버퍼 지우기
- `SetDTR` / `SetRTS`: 제어 신호 설정

**이벤트**:
- `OnRxChar`: 데이터 수신 시 발생
- `OnRxBuf`: 버퍼에 데이터 수신 시 발생
- `OnTxEmpty`: 전송 버퍼 비었을 때 발생
- `OnError`: 통신 오류 시 발생
- `OnCTSChange` / `OnDSRChange` / `OnRLSDChange`: 제어 신호 변경 시 발생

##### TComFlowControl
```pascal
TComFlowControl = class(TPersistent)
  property FlowControl: TFlowControl;
  property OutCTSFlow: Boolean;
  property OutDSRFlow: Boolean;
  property ControlDTR: TDTRFlowControl;
  property ControlRTS: TRTSFlowControl;
  property XonXoffIn: Boolean;
  property XonXoffOut: Boolean;
end;
```
- **역할**: 흐름 제어 설정

##### TComTimeouts
```pascal
TComTimeouts = class(TPersistent)
  property ReadInterval: Integer;
  property ReadTotalMultiplier: Integer;
  property ReadTotalConstant: Integer;
  property WriteTotalMultiplier: Integer;
  property WriteTotalConstant: Integer;
end;
```
- **역할**: 읽기/쓰기 타임아웃 설정

##### TComBuffer
```pascal
TComBuffer = class(TPersistent)
  property InputSize: Integer;   // 기본: 1024
  property OutputSize: Integer;   // 기본: 1024
end;
```
- **역할**: 입력/출력 버퍼 크기 설정

##### TComDataPacket
```pascal
TComDataPacket = class(TComponent)
  property ComPort: TCustomComPort;
  property StartString: AnsiString;
  property StopString: AnsiString;
  property Size: Integer;
  property OnPacket: TComStrEvent;
end;
```
- **역할**: 패킷 단위 데이터 수신
- **작동 방식**: 시작 문자열과 종료 문자열 사이의 데이터를 패킷으로 인식

### 2. BarCodeScaner.pas - 바코드 스캐너

**역할**: 시리얼 포트로 연결된 바코드 스캐너를 제어합니다.

#### 주요 클래스

##### TBarCodeScaner
```pascal
TBarCodeScaner = class(TCustomComPort)
  property TermChar: Char;  // 종료 문자 (기본: #0)
  property OnBarCode: TBarCodeEvent;
end;
```
- **역할**: 바코드 스캐너 전용 컴포넌트
- **특징**: 종료 문자를 만나면 자동으로 `OnBarCode` 이벤트 발생

**작동 방식**:
1. 시리얼 포트에서 데이터 수신
2. `TermChar`를 만날 때까지 버퍼에 저장
3. `TermChar`를 만나면 `OnBarCode` 이벤트 발생

**사용 예제**:
```pascal
procedure TForm.BarCodeScaner1BarCode(var BarCode: String);
begin
  Edit1.Text := BarCode;
  // 바코드 처리
end;
```

---

## BarcodePack - 바코드 처리

### 1. dbc2dun.pas - 2D 바코드 생성

**역할**: 2D 바코드를 생성하는 컴포넌트를 제공합니다.

#### 주요 클래스

##### TBarcode2Dun
```pascal
TBarcode2Dun = class(TImage)
  property Caption: Ansistring;  // 바코드 데이터
  property BarcodeType: integer;  // 바코드 종류
  property Xunit: integer;  // X 단위
  property Yunit: integer;  // Y 단위
  property StartMode: integer;  // 시작 모드
  property SecurityLevel: integer;  // 보안 레벨
  property AspectRatio: single;  // 종횡비
  property Orientation: integer;  // 방향
  property ForeColor: TColorRef;  // 전경색
  property BackColor: TColorRef;  // 배경색
  
  procedure Refresh;  // 바코드 새로고침
  function GetBarcodeName(n: integer): Ansistring;  // 바코드 종류 이름
  function GetError(n: integer): Ansistring;  // 오류 메시지
end;
```
- **역할**: 2D 바코드 생성 및 표시
- **외부 DLL**: `dls2dun.dll` 사용

**주요 메서드**:
- `Refresh`: 바코드 이미지 생성
- `GetBarcodeName`: 바코드 종류 이름 조회
- `GetError`: 오류 메시지 조회
- `SetSerial`: 시리얼 번호 설정

**작동 방식**:
1. `Caption`에 바코드 데이터 설정
2. `Refresh` 호출
3. `Bar2Dmx` DLL 함수로 메타파일 생성
4. `Picture.Metafile`에 할당하여 표시

---

## ExternalPack - 외부 라이브러리

**역할**: 외부 OCX/DLL의 타입 라이브러리를 정의합니다.

### 주요 타입 라이브러리

#### 인증서 관련
- `AXCROSSCERTLib_TLB.pas`: 크로스 인증서 라이브러리
- `SKCertMgr.pas`: SK 인증서 관리자

#### 이미지/문서 처리
- `EDOfficeLib_TLB.pas`: 전자문서 오피스
- `HiraDiCom_TLB.pas`: 히라 DICOM 뷰어
- `Hira_Document_TLB.pas`: 히라 문서 뷰어
- `Hira_Dynamic_TLB.pas`: 히라 동적 뷰어
- `Hira_Ef_Eform_TLB.pas`: 히라 전자서식
- `Hira_Ef_Chuna_TLB.pas`: 히라 전자처방전
- `fOcxImageViewer_TLB.pas`: 이미지 뷰어
- `fepositoryCLib_TLB.pas`: 이미지 저장소

#### 스캐너/프린터
- `bitScannerLib_TLB.pas`: 비트 스캐너
- `BITIMGPRINTLib_TLB.pas`: 비트 이미지 프린터
- `bitPageViewLib_TLB.pas`: 비트 페이지 뷰어
- `bitThumbViewLib_TLB.pas`: 비트 썸네일 뷰어
- `BITSFTLib_TLB.pas`: 비트 SFT
- `friendlyPrinterLib_TLB.pas`: 프렌들리 프린터
- `friendlyViewLib_TLB.pas`: 프렌들리 뷰어

#### 리포트/서식
- `SurveyFormDesignerLib_TLB.pas`: 설문지 디자이너
- `SurveyFormPrinterLib_TLB.pas`: 설문지 프린터
- `SurveyFormRecognizerLib_TLB.pas`: 설문지 인식기
- `SignPopUp_TLB.pas`: 서명 팝업

#### 기타
- `WMPLib_TLB.pas`: Windows Media Player
- `ShockwaveFlashObjects_TLB.pas`: Flash 객체
- `rmpHTML_TLB.pas`: HTML 뷰어
- `LCViewer_TLB.pas`: LC 뷰어
- `lctechIGc_TLB.pas`: LC 기술 IGc
- `KMCLIENTAXLib_TLB.pas`: KM 클라이언트
- `KiccPosIE_TLB.pas`: KICC POS
- `FaxApp_TLB.pas`: 팩스 애플리케이션
- `PostGW_Smis_TLB.pas`: 포털 게이트웨이
- `SDSCTLLib_TLB.pas`: SDS 컨트롤
- `SSCANCTLLib_TLB.pas`: 스캔 컨트롤
- `SIMGCTLLib_TLB.pas`: 이미지 컨트롤
- `CONNECTIONMODULELib_TLB.pas`: 연결 모듈
- `RT_ConnectionModuleLib_TLB.pas`: RT 연결 모듈
- `DataCompatiableLib_TLB.pas`: 데이터 호환성
- `CLIPeFormAgentV2AdapterLib_TLB.pas`: 전자서식 에이전트
- `mscorlib_TLB.pas`: .NET 라이브러리

---

## ZipMaster - ZIP 파일 처리

### 1. ZipMstr.pas - ZIP 파일 컴포넌트

**역할**: ZIP 파일의 압축 및 해제를 처리합니다.

#### 주요 클래스

##### TZipMaster
```pascal
TZipMaster = class(TComponent)
  property ZipFileName: TZMString;
  property FSpecArgs: TStrings;  // 처리할 파일 목록
  property AddOptions: TZMAddOpts;
  property ExtrOptions: TZMExtrOpts;
  property Password: string;
  
  function Add: Integer;  // 파일 추가
  function Extract: Integer;  // 파일 추출
  function Delete: Integer;  // 파일 삭제
  function List: Integer;  // 파일 목록 조회
end;
```

**주요 옵션**:
- `AddOptions`: 
  - `AddDirNames`: 디렉토리 이름 포함
  - `AddRecurseDirs`: 하위 디렉토리 포함
  - `AddEncrypt`: 암호화
  - `AddUpdate`: 업데이트 모드
- `ExtrOptions`:
  - `ExtrOverWrite`: 덮어쓰기
  - `ExtrDirNames`: 디렉토리 생성
  - `ExtrTest`: 테스트 모드

**사용 예제**:
```pascal
var
  ZipMaster: TZipMaster;
begin
  ZipMaster := TZipMaster.Create(nil);
  try
    ZipMaster.ZipFileName := 'C:\Test.zip';
    ZipMaster.FSpecArgs.Add('C:\Source\*.*');
    ZipMaster.Add;
  finally
    ZipMaster.Free;
  end;
end;
```

---

## Rexpert - 리포트 생성

### 1. Clipsoft_TLB.pas - 리포트 뷰어 타입 정의

**역할**: Rexpert 리포트 뷰어의 타입 라이브러리를 정의합니다.

**주요 인터페이스**:
- `TRexViewerCtrl30`: 리포트 뷰어 컨트롤
- 리포트 인쇄, 미리보기, 내보내기 기능

---

## RemotePrintServer - 원격 프린트 서버

### 1. fSMCServer.pas - 프린트 서버 메인

**역할**: TCP 서버로 프린트 요청을 수신하고 처리합니다.

#### 주요 클래스

##### TfrmSMCPrintServer
```pascal
TfrmSMCPrintServer = class(TForm)
  IdTCPServer: TIdTCPServer;
  
  procedure IdTCPServerExecute(AContext: TIdContext);
  function RexPert_Print(AStream: TStream; APrintPort: String; 
    ABeginPage, AEndPage, ACopies, ABinId: integer; 
    OpenDialog: Boolean): Boolean;
  function RexPert_View(AFileName: String; APrintPort: String; 
    OpenDialog: Boolean): Boolean;
end;
```

**주요 기능**:
1. **TCP 서버**: `TIdTCPServer`로 프린트 요청 수신
2. **프린트 큐**: `ThreadPrintQueue`로 비동기 처리
3. **프린트 타입**:
   - `0`: Rexpert 리포트 (UI)
   - `1`: Sato 라인 프린터
   - `2`: 바코드 프린터
   - `3`: 텍스트 프린터
   - `4`: 이미지 프린터

#### 프린트 요청 프로토콜

##### TServerBlock
```pascal
TServerBlock = packed record
  Command: array[0..7] of ansiChar;  // "Print001", "Print002" 등
  PrintType: integer;  // 프린트 타입
  BeginPage: integer;  // 시작 페이지
  EndPage: integer;  // 종료 페이지
  Copies: Integer;  // 복사본 수
  BinId: Integer;  // 빈 ID
  UserName: array[0..19] of ansiChar;
  PrintName: array[0..49] of ansiChar;
  FileName: array[0..49] of ansiChar;
end;
```

**프린트 처리 흐름**:
1. TCP 연결 수신
2. `TServerBlock` 헤더 읽기
3. 프린트 데이터 수신
4. `ThreadPrintQueue`에 추가
5. 백그라운드 스레드에서 프린트 실행

#### ThreadPrintQueue - 프린트 큐 스레드

```pascal
ThreadPrintQueue = class(TThread)
  procedure Execute; override;
  procedure AddRealItem(Data: AnsiString);
  procedure ExePrint(APrintDatas: TSMCPrintData);
  
  procedure Print_RexpertCall();
  procedure Print_TxtCall();
  procedure Print_CarCall();
  procedure Print_BarCordLptCall();
  procedure Print_ImageCall();
end;
```

**주요 메서드**:
- `Print_RexpertCall`: Rexpert 리포트 인쇄
- `Print_TxtCall`: 텍스트 프린터 인쇄
- `Print_CarCall`: 차량 프린터 인쇄
- `Print_BarCordLptCall`: 바코드 LPT 프린터 인쇄
- `Print_ImageCall`: 이미지 프린터 인쇄

### 2. TextPrintPas.pas - 텍스트 프린터 처리

**역할**: 텍스트 프린터로 직접 출력합니다.

#### 주요 함수

##### InitialPrintVariable_X
```pascal
function InitialPrintVariable_X(myprnport, myprntype, KibonPrinter: AnsiString): Boolean;
```
- **역할**: 프린터 초기화
- **파라미터**:
  - `myprnport`: 포트 (예: "LPT1", "LPT2")
  - `myprntype`: 프린터 타입 ("윈도우프린터", "텍스트파일")
  - `KibonPrinter`: 기본 프린터 이름

##### Writestr_x / WritestrLn_x
```pascal
procedure Writestr_x(S: AnsiString);
procedure WritestrLn_x(Mystr: AnsiString);
```
- **역할**: 프린터로 문자열 출력

##### LineFeed_X / FormFeed_X
```pascal
procedure LineFeed_X(LoopCount: integer);
procedure FormFeed_X(HowMuchLine: Integer);
```
- **역할**: 
  - `LineFeed_X`: 줄바꿈
  - `FormFeed_X`: 폼 피드 (페이지 넘김)

##### ClosePrintVariable_X
```pascal
procedure ClosePrintVariable_X;
```
- **역할**: 프린터 연결 종료

**사용 예제**:
```pascal
InitialPrintVariable_X('LPT1', '윈도우프린터', '기본프린터');
WritestrLn_x('인쇄할 텍스트');
LineFeed_X(3);
FormFeed_X(0);
ClosePrintVariable_X;
```

---

## 핵심 기능 상세 분석

### 1. 인증서 처리 시스템

#### 인증서 API 로드

```pascal
function CertAPI: Integer;
begin
  if not CertInf then
  begin
    EMRDLLHandle := LoadLibrary('EMRDLL.DLL');
    if EMRDLLHandle <> 0 then
    begin
      // 함수 포인터 할당
      @CERT_SetMASInfo := GetProcAddress(EMRDLLHandle, 'CERT_SetMASInfo');
      @CERT_SignEMR := GetProcAddress(EMRDLLHandle, 'CERT_SignEMR');
      // ... 기타 함수들
      CertInf := True;
    end;
  end;
  Result := EMRDLLHandle;
end;
```

#### EMR 서명 프로세스

1. **인증서 로드**
   ```pascal
   CERT_LoadCertCard(PinNumber, CertPassword);
   ```

2. **EMR 서명**
   ```pascal
   CERT_SignEMR(PlainEMR, PlainEMRLen, SignedEMR, SignedEMRLen);
   ```

3. **서명 검증**
   ```pascal
   CERT_VerifyEMR(SignedEMR, SignedEMRLen, PlainEMR, PlainEMRLen);
   ```

#### EMR 암호화 프로세스

1. **키 생성** (서버 인증서 사용)
2. **암호화**
   ```pascal
   CERT_EncryptEMRWithKey(Key, IV, SignedEMR, SignedEMRLen, 
     EncryptedEMR, EncryptedEMRLen);
   ```

3. **복호화**
   ```pascal
   CERT_DecryptEMRWithKey(Key, IV, EncryptedEMR, EncryptedEMRLen, 
     SignedEMR, SignedEMRLen);
   ```

### 2. 전화기 연동 시스템

#### 전화기 초기화

```pascal
function MctmLoad: Integer;
begin
  if not MctmInf then
  begin
    MCTMDLLHandle := LoadLibrary('MCTM.DLL');
    if MCTMDLLHandle <> 0 then
    begin
      @MCTM_Init := GetProcAddress(MCTMDLLHandle, 'MCTM_Init');
      @MCTM_LogON := GetProcAddress(MCTMDLLHandle, 'MCTM_LogON');
      // ... 기타 함수들
      MctmInf := True;
    end;
  end;
  Result := MCTMDLLHandle;
end;
```

#### 전화기 로그인

```pascal
MCTM_Init(Handle);
Result := MCTM_LogON(ExtNumber, UserID, UserPassword, Option, 
  LogOnType, ServerIP);
```

#### 통화 제어

```pascal
// 통화 발신
MCTM_MakeCall(ExtNumber, DestNumber);

// 통화 전환
MCTM_TransferCall(ExtNumber, DestNumber);

// 상태 조회
MCTM_GetExtStatus(ExtNumber, LogStatus, ExtStatus, FwdNumber, FwdType);
```

### 3. ZIP 파일 처리 시스템

#### ZIP 파일 생성

```pascal
var
  Zip: TZip;
  Stream: TMemoryStream;
begin
  Zip := TZip.Create;
  try
    Stream := TMemoryStream.Create;
    try
      Zip.CreateZip(Stream);
      // 파일 추가
      Zip.AddStream(FileStream);
      // ZIP 파일 저장
      Stream.SaveToFile('C:\Test.zip');
    finally
      Stream.Free;
    end;
  finally
    Zip.Free;
  end;
end;
```

#### ZIP 파일 읽기

```pascal
var
  Zip: TZip;
  Stream: TFileStream;
  Entry: TZipEntriesEntry;
begin
  Zip := TZip.Create;
  try
    Stream := TFileStream.Create('C:\Test.zip', fmOpenRead);
    try
      Zip.OpenZip(Stream);
      // 첫 번째 항목 추출
      Entry := Zip.Entries[0];
      Zip.ExtractToStream(Entry, OutputStream);
    finally
      Stream.Free;
    end;
  finally
    Zip.Free;
  end;
end;
```

### 4. 원격 프린트 서버 시스템

#### 서버 시작

```pascal
procedure TfrmSMCPrintServer.FormCreate(Sender: TObject);
begin
  IdTCPServer.DefaultPort := IniFilePort;
  IdTCPServer.Active := True;
  AThreadPrintQueue := ThreadPrintQueue.Create;
end;
```

#### 프린트 요청 처리

```pascal
procedure TfrmSMCPrintServer.IdTCPServerExecute(AContext: TIdContext);
var
  Block: TServerBlock;
  PrintData: TSMCPrintData;
begin
  // 헤더 읽기
  AContext.Connection.IOHandler.ReadBytes(Block, SizeOf(TServerBlock));
  
  // 프린트 데이터 읽기
  PrintData := DataListHeader(DataString);
  
  // 큐에 추가
  AThreadPrintQueue.AddRealItem(DataString);
end;
```

#### 프린트 실행

```pascal
procedure ThreadPrintQueue.Execute;
begin
  while not Terminated do
  begin
    if AQueue.Count > 0 then
    begin
      Data := AQueue.Pop;
      case Qr_PrintType of
        0: Print_RexpertCall();
        1: Print_TxtCall();
        2: Print_BarCordLptCall();
        3: Print_CarCall();
        4: Print_ImageCall();
      end;
    end;
    Sleep(100);
  end;
end;
```

---

## 주요 클래스 및 함수 요약

### CommonPack

| 클래스/함수 | 파일 | 설명 |
|------------|------|------|
| `StringParseString` | CommonFunc.pas | 문자열 분리 |
| `NVL` | CommonFunc.pas | NULL 값 처리 |
| `LPAD` / `RPAD` | CommonFunc.pas | 문자열 패딩 |
| `ConvertSDate` / `InvertSDate` | CommonFunc.pas | 날짜 형식 변환 |
| `CalcAge` / `GetAge` | CommonFunc.pas | 나이 계산 |
| `EncryptPassword` / `DecryptPassword` | CommonFunc.pas | 패스워드 암호화 |
| `HashSHA256` | CommonFunc.pas | SHA256 해시 |
| `CitizenChk` | CommonFunc.pas | 주민등록번호 검증 |
| `InputNum` / `InputInt` / `InputReal` | CommonFunc.pas | 숫자 입력 검증 |
| `CertAPI` / `CertFree` | CommonFunc.pas | 인증서 API 로드 |
| `MctmLoad` / `MctmFree` | CommonFunc.pas | 전화기 API 로드 |
| `TZip` | XMLZip.pas | ZIP 파일 처리 |
| `TDCP_sha256` | DCPsha256.pas | SHA256 해시 |
| `Base64EncodeStr` / `Base64DecodeStr` | DCPbase64.pas | Base64 인코딩 |
| `TSMCPopupTree` | PopupTree.pas | 팝업 트리 메뉴 |

### CPortLib

| 클래스/함수 | 파일 | 설명 |
|------------|------|------|
| `TComPort` | CPort.pas | 시리얼 포트 컴포넌트 |
| `TComFlowControl` | CPort.pas | 흐름 제어 설정 |
| `TComTimeouts` | CPort.pas | 타임아웃 설정 |
| `TComDataPacket` | CPort.pas | 패킷 수신 |
| `TBarCodeScaner` | BarCodeScaner.pas | 바코드 스캐너 |

### BarcodePack

| 클래스/함수 | 파일 | 설명 |
|------------|------|------|
| `TBarcode2Dun` | dbc2dun.pas | 2D 바코드 생성 |

### RemotePrintServer

| 클래스/함수 | 파일 | 설명 |
|------------|------|------|
| `TfrmSMCPrintServer` | fSMCServer.pas | 프린트 서버 메인 |
| `ThreadPrintQueue` | fSMCServer.pas | 프린트 큐 스레드 |
| `InitialPrintVariable_X` | TextPrintPas.pas | 프린터 초기화 |
| `Writestr_x` | TextPrintPas.pas | 텍스트 출력 |
| `ClosePrintVariable_X` | TextPrintPas.pas | 프린터 종료 |

---

## 결론

COM_LV0 프로젝트는 Nexmed EHR 시스템의 가장 기본적인 인프라 레이어로, 다음과 같은 특징을 가집니다:

1. **기초 유틸리티**: 문자열, 날짜, 숫자 처리 등 기본적인 유틸리티 함수 제공
2. **하드웨어 통신**: 시리얼 포트, 바코드 스캐너 등 하드웨어 연동
3. **보안**: 암호화, 해시, 인증서 처리
4. **외부 연동**: 다양한 외부 라이브러리(OCX/DLL) 타입 정의
5. **파일 처리**: ZIP 압축/해제, 리포트 생성
6. **프린트 서버**: 원격 프린트 요청 처리

이 레이어는 다른 모든 상위 레이어(COM_LV1, COM_LV2, COM_LV3)의 기반이 되며, 시스템 전반에서 공통으로 사용되는 기능들을 제공합니다.

