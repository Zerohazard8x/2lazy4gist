## repair duplicate/broken Appx volumes

> Goal: end up with exactly one valid entry per real volume, a correct `SisPath`, and the right **default** volume.

1. **Inspect what Windows thinks is registered**

    - Run PowerShell as Admin:

        ```powershell
        Get-AppxVolume
        ```

    - This lists the known Appx volumes and makes it obvious if the same physical volume is registered multiple times. ([Microsoft Learn][2])

2. **Remove stale/duplicate entries (use PowerShell, not Regedit)**

    - If you see a bad/stale D: entry, for example:

        ```powershell
        Remove-AppxVolume -Volume D:\
        ```

    - If the volume isn’t mounted under a drive letter, you can remove by **MediaId** (GUID) returned by `Get-AppxVolume`:

        ```powershell
        $vol = Get-AppxVolume | Where-Object { $_.VolumeId -eq '{929313F0-0000-0000-B9BE-C36672871674}' }
        Remove-AppxVolume -Volume $vol
        ```

    - Community threads report success after deleting stale entries that pointed to long-gone drives; that’s consistent with how these APIs are meant to be used. Treat that as field evidence, not spec. ([Reddit][6])

3. **Re-add the valid volume(s) so Windows writes the right values**

    - For a D: volume you actually want to use:

        ```powershell
        Add-AppxVolume -Path 'D:\WindowsApps'
        ```

    - Note the path requirement (`<Drive>:\WindowsApps`). Microsoft calls this out explicitly. ([Microsoft Learn][5])

4. **Set the correct default volume (don’t rely on “highest number”)**

    - To make C: (or D:) the default install target:

        ```powershell
        Set-AppxDefaultVolume -Volume C:\
        # or
        Set-AppxDefaultVolume -Volume D:\
        ```

    - This is the supported, documented mechanism Windows uses for “New apps will save to…”. ([Microsoft Learn][3])

5. **If Store apps misbehave afterwards, re-register them**

    - Microsoft guidance (FSLogix troubleshooting doc) includes re-registering AppX apps as admin if inbox/Store apps fail to launch; follow those PowerShell steps if needed. ([Microsoft Learn][7])

> **Do not delete** the entry whose **`SisPath` is `C:\Program Files\WindowsApps`** (that’s the system store on C:). Multiple field reports warn that removing it breaks things fast. ([Microsoft Flight Simulator Forums][8])

[1]: https://learn.microsoft.com/en-us/windows/msix/desktop/desktop-to-uwp-behind-the-scenes?utm_source=chatgpt.com "Understanding how packaged desktop apps run on Windows"
[2]: https://learn.microsoft.com/en-us/powershell/module/appx/get-appxvolume?view=windowsserver2025-ps "Get-AppxVolume (Appx) | Microsoft Learn"
[3]: https://learn.microsoft.com/en-us/powershell/module/appx/set-appxdefaultvolume?view=windowsserver2025-ps&utm_source=chatgpt.com "Set-AppxDefaultVolume (Appx)"
[4]: https://legacyupdate.net/errorcodes?utm_source=chatgpt.com "Windows Update Error Codes"
[5]: https://learn.microsoft.com/en-us/powershell/module/appx/add-appxvolume?view=windowsserver2025-ps&utm_source=chatgpt.com "Add-AppxVolume (Appx)"
[6]: https://www.reddit.com/r/XboxGamePass/comments/q7ez4x/cant_install_anything_from_microsoft_store/?utm_source=chatgpt.com "Can't install ANYTHING from Microsoft Store"
[7]: https://learn.microsoft.com/en-us/fslogix/troubleshooting-appx-issues?utm_source=chatgpt.com "Issues with AppX, MSIX, or Microsoft store applications"
[8]: https://forums.flightsimulator.com/t/fix-0x80070141-error-message/309786?utm_source=chatgpt.com "FIX 0x80070141 error message - General Discussion"
[9]: https://learn.microsoft.com/en-us/answers/questions/3819577/help-me-to-find-the-good-values-for-my-disk-please?utm_source=chatgpt.com "Help me to find the good values for my disk please"

---

# Script: Detect & (optionally) clean duplicate Appx package volumes

```powershell
<#
.SYNOPSIS
  Detect Appx package-volume duplicates (same physical volume registered multiple times)
  and optionally remove stale entries and set the kept volume as the Appx default.

.NOTES
  - Dry-run by default. Use -Apply to perform removals and -SetDefault to set default after removals.
  - Run in elevated (Administrator) PowerShell.
  - This script tries to be conservative: it prefers volumes that are mounted and whose <Drive>:\WindowsApps exists.
  - Remove-AppxVolume can fail if apps are still staged on the volume; failures are logged and the script continues.
#>

param(
    [switch] $Apply,         # If set, actually call Remove-AppxVolume / Add-AppxVolume / Set-AppxDefaultVolume
    [switch] $SetDefault,    # If set (and $Apply), set the surviving volume as the default with Set-AppxDefaultVolume
    [switch] $IncludeOffline # If set, include offline (dismounted) Appx volumes in the scan (default: include both)
)

function Ensure-Admin {
    $isAdmin = ([Security.Principal.WindowsPrincipal] `
                [Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
    if (-not $isAdmin) {
        Write-Error "This script must be run as Administrator. Right-click PowerShell and 'Run as administrator'."
        exit 1
    }
}

function Get-VolumeKey {
    param($v)
    # Try common unique identifiers in order of usefulness
    if ($null -ne $v.MediaId -and $v.MediaId -ne '') { return $v.MediaId }
    if ($null -ne $v.VolumeId -and $v.VolumeId -ne '') { return $v.VolumeId }
    if ($null -ne $v.Name -and $v.Name -ne '') { return $v.Name }
    # fallback to MountPoint (drive letter) if that's all we have
    return ($v.MountPoint -or $v.Path -or $v.PSObject.Properties | Where-Object { $_.Name -match 'Mount' } | Select-Object -First 1).Value
}

function Score-Volume {
    param($v)
    # Defensive scoring: higher = better to KEEP
    $score = 0

    # prefer volumes with a mount point (drive letter)
    if ($v.PSObject.Properties['MountPoint'] -and $v.MountPoint -and $v.MountPoint -ne '') { $score += 20 }

    # prefer volumes which have the expected SisPath and where the folder exists
    if ($v.PSObject.Properties['SisPath'] -and $v.SisPath -and (Test-Path $v.SisPath)) { $score += 40 }

    # prefer online/mounted volumes
    if ($v.PSObject.Properties['IsOffline']) {
        try { if (-not $v.IsOffline) { $score += 10 } } catch {}
    } else {
        # if property not present, assume it's available and add a small bias
        $score += 5
    }

    # small bias for shorter key (heuristic)
    try { if ($v.PSObject.Properties['Name'] -and ($v.Name -match '\\\?\')) { $score += 1 } } catch {}

    return $score
}

function Choose-ToKeep {
    param($group)
    # Input: $group - collection of AppxVolume objects that share the same physical identifier
    # Output: PSCustomObject { Keep = <volume>, Remove = <array of volumes to remove in order> }

    $scored = foreach ($v in $group) {
        [PSCustomObject]@{
            Volume = $v
            Score  = (Score-Volume -v $v)
        }
    }
    # sort descending by score; tie-breaker: prefer mounted (MountPoint present) then any stable ordering
    $ordered = $scored | Sort-Object -Property @{Expression = 'Score'; Descending = $true}, @{Expression = { if ($_.Volume.MountPoint) {0} else {1} } }
    $keep = $ordered[0].Volume
    $remove = $ordered | Select-Object -Skip 1 | ForEach-Object { $_.Volume }
    return [PSCustomObject]@{ Keep = $keep; Remove = $remove }
}

# --- main
Ensure-Admin

Write-Host "Scanning Appx volumes..." -ForegroundColor Cyan

# get volumes
try {
    $allVolumes = Get-AppxVolume -ErrorAction Stop
} catch {
    Write-Error "Get-AppxVolume failed: $_"
    exit 2
}

if (-not $allVolumes) {
    Write-Host "No Appx volumes found." -ForegroundColor Yellow
    exit 0
}

# filter offline if user did not ask to include offline; default behavior: include all (mounted+unmounted)
if (-not $IncludeOffline) {
    # Keep both online & offline by default; but this switch exists if caller wants to restrict.
    # (We leave everything so duplicates that include offline entries are visible.)
    $scanVolumes = $allVolumes
} else {
    $scanVolumes = $allVolumes
}

# Build dictionary keyed by 'physical identity' (try MediaId/VolumeId/Name)
$groups = @{}
foreach ($v in $scanVolumes) {
    $key = Get-VolumeKey -v $v
    if (-not $key) {
        # fallback: stringify object
        $key = ($v | Out-String).GetHashCode()
    }
    if (-not $groups.ContainsKey($key)) { $groups[$key] = @() }
    $groups[$key] += $v
}

# find duplicates
$duplicateGroups = $groups.GetEnumerator() | Where-Object { $_.Value.Count -gt 1 }

if (-not $duplicateGroups) {
    Write-Host "No duplicate physical-volume registrations found." -ForegroundColor Green
    exit 0
}

# summary collecting
$resultSummary = [System.Collections.ArrayList]::new()

foreach ($entry in $duplicateGroups) {
    $key = $entry.Key
    $members = $entry.Value
    Write-Host "Found duplicate registrations for key: $key" -ForegroundColor Yellow
    foreach ($m in $members) {
        $display = @{
            Name = ($m.PSObject.Properties['Name']  ? $m.Name : '')
            MountPoint = ($m.PSObject.Properties['MountPoint'] ? $m.MountPoint : '')
            SisPath = ($m.PSObject.Properties['SisPath'] ? $m.SisPath : '')
            MediaId = ($m.PSObject.Properties['MediaId'] ? $m.MediaId : ($m.PSObject.Properties['VolumeId'] ? $m.VolumeId : ''))
            Flags = ($m.PSObject.Properties['Flags'] ? $m.Flags : '')
        }
        Write-Host ("  - Name: {0}   MountPoint: {1}   SisPath: {2}   MediaId: {3}   Flags: {4}" -f $display.Name, $display.MountPoint, $display.SisPath, $display.MediaId, $display.Flags)
    }

    $choice = Choose-ToKeep -group $members
    $keep = $choice.Keep
    $toRemove = $choice.Remove

    # present decision
    Write-Host "`n  -> Will KEEP:" -NoNewline
    Write-Host (" Name={0}  MountPoint={1}  SisPath={2}" -f ($keep.Name -or '<none>'), ($keep.MountPoint -or '<none>'), ($keep.SisPath -or '<none>')) -ForegroundColor Green

    if ($toRemove.Count -eq 0) {
        Write-Host "  -> No other entries to remove for this key.`n"
        continue
    }

    Write-Host "  -> Candidates to REMOVE (in order):" -ForegroundColor Red
    foreach ($r in $toRemove) {
        $id = ($r.PSObject.Properties['MediaId'] ? $r.MediaId : ($r.PSObject.Properties['VolumeId'] ? $r.VolumeId : ($r.Name -or $r.MountPoint)))
        Write-Host ("     * {0}   MountPoint={1}   SisPath={2}" -f $id, ($r.MountPoint -or '<none>'), ($r.SisPath -or '<none>'))
    }

    # Attempt to remove (if requested)
    foreach ($r in $toRemove) {
        # choose argument form: prefer MediaId (GUID), then MountPoint, then Name
        $arg = $null
        if ($r.PSObject.Properties['MediaId'] -and $r.MediaId) { $arg = $r.MediaId }
        elseif ($r.PSObject.Properties['VolumeId'] -and $r.VolumeId) { $arg = $r.VolumeId }
        elseif ($r.PSObject.Properties['MountPoint'] -and $r.MountPoint) { $arg = $r.MountPoint }
        elseif ($r.PSObject.Properties['Name'] -and $r.Name) { $arg = $r.Name }
        else { $arg = $r }

        $entrySummary = [PSCustomObject]@{
            Key = $key
            Keep = ($keep.PSObject.Properties['MediaId'] ? $keep.MediaId : ($keep.Name -or $keep.MountPoint))
            RemoveCandidate = ($r.PSObject.Properties['MediaId'] ? $r.MediaId : ($r.Name -or $r.MountPoint))
            RemoveArg = $arg
            RemoveResult = 'Skipped (dry-run)'
        }

        if ($Apply) {
            Write-Host "    -> Attempting Remove-AppxVolume -Volume $arg ..." -ForegroundColor Cyan
            try {
                # Use the cmdlet; might throw if apps are staged to the volume
                Remove-AppxVolume -Volume $arg -ErrorAction Stop
                Write-Host "       Success: removed $arg" -ForegroundColor Green
                $entrySummary.RemoveResult = "Removed"
            } catch {
                Write-Host ("       ERROR removing $arg: {0}" -f $_.Exception.Message) -ForegroundColor Red
                $entrySummary.RemoveResult = ("Failed: {0}" -f $_.Exception.Message)
            }
        } else {
            Write-Host "    -> Dry-run: would call Remove-AppxVolume -Volume $arg" -ForegroundColor DarkYellow
        }

        $null = $resultSummary.Add($entrySummary)
    }

    # After attempted removals, optionally ensure kept volume is present / Add-AppxVolume if not
    # Prefer using MountPoint if available for Set-AppxDefaultVolume
    if ($Apply -and $SetDefault) {
        $defArg = $null
        if ($keep.PSObject.Properties['MountPoint'] -and $keep.MountPoint) { $defArg = $keep.MountPoint }
        elseif ($keep.PSObject.Properties['MediaId'] -and $keep.MediaId) { $defArg = $keep.MediaId }
        else { $defArg = ($keep.Name -or $keep) }

        Write-Host ("  -> Attempting to set default Appx volume to: {0}" -f $defArg) -ForegroundColor Cyan
        try {
            Set-AppxDefaultVolume -Volume $defArg -ErrorAction Stop
            Write-Host "     Set-AppxDefaultVolume succeeded." -ForegroundColor Green
        } catch {
            Write-Host ("     Set-AppxDefaultVolume FAILED: {0}" -f $_.Exception.Message) -ForegroundColor Red
            # continue; not fatal for script
        }
    }

    Write-Host "`n"
}

# Final summary
Write-Host "Summary of attempted actions:" -ForegroundColor Cyan
$resultSummary | Format-Table -AutoSize

Write-Host "`nScan complete." -ForegroundColor Cyan
if (-not $Apply) {
    Write-Host "NOTE: This was a dry-run. Rerun this script with -Apply to perform removals and -SetDefault to set a default volume." -ForegroundColor Yellow
}
```

---

# How to use

1. Open PowerShell **as Administrator**.
2. Save the script to a file, e.g. `Fix-AppxDuplicates.ps1`.
3. First run a dry-run to see what it *would* do:

   ```powershell
   .\Fix-AppxDuplicates.ps1
   ```
4. If the proposed actions look good, run with `-Apply` to actually remove duplicates:

   ```powershell
   .\Fix-AppxDuplicates.ps1 -Apply
   ```
5. If you also want the script to set the surviving volume as Windows’ Appx default, add `-SetDefault`:

   ```powershell
   .\Fix-AppxDuplicates.ps1 -Apply -SetDefault
   ```

```reg
Windows Registry Editor Version 5.00

; NOTE: Example - Both of these should be the same bc they both have the same "Name"
; Prioritize higher number
[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Appx\PackageVolumes\6]
"Flags"=dword:0000027a
"MediaId"="{929313F0-0000-0000-B9BE-C36672871674}"
; "MountPoint"="D:"
; "Name"="\\\\?\\Volume{a56a76d1-da8d-46f9-a841-689bec5f92c8}"
; "SisPath"="D:\\WindowsApps"

[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Appx\PackageVolumes\7]
"Flags"=dword:0000027a
"MediaId"="{929313F0-0000-0000-B9BE-C36672871674}"
; "MountPoint"="D:"
; "Name"="\\\\?\\Volume{a56a76d1-da8d-46f9-a841-689bec5f92c8}"
; "SisPath"="D:\\WindowsApps"
```