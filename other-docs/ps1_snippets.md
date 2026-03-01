## 1. DNS over HTTPS (DoH) Reference Servers

### Google DNS

``` powershell
2001:4860:4860::8888 -DohTemplate https://dns.google/dns-query
2001:4860:4860::8844 -DohTemplate https://dns.google/dns-query
8.8.8.8 -DohTemplate https://dns.google/dns-query
8.8.4.4 -DohTemplate https://dns.google/dns-query
```

### ControlD Free DNS

``` powershell
2606:1a40::2 -DohTemplate https://freedns.controld.com/p2
2606:1a40:1::2 -DohTemplate https://freedns.controld.com/p2
76.76.2.2 -DohTemplate https://freedns.controld.com/p2
76.76.10.2 -DohTemplate https://freedns.controld.com/p2
```

### Quad9 DNS

``` powershell
9.9.9.11 -DohTemplate https://dns11.quad9.net/dns-query
149.112.112.11 -DohTemplate https://dns11.quad9.net/dns-query
2620:fe::11 -DohTemplate https://dns11.quad9.net/dns-query
2620:fe::fe:11 -DohTemplate https://dns11.quad9.net/dns-query
```

------------------------------------------------------------------------

## 2. Set DNS Servers for All Network Adapters

Sets Cloudflare Family DNS (malware filtering).

``` powershell
$adapters = Get-NetAdapter
foreach ($adapter in $adapters) {
    $alias = $adapter.InterfaceAlias

    Set-DnsClientServerAddress -InterfaceAlias $alias -ServerAddresses ("2606:4700:4700::1112", "2606:4700:4700::1002")
    Set-DnsClientServerAddress -InterfaceAlias $alias -ServerAddresses ('1.1.1.2','1.0.0.2')
}
```

------------------------------------------------------------------------

## 3. Remove All Scheduled Task Triggers

Re-registers every scheduled task without triggers (manual execution
only).

``` powershell
$scheduledTasks = Get-ScheduledTask
foreach ($task in $scheduledTasks) {
    try {
        $taskName = $task.TaskName
        $taskPath = $task.TaskPath
        $fullTask = Get-ScheduledTask -TaskName $taskName -TaskPath $taskPath
        $actions = $fullTask.Actions
        $principal = $fullTask.Principal
        $settings = $fullTask.Settings
        $newTask = New-ScheduledTask -Action $actions -Principal $principal -Settings $settings
        Register-ScheduledTask -TaskName $taskName -TaskPath $taskPath -InputObject $newTask -Force
        Write-Output "Updated task: $taskPath$taskName"
    }
    catch {
        Write-Warning "Failed: $taskPath$taskName. Error: $_"
    }
}
```

------------------------------------------------------------------------

## 4. Disk Optimization & Conversion

### Optimize Drives

``` powershell
defrag /o /c /m
```

### Convert MBR to GPT (All Disks)

``` powershell
$drives = Get-Disk | Select-Object -ExpandProperty Number
foreach ($drive in $drives) {
    try {
        mbr2gpt /allowfullos /convert /disk=$drive
    }
    catch {
        Write-Warning "Error converting drive $drive`: $_"
    }
}
```

------------------------------------------------------------------------

## 5. Volume Repair, Cleanup, and App Re-registration

``` powershell
$drives = Get-Volume | Select-Object -ExpandProperty DriveLetter
foreach ($drive in $drives) {
    try {
        Repair-Volume -DriveLetter $drive -OfflineScanAndFix -ErrorAction Stop
        cleanmgr /verylowdisk /d $drive
        cleanmgr /sagerun:0 /d $drive
        Repair-Volume -DriveLetter $drive -SpotFix -ErrorAction Stop

        Get-ChildItem -Path $drive`:\ -Filter "AppxManifest.xml" -Recurse -File | ForEach-Object {
            try {
                Add-AppxPackage -DisableDevelopmentMode -Register $_.FullName -ErrorAction Stop
            }
            catch {
                Write-Warning "Error: $_"
            }
        }

        vssadmin Resize ShadowStorage /For=$drive`: /On=$drive`: /MaxSize=3%
    }
    catch {
        Write-Warning "Error repairing drive $drive`: $_"
    }
}
```

------------------------------------------------------------------------

## 6. Re-register All Windows UWP Apps

``` powershell
$appxManifestPaths = @(
    "$Env:ProgramFiles\WindowsApps",
    "$Env:WINDIR\SystemApps"
)
foreach ($path in $appxManifestPaths) {
    Get-ChildItem -Path $path -Filter "AppxManifest.xml" -Recurse -File | ForEach-Object {
        try {
            Add-AppxPackage -DisableDevelopmentMode -Register $_.FullName -ErrorAction Stop
        }
        catch {
            Write-Warning "Error registering app package: $_"
        }
    }
}
```

------------------------------------------------------------------------

## 7. System Image & File Repair

``` powershell
dism /online /cleanup-image /restorehealth /startcomponentcleanup
sfc /scannow
```

------------------------------------------------------------------------

## 8. Optional: Install Chocolatey

``` powershell
if (-not(Get-Command choco -ErrorAction SilentlyContinue)) {
    powershell.exe -c Set-ExecutionPolicy Bypass -Scope Process -Force; `
    [System.Net.ServicePointManager]::SecurityProtocol = `
    [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; `
    Invoke-Expression ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
    refreshenv
}
```
