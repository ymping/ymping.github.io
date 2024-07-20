+++
title = 'PowerShell with Windows Job Objects'
date = 2024-07-14T13:19:00+08:00
keywords = ['powershell', 'NamedJobObject', 'JobObject', '作业对象']
tags = ['windows', 'powershell']
draft = false
+++

Windows 的 `Job Objects`（作业对象）允许将进程组作为一个单元进行管理。 作业对象是可访问的、安全的、可共享的对象，用于控制与其关联的进程的属性。
针对某个作业对象执行的操作会影响与该作业对象关联的所有进程。 包括如限制工作集大小和设置进程优先级，或终止与作业对象关联的所有进程等。

本文给出的示例代码的功能为使用 PowerShell 脚本创建并关联进程信息到 `Job Objects`，然后再使用 PowerShell
命令获取与 `Job Objects` 关联的进程信息。

## New `Job Objects` and Assign Process

脚本 `New-JobObject.ps1` 如下，在 PowerShell 中执行命令 `.\New-JobObject.ps1 -Name mydemojobobject` 创建一个名为
`mydemojobobject` 的 `Job Objects` 对象，并次当前进程关联到该对象。

```powershell
param(
    [Parameter(Mandatory = $true, Position = 0, HelpMessage = "Windows Job Object Name")]
    [string] $Name
)

$signature = @"
[DllImport("kernel32.dll", CharSet = CharSet.Unicode)]
public static extern IntPtr CreateJobObject(IntPtr lpJobAttributes, string lpName);

[DllImport("kernel32.dll", SetLastError = true)]
[return: MarshalAs(UnmanagedType.Bool)]
public static extern bool AssignProcessToJobObject(IntPtr hJob, IntPtr hProcess);

[DllImport("kernel32.dll", SetLastError = true)]
public static extern IntPtr GetCurrentProcess();
"@

Add-Type -MemberDefinition $signature -Namespace Win32 -Name JobObject

$jobHandle = [Win32.JobObject]::CreateJobObject([IntPtr]::Zero, "Global\$Name")

if ($jobHandle -eq [IntPtr]::Zero) {
    Write-Error "Failed to create job object"
    Exit 1
}
else {
    Write-Output "Job object '$Name' created successfully"
}

$ok = [Win32.JobObject]::AssignProcessToJobObject($jobHandle, [Win32.JobObject]::GetCurrentProcess())

if ($ok) {
    Write-Output "Current process $PID assigned to job object '$Name' successfully"
}
else {
    Write-Error "Failed to assign current process $PID to job object"
    Exit 1
}

Start-Sleep -Seconds 60
```

执行该脚本：

```powershell
PS C:\Users\admin\Desktop\code\windows_job_objects> .\New-JobObject.ps1 -Name mydemojobobject
Job object 'mydemojobobject' created successfully.
Current process 6216 assigned to job object 'mydemojobobject' successfully.
PS C:\Users\admin\Desktop\code\windows_job_objects>
```

## Get Process by Job Objects

在脚本 `New-JobObject.ps1` 执行期间，新打开一个 PowerShell 执行窗口，通过以下命令获取到关联到指定 `Job Objects` 的进程信息。

```powershell
PS C:\Users\admin> $Obj = Get-CimInstance -ClassName Win32_NamedJobObject -Filter "CollectionID = 'mydemojobobject'"
PS C:\Users\admin>
PS C:\Users\admin> $Obj


Caption             :
CollectionID        : mydemojobobject
Description         :
BasicUIRestrictions : 0
PSComputerName      :



PS C:\Users\admin> Get-CimAssociatedInstance -InputObject $Obj -ResultClassName Win32_Process

ProcessId Name           HandleCount WorkingSetSize VirtualSize
--------- ----           ----------- -------------- -----------
6216      powershell.exe 559         67383296       2204011286528


PS C:\Users\admin>
```

## Get Job Objects by Process

同理，也可以通过进程信息查找到该进程关联的 `Job Objects` 对象。

在脚本 `New-JobObject.ps1` 执行期间，打印了关联到 `mydemojobobject` 作业对象的进程 PID。
可以通过 PID 查找到其关联的 `Job Objects` 信息。

```powershell
PS C:\Users\admin> $Process = Get-CimInstance -ClassName Win32_Process -Filter "ProcessId = 6216"
PS C:\Users\admin>
PS C:\Users\admin> $Process

ProcessId Name           HandleCount WorkingSetSize VirtualSize
--------- ----           ----------- -------------- -----------
6216      powershell.exe 560         67547136       2204013973504


PS C:\Users\admin>
PS C:\Users\admin> Get-CimAssociatedInstance -InputObject $Process -ResultClassName Win32_NamedJobObject


Caption             :
CollectionID        : mydemojobobject
Description         :
BasicUIRestrictions : 0
PSComputerName      :



PS C:\Users\admin>
```
