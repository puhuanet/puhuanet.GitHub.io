# Get-HBAWin

## 功能

获取光纤卡的WWN号  
Find World Wide Name (WWN) for a fibre channel HBA  

## 下载

[Get-HBAWin.exe](https://github.com/puhuanet/Get-HBAWin/raw/master/Get-HBAWin.exe)

## 界面

![Get-HBAWin](https://github.com/puhuanet/Get-HBAWin/raw/master/screen.png)


## PowerShell

```
 Get-HBAWin -ComputerName <ip or computername>  | Format-Table -AutoSize
```

```
function Get-HBAWin {  
param(  
[String[]]$ComputerName = $ENV:ComputerName, 
[Switch]$LogOffline  
)  
  
$ComputerName | ForEach-Object {  
try { 
    $Computer = $_ 
     
    $Params = @{ 
        Namespace    = 'root\WMI' 
        class        = 'MSFC_FCAdapterHBAAttributes' 
        ComputerName = $Computer  
        ErrorAction  = 'Stop' 
        } 
     
    Get-WmiObject @Params  | ForEach-Object {  
            $hash=@{  
                ComputerName     = $_.__SERVER  
                NodeWWN          = (($_.NodeWWN) | ForEach-Object {"{0:X2}" -f $_}) -join ":"  
                Active           = $_.Active  
                DriverName       = $_.DriverName  
                DriverVersion    = $_.DriverVersion  
                FirmwareVersion  = $_.FirmwareVersion  
                Model            = $_.Model  
                ModelDescription = $_.ModelDescription  
                }  
            New-Object psobject -Property $hash  
        }#Foreach-Object(Adapter)  
}#try 
catch { 
    Write-Warning -Message $_ 
    if ($LogOffline) 
    { 
        "$Computer is offline or not supported" >> "$home\desktop\Offline.txt" 
    } 
} 
 
}#Foreach-Object(Computer)  
  
}#Get-HBAWin
```
