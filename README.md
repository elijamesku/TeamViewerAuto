# TeamViewer Auto-Updater 

This solution automates the installation and ongoing updating of **TeamViewer** on Windows machines using Microsoft Intune. It ensures non-admin users receive the latest version of TeamViewer without UAC prompts by using system-level deployment and scheduled task automation


```                                
      _//        _//           _//                                                _//                    _//                          _///////    _////////      _/       _/////    _//       _//_////////
_//        _//           _//                                                _//                    _//  _//                     _//    _//  _//           _/ //     _//   _// _/ _//   _///_//      
_//   _/   _//   _//     _//   _///   _//    _/// _// _//    _//          _/_/ _/   _//          _/_/ _/_//        _//          _//    _//  _//          _/  _//    _//    _//_// _// _ _//_//      
_//  _//   _// _/   _//  _// _//    _//  _//  _//  _/  _// _/   _//         _//   _//  _//         _//  _/ _/    _/   _//       _/ _//      _//////     _//   _//   _//    _//_//  _//  _//_//////  
_// _/ _// _//_///// _// _//_//    _//    _// _//  _/  _//_///// _//        _//  _//    _//        _//  _//  _//_///// _//      _//  _//    _//        _////// _//  _//    _//_//   _/  _//_//      
_/ _/    _////_/         _// _//    _//  _//  _//  _/  _//_/                _//   _//  _//         _//  _/   _//_/              _//    _//  _//       _//       _// _//   _// _//       _//_//      
_//        _//  _////   _///   _///   _//    _///  _/  _//  _////            _//    _//             _// _//  _//  _////         _//      _//_////////_//         _//_/////    _//       _//_////////
                                                                                                                                                                              
                                                                                                
```
## Features

- Silently installs TeamViewer  
- Automatically uninstalls any older versions  
- Downloads the latest version from the official website  
- Registers a scheduled task that re-checks for updates every 14 days  
- Designed to run under **SYSTEM** context — no admin rights required  
- Logs actions to: `C:\ProgramData\TeamViewerUpdater\install_log.txt`


## Requirements

- Windows 10 or later  
- Microsoft Intune environment  
- [IntuneWinAppUtil.exe](https://learn.microsoft.com/en-us/mem/intune/apps/apps-win32-app-management#prepare-the-win32-app-content-for-upload)


## Folder Structure

```
TeamViewerAutoWin32/
├── Install-TeamViewer.ps1
├── Register-TeamViewerUpdaterTask.ps1
├── install.cmd
├── Output/
```


## How It Works

### Install-TeamViewer.ps1

```powershell
# This script uninstalls old versions of TeamViewer,
# downloads the latest version, installs it silently,
# and logs everything to a log file.

$LogDir = "$env:ProgramData\TeamViewerUpdater"
$LogFile = "$LogDir\install_log.txt"
$InstallerPath = "$env:TEMP\TeamViewer_Setup_x64.exe"
$DownloadUrl = "https://download.teamviewer.com/download/TeamViewer_Setup_x64.exe"
$ExePath = "C:\Program Files\TeamViewer\TeamViewer.exe" 
$LatestVersion = "15.66.5"

New-Item -ItemType Directory -Path $LogDir -Force | Out-Null

Function Write-Log {
    param([string]$Message)
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    Add-Content -Path $LogFile -Value "$timestamp : $Message"
}

Write-Log "==== Starting TeamViewer Install Script ===="

$NeedsInstall = $true
if (Test-Path $ExePath) {
    $InstalledVersion = (Get-Item $ExePath).VersionInfo.FileVersion
    Write-Log "Installed TeamViewer version: $InstalledVersion"

    if ([version]$InstalledVersion -ge [version]$LatestVersion) {
        Write-Log "TeamViewer is already up to date."
        $NeedsInstall = $false
    } else {
        Write-Log "Outdated version detected. Preparing to update..."
    }
} else {
    Write-Log "TeamViewer not detected. Preparing to install..."
}

if ($NeedsInstall) {
    Write-Log "Uninstalling old versions of TeamViewer..."
    $uninstallers = Get-WmiObject -Query "SELECT * FROM Win32_Product WHERE Name LIKE 'TeamViewer%'" -ErrorAction SilentlyContinue
    foreach ($app in $uninstallers) {
        Write-Log "Uninstalling: $($app.Name)"
        try {
            $app.Uninstall() | Out-Null
            Write-Log "Successfully uninstalled $($app.Name)"
        } catch {
            Write-Log "Error uninstalling $($app.Name): $_"
        }
    }

    Write-Log "Downloading latest TeamViewer installer..."
    try {
        Invoke-WebRequest -Uri $DownloadUrl -OutFile $InstallerPath -UseBasicParsing
        Write-Log "Downloaded installer to $InstallerPath"
    } catch {
        Write-Log "Failed to download TeamViewer: $_"
        Exit 1
    }

    Write-Log "Installing TeamViewer silently..."
    try {
        Start-Process -FilePath $InstallerPath -ArgumentList "/S" -Wait
        Write-Log "TeamViewer installed successfully."
    } catch {
        Write-Log "Installation failed: $_"
        Exit 1
    }
}

Write-Log "==== TeamViewer Install Script Complete ===="
```

## Register-TeamViewerUpdaterTask.ps1 -
```powershell

# Registers a scheduled task that runs the TeamViewer install script
# every 14 days as SYSTEM, even if no user is logged in.

$TaskName = "TeamViewer Auto-Update"
$ScriptPath = "$env:ProgramData\TeamViewerUpdater\Install-TeamViewer.ps1"

# Ensure the install script is present in ProgramData
Copy-Item -Path "$PSScriptRoot\Install-TeamViewer.ps1" -Destination $ScriptPath -Force

$Action = New-ScheduledTaskAction -Execute "powershell.exe" -Argument "-ExecutionPolicy Bypass -File `"$ScriptPath`""
$Trigger = New-ScheduledTaskTrigger -Daily -DaysInterval 14 -At 2am
$Principal = New-ScheduledTaskPrincipal -UserId "SYSTEM" -RunLevel Highest

Register-ScheduledTask -TaskName $TaskName -Action $Action -Trigger $Trigger -Principal $Principal -Description "Checks and updates TeamViewer every 14 days" -Force
```

## install.cmd
```
@echo off
powershell.exe -ExecutionPolicy Bypass -File "%~dp0Install-TeamViewer.ps1"
powershell.exe -ExecutionPolicy Bypass -File "%~dp0Register-TeamViewerUpdaterTask.ps1"
```

### Packaging Instructions
Use Microsoft’s packaging tool to create the .intunewin file:
```
.\IntuneWinAppUtil.exe -c "C:\Path\To\TeamViewer" -s "install.cmd" -o "C:\Path\To\TeamViewer\Output"
```

## Intune App Configuration
App Type: Windows app (Win32)
  
**Install command:**

``install.cmd
``

**Uninstall command:**
```
powershell.exe -ExecutionPolicy Bypass -Command "Get-WmiObject -Query \"SELECT * FROM Win32_Product WHERE Name LIKE 'TeamViewer%'\" | ForEach-Object { $_.Uninstall() }"
```

**Install behavior:** System  

**Detection rules:** Use custom PowerShell detection script below  


## Detection Script
```
$exePath = "C:\Program Files\TeamViewer\TeamViewer.exe"

if (-Not (Test-Path $exePath)) {
    exit 1
}

$version = (Get-Item $exePath).VersionInfo.FileVersion
if ([version]$version -ge [version]"15.66.5") {
    exit 0
} else {
    exit 1
}
```

## Local Validation (PowerShell)

**Check if the scheduled task exists:**  

```
Get-ScheduledTask -TaskName "TeamViewer Auto-Update"
```

**View last and next run times:**  

```
Get-ScheduledTaskInfo -TaskName "TeamViewer Auto-Update"
```

**Run the task manually:**  

```
Start-ScheduledTask -TaskName "TeamViewer Auto-Update"
```

**View the trigger settings:**  

```
(Get-ScheduledTask -TaskName "TeamViewer Auto-Update").Triggers
```

## Summary of script

- Removes older versions of TeamViewer

- Installs the latest silently

- Auto-updates every 14 days via SYSTEM task

- Fully Intune-compatible via Win32 app model

- No admin rights needed by end users
