+++
title = 'PowerShell Interop C#'
date = 2024-07-12T22:49:00+08:00
keywords = ['powershell', 'C#', 'CSharp', 'JobObject', '作业对象']
tags = ['windows', 'powershell']
draft = false
+++

PowerShell 主要使用 C# 编写，其内部运行环境也是基于 .NET Framework，因此 PowerShell 与 .NET 交互十分简单。

本文给出了一个使用 PowerShell 脚本调用 C# 代码操作 `Job Objects` 的示例，该示例代码的功能是传入一个 `Job Objects` 的名称，
返回与该 `Job Objects` 关联的所有进程的 PID 列表。

> 示例功能使用纯 PowerShel 实现请参考文章 [named_job_object](/powershell/named_job_object/)

## 示例代码

文件 `JobObjectHelper.cs` 与 `Get-JobObjectPID.ps1` 需处理同一目录。

`JobObjectHelper.cs` 中接收一个 `Job Objects` 名称参数，并返回与该 `Job Objects` 关联的进程 PID 列表。
`Get-JobObjectPID.ps1` 中使用 `Add-Type` 命令将 C# 代码中定义的 `JobObjectHelper` 类添加到当前 PowerShell 
的当前作用域并调用类方法，输出打印进程 PID 列表。

### JobObjectHelper.cs

```cs
using System;
using System.Runtime.InteropServices;

public class JobObjectHelper
{
    [DllImport("kernel32.dll")]
    public static extern IntPtr OpenJobObject(int dwDesiredAccess, bool bInheritHandle, string lpName);

    [DllImport("kernel32.dll", SetLastError = true)]
    public static extern bool QueryInformationJobObject(IntPtr hJob, JobObjectInfoClass infoClass, IntPtr lpJobObjectInfo, int cbJobObjectInfoLength, out int lpReturnLength);

    public enum JobObjectInfoClass
    {
        JobObjectBasicProcessIdList = 3
    }

    public const int JOB_OBJECT_QUERY = 0x0004;

    [StructLayout(LayoutKind.Sequential)]
    struct JOBOBJECT_BASIC_PROCESS_ID_LIST
    {
        public uint NumberOfAssignedProcesses;
        public uint NumberOfProcessIdsInList;
        [MarshalAs(UnmanagedType.ByValArray, SizeConst = 2048)]
        public ulong[] ProcessIdList;
    }

    public static uint[] GetProcessIdsFromJobObject(string jobObjectName)
    {
        IntPtr hJob = OpenJobObject(JOB_OBJECT_QUERY, false, "Global\\" + jobObjectName);
        if (hJob == IntPtr.Zero)
        {
            throw new System.Exception("Failed to open Job Object. Error code: " + Marshal.GetLastWin32Error());
        }

        JOBOBJECT_BASIC_PROCESS_ID_LIST info = new JOBOBJECT_BASIC_PROCESS_ID_LIST();
        int bufferSize = Marshal.SizeOf(typeof(JOBOBJECT_BASIC_PROCESS_ID_LIST));
        IntPtr pInfo = Marshal.AllocHGlobal(bufferSize);
        int returnLength = 0;

        if (!QueryInformationJobObject(hJob, JobObjectInfoClass.JobObjectBasicProcessIdList, pInfo, bufferSize, out returnLength))
        {
            Marshal.FreeHGlobal(pInfo);
            throw new System.Exception("Failed to query Job Object information. Error code: " + Marshal.GetLastWin32Error());
        }

        info = (JOBOBJECT_BASIC_PROCESS_ID_LIST)Marshal.PtrToStructure(pInfo, typeof(JOBOBJECT_BASIC_PROCESS_ID_LIST));
        Marshal.FreeHGlobal(pInfo);

        uint[] ret = new uint[info.NumberOfProcessIdsInList];
        for (int i = 0; i < info.NumberOfProcessIdsInList; i++)
        {
            ret[i] = (uint)info.ProcessIdList[i];
        }
        return ret;
    }
}
```

### Get-JobObjectPID.ps1

```powershell
param(
    [Parameter(Mandatory = $true, Position = 0, HelpMessage = "Windows Job Object Name")]
    [string] $JobObjectName
)

function Get-JobObjectProcessIds {
    param (
        [Parameter(Mandatory = $true)]
        [string] $JobObjectName
    )

    if (-not ([System.Management.Automation.PSTypeName]'JobObjectHelper').Type) {
        Add-Type -Path "$PSScriptRoot\JobObjectHelper.cs"
        if ($? -eq $false) {
            Write-Log "powershell command failed: Add-Type -Path $PSScriptRoot\JobObjectHelper.cs"
            return $null
        }
    }

    try {
        return [JobObjectHelper]::GetProcessIdsFromJobObject($JobObjectName)
    }
    catch {
        Write-Host "An unexpected error occurred during get process id in JobObject $JobObjectName : $_"
        return $null
    }
}

Get-JobObjectProcessIds $JobObjectName
```

## 参考

1. about [Add-Type](https://learn.microsoft.com/en-us/powershell/module/microsoft.powershell.utility/add-type)
2. about [QueryInformationJobObject](https://learn.microsoft.com/en-us/windows/win32/api/jobapi2/nf-jobapi2-queryinformationjobobject)
