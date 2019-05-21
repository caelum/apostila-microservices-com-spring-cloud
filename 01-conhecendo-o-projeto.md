# Conhecendo o Caelum Eats

## Exercício: Executando o back-end

1. Acesse o diretório que contém o código do curso em: `Desktop/cursos/33`.
2. Copie o diretório `eats-monolito` e cole em seu `Desktop`.
3. Abra o Eclipse, definindo como workspace `/home/<usuario-do-curso>/workspace-monolito`. Troque `<usuario-do-curso>` pelo login utilizado no curso.
4. No Eclipse, acesse _File > Import > Existing Maven Projects_ e clique em _Next_. Em _Root Directory_, aponte para o diretório copiado em um dos passos anteriores.
5. Acesse a classe `EatsApplication` e a execute com _CTRL+F11_. O banco de dados será criado automaticamente e alguns dados serão populados.
6. Teste a URL `http://localhost:8080/restaurantes/1` pelo navegador e verifique se um JSON com os dados de um restaurante foi retornado.
7. Analise o código. Veja:
	- as entidades de negócio
	- os recursos e suas respectivas URIs
	- os serviços e suas funcionalidades.

## Exercício: Executando o front-end

1. Copie o diretório `Desktop/cursos/33/eats-ui` para o seu `Desktop`.
2. Abra um Terminal e digite:
	
	```sh
	cd ~/Desktop/eats-ui
	```

3. Instale as dependências do front-end com o comando:

	```sh
	npm install
	```
4. Execute a aplicação com o comando:

	```sh
	ng serve
	```

5. Abra um navegador e teste a URL: `http://localhost:4200`. Explore o projeto, fazendo um pedido, confirmando um pedido efetuado, cadastrando um novo restaurante e aprovando-o. Peça ajuda ao instrutor.

