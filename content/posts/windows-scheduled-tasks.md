---
title: "Scheduling jobs with Task Scheduler and PowerShell"
date: 2026-09-09
categories: ["windows"]
tags: ["powershell", "administration", "windows"]
---
You can register a scheduled task entirely from PowerShell:

```powershell
$action  = New-ScheduledTaskAction -Execute "pwsh" -Argument "-File C:\jobs\backup.ps1"
$trigger = New-ScheduledTaskTrigger -Daily -At 2am
Register-ScheduledTask -TaskName "Nightly backup" -Action $action -Trigger $trigger
```
