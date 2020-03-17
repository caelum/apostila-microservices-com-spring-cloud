# Apêndice: Uma introdução ao Docker

## Criando uma nova instância do MySQL a partir do Docker

Abra um Terminal e baixe a imagem do MySQL 5.7 para sua máquina com o seguinte comando:

```sh
docker image pull mysql:5.7
```

Suba um container do MySQL 5.7 com o seguinte comando:

```sh
docker container run --rm -d -p 3308:3306 --name eats.mysql -e MYSQL_ROOT_PASSWORD=caelum123 -e MYSQL_DATABASE=eats_pagamento -e MYSQL_USER=pagamento -e MYSQL_PASSWORD=pagamento123 -v eats.mysql:/var/lib/mysql mysql:5.7
```

Usamos as configurações:

- `--rm` para remover o container quando ao sair.
- `-d`, ou `--detach`, para rodar o container no background, imprimindo o id do container e liberando o Terminal para outros comandos.
- `-p`, ou `--publish`, que associa a porta do container ao host. No nosso caso, associamos a porta `3308` do host à porta padrão do MySQL (`3306`) do container.
- `--name`, define um apelido para o container.
- `-e`, ou `--env`, define variáveis de ambiente para o container. No caso, definimos a senha do usuário `root` por meio da variável `MYSQL_ROOT_PASSWORD`. Também definimos um database a ser criado na inicialização do container e seu usuário e senha, pelas variáveis `MYSQL_DATABASE`, `MYSQL_USER` e `MYSQL_PASSWORD`, respectivamente.
- `-v` para preservar os dados fora do container

Mais detalhes sobre essas opções podem ser encontrados em: https://docs.docker.com/engine/reference/commandline/run/

> O curso [Infraestrutura ágil com Docker e Docker Swarm](https://www.caelum.com.br/curso-infraestrutura-agil-com-docker-e-docker-swarm) (DO-26) da Caelum aprofunda nos conceitos do Docker e tecnologias relacionadas.

Liste os containers que estão sendo executados pelo Docker com o comando:

```sh
docker container ps
```

Deve aparecer algo como:

```txt
CONTAINER ID                                                       IMAGE               COMMAND                         CREATED             STATUS              PORTS                               NAMES
183bc210a6071b46c4dd790858e07573b28cfa6394a7017cb9fa6d4c9af71563   mysql:5.7           "docker-entrypoint.sh mysqld"   16 minutes ago      Up 16 minutes       33060/tcp, 0.0.0.0:3308->3306/tcp   eats.mysql
```

Acesse os logs do container `eats.mysql` com o comando:

```sh
docker container logs eats.mysql
```

Podemos executar um comando dentro de um container por meio do `docker exec`. Para acessar a CLI do MySQL com o database e usuário criados anteriormente, devemos executar:

```sh
docker container exec -it eats.mysql mysql -upagamento -p eats_pagamento
```

A opção `-i` (ou `--interactive`) repassa a entrada padrão do host para o container do Docker.

Já a opção `-t` (ou `--tty`) simula um Terminal dentro do container.

Informe a senha `pagamento123`, registrada em passos anteriores.

Devem ser impressas informações sobre o MySQL. Entre elas, uma versão semelhante a: _5.7.26 MySQL Community Server (GPL)_.

Digite o seguinte comando:

```sql
show databases;
```

Deve ser exibido algo semelhante a:

```txt
+--------------------+
| Database           |
+--------------------+
| information_schema |
| eats_pagamento     |
+--------------------+
2 rows in set (0.00 sec)
```

Para sair, digite `exit`.

Pare a execução do container `eats.mysql` com o comando a seguir:

```sh
docker container stop eats.mysql
```

## Criando uma instância do MongoDB a partir do Docker

Baixe a imagem do MongoDB 3.6 com o comando a seguir:

```sh
docker image pull mongo:3.6
```

Execute o MongoDb 3.6 em um container com o comando:

```sh
docker container run --rm -d -p 27018:27017 --name eats.mongo -v eats.mongo:/data/db mongo:3.6
```

Note que mudamos a porta do host para `27018`. A porta padrão do MongoDB é `27017`.

Liste os containers e obtenha os logs de `eats.mongo`. Use como exemplo os comandos listados para o MySQL.

Acesse a CLI do `eats.mongo` com o comando:

```sh
docker container exec -it eats.mongo mongo
```

Devem aparecer informações sobre o MongoDB, como a versão, que deve ser algo como: _MongoDB server version: 3.6.12_.

Digite o seguinte comando:

```sh
show dbs
```

Deve ser impresso algo parecido com:

```txt
admin   0.000GB
config  0.000GB
local   0.000GB
```

Para sair, digite `quit()`, com os parênteses.

Pare a execução do container `eats.mongo` com o comando a seguir:

```sh
docker container stop eats.mongo
```

## Simplificando o gerenciamento dos containers com Docker Compose

O Docker Compose permite definir uma série de _services_ (cuidado com o nome!) que permitem descrever a configuração de containers. Com essa ferramenta, é possível disparar novas instâncias de maneira muito fácil!

Para isso, basta definirmos um arquivo `docker-compose.yml`. Os services devem ter um nome e referências às imagens do Docker Hub e podem ter definições de portas utilizadas, variáveis de ambiente e diversas outras configurações.

####### docker-compose.yml

```yml
version: '3'

services:
  mysql.pagamento:
    image: mysql:5.7
    ports:
      - "3308:3306"
    environment:
      MYSQL_ROOT_PASSWORD: caelum123
      MYSQL_DATABASE: eats_pagamento
      MYSQL_USER: pagamento
      MYSQL_PASSWORD: pagamento123
    volumes:
      - mysql.eats.pagamento:/var/lib/mysql
  mongo.distancia:
    image: mongo:3.6
    ports:
      - "27018:27017"
    volumes:
      - mongo.eats.distancia:/data/db
volumes:
  mysql.eats.pagamento:
  mongo.eats.distancia:
```

Para subir os serviços definidos no `docker-compose.yml`, execute o comando:

```sh
docker-compose up -d
```
A opção `-d`, ou `--detach`, roda os containers no background, liberando o Terminal.

É possível executar um Terminal diretamente em uma dos containers criados pelo Docker Compose com o comando `docker-compose exec`.

Por exemplo, para acessar o comando `mongo`, a interface de linha de comando do MongoDB, do service `mongo.distancia`, faça:

```sh
docker-compose exec mongo.distancia mongo
```

Você pode obter os logs de ambos os containers com o seguinte comando:

```sh
docker-compose logs
```

Caso queira os logs apenas de um container específico, basta passar o nome do _service_ (o termo para uma configuração do Docker Compose). Para o MySQL, seria algo como:

```sh
docker-compose logs mysql.pagamento
```

Para interromper todos os _services_ sem remover os containers, volumes e imagens associados, use:

```sh
docker-compose stop
```

Depois de parados com `stop`, para iniciá-los novamente, faça um `docker-compose start`.

É possível parar um _service_ específico, passando seu nome no final do comando.
