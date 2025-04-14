# ready-room-selfhosted

[ready-room.net](https://ready-room.net/) was a service to host DCS World servers on demand on AWS that closed down on 1/2/2025. ready-room-selfhosted takes the original code and lessons learnt to deliver a script that can be used for self hosting on AWS.

[Discord](https://discord.gg/hURRqGP)

## Hosting principle

The main idea of ready-room.net and ready-room-selfhosted is to optimise cost for relativley infrequent hosting - i.e. squadron sessions for a few hours a couple of times a week / not 24x7. The biggest expense for relatively infrequent hosting is persistent storage (disks) becuase its a 24x7 expense. Optimising cost in this case means avoiding persistent storage. ready-room.net did it by caching files in s3, ready-room-selfhosted does it by installing from scratch each time. Installing from scratch each time will take longer than using cached s3 files, but since DCS introduced the modular installer for DCS World server it should be managable since only the required map module will need to be downloaded (as opposed to all of them). An added benefit of scratch installing every time is that the server can be installed onto the physically co-located SSD storage for best performance.

## How To

Powershell script to install and start DCS

```
### Work in progress

<powershell>
$terrain = "Caucasus terrain"
$WindowsAdminPassword = "dcsadmin"
$missionUrl = "https://ready-room-public.s3.ap-southeast-2.amazonaws.com/tacview_test.miz"
$missionName = "tacview_test.miz"

# Consider keeping exports disabled to minimise bandwidth
$allow_object_export = "false"
$allow_sensor_export = "false"
$name = "Test Server"
$password = "mypassword"
$allow_trial_only_clients = "true"
$maxPlayers = "64"

# Save config
$config=@"
`$terrain="$terrain"
`$WindowsAdminPassword="$WindowsAdminPassword"
`$missionUrl = "$missionUrl"
`$missionName = "$missionName"
`$allow_object_export = "$allow_object_export"
`$allow_sensor_export = "$allow_sensor_export"
`$name = "$name"
`$password = "$password"
`$allow_trial_only_clients = "$allow_trial_only_clients"
`$maxPlayers = "$maxPlayers"
"@
 echo $config | Out-File -Encoding ascii -Filepath "C:\config.ps1"

# Configure auto-login on startup and run the install script as an on statup (login) script
# This creates a desktop environment for the DCS GUI installer to run in
# https://learn.microsoft.com/en-au/troubleshoot/windows-server/user-profiles-and-logon/turn-on-automatic-logon

$newItemPropertySplat = @{
    Path = 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon'
    Name = 'AutoAdminLogon'
    PropertyType = 'String'
    Value = '1'
}
New-ItemProperty @newItemPropertySplat

$newItemPropertySplat = @{
    Path = 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon'
    Name = 'DefaultUserName'
    PropertyType = 'String'
    Value = 'Administrator'
}
New-ItemProperty @newItemPropertySplat

$newItemPropertySplat = @{
    Path = 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon'
    Name = 'DefaultPassword'
    PropertyType = 'String'
    Value = $WindowsAdminPassword
}
New-ItemProperty @newItemPropertySplat

# Disable windows password complexity
secedit /export /cfg c:\secpol.cfg
(gc C:\secpol.cfg).replace("PasswordComplexity = 1", "PasswordComplexity = 0") | Out-File C:\secpol.cfg
secedit /configure /db c:\windows\security\local.sdb /cfg c:\secpol.cfg /areas SECURITYPOLICY

# Set windows password
net user Administrator $WindowsAdminPassword


# Install startup (aka login) script
$StartupScript=@'
# Source config
. C:\config.ps1

$installStart = [System.DateTime]::Now

Write-Host "Started: $installStart"

# Format SSD
Get-Disk | Where partitionstyle -eq 'raw' | Initialize-Disk -PartitionStyle MBR -PassThru | New-Partition -DriveLetter Z  -UseMaximumSize | Format-Volume -FileSystem NTFS -Confirm:$false

# Download and install vc++ redistributable
$webClient = [System.Net.WebClient]::new()
$webClient.DownloadFile("https://aka.ms/vs/17/release/vc_redist.x86.exe", "Z:\vc_redist.x86.exe")
# Headless install mode
Z:\vc_redist.x86.exe /q

# Download and install DCS Dedicated Server
$Html = $webClient.DownloadString('https://www.digitalcombatsimulator.com/en/downloads/world/server/')
$ExeRelLink = [Regex]::Matches($html, '<a href="(.*?)"') | % {$_.Groups[1].Value} | ? {$_.StartsWith("/upload")}
$DownloadLink = "https://www.digitalcombatsimulator.com" + $ExeRelLink

$webClient.DownloadFile($DownloadLink, "Z:\DCS_World_Server_modular.exe")
Write-Host "Starting DCS installer"
Z:\DCS_World_Server_modular.exe

Write-Host "Starting DCS installer"
$webClient = [System.Net.WebClient]::new()
$webClient.DownloadFile("https://download.visualstudio.microsoft.com/download/pr/6f043b39-b3d2-4f0a-92bd-99408739c98d/fa16213ea5d6464fa9138142ea1a3446/dotnet-sdk-8.0.407-win-x64.exe", "Z:\dotnet-sdk-8.0.407-win-x64.exe")

$job = Start-Job {Z:\dotnet-sdk-8.0.407-win-x64.exe /q}
Wait-Job $job

$webClient.DownloadFile("https://download.visualstudio.microsoft.com/download/pr/2d6bb6b2-226a-4baa-bdec-798822606ff1/9b7b8746971ed51a1770ae4293618187/ndp48-web.exe", "Z:\ndp48-web.exe")

$job = Start-Job {Z:\ndp48-web.exe /q}
Wait-Job $job

$webClient.DownloadFile("https://github.com/microsoft/accessibility-insights-windows/releases/download/v1.1.2924.01/AccessibilityInsights.msi", "Z:\AccessibilityInsights.msi")

function Poll {
    param (
        $expression,
        $predicate,
        $interval,
        $timeout,
        $delay,
        $reportingInterval
    )

    $start = [System.DateTime]::Now;
    $lastReport = [System.DateTime]::Now;

    Start-Sleep -Milliseconds $delay.TotalMilliseconds

    $result = $expression.Invoke();
    while(!($predicate.Invoke($result)) ) {

        if( ([System.DateTime]::Now.Subtract($start)).CompareTo($timeout) -gt 0) {
            throw [System.TimeoutException] "timed out: $expression"
        }

        Start-Sleep -Milliseconds $interval.TotalMilliseconds

        $result = $expression.Invoke();

        if( ([System.DateTime]::Now.Subtract($lastReport)).CompareTo($reportingInterval) -gt 0) {
            Write-Host "Polling... $expression - last result: $result"
            $lastReport = [System.DateTime]::Now;
        }
    }

    $result
}

$interval = [System.TimeSpan]::FromMilliseconds(1000)
$timeout = [System.TimeSpan]::FromMilliseconds(20000)
$delay = [System.TimeSpan]::FromMilliseconds(500)
$reportingInterval = [System.TimeSpan]::FromMilliseconds(3000)

Write-Host "Wait for dotnet to be available"
for ($i = 1; $i -le 10; $i++) {
    $env:Path = [System.Environment]::GetEnvironmentVariable("Path","Machine")
    [System.Environment]::GetEnvironmentVariable("Path","User")
    $j = dotnet
    if($j) {
        Write-Host "dotnet available"
        break
    }
    Write-Host "Waiting for dotnet"
    Start-Sleep -Milliseconds 1000
}

Write-Host "installing FlaUI dependencies"

cd Z:/
Set-Location (New-Item -Type Directory assemblies)

# Create a dummy project ("-f netstandard2.0" to create a compatible .cs file)
# Poll becuase sdk install is async
$timeout = [System.TimeSpan]::FromMilliseconds(180000)
Poll { dotnet new classlib -f netstandard2.0 } { param($i) $i } $interval $timeout $delay $reportingInterval
$timeout = [System.TimeSpan]::FromMilliseconds(20000)

# Target .NET 4.8 for compatibility with both PowerShell 5.1 and PowerShell (Core) 7+
(Get-Content assemblies.csproj).Replace('netstandard2.0', 'net48') | Set-Content assemblies.csproj

# Download the assemblies from NuGet
dotnet add package FlaUI.UIA3 --version 5.0.0

# Copy all assemblies including dependencies into Release directory
dotnet publish -c Release

Add-Type -Path z:\assemblies\bin\Release\net48\publish\FlaUI.UIA3.dll

Write-Host "FlaUI dependencies installed"

# Complete the DCS installer window
function Nested-Children {
    param (
        $element
    )

    @($element) + ($element.FindAllChildren() | ForEach-Object { Nested-Children($_) })
}

$automation = [FlaUI.UIA3.UIA3Automation]::new()

(Poll { (Get-Process DCS_World_Server_modular.tmp).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.Name -eq "OK" } } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval).Click("true")

(Poll { (Get-Process DCS_World_Server_modular.tmp).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.Name -eq "I accept the agreement" } } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval).Click("true")

(Poll { (Get-Process DCS_World_Server_modular.tmp).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.Name -eq "Next" } } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval).Click("true")

(Poll { (Get-Process DCS_World_Server_modular.tmp).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.ControlType -eq "Edit" } } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval).Patterns.Value.Pattern.SetValue("Z:\DCS World Server")

(Poll { (Get-Process DCS_World_Server_modular.tmp).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.Name -eq "Next" } } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval).Click("true")

Start-Sleep -Milliseconds 500;
[FlaUI.Core.Input.Keyboard]::Type([FlaUI.Core.WindowsAPI.VirtualKeyShort]::DOWN)
(Poll { (Get-Process DCS_World_Server_modular.tmp).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.Name -eq $terrain } } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval).Click("true")

(Poll { (Get-Process DCS_World_Server_modular.tmp).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.Name -eq "Next" } } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval).Click("true")
(Poll { (Get-Process DCS_World_Server_modular.tmp).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.Name -eq "Next" } } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval).Click("true")
(Poll { (Get-Process DCS_World_Server_modular.tmp).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.Name -eq "Next" } } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval).Click("true")

(Poll { (Get-Process DCS_World_Server_modular.tmp).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.Name -eq "Install" } } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval).Click("true")

$timeout = [System.TimeSpan]::FromMilliseconds(1000 * 60)

(Poll { (Get-Process DCS_World_Server_modular.tmp).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.Name -eq "Finish" } } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval).Click("true")

(Poll { (Get-Process DCS_Updater).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.Name -eq "Proceed" } } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval).Click("true")

$interval = [System.TimeSpan]::FromMilliseconds(5000)
# 120min timeout
$timeout = [System.TimeSpan]::FromMilliseconds(1000 * 60 * 120)
$delay = [System.TimeSpan]::FromMilliseconds(500)
$reportingInterval = [System.TimeSpan]::FromMilliseconds(1000 * 60 * 5)

$i = (Poll { (Get-Process DCS_Updater).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.Name} | Where-Object { $_.Name.StartsWith("Successfully installed") } } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval)
(Nested-Children($i.Parent) | Where-Object { $_.Name -eq "OK" }).Click("true")

Write-Host "Starting DCS Server"
New-Item -Path "C:\Users\Administrator\Saved Games\DCS.dcs_serverrelease" -Name "Missions" -ItemType "Directory" -Force
$webClient.DownloadFile($missionUrl, "C:\Users\Administrator\Saved Games\DCS.dcs_serverrelease\Missions\$missionName" )

$ServerSettings=@"
cfg =
{
	["description"] = "",
	["require_pure_textures"] = true,
	["listStartIndex"] = 1,
	["advanced"] =
	{
		["allow_change_tailno"] = true,
		["disable_events"] = false,
		["allow_ownship_export"] = true,
		["allow_object_export"] = $allow_object_export,
		["pause_on_load"] = true,
		["allow_sensor_export"] = $allow_sensor_export,
		["event_Takeoff"] = true,
		["pause_without_clients"] = false,
		["client_outbound_limit"] = 0,
		["client_inbound_limit"] = 0,
		["server_can_screenshot"] = false,
		["allow_players_pool"] = true,
		["voice_chat_server"] = true,
		["allow_change_skin"] = true,
		["event_Connect"] = true,
		["event_Ejecting"] = true,
		["event_Kill"] = true,
		["event_Crash"] = true,
		["event_Role"] = true,
		["resume_mode"] = 0,
		["maxPing"] = 0,
		["allow_trial_only_clients"] = $allow_trial_only_clients,
		["allow_dynamic_radio"] = true,
	}, -- end of ["advanced"]
	["require_pure_clients"] = true,
	["listLoop"] = false,
	["listShuffle"] = false,
	["isPublic"] = true,
	["missionList"] =
	{
		[1] = "C:\\Users\\Administrator\\Saved Games\\DCS.dcs_serverrelease\\Missions\\$missionName",
	}, -- end of ["missionList"]
	["password"] = "$password",
	["bind_address"] = "",
	["name"] = "$name",
	["require_pure_scripts"] = false,
	["mode"] = 0,
	["port"] = 10308,
	["require_pure_models"] = true,
	["maxPlayers"] = $maxPlayers,
} -- end of cfg
"@
echo $ServerSettings | Out-File -Encoding ascii -Filepath "C:\Users\Administrator\Saved Games\DCS.dcs_serverrelease\Config\serverSettings.lua"

cd "Z:\DCS World Server"
DCS_server.exe

(Poll { ((Get-Process DCS_Server).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.ControlType -eq "Edit"}) } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval)[0].Patterns.Value.Pattern.SetValue("readyroom1")

(Poll { ((Get-Process DCS_Server).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.ControlType -eq "Edit"}) } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval)[1].Patterns.Value.Pattern.SetValue("kaUEE9k0td4I2nev")

(Poll { (Get-Process DCS_Server).Id | ForEach-Object { [FlaUI.Core.Application]::Attach($_) } | ForEach-Object { $_.GetAllTopLevelWindows( $automation ) } | ForEach-Object { $_.SetForeground(); $_ } | ForEach-Object { Nested-Children($_) } | Where-Object { $_.Name -eq "Log In"} } { param($i) $i.Count -ne 0 } $interval $timeout $delay $reportingInterval).Click("true")

Write-Host "Completed"
([System.DateTime]::Now).subtract($installStart)
'@
echo $StartupScript | Out-File -Filepath "C:\startup.ps1"

$StartupBash=@'
PowerShell -Command "Set-ExecutionPolicy -Force Unrestricted >> C:\StartupLog.txt 2>&1"
PowerShell -Command "C:\startup.ps1 >> C:\StartupLog.txt 2>&1"
'@
echo $StartupBash | Out-File -Encoding ascii -Filepath "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup\startup.cmd"
echo $StartupBash | Out-File -Encoding ascii -Filepath "C:\startup.cmd"
Restart-Computer
</powershell>

```
