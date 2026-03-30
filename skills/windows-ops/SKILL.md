---
name: windows-ops
description: Windows服务器运维管理。当用户提到Windows运维、Windows Server、AD域控、IIS管理、Windows防火墙、Windows更新、PowerShell运维、事件日志、Windows性能监控、共享文件夹、Windows服务、注册表、组策略、WinRM、远程桌面、Windows安全加固时使用。
---

# Windows 服务器运维

## 核心巡检项

所有操作优先使用 PowerShell，以管理员权限执行。

### 1. 系统基础信息
```powershell
Get-ComputerInfo | Select-Object CsName, OsName, OsVersion, OsArchitecture, WindowsProductName, WindowsEditionId
systeminfo
hostname
$env:COMPUTERNAME
(Get-WmiObject Win32_OperatingSystem).InstallDate
```

### 2. 资源使用情况
```powershell
# CPU
Get-Counter '\Processor(_Total)\% Processor Time' -SampleInterval 2 -MaxSamples 3
$env:NUMBER_OF_PROCESSORS
wmic cpu get Name,NumberOfCores,NumberOfLogicalProcessors

# 内存
Get-Counter '\Memory\Available MBytes'
Get-Counter '\Memory\% Committed Bytes In Use'
$os = Get-CimInstance Win32_OperatingSystem
[PSCustomObject]@{
    TotalGB = [math]::Round($os.TotalVisibleMemorySize/1MB, 2)
    FreeGB  = [math]::Round($os.FreePhysicalMemory/1MB, 2)
    UsedPercent = [math]::Round(($os.TotalVisibleMemorySize - $os.FreePhysicalMemory) / $os.TotalVisibleMemorySize * 100, 1)
}

# 磁盘
Get-CimInstance Win32_LogicalDisk | Where-Object DriveType -eq 3 | Format-Table DeviceID, @{N='TotalGB';E={[math]::Round($_.Size/1GB,2)}}, @{N='FreeGB';E={[math]::Round($_.FreeSpace/1GB,2)}}, @{N='UsedPercent';E={[math]::Round(($_.Size-$_.FreeSpace)/$_.Size*100,1)}}

# 大文件查找
Get-ChildItem c:\ -Recurse -File -ErrorAction SilentlyContinue | Sort-Object Length -Descending | Select-Object -First 20 FullName, @{N='SizeMB';E={[math]::Round($_.Length/1MB)}}
```

### 3. 系统负载与进程
```powershell
# 高CPU进程
Get-Process | Sort-Object CPU -Descending | Select-Object -First 10 Name, Id, CPU, @{N='MemMB';E={[math]::Round($_.WorkingSet64/1MB)}}

# 高内存进程
Get-Process | Sort-Object WorkingSet64 -Descending | Select-Object -First 10 Name, Id, @{N='MemMB';E={[math]::Round($_.WorkingSet64/1MB)}}

# 服务状态
Get-Service | Where-Object Status -eq Running | Select-Object Name, DisplayName, Status | Format-Table -AutoSize
Get-Service | Where-Object Status -eq Stopped | Select-Object Name, DisplayName | Format-Table -AutoSize
```

### 4. 网络配置
```powershell
Get-NetIPAddress -AddressFamily IPv4 | Format-Table IPAddress, InterfaceAlias, PrefixLength
Get-NetRoute | Where-Object DestinationPrefix -eq '0.0.0.0/0'
Get-DnsClientServerAddress -AddressFamily IPv4
Test-NetConnection -ComputerName 8.8.8.8 -Traceroute
netstat -an | findstr ESTABLISHED
Get-NetFirewallRule -Enabled True | Where-Object Direction -eq Inbound | Select-Object DisplayName, Profile, Action | Format-Table
```

### 5. 安全检查
```powershell
# Windows更新状态
Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 10 HotFixID, Description, InstalledOn

# 防病毒状态
Get-MpComputerStatus 2>$null | Select-Object RealTimeProtectionEnabled, AntivirusEnabled, AntispywareEnabled

# RDP安全
Get-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name UserAuthentication
Get-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name fDenyTSConnections

# 账户策略
net accounts
# 锁定账户检查
net user | findstr "锁定"

# 审计策略
auditpol /get /category:*

# 计划任务
Get-ScheduledTask | Where-Object State -eq Ready | Select-Object TaskName, TaskPath, State
```

### 6. 事件日志
```powershell
# 系统错误（近24小时）
Get-WinEvent -FilterHashtable @{LogName='System'; Level=2; StartTime=(Get-Date).AddHours(-24)} -MaxEvents 30 | Format-Table TimeCreated, Id, ProviderName, Message -Wrap

# 应用错误
Get-WinEvent -FilterHashtable @{LogName='Application'; Level=2; StartTime=(Get-Date).AddHours(-24)} -MaxEvents 30 | Format-Table TimeCreated, Id, ProviderName, Message -Wrap

# 安全日志（登录失败）
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4625; StartTime=(Get-Date).AddHours(-24)} -MaxEvents 20 | Format-Table TimeCreated, Message -Wrap

# 关键事件（服务崩溃等）
Get-WinEvent -FilterHashtable @{LogName='System'; Level=1; StartTime=(Get-Date).AddDays(-7)} -MaxEvents 20
```

## 常见运维操作

### IIS 管理
```powershell
Import-Module WebAdministration
# 站点状态
Get-Website | Select-Object Name, State
# 应用池
Get-WebAppPoolState
# 重启应用池
Restart-WebAppPool -Name "DefaultAppPool"
# IIS日志
Get-WebLog -Name "IIS日志路径"
```

### Windows 防火墙
```powershell
# 开放端口
New-NetFirewallRule -DisplayName "Allow Port 8080" -Direction Inbound -LocalPort 8080 -Protocol TCP -Action Allow
# 关闭端口
Remove-NetFirewallRule -DisplayName "Allow Port 8080"
# 查看规则
Get-NetFirewallPortFilter | Get-NetFirewallRule
```

### 性能监控
```powershell
# 实时监控
Get-Counter '\Processor(_Total)\% Processor Time', '\Memory\Available MBytes', '\PhysicalDisk(_Total)\% Disk Time' -SampleInterval 2 -Continuous
# 导出性能数据
Get-Counter -ListSet * | Select-Object CounterSetName
```

### 服务管理
```powershell
# 服务配置
Set-Service -Name "serviceName" -StartupType Automatic
Set-Service -Name "serviceName" -Status Running
# 查看服务依赖
Get-Service -Name "serviceName" | Select-Object -ExpandProperty RequiredServices
```

## 巡检报告模板

```markdown
# Windows 服务器巡检报告

**主机名:** xxx | **IP:** x.x.x.x | **系统:** Windows Server 2019 | **巡检时间:** 2026-03-30 16:00

## 1. 资源概览
| 指标 | 当前值 | 阈值 | 状态 |
|------|--------|------|------|
| CPU使用率 | | < 80% | ✅/⚠️ |
| 内存使用率 | | < 85% | ✅/⚠️ |
| 磁盘C:使用率 | | < 85% | ✅/⚠️ |
| 系统运行时间 | | | |

## 2. 安全状态
| 检查项 | 状态 |
|--------|------|
| Windows Defender | ✅/❌ |
| RDP安全认证 | ✅/❌ |
| 防火墙状态 | ✅/❌ |
| 密码策略 | 合规/不合规 |

## 3. 事件日志摘要
## 4. 服务状态
## 5. Windows更新
## 6. 异常与建议
```
