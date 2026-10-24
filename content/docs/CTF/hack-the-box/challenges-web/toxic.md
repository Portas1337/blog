+++
title = 'Toxic'
date = 2024-01-14T07:07:07+01:00
draft = true
+++

# Toxic

Toxic é um desafio easy do hack the box da categoria web.

Esse é um desafio onde temos o código fonte disponível e um docker para subir a aplicação, então, começo os testes em um ambiente local.

## Enumeration

Analisando a aplicação, temos apenas uma página e nenhuma funcionalidade:<br>

{{< figure src="/assets/image (86).png" >}}

Com fuzzing não acho nenhum outro endpoint, vou olhar o código fonte da aplicação.

Temos arquivos docker, configuração de nginx e essa página HTML que vemos na web. Os principais arquivos são index.php e PageModel.php

Vamos olhar o arquivo index.php<br>

{{< figure src="/assets/image (87).png" >}}

Agora, entender o PageModel.php<br>

{{< figure src="/assets/image (88).png" >}}

Os outros arquivos são os arquivos HTML e de configuração<br>

{{< figure src="/assets/image (89).png" >}}

Temos um código fonte pequeno, assim, fica fácil entender o quadro geral da aplicação e os mecanismos dela. Nesse caso, o cookie controla a página que será exibida, já que, o index.php recupera a classe PageModel vindo do cookie e deserializa ela e, no final, PageModel faz um include nesse arquivo especificado.\
&#x20;

{{< figure src="/assets/image (90).png" alt="">}}
<p>Cookie que nosso browser envia para o backend</p>

{{< figure src="/assets/image (92).png" alt="">}}<figcaption><p>Cookie decodificado em base64</p></figcaption></figure>

## Exploit

Como vimos no comportamento na aplicação, o PHP faz um include no parâmetro `$file`, que pertence ao objeto PageModel que enviamos serializado via cookie, com isso, podemos controlar o include feito pelo PHP.

Para me auxiliar nesse processo programei um pequeno script em PHP para simular o comportamento do backend do nosso alvo e ser mais certeiro nos exploits, sendo esse código aqui:

<pre class="language-php"><code class="lang-php"><strong>&#x3C;?php
</strong>class PageModel
{
    public $file;

    public function __destruct() 
    {
        include($this->file); 
    }
}
$model = new PageModel();
$model->file = /* insira payload aqui */
$serialize = serialize($model);

$base64 = base64_encode($serialize);
echo "$base64\n";
</code></pre>

Nesse código, podemos inserir nossa payload em $model->file e ela será serializada e encodada para base64, podendo assim, colocarmos no cookie para a aplicação processar nossa payload.

### PHP Filter Chain

A primeira ideia que eu tive com o cenário atual, foi uma técnica de PHP Filter chain, onde a função `include()` do PHP aceita alguns protocolos diferentes de arquivos, onde, em uma pesquisa, conseguiram usar helper de filtros do PHP para levar a um RCE.

Aqui você pode acessar o link da pesquisa: [https://www.synacktiv.com/publications/php-filters-chain-what-is-it-and-how-to-use-it.html](https://www.synacktiv.com/publications/php-filters-chain-what-is-it-and-how-to-use-it.html), e aqui um repositório com um gerador de payload para essa técnica: [https://github.com/synacktiv/php\_filter\_chain\_generator](https://github.com/synacktiv/php_filter_chain_generator)

Com essa ferramenta, gerei uma payload que fizesse um `system('ls -la')`; e pude rodar no meu script, no qual pude ver que funciona:<br>

{{< figure src="/assets/image (93).png" >}}

Podemos testar agora a aplicação com esse base64:<br>

{{< figure src="/assets/image (94).png" >}}

Na imagem podemos ver o cookie contento o base64 criado pelo script, mas a payload não foi executada pelo backend e no terminal onde está o docker com a aplicação existem vários logs de erro. Essa ideia parece não dar certo.\
{{<figure src="/assets/image (95).png" alt="" data-size="original">}}

### RFI

Lendo a documentação da função `include()`, pude tirar algumas conclusões:<br>

* A função executa scripts PHP, logo se passarmos um arquivo PHP para ele, ele vai executar os comandos do PHP
* A função aceita URLs como parâmetro (e nós controlamos esse parâmetro)

{{< figure src="/assets/image (110).png" alt="" width="563">}}<figcaption><p>Documentação do PHP</p></figcaption></figure>

{{< figure src="/assets/image (109).png" alt="">}}<figcaption><p>Documentação do PHP</p></figcaption></figure>

Com isso, a ideia é fazer ele incluir um arquivo PHP remoto e executar os comandos feitos por ele.\
Criei um simples arquivo PHP para fazer o backend da aplicação alvo consumir:<br>


```php
<?php
system("ls -la");
?>
```


Subi o arquivo usando um servidor python e gerei o objeto serializado com o script auxiliar que temos usado:<br>

{{< figure src="/assets/image (96).png" alt="">}}<figcaption><p>Tentativa de execução</p></figcaption></figure>

No script de teste, a payload deu erro, pois a função include() não está aceitando URLs como parâmetro, é uma configuração do PHP que pode ser mudada, então podemos ver se o servidor alvo está aceitando URLs:\
![](</assets/image (97).png>)![](</assets/image (98).png>)

Então não, o backend está recusando URLs também e essa é outra ideia que não deu certo.

### Wrappers

Como dito na descrição da ideia de ataque anterior, o PHP aceita alguns parâmetros diferentes nas suas funções de leitura de dados, conhecidos como PHP Wrappers.

Com alguns deles, podemos encodar e zipar arquivos e etc. Nem todos são muito úteis para o nosso cenário atual, pois como não sabemos o nome e nem onde está a flag, não conseguimos a encontrar para ler (mesmo sem os wrappers, isso seria possível com o LFI normal que temos nesse backend). Porém, alguns deles podem ser úteis para conseguir algum RCE.

Temos uma lista desse no hacktricks ([https://book.hacktricks.xyz/pentesting-web/file-inclusion#input](https://book.hacktricks.xyz/pentesting-web/file-inclusion#input)), tentei usar o input para tentar um RCE, mas já aviso aqui que foi outra ideia que não deu certo.  :cry:  &#x20;



### Log Poisoning

Voltando para o código fonte do arquivo, olho na configuração do nginx o caminho do arquivo de logs:<br>

{{< figure src="/assets/image (99).png" >}}

Isso brilha os olhos de qualquer hacker que está nesse contexto atual! A ideia por trás da minha tentativa de RFI foi fazer o include executar um código PHP, porém, de uma URL externa que ele não esta aceitando. Porém, com os logs, nós podemos fazer ele puxar um arquivo interno e não de uma URL externa. \
O workflow desse ataque se consiste em envenenar o arquivo de log com algum código PHP e fazer o include chamar esse arquivo e executar a parte em PHP. Isso é log poisoning!

Entrando no container que está executando a aplicação, podemos ler os logs e vê que ele guarda o user-agent da requisição:<br>

{{< figure src="/assets/image (100).png" >}}

Com isso, podemos escrever um trecho PHP nesse user-agent e usar do LFI para ler esse arquivo e executar.\
Usei o próprio DevTools do firefox para editar uma request HTTP, mas isso, também, poderia ser feito com o CURL ou alguma proxy como o Burp Suite:

{{< figure src="/assets/image (101).png" >}}

Mudamos a payload para ser o arquivo de logs:<br>

{% code title="index.php" overflow="wrap" lineNumbers="true" %}
```php
<?php
class PageModel
{
    public $file;

    public function __destruct() 
    {
        include($this->file); 
    }
}
$model = new PageModel();
$model->file = "/var/log/nginx/access.log"
$serialize = serialize($model);

$base64 = base64_encode($serialize);
echo "$base64\n";
```
{% endcode %}

Executando o script, temos o base64 para inserir no cookie.\
Trocando o cookie e recarregando a página:<br>

{{< figure src="/assets/image (103).png" >}}

Temos ai o nosso RCE!!\
Podemos fazer uma simples webshell para interagir diretamente com o servidor:<br>

{{< figure src="/assets/image (104).png" >}}

{{< figure src="/assets/image (105).png" >}}

{{< figure src="/assets/image (106).png" >}}

Conseguimos o nome da flag!!\
Agora podemos apenas ler o conteúdo:<br>

{{< figure src="/assets/image (107).png" >}}

O exploit deu certo! Agora podemos reproduzir os mesmos passos no desafio de produção (não o que está local no nosso docker)

{{< figure src="/assets/image (108).png" >}}

Apesar de ser da categoria easy, eu adorei o desafio, pois no processo tive muitas ideias diferentes de como tentar resolver e algumas pesquisas que foram me dando outras ideias de ataques, eu amo esse processo!

