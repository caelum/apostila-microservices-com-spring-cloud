# Conhecendo o Caelum Eats

## Exercício: Executando o back-end

1. Clone o projeto do back-end para seu Desktop com os seguintes comandos:

  ```sh
  cd ~/Desktop
  git clone https://gitlab.com/aovs/projetos-cursos/fj33-eats-monolito.git
  ```

2. Abra o Eclipse, definindo como workspace `/home/<usuario-do-curso>/workspace-monolito`. Troque `<usuario-do-curso>` pelo login utilizado no curso.
3. No Eclipse, acesse _File > Import > Existing Maven Projects_ e clique em _Next_. Em _Root Directory_, aponte para o diretório clonado anteriormente.
4. Acesse a classe `EatsApplication` e a execute com _CTRL+F11_. O banco de dados será criado automaticamente e alguns dados serão populados.
5. Teste a URL `http://localhost:8080/restaurantes/1` pelo navegador e verifique se um JSON com os dados de um restaurante foi retornado.
6. Analise o código. Veja:

  - as entidades de negócio
  - os recursos e suas respectivas URIs
  - os serviços e suas funcionalidades.

## Exercício: Executando o front-end

1. Baixe para o Desktop o projeto do front-end, usando o Git, com os comandos:

  ```sh
  cd ~/Desktop
  git clone https://gitlab.com/aovs/projetos-cursos/fj33-eats-ui.git
  ```

2. Abra um Terminal e digite:

  ```sh
  cd ~/Desktop/fj33-eats-ui
  ```

3. Instale as dependências do front-end com o comando:

  ```sh
  npm install
  ```

4. Execute a aplicação com o comando:

  ```sh
  ng serve
  ```

5. Abra um navegador e teste a URL: `http://localhost:4200`. Explore o projeto, fazendo um pedido, confirmando um pedido efetuado, cadastrando um novo restaurante e aprovando-o. Em caso de dúvidas, peça ajuda ao instrutor.
