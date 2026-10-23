# Templated

Templated é um desafio easy do hack the box, da categoria web

{{<figure src="/assets/image (71).png" >}}

### Enumeration

A primeira vista, quando acessamos o IP fornecido, damos de cara com a home page do serviço, que não tem nenhuma funcionalidade além de informações das tecnologias em que o sistema foi construído.

{{<figure src="/assets/image (72).png" >}}

Tentando achar alguma direção para seguir, procuro pela página robots.txt para ver se consigo alguma informação importante, mas obtive uma página de erro 404.\
Mas nessa página de erro, é importante notar que o nome da página que buscamos no endpoint é refletido no front-end:

{{<figure src="/assets/image (73).png" >}}

Quando temos controle de um dado que é refletido em uma página HTML, já surgem algumas possíveis vulnerabilidades que podem ser testadas, como o XSS, e nesse caso, como sabemos que é uma aplicação Flask com Jinja2, o SSTI se torna outra possibilidade, já que podemos estar controlando uma parte da linguagem de template.

### Exploitation

Uma maneira simples de descobrirmos se o sistema é vulnerável a SSTI, é passando como input o parâmetro \{{7\*7\}} e observar se o retorno é 49

{{<figure src="/assets/image (74).png" >}}

Com esse retorno, sabemos que temos um SSTI para explorar, e podemos ter um RCE com isso.

Nesse [link](https://book.hacktricks.xyz/pentesting-web/ssti-server-side-template-injection/jinja2-ssti#rce-escaping) temos um ótimo guia sobre SSTI em aplicações Flask, é uma vulnerbilidade bem documentada.

Meu objetivo aqui agora é usar o MRO do python para acessar uma classe que me forneça comandos do sistema operacional, como o OS ou subprocess

Para isso, podemos acessar uma classe string, com o input \{{''.\_\_class\_\_\}} e a partir dela, acessar as parent classes dessa classe, com \{{''.\_\_class.\_\_.mro()\}}

{{<figure src="/assets/image (75).png" >}}

Agora, temos acesso a class 'object' do python, onde todas as outras classes herdam propriedades dela, com isso, conseguimos buscar pela classe que precisamos para executar comandos no sistema operacional:

{{<figure src="/assets/image (76).png" >}}

Temos uma lista gigante de classes, e fica muito difícil saber em que índice alguma classe que nos interessa está.

Uma maneira que encontrei de achar o índice foi por essa lista no VS Code, quebrar pelo caractere '>', e ver a linha em que a classe subprocess.Popen se encontra:<br>

{{<figure src="/assets/image (77).png" >}}

Se encontra na linha 415, mas como os índices de array começam em 0, então é o índice 414

{{<figure src="/assets/image (78).png" >}}

Com acesso a classe subprocess.Popen, temos agora como usar métodos do python para executar comandos do sistema operacional, como o \`ls\`

{{<figure src="/assets/image (79).png" >}}

Como a flag está nesse mesmo diretório da aplicação ( / ), podemos ler a flag com o \`cat\`

{{<figure src="/assets/image (80).png" >}}

Ou, caso não queira fazer todo esse processo de achar os índices das classes, existe uma payload com um algoritmo em jinja para achar as classes dinamicamente, pulando todas essas etapas que acabei de fazer 😜


```django
{% for x in ().__class__.__base__.__subclasses__() %}{% if "warning" in x.__name__ %}{{x()._module.__builtins__['__import__']('os').popen("ls").read()}}{%endif%}{% endfor %}
```


Essa payload e outras podem ser vistas no [HackTricks](breaking-grad.md).



