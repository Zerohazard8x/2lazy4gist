# Fix: Windows Search doesn’t detect apps (Windows 10/11)

This guide documents the steps that resolved app detection issues in Windows Search (e.g., typing **Notepad** shows **Run command** instead of the **Notepad** app). The procedure follows **Solution 6** from Microsoft’s official troubleshooting page and includes direct citations with text fragments.

---

## Symptoms
- Start menu search fails to list installed apps, or labels them as “Run command”
- Apps appear under **All apps**, but not via search
- Rebuilding index / restarting Search didn’t help

---

## Prerequisites & Warnings
- **Back up first** (restore point and registry export). Microsoft warns registry edits can cause problems if done incorrectly. See the “Important” box on the official page.  
  Source: [Microsoft Learn — Fix problems in Windows Search](https://learn.microsoft.com/en-us/troubleshoot/windows-client/shell-experience/fix-problems-in-windows-search).

---

## Step‑by‑step fix (Solution 6)

1) **(Optional) Sanity check with a new user profile**  
   Create a new Windows user and confirm Search works there.  
   Source: [Microsoft Learn — “Make sure that Windows Search works for a newly created Windows account.”](https://learn.microsoft.com/en-us/troubleshoot/windows-client/shell-experience/fix-problems-in-windows-search)

2) **Delete the Search package folder (per OS)**  
   - **Windows 10:** Delete `%USERPROFILE%\AppData\Local\Packages\Microsoft.Windows.Search_cw5n1h2txyewy`  
     Source: [Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/windows-client/shell-experience/fix-problems-in-windows-search)
   - **Windows 11:** Delete `%USERPROFILE%\AppData\Local\Packages\MicrosoftWindows.Client.CBS_cw5n1h2txyewy`  
     Source: [Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/windows-client/shell-experience/fix-problems-in-windows-search)  

   > Tip: If the folder is locked, do it from **Windows Recovery Environment (WinRE)** or from another admin account.  
   > Source: [Microsoft Learn — “Use the Windows Recovery Environment, or sign out and then sign in to another user account.”](https://learn.microsoft.com/en-us/troubleshoot/windows-client/shell-experience/fix-problems-in-windows-search)

3) **Delete the per‑user Search registry key** (affects only the current user)  
   Path: `HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Search` → **Delete** the **Search** key  
   Sources:  
   - [Microsoft Learn — registry path](https://learn.microsoft.com/en-us/troubleshoot/windows-client/shell-experience/fix-problems-in-windows-search)  
   - [Microsoft Learn — delete the key](https://learn.microsoft.com/en-us/troubleshoot/windows-client/shell-experience/fix-problems-in-windows-search)

4) **Re‑register the package** (run as Administrator)  
   - **Windows 10:**  
     ```powershell
     Add-AppxPackage -Path "C:\Windows\SystemApps\Microsoft.Windows.Search_cw5n1h2txyewy\Appxmanifest.xml" -DisableDevelopmentMode -Register
     ```  
     Source: [Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/windows-client/shell-experience/fix-problems-in-windows-search)
   - **Windows 11:**  
     ```powershell
     Add-AppxPackage -Path "C:\Windows\SystemApps\MicrosoftWindows.Client.CBS_cw5n1h2txyewy\Appxmanifest.xml" -DisableDevelopmentMode -Register
     ```  
     Source: [Microsoft Learn](https://learn.microsoft.com/en-us/troubleshoot/windows-client/shell-experience/fix-problems-in-windows-search)

5) **Restart the PC and test search**  
   This restarts indexing and regenerates the deleted items.  
   Source: [Microsoft Learn — “Restart the computer… This action restarts search indexing and regenerates the registry key and the AppData folder.”](https://learn.microsoft.com/en-us/troubleshoot/windows-client/shell-experience/fix-problems-in-windows-search)

---

## Why this works
Deleting the **per‑user** package data + the **HKCU Search key** clears corrupted state. Re‑registering restores a clean configuration so Start menu search can enumerate apps correctly.  
Primary source: Microsoft’s **Solution 6** (last updated on the page header).  
Source: [Microsoft Learn — Fix problems in Windows Search](https://learn.microsoft.com/en-us/troubleshoot/windows-client/shell-experience/fix-problems-in-windows-search)

---

## Verification checklist
- Searching **Notepad** shows **Notepad — App** (not “Run command”)  
- Previously missing apps now appear in search results

---

## References (primary, authoritative)
- Microsoft Learn — *Fix problems in Windows Search* (Solution 6 with explicit steps and commands).  
  [Direct link](https://learn.microsoft.com/en-us/troubleshoot/windows-client/shell-experience/fix-problems-in-windows-search)
