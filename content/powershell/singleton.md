+++
title = 'PowerShell Singleton Lock'
date = 2024-01-20T15:51:00+08:00
keywords = ['powershell', 'lock', 'singletonlock', '单实例', '防重']
tags = ['windows', 'powershell']
draft = false
+++

PowerShell 可以使用 Windows Global Event 来实现防止在同一时刻同一个脚本重复运行。

## 示例脚本

```powershell
function Lock-Singleton
{
    $Locked = $false
    $Mutex = New-Object System.Threading.Mutex($true, "Global\ChangeToYourUniqueEventNameHere", [ref]$Locked)
    if (-not $Locked)
    {
        $Mutex.Close()
        Write-Host "another process is running now, exit"
        Exit 1
    }
}

Lock-Singleton

Write-Host "doing something here, sleep 10s"
Start-Sleep -Seconds 10

Exit 0
```

## 参考文档

1. [Mutex Class](https://learn.microsoft.com/en-us/dotnet/api/system.threading.mutex?view=net-8.0)
