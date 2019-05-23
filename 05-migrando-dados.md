# Migrando dados

## Exercício: separando schema do BD de pagamentos do monólito

1. Antes de começar o exercício, é importante que o serviço de pagamentos esteja parado. Certifique-se que a classe `EatsPagamentoServiceApplication` **não** esteja rodando.
2. O Flyway será usado como ferramenta de migração de dados do `eats-pagamento-service`. Para isso, vamos alterar o `pom.xml`:

  ####### eats-pagamento-service/pom.xml

  ```xml
  <dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
  </dependency>
  ```

3. Vamos modificar para `eats_pagamento` o database que o serviço de pagamentos utiliza, alterando o `application.properties`:

  ####### eats-pagamento-service/src/main/resources/application.properties

  ```properties
  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶u̶r̶l̶=̶j̶d̶b̶c̶:̶m̶y̶s̶q̶l̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶/̶e̶a̶t̶s̶?̶c̶r̶e̶a̶t̶e̶D̶a̶t̶a̶b̶a̶s̶e̶I̶f̶N̶o̶t̶E̶x̶i̶s̶t̶=̶t̶r̶u̶e̶
  spring.datasource.url=jdbc:mysql://localhost/eats_pagamento?createDatabaseIfNotExist=true
  ```

  Mantenha o mesmo usuário, que terá acesso a ambos os databases: `eats`, do monólito, e `eats_pagamento`, do serviço de pagamentos.

4. Crie uma pasta `db` em `src/main/resources`. Dentro da pasta criada, crie uma outra chamada `migration`.

  Vamos, então, criar uma primeira migration, que cria a tabela de `pagamento`. Em `db/migration`, crie um arquivo com o nome `V0001__cria-tabela-pagamento.sql` e o seguinte conteúdo:

  ####### eats-pagamento-service/src/main/resources/db/migration/V0001__cria-tabela-pagamento.sql

  ```sql
  CREATE TABLE pagamento (
    id bigint(20) NOT NULL AUTO_INCREMENT,
    valor decimal(19,2) NOT NULL,
    nome varchar(100) DEFAULT NULL,
    numero varchar(19) DEFAULT NULL,
    expiracao varchar(7) NOT NULL,
    codigo varchar(3) DEFAULT NULL,
    status varchar(255) NOT NULL,
    forma_de_pagamento_id bigint(20) NOT NULL,
    pedido_id bigint(20) NOT NULL,
    PRIMARY KEY (id)
  ) ENGINE=InnoDB DEFAULT CHARSET=latin1;
  ```

  Se você achar que é muita digitação, copie o conteúdo acima da seguinte URL: https://gitlab.com/snippets/1859564

5. Então, vamos criar uma segunda migration, que obtem os dados do database `eats`, do monólito, e os insere no database `eats_pagamento`. Crie o arquivo `V0002__migra-dados-de-pagamento.sql` em `db/migration`, conforme a seguir:

  ####### eats-pagamento-service/src/main/resources/db/migration/V0002__migra-dados-de-pagamento.sql

  ```sql
  insert into eats_pagamento.pagamento
    (id, valor, nome, numero, expiracao, codigo, status, forma_de_pagamento_id, pedido_id)
      select id, valor, nome, numero, expiracao, codigo, status, forma_de_pagamento_id, pedido_id
        from eats.pagamento;
  ```

  O trecho de código acima pode ser encontrado em: https://gitlab.com/snippets/1859568

  Essa migração só é possível porque o usuário tem acesso aos dois databases.

6. Execute `EatsPagamentoServiceApplication`. Nos logs, devem aparecer informações sobre a execução dos scripts `.sql`. Algo como:

  ```txt
  2019-05-22 18:33:56.439  INFO 30484 --- [  restartedMain] o.f.c.internal.license.VersionPrinter    : Flyway Community Edition 5.2.4 by Boxfuse
  2019-05-22 18:33:56.448  INFO 30484 --- [  restartedMain] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Starting...
  2019-05-22 18:33:56.632  INFO 30484 --- [  restartedMain] com.zaxxer.hikari.HikariDataSource       : HikariPool-1 - Start completed.
  2019-05-22 18:33:56.635  INFO 30484 --- [  restartedMain] o.f.c.internal.database.DatabaseFactory  : Database: jdbc:mysql://localhost/eats_pagamento (MySQL 5.6)
  2019-05-22 18:33:56.708  INFO 30484 --- [  restartedMain] o.f.core.internal.command.DbValidate     : Successfully validated 2 migrations (execution time 00:00.016s)
  2019-05-22 18:33:56.840  INFO 30484 --- [  restartedMain] o.f.c.i.s.JdbcTableSchemaHistory         : Creating Schema History table: `eats_pagamento`.`flyway_schema_history`
  2019-05-22 18:33:57.346  INFO 30484 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Current version of schema `eats_pagamento`: << Empty Schema >>
  2019-05-22 18:33:57.349  INFO 30484 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Migrating schema `eats_pagamento` to version 0001 - cria-tabela-pagamento
  2019-05-22 18:33:57.596  INFO 30484 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Migrating schema `eats_pagamento` to version 0002 - migra-dados-de-pagamento
  2019-05-22 18:33:57.650  INFO 30484 --- [  restartedMain] o.f.core.internal.command.DbMigrate      : Successfully applied 2 migrations to schema `eats_pagamento` (execution time 00:00.810s)
  ```

7. Verifique se o conteúdo do database `eats_pagamento` condiz com o esperado, digitando os seguintes comandos em um Terminal:

  ```sh
  mysql -u <SEU USUÁRIO> -p eats_pagamento
  ```

  Troque `<SEU USUÁRIO>` pelo usuário informado pelo instrutor. Quando solicitada, digite a senha informada pelo instrutor.

  Dentro do MySQL, execute a seguinte query:

  ```sql
  select * from pagamento;
  ```

  Os pagamentos devem ter sido migrados. Note as colunas `forma_de_pagamento_id` e `pedido_id`.

## Exercício: migrando dados de pagamento para um servidor MySQL específico

1. Abra um Terminal e faça um dump do dados de pagamento com o comando a seguir:

  ```sh
  mysqldump -u <SEU USUÁRIO> -p --opt eats_pagamento > eats_pagamento.sql
  ```

  Peça ao instrutor o usuário do BD e use em `<SEU USUÁRIO>`. Peça também a senha.

  O comando anterior cria um arquivo `eats_pagamento.sql` com todo o schema e dados do database `eats_pagamento`.

  A opção `--opt` equivale às opções:
  
    - `--add-drop-table`, que adiciona um `DROP TABLE` antes de cada `CREATE TABLE`
    - `--add-locks`, que faz um `LOCK TABLES` e `UNLOCK TABLES` em volta de cada dump
    - `--create-options`, que inclui opções específicas do MySQL nos `CREATE TABLE`
    - `--disable-keys`, que desabilita e habilita PKs e FKs em volta de cada `INSERT`
    - `--extended-insert`, que faz um `INSERT` de múltiplos registros de uma vez
    - `--lock-tables`, que trava as tabelas antes de realizar o dump
    - `--quick`, que lê os registros um a um
    - `--set-charset`, que adiciona `default_character_set` ao dump

2. Garanta que o container MySQL do serviço de pagamentos está sendo executado. Para isso, execute em um Terminal:

  ```sh
  docker-compose up -d mysql.pagamento
  ```

  Anote o nome do container criado pelo Docker Compose, através do comando:

  ```sh
  docker ps --format "{{.Image}}\t{{.Names}}"
  ```

  A saída será parecida com:

  ```txt
  mysql:5.7     eats-microservices_mysql.pagamento_1
  ```

  O nome do container que está executando o MySQL de pagamentos é `eats-microservices_mysql.pagamento_1`. Usaremos esse nome logo mais.

3. Pela linha de comando, vamos executar o script `eats_pagamento.sql`, que contém o dump dos dados de pagamento, no container criado pelo Docker Compose.

  Vamos obter o conteúdo do arquivo `eats_pagamento.sql` por meio do comando `cat` e repassá-lo, por meio de um _pipe_ do Unix, para a aplicação de linha de comando do MySQL, que será executada pelo Docker.

  O comando completo de execução do script fica algo como:

  ```sh
  cat eats_pagamento.sql | docker exec -i eats-microservices_mysql.pagamento_1 mysql -upagamento -ppagamento123 eats_pagamento
  ```

  O `docker exec`, executa um comando dentro do container e a opção `-i` (ou `--interactive`) repassa a entrada padrão do host para o container do Docker.

4. Para verificar se o dump foi realizado com sucesso no container Docker que roda o MySQL de pagamentos, vamos acessar pelo cliente de linha de comando do MySQL. Para isso, execute em um Terminal:

  ```sh
  docker exec -it eats-microservices_mysql.pagamento_1 mysql -upagamento -ppagamento123 eats_pagamento
  ```

  Observe que a única diferença em relação ao `docker exec` anterior é a opção `-t` (ou `--tty`), que simula um Terminal dentro do container.

  Devem ser impressas informações sobre o MySQL, que deve estar na versão _5.7.26 MySQL Community Server (GPL)_.

  Digite o seguinte comando SQL e verifique o resultado:

  ```sql
  select * from pagamento;
  ```

  Devem ser exibidos todos os pagamentos já efetuados!
