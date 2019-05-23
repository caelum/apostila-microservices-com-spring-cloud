# Um pouco de Docker

## Exercício opcional: criando uma nova instância do MySQL a partir do Docker

1. Abra um Terminal e baixe a imagem do MySQL 5.7 para sua máquina com o seguinte comando:

  ```sh
  docker pull mysql:5.7
  ```

2. Suba um container do MySQL 5.7 com o seguinte comando:

  ```sh
  docker run --rm -d -p 3307:3306 --name eats.mysql -e MYSQL_ROOT_PASSWORD=caelum123 mysql:5.7
  ```

  Usamos as configurações:

  - `--rm` para remover o container quando ao sair.
  - `-d`, ou `--detach`, para rodar o container no background, imprimindo o id do container e liberando o Terminal para outros comandos.
  - `-p`, ou `--publish`, que associa a porta do container ao host. No nosso caso, associamos a porta `3307` do host à porta padrão do MySQL (`3306`) do container.
  - `--name`, define um apelido para o container.
  - `-e`, ou `--env`, define variáveis de ambiente para o container. No caso, definimos a senha do usuário `root` por meio da variável `MYSQL_ROOT_PASSWORD`.

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

## Exercício: gerenciando containers de infraestrutura com Docker Compose
