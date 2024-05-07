---
title: Client Deployment
weight: 400
pre: "<b>2.4. </b>"
---

You can deploy using a number of methods, some are covered in [Client Configuration](https://rustdesk.com/docs/en/self-host/client-configuration/).

Alternatively you can use mass deployment scripts with your RMM, Intune, etc. The ID and password are output by the script. You should collect this or split it into different scripts to collect the ID and password.

The permanent password can be changed from random to one you prefer using by changing the content inside `()` after `rustdesk_pw` to your preferred password for PowerShell and the corresponding line for any other platform.

### PowerShell

```ps
$ErrorActionPreference = 'silentlycontinue'

# Assign the value random password to the password variable
Write-Output "Generating Password"
$rustdesk_pw=(-join ((65..90) + (97..122) | Get-Random -Count 12 | ForEach-Object {[char]$_}))

# Get your config string from your Web portal and Fill Below
$rustdesk_cfg="configstring"

################################### Please Do Not Edit Below This Line #########################################

# Run as administrator and stays in the current directory
if (-Not ([Security.Principal.WindowsPrincipal][Security.Principal.WindowsIdentity]::GetCurrent()).IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator))
{
    if ([int](Get-CimInstance -Class Win32_OperatingSystem | Select-Object -ExpandProperty BuildNumber) -ge 6000)
    {
        Start-Process PowerShell -Verb RunAs -ArgumentList "-NoProfile -ExecutionPolicy Bypass -Command `"cd '$pwd'; & '$PSCommandPath';`"";
        Exit;
    }
}

$rdver = ((Get-ItemProperty  "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\RustDesk\").Version)

Write-Output "Getting latest version and download url from Github repository"
$repo = "rustdesk/rustdesk"
$filenamePattern = "*x86_64.exe"
$releasesUri = "https://api.github.com/repos/$repo/releases/latest"
$downloadUri = ((Invoke-RestMethod -Method GET -Uri $releasesUri).assets | Where-Object name -like $filenamePattern ).browser_download_url
$latestversion = (Invoke-RestMethod -Method GET -Uri $releasesUri).name

if ($rdver -eq $latestversion)
{
    Write-Output "RustDesk $rdver is the newest version"
    Exit
}

Set-Location "$env:TEMP"

Write-Output "Downloading latest Rustdesk"
Invoke-WebRequest $downloadUri -Outfile "rustdesk.exe"

Write-Output "Installing Rustdesk"
Start-Process .\rustdesk.exe --silent-install
Start-Sleep -Seconds 15

$ServiceName = 'Rustdesk'
$arrService = Get-Service -Name $ServiceName -ErrorAction SilentlyContinue

if ($null -eq $arrService)
{
    Write-Output "Installing service"
    Set-Location "$env:ProgramFiles\RustDesk"
    Start-Process .\rustdesk.exe --install-service -wait -Verbose
    Start-Sleep -Seconds 20
}

while ($arrService.Status -ne 'Running')
{
    Write-Output "Starting service"
    Start-Service $ServiceName
    Start-Sleep -Seconds 5
    $arrService.Refresh()
}

Set-Location "$env:ProgramFiles\RustDesk\"
Write-Output "Reading Rustdesk Id"
.\rustdesk.exe --get-id | Write-Output -OutVariable rustdesk_id

Write-Output "Start Rustdesk"
.\rustdesk.exe --config $rustdesk_cfg

Write-Output "Setting Rustdesk Password"
.\rustdesk.exe --password $rustdesk_pw

Write-Output "..............................................."
# Show the value of the ID Variable
Write-Output "RustDesk ID: $rustdesk_id"

# Show the value of the Password Variable
Write-Output "Password: $rustdesk_pw"
Write-Output "..............................................."
```
