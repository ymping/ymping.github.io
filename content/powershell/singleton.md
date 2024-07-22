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
    $Global:Mutex = New-Object System.Threading.Mutex($true, "Global\ChangeToYourUniqueEventNameHere", [ref]$Locked)
    if (-not $Locked)
    {
        $Global:Mutex.Close()
        Write-Host "another process is running now, exit"
        Exit 1
    }
}

Lock-Singleton

Write-Host "doing something here, sleep 10s"
Start-Sleep -Seconds 10

Exit 0
```

防重也可以使用脚本名称和参数来查找该脚本是否正在运行，但此方案在脚本名称和参数唯一性（存在同名脚本）和一致性（变更了脚本名称）得不到保证，
可能存在误判的概率。

使用 Global Event 的防重方案只要确保 event name 唯一即可完全避免误判。 
在编码时，可以给 event name 中加入随机 UUID 来确保 event name 唯一。

另外需要注意的是需要把变量 `$Mutex` 申明为 `Global`（或者把该变量作为函数的返回值返回并在外部作用域中持有），
以避免该变量被释放后资源被销毁，导致锁失效。

## 参考文档

1. [Mutex Class](https://learn.microsoft.com/en-us/dotnet/api/system.threading.mutex?view=net-8.0)
