# RemoteInstall

This is a powershell script for older software that only seems to successfully install with bat files.

```
########### Functions ##################
Function Push-file {
  [CmdletBinding()]
  param (
    [Parameter(Mandatory=$true, Position=0)]
    [String] $File
  )
  $vFile = (Get-Item $File).Name
  Foreach ($Client in $Clients) {
    copy-item -path $File -Destination "\\$Client\c`$\Temp" -Force
    If (Test-Path -path \\$Client\c`$\Temp\$vFile) {
      Write-Host "$vFile has been copied to $Client"
    } else {
      Write-Host "$vFile didn't copy to $Client !!!"
    }
  }
}

Function Install-Program {
  [CmdletBinding()]
  param (
    [Parameter(Mandatory=$true, Position=0)]
    [String] $JobName,
    [Parameter(Mandatory=$true, Position=1)]
    [int] $InMinutes
  )

  #Creating bat file to run msi installer (Only way I've gotten it to work is through scheduled task)
  New-Item -path "\\$Clients\c`$\Temp\$JobName.bat" -Force
  $path ='msiexec /qn /i C:\Temp\' + $JobName + '.msi EXTRAARGUMENTS=TRUE GOHERE=TRUE'
  set-Content "\\$Clients\c`$\Temp\$JobName.bat" -Value $path

  #Creating Scheduled Task
  New-CimSession -ComputerName $Clients -Name $Clients
  $Action = New-ScheduledTaskAction -Execute "C:\Temp\$JobName.bat"
  $Time = (Get-Date).AddMinutes($InMinutes)
  $Trigger = New-ScheduledTaskTrigger -Once -At $Time
  Register-ScheduledTask "$JobName" -Action $Action -Trigger $Trigger -CimSession $Clients -user $Creds.UserName -Password $Creds.GetnetworkCredential().Password
  Remove-CimSession -Name $Clients
}

########### End of Functions ##################

#Setting Credentials and targeted PCs.
$Creds = Get-Credential
#$Clients = Read-Host -Prompt 'What is the PC name you are upgrading?'
$MultiClients = Get-Content -path "C:\Temp\Update.txt"

Foreach ( $Clients in $MultiClients ) {
  #Making sure remote PC has a Temp folder and that it's empty
  $Directory = Test-Path "\\$Clients\c`$\Temp" -PathType Container
  If ($Directory){
    Remove-item "\\$Clients\c`$\Temp\*" -Recurse
  }else{
    New-Item -path "\\$Clients\c`$" -Name "Temp" -ItemType Directory
  }

  #Clearing out any old Scheduled tasks
  Invoke-Command -ComputerName $Clients {
    get-scheduledtask -TaskName *Program* | Unregister-ScheduledTask -Confirm:$false
  }

  #Saving the alias incase it doesn't match PC Name
  $inputfile = "\\$Clients\c`$\Temp\tempalias.txt"
  $outputfile = "\\$Clients\c`$\Temp\alias.txt"
  Get-content -path "\\$Clients\c`$\Windows\metrics.ini" | Select-String -Pattern '(^PCAlias=.*)' | Out-File -FilePath $inputfile  -enc ascii
  (Get-Content $inputFile) | Where-Object {$_.trim() -ne "" } | Set-Content $outputFile

  #Removing Old Program versions (comment line below out for only patches)
  Invoke-Command -ComputerName $Clients { wmic product where "description='Name From Control panel program and features'" uninstall } -ErrorAction SilentlyContinue

  Push-File -File "C:\Temp\program.msi"
  Push-File -File "C:\Temp\programPatch.msi"
  install-Program -JobName "Program" -InMinutes "1"
  install-Program -JobName "ProgramPatch" -InMinutes "5"
}

#
# $final = "\\$Clients\c`$\Windows\metrics.ini"
# $alias = Get-Content -path "\\$Clients\c`$\Temp\alias.txt"
# (Get-content -path $final) -replace '(^PCAlias=.*)',"$alias" | out-file $final -enc ascii
#
```
