+++
title = 'PowerShell Multithreading'
date = 2024-07-14T17:23:00+08:00
keywords = ['powershell', 'multithreading', 'runspace']
tags = ['windows', 'powershell']
draft = false
+++

本文介绍 PowerShell 中多线程相关内容。

## Runspace

在 PowerShell 中，Runspace 是一个独立的执行环境，每个 Runspace 都有自己的变量、函数和状态，互相独立，各 Runspace 之间可以独立的运行
PowerShell 代码，相当于一个独立的会话或线程，允许在同一个进程中并行地运行多个 PowerShell 命令或脚本。

## 示例代码

该示例代码 `multithreading.ps1` 模拟了5个线程同时进行某种耗时操作，可以通过打印的时间确认5个线程在并行运行。

```powershell
# 最大线程数量
$MaxThreadNo = 5
# ShareContext 用于各线程间共享数据
$ShareContext = [Hashtable]::Synchronized(@{ })

$ScriptBlock = {
    param (
        [Parameter(ValueFromRemainingArguments = $true)]
        [string]$ThreadNo
    )

    $TID = [System.Threading.Thread]::CurrentThread.ManagedThreadId
    $TS1 = Get-Date
    Write-Host "$(Get-Date -Date $TS1 -Format 'yyyy-MM-dd HH:mm:ss') $TID - thread No: $ThreadNo running"

    $Sec = Get-Random -Minimum 5 -Maximum 10
    Start-Sleep -Seconds $Sec
    $ShareContext[$TID] = "$TID do something took $Sec seconds"

    $TS2 = Get-Date
    Write-Host "$(Get-Date -Date $TS2 -Format 'yyyy-MM-dd HH:mm:ss') $TID - thread No: $ThreadNo finish, took $(($TS2 - $TS1).TotalSeconds) seconds"
}

# 创建 InitialSessionState 对象，用于初始化 runspace 执行环境
$InitialSessionState = [InitialSessionState]::CreateDefault()
$VariableEntry = New-Object System.Management.Automation.Runspaces.SessionStateVariableEntry -ArgumentList @("ShareContext", $ShareContext, "A thread safe shared context across runspaces")
$initialSessionState.Variables.Add($VariableEntry)

# 创建 Runspace Pool
$RunspacePool = [RunspaceFactory]::CreateRunspacePool(1, $MaxThreadNo, $InitialSessionState, $Host)
$RunspacePool.Open()

$TS1 = Get-Date

# 准备 Runspaces
$Threads = @()
for ($i = 0; $i -lt $MaxThreadNo; $i++) {
    $ps = [powershell]::Create().AddScript($ScriptBlock).AddParameter("ThreadNo", $i)
    $ps.RunspacePool = $RunspacePool
    $status = $ps.BeginInvoke()

    $Threads += [PSCustomObject]@{ Powershell = $ps; Status = $status }
}

# 等待所有 Runspace 完成
foreach ($t in $Threads) {
    $t.Powershell.EndInvoke($t.Status)
    $t.Powershell.Dispose()
}

$TS2 = Get-Date
$TID = [System.Threading.Thread]::CurrentThread.ManagedThreadId
Write-Host "$(Get-Date -Date $TS2 -Format 'yyyy-MM-dd HH:mm:ss')  $TID - wait thread finish took $(($TS2 - $TS1).TotalSeconds) seconds"

# 关闭 Runspace Pool
$RunspacePool.Close()
$RunspacePool.Dispose()

# 输出 ShareContext 的内容
Write-Host "---------- Share Context ----------"
foreach ($i in $ShareContext.GetEnumerator() )
{
  Write-Host "$($i.Name) : $($i.Value)"
}
```

在使用多线程时需要注意：

1. 示例中的 `$InitialSessionState` 用于初始化 `$RunspacePool` 的执行环境，多个 `Powershell` 实例使用同一个`RunspacePool`，
   用于在多线程间共享上下文，如果不需要共享上下文，这两个变量是非必须。
2. PowerShell 中的 Class 默认情况与申明该 Class 的 Runspace 间具备亲和性(affinity)。类上的方法调用会被编组回创建它的运行空间执行，
   这可能会破坏 Runspace 的状态或导致死锁。如果需要在 Runspace 中使用到 Class，在 PowerShell 5.x 中， 
   可以在执行任务的每个 Runspace 中单独申明 Class，在 PowerShell 7.x 中，可以在定义类时添加 `NoRunspaceAffinity` 属性以避免亲和性问题。
   更多请参考 [NoRunspaceAffinity attribute](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.core/about/about_classes?view=powershell-7.4#norunspaceaffinity-attribute)
3. 如在线程中遇到 `Write-Host` 命令输出内容在终端无显示时，可以尝试将主线程的 `$Host` 变量注入子线程中，假设注入后名称为 `$OuterHost`，
   在子线程中可以使用 `$OuterHost.UI.WriteLine("Hello")` 方法进行输出打印。

执行示例代码：

```text
PS C:\Users\admin\Desktop\code\multithreading> powershell.exe -ExecutionPolicy Bypass -File .\multithreading.ps1
2024-07-16 23:29:55 17 - thread No: 2 running
2024-07-16 23:29:55 15 - thread No: 0 running
2024-07-16 23:29:55 16 - thread No: 1 running
2024-07-16 23:29:55 19 - thread No: 3 running
2024-07-16 23:29:55 20 - thread No: 4 running
2024-07-16 23:30:00 16 - thread No: 1 finish, took 5.0827461 seconds
2024-07-16 23:30:00 20 - thread No: 4 finish, took 5.0181769 seconds
2024-07-16 23:30:01 19 - thread No: 3 finish, took 6.0531642 seconds
2024-07-16 23:30:03 15 - thread No: 0 finish, took 8.0786149 seconds
2024-07-16 23:30:04 17 - thread No: 2 finish, took 9.0707772 seconds
2024-07-16 23:30:04  7 - wait thread finish took 9.1223416 seconds
---------- Share Context ----------
20 : 20 do something took 5 seconds
19 : 19 do something took 6 seconds
17 : 17 do something took 9 seconds
16 : 16 do something took 5 seconds
15 : 15 do something took 8 seconds
PS C:\Users\admin\Desktop\code\multithreading>
```

## 参考

1. about [PowerShell Class](https://learn.microsoft.com/en-us/dotnet/api/system.management.automation.powershell)
2. about [InitialSessionState Class](https://learn.microsoft.com/en-us/dotnet/api/system.management.automation.runspaces.initialsessionstate)
3. about [RunspaceFactory Class](https://learn.microsoft.com/en-us/dotnet/api/system.management.automation.runspaces.runspacefactory)
