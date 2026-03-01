## 1. Set Active Power Plan

``` bat
powercfg /setactive 381b4222-f694-41f0-9685-ff5bb260df2e
```

------------------------------------------------------------------------

## 2. Resync Windows Time (if available)

``` bat
WHERE w32tm
if %ERRORLEVEL% EQU 0 (
    w32tm /config /update
    w32tm /resync
)
```

------------------------------------------------------------------------

## 3. (Optional) Clear Clipboard

``` bat
@REM echo off | clip
```

------------------------------------------------------------------------

## 4. Registry Tweaks

### 4.1 Check for `reg` command

``` bat
WHERE reg
if %ERRORLEVEL% EQU 0 (
    del /s /q /f "%USERPROFILE%\Downloads\tweaks.reg"

    WHERE curl 
    if %ERRORLEVEL% EQU 0 (
        curl --remote-time -C - -Lo "%USERPROFILE%\Downloads\tweaks.reg" https://raw.githubusercontent.com/Zerohazard8x/scripts/main/tweaks.reg
    ) else (
        WHERE wget 
        if %ERRORLEVEL% EQU 0 (
            wget -c --timestamping -O "%USERPROFILE%\Downloads\tweaks.reg" https://raw.githubusercontent.com/Zerohazard8x/scripts/main/tweaks.reg
        )
    )

    reg import "%USERPROFILE%\Downloads\tweaks.reg"
)
```

------------------------------------------------------------------------

## 5. Reset Group Policy

``` bat
RD /S /Q "%windir%\System32\GroupPolicyUsers"
RD /S /Q "%windir%\System32\GroupPolicy"
gpupdate /force
```

------------------------------------------------------------------------

## 6. Reset Security Policy to Default

``` bat
secedit /configure /cfg %windir%\inf\defltbase.inf /db defltbase.sdb /verbose
```

------------------------------------------------------------------------

## 7. Reset WMI Repository (Password / Policy Issues)

``` bat
net stop winmgmt
winmgmt /resetrepository
net start winmgmt

@REM if fails
@REM winmgmt /salvagerepository
```

------------------------------------------------------------------------

## 8. Reset Windows Search

### 8.1 Stop Explorer & Search Services

``` bat
taskkill /f /im explorer.exe
net stop "Windows Search"
taskkill /f /im SearchFilterHost.exe
taskkill /f /im SearchHost.exe
taskkill /f /im SearchIndexer.exe
taskkill /f /im SearchProtocolHost.exe
taskkill /f /im ShellExperienceHost.exe
taskkill /f /im StartMenuExperienceHost.exe
```

### 8.2 Windows 11 Database Reset

``` bat
del "%ProgramData%\Microsoft\Search\Data\Applications\Windows\Windows.db"
ren "%LocalAppData%\Packages\MicrosoftWindows.Client.CBS_cw5n1h2txyewy\LocalState" LocalState.old
```

### 8.3 Windows 10 Database Reset

``` bat
del "%ProgramData%\Microsoft\Search\Data\Applications\Windows\Windows.edb"
ren "%LocalAppData%\Packages\Microsoft.Windows.Search_cw5n1h2txyewy\LocalState" LocalState.old
```

### 8.4 Restart Search Service

``` bat
net start "Windows Search"
```

------------------------------------------------------------------------

## 9. Restore Default Power Schemes

``` bat
powercfg -restoredefaultschemes
```

------------------------------------------------------------------------

## 10. Power Timeout Settings

### 10.1 Plugged In (AC)

``` bat
:: NEVER turn off display
powercfg /change monitor-timeout-ac 0

:: NEVER sleep
powercfg /change standby-timeout-ac 0
```

### 10.2 On Battery (DC)

``` bat
:: NEVER turn off display
powercfg /change monitor-timeout-dc 0
```
