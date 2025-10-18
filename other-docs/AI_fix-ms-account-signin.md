# Fix — Blank white "Manage" sign-in dialog (WebView2 / WAM)

## Summary
This article documents a reproducible, safe, and reversible procedure to fix the blank white sign-in popup that appears when you click **Settings → Accounts → Email & accounts → Accounts used by other apps → Manage** (or when OneDrive / Xbox / Office show a "Sign in cancelled" or blank dialog).

- Symptom: a white window appears and then immediately closes when attempting to manage or add a Microsoft account in Windows 10/11.
- Likely root cause: a broken or missing **Microsoft Edge WebView2 runtime** or corrupted **Windows Account Manager (WAM)** token/cache. These components host the embedded web UI that performs OAuth flows for Microsoft accounts.


---

## Who this is for
- Users of Windows 10 or Windows 11 experiencing blank/vanishing sign-in dialogs in Settings, OneDrive, Xbox, or Office.
- IT support staff who need a stepwise, verifiable checklist to resolve the issue reproducibly.


---

## High-level explanation (short)
- Windows hosts embedded web authentication dialogs using a combination of **WAM** (Web Account Manager) and **WebView2** (embedded Edge runtime). When either the runtime or the broker token cache is missing or corrupted, the web UI will not render and the process will close.


---

## Safety and reversibility
- All steps in this article are reversible or non-destructive.
- Where a folder or file is modified, the instructions rename the item rather than deleting it; the original files can be restored.
- If you are using a managed (work or school) device, check with IT before applying steps that modify system app packages or clear account caches.


---

## Quick checklist (one-pass)
1. Repair/install **WebView2 Runtime** → reboot.
2. Re-register WAM host packages (AAD.BrokerPlugin / CloudExperienceHost) via PowerShell → reboot.
3. Clear TokenBroker / BrokerPlugin account caches → reboot.
4. Rename WebView2 user-data folders (safe backup) → reboot.
5. Run `wsreset.exe` (Store cache) and test.
6. If needed: `DISM /Online /Cleanup-Image /RestoreHealth` then `sfc /scannow` → reboot.


---

## Detailed step-by-step procedure

### Pre-checks (2 minutes)
- Confirm that the Microsoft account credentials work in a normal browser at `https://account.microsoft.com`.
- Confirm that Date, Time, Timezone, and Region are correct (Settings → Time & language).


### Step A — Repair or install Microsoft Edge WebView2 runtime
- Why: WebView2 hosts the embedded browser control used by the sign-in dialog. A missing or corrupted runtime will prevent rendering.
- How:
  - Open **Settings → Apps → Installed apps** and search for **Microsoft Edge WebView2 Runtime**.
  - If present: choose **Modify** or **Repair**.
  - If missing or Repair fails: download the **Evergreen WebView2 Runtime** from Microsoft and run the installer, then reboot.


### Step B — Re-register WAM host packages (safe, official)
- Why: Windows provides broker package(s) that host the WAM flows; re-registering fixes package registration corruption.
- How (PowerShell as Administrator — paste each block appropriate to your account type):

  - If you use a **work or school account (Azure AD / Entra ID)**:

  ```powershell
  if (-not (Get-AppxPackage Microsoft.AAD.BrokerPlugin)) {
    Add-AppxPackage -Register "$env:windir\SystemApps\Microsoft.AAD.BrokerPlugin_cw5n1h2txyewy\Appxmanifest.xml" -DisableDevelopmentMode -ForceApplicationShutdown
  }
  Get-AppxPackage Microsoft.AAD.BrokerPlugin
  ```

  - If you use a **personal Microsoft account** (`@outlook.com`, `@hotmail.com`, etc.):

  ```powershell
  if (-not (Get-AppxPackage Microsoft.Windows.CloudExperienceHost)) {
    Add-AppxPackage -Register "$env:windir\SystemApps\Microsoft.Windows.CloudExperienceHost_cw5n1h2txyewy\Appxmanifest.xml" -DisableDevelopmentMode -ForceApplicationShutdown
  }
  Get-AppxPackage Microsoft.Windows.CloudExperienceHost
  ```

  - Reboot after you run the commands.


### Step C — Purge WAM token caches (safe — keep backups)
- Why: Token caches used by WAM or TokenBroker can be corrupted and must be cleared for a fresh sign-in.
- How:
  - Close Settings and all Microsoft apps.
  - Open **File Explorer** or **Win+R** and paste the following paths one at a time, then delete the contents of the `Accounts` or `Cache` folders (do not delete the parent package folder):

    - `%LOCALAPPDATA%\Packages\Microsoft.AAD.BrokerPlugin_cw5n1h2txyewy\AC\TokenBroker\Accounts`
    - `%LOCALAPPDATA%\Packages\Microsoft.Windows.CloudExperienceHost_cw5n1h2txyewy\AC\TokenBroker\Accounts`
    - `%LOCALAPPDATA%\Microsoft\TokenBroker\Cache`

  - Reboot after clearing these caches.


### Step D — Rename WebView2 user-data folders (non-destructive)
- Why: Many WebView2 failures are due to a corrupted user-data folder. Renaming forces regeneration while preserving a backup.
- How:
  - End `msedgewebview2.exe` in Task Manager.
  - Open **Win+R** → `%LOCALAPPDATA%\Microsoft\EdgeWebView\`
  - If a **User Data** folder exists, rename it to `User Data.old`.
  - If an `EdgeWebView` folder exists, rename it to `EdgeWebView.old`.
  - Note: some apps use vendor-specific `EBWebView` locations; if you see them under `%LOCALAPPDATA%` for a vendor, rename them as well.
  - Reboot and test.


### Step E — Reset Microsoft Store cache and run Store Apps troubleshooter
- Run **Win+R** → `wsreset.exe` and allow it to finish.
- Open Settings → System → Troubleshoot → Other troubleshooters → **Windows Store Apps** → Run.
- Reboot and test.


### Step F — System file checks (last-resort repair)
- Run the two Windows component repair commands from an elevated Command Prompt:

```cmd
DISM /Online /Cleanup-Image /RestoreHealth
sfc /scannow
```

- Reboot and test again.


---

## Verification steps (how to know it is fixed)
- Click **Settings → Accounts → Email & accounts → (the Microsoft account) → Manage** and confirm the account picker or sign-in UI appears (not a blank window).
- Sign in to OneDrive and Xbox from the apps and confirm they complete interactive sign-in.
- If the problems persist on your main profile but the Manage dialog works in a new local user profile, the issue is profile-specific. Consider migrating user data or recreating the profile.


---

## Rollback guidance
- If a step created an issue, restore any renamed folders: rename `User Data.old` back to `User Data` and `EdgeWebView.old` back to `EdgeWebView`.
- If you re-registered packages and wish to revert, a reboot normally retains the restored package state; uninstalling system packages is not recommended without IT support.


---

## Troubleshooting notes to collect (if you escalate)
- Exact Windows version and build: open **Win+R** → `winver` → copy version/build.
- Task Manager processes observed while you click Manage: any `Microsoft WWA Host`, `Runtime Broker`, or `msedgewebview2.exe` entries and their status (running/suspended).
- Output or errors from PowerShell appx package registration if any.
- Log lines from Event Viewer under **Applications and Services Logs → Microsoft → Windows → WebAccountManager** (if present).


---

## Quick one-liners you can copy/paste
- Reset OneDrive (safe):
```
%localappdata%\Microsoft\OneDrive\onedrive.exe /reset
```
- Reset Microsoft Store cache:
```
wsreset.exe
```
- Re-register Cloud Experience Host (personal accounts — PowerShell as Admin):
```powershell
Add-AppxPackage -Register "$env:windir\SystemApps\Microsoft.Windows.CloudExperienceHost_cw5n1h2txyewy\Appxmanifest.xml" -DisableDevelopmentMode -ForceApplicationShutdown
```


---

## Definitions (simple versions)
- **WebView2 (simple version):** an embedded version of Microsoft Edge used by desktop apps to show web content inside an application.
- **WAM — Web Account Manager (simple version):** the Windows component that brokers sign-ins for desktop apps so a single interactive sign-in can be shared across apps.