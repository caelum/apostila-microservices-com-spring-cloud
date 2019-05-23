# Um pouco de Docker

## Exercício opcional: criando uma nova instância do MySQL a partir do Docker

1. Abra um Terminal e baixe a imagem do MySQL 5.7 para sua máquina com o seguinte comando:

  ```sh
  docker pull mysql:5.7
  ```

2. Suba um container do MySQL 5.7 com o seguinte comando:

  ```sh
  docker run --rm -d -p 3307:3306 --name eats.mysql -e MYSQL_ROOT_PASSWORD=caelum123 -e MYSQL_DATABASE=eats_pagamento -e MYSQL_USER=pagamento -e MYSQL_PASSWORD=pagamento123 mysql:5.7
  ```

  Usamos as configurações:

  - `--rm` para remover o container quando ao sair.
  - `-d`, ou `--detach`, para rodar o container no background, imprimindo o id do container e liberando o Terminal para outros comandos.
  - `-p`, ou `--publish`, que associa a porta do container ao host. No nosso caso, associamos a porta `3307` do host à porta padrão do MySQL (`3306`) do container.
  - `--name`, define um apelido para o container.
  - `-e`, ou `--env`, define variáveis de ambiente para o container. No caso, definimos a senha do usuário `root` por meio da variável `MYSQL_ROOT_PASSWORD`. Também definimos um database a ser criado na inicialização do container e seu usuário e senha, pelas variáveis `MYSQL_DATABASE`, `MYSQL_USER` e `MYSQL_PASSWORD`, respectivamente.

  Mais detalhes sobre essas opções podem ser encontrados em: https://docs.docker.com/engine/reference/commandline/run/

3. Liste os containers que estão sendo executados pelo Docker com o comando:

  ```sh
  docker ps
  ```

  Deve aparecer algo como:

  ```txt
  CONTAINER ID                                                       IMAGE               COMMAND                         CREATED             STATUS              PORTS                               NAMES
  183bc210a6071b46c4dd790858e07573b28cfa6394a7017cb9fa6d4c9af71563   mysql:5.7           "docker-entrypoint.sh mysqld"   16 minutes ago      Up 16 minutes       33060/tcp, 0.0.0.0:3307->3306/tcp   eats.mysql
  ```

4. Acesse os logs do container `eats.mysql` com o comando:

  ```sh
  docker logs eats.mysql
  ```

5. Pare a execução do container `eats.mysql` com o comando a seguir:

  ```sh
  docker stop eats.mysql
  ```

## Exercício opcional: criando uma instância do MongoDB a partir do Docker

1. Baixe a imagem do MongoDB 3.6 com o comando a seguir:

  ```sh
  docker pull mongo:3.6
  ```

2. Execute o MongoDb 3.6 em um container com o comando:

  ```sh
  docker run --rm -d -p 27018:27017 --name eats.mongo mongo:3.6
  ```

  Note que mudamos a porta do host para `27018`. A porta padrão do MongoDB é `27017`.

3. Liste os containers, obtenha os logs de `eats.mongo` e pare a execução. Use como exemplo os comandos listados no exercício do MySQL.

## Exercício: gerenciando containers de infraestrutura com Docker Compose

1. No seu Desktop, defina um arquivo `docker-compose.yml` com o seguinte conteúdo:

  ####### docker-compose.yml

  ```yml
  version: '3'

  services:
    mysql.pagamento:
      image: mysql:5.7
      restart: on-failure
      ports:
        - "3307:3306"
      environment:
        MYSQL_ROOT_PASSWORD: caelum123
        MYSQL_DATABASE: eats_pagamento
        MYSQL_USER: pagamento
        MYSQL_PASSWORD: pagamento123
    mongo.distancia:
      image: mongo:3.6
      restart: on-failure
      ports:
        - "27018:27017"
  ```

  Observação: mantenha os TABs certinhos. São muito importantes em um arquivo `.yml`. Em caso de dúvida, peça ajuda ao instrutor.

  Caso não queira digita, o conteúdo do `docker-compose.yml` pode ser encontrado em: https://gitlab.com/snippets/1859850

2. Mude para o diretório do `docker-compose.yml` , o Desktop, no caso. Suba ambos os containers, do MySQL e do MongoDB, com o comando:

  ```sh
  docker-compose up -d
  ```

  A opção `-d`, ou `--detach`, roda os containers no background, liberando o Terminal.

  Observe os containers sendo executados com o comando do Docker:

  ```sh
  docker ps
  ```

  Deverá ser impresso algo como:

  ```txt
  CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
  49bf0d3241ad        mysql:5.7           "docker-entrypoint..."   26 minutes ago      Up 3 minutes        33060/tcp, 0.0.0.0:3307->3306/tcp   eats-microservices_mysql.pagamento_1
  4890dcb9e898        mongo:3.6           "docker-entrypoint..."   26 minutes ago      Up 3 minutes        0.0.0.0:27018->27017/tcp            eats-microservices_mongo.distancia_1
  ```

  É possível formatar as informações, deixando a saída do comando mais enxuta. Para isso, use a opção `--format`:

  ```sh
  docker ps --format "{{.Image}}\t{{.Names}}"
  ```

  O resultado será semelhante a:

  ```txt
  mysql:5.7     eats-microservices_mysql.pagamento_1
  mongo:3.6     eats-microservices_mongo.distancia_1
  ```

3. Você pode obter os logs de ambos os containers com o seguinte comando:

  ```sh
  docker-compose logs
  ```

  Caso queira os logs apenas de um container específico, basta passar o nome do _service_ (o termo para uma configuração do Docker Compose). Para o MySQL, seria algo como:

  ```sh
  docker-compose logs mysql.pagamento
  ```

4. Para parar todos os _services_ e remover os containers, volumes e imagens associados, use:

  ```sh
  docker-compose down
  ```

  Se desejar apenas parar os _services_, sem remover nada, o seguinte comando dever ser usado:

  ```sh
  docker-compose stop
  ```

  Para iniciá-los novamente, faça um `docker-compose start`.

  É possível parar e remover um _service_ específico, passando-o no final do comando.
