+++
title = 'Remote Template Injection'
date = 2024-01-14T07:07:07+01:00
draft = false
+++

# Remote Template Injection

A técica de Remote Template Injection permite a execução de macros em arquivos `.docx` sem que a macro seja carregada e enviada junto ao arquivo. Ao invés disso, o documento aberto baixa um template específico que contém a macro que queremos que seja executado.

A criação do documento malicioso acontece dessa forma:

* Criação de um arquivo `.dotm` com a macro para ser executada e hospede em um servidor;
* Crie um arquivo `.docx` qualquer;
* Renomeie o arquivo de `.docx` para `.zip`, e extraia o arquivo `.zip` ;
* Edite o parâmetro `Target` em `./world/_rels/settins.xml.rels` e coloque o endereço onde se encontra o template com a macro;
* Compacte todos os arquivos para um `.zip` novamente (todos os arquivos, não a pasta raiz) e renomeie para `.docx` novamente

## Exemplo Prático

### Criação do template

Crie um documento `word` comum:

{{< figure src="/assets/image (28).png" >}}

Vá em `Exibir > Macros > Exibir Macros`, ou aperte `Alt + F8` para abrir o editor de macros:

{{< figure src="/assets/image (29).png" >}}

Importante a configuração ser de Macros em apenas esse documento:

{{< figure src="/assets/image (30).png" >}}

De um nome qualquer a sua macro e clique em criar, onde seremos enviados para o editor da macro.\
Para testar se a macro irá funcionar, a primeira payload pode ser algo simples, como abrir uma calculadora ou notepad, com um código dessa forma:

```vba
Set objShell = CreateObject("Wscript.Shell")
objShell.Run "calc"
```

Podemos criar duas funções com que farão que nossa macro seja executada assim que o arquivo for aberto, sendo essas funções `Document_Open()` e `AutoOpen()`, nossa macro final ficaria parecida com isso:

{{< figure src="/assets/image (31).png" >}}

Podemos clicar no botão de play ou apertar `F5` para testar nossa macro, se a calculadora abrir, é por que está funcionando:&#x20;

{{< figure src="/assets/image (32).png" >}}

Agora podemos salvar o arquivo em uma pasta de sua preferência, com a extensão `.dotm`:

{{< figure src="/assets/image (33).png" >}}

Agora equipados com o template, vamos hospedar em um webserver.

### Servindo o template

Várias maneiras podem ser utilizadas para disponibilizar esse template para download, como Ngrok, um servidor dedicado, uma VPS e etc. No caso desse laboratório, estou utilizando o servidor HTTP do python na minha máquina Linux para servir o arquivo que está na mesma rede do Windows. Envie seu template para seu servidor e o deixe disponível:

{{< figure src="/assets/image (34).png" >}}

Com isso, o template está pronto para ser baixado, vamos criar o arquivo docx.

### Criação do arquivo docx

Para isso, crie um documento word escolhendo um template qualquer, nesse exemplo, escolhi o currículo, em cenários reais é importante editar o documento para ser convincente o suficiente, no laboratório isso não importa tanto.

{{< figure src="/assets/image (36).png" >}}

E somente isso, já podemos salvar o arquivo como arquivo `.docx` para continuar com as outras etapas. Mude o arquivo de .docx para .zip e descompacte:

{{< figure src="/assets/image (37).png" >}}

Entre na pasta `./word/_rels` e edite o arquivo `settings.xml.rels`, nele, procure pelo parâmetro Target e coloque o endereço do servidor que está hospedando seu template malicioso:

{{< figure src="/assets/image (38).png" >}}

Agora só salvar esse arquivo e zipar todos os arquivos dentro da raiz novamente:

{{< figure src="/assets/image (39).png" >}}

Renomeie o .zip para .docx e teremos um arquivo Word válido que carregará nossa macro.

### Executando a prova de conceito

Chegou a hora de testar nosso projeto, comece abrindo o documento, isso deverá baixar nosso template no servidor linux:

{{< figure src="/assets/image (40).png" >}}   

O documento foi aberto com um aviso de macro a ser executada, e se olharmos no nosso servidor linux, poderemos ver os logs do download do template:

{{< figure src="/assets/image (41).png" >}}

Confirmamos então que essa etapa funcionou, vamos permitir a macro e conferir se a calculadora será aberta:

{{< figure src="/assets/image (42).png" >}}

E nossa macro funcionou! A prova de conceito funcionou, então podemos evoluir.

### Reverse Shell
