# VS Code 오프라인 Extension 다운로드 및 설치 가이드

Java/Spring 개발, 코드 품질, Git, YAML/XML, Markdown/Mermaid,
Container/Kubernetes 개발에 유용한 VS Code Extension을 **인터넷이 가능한
장비에서 VSIX 파일로 다운로드**한 뒤 **폐쇄망/오프라인 장비에 일괄
설치**하는 방법을 설명합니다.

------------------------------------------------------------------------

## 1. 전체 구성

``` text
인터넷 가능 PC
    │
    ├─ download-vscode-extensions.ps1
    │
    ▼
extensions/*.vsix
    │
    │ 보안 반입 / 파일 복사
    ▼
폐쇄망 PC
    │
    ├─ install-vscode-extensions.ps1
    │
    ▼
VS Code + Java/Spring/Markdown/Mermaid 개발환경
```

### 포함 Extension

  -----------------------------------------------------------------------------------------------
  구분                    Extension ID                                    용도
  ----------------------- ----------------------------------------------- -----------------------
  Java                    `redhat.java`                                   Java Language Support

  Java                    `vscjava.vscode-java-debug`                     Java Debugger

  Java                    `vscjava.vscode-java-test`                      Java Test Runner

  Java                    `vscjava.vscode-maven`                          Maven

  Java                    `vscjava.vscode-gradle`                         Gradle

  Java                    `vscjava.vscode-java-dependency`                Java Project Manager

  Spring                  `vmware.vscode-spring-boot`                     Spring Boot Tools

  Spring                  `vscjava.vscode-spring-initializr`              Spring Initializr

  Spring                  `vscjava.vscode-spring-boot-dashboard`          Spring Boot Dashboard

  품질                    `SonarSource.sonarlint-vscode`                  SonarQube for IDE

  품질                    `shengchen.vscode-checkstyle`                   Checkstyle

  Git                     `eamodio.gitlens`                               GitLens

  편의                    `usernamehw.errorlens`                          Error Lens

  설정                    `redhat.vscode-yaml`                            YAML

  설정                    `redhat.vscode-xml`                             XML

  Markdown                `yzhang.markdown-all-in-one`                    Markdown 편집

  Markdown                `DavidAnson.vscode-markdownlint`                Markdown lint

  Mermaid                 `bierner.markdown-mermaid`                      Markdown Preview
                                                                          Mermaid

  Markdown                `shd101wyy.markdown-preview-enhanced`           Enhanced Preview

  Mermaid                 `tomoyukim.vscode-mermaid-editor`               Mermaid Editor

  Markdown                `yzane.markdown-pdf`                            Markdown PDF

  Markdown                `mushan.vscode-paste-image`                     Paste Image

  Container               `ms-azuretools.vscode-containers`               Container Tools

  Kubernetes              `ms-kubernetes-tools.vscode-kubernetes-tools`   Kubernetes
  -----------------------------------------------------------------------------------------------

------------------------------------------------------------------------

## 2. 권장 디렉터리 구조

온라인 장비에서 다음과 같이 준비합니다.

``` text
C:\vscode-offline\
├─ download-vscode-extensions.ps1
├─ install-vscode-extensions.ps1
└─ extensions\
```

다운로드 후 `extensions` 디렉터리에 VSIX 파일들이 생성됩니다.

------------------------------------------------------------------------

# 3. Extension 다운로드

## 3.1 다운로드 스크립트

다음 내용을 `download-vscode-extensions.ps1` 파일로 저장합니다.

``` powershell
param(
    [string]$OutputDir = ".\extensions",

    # 예:
    # win32-x64
    # win32-arm64
    # linux-x64
    # linux-arm64
    # darwin-x64
    # darwin-arm64
    #
    # 빈 문자열이면 universal package를 요청합니다.
    [string]$TargetPlatform = ""
)

$ErrorActionPreference = "Continue"

$extensions = @(

    # Java
    "redhat.java",
    "vscjava.vscode-java-debug",
    "vscjava.vscode-java-test",
    "vscjava.vscode-maven",
    "vscjava.vscode-gradle",
    "vscjava.vscode-java-dependency",

    # Spring
    "vmware.vscode-spring-boot",
    "vscjava.vscode-spring-initializr",
    "vscjava.vscode-spring-boot-dashboard",

    # Code Quality
    "SonarSource.sonarlint-vscode",
    "shengchen.vscode-checkstyle",

    # Git
    "eamodio.gitlens",

    # Editor / Configuration
    "usernamehw.errorlens",
    "redhat.vscode-yaml",
    "redhat.vscode-xml",

    # Markdown / Mermaid
    "yzhang.markdown-all-in-one",
    "DavidAnson.vscode-markdownlint",
    "bierner.markdown-mermaid",
    "shd101wyy.markdown-preview-enhanced",
    "tomoyukim.vscode-mermaid-editor",
    "yzane.markdown-pdf",
    "mushan.vscode-paste-image",

    # Container / Kubernetes
    "ms-azuretools.vscode-containers",
    "ms-kubernetes-tools.vscode-kubernetes-tools"
)

if (-not (Test-Path $OutputDir)) {
    New-Item -ItemType Directory -Force -Path $OutputDir | Out-Null
}

function Download-Extension {

    param(
        [Parameter(Mandatory = $true)]
        [string]$ExtensionId
    )

    $index = $ExtensionId.IndexOf(".")

    if ($index -lt 1) {
        Write-Warning "Invalid Extension ID: $ExtensionId"
        return $false
    }

    $publisher = $ExtensionId.Substring(0, $index)
    $extension = $ExtensionId.Substring($index + 1)

    $baseUrl =
        "https://marketplace.visualstudio.com/_apis/public/gallery/" +
        "publishers/$publisher/vsextensions/$extension/latest/vspackage"

    if ([string]::IsNullOrWhiteSpace($TargetPlatform)) {
        $url = $baseUrl
    }
    else {
        $url = "${baseUrl}?targetPlatform=${TargetPlatform}"
    }

    $fileName = "$ExtensionId.vsix"
    $outputFile = Join-Path $OutputDir $fileName

    Write-Host ""
    Write-Host "============================================================"
    Write-Host "Extension : $ExtensionId"
    Write-Host "Platform  : $TargetPlatform"
    Write-Host "Output    : $outputFile"
    Write-Host "============================================================"

    try {
        Invoke-WebRequest `
            -Uri $url `
            -OutFile $outputFile `
            -UseBasicParsing

        Write-Host "[OK] $ExtensionId"
        return $true
    }
    catch {
        Write-Warning "Download failed: $ExtensionId"
        Write-Warning $_.Exception.Message

        # 플랫폼별 패키지가 없으면 universal package 재시도
        if (-not [string]::IsNullOrWhiteSpace($TargetPlatform)) {

            Write-Host "Retrying universal package..."

            try {
                Invoke-WebRequest `
                    -Uri $baseUrl `
                    -OutFile $outputFile `
                    -UseBasicParsing

                Write-Host "[OK] $ExtensionId (universal)"
                return $true
            }
            catch {
                Write-Warning "[FAIL] $ExtensionId"
                Write-Warning $_.Exception.Message

                if (Test-Path $outputFile) {
                    Remove-Item $outputFile -Force
                }

                return $false
            }
        }

        if (Test-Path $outputFile) {
            Remove-Item $outputFile -Force
        }

        return $false
    }
}

Write-Host ""
Write-Host "============================================================"
Write-Host " VS Code Offline Extension Downloader"
Write-Host "============================================================"
Write-Host "Output Directory : $OutputDir"

if ([string]::IsNullOrWhiteSpace($TargetPlatform)) {
    Write-Host "Target Platform  : universal"
}
else {
    Write-Host "Target Platform  : $TargetPlatform"
}

Write-Host "Extension Count  : $($extensions.Count)"
Write-Host ""

$success = 0
$failed = 0
$failedExtensions = @()

foreach ($extension in $extensions) {

    $result = Download-Extension $extension

    if ($result) {
        $success++
    }
    else {
        $failed++
        $failedExtensions += $extension
    }
}

Write-Host ""
Write-Host "============================================================"
Write-Host " Download Result"
Write-Host "============================================================"
Write-Host "SUCCESS : $success"
Write-Host "FAILED  : $failed"

if ($failedExtensions.Count -gt 0) {
    Write-Host ""
    Write-Host "Failed Extensions:"

    foreach ($extension in $failedExtensions) {
        Write-Host "  $extension"
    }
}

Write-Host ""
Write-Host "Downloaded VSIX Files"
Write-Host "------------------------------------------------------------"

Get-ChildItem $OutputDir -Filter "*.vsix" |
    Sort-Object Name |
    Select-Object Name,
        @{Name = "Size(MB)";
          Expression = {[math]::Round($_.Length / 1MB, 2)}} |
    Format-Table -AutoSize

Write-Host ""
Write-Host "============================================================"
Write-Host "Download completed."
Write-Host "============================================================"
```

------------------------------------------------------------------------

## 3.2 PowerShell 실행 정책 설정

PowerShell 스크립트 실행이 차단된 경우 현재 PowerShell 프로세스에
대해서만 임시로 허용합니다.

``` powershell
Set-ExecutionPolicy -Scope Process Bypass
```

이 설정은 현재 PowerShell 프로세스가 종료되면 유지되지 않습니다.

------------------------------------------------------------------------

## 3.3 Windows x64용 다운로드

일반적인 Intel/AMD 기반 Windows PC라면 다음과 같이 실행합니다.

``` powershell
cd C:\vscode-offline

.\download-vscode-extensions.ps1 `
    -OutputDir C:\vscode-offline\extensions `
    -TargetPlatform win32-x64
```

------------------------------------------------------------------------

## 3.4 Apple Silicon Mac용 다운로드

Apple Silicon(M1/M2/M3/M4 등) Mac용 VSIX가 필요한 경우:

``` powershell
.\download-vscode-extensions.ps1 `
    -OutputDir C:\vscode-offline\extensions `
    -TargetPlatform darwin-arm64
```

------------------------------------------------------------------------

## 3.5 Linux x64용 다운로드

``` powershell
.\download-vscode-extensions.ps1 `
    -OutputDir C:\vscode-offline\extensions `
    -TargetPlatform linux-x64
```

------------------------------------------------------------------------

## 3.6 Universal 패키지 다운로드

플랫폼을 지정하지 않으려면 `TargetPlatform`을 생략합니다.

``` powershell
.\download-vscode-extensions.ps1 `
    -OutputDir C:\vscode-offline\extensions
```

단, Extension에 플랫폼별 네이티브 구성요소가 있는 경우 대상 플랫폼을
명시하는 방식을 권장합니다.

------------------------------------------------------------------------

# 4. 폐쇄망으로 파일 반입

다운로드가 완료되면 다음 디렉터리 전체를 폐쇄망 장비로 복사합니다.

``` text
C:\vscode-offline\
├─ install-vscode-extensions.ps1
└─ extensions\
   ├─ redhat.java.vsix
   ├─ vscjava.vscode-java-debug.vsix
   ├─ vscjava.vscode-java-test.vsix
   ├─ ...
   ├─ bierner.markdown-mermaid.vsix
   └─ ms-kubernetes-tools.vscode-kubernetes-tools.vsix
```

조직의 보안 정책에 따라 USB, 보안 파일 전송 시스템, 망간 자료전송 시스템
등의 승인된 방법을 사용합니다.

------------------------------------------------------------------------

# 5. 오프라인 장비에서 Extension 설치

## 5.1 설치 스크립트

다음 내용을 `install-vscode-extensions.ps1` 파일로 저장합니다.

``` powershell
param(
    [string]$ExtensionDir = ".\extensions",
    [string]$CodeCommand = "code"
)

$ErrorActionPreference = "Continue"

if (-not (Test-Path $ExtensionDir)) {
    Write-Error "Extension directory not found: $ExtensionDir"
    exit 1
}

$files = @(
    Get-ChildItem `
        -Path $ExtensionDir `
        -Filter "*.vsix" |
        Sort-Object Name
)

if ($files.Count -eq 0) {
    Write-Error "No VSIX files found."
    exit 1
}

Write-Host ""
Write-Host "============================================================"
Write-Host " VS Code Offline Extension Installer"
Write-Host "============================================================"
Write-Host "Directory : $ExtensionDir"
Write-Host "Count     : $($files.Count)"
Write-Host ""

$success = 0
$failed = 0
$failedFiles = @()

foreach ($file in $files) {

    Write-Host ""
    Write-Host "------------------------------------------------------------"
    Write-Host "Installing: $($file.Name)"
    Write-Host "------------------------------------------------------------"

    & $CodeCommand `
        --install-extension $file.FullName `
        --force

    if ($LASTEXITCODE -eq 0) {
        Write-Host "[OK] $($file.Name)"
        $success++
    }
    else {
        Write-Warning "[FAIL] $($file.Name)"
        $failed++
        $failedFiles += $file.Name
    }
}

Write-Host ""
Write-Host "============================================================"
Write-Host " Installation Result"
Write-Host "============================================================"
Write-Host "SUCCESS : $success"
Write-Host "FAILED  : $failed"

if ($failedFiles.Count -gt 0) {

    Write-Host ""
    Write-Host "Failed Extensions:"

    foreach ($file in $failedFiles) {
        Write-Host "  $file"
    }
}

Write-Host ""
Write-Host "============================================================"
Write-Host " Installed Extensions"
Write-Host "============================================================"
Write-Host ""

& $CodeCommand --list-extensions --show-versions
```

------------------------------------------------------------------------

## 5.2 폐쇄망 장비에서 설치 실행

먼저 PowerShell을 실행하고:

``` powershell
cd C:\vscode-offline
```

필요한 경우 현재 프로세스에서 PowerShell 스크립트 실행을 허용합니다.

``` powershell
Set-ExecutionPolicy -Scope Process Bypass
```

Extension을 일괄 설치합니다.

``` powershell
.\install-vscode-extensions.ps1 `
    -ExtensionDir C:\vscode-offline\extensions
```

------------------------------------------------------------------------

# 6. 설치 결과 확인

설치된 Extension과 버전을 확인합니다.

``` powershell
code --list-extensions --show-versions
```

예:

``` text
redhat.java@...
vscjava.vscode-java-debug@...
vscjava.vscode-maven@...
vmware.vscode-spring-boot@...
eamodio.gitlens@...
yzhang.markdown-all-in-one@...
bierner.markdown-mermaid@...
...
```

------------------------------------------------------------------------

# 7. `code` 명령을 찾지 못하는 경우

다음과 같은 오류가 발생할 수 있습니다.

``` text
code : The term 'code' is not recognized...
```

VS Code의 `bin` 디렉터리가 PATH에 등록되어 있는지 확인합니다.

일반적인 사용자 설치 경로 예시는 다음과 같습니다.

``` text
C:\Users\<사용자>\AppData\Local\Programs\Microsoft VS Code\bin
```

PATH를 수정하기 어려운 환경에서는 설치 스크립트의 `CodeCommand`에
`code.cmd`의 전체 경로를 지정할 수도 있습니다.

예:

``` powershell
.\install-vscode-extensions.ps1 `
    -ExtensionDir C:\vscode-offline\extensions `
    -CodeCommand "C:\Users\<사용자>\AppData\Local\Programs\Microsoft VS Code\bin\code.cmd"
```

실제 설치 위치는 VS Code 설치 방식에 따라 다를 수 있으므로 해당 장비에서
확인해야 합니다.

------------------------------------------------------------------------

# 8. 개별 VSIX 수동 설치

일괄 설치가 아닌 특정 Extension 하나만 설치하려면:

``` powershell
code --install-extension C:\vscode-offline\extensions\redhat.java.vsix
```

강제로 재설치하려면:

``` powershell
code --install-extension C:\vscode-offline\extensions\redhat.java.vsix --force
```

VS Code GUI에서도 설치할 수 있습니다.

``` text
VS Code
  → Extensions
  → ...
  → Install from VSIX...
  → VSIX 파일 선택
```

------------------------------------------------------------------------

# 9. 폐쇄망 환경 주의사항

## VSIX 설치와 완전한 오프라인 동작은 다름

Extension의 VSIX 설치가 성공하더라도 일부 Extension은 실행 과정에서 추가
파일이나 외부 서비스를 사용할 수 있습니다.

대표적으로 다음 항목을 사전에 검증하는 것이 좋습니다.

-   Java Extension 및 Java Language Server 동작
-   Spring 관련 Extension의 외부 metadata 접근 여부
-   YAML schema 외부 조회
-   XML Language Server 관련 구성
-   Markdown Preview/PDF 관련 외부 바이너리
-   Kubernetes의 `kubectl`
-   Helm을 사용하는 경우 `helm`
-   Container Tools가 사용하는 Docker/Podman 등의 CLI

따라서 실제 폐쇄망 표준 이미지에 배포하기 전에 인터넷이 완전히 차단된
테스트 장비에서 검증하는 것을 권장합니다.

------------------------------------------------------------------------

# 10. Java 폐쇄망 개발환경 권장 구성

VSIX만 반입하기보다는 Java 개발에 필요한 도구를 함께 표준화하는 것이
좋습니다.

``` text
java-dev-offline/
│
├─ vscode/
│   └─ VS Code 설치 파일
│
├─ extensions/
│   ├─ redhat.java.vsix
│   ├─ vscjava.vscode-java-debug.vsix
│   ├─ ...
│   └─ bierner.markdown-mermaid.vsix
│
├─ jdk/
│   └─ jdk-21/
│
├─ maven/
│   └─ apache-maven-3.9.x/
│
├─ config/
│   ├─ settings.json
│   ├─ settings.xml
│   └─ checkstyle.xml
│
└─ scripts/
    ├─ download-vscode-extensions.ps1
    └─ install-vscode-extensions.ps1
```

Maven dependency는 폐쇄망 내부의 Nexus/Artifactory 등의 Maven
Repository를 이용하도록 `settings.xml`을 구성하는 것이 좋습니다.

------------------------------------------------------------------------

# 11. 운영 환경에서는 버전 고정 권장

현재 다운로드 스크립트는 Marketplace의 `latest` 버전을 가져옵니다.

따라서 스크립트를 실행하는 날짜에 따라 다운로드되는 Extension 버전이
달라질 수 있습니다.

개인 개발환경에서는 큰 문제가 없지만 여러 개발자에게 동일한 개발환경을
배포하는 기업 환경이라면 다음 정보를 별도로 관리하는 것을 권장합니다.

``` text
Extension ID
Extension Version
Target Platform
VSIX File Name
SHA-256
승인/반입 일자
```

예:

``` text
redhat.java
vscjava.vscode-java-debug
vscjava.vscode-maven
vmware.vscode-spring-boot
bierner.markdown-mermaid
...
```

이렇게 관리하면 Extension 업그레이드 전후 비교, 보안 검증, 장애 발생 시
버전 추적, 개발자 간 동일 환경 재현이 쉬워집니다.
