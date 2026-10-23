# MonitorsThree

{{< figure src="/assets/image (123).png" alt="">}}

## Recon

Começamos com o IP `10.10.11.30`. Mapeando a rede, encontramos 3 serviços rodando nas respectivas portas: 22, 80, 8084 (filtered):

```bash
 sudo nmap -sV -sS -Pn -O -oA nmap/monitorsthree -p22,80,8084 monitorsthree.htb
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-09-22 14:05 EDT
Nmap scan report for monitorsthree.htb (10.10.11.30)
Host is up (0.18s latency).

PORT     STATE    SERVICE VERSION
22/tcp   open     ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
80/tcp   open     http    nginx 1.18.0 (Ubuntu)
8084/tcp filtered websnp
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Aggressive OS guesses: Linux 5.0 (96%), Linux 5.0 - 5.5 (96%), Linux 4.15 - 5.8 (95%), Linux 3.1 (95%), Linux 3.2 (95%), AXIS 210A or 211 Network Camera (Linux 2.6.17) (95%), Linux 2.6.32 (94%), Linux 5.3 - 5.4 (94%), ASUS RT-N56U WA
P (Linux 3.4) (93%), Linux 3.16 (93%)
No exact OS matches for host (test conditions non-ideal).
Network Distance: 2 hops
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 20.83 seconds
```

As únicas portas acessíveis publicamente são a 22 e a 80, vou começar analisando a aplicação web na porta 80.

Entrando no IP `10.10.11.30`, somos redirecionados para o domínio `monitorsthree.htb` e, adicionando o mesmo ao arquivo `/etc/hosts`, conseguimos acessar o site:

{{< figure src="/assets/image (125).png" alt="">}}

O site não tem muitas interações, apenas a página de login e recuperação de senha. Em paralelo, rodo um fuzzing para descobrir possíveis subdomínios, encontro o `cacti` e também o adiciono ao arquivo `/etc/hosts` :

{{< figure src="/assets/image (126).png" alt="">}}

### cacti.monitorsthree.htb

{{< figure src="/assets/image (127).png" alt="">}}

Se trata do software Cacti ([https://www.cacti.net/](https://www.cacti.net/)) na versão 1.2.26, cujo tem uma CVE de RCE, porém, somente autenticado que conseguimos explorar e no momento não temos credenciais, e as padrões/fracas não são válidas aqui.

### monitorthree.htb

Voltando ao site principal, começo a interagir com os endpoints disponíveis junto de uma proxy para analisar seu comportamento.

Analisando o endpoint `forgot_password.php`, enviamos o input `username` e o nos informa que nos enviará um email:

{{< figure src="/assets/image (128).png" alt="">}}

Para esse endpoint funcionar uma possível implementação seria resgatar o Email desse `username` através de uma query SQL e enviar esse email para o serviço de mensageria. \
Se o sistema confia no nosso input para montar a query, podemos ter um `SQL Injection`.

Adicionando uma aspas simples no formulário já gera um erro nos dando detalhes da implementação no server-side e confirma nosso SQL injection em um banco de dados `MariaDB`:

{{< figure src="/assets/image (129).png" alt="">}}

## Exploração

O erro não nos da o resultado da query, então temos um blind SQL Injection, possuo um script para explorar esse tipo de SQL Injection ([https://github.com/Arthur-Juan/blind-sqli-script](https://github.com/Arthur-Juan/blind-sqli-script)), porém, ele é bastante lento comparado ao SQLMap que é preferível nessa situação. Salvamos a request do burp em um arquivo de texto para importar no SQLMap.

```bash
 sqlmap -r req.req --dbs
```

{{< figure src="/assets/image (130).png" alt="">}}

```bash
sqlmap -r req.req --dbs -D monitorsthree_db --tables
```

{{< figure src="/assets/image (132).png" alt="">}}

```bash
 sqlmap -r req.req --dbs -D monitorsthree_db -T users --dump
```

{{< figure src="/assets/image (134).png" alt="">}}

Com o dump feito da tabela de users do banco de dados, podemos tentar quebrar essas hashes:

{{< figure src="/assets/image (135).png" alt="">}}

Agora temos uma credencial para usar: `admin:greencacti2001` (Marcus Higgins).\
Tento usar essa credencial para logar via SSH, mas sem sucesso. O servidor permite apenas login via public key.<br>

{{< figure src="/assets/image (136).png" alt="">}}

### Login em cacti.monitorsthree.htb

Com as credenciais obtidas na etapa anterior podemos logar na plataforma `Cacti`:

{{< figure src="/assets/image (137).png" alt="">}}

Agora com credenciais válidas, posso abusar da CVE disponível para essa versão do Cacti utilizando alguma POC como essa: [https://github.com/5ma1l/CVE-2024-25641](https://github.com/5ma1l/CVE-2024-25641)

No repositório, edito o arquivo `php/monkey.php` para colocar o IP e a Porta para receber a shell

{{< figure src="/assets/image (138).png" alt="">}}

Executamos o exploit e temos Shell :)

{{< figure src="/assets/image (140).png" alt="">}}

Antes de continuar, faço um upgrade na shell para melhorar o uso:

{{< figure src="/assets/image (142).png" alt="">}}

## Escalação de privilégio

### marcus

O usuário da shell é o `www-data`, rapidamente listo a pasta `/home` para ver se encontro algum usuário nessa máquina, encontro Marcus, tento listar a home de Marcus, mas não tenho permissão:

{{< figure src="/assets/image (143).png" alt="">}}

Em `/etc/passwd` só nos mostra o usuário Marcus e root, não temos outros na máquina:

{{< figure src="/assets/image (144).png" alt="">}}

Temos duas aplicações rodando nesse servidor, o Cacti, e o site principal, podemos olhar os arquivos de configuração em busca de informações importantes.\
No site principal, tem um arquivo de conexão com o banco de dados, mas é o mesmo banco que foi dumpado com o SQL Injection, então nada de novo, vou olhar o cacti.\
No código fonte do cacti, temos o arquivo `/includes/config.php`, onde temos informações importantes, como credenciais de banco de dados:

{{< figure src="/assets/image (145).png" alt="">}}

Podemos então logar no banco de dados com essas credenciais adquiridas.

Dentro do banco de dados, temos o banco Cacti com várias tabelas, porém, a mais interessante é a `user_auth`, vamos ver o que tem:

{{< figure src="/assets/image (146).png" alt="">}}

Temos usuários e senhas, fora o guest, ambos os usuários pertencem a marcus, vamos tentar quebrar essa hash. \
Salvo a hash em um arquivo e tento quebrar com `John`, consegui apenas a da linha de username marcus:<br>

{{< figure src="/assets/image (147).png" alt="">}}

Agora é possível logar como marcus no servidor.

{{< figure src="/assets/image (148).png" alt="">}}

Agora posso tentar roubar a private\_key do servidor e ter acesso SSH ao sistema.\
Eu simplesmente copio e colo o arquivo de `~/.ssh/id_rsa` para um arquivo `id_rsa` em meu host:<br>

{{< figure src="/assets/image (149).png" alt="">}}

E agora, é possível logar facilmente como marcos via SSH:<br>

{{< figure src="/assets/image (150).png" alt="">}}

### Root flag

Se lembrar do mapeamento de rede inicial, temos uma porta 8084 Filtered que, talvez, dentro do servidor ela esteja acessível, vamos ver as postas abertas no servidor:

{{< figure src="/assets/image (152).png" alt="">}}

Tentando me conectar com a porta 8084 não recebo nenhuma resposta, porém, tentando na porta 8200 recebo uma resposta de um `webserver` me redirecionando para a rota `/login.html` e, fazendo uma chamada para essa rota, vejo que se trata de um sistema chamado `Duplicati`:

{{< figure src="/assets/image (153).png" alt="">}}

Fazendo algumas pesquisas, descubro que se trata de um sistema de backup. Esse sistema só está disponível internamente no server, para poder interagir com ele pelo browser pode ser feito um `port fowarding` usando o próprio SSH:

{{< figure src="/assets/image (154).png" alt="">}}

Agora, o conteúdo da porta 8200 do servidor fica disponível na minha máquina:

{{< figure src="/assets/image (155).png" alt="">}}

As credencias obtidas até aqui não funcionam nesse formulário, porém, esse sistema possui uma falha documentada, onde é possível bypassar a autenticação, seguindo os passos descritos aqui: [https://medium.com/@STarXT/duplicati-bypassing-login-authentication-with-server-passphrase-024d6991e9ee](https://medium.com/@STarXT/duplicati-bypassing-login-authentication-with-server-passphrase-024d6991e9ee)\
Os passos estão bem descritos nesse link, então não vou detalhar muito aqui.\
Subindo um servidor simples no server com `python3 -m http.server em /opt/duplicati/config` (pasta com o banco de dados SQLite do duplicati), consigo baixar o banco para minha máquina para ir atrás de informações específicas para o ataque, especialmente o `server-passphrase`.

{{< figure src="/assets/image (156).png" alt="">}}

Com os passos do artigo, consigo gerar uma senha válida:

{{< figure src="/assets/image (157).png" alt="">}}

E logar com sucesso na aplicação:

{{< figure src="/assets/image (158).png" alt="">}}

Para essa aplicação, não temos nenhum outro exploit conhecido que nos ajude a escalar privilégio ou algo assim, mas isso não indica que seja seguro, portanto, posso fazer uma auditoria nessa aplicação para procurar por outros pontos fracos.

Vou começar interagindo com o sistema e enumerando suas funcionalidades, e consigo verificar algumas como trocar configurações da conta, de backup e etc, mas a principal funcionalidade do sistema é criar um backup e restaurar um backup.

Vamos debugar essa funcionalidade para poder entender melhor:

* É possível criar um backup, passando algumas especificações como diretório de origem, diretório de destino, senha, tipo de criptografia, entre outros inputs.
* É possível restituir o backup, passando o diretório do backup, o diretório onde o backup será restituído, a senha configurada, entre outros inputs.

Podemos notar que temos o controle de um diretório ou arquivo de origem qualquer para um local de destino qualquer, então se o sistema confia nesses inputs, podemos mover arquivos de locais ou para locais que não temos permissão, e podemos abusar disso.

#### Criando o backup

Primeiro, vamos configurar um backup para direcionar um arquivo para uma pasta destino, sem encriptação:<br>

{{< figure src="/assets/image (159).png" alt="">}}

Escolhemos a pasta de destino:

{{< figure src="/assets/image (160).png" alt="">}}

O arquivo de origem (root.txt :hushed:)&#x20;

{{< figure src="/assets/image (161).png" alt="">}}

As outras opção apenas avançamos, e temos nosso backup configurado.

{{< figure src="/assets/image (163).png" alt="">}}

E então, podes executar o backup clicando em 'Run Now'

#### Restaurando o backup

Agora, vamos restaurar o arquivo para sua versão original, escolhendo uma pasta em que temos acesso. Aqui pode ser escolhido o arquivo que será restaurado:<br>

{{< figure src="/assets/image (164).png" alt="">}}

E agora a pasta onde ele será restaurado:

{{< figure src="/assets/image (165).png" alt="">}}

E clique em restaurar o arquivo, com isso, na pasta selecionada terá o conteúdo de root.txt de forma que possamos ler.

{{< figure src="/assets/image (166).png" alt="">}}

###

Obviamente, com a possibilidade de controlar o envio de arquivos de um lugar para o outro no file system do servidor, pode ser feito muito mais além de ler arquivos sensíveis, e uma dessas possibilidades é criar uma chave SSH e enviar a public key para root/.ssh, e com isso, poderíamos entrar no servidor como root.
