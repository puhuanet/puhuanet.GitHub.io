# 准备Windows主机以使用Ansible进行远程管理

```
#NoTrayIcon
#RequireAdmin
#Region ;**** Directives created by AutoIt3Wrapper_GUI ****
#AutoIt3Wrapper_Icon=Ansible.ico
#AutoIt3Wrapper_Outfile=WinRM4Ansible.exe
#AutoIt3Wrapper_UseUpx=y
#AutoIt3Wrapper_UseX64=n
#AutoIt3Wrapper_Change2CUI=y
#EndRegion ;**** Directives created by AutoIt3Wrapper_GUI ****
#include <Misc.au3>
_Singleton("WinRM4Ansible")

#include <File.au3>
#include <Date.au3>
#include <Array.au3>
#include <Process.au3>

; Windows system prep for remote management with Ansible
; https://docs.ansible.com/ansible/latest/user_guide/windows.html
; 不支持                  WIN_XP/WIN_XPe/WIN_2003
; 需安装PS3.0和.NET4.5.2  WIN_VISTA/WIN_2008/WIN_7/WIN_2008R2
; 支持                    WIN_8/WIN_2012/WIN_81/WIN_10/WIN_2012R2/WIN_2016

Global $PSVer, $DotNetVer

Main()


Func Main()
	If Init() Then
		If CheckPSVersion() And CheckDotNETVersion() Then
			ConfWinRM()
		EndIf
	EndIf
	While True
		Sleep(250)
	WEnd
EndFunc   ;==>Main

Func Init()
	$PSVer = _Get_PowerShellVersion()
	$DotNetVer = _Get_dotNETFrameworkVersion()
	_WriteLogFile("准备Windows主机以使用Ansible进行远程管理 jacky@puhua.net")
	_WriteLogFile("OS Version: " & @OSVersion)
	_WriteLogFile("OS Arch: " & @OSArch)
	_WriteLogFile("Powershell Version: " & $PSVer)
	_WriteLogFile(".NET Framework Version: " & $DotNetVer)

	Switch @OSType
		Case "WIN_XP", "WIN_XPe", "WIN_2003"
			_WriteLogFile(@OSType & " - Ansible does not support this operating system type.")
			Return False
	EndSwitch

	Return True
EndFunc   ;==>Init

Func CheckPSVersion()
	; 如果PS版本<3，则
	; 	如果OS版本=6.0(WIN_VISTA WIN_2008)，则
	; 	如果OS版本=6.1(WIN_7 WIN_2008R2)，则
	; 	安装相应OS版本和架构的程序
	If $PSVer < "3" Then
		Switch @OSVersion
			Case "WIN_VISTA", "WIN_2008"
				FileInstall("D:\WinRM4Ansible\Windows6.0-KB2506146-x64.msu", @TempDir & "\Windows6.0-KB2506146-x64.msu", $FC_OVERWRITE)
				FileInstall("D:\WinRM4Ansible\Windows6.0-KB2506146-x86.msu", @TempDir & "\Windows6.0-KB2506146-x86.msu", $FC_OVERWRITE)
				FileInstall("D:\WinRM4Ansible\Windows6.0-KB2842230-x64.msu", @TempDir & "\Windows6.0-KB2842230-x64.msu", $FC_OVERWRITE)
				FileInstall("D:\WinRM4Ansible\Windows6.0-KB2842230-x86.msu", @TempDir & "\Windows6.0-KB2842230-x86.msu", $FC_OVERWRITE)
				Local $WMF = @TempDir & "\Windows6.0-KB2506146-" & @OSArch & ".msu"
				Local $WMFFix =  @TempDir & "\Windows6.0-KB2842230-" & @OSArch & ".msu"
			Case "WIN_7", "WIN_2008R2"
				FileInstall("D:\WinRM4Ansible\Windows6.1-KB2506143-x64.msu", @TempDir & "\Windows6.1-KB2506143-x64.msu", $FC_OVERWRITE)
				FileInstall("D:\WinRM4Ansible\Windows6.1-KB2506143-x86.msu", @TempDir & "\Windows6.1-KB2506143-x86.msu", $FC_OVERWRITE)
				FileInstall("D:\WinRM4Ansible\Windows6.1-KB2842230-x64.msu", @TempDir & "\Windows6.1-KB2842230-x64.msu", $FC_OVERWRITE)
				FileInstall("D:\WinRM4Ansible\Windows6.1-KB2842230-x86.msu", @TempDir & "\Windows6.1-KB2842230-x86.msu", $FC_OVERWRITE)
				Local $WMF = @TempDir & "\Windows6.1-KB2506143-" & @OSArch & ".msu"
				Local $WMFFix =  @TempDir & "\Windows6.1-KB2842230-" & @OSArch & ".msu"
		EndSwitch
		
		_WriteLogFile("正在安装Windows Management Framework 3.0，请稍候 ……")
		Local $cmd = @SystemDir & '\wusa.exe' & ' "' & $WMF & '" /quiet /norestart'
		Local $cmd2 = @SystemDir & '\wusa.exe' & ' "' & $WMFFix & '" /quiet /norestart'
		RunWait($cmd)
		RunWait($cmd2)
		If Not @error Then
			_WriteLogFile("Windows Management Framework 3.0 安装成功")
			_WriteLogFile("请重新启动计算机后，再次运行本程序。")
			Return False
		Else
			_WriteLogFile("Windows Management Framework 3.0 安装失败，")
			_WriteLogFile("请访问以下链：https://www.microsoft.com/en-us/download/details.aspx?id=34595，下载安装程序，进行手动安装。")
			Return False
		EndIf
	Else
		Return True
	EndIf
EndFunc   ;==>CheckPSVersion

Func CheckDotNETVersion()
	; 如果.NET版本<4.5.2，则
	; 	安装.ENT4.5.2程序
	If $DotNetVer < "4.5.2" Then
		FileInstall("D:\WinRM4Ansible\NDP452-KB2901907-x86-x64-AllOS-ENU.exe", @TempDir & "\NDP452-KB2901907-x86-x64-AllOS-ENU.exe", $FC_OVERWRITE)
		_WriteLogFile("正在安装.NET Framework 4.5.2，请稍候 ……")
		Local $DNF = @TempDir & "\NDP452-KB2901907-x86-x64-AllOS-ENU.exe"
		Local $cmd = $DNF & " /q /norestart"
		RunWait($cmd)
		If Not @error Then
			_WriteLogFile(".NET Framework 4.5.2 安装成功")
			_WriteLogFile("请重新启动计算机后，再次运行本程序。")
			Return False
		Else
			_WriteLogFile(".NET Framework 4.5.2 安装失败，")
			_WriteLogFile("请访问以下链：https://www.microsoft.com/en-us/download/details.aspx?id=42642，下载安装程序，进行手动安装。")
			Return False
		EndIf
	Else
		Return True
	EndIf
EndFunc   ;==>CheckDotNETVersion

Func ConfWinRM()
	_WriteLogFile("开始配置WinRM，请稍候 ……")
	FileInstall("D:\WinRM4Ansible\ConfigureRemotingForAnsible.ps1", @TempDir & "\ConfigureRemotingForAnsible.ps1", $FC_OVERWRITE)
	Local $CRF = @TempDir & "\ConfigureRemotingForAnsible.ps1"
	Local $cmd = 'powershell.exe -ExecutionPolicy ByPass -File "' & $CRF & '"'
	_RunDos($cmd)
	If Not @error Then
		_WriteLogFile("WinRM 设置完成，")
		$cmd = "winrm enumerate winrm/config/Listener"
		Run(@ComSpec & " /c " & $cmd, "")
	Else
		_WriteLogFile("WinRM 设置失败，")
	EndIf
EndFunc   ;==>ConfWinRM

Func _WriteLogFile($sMsg)
	Local $logfile = @TempDir & "\" & @ScriptName & ".log"
	_FileWriteLog($logfile, $sMsg)
	ConsoleWrite($sMsg & @CRLF)
EndFunc   ;==>_WriteLogFile

Func _Get_PowerShellVersion()
	Local $Result
	Local $keyname = "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\PowerShell\3\PowerShellEngine"
	Local $valuename = "PowerShellVersion"
	$Result = RegRead($keyname, $valuename)
	If @error Then
		Local $keyname = "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\PowerShell\1\PowerShellEngine"
		Local $valuename = "PowerShellVersion"
		$Result = RegRead($keyname, $valuename)
	EndIf
	Return $Result
EndFunc   ;==>_Get_PowerShellVersion

Func _Get_dotNETFrameworkVersion()
	; 如何确定.NET Framework版本
	; https://docs.microsoft.com/en-us/dotnet/framework/migration-guide/how-to-determine-which-versions-are-installed
	Local $keyname = "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\NET Framework Setup\NDP\v4\Full"
	Local $valuename = "Release"
	Local $release = RegRead($keyname, $valuename)
	If $release >= 528040 Then Return "4.8"
	If $release >= 461808 Then Return "4.7.2"
	If $release >= 461308 Then Return "4.7.1"
	If $release >= 460798 Then Return "4.7"
	If $release >= 394802 Then Return "4.6.2"
	If $release >= 394254 Then Return "4.6.1"
	If $release >= 393295 Then Return "4.6"
	If $release >= 379893 Then Return "4.5.2"
	If $release >= 378675 Then Return "4.5.1"
	If $release >= 378389 Then Return "4.5"
EndFunc   ;==>_Get_dotNETFrameworkVersion

```