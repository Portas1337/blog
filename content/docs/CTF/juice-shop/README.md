# Juice Shop

O OWASP Juice Shop é uma aplicação web desenvolvido pela Open Web Application Security Project (OWASP), onde contém uma série de vulnerabilidades críticas que ocorrem em aplicações web, como uma forma de fornecer um ambiente moderno para praticar e treinar habilidades de hacking.

A aplicação é desenvolvida em Node.js (Express) e Angular, e contém um dashboard com cards para cada vulnerabilidade existente no sistema, em forma de desafios.

Os desafios são separados por módulos, agrupados por categorias de vulnerabilidades, e para escrever meu processo de resolução dos desafios, escrevei uma página por módulo para facilitar a busca e ter uma melhor separação.

Nessa seção, mostrarei como foi para achar o dashboard e como ele se é.

Iniciei minha aplicação usando docker-container, que pode ser feito seguindo a seguinte documentação: [https://hub.docker.com/r/bkimminich/juice-shop](https://hub.docker.com/r/bkimminich/juice-shop) na parte de Docker container.

Quando entramos na aplicação, vamos para a homepage:

<figure><img src="../../.gitbook/assets/image (66).png" >}}

O primeiro desafio apresentado pelo Juice Shop (assim que entramos a primeira vez) é encontrar o scoreboard das vulnerabilidades, podemos fazer isso de duas formas:

* Fuzzing
* Procurar dentro do código fonte (HTML, Javascript e etc...)

Com a ajuda de uma proxy, consigo fazer uma busca por todos os dados trafegados entre o client e o server, inclusive javascript e HTML:

<figure><img src="../../.gitbook/assets/image (67).png" >}}

Entrando nesse arquivo, conseguimos achar a URL /#/score-board:

<figure><img src="../../.gitbook/assets/image (68).png" >}}

Nesse scoreboard, temos um card para cada vulnerabilidade e para desafios de exploração de cada um, que pode é dividido em módulos, além disso, um aviso que que 16 desafios não estão disponíveis na versão do docker (então não farei esses desafios no momento)

<figure><img src="../../.gitbook/assets/image (69).png" >}}

A resolução de cada módulo estarão nas páginas a seguir.

