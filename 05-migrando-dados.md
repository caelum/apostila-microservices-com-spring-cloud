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

3. Pela linha de comando, vamos executar, no container de `mysql.pagamento`, o script `eats_pagamento.sql`.

  Para isso, vamos usar o comando `mysql` informando _host_ e porta do container Docker:

  ```sh
  mysql -u pagamento -p --host 127.0.0.1 --port 3307 eats_pagamento < eats_pagamento.sql
  ```

  _Observação: o comando `mysql` não aceita `localhost`, apenas o IP `127.0.0.1`._

  Quando for solicitada a senha, informe a que definimos no arquivo do Docker Compose: `pagamento123`.

4. Para verificar se a importação do dump foi realizada com sucesso, vamos acessar o comando `mysql` sem passar nenhum arquivo:

  ```sh
  mysql -u pagamento -p --host 127.0.0.1 --port 3307 eats_pagamento
  ```

  Informe a senha `pagamento123`.

  Perceba que o MySQL deve estar na versão _5.7.26 MySQL Community Server (GPL)_, a que definimos no arquivo do Docker Compose.

  Digite o seguinte comando SQL e verifique o resultado:

  ```sql
  select * from pagamento;
  ```

  Devem ser exibidos todos os pagamentos já efetuados!

  Para sair, digite `exit`.

## Exercício: apontando serviço de pagamentos para o BD específico

1. Altere a URL, usuário e senha de BD do serviço de pagamentos, para que apontem para o container Docker do `mysql.pagamento`:

  ####### eats-pagamento-service/src/main/resources/application.properties

  ```properties
  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶u̶r̶l̶=̶j̶d̶b̶c̶:̶m̶y̶s̶q̶l̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶/̶e̶a̶t̶s̶_̶p̶a̶g̶a̶m̶e̶n̶t̶o̶?̶c̶r̶e̶a̶t̶e̶D̶a̶t̶a̶b̶a̶s̶e̶I̶f̶N̶o̶t̶E̶x̶i̶s̶t̶=̶t̶r̶u̶e̶
  spring.datasource.url=jdbc:mysql://localhost:3307/eats_pagamento?createDatabaseIfNotExist=true

  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶u̶s̶e̶r̶n̶a̶m̶e̶=̶<̶S̶E̶U̶ ̶U̶S̶U̶A̶R̶I̶O̶>̶
  spring.datasource.username=pagamento

  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶p̶a̶s̶s̶w̶o̶r̶d̶=̶<̶S̶U̶A̶ ̶S̶E̶N̶H̶A̶>̶
  spring.datasource.password=pagamento123
  ```

  Note que a porta `3307` foi incluída na URL, mas mantivemos ainda `localhost`.

2. Com a classe `EatsPagamentoServiceApplication` sendo executada, abra um Terminal e crie um novo pagamento:

   ```sh
  curl -X POST
    -i
    -H 'Content-Type: application/json'
    -d '{ "valor": 9.99, "nome": "MARIA DE SOUSA", "numero": "777 2222 8888 4444", "expiracao": "2025-04", "codigo": "777", "formaDePagamentoId": 1, "pedidoId": 2 }'
    http://localhost:8081/pagamentos
  ```

  Se desejar, baseie-se na seguinte URL, modificando os valores: https://gitlab.com/snippets/1859389

  A resposta deve ter sucesso, com status `200` e o um id e status `CRIADO` no corpo da resposta.

3. Pelo Eclipse, inicie o monólito e o serviço de distância. Suba também o front-end. Faça um novo pedido, até efetuar o pagamento. Deve funcionar!

4. (opcional) Apague a tabela `pagamento` do database `eats`, do monólito. Remova também o database `eats_pagamento` do MySQL do monólito. Atenção: muito cuidado para não remover dados indesejados!


## Exercício: simplificando o restaurante do serviço de distância

1. O `eats-distancia-service` necessita apenas de um subconjunto das informações do restaurante: o `id`, o `cep`, se o restaurante está `aprovado` e o `tipoDeCozinhaId`.

  Enxugue a classe `Restaurante` do pacote `br.com.caelum.eats.distancia`, deixando apenas as informações realmente necessárias:

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/Restaurante.java

  ```java
  // anotações ...
  public class Restaurante {

    @Id @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Long id;

    @̶N̶o̶t̶B̶l̶a̶n̶k̶ ̶@̶S̶i̶z̶e̶(̶m̶a̶x̶=̶1̶8̶)̶
    p̶r̶i̶v̶a̶t̶e̶ ̶S̶t̶r̶i̶n̶g̶ ̶c̶n̶p̶j̶;̶

    @̶N̶o̶t̶B̶l̶a̶n̶k̶ ̶@̶S̶i̶z̶e̶(̶m̶a̶x̶=̶2̶5̶5̶)̶
    p̶r̶i̶v̶a̶t̶e̶ ̶S̶t̶r̶i̶n̶g̶ ̶n̶o̶m̶e̶;̶ 

    @̶S̶i̶z̶e̶(̶m̶a̶x̶=̶1̶0̶0̶0̶)̶
    p̶r̶i̶v̶a̶t̶e̶ ̶S̶t̶r̶i̶n̶g̶ ̶d̶e̶s̶c̶r̶i̶c̶a̶o̶;̶

    @̶N̶o̶t̶B̶l̶a̶n̶k̶ ̶@̶S̶i̶z̶e̶(̶m̶a̶x̶=̶9̶)̶
    private String cep;

    @̶N̶o̶t̶B̶l̶a̶n̶k̶ ̶@̶S̶i̶z̶e̶(̶m̶a̶x̶=̶3̶0̶0̶)̶
    p̶r̶i̶v̶a̶t̶e̶ ̶S̶t̶r̶i̶n̶g̶ ̶e̶n̶d̶e̶r̶e̶c̶o̶;̶

    @̶P̶o̶s̶i̶t̶i̶v̶e̶
    ̶p̶r̶i̶v̶a̶t̶e̶ ̶B̶i̶g̶D̶e̶c̶i̶m̶a̶l̶ ̶t̶a̶x̶a̶D̶e̶E̶n̶t̶r̶e̶g̶a̶E̶m̶R̶e̶a̶i̶s̶;̶

    @̶P̶o̶s̶i̶t̶i̶v̶e̶ ̶@̶M̶i̶n̶(̶1̶0̶)̶ ̶@̶M̶a̶x̶(̶1̶8̶0̶)̶
    p̶r̶i̶v̶a̶t̶e̶ ̶I̶n̶t̶e̶g̶e̶r̶ ̶t̶e̶m̶p̶o̶D̶e̶E̶n̶t̶r̶e̶g̶a̶M̶i̶n̶i̶m̶o̶E̶m̶M̶i̶n̶u̶t̶o̶s̶;̶

    @̶P̶o̶s̶i̶t̶i̶v̶e̶ ̶@̶M̶i̶n̶(̶1̶0̶)̶ ̶@̶M̶a̶x̶(̶1̶8̶0̶)̶
    p̶r̶i̶v̶a̶t̶e̶ ̶I̶n̶t̶e̶g̶e̶r̶ ̶t̶e̶m̶p̶o̶D̶e̶E̶n̶t̶r̶e̶g̶a̶M̶a̶x̶i̶m̶o̶E̶m̶M̶i̶n̶u̶t̶o̶s̶;̶

    private Boolean aprovado;

    private Long tipoDeCozinhaId;

  }
  ```

  O conteúdo da classe `Restaurante` ficará da seguinte maneira:

  ```java
  // anotações ...
  public class Restaurante {

    @Id @GeneratedValue(strategy=GenerationType.IDENTITY)
    private Long id;

    private String cep;

    private Boolean aprovado;

    private Long tipoDeCozinhaId;

  }
  ```

  Alguns dos imports podem ser removidos:

  ```java
  i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶.̶m̶a̶t̶h̶.̶B̶i̶g̶D̶e̶c̶i̶m̶a̶l̶;̶

  i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶T̶a̶b̶l̶e̶;̶
  i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶v̶a̶l̶i̶d̶a̶t̶i̶o̶n̶.̶c̶o̶n̶s̶t̶r̶a̶i̶n̶t̶s̶.̶M̶a̶x̶;̶
  i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶v̶a̶l̶i̶d̶a̶t̶i̶o̶n̶.̶c̶o̶n̶s̶t̶r̶a̶i̶n̶t̶s̶.̶M̶i̶n̶;̶
  i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶v̶a̶l̶i̶d̶a̶t̶i̶o̶n̶.̶c̶o̶n̶s̶t̶r̶a̶i̶n̶t̶s̶.̶N̶o̶t̶B̶l̶a̶n̶k̶;̶
  i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶v̶a̶l̶i̶d̶a̶t̶i̶o̶n̶.̶c̶o̶n̶s̶t̶r̶a̶i̶n̶t̶s̶.̶P̶o̶s̶i̶t̶i̶v̶e̶;̶
  i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶v̶a̶l̶i̶d̶a̶t̶i̶o̶n̶.̶c̶o̶n̶s̶t̶r̶a̶i̶n̶t̶s̶.̶S̶i̶z̶e̶;̶
  ```

## Exercício: configurando MongoDB no serviço de distância

2. Certifique-se que o container MongoDB do serviço de distância definido no Docker Compose esteja no ar. Para isso, execute em um Terminal:

  ```sh
  docker-compose up -d mongo.distancia
  ```

1. Adicione o _starter_ do Spring Data MongoDB no `pom.xml` do `eats-distancia-service`:

  ####### eats-distancia-service/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
  </dependency>
  ```

2. Crie a classe `RestauranteMongo` no pacote `br.com.caelum.eats.distancia.mongo` com o seguinte código:

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/mongo/RestauranteMongo.java

  ```java
  @Document(collection = "restaurantes")
  @Data
  @AllArgsConstructor
  public class RestauranteMongo {

    @Id
    private Long id;

    private String cep;

    private Long tipoDeCozinhaId;

  }
  ```

  Os imports corretos são os seguintes:

  ```java
  import org.springframework.data.annotation.Id;
  import org.springframework.data.mongodb.core.mapping.Document;

  import lombok.AllArgsConstructor;
  import lombok.Data;
  ```

  Note que o `@Id` foi importado de `org.springframework.data.annotation` e **não** de `javax.persistence` (do JPA).

3. No mesmo pacote `br.com.caelum.eats.distancia.mongo`, crie uma interface `RestauranteMongoRepository` que estende `MongoRepository`:

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/mongo/RestauranteMongoRepository.java

  ```java
  public interface RestauranteMongoRepository extends MongoRepository<RestauranteMongo, Long> {
  }
  ```

4. No arquivo `application.properties` do `eats-distancia-service`, ajuste as configurações do MongoDB:

  ####### eats-distancia-service/src/main/resources/application.properties

  ```properties
  spring.data.mongodb.database=eats_distancia
  spring.data.mongodb.port=27018
  ```

  O database padrão do MongoDB é `test`. A porta padrão é `27017`.

  Para saber sobre outras propriedades, consulte: https://docs.spring.io/spring-boot/docs/current/reference/html/common-application-properties.html

## Exercício: migrando dados de distância do BD do monólito para o MongoDB

1. O serviço de distância apenas considera restaurantes já aprovados. Então, primeiramente, vamos definir em `RestauranteRepository`, do pacote `br.com.caelum.eats.distancia`, um método que obtém todos os restaurantes aprovados, sem paginação:

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/RestauranteRepository.java

  ```java
  interface RestauranteRepository extends JpaRepository<Restaurante, Long> {

    // outros métodos ...

    List<Restaurante> findAllByAprovado(boolean aprovado); // adicionado ...

  }
  ```

  Adicione aos imports:

  ```java
  import java.util.List;
  ```

2. Poderíamos fazer algum script ou usar uma ferramenta para fazer o dump dos dados do MySQL do monólito para o MongoDB do serviço de distância. Mas uma alternativa é fazer essa migração com código Java, disparado assim que o serviço subir. A interface `CommandLineRunner` do Spring Boot ajuda nessa tarefa.

  Observação: antes de continuar, certifique-se que o serviço de distância esteja **parado**.

  Crie uma classe `MigracaoParaMongo` no pacote `br.com.caelum.eats.distancia` e a faça implementar `CommandLineRunner`, definindo o método `run`. Anote essa classe com `@Configuration`.

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/MigracaoParaMongo.java

  ```java
  @Configuration
  public class MigracaoParaMongo implements CommandLineRunner {

    @Override
    public void run(String... args) throws Exception {

    }

  }
  ```

  Os imports corretos são:

  ```java
  import org.springframework.boot.CommandLineRunner;
  import org.springframework.context.annotation.Configuration;
  ```

3. Defina atributos para os repositórios do JPA e do MongoDB na classe `MigracaoParaMongo`. Usando a anotação `@AllArgsConstructor` do Lombok, gere um construtor com todos os atributos, que será usado pelo Spring para injetar instâncias dos repositórios.

  Adicione também a anotação `@Slf4j` do Lombok, que configura um atributo para log.

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/MigracaoParaMongo.java

  ```java
  @Configuration
  @AllArgsConstructor // adicionado
  @Slf4j
  public class MigracaoParaMongo implements CommandLineRunner {

    private RestauranteRepository jpaRepo; // adicionado
    private RestauranteMongoRepository mongoRepo; // adicionado

    @Override
    public void run(String... args) throws Exception {

    }

  }
  ```

  Os novos imports devem ser os seguintes:

  ```java
  import br.com.caelum.eats.distancia.mongo.RestauranteMongoRepository;
  import lombok.AllArgsConstructor;
  import lombok.extern.slf4j.Slf4j;
  ```

4. Implemente um algoritmo com os seguintes passos:

  - obtenha todos os restaurantes aprovados pelo repositório JPA, que aponta para o BD do monólito.
  - percorra os restaurantes aprovado e, pra cada um deles, pegue `id`, `cep` e `tipoDeCozinhaId`.
  - verifique se `id` não existe no MongoDB, protegendo de uma segunda execução dessa classe.
  - se o `id` não existir, instancie um `RestauranteMongo` e use o repositório MongoDB para inseri-lo.

  O código do método `run` ficará semelhante a:

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/MigracaoParaMongo.java

  ```java
  log.info("Iniciando a migração de restaurantes aprovados para Mongo");

  // obtendo todos os restaurantes aprovados
  for (Restaurante restauranteJpa : jpaRepo.findAllByAprovado(true)) {

    // recuperando id, cep e tipo de cozinha
    Long id = restauranteJpa.getId();
    String cep = restauranteJpa.getCep();
    Long tipoDeCozinhaId = restauranteJpa.getTipoDeCozinhaId();

    // verificando se o id existe no mongo
    if (!mongoRepo.existsById(id)) {

      // se não existe, inserindo restaurante no mongo
      RestauranteMongo restauranteMongo = new RestauranteMongo(id, cep, tipoDeCozinhaId);
      mongoRepo.insert(restauranteMongo);

      log.info("Migrado para Mongo: " + restauranteMongo);
    }
  }

  log.info("Fim da migração de restaurantes aprovados para Mongo");
  ```

5. Coloque o serviço de distância no ar, executando a classe `EatsDistanciaServiceApplication`.

  No console do Eclipse, devem aparecer logs parecido com os a seguir:

  ```txt
  2019-05-24 11:20:46.278  INFO 10264 --- [  restartedMain] b.c.c.eats.distancia.MigracaoParaMongo   : Iniciando a migração de restaurantes aprovados para Mongo
  2019-05-24 11:20:46.309  INFO 10264 --- [  restartedMain] o.h.h.i.QueryTranslatorFactoryInitiator  : HHH000397: Using ASTQueryTranslatorFactory
  Hibernate: select restaurant0_.id as id1_0_, restaurant0_.aprovado as aprovado2_0_, restaurant0_.cep as cep3_0_, restaurant0_.tipo_de_cozinha_id as tipo_de_4_0_ from restaurante restaurant0_ where restaurant0_.aprovado=?
  2019-05-24 11:20:46.563  INFO 10264 --- [  restartedMain] org.mongodb.driver.connection            : Opened connection [connectionId{localValue:2, serverValue:6}] to localhost:27018
  2019-05-24 11:20:46.741  INFO 10264 --- [  restartedMain] b.c.c.eats.distancia.MigracaoParaMongo   : Migrado para Mongo: RestauranteMongo(id=1, cep=70238500, tipoDeCozinhaId=1)
  2019-05-24 11:20:46.762  INFO 10264 --- [  restartedMain] b.c.c.eats.distancia.MigracaoParaMongo   : Migrado para Mongo: RestauranteMongo(id=2, cep=79878-978, tipoDeCozinhaId=9)
  2019-05-24 11:20:46.770  INFO 10264 --- [  restartedMain] b.c.c.eats.distancia.MigracaoParaMongo   : Migrado para Mongo: RestauranteMongo(id=3, cep=32131-232, tipoDeCozinhaId=7)
  2019-05-24 11:20:46.770  INFO 10264 --- [  restartedMain] b.c.c.eats.distancia.MigracaoParaMongo   : Fim da migração de restaurantes aprovados para Mongo
  ```

  Note que, segundo os logs, os restaurantes foram migrados.

6. Acesse a interface de linha de comando (CLI) do MongoDB, executando o seguinte comando do Docker Compose:

  ```sh
  docker-compose exec mongo.distancia mongo
  ```

  _Observação: você precisa estar no diretório onde está o arquivo `docker-compose.yml`._

  Na CLI do MongoDB, mostre todos os databases com o comando:

  ```sh
  show dbs
  ```

  Note que o database `eats_distancia`, configurado no serviço, está nos resultados:

  ```txt
  admin           0.000GB
  config          0.000GB
  eats_distancia  0.000GB
  local           0.000GB
  ```

  Acesse o database de distancia com o comando:

  ```sh
  use eats_distancia
  ```

  Deve ser impresso:

  ```txt
  switched to db eats_distancia
  ```

  Execute o comando a seguir para exibir todas as _collections_ do database:

  ```sh
  show collections
  ```

  A collections `restaurantes` deve ser exibida no resultado.

  Liste os restaurantes da collection com o comando:

  ```sh
  db.restaurantes.find();
  ```

  Devem ser exibidos os dados dos restaurantes migrados. Algo como:

  ```txt
  { "_id" : NumberLong(1), "cep" : "70238500", "tipoDeCozinhaId" : NumberLong(1), "_class" : "br.com.caelum.eats.distancia.mongo.RestauranteMongo" }
  { "_id" : NumberLong(2), "cep" : "79878-978", "tipoDeCozinhaId" : NumberLong(9), "_class" : "br.com.caelum.eats.distancia.mongo.RestauranteMongo" }
  { "_id" : NumberLong(3), "cep" : "32131-232", "tipoDeCozinhaId" : NumberLong(7), "_class" : "br.com.caelum.eats.distancia.mongo.RestauranteMongo" }
  ```

  Para sair da CLI do MongoDB, digite `quit()`, com os parênteses.

  > Novos restaurantes aprovados só serão migrados para o MongoDB quando reiniciarmos o serviço de distância. Além disso, como listamos _todos_ os restaurantes aprovados a performance de inicialização pode ser bastante afetada. Mais adiante, corrigiremos a migração de novos restaurantes e alguns outros detalhes. De qualquer forma, talvez a solução ideal envolva ETL ou alguma outra técnica de BD.

## Exercício: usando MongoDB no cálculo de distâncias

1. Adicione na interface `RestauranteMongoRepository`, métodos que recuperam todos os restaurantes e todos de um determinado tipo de cozinha de maneira paginada:

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/mongo/RestauranteMongoRepository.java

  ```java
  public interface RestauranteMongoRepository extends MongoRepository<RestauranteMongo, Long> {

    Page<RestauranteMongo> findAll(Pageable limit);

    Page<RestauranteMongo> findAllByTipoDeCozinhaId(Long tipoDeCozinhaId, Pageable limit);

  }
  ```

  Não deixe de fazer os imports corretos:

  ```java
  import org.springframework.data.domain.Page;
  import org.springframework.data.domain.Pageable;
  ```

2. Modifique a classe `DistanciaService` para que use `RestauranteMongoRepository` e `RestauranteMongo`:

  ```java
  // anotações ....
  class DistanciaService {

    private static final Pageable LIMIT = PageRequest.of(0,5);

    p̶r̶i̶v̶a̶t̶e̶ ̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶R̶e̶p̶o̶s̶i̶t̶o̶r̶y̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶;̶
    private RestauranteMongoRepository restaurantes; // modificado

    public List<RestauranteComDistanciaDto> restaurantesMaisProximosAoCep(String cep) {
      L̶i̶s̶t̶<̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶>̶ ̶a̶p̶r̶o̶v̶a̶d̶o̶s̶ ̶=̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶.̶f̶i̶n̶d̶A̶l̶l̶B̶y̶A̶p̶r̶o̶v̶a̶d̶o̶(̶t̶r̶u̶e̶,̶ ̶L̶I̶M̶I̶T̶)̶.̶g̶e̶t̶C̶o̶n̶t̶e̶n̶t̶(̶)̶;̶
      List<RestauranteMongo> aprovados = restaurantes.findAll(LIMIT).getContent(); // modificado
      return calculaDistanciaParaOsRestaurantes(aprovados, cep);
    }

    public List<RestauranteComDistanciaDto> restaurantesDoTipoDeCozinhaMaisProximosAoCep(Long tipoDeCozinhaId, String cep) {
      L̶i̶s̶t̶<̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶>̶ ̶a̶p̶r̶o̶v̶a̶d̶o̶s̶D̶o̶T̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶ ̶=̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶.̶f̶i̶n̶d̶A̶l̶l̶B̶y̶A̶p̶r̶o̶v̶a̶d̶o̶A̶n̶d̶T̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶I̶d̶(̶t̶r̶u̶e̶,̶ ̶t̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶I̶d̶,̶ ̶L̶I̶M̶I̶T̶)̶.̶g̶e̶t̶C̶o̶n̶t̶e̶n̶t̶(̶)̶;̶
      List<RestauranteMongo> aprovadosDoTipoDeCozinha = restaurantes.findAllByTipoDeCozinhaId(tipoDeCozinhaId, LIMIT).getContent();
      return calculaDistanciaParaOsRestaurantes(aprovadosDoTipoDeCozinha, cep);
    }

    public RestauranteComDistanciaDto restauranteComDistanciaDoCep(Long restauranteId, String cep) {
      R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶ ̶=̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶.̶f̶i̶n̶d̶B̶y̶I̶d̶(̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶I̶d̶)̶.̶o̶r̶E̶l̶s̶e̶T̶h̶r̶o̶w̶(̶(̶)̶ ̶-̶>̶ ̶n̶e̶w̶ ̶R̶e̶s̶o̶u̶r̶c̶e̶N̶o̶t̶F̶o̶u̶n̶d̶E̶x̶c̶e̶p̶t̶i̶o̶n̶(̶)̶)̶;̶
      RestauranteMongo restaurante = restaurantes.findById(restauranteId).orElseThrow(() -> new ResourceNotFoundException()); // modificado
      String cepDoRestaurante = restaurante.getCep();
      BigDecimal distancia = distanciaDoCep(cepDoRestaurante, cep);
      return new RestauranteComDistanciaDto(restauranteId, distancia);
    }

    p̶r̶i̶v̶a̶t̶e̶ ̶L̶i̶s̶t̶<̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶C̶o̶m̶D̶i̶s̶t̶a̶n̶c̶i̶a̶D̶t̶o̶>̶ ̶c̶a̶l̶c̶u̶l̶a̶D̶i̶s̶t̶a̶n̶c̶i̶a̶P̶a̶r̶a̶O̶s̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶(̶L̶i̶s̶t̶<̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶>̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶,̶ ̶S̶t̶r̶i̶n̶g̶ ̶c̶e̶p̶)̶ ̶{̶
    private List<RestauranteComDistanciaDto> calculaDistanciaParaOsRestaurantes(List<RestauranteMongo> restaurantes, String cep) {
      // código omitido ...
    }

    // restante do código ...

  }
  ```

  Adicione os imports necessários:

  ```java
  import br.com.caelum.eats.distancia.mongo.RestauranteMongo;
  import br.com.caelum.eats.distancia.mongo.RestauranteMongoRepository;
  ```

3. Com a classe `EatsDistanciaServiceApplication` sendo executada, teste o serviço de distância.

  Acesse pelo navegador ou pelo cURL uma URL como: http://localhost:8082/restaurantes/mais-proximos/71503510

  Suba o front-end e, dado um CEP, pesquise os restaurantes mais próximos. Deve funcionar!

## Exercício: removendo JPA do serviço de distância

1. Remova as classes `Restaurante` e `RestauranteRepository` do pacote `br.com.caelum.eats.distancia` do serviço de distância:

  - R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶
  - R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶R̶e̶p̶o̶s̶i̶t̶o̶r̶y̶

2. Delete também a classe de migração de dados:

  - M̶i̶g̶r̶a̶c̶a̶o̶P̶a̶r̶a̶M̶o̶n̶g̶o̶

3. Remova as propriedades relacionadas ao datasource do MySQL e a configurações do JPA do `application.properties`:

  ####### eats-distancia-service/src/main/resources/application.properties

  ```properties
  #̶D̶A̶T̶A̶S̶O̶U̶R̶C̶E̶ ̶C̶O̶N̶F̶I̶G̶S̶
  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶u̶r̶l̶=̶j̶d̶b̶c̶:̶m̶y̶s̶q̶l̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶/̶e̶a̶t̶s̶?̶c̶r̶e̶a̶t̶e̶D̶a̶t̶a̶b̶a̶s̶e̶I̶f̶N̶o̶t̶E̶x̶i̶s̶t̶=̶t̶r̶u̶e̶
  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶u̶s̶e̶r̶n̶a̶m̶e̶=̶<̶S̶E̶U̶ ̶U̶S̶U̶Á̶R̶I̶O̶>̶
  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶p̶a̶s̶s̶w̶o̶r̶d̶=̶<̶S̶U̶A̶ ̶S̶E̶N̶H̶A̶>̶

  #̶J̶P̶A̶ ̶C̶O̶N̶F̶I̶G̶S̶
  s̶p̶r̶i̶n̶g̶.̶j̶p̶a̶.̶h̶i̶b̶e̶r̶n̶a̶t̶e̶.̶d̶d̶l̶-̶a̶u̶t̶o̶=̶v̶a̶l̶i̶d̶a̶t̶e̶
  s̶p̶r̶i̶n̶g̶.̶j̶p̶a̶.̶s̶h̶o̶w̶-̶s̶q̶l̶=̶t̶r̶u̶e̶
  ```

4. Exclua as dependências relacionados ao MySQL e Spring Data JPA do `pom.xml` do `eats-distancia-service`:

  ####### eats-distancia-service/pom.xml

  ```xml
  <̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
    <̶g̶r̶o̶u̶p̶I̶d̶>̶m̶y̶s̶q̶l̶<̶/̶g̶r̶o̶u̶p̶I̶d̶>̶
    <̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶m̶y̶s̶q̶l̶-̶c̶o̶n̶n̶e̶c̶t̶o̶r̶-̶j̶a̶v̶a̶<̶/̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶
    <̶s̶c̶o̶p̶e̶>̶r̶u̶n̶t̶i̶m̶e̶<̶/̶s̶c̶o̶p̶e̶>̶
  <̶/̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
  <̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
    <̶g̶r̶o̶u̶p̶I̶d̶>̶o̶r̶g̶.̶s̶p̶r̶i̶n̶g̶f̶r̶a̶m̶e̶w̶o̶r̶k̶.̶b̶o̶o̶t̶<̶/̶g̶r̶o̶u̶p̶I̶d̶>̶
    <̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶s̶p̶r̶i̶n̶g̶-̶b̶o̶o̶t̶-̶s̶t̶a̶r̶t̶e̶r̶-̶d̶a̶t̶a̶-̶j̶p̶a̶<̶/̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶
  <̶/̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
  ```
