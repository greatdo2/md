# LOGIN & MAIN 프로젝트 상세 분석서

## 📋 목차
1. [LOGIN 프로젝트 상세 분석](#login-프로젝트-상세-분석)
2. [MAIN 프로젝트 상세 분석](#main-프로젝트-상세-분석)
3. [주요 함수별 상세 설명](#주요-함수별-상세-설명)
4. [실행 흐름도](#실행-흐름도)

---

# LOGIN 프로젝트 상세 분석

## 📁 프로젝트 구조

```
LOGIN/
├── Nexmed_EHR_LOADER.dpr      # 메인 진입점
├── Loader.pas                  # 로더 폼 및 핵심 로직 (약 2,500줄)
├── Loader.dfm                  # 로더 폼 디자인
├── LoginCommon.pas             # 로그인 공통 유틸리티
├── CoManager.pas               # 컴포넌트 관리자
├── FileCopy.pas                # 파일 복사 유틸리티
└── NetFwTypeLib_TLB.pas        # 방화벽 타입 라이브러리
```

---

## 🔍 주요 파일 상세 분석

### 1. Nexmed_EHR_LOADER.dpr (진입점)

**역할**: 프로그램의 시작점

**코드 구조**:
```pascal
program Nexmed_EHR_LOADER;

uses
  Vcl.Forms,
  SysUtils,
  LoginCommon in 'LoginCommon.pas',
  Loader in 'Loader.pas' {frmLoader: TSMCForm},
  CoManager in 'CoManager.pas';

begin
  Application.Initialize;
  Application.MainFormOnTaskbar := True;
  Application.Title := 'Nexmed EHR Loader';
  Application.CreateForm(TfrmLoader, frmLoader);
  Application.Run;
  Application.Terminate;
end.
```

**작동 방식**:
1. 애플리케이션 초기화
2. 로더 폼(`TfrmLoader`) 생성
3. 애플리케이션 실행

---

### 2. Loader.pas (핵심 로직)

**파일 크기**: 약 2,500줄  
**주요 클래스**: `TfrmLoader` (TSMCForm 상속)

#### 📌 주요 멤버 변수

```pascal
private
  MainGlobalVar: TSMCGlobalVar;                // 전역변수
  ServerVerTable: TSHISDataSet;                // 서버 파일 버전정보
  LocalVerTable: TSMCMemTable;                 // 로컬 파일 버전정보
  DiffVerTable: TSMCMemTable;                  // 버전이 다른 파일정보
  ExeAppTable: TSMCMemTable;                   // 애플리케이션 정보 테이블
  MessageCodeTable: TSMCMemTable;              // 메시지 코드
  FileDownInfoTable: TSMCMemTable;             // 파일 다운로드 경로 정보
  DownCompleteFileList: TList<TDownFileInfo>;  // 다운로드 성공 파일 리스트
  
  FConfigIni: TIniFile;                        // Config.ini 파일
  FDevManIni: TIniFile;                        // DevMan.ini 파일
  FSystemEnv: string;                          // 시스템 환경 (D/P/T/R)
  FSystemDir: string;                          // 시스템 디렉토리
  FParam1, FParam2: string;                    // 실행 파라미터
  FVersionCheck: Boolean;                       // 버전 체크 여부
  FAddress: String;                            // 서버 주소
  FPort: Integer;                              // 서버 포트
```

#### 📌 주요 이벤트 핸들러

##### `SMCFormCreate` (폼 생성 시)
**위치**: `Loader.pas:1202`  
**역할**: 초기화 작업 수행

**주요 작업**:
1. **파라미터 파싱**
   ```pascal
   FParam1 := UpperCase(Trim(ParamStr(1)));  // 첫 번째 파라미터 (예: DEV.EXE)
   FParam2 := Trim(ParamStr(2));             // 두 번째 파라미터
   ```

2. **시스템 환경 결정**
   ```pascal
   if Pos('DR.EXE', FParam1) > 0 then
     FSystemEnv := 'R';  // 운영
   else if Pos('PROD.EXE', FParam1) > 0 then
     FSystemEnv := 'P';  // 운영
   else if Pos('DEV.EXE', FParam1) > 0 then
     FSystemEnv := 'D';  // 개발
   else if Pos('TEST.EXE', FParam1) > 0 then
     FSystemEnv := 'T';  // 테스트
   ```

3. **설정 파일 로드**
   ```pascal
   FConfigIni := TIniFile.Create(MainGlobalVar.Path.Env + INI_Config);
   FDevManIni := TIniFile.Create(MainGlobalVar.Path.Ini + INI_DevMan);
   Load2Config(FConfigIni);
   ```

4. **메모리 테이블 생성**
   - `LocalVerTable`: 로컬 파일 버전 정보 저장
   - `DiffVerTable`: 서버와 다른 파일 목록
   - `ExeAppTable`: 실행할 애플리케이션 목록
   - `MessageCodeTable`: 메시지 코드 테이블
   - `FileDownInfoTable`: 파일 다운로드 경로 정보

5. **버전 정보 로드**
   ```pascal
   LoadVersionInfo(MainGlobalVar.Path.Ini + VER_Common);
   ```

6. **애플리케이션 목록 로드**
   ```pascal
   LoadExeAppList;      // ExeAppList.dat 파일 읽기
   LoadFileDownInfo;    // FileDownInfo.dat 파일 읽기
   ```

##### `Timer1Timer` (타이머 이벤트)
**위치**: `Loader.pas:1953`  
**역할**: 폼 표시 완료 후 버전 체크 시작

**작동 순서**:
1. 타이머 비활성화
2. 서버 연결 시도
3. `CommonFileVersionCheck` 호출

##### `btnUpdateClick` (업데이트 버튼 클릭)
**위치**: `Loader.pas:1158`  
**역할**: 수동 업데이트 시작

```pascal
procedure TfrmLoader.btnUpdateClick(Sender: TObject);
begin
  CommonFileVersionCheck;  // 버전 체크 및 다운로드
  Close;                   // 로더 종료
end;
```

---

#### 📌 핵심 함수 상세 분석

##### `Load2Config` (설정 파일 로드)
**위치**: `Loader.pas:1432`  
**역할**: Config.ini 파일에서 서버 정보 읽기

**주요 작업**:
1. **서버 타입 읽기**
   ```pascal
   ConfigServerType := ReadString('SHIS', 'SERVERTYPE', 'Local').ToUpper;
   ```

2. **SSL 설정**
   ```pascal
   IsSSL := ReadString(ConfigServerType, 'SSLYN', 'Y');
   if SameText(IsSSL, 'Y') then
   begin
     IS_TRANSMIT_SSL := True;
     SSLHandler := TIdSSLIOHandlerSocketOpenSSL.Create(IdHTTPDownload);
     SSLHandler.SSLOptions.Method := sslvTLSv1_2;
     IdHTTPDownload.IOHandler := SSLHandler;
   end;
   ```

3. **서버 주소/포트 디코딩** (Base64)
   ```pascal
   ServerName := Decode64ForConfigIni(ServerName);
   ServerPort := StrToInt(Decode64ForConfigIni(ServerPortStr));
   ```

4. **압축 설정**
   ```pascal
   if SameText(IsCompress, 'Y') then
   begin
     SEHR.DataSet.RestClient.UseCompress := True;
     IdHTTPDownload.Compressor := TIdCompressorZLib.Create(IdHTTPDownload);
   end;
   ```

##### `GetCommonFileList` (서버 파일 목록 조회)
**위치**: `Loader.pas:2197`  
**역할**: 서버에서 업데이트할 파일 목록 가져오기

**작동 방식**:
```pascal
function TfrmLoader.GetCommonFileList(var AResultTable: TSHISDataSet): Boolean;
begin
  Result := False;
  AResultTable := dtPrgmFmtL04;
  
  dp_PrgmFmtL04.Request.Clear;
  dp_PrgmFmtL04.Request.FieldByName('afiRturYn').AsString := 'Y';
  
  try
    dp_PrgmFmtL04.Start(Self);  // REST API 호출
    Result := True;
  except
    on E: Exception do
      Exit;
  end;
end;
```

**서버 응답 데이터 구조**:
- `dsrbLctnNm`: 파일 위치 (예: "DLL/", "BPL/")
- `cfrtPrgmId`: 파일명 (예: "Nexmed_EHR.exe")
- `ceckPhspDt`: 파일 수정 시간 (yyyymmddhhnnss)
- `prgmSizeCnt`: 파일 크기
- `devpVrsnNm`: 개발 버전
- `comnYn`: 공통 파일 여부 ('Y'/'N')

##### `CommonFileVersionCheck` (버전 체크 및 다운로드)
**위치**: `Loader.pas:2240`  
**역할**: 전체 버전 체크 및 다운로드 프로세스 관리

**실행 흐름**:
```pascal
function TfrmLoader.CommonFileVersionCheck: Boolean;
begin
  // 1. 서버에서 파일 목록 가져오기
  if GetCommonFileList(ServerVerTable) and not ServerVerTable.IsEmpty then
  begin
    // 2. 파일 다운로드 실행
    FCommonFileDownResult := CommonFileDownload();
    
    case FCommonFileDownResult of
      cfdrOk:  // 성공
        begin
          RegisterOcxDll;              // OCX/DLL 등록
          ExecuteAppMainRun;           // 구동 시 실행할 앱 실행
          ExecuteAppDownload;          // 버전처리 다운로드 시 실행
          ExecuteFullPathAppDownload;  // 절대경로 앱 실행
          
          // 메인 프로그램 실행
          ExecAsAdmin(ExtractFilePath(Application.ExeName) + MAIN_EXEFILENAME,
                      FSystemEnv + ' ' + FParam2,
                      SW_SHOWNORMAL);
        end;
      cfdrLoader, cfdrLauncher:  // 로더/런처 업데이트 필요
        begin
          // 새 로더로 재시작
          CopyFile(PChar(sTempLoader), PChar(sLoader), False);
          ExecAsAdmin(MainGlobalVar.Path.Bin + FParam1, FILE_LOADER, SW_SHOWNORMAL);
        end;
      cfdrFail:  // 실패
        begin
          SMCShowMessage('IC03059');
        end;
    end;
  end;
end;
```

##### `CommonFileDownload` (파일 다운로드)
**위치**: `Loader.pas:617`  
**역할**: 변경된 파일 다운로드

**핵심 로직**:

1. **버전 비교**
   ```pascal
   // 서버 파일 목록 순회
   ServerVerTable.First;
   while not ServerVerTable.EOF do
   begin
     // 로컬 파일 존재 여부 확인
     LFileLastModTime := GetFileLastModTime(sLocalFile);
     
     if LFileLastModTime = 0 then
       LState := dsInsert  // 새 파일
     else if LocalVerTable.Locate('cfrtPrgmId', sProgId, []) then
     begin
       // 버전 비교
       if (서버버전 = 로컬버전) then
         LState := dsBrowse  // 동일
       else
         LState := dsEdit;   // 업데이트 필요
     end;
     
     // 다운로드 목록에 추가
     if LState <> dsBrowse then
     begin
       DiffVerTable.Append;
       DiffVerTable.Fields[0].AsString := sProgId;
       DiffVerTable.Fields[1].AsInteger := Integer(LState);
     end;
   end;
   ```

2. **파일 다운로드** (`ProcessCursor` 내부 함수)
   ```pascal
   function ProcessCursor: Boolean;
   begin
     // HTTP 다운로드
     sServerFile := 'EHR/' + FSystemDir + sSubDir + sProgId;
     bDownload := HttpDownload(sServerFile, FStream);
     
     // LZMA 압축 해제 또는 일반 저장
     if TMemoryStream(FStream).IsLZMA then
       DecompressFile(FStream, sTempFile)
     else
       TMemoryStream(FStream).SaveToFile(sTempFile);
     
     // 파일 복사
     SetFileLastModTime(sTempFile, 서버수정시간);
     CopyFile(PChar(sTempFile), PChar(sLocalFile), False);
     DeleteFile(sTempFile);
     
     // 로컬 버전 정보 업데이트
     LocalVerTable.Edit;
     LocalVerTable.Fields[icCECK_PHSP_DT].AsString := 서버수정시간;
     LocalVerTable.Post;
   end;
   ```

3. **HTTP 다운로드** (`HttpDownload` 내부 함수)
   ```pascal
   function HttpDownload(AServerFile: string; AStream: TStream): Boolean;
   var
     sURL: string;
   begin
     // URL 구성
     sURL := sProtocol[IS_TRANSMIT_SSL] + FAddress + ':' + IntToStr(FPort) + '/';
     
     // HTTP GET 요청
     IdHttpDownload.Get(sURL + AServerFile, AStream);
     
     Result := True;
   end;
   ```

##### `RegisterOcxDll` (OCX/DLL 등록)
**위치**: `Loader.pas:2128`  
**역할**: 필요한 ActiveX 컨트롤 등록

**작동 방식**:
```pascal
procedure TfrmLoader.RegisterOcxDll;
var
  LRegList: TStringList;
  sFileName: string;
begin
  sFileName := MainGlobalVar.Path.Env + DAT_RegisterOcxDll;
  if FileExists(sFileName) then
  begin
    LRegList := TStringList.Create;
    try
      LRegList.LoadFromFile(sFileName);
      RegisterDllList(MainGlobalVar.Path.Root, LRegList);
    finally
      LRegList.Free;
    end;
  end;
end;
```

**RegisterOcxDll.dat 파일 형식**:
```
파일경로|CLSID|ProgID
예: DLL\SomeControl.ocx|{12345678-1234-1234-1234-123456789012}|SomeControl.ProgID
```

**등록 프로세스**:
1. `CoManager`로 등록 필요 여부 확인
2. `regsvr32.exe` 실행 (관리자 권한)
3. 등록 결과 로그 기록

##### `KillProcessApp` (프로세스 종료)
**위치**: `Loader.pas:1674`  
**역할**: 업데이트 전 실행 중인 프로세스 종료

```pascal
procedure TfrmLoader.KillProcessApp;
begin
  with ExeAppTable do
  begin
    Filter := 'appType = ''3''';  // 구동 시 종료할 앱
    Filtered := True;
    
    First;
    while not Eof do
    begin
      sFileName := FieldByName('appName').AsString;
      if DiffVerTable.Locate('cfrtPrgmId', sFileName, []) then
      begin
        CloseApp(sFileName);  // 안전하게 종료
      end;
      Next;
    end;
  end;
end;
```

**ExeAppList.dat 파일 형식**:
```
1,애플리케이션경로    // 구동 시 실행
2,애플리케이션경로    // 방화벽 예외
3,애플리케이션경로    // 구동 시 종료
4,애플리케이션경로    // 버전처리 다운로드 시 실행
5,애플리케이션경로    // 절대경로 버전처리 다운로드 시 실행
```

##### `LoadVersionInfo` / `SaveVersionInfo` (버전 정보 관리)
**위치**: `Loader.pas:2163, 2181`  
**역할**: 로컬 파일 버전 정보 저장/로드

**파일 형식**: CSV (kbmMemTable 형식)

**데이터 구조**:
- `cfrtPrgmId`: 파일명
- `prgmDscrCtn`: 파일 설명
- `prgmSizeCnt`: 파일 크기
- `devpVrsnNm`: 개발 버전
- `ceckPhspDt`: 수정 시간 (yyyymmddhhnnss)
- `dsrbLctnNm`: 저장 위치
- `comnYn`: 공통 파일 여부

---

### 3. LoginCommon.pas (공통 유틸리티)

**주요 함수**:

##### `SetSMCPath` (경로 설정)
**위치**: `LoginCommon.pas:172`  
**역할**: 환경 변수 PATH에 LIB, DLL 경로 추가

```pascal
procedure SetSMCPath(APath: string);
var
  sNewPath: string;
begin
  sRootPath := IncludeTrailingPathDelimiter(APath);
  sNewPath := GetEnvironmentVariable('Path');
  sNewPath := Format('%sLIB;%sDLL;', [sRootPath, sRootPath]) + sNewPath;
  SetEnvironmentVariable('Path', PWideChar(sNewPath));
end;
```

##### `ExecAsAdmin` / `ExecAsAdminAndWait` (관리자 권한 실행)
**위치**: `LoginCommon.pas:112, 133`  
**역할**: 관리자 권한으로 프로그램 실행

```pascal
procedure ExecAsAdmin(const AFileName: string; const AParams: string; AShowWindow: Cardinal);
var
  SEI: TShellExecuteInfo;
begin
  FillChar(SEI, SizeOf(SEI), 0);
  SEI.cbSize := SizeOf(SEI);
  SEI.lpVerb := 'runas';  // 관리자 권한
  SEI.lpFile := Pointer(AFileName);
  SEI.lpParameters := Pointer(AParams);
  SEI.nShow := AShowWindow;
  ShellExecuteEx(@SEI);
end;
```

##### `TaskKill` (프로세스 종료)
**위치**: `LoginCommon.pas:183`  
**역할**: taskkill.exe를 관리자 권한으로 실행

```pascal
procedure TaskKill(const AParam: string; AShowWindow: Cardinal; AWait: Boolean);
var
  sFileName: string;
begin
  sFileName := GetSMCRootPath(Application.ExeName) + 'EXE\Nexmed_EHR_TASKKILL.exe';
  if FileExists(sFileName) then
  begin
    if AWait then
      ExecAsAdminAndWait(sFileName, AParam, AShowWindow)
    else
      ExecAsAdmin(sFileName, AParam, AShowWindow);
  end;
end;
```

##### `DecompressFile` (LZMA 압축 해제)
**위치**: `LoginCommon.pas:52`  
**역할**: LZMA 압축 파일 해제

```pascal
procedure DecompressFile(const AStream: TStream; AFileName: string);
var
  LTarget: TMemoryStream;
begin
  LTarget := TMemoryStream.Create;
  try
    AStream.Position := 0;
    LZMADecodeStream(AStream, LTarget);  // Abbrevia 라이브러리 사용
    LTarget.SaveToFile(AFileName);
  finally
    LTarget.Free;
  end;
end;
```

---

### 4. CoManager.pas (컴포넌트 관리자)

**역할**: OCX/DLL 등록 상태 확인

**주요 메서드**:

##### `Open` (파일 열기)
**위치**: `CoManager.pas:108`  
**역할**: RegisterOcxDll.dat 파일 읽기

```pascal
procedure TCoManager.Open(const Path: string);
var
  Lines: TStrings;
  Strings: TStrings;
  Item: TCoItem;
begin
  Lines := TStringList.Create;
  Strings := TStringList.Create;
  try
    Lines.LoadFromFile(Path);
    for Line in Lines do
    begin
      Strings.DelimitedText := Line;  // '|' 구분자
      if Strings.Count >= 3 then
      begin
        Item := TCoItem.Create;
        Item.Path := Trim(Strings[0]);    // 파일 경로
        Item.CLSID := Trim(Strings[1]);   // CLSID
        Item.ProgID := Trim(Strings[2]);  // ProgID
        FItems.AddOrSetValue(Item.Path, Item);
      end;
    end;
  finally
    Lines.Free;
    Strings.Free;
  end;
end;
```

##### `Check` (등록 상태 확인)
**위치**: `CoManager.pas:145`  
**역할**: 레지스트리에서 OCX/DLL 등록 상태 확인

**반환 값**:
- `0`: 정상 등록됨
- `-1`: 목록에 없음
- `-2`: CLSID 형식 오류
- `-3`: CLSID에서 ProgID 변환 실패
- `-4`: ProgID 불일치
- `-5`: 레지스트리에 없음 또는 파일 없음

```pascal
function TCoManager.Check(const Path: string): Integer;
var
  Item: TCoItem;
  CLSID: TGUID;
  ProgID: PChar;
  R: TRegistry;
begin
  Result := -1;
  
  if FItems.TryGetValue(Path, Item) then
  begin
    // CLSID 문자열을 GUID로 변환
    if Succeeded(CLSIDFromString(PChar(Item.CLSID), CLSID)) then
    begin
      // CLSID에서 ProgID 가져오기
      if Succeeded(ProgIDFromCLSID(CLSID, ProgID)) then
      begin
        try
          if IsSameProgID(Item.ProgID, ProgID) then
          begin
            // 레지스트리에서 파일 경로 확인
            R := TRegistry.Create(KEY_READ or KEY_WOW64_32KEY);
            try
              R.RootKey := HKEY_CLASSES_ROOT;
              if R.OpenKey('\CLSID\' + Item.CLSID + '\InprocServer32', False) then
              begin
                LPath := R.ReadString('');
                if TFile.Exists(LPath) then
                  Result := 0  // 정상
                else
                  Result := -5;  // 파일 없음
              end;
            finally
              R.Free;
            end;
          end;
        finally
          CoTaskMemFree(ProgID);
        end;
      end;
    end;
  end;
end;
```

---

## 🔄 LOGIN 프로젝트 실행 흐름

```
1. 사용자가 Nexmed_EHR_LOADER.exe 실행
   ↓
2. Nexmed_EHR_LOADER.dpr 시작
   ↓
3. TfrmLoader 생성 (SMCFormCreate 이벤트)
   ├─ 파라미터 파싱 (DEV.EXE, PROD.EXE 등)
   ├─ 시스템 환경 결정 (D/P/T/R)
   ├─ Config.ini 로드 (Load2Config)
   ├─ 메모리 테이블 생성
   ├─ 버전 정보 로드 (LoadVersionInfo)
   └─ 애플리케이션 목록 로드
   ↓
4. 폼 표시 완료 (SMCFormShowComplete)
   ↓
5. Timer1 활성화
   ↓
6. Timer1Timer 이벤트 발생
   ├─ 서버 연결 시도
   └─ CommonFileVersionCheck 호출
   ↓
7. CommonFileVersionCheck 실행
   ├─ GetCommonFileList (서버 파일 목록 조회)
   ├─ CommonFileDownload (파일 다운로드)
   │   ├─ 버전 비교
   │   ├─ 변경 파일 목록 생성
   │   ├─ MainAliveCheck (메인 프로그램 실행 중 확인)
   │   ├─ KillProcessApp (프로세스 종료)
   │   ├─ 파일 다운로드 (HTTP/HTTPS)
   │   ├─ 압축 해제 (LZMA)
   │   └─ 파일 복사 및 버전 정보 저장
   ├─ RegisterOcxDll (OCX/DLL 등록)
   ├─ ExecuteAppMainRun (구동 시 실행)
   ├─ ExecuteAppDownload (버전처리 다운로드 시 실행)
   └─ 메인 프로그램 실행 (Nexmed_EHR.exe)
   ↓
8. 로더 종료
```

---

# MAIN 프로젝트 상세 분석

## 📁 프로젝트 구조

```
MAiN/
├── Nexmed_EHR.dpr              # 메인 진입점
├── Main.pas                    # 메인 폼 (약 8,400줄)
├── Main.dfm                    # 메인 폼 디자인
├── MainHandler.pas             # 메인 이벤트 핸들러
├── Common.pas                  # 공통 함수 및 변수
├── PatientForm.pas             # 환자 정보 폼
├── PatientList.pas             # 환자 목록 폼
└── Dlg/
    ├── dlgLogin.pas            # 로그인 폼
    ├── dlgConfig.pas           # 설정 폼
    └── dlgPopupWindow.pas      # 팝업 윈도우
```

---

## 🔍 주요 파일 상세 분석

### 1. Nexmed_EHR.dpr (진입점)

**코드 구조**:
```pascal
program Nexmed_EHR;

uses
  SysUtils, Forms, Controls,
  Common in 'Common.pas',
  MainHandler in 'MainHandler.pas',
  dlgLogin in 'Dlg\dlgLogin.pas' {frmLogin},
  Main in 'Main.pas' {SMCMainForm};

begin
  Application.Initialize;
  Application.Title := 'Nexmed EHR';
  Application.CreateForm(TSMCMainForm, SMCMainForm);
  
  // 패키지 로드
  SMCMainForm.LoadMainComPack;  // 메인 공통 패키지
  SMCMainForm.LoadComPack;      // 기타 패키지
  
  // 로그인 폼 생성
  frmLogin := TfrmLogin.Create(Application);
  
  // 로그인 확인
  if frmLogin.ShowModal = mrOk then
    Application.Run
  else
    Application.Terminate;
end.
```

---

### 2. Main.pas (메인 폼)

**파일 크기**: 약 8,400줄  
**주요 클래스**: `TSMCMainForm` (TSMCForm 상속)

#### 📌 주요 멤버 변수

```pascal
public
  MainGlobalVar: TMainGlobalVar;            // 전역변수
  MainConfig: TSMCMainConfig;               // 환경파일 관련
  MainMsgHandler: TSMCMainMsgHandler;       // 메시지 핸들러
  MainMemTable: TSMCMainTable;              // 메모리테이블
  MainPageDivision: TSMCPageDivision;       // 메인페이지
  MainPatientForm: TSMCPatientForm;         // 환자정보
  MainPatientList: TSMCPatientList;         // 환자리스트
  MainHomePanel: TPanel;                    // 홈 패널
  MainHomePageControl: TSMCMainPageControl; // 홈 페이지컨트롤
  MainMenuAll: TMainMenu;                   // 메인메뉴
  MainMenuVisible: TMainMenu;               // 메인메뉴 툴바용
  MainWaitCursor: TMainWaitCursor;          // 대기 커서
```

#### 📌 주요 이벤트 핸들러

##### `SMCFormCreate` (폼 생성 시)
**위치**: `Main.pas:2814`  
**역할**: 메인 폼 초기화

**주요 작업**:
1. **전역 변수 초기화**
   ```pascal
   MainGlobalVar := TMainGlobalVar.Create;
   MainConfig := TSMCMainConfig.Create;
   MainMsgHandler := TSMCMainMsgHandler.Create;
   MainMemTable := TSMCMainTable.Create;
   ```

2. **경로 설정**
   ```pascal
   SetSMCPath(MainGlobalVar.Path.Root);
   SetLibraryPath;  // BPL 패키지 경로 설정
   ```

3. **마스터 파일 로드**
   ```pascal
   LoadDefaultMessageCodeFile;    // 메시지 코드
   LoadDefaultCommonCodeFile;     // 공통 코드
   LoadDefaultDeptFile;            // 부서 정보
   LoadBswrFnctCdFile;            // 병원별 기능 코드
   LoadWardFile;                  // 병실 정보
   ```

4. **타이머 설정**
   ```pascal
   SetTimer(Handle, TIMER_MAINMENU, 1000, nil);      // 메인메뉴 타이머
   SetTimer(Handle, TIMER_PATIENT, 1000, nil);      // 환자 타이머
   SetTimer(Handle, TIMER_CONNECTINFO, 5000, nil);   // 연결 정보 타이머
   SetTimer(Handle, TIMER_NOTI, 30000, nil);        // 알림 타이머
   ```

5. **환자 정보/목록 폼 생성**
   ```pascal
   CreatePatInfoForm;  // 환자 정보 폼 생성
   ```

##### `LoadMainComPack` (메인 공통 패키지 로드)
**위치**: `Main.pas:2333`  
**역할**: 필수 BPL 패키지 동적 로드

```pascal
procedure TSMCMainForm.LoadMainComPack;
var
  sPackageName: string;
  hPackage: THandle;
begin
  // COM_LV2 패키지들
  sPackageName := MainGlobalVar.Path.Bpl + 'SMC.bpl';
  hPackage := LoadPackage(sPackageName);
  MainPackageHandleList.Add(Pointer(hPackage));
  
  sPackageName := MainGlobalVar.Path.Bpl + 'SMCExpress.bpl';
  hPackage := LoadPackage(sPackageName);
  MainPackageHandleList.Add(Pointer(hPackage));
  
  // 기타 필수 패키지들...
end;
```

##### `LoadComPack` (기타 패키지 로드)
**위치**: `Main.pas:2426`  
**역할**: 추가 BPL 패키지 동적 로드

```pascal
procedure TSMCMainForm.LoadComPack;
var
  sPackageName: string;
  hPackage: THandle;
begin
  // COM_LV3 패키지들
  sPackageName := MainGlobalVar.Path.Bpl + 'NRCViewerPackage.bpl';
  if FileExists(sPackageName) then
  begin
    hPackage := LoadPackage(sPackageName);
    MainPackageHandleList.Add(Pointer(hPackage));
  end;
  
  // 기타 선택적 패키지들...
end;
```

##### `CallMainMenuInfoService` (메인 메뉴 정보 조회)
**위치**: `Main.pas:1245`  
**역할**: 서버에서 메뉴 정보 가져오기

```pascal
function TSMCMainForm.CallMainMenuInfoService: Boolean;
begin
  Result := False;
  
  // REST API 호출
  if GetMainMenu(MainGlobalVar.lginId,
                 TSHISDataSet(MainMemTable.MainMenuTable),
                 TSHISDataSet(MainMemTable.HomeMenuTable),
                 TSHISDataSet(MainMemTable.PatToolTable),
                 TSHISDataSet(MainMemTable.RoleTable),
                 TSHISDataSet(MainMemTable.ToolGroupTable),
                 TSHISDataSet(MainMemTable.MainAreaTable),
                 TSHISDataSet(MainMemTable.SubAreaTable)) then
  begin
    Result := True;
    
    // 메뉴 생성
    CreateMainMenu(MainMenuAll, MainMemTable.MainMenuTable);
    CreateMainMenu(MainMenuVisible, MainMemTable.MainMenuTable, False);
    
    // 부서 정보 설정
    if MainMemTable.RoleTable.Locate('mnsbYn', 'Y', []) then
    begin
      MainMsgHandler.UserId := MainMemTable.RoleTable.FieldByName('userId').AsString;
      MainMsgHandler.DprtCode := MainMemTable.RoleTable.FieldByName('abrvDprtCd').AsString;
      MainMsgHandler.DprtName := MainMemTable.RoleTable.FieldByName('kornDprtNm').AsString;
    end;
  end;
end;
```

##### `SetMainFormLayout` (메인 폼 레이아웃 설정)
**위치**: `Main.pas:1646`  
**역할**: 메인 화면 구성

**실행 순서**:
```pascal
procedure TSMCMainForm.SetMainFormLayout;
begin
  // 1. 메인메뉴 조회
  SMCMainForm.MainWaitCursor.SetDisplayText('메인메뉴 조회 중입니다.');
  CallMainMenuInfoService;
  
  // 2. 마스킹 프로그램 목록 조회
  GetMaskingProgList(TSHISDataSet(MainMemTable.UseMaskingProgTable));
  
  // 3. 설정 정보 읽기
  MainConfig.ReadWorkspaceInfos;      // 작업공간 정보
  MainConfig.ReadFavoriteInfos;       // 즐겨찾기
  MainConfig.ReadUserConfig;          // 사용자 설정
  MainConfig.ReadOpenedScrInfos;      // 열린 화면 정보
  
  // 4. 메모리 체크 설정
  SetMemCheck;
  
  // 5. 환자정보 화면 생성
  SMCMainForm.MainWaitCursor.SetDisplayText('환자정보 화면 생성 중입니다.');
  CreatePatInfoForm;
  
  // 6. 툴바 화면 생성
  SMCMainForm.MainWaitCursor.SetDisplayText('툴바화면 생성 중입니다.');
  LoadToolBarForm(MainMemTable.PatToolTable);
  
  // 7. 스피드버튼 툴바 조회
  SMCMainForm.MainWaitCursor.SetDisplayText('스피드툴바 조회 중입니다.');
  CallToolBarGroupInfoService;
  SetToolBarButton;
  
  // 8. 메인 레이아웃 설정
  SetMainLayout;
  
  // 9. 노티카운터 조회
  CallNotiCountService;
  
  // 10. 혈액 카운터 조회
  CallBloodCountService;
  
  // 11. 알람 목록 조회
  CallAlarmListService;
  
  // 12. 간호사 당일팀 조회
  MainGlobalVar.SetNurseInfo(GetNursTeamCode(MainGlobalVar.lginId));
end;
```

##### `CreateSMCForm` (폼 생성)
**위치**: `Main.pas:3554`  
**역할**: 동적으로 폼 생성 (BPL 패키지에서)

```pascal
function TSMCMainForm.CreateSMCForm(const AFormShowDestination: TFormShowDestination;
                                    const APackageName: string;
                                    AClassName: string = ''): TSMCForm;
var
  FormClass: TFormClass;
  hPackage: THandle;
  LSMCForm: TSMCForm;
begin
  Result := nil;
  
  // 1. 패키지 로드
  hPackage := LoadPackage(MainGlobalVar.Path.Bpl + APackageName);
  MainPackageHandleList.Add(Pointer(hPackage));
  
  // 2. 폼 클래스 찾기
  FormClass := TFormClass(GetClass(AClassName));
  if FormClass = nil then
    Exit;
  
  // 3. 폼 생성
  LSMCForm := TSMCForm(FormClass.Create(Application));
  
  // 4. 폼 속성 설정
  LSMCForm.Name := AClassName;
  LSMCForm.PackageName := APackageName;
  
  // 5. 도킹 처리
  case AFormShowDestination of
    fsdDockSite:
      LSMCForm.ManualDock(MainPageDivision.PageControl1, nil, alClient);
    fsdFloat:
      LSMCForm.Show;
    // 기타...
  end;
  
  Result := LSMCForm;
end;
```

##### `AppEventsShortCut` (단축키 처리)
**위치**: `Main.pas:882`  
**역할**: 전역 단축키 처리

**지원 단축키**:
- `Alt + F1`: 결함등록
- `Ctrl + Tab`: 페이지 전환
- `Shift + Ctrl + F2`: 열린화면 목록
- `Shift + Ctrl + F5`: 프로그램 관리
- `Shift + Ctrl + F6`: 프로그램개발요건 목록
- `Shift + Ctrl + F9`: 화면 비정상 처리 정보
- `F1`: 도움말
- `F10`: 메인메뉴 툴바 표시
- `Alt + 숫자`: 화면 실행

```pascal
procedure TSMCMainForm.AppEventsShortCut(var Msg: TWMKey; var Handled: Boolean);
var
  LShortCut: TShortCut;
  LSMCForm: TSMCForm;
begin
  Handled := False;
  
  // Alt + F1: 결함등록
  if (GetKeyState(VK_MENU) < 0) and (Msg.CharCode = VK_F1) then
  begin
    MainMsgHandler.ShowBugReg(LSMCForm);
    Handled := True;
    Exit;
  end;
  
  // Ctrl + Tab: 페이지 전환
  if (Msg.CharCode = VK_TAB) and (GetKeyState(VK_CONTROL) < 0) then
  begin
    LPageControl.SelectNextPage(GetKeyState(VK_SHIFT) >= 0);
    Handled := True;
    Exit;
  end;
  
  // Alt + 숫자: 화면 실행
  if (ssAlt in Shift) and (Msg.CharCode >= 48) then
  begin
    keyData := IntToStr(Msg.CharCode-48);
    keyData := MainGlobalVar.ShortCutList.FindForm(keyData, sctAlt);
    if keyData <> '' then
      CreateSMCFormWithLogin(keyData, 0);
  end;
end;
```

##### `DoMainMenuTimer` (메인 메뉴 타이머)
**위치**: `Main.pas:4106`  
**역할**: 주기적으로 메인 메뉴 업데이트

```pascal
procedure TSMCMainForm.DoMainMenuTimer(AForce: Boolean = False);
begin
  // 30초마다 또는 강제 실행 시
  if (AForce) or ((GetTickCount - MainMenuTimerTick) > 30000) then
  begin
    MainMenuTimerTick := GetTickCount;
    
    // 메인 메뉴 정보 재조회
    CallMainMenuInfoService;
    
    // 메뉴 재생성
    CreateMainMenu(MainMenuAll, MainMemTable.MainMenuTable);
    CreateMainMenu(MainMenuVisible, MainMemTable.MainMenuTable, False);
  end;
end;
```

##### `MainLogOut` (로그아웃)
**위치**: `Main.pas` (검색 필요)  
**역할**: 사용자 로그아웃 처리

```pascal
procedure TSMCMainForm.MainLogOut(AReLogin: Boolean = True);
begin
  // 1. 로그아웃 서비스 호출
  SendLogOutService;
  
  // 2. 열린 폼 모두 닫기
  CloseAllForm;
  
  // 3. 사용자 정보 정리
  FinalizeUserInfo(True);
  
  // 4. 재로그인 또는 종료
  if AReLogin then
  begin
    frmLogin.IsReLogin := True;
    frmLogin.ShowModal;
  end
  else
    Application.Terminate;
end;
```

---

### 3. dlgLogin.pas (로그인 폼)

**주요 기능**:

##### `FormCreate` (폼 생성)
**역할**: 로그인 폼 초기화

**주요 작업**:
1. 설정 파일 로드
2. 저장된 로그인 ID 불러오기
3. 마지막 접속 정보 표시

##### `btnLoginClick` (로그인 버튼 클릭)
**역할**: 사용자 인증

**실행 흐름**:
```pascal
procedure TfrmLogin.btnLoginClick(Sender: TObject);
begin
  // 1. 입력값 검증
  if not CheckCanLogin then
    Exit;
  
  // 2. 사용자 인증
  if not GetUserValidation(edtLoginId.Text, edtLoginPassword.Text, AUserId, AResultTable) then
  begin
    SMCShowMessage('IC03040');  // 로그인 실패
    Exit;
  end;
  
  // 3. 세션 생성
  NewSessionLogin;
  
  // 4. 사용자 정보 조회
  if not GetUserInfo(edtLoginId.Text, '', UserInfoTable, ...) then
    Exit;
  
  // 5. 로그인 성공
  ModalResult := mrOk;
end;
```

##### `GetUserValidation` (사용자 인증)
**역할**: 서버에 사용자 인증 요청

```pascal
function TfrmLogin.GetUserValidation(const ALoginId, ALoginPwd: string;
                                     var AUserId: string;
                                     var AResultTable: TSHISDataSet): Boolean;
begin
  Result := False;
  
  // REST API 호출
  dp_LogiohtS01.Request.Clear;
  dp_LogiohtS01.Request.FieldByName('lginId').AsString := ALoginId;
  dp_LogiohtS01.Request.FieldByName('lginPswd').AsString := ALoginPwd;
  
  try
    dp_LogiohtS01.Start(Self);
    
    if not dt_LogiohtS01.IsEmpty then
    begin
      AUserId := dt_LogiohtS01.FieldByName('userId').AsString;
      AResultTable := dt_LogiohtS01;
      Result := True;
    end;
  except
    on E: Exception do
      SMCShowMessage(E.Message);
  end;
end;
```

---

### 4. MainHandler.pas (메인 핸들러)

**역할**: 메인 폼의 이벤트 및 메시지 처리

**주요 기능**:
- 폼 생성/관리
- 메뉴 처리
- 알림 처리
- 예외 처리

---

### 5. Common.pas (공통 함수)

**주요 함수들**:

##### `CreateMainMenu` (메인 메뉴 생성)
**위치**: `Common.pas:568`  
**역할**: 데이터셋에서 메인 메뉴 생성

```pascal
procedure CreateMainMenu(AMenu: TMainMenu; AMenuTable: TDataSet; AIncludeHide: Boolean);
var
  LMenuItem: TSMCMainMenuItem;
begin
  AMenu.Items.Clear;
  
  with TkbmMemTable(AMenuTable) do
  begin
    DisableControls;
    SortOn('inqrSqnc', []);  // 순번으로 정렬
    
    // ROOT 메뉴 찾기
    First;
    while not EOF do
    begin
      if Fields[iParentMenuId].AsString = 'ROOT' then
      begin
        LMenuItem := CreateMenuItem;
        if LMenuItem <> nil then
          AMenu.Items.Add(LMenuItem);
      end;
      Next;
    end;
    
    // 하위 메뉴 추가
    for I := 0 to AMenu.Items.Count - 1 do
      AddMenuItem(TSMCMainMenuItem(AMenu.Items[I]));
  end;
end;
```

##### `GetMenuItemInfo` (메뉴 항목 정보 조회)
**위치**: `Common.pas:433`  
**역할**: 메뉴 그룹ID와 메뉴ID로 메뉴 정보 조회

**반환 정보**:
- `MenuGroupId`: 메뉴 그룹 ID
- `MenuId`: 메뉴 ID
- `MenuType`: 메뉴 형태 (F: 프로그램, M: 순수메뉴, S: 구분선)
- `ProgId`: 프로그램 ID
- `Caption`: 메뉴명
- `DupExec`: 중복 실행 가능 여부
- `ShowModal`: 모달 표시 여부
- `ExecScope`: 실행 영역 코드
- `Visible`: 표시 여부
- `Enable`: 활성화 여부

---

## 🔄 MAIN 프로젝트 실행 흐름

```
1. LOGIN에서 Nexmed_EHR.exe 실행
   ↓
2. Nexmed_EHR.dpr 시작
   ├─ Application.Initialize
   ├─ TSMCMainForm 생성
   ├─ LoadMainComPack (필수 패키지 로드)
   ├─ LoadComPack (기타 패키지 로드)
   └─ TfrmLogin 생성
   ↓
3. 로그인 화면 표시 (frmLogin.ShowModal)
   ├─ FormCreate (초기화)
   ├─ 사용자 입력 대기
   └─ btnLoginClick (인증)
   ↓
4. 로그인 성공 (ModalResult = mrOk)
   ↓
5. Application.Run
   ↓
6. SMCFormCreate (메인 폼 초기화)
   ├─ 전역 변수 초기화
   ├─ 경로 설정
   ├─ 마스터 파일 로드
   ├─ 타이머 설정
   └─ 환자 정보/목록 폼 생성
   ↓
7. SMCFormShowComplete
   ├─ InitializeUserInfo
   ├─ SetMainFormLayout
   │   ├─ CallMainMenuInfoService (메뉴 조회)
   │   ├─ CreatePatInfoForm (환자 정보)
   │   ├─ LoadToolBarForm (툴바)
   │   ├─ SetMainLayout (레이아웃)
   │   └─ CallNotiCountService (알림)
   └─ SetMainShowComplete
   ↓
8. 사용자 작업
   ├─ 메뉴 클릭 → CreateSMCForm (폼 생성)
   ├─ 환자 선택 → 환자 정보 표시
   ├─ 데이터 입력/조회 → REST API 호출
   └─ 리포트 생성/출력
   ↓
9. 종료
   ├─ MainLogOut (로그아웃)
   ├─ CloseAllForm (폼 닫기)
   ├─ FinalizeUserInfo (사용자 정보 정리)
   └─ Application.Terminate
```

---

## 📊 주요 데이터 구조

### 버전 정보 테이블 (LocalVerTable)

| 필드명 | 타입 | 설명 |
|--------|------|------|
| `cfrtPrgmId` | String(600) | 파일명 |
| `prgmDscrCtn` | String(300) | 파일 설명 |
| `prgmSizeCnt` | String(11) | 파일 크기 |
| `devpVrsnNm` | String(5) | 개발 버전 |
| `ceckPhspDt` | String(14) | 수정 시간 (yyyymmddhhnnss) |
| `dsrbLctnNm` | String(600) | 저장 위치 |
| `comnYn` | String(1) | 공통 파일 여부 |

### 메뉴 정보 테이블 (MainMenuTable)

| 필드명 | 설명 |
|--------|------|
| `menuGrpId` | 메뉴 그룹 ID |
| `menuId` | 메뉴 ID |
| `hgrnMenuId` | 상위 메뉴 ID (ROOT면 최상위) |
| `menuFrmtCd` | 메뉴 형태 (F/M/S) |
| `prgmId` | 프로그램 ID |
| `bsisLnggMenuNm` | 메뉴명 |
| `dplcExctPsblYn` | 중복 실행 가능 여부 |
| `swmdYn` | ShowModal 여부 |
| `exctScopCd` | 실행 영역 코드 |
| `inqrSqnc` | 순번 |

---

## 🔐 보안 관련 상세

### 1. 통신 보안

#### HTTPS 설정
```pascal
// Loader.pas:1503
if SameText(IsSSL, 'Y') then
begin
  IS_TRANSMIT_SSL := True;
  SSLHandler := TIdSSLIOHandlerSocketOpenSSL.Create(IdHTTPDownload);
  SSLHandler.SSLOptions.Method := sslvTLSv1_2;  // TLS 1.2 사용
  SSLHandler.SSLOptions.VerifyMode := [];       // 인증서 검증 생략 (내부망)
  IdHTTPDownload.IOHandler := SSLHandler;
end;
```

#### 데이터 압축
```pascal
// Loader.pas:1493
if SameText(IsCompress, 'Y') then
begin
  SEHR.DataSet.RestClient.UseCompress := True;
  IdHTTPDownload.Compressor := TIdCompressorZLib.Create(IdHTTPDownload);
  IdHTTPDownload.Request.AcceptEncoding := 'gzip, deflate';
end;
```

### 2. 파일 무결성

#### 버전 비교 알고리즘
```pascal
// Loader.pas:917
LFileLastModTime := GetFileLastModTime(sLocalFile);

if LFileLastModTime = 0 then
  LState := dsInsert  // 파일 없음
else if LocalVerTable.Locate('cfrtPrgmId', sProgId, []) then
begin
  // 서버 수정시간과 로컬 수정시간 비교
  if (서버수정시간 = 로컬수정시간) and
     (서버수정시간 = FormatDateTime('yyyymmddhhnnss', LFileLastModTime)) then
    LState := dsBrowse  // 동일
  else
    LState := dsEdit;   // 업데이트 필요
end;
```

### 3. 프로세스 보안

#### 중복 실행 방지
```pascal
// Loader.pas:442
function MainAliveCheck(const AFileName, AClassName: string; KillMain: Boolean): boolean;
begin
  // 프로세스 목록 조회
  LSnapProcHandle := CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
  
  while LNextProc do
  begin
    if SameText(ExtractFileName(LFileName), ExtractFileName(AFileName)) then
    begin
      // 윈도우 핸들 찾기
      LAppWnd := GetWndFromPid(LProcEntry.th32ProcessID, AClassName);
      
      // 응답 확인
      if IsWindowResponding(LAppWnd, 1000) then
      begin
        if KillMain then
          TaskKill(Format(' /PID %d', [LProcEntry.th32ProcessID]), SW_HIDE, True);
        Result := True;
        Break;
      end;
    end;
  end;
end;
```

---

## 📝 주요 상수 및 설정 파일

### LOGIN 프로젝트

| 상수 | 값 | 설명 |
|------|-----|------|
| `INI_DevMan` | 'DevMan.ini' | 개발 관리 설정 파일 |
| `INI_Config` | 'Config.ini' | 서버 설정 파일 |
| `VER_Common` | 'VerCommon.dat' | 버전 정보 파일 |
| `DAT_ExeAppList` | 'ExeAppList.dat' | 애플리케이션 목록 |
| `DAT_FileDownInfo` | 'FileDownInfo.dat' | 파일 다운로드 경로 정보 |
| `DAT_RegisterOcxDll` | 'RegisterOcxDll.dat' | OCX/DLL 등록 목록 |
| `MAIN_EXEFILENAME` | 'Nexmed_EHR.exe' | 메인 실행 파일 |
| `MAIN_CLASSNAME` | 'TSMCMainForm' | 메인 폼 클래스명 |

### MAIN 프로젝트

| 상수 | 값 | 설명 |
|------|-----|------|
| `TIMER_MAINMENU` | 0 | 메인메뉴 타이머 ID |
| `TIMER_PATIENT` | 1 | 환자 타이머 ID |
| `TIMER_CONNECTINFO` | 2 | 연결 정보 타이머 ID |
| `TIMER_ALARM` | 3 | 알람 타이머 ID |
| `TIMER_NOTI` | 5 | 알림 타이머 ID |
| `FILE_MSG_CODE` | 'msg_%s.dat' | 메시지 코드 파일 |
| `FILE_COM_CODE` | 'ComCode.dat' | 공통 코드 파일 |
| `FILE_DEPT` | 'Dept.dat' | 부서 정보 파일 |

---

## 🎯 핵심 알고리즘

### 1. 파일 버전 비교 알고리즘

```
1. 서버에서 파일 목록 조회
   ↓
2. 각 파일에 대해:
   ├─ 로컬 파일 존재 여부 확인
   ├─ 로컬 버전 정보 조회
   ├─ 서버 수정시간 vs 로컬 수정시간 비교
   └─ 다르면 다운로드 목록에 추가
   ↓
3. 다운로드 목록 순회:
   ├─ HTTP/HTTPS로 파일 다운로드
   ├─ LZMA 압축 해제 (필요시)
   ├─ 임시 파일을 최종 위치로 복사
   ├─ 파일 수정시간 설정
   └─ 로컬 버전 정보 업데이트
```

### 2. 동적 폼 로드 알고리즘

```
1. 메뉴 클릭 또는 프로그램 ID로 폼 요청
   ↓
2. 패키지명 확인 (ProgId → PackageName 매핑)
   ↓
3. 패키지 로드 여부 확인
   ├─ 이미 로드됨 → 스킵
   └─ 미로드 → LoadPackage 호출
   ↓
4. 폼 클래스 찾기 (GetClass)
   ↓
5. 폼 인스턴스 생성
   ↓
6. 폼 속성 설정 및 표시
   ├─ 도킹 (fsdDockSite)
   ├─ 플로팅 (fsdFloat)
   └─ 모달 (fsdModal)
```

### 3. 메뉴 생성 알고리즘

```
1. 서버에서 메뉴 정보 조회
   ↓
2. 데이터셋 정렬 (inqrSqnc 기준)
   ↓
3. ROOT 메뉴 찾기 (hgrnMenuId = 'ROOT')
   ↓
4. 각 ROOT 메뉴에 대해:
   ├─ TSMCMainMenuItem 생성
   ├─ 속성 설정 (Caption, ProgId 등)
   └─ 하위 메뉴 재귀적으로 추가
   ↓
5. 메뉴 표시
```

---

## 📌 주요 함수 요약표

### LOGIN 프로젝트

| 함수명 | 위치 | 역할 |
|--------|------|------|
| `SMCFormCreate` | Loader.pas:1202 | 폼 초기화 |
| `Load2Config` | Loader.pas:1432 | 설정 파일 로드 |
| `GetCommonFileList` | Loader.pas:2197 | 서버 파일 목록 조회 |
| `CommonFileVersionCheck` | Loader.pas:2240 | 버전 체크 시작 |
| `CommonFileDownload` | Loader.pas:617 | 파일 다운로드 |
| `RegisterOcxDll` | Loader.pas:2128 | OCX/DLL 등록 |
| `KillProcessApp` | Loader.pas:1674 | 프로세스 종료 |
| `LoadVersionInfo` | Loader.pas:2163 | 버전 정보 로드 |
| `SaveVersionInfo` | Loader.pas:2181 | 버전 정보 저장 |
| `MainAliveCheck` | Loader.pas:442 | 메인 프로그램 실행 확인 |

### MAIN 프로젝트

| 함수명 | 위치 | 역할 |
|--------|------|------|
| `SMCFormCreate` | Main.pas:2814 | 메인 폼 초기화 |
| `LoadMainComPack` | Main.pas:2333 | 필수 패키지 로드 |
| `LoadComPack` | Main.pas:2426 | 기타 패키지 로드 |
| `CallMainMenuInfoService` | Main.pas:1245 | 메뉴 정보 조회 |
| `SetMainFormLayout` | Main.pas:1646 | 메인 레이아웃 설정 |
| `CreateSMCForm` | Main.pas:3554 | 동적 폼 생성 |
| `AppEventsShortCut` | Main.pas:882 | 단축키 처리 |
| `DoMainMenuTimer` | Main.pas:4106 | 메뉴 타이머 |
| `MainLogOut` | Main.pas | 로그아웃 |
| `SetGlobalVar` | Main.pas:1385 | 전역 변수 설정 |

---

**작성일**: 2024년  
**버전**: 1.0  
**상세도**: 매우 상세 (함수 단위까지)

