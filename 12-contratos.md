# Contratos

## Exercício: fornecendo stubs do contrato a partir do servidor

1. Adicione ao `pom.xml` do serviço de distância, uma dependência ao starter do Spring Cloud Contract Verifier:

  ####### eats-distancia-service/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-verifier</artifactId>
    <scope>test</scope>
  </dependency>
  ```

  Adicione também o plugin Maven do Spring Cloud Contract:

  ####### eats-distancia-service/pom.xml

  ```xml
  <plugin>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-maven-plugin</artifactId>
    <version>2.1.2.RELEASE</version>
    <extensions>true</extensions>
    <configuration>
      <packageWithBaseClasses>br.com.caelum.eats.distancia.base</packageWithBaseClasses>
    </configuration>
  </plugin>
  ```

  Note que na configuração `packageWithBaseClasses` definimos um pacote para as classes base, que serão usadas na execução de testes.

2. No pacote `br.com.caelum.eats.distancia.base`, definido anteriormente no plugin do Maven, crie a classe a classe `RestaurantesBase`, que será a base para a execução de testes baseados no contrato do controller de restaurantes.

  Nessa classe injete o `RestaurantesController`, passando a instância para o `RestAssuredMockMvc`, uma integração da biblioteca REST Assured com o MockMvc do Spring.

  Além disso, injetaremos um `RestauranteMongoRepository` anotado com `@MockBean`, fazendo com que a instância seja gerenciada pelo Mockito. Usaremos essa instância como um _stub_, registrando uma chamada ao método `insert` que retorna o próprio objeto passado como parâmetro.

  ####### eats-distancia-service/src/test/java/br/com/caelum/eats/distancia/base/RestaurantesBase.java

  ```java
  @SpringBootTest
  @RunWith(SpringRunner.class)
  public class RestaurantesBase {

    @Autowired
    private RestaurantesController restaurantesController;

    @MockBean
    private RestauranteMongoRepository restauranteMongoRepository;

    @Before
    public void before() {
      RestAssuredMockMvc.standaloneSetup(restaurantesController);

      Mockito.when(restauranteMongoRepository.insert(Mockito.any(RestauranteMongo.class)))
        .thenAnswer((InvocationOnMock invocation) -> {
          RestauranteMongo restauranteMongo = invocation.getArgument(0);
          return restauranteMongo;
        });

    }
  }
  ```

  Os imports são os seguintes:

  ```java
  import org.junit.Before;
  import org.junit.runner.RunWith;
  import org.mockito.Mockito;
  import org.mockito.invocation.InvocationOnMock;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.boot.test.context.SpringBootTest;
  import org.springframework.boot.test.mock.mockito.MockBean;
  import org.springframework.test.context.junit4.SpringRunner;

  import br.com.caelum.eats.distancia.RestaurantesController;
  import br.com.caelum.eats.distancia.mongo.RestauranteMongo;
  import br.com.caelum.eats.distancia.mongo.RestauranteMongoRepository;

  import io.restassured.module.mockmvc.RestAssuredMockMvc;
  ```

3. No Eclipse, com o botão direito no projeto `eats-distancia-service`, acesse o menu _New > Folder..._. Define em _Folder name_, o caminho `src/test/resources/contracts/restaurantes`.

  _Dica: faça um refresh no projeto para que o diretório `src/test/resources` seja reconhecido como um source folder._

  Dentro desse diretório, crie o arquivo `deveAdicionarNovoRestaurante.groovy`. Esse arquivo conterá o contrato que estamos definindo, utilizando uma DSL Groovy:

  ####### eats-distancia-service/src/test/resources/contracts/restaurantes/deveAdicionarNovoRestaurante.groovy

  ```groovy
  import org.springframework.cloud.contract.spec.Contract
  Contract.make {
      description "deve adicionar novo restaurante"
      request{
          method POST()
          url("/restaurantes")
          body([
        id: 2,
        cep: '71500-000',
        tipoDeCozinhaId: 1
      ])
          headers {
        contentType('application/json')
      }
      }
      response {
          status 201
          body([
        id: 2,
        cep: '71500-000',
        tipoDeCozinhaId: 1
      ])
          headers {
        contentType('application/json')
      }
      }
  }
  ```

4. Ajuste a classe `RestauranteMongo.java` para que seja definido um construtor padrão, que será utilizado na execução do teste gerado a partir do contrato:

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/mongo/RestauranteMongo.java

  ```java
  @Document(collection = "restaurantes")
  @Data
  @AllArgsConstructor
  @NoArgsConstructor // adicionado
  public class RestauranteMongo {
  
    // código omitido ...
  
  }
  ```

5. Abra um Terminal e, no diretório do serviço de distância, execute os comandos a seguir:

  ```sh
  cd ~/Desktop/eats-distancia-service
  mvn clean install
  ```

  Aguarde a execução do build. As mensagens finais devem conter:

  ```txt
  [INFO] Results:
  [INFO]
  [INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0
  [INFO]
  [INFO]
  ...
  [INFO] ------------------------------------------------------------------------
  [INFO] BUILD SUCCESS
  [INFO] ------------------------------------------------------------------------
  [INFO] Total time: 02:45 min
  [INFO] Finished at: 2019-07-03T16:45:17-03:00
  [INFO] ------------------------------------------------------------------------
  ```

  Observe o conteúdo da seguinte classe gerada pelo Spring Cloud Contract: eats-distancia-service/target/generated-test-sources/contracts/br/com/caelum/eats/distancia/base/RestaurantesTest.java

  Essa classe é responsável por verificar que o próprio servidor segue o contrato.

  Abra, com o _Archive Manager_ ou algum outro gerenciador de arquivos compactados, o arquivo `eats-distancia-service/target/eats-distancia-service-0.0.1-SNAPSHOT-stubs.jar`.

  Note que, dentro do JAR, há o arquivo `/META-INF/br.com.caelum/eats-distancia-service/0.0.1-SNAPSHOT/mappings/restaurantes/deveAdicionarNovoRestaurante.json`.

  Esse arquivo é compatível com a ferramenta WireMock, que permite a execução de um _mock server_ para testes de API.

## Exercício: usando stubs do contrato no cliente

1. No `pom.xml` do módulo `eats-application` do monólito, adicione os starters do Spring Boot Test e do Spring Cloud Contract Stub Runner:

  ####### eats-monolito-modular/eats-application/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
    <scope>test</scope>
  </dependency>
  ```

2. No source folder `src/test/java`, dentro do pacote `br.com.caelum.eats`, crie a classe `DistanciaRestClientWiremockTest`.

  Anote-a com `@AutoConfigureStubRunner`, passando no parâmetro `ids`, o `groupId` e `artifactId` do JAR gerado no exercício anterior. Use um `+` para sempre obter a última versão. Passe também a porta que deve ser usada pelo servidor do WireMock. No parâmetro `stubsMode`, informe que o JAR do contrato será obtido do repositório `LOCAL` (o diretório `.m2`).

  Em um método anotado com `@Before`, crie uma instância do `DistanciaRestClient`, o ponto de integração do monólito com o serviço de distância. Passe um `RestTemplate` sem balanceamento de carga e fixe a URL para a porta definida na anotação `@AutoConfigureStubRunner`.

  Invoque o método `novoRestauranteAprovado` de `DistanciaRestClient`, passando um objeto `Restaurante` com valores condizentes com o contrato. Como o método é `void`, em caso de exceção force a falha do teste.

  ####### eats-monolito-modular/eats-application/src/test/java/br/com/caelum/eats/DistanciaRestClientWiremockTest.java

  ```java
  @SpringBootTest
  @RunWith(SpringRunner.class)
  @AutoConfigureStubRunner(ids = "br.com.caelum:eats-distancia-service:+:stubs:9992", stubsMode = StubsMode.LOCAL)
  public class DistanciaRestClientWiremockTest {

    private DistanciaRestClient distanciaClient;

    @Before
    public void before() {
      RestTemplate restTemplate = new RestTemplate();
      distanciaClient = new DistanciaRestClient(restTemplate, "http://localhost:9992");
    }

    @Test
    public void deveAdicionarUmNovoRestaurante() {
      try {

        TipoDeCozinha tipoDeCozinha = new TipoDeCozinha(1L, "Chinesa");

        Restaurante restaurante = new Restaurante();
        restaurante.setId(2L);
        restaurante.setCep("71500-000");
        restaurante.setTipoDeCozinha(tipoDeCozinha);

        distanciaClient.novoRestauranteAprovado(restaurante);

      } catch (Exception ex) {
        Assert.fail("Não deveria lançar exceção!");
      }
    }
  }
  ```

  _Observação: o try/catch e a falha forçada podem ser omitidos sem prejudicar o funcionamento dos testes._

  Seguem os imports:

  ```java
  import org.junit.Assert;
  import org.junit.Before;
  import org.junit.Test;
  import org.junit.runner.RunWith;
  import org.springframework.boot.test.context.SpringBootTest;
  import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
  import org.springframework.cloud.contract.stubrunner.spring.StubRunnerProperties.StubsMode;
  import org.springframework.test.context.junit4.SpringRunner;
  import org.springframework.web.client.RestTemplate;

  import br.com.caelum.eats.admin.TipoDeCozinha;
  import br.com.caelum.eats.restaurante.DistanciaRestClient;
  import br.com.caelum.eats.restaurante.Restaurante;
  ```

3. No Eclipse, clique com o botão direito na classe `DistanciaRestClientWiremockTest` e, então, em _Run As... > JUnit Test_. Use o JUnit 4.

  Aguarde a execução dos testes. Sucesso!

  Observe, nos logs, a definição no WireMock do contrato descrito no arquivo `deveAdicionarNovoRestaurante.json` do JAR de stubs.

  ```txt
  2019-07-03 17:41:27.681  INFO [monolito,,,] 32404 --- [tp1306763722-35] WireMock                                 : Admin request received:
  127.0.0.1 - POST /mappings

  Connection: [keep-alive]
  User-Agent: [Apache-HttpClient/4.5.5 (Java/1.8.0_201)]
  Host: [localhost:9992]
  Content-Length: [718]
  Content-Type: [text/plain; charset=UTF-8]
  {
    "id" : "64ce3139-e460-405d-8ebb-fe7f527018c3",
    "request" : {
      "url" : "/restaurantes",
      "method" : "POST",
      "headers" : {
        "Content-Type" : {
          "matches" : "application/json.*"
        }
      },
      "bodyPatterns" : [ {
        "matchesJsonPath" : "$[?(@.['tipoDeCozinhaId'] == 1)]"
      }, {
        "matchesJsonPath" : "$[?(@.['cep'] == '71500-000')]"
      }, {
        "matchesJsonPath" : "$[?(@.['id'] == 2)]"
      } ]
    },
    "response" : {
      "status" : 201,
      "body" : "{\"tipoDeCozinhaId\":1,\"id\":2,\"cep\":\"71500-000\"}",
      "headers" : {
        "Content-Type" : "application/json"
      },
      "transformers" : [ "response-template" ]
    },
    "uuid" : "64ce3139-e460-405d-8ebb-fe7f527018c3"
  }
  ```

  Mais adiante, observe que o WireMock recebeu uma requisição POST na URL `/restaurantes` e enviou a resposta descrita no contrato:

  ```txt
  2019-07-03 17:41:37.689  INFO [monolito,,,] 32404 --- [tp1306763722-36] WireMock                                 : Request received:
  127.0.0.1 - POST /restaurantes

  User-Agent: [Java/1.8.0_201]
  Connection: [keep-alive]
  Host: [localhost:9992]
  Accept: [application/json, application/*+json]
  Content-Length: [46]
  Content-Type: [application/json;charset=UTF-8]
  {"id":2,"cep":"71500-000","tipoDeCozinhaId":1}


  Matched response definition:
  {
    "status" : 201,
    "body" : "{\"tipoDeCozinhaId\":1,\"id\":2,\"cep\":\"71500-000\"}",
    "headers" : {
      "Content-Type" : "application/json"
    },
    "transformers" : [ "response-template" ]
  }

  Response:
  HTTP/1.1 201
  Content-Type: [application/json]
  Matched-Stub-Id: [64ce3139-e460-405d-8ebb-fe7f527018c3]
  ```
