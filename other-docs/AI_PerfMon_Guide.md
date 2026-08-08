# Windows Performance Monitor BLG Logging

**NOTE: This guide was created using AI (LLM / ChatGPT) assistance**

This guide creates a Windows Performance Monitor Data Collector Set manually in the graphical interface, records the process counters needed for later per-process CPU, private-memory, and disk-I/O analysis, and starts the Data Collector Set at Windows startup with a short Task Scheduler command.

Microsoft documents the Data Collector Set wizard, `logman`, Performance Logs and Alerts, and `schtasks` separately. This guide combines those supported mechanisms into one reproducible setup. The relevant Microsoft documentation is linked beside the steps that depend on it rather than collected only at the end.

## Create the Data Collector Set

1. Open Performance Monitor with administrative rights. Press `Win + R`, type the following command, and press `Ctrl + Shift + Enter` rather than only pressing Enter.

```text
perfmon
```

Approve the User Account Control prompt if Windows displays one. Microsoft documents the same `Data Collector Sets > User Defined > New > Data Collector Set` wizard path in its [Data Collector Set creation instructions](https://learn.microsoft.com/en-us/dynamics365/business-central/dev-itpro/administration/create-data-collector-performance-counters).

2. In the left navigation pane, expand `Data Collector Sets`, then right-click `User Defined`, choose `New`, and choose `Data Collector Set`. On the first wizard page, enter the exact name you chose for `<COLLECTOR_NAME>`, select `Create manually (Advanced)`, and click `Next`.

```text
Performance
└─ Data Collector Sets
   └─ User Defined
      └─ New
         └─ Data Collector Set
```

3. On the page asking what type of data to include, select `Create data logs`, select `Performance counter`, leave the other collector types unselected unless you independently need them, and click `Next`. The purpose of this guide is a counter log in BLG format, so configuration data and event trace data are unnecessary for this particular setup.

## Add the Required Per Process Counters

4. Click `Add` to open the Add Counters window. If `Process V2` appears in the Available counters list, expand `Process V2` and use it because Windows 11 and later provide this counter set with the process identifier in the instance name, which avoids ambiguity that can occur when multiple processes share the same executable name. Microsoft specifically recommends `Process V2` for process collection on Windows 11 and later in its [performance-counter documentation](https://learn.microsoft.com/en-us/windows/win32/perfctrs/enumerating-process-objects).

If `Process V2` does not appear on the computer, use `Process` instead.

5. Add all seven of the following per-process counters. In the `Instances of selected object` area, choose `<All instances>` before clicking `Add`, because selecting only one currently running instance would omit other processes and processes that start later.

```text
\Process V2(*)\% Processor Time
\Process V2(*)\Working Set - Private
\Process V2(*)\Private Bytes
\Process V2(*)\IO Read Bytes/sec
\Process V2(*)\IO Write Bytes/sec
\Process V2(*)\IO Read Operations/sec
\Process V2(*)\IO Write Operations/sec
```

If the computer has only the older counter set, use the same seven counter names under `Process` instead.

```text
\Process(*)\% Processor Time
\Process(*)\Working Set - Private
\Process(*)\Private Bytes
\Process(*)\IO Read Bytes/sec
\Process(*)\IO Write Bytes/sec
\Process(*)\IO Read Operations/sec
\Process(*)\IO Write Operations/sec
```

The per-process CPU counter is required if you want per-process CPU results later. A whole-machine counter such as `\Processor(_Total)\% Processor Time` cannot be substituted for it because `_Total` describes the machine rather than each process instance.

6. Add whole-machine counters only if you want additional context. The following five counters are useful for comparing a process spike with overall CPU and disk activity, but they are optional for the per-process analysis.

```text
\PhysicalDisk(_Total)\Avg. Disk Bytes/Read
\PhysicalDisk(_Total)\Avg. Disk Bytes/Write
\PhysicalDisk(_Total)\Disk Read Bytes/sec
\PhysicalDisk(_Total)\Disk Write Bytes/sec
\Processor(_Total)\% Processor Time
```

This optional set matches useful whole-machine context from the existing collector without making those values mandatory. Microsoft also uses broad processor, process, disk, memory, and other counter groups in its current [Performance Monitor troubleshooting guidance](https://learn.microsoft.com/en-us/troubleshoot/windows-server/support-tools/troubleshoot-issues-performance-monitor).

7. Before leaving the Add Counters window, read the Added counters list from top to bottom and confirm that the seven required per-process counter names are present. Click `OK` only after all seven are visible, then return to the wizard.

## Choose the Sample Interval

8. Choose the sample interval according to the shortest event that you need to see. I have tested 60 seconds.
9. Click `Next`, then set the root directory. A conventional local path is shown below, but another local volume is acceptable if it has more free space.

```text
%SystemDrive%\PerfLogs\Admin\<COLLECTOR_NAME>
```

Click `Next`. On the final wizard page, select `Open properties for this data collector set`, then click `Finish`.

## Set the Run Account and Binary BLG Format

10. In the Data Collector Set Properties window, use the `General` tab to confirm the Run As account. For unattended machine-wide collection, use the local `SYSTEM` account so collection does not depend on a particular interactive user remaining signed in. If the wizard does not already show `SYSTEM`, use the `Change` button and select the local System account according to the controls offered by your Windows version.
11. Click `OK` if necessary.

## Making the active log overwrite samples after a size is reached

12. To make the active log overwrite samples after a size is reached, use the short command below. Replace `<TASK_NAME>` with the exact name you chose, including spaces. The command below enables circular logging with a 500 MB maximum log size.

```powershell
logman update counter "<COLLECTOR_NAME>" -f bincirc -max 500
```

## Test the Collector Before Creating the Startup Task

13. In Performance Monitor, right-click the Data Collector Set and choose `Start`. Wait for at least two sample intervals because rate-based counters need successive samples to become useful, then open an elevated PowerShell window and query the Data Collector Set by its exact name.

```powershell
logman query "<COLLECTOR_NAME>"
```

Microsoft documents `logman query` as the command for querying Data Collector Set properties. [See the `logman query` documentation](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/logman-query).

14. Stop the test run so the current BLG is closed and easy to inspect.

```powershell
logman stop "<COLLECTOR_NAME>"
```

Microsoft documents `logman start` and `logman stop` as the command-line controls for data collection. [See the main `logman` documentation](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/logman).

15. Locate the completed BLG under the root directory. You can inspect its available counters without converting the file by using `relog /q`.

```powershell
relog "C:\full\path\to\DataCollector01.blg" /q
```

Replace the example path with the actual BLG path. Confirm again that the seven required per-process counter types are present.

16. Make sure the Data Collector Set is currently stopped, then run the scheduled task manually.

## Create a Startup Task

17. Open PowerShell as administrator.
18. Create the startup task with the short command below. Replace `<TASK_NAME>` and `<COLLECTOR_NAME>` with the exact names you chose, including spaces.

```powershell
schtasks /create /sc onstart /tn "<TASK_NAME>" /tr 'logman start "<COLLECTOR_NAME>"' /ru system
```

`/sc onstart` runs the task when the system starts, `/ru system` runs it as the local System account, and `/tr` stores the command that starts the named Data Collector Set.

Microsoft states that `ONSTART` runs every time the system starts, that `/RU SYSTEM` selects the local System account, that `LIMITED` is the default run level, and that when `/TR` omits a path `schtasks` assumes the executable is in `%SystemRoot%\System32`. Those documented defaults are why the command does not need `/rl limited` or a full path to `logman.exe`. [See Microsoft&#39;s `schtasks /create` documentation](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/schtasks-create).

## Test the Startup Task Without Rebooting

```powershell
logman stop "<COLLECTOR_NAME>"
schtasks /run /tn "<TASK_NAME>"
```

If the first command reports that the Data Collector Set is already stopped, that is acceptable. Microsoft documents `schtasks /run` as an immediate run that uses the program and account stored in the task without changing its normal schedule. [See the `schtasks /run` documentation](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/schtasks-run).

19. Wait several seconds, then query the collector.

```powershell
logman query "<COLLECTOR_NAME>"
```

The expected state is shown below.

```text
Status    Running
```

The scheduled task itself can return to a ready state quickly because `logman start` sends the start request and exits while the Data Collector Set continues collecting independently.

20. Query the task in verbose form and inspect the Run As account and task command.

```powershell
schtasks /query /tn "<TASK_NAME>" /v /fo list
```

Confirm that the task identifies the System account and that its action starts the exact Data Collector Set name you intended. Microsoft notes that a verbose query of a System task reports `NT AUTHORITY\SYSTEM` as the Run As user. [See the System-task section of the `schtasks /create` documentation](https://learn.microsoft.com/en-us/windows-server/administration/windows-commands/schtasks-create).

## Verify the Entire Setup After Reboot

21. Reboot Windows. After the system is back up, open an elevated PowerShell window and run the following query.

```powershell
logman query "<COLLECTOR_NAME>"
```

The Data Collector Set should report `Running`. If it reports `Stopped`, run the scheduled task manually, query it verbosely, and verify that the collector name in `/tr` exactly matches the name under `Data Collector Sets > User Defined`.

22. Open `taskschd.msc`, select `Task Scheduler Library`, and inspect `<TASK_NAME>`. Confirm that its trigger is `At startup`, its account is `SYSTEM`, and its action starts the intended collector.

```text
Program or command    logman
Arguments             start "<COLLECTOR_NAME>"
```

The exact way Task Scheduler displays the action can vary, but the command and collector name must be equivalent.