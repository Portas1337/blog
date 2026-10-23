# Blurry

{{< figure src="/assets/image (167).png" alt="">}}

## Recon



Começamos a máquina apenas com um IP, então podemos mapear o que temos nesse IP usando o nmap:

{{< figure src="/assets/image (168).png" alt="">}}

Com apenas duas portas abertas, podemos acessar a porta 80 e ver o que temos se serviço web. Fazendo isso, o navegador redireciona para \`app.blurry.htb\`, então vamos adicionar esse domínio ao arquivo /etc/hosts para termos acesso, e assim conseguimos visualizar a página:

{{< figure src="/assets/image (169).png" alt="">}}<figcaption><p>Web Page</p></figcaption></figure>

Temos uma aplicação chamada ClearML, enquanto estudamos o que é essa aplicação, podemos deixar um fuzzing procurando por outros subdomínios.\
Voltando para o clearML e fazendo uma pesquisa, descobrimos que é uma plataforma para criação de inteligência artificial, onde é possível construir, treinar e dar deploy de uma IA.

{% embed url="https://clear.ml/?gad_source=1&gclid=EAIaIQobChMIh8D8j5-FiQMVjSitBh1TVCfPEAAYASAAEgJXd_D_BwE&hsa_acc=4043203093&hsa_ad=636186176607&hsa_cam=16563267736&hsa_grp=134088880946&hsa_kw=clearml&hsa_mt=e&hsa_net=adwords&hsa_src=g&hsa_tgt=kwd-784171082579&hsa_ver=3&utm_campaign=Search+|+Brand&utm_device=c&utm_feeditemid=&utm_medium=ppc&utm_source=adwords&utm_term=clearml" %}

## Exploit



Fazendo uma outra pesquisa, podemos descobrir que o ClearML possui uma vulnerabilidade de RCE ([https://github.com/xffsec/CVE-2024-24590-ClearML-RCE-Exploit](https://github.com/xffsec/CVE-2024-24590-ClearML-RCE-Exploit)), então podemos tentar abusar dessa CVE para conseguir uma shell no servidor.

O Exploit foi escrito em python, então eu gosto de criar um ambiente virtual para isolar as dependências usando \`python -m venv venv; source venv/bin/activate\`.\
Instalando as dependências e executando o script,  temos algumas opções, sendo a primeira de inicializar o ClearML para conectar nosso CLI ao web app em que achamos no alvo:

{{< figure src="/assets/image (170).png" alt="">}}

Podemos criar um nome de usuário qualquer na tela inicial, e navegar até a rota indicada pelo script (/settings/workspace-configuration) onde seremos capazes de gerar uma credencial para estabelecer uma comunicação entre o web app e o CLI:

{{< figure src="/assets/image (172).png" alt="">}}

Agora, precisamos apenas copiar e colar esse JSON no terminal, e podemos ir para a segunda opção, porém teremos um erro indicando que o script não conseguiu se conectar com api.blurry.htb:<br>

{{< figure src="/assets/image (173).png" alt="">}}

Então vamos colocar esse domínio e os outros que foram encontrados no fuzzing em /etc/hosts:<br>

{{< figure src="/assets/image (174).png" alt="">}}

Agora, vamos para a opção 2 do Script que é rodar o exploit, para isso vamos deixar o netcat ouvindo na porta 1337, e preencher os dados que o script solicita, sendo um deles o nome de um projeto, então vamos procurar ou criar um projeto no web app, e achamos um chamado Black Swan:<br>

{{< figure src="/assets/image (176).png" alt="">}}

Preenchemos então com esse nome e executamos o exploit, note que ele pode demorar a acontecer ou até mesmo não funcionar precisando ser executado mais de uma vez, porém conseguimos uma reverse shell:

{{< figure src="/assets/image (177).png" alt="">}}

## Privilege Escalation



Para termos uma shell melhor, podemos ou fazer o upgrade dessa shell, ou então pegar as chaves SSH e conectar via SSH na máquina,  vou com a opção do SSH:

{{< figure src="/assets/image (178).png" alt="">}}

Agora temos uma shell completa, e maneiras de retornar ao servidor sem precisar executar o exploit novamente. \
Algo interessando que podemos analisar quando estamos logados como algum usuário em uma máquina linux é saber quais comandos esse usuários pode executar com permissão de root usando o \`sudo -l\`:<br>

{{< figure src="/assets/image (179).png" alt="">}}

Podemos notar que nosso usuário consegue executar um programa chamado `evaluate_model` com permissão de root, então vamos analisar e estudar o que esse programa faz para procurarmos por algum ponto fraco que possa ser explorado. \
O arquivo se trata de um shell script, então baixo ele para minha máquina para poder estudar o que ele faz:

{{< figure src="/assets/image (43).png" alt="">}}

Observando o script, podemos ver que ele chama outro programa depois, o `evaluate_model.py`, que também vou baixar para a minha máquina para poder entender também, da mesma forma que baixei o shell script.\
Olhando o script, o funcionamento é o seguinte:

* Descompacta o arquivo enviado como parâmetro baseado no tipo do arquivo (ZIP ou TAR)
* Verifica se é um arquivo extraído Picke malicioso, utilizando a ferramenta fickling
  * Caso seja, deleta o arquivo
  * Caso seja considerado seguro, envia para o programa evaluate\_model.py

{{< figure src="/assets/image (44).png" alt="">}}

Então, vamos entender o evaluate\_model.py

O script usa a biblioteca `Torch` para processar arquivos de modelo de inteligência artificial. Eu não entendo muito bem como funciona esse tipo de programação, então é preciso ir lendo e entendendo o programa aos poucos, e tento manter em mente em quais partes do programa ele confia no meu input para fazer algum processamento, pois nesses pontos que podem existir as vulnerabilidades.\
Primeiro ele confere se recebeu algum arquivo, envia para uma função chamada `load_model`, que usa a função da biblioteca `torch.load(model_path)` , e sendo o `model_path` um valor que podemos controlar, então é uma boa função para pesquisar o que faz e como foi implementada.\
Lendo a documentação, vemos que essa função utiliza a biblioteca `pickle` do python para deserializar o arquivo com um `unpickle` , que é bem conhecido por ser inseguro caso use dados não confiáveis, e um dado em que podemos controlar é um dado não confiável para o programa.<br>

{{< figure src="/assets/image (45).png" alt="">}}

{{< figure src="/assets/image (46).png" alt="">}}

Com essas informações, já temos um possível plano de ataque, onde enviamos um arquivo arquivo malicioso para o `toch.load` para conseguirmos um RCE, porém, temos que bypassar o shell script que provavelmente vai bloquear nossa payload. Então, com o arquivo na nossa máquina, podemos estudar e pesquisar formas de bypassar esse script.\
Para testar localmente, primeiro precisamos instalar as bibliotecas usadas pelo programa, pode ser feito apenas lendo o código fonte e ir instalando, ou executando o programa, lendo o log de erro e instalar a biblioteca que ele indicou que falta. Após isso, temos a simulação do ambiente do servidor em nossa máquina, o que torna perfeito para fazermos testes para conseguir explorar uma vulnerabilidade crítica de forma precisa. \
Primeiro, eu crio uma payload em python, porém usando a biblioteca `torch`, pois ele gera o tipo de arquivo que a função `Torch.load()` consegue entender para processar, para poder explorar a deserialização insegura que apenas vai executar o comando `whoami`, como uma prova de conceito de que a falha foi explorada. A payload será salva em um arquivo `model.pth`:

{{< figure src="/assets/image (183).png" alt="">}}

\
Executando o shell `evaluate_model.sh` (modificado para identificar o `evalute_model.py` da minha máquina que está na mesma pasta, ao invés de procurar em /models) para analisar se ele passa ou não pelo `fickling`, e a payload passou:

{{< figure src="/assets/image (185).png" alt="">}}

Agora, eu executo o `evaluate_model.py` passando o `model.pth`, para simular o final do comportamento do script, e ele executa o comando `whoami`:

{{< figure src="/assets/image (182).png" alt="">}}

Sabendo que a payload funciona, posso trocar o `whoami` para um comando `bash`, para executar uma shell com as permissões do programa (no servidor, root):

{{< figure src="/assets/image (181).png" alt="">}}

Executando esse arquivo na minha máquina e enviado o `model.pth` enviando e executando no servidor, ele executa o `bash` com permissão de root:

{{< figure src="/assets/image (180).png" alt="">}}

E com isso, finalizamos a máquina.
