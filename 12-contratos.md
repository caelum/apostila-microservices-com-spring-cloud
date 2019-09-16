# Contratos

## Fornecendo stubs do contrato a partir do servidor

Adicione ao `pom.xml` do serviço de distância, uma dependência ao starter do Spring Cloud Contract Verifier:

####### fj33-eats-distancia-service/pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-contract-verifier</artifactId>
  <scope>test</scope>
</dependency>
```

Adicione também o plugin Maven do Spring Cloud Contract:

####### fj33-eats-distancia-service/pom.xml

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

No Eclipse, com o botão direito no projeto `eats-distancia-service`, acesse o menu _New > Folder..._. Defina em _Folder name_, o caminho `src/test/resources/contracts/restaurantes`.

_Dica: para que o diretório `src/test/resources` seja reconhecido como um source folder faça um refresh no projeto e, com o botão direito no projeto, clique em Maven > Update Project... e, então, em OK._

Dentro desse diretório, crie o arquivo `deveAdicionarNovoRestaurante.groovy`. Esse arquivo conterá o contrato que estamos definindo, utilizando uma DSL Groovy:

####### fj33-eats-distancia-service/src/test/resources/contracts/restaurantes/deveAdicionarNovoRestaurante.groovy

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

No serviço de distância, clicando com o botão direto no projeto e então acesse o menu _New > Folder..._ e defina, em _Folder name_, o caminho `src/test/java`.

No pacote `br.com.caelum.eats.distancia.base` do _source folder_ `src/test/java`, definido anteriormente no plugin do Maven, crie a classe a classe `RestaurantesBase`, que será a base para a execução de testes baseados no contrato do controller de restaurantes.

_Dica: para que o diretório `src/test/java` seja reconhecido como um source folder faça um refresh no projeto e, com o botão direito no projeto, clique em Maven > Update Project... e, então, em OK._

Nessa classe injete o `RestaurantesController`, passando a instância para o `RestAssuredMockMvc`, uma integração da biblioteca REST Assured com o MockMvc do Spring.

Além disso, injetaremos um `RestauranteRepository` anotado com `@MockBean`, fazendo com que a instância seja gerenciada pelo Mockito. Usaremos essa instância como um _stub_, registrando uma chamada ao método `insert` que retorna o próprio objeto passado como parâmetro.

Para evitar que o Spring tente conectar com o MongoDB durante os testes, anote a classe com `@ImportAutoConfiguration`, passando na propriedade `exclude` a classe `MongoAutoConfiguration`.

_Observação: o nome da classe `RestaurantesBase` usa como prefixo o diretório de nosso contrato (`restaurantes`). O sufixo `Base` é um requisito do Spring Cloud Contract._

####### fj33-eats-distancia-service/src/test/java/br/com/caelum/eats/distancia/base/RestaurantesBase.java

```java
@ImportAutoConfiguration(exclude=MongoAutoConfiguration.class)
@SpringBootTest
@RunWith(SpringRunner.class)
class RestaurantesBase {

  @Autowired
  private RestaurantesController restaurantesController;

  @MockBean
  private RestauranteRepository restauranteRepository;

  @Before
  public void before() {
    RestAssuredMockMvc.standaloneSetup(restaurantesController);

    Mockito.when(restauranteRepository.insert(Mockito.any(Restaurante.class)))
      .thenAnswer((InvocationOnMock invocation) -> {
        Restaurante restaurante = invocation.getArgument(0);
        return restaurante;
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

import br.com.caelum.eats.distancia.Restaurante;
import br.com.caelum.eats.distancia.RestauranteRepository;
import br.com.caelum.eats.distancia.RestaurantesController;
import io.restassured.module.mockmvc.RestAssuredMockMvc;

import io.restassured.module.mockmvc.RestAssuredMockMvc;
```

Altere a classe `RestaurantesController`, tornando-a pública:

```java
// anotações omitidas ...
public class RestaurantesController { // modificado

  // código omitido ...

}
```

Também torne pública a interface `RestauranteRepository`:

```java
public interface RestauranteRepository extends MongoRepository<Restaurante, Long> {

  // código omitido ...

}
```

Abra um Terminal e, no diretório do serviço de distância, execute:

```sh
mvn clean install
```

Depois do sucesso no build, podemos observar que uma classe `RestaurantesTest` foi gerada pelo Spring Cloud Contract: 

####### fj33-eats-distancia-service/target/generated-test-sources/contracts/br/com/caelum/eats/distancia/base/RestaurantesTest.java

```java
public class RestaurantesTest extends RestaurantesBase {

  @Test
  public void validate_deveAdicionarNovoRestaurante() throws Exception {
    // given:
      MockMvcRequestSpecification request = given()
          .header("Content-Type", "application/json")
          .body("{\"id\":2,\"cep\":\"71500-000\",\"tipoDeCozinhaId\":1}");

    // when:
      ResponseOptions response = given().spec(request)
          .post("/restaurantes");

    // then:
      assertThat(response.statusCode()).isEqualTo(201);
      assertThat(response.header("Content-Type")).matches("application/json.*");
    // and:
      DocumentContext parsedJson = JsonPath.parse(response.getBody().asString());
      assertThatJson(parsedJson).field("['tipoDeCozinhaId']").isEqualTo(1);
      assertThatJson(parsedJson).field("['cep']").isEqualTo("71500-000");
      assertThatJson(parsedJson).field("['id']").isEqualTo(2);
  }

}
```

A classe `RestaurantesTest` é responsável por verificar que o próprio servidor segue o contrato.

Além do _fat JAR_ gerado pelo Spring Boot com a aplicação, o Spring Cloud Contract gera um outro JAR com stubs do contrato:
`eats-distancia-service/target/eats-distancia-service-0.0.1-SNAPSHOT-stubs.jar`.

Dentro do diretório `/META-INF/br.com.caelum/eats-distancia-service/0.0.1-SNAPSHOT/` desse JAR, no subdiretório `contracts/restaurantes/`, há a DSL Groovy que descreve o contrato, no arquivo `deveAdicionarNovoRestaurante.groovy`.

Já no subdiretório `mappings/restaurantes/`, há o arquivo `deveAdicionarNovoRestaurante.json`:

```json
{
  "id" : "80bcbe99-0504-4ff9-8f32-e9eb0645b646",
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
  "uuid" : "80bcbe99-0504-4ff9-8f32-e9eb0645b646"
}
```

Esse JSON é compatível com a ferramenta WireMock, que permite a execução de um _mock server_ para testes de API.

## Usando stubs do contrato no cliente

No `pom.xml` do módulo `eats-application` do monólito, adicione o starter do Spring Cloud Contract Stub Runner:

####### fj33-eats-monolito-modular/eats/eats-application/pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
  <scope>test</scope>
</dependency>
```

No source folder `src/test/java` do módulo `eats-application`, dentro do pacote `br.com.caelum.eats`, crie a classe `DistanciaRestClientWiremockTest`.

Anote-a com `@AutoConfigureStubRunner`, passando no parâmetro `ids`, o `groupId` e `artifactId` do JAR gerado no exercício anterior. Use um `+` para sempre obter a última versão. Passe também a porta que deve ser usada pelo servidor do WireMock. No parâmetro `stubsMode`, informe que o JAR do contrato será obtido do repositório `LOCAL` (o diretório `.m2`).

Em um método anotado com `@Before`, crie uma instância do `DistanciaRestClient`, o ponto de integração do monólito com o serviço de distância. Passe um `RestTemplate` sem balanceamento de carga e fixe a URL para a porta definida na anotação `@AutoConfigureStubRunner`.

Invoque o método `novoRestauranteAprovado` de `DistanciaRestClient`, passando um objeto `Restaurante` com valores condizentes com o contrato. Como o método é `void`, em caso de exceção force a falha do teste.

####### fj33-eats-monolito-modular/eats/eats-application/src/test/java/br/com/caelum/eats/DistanciaRestClientWiremockTest.java

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
      TipoDeCozinha tipoDeCozinha = new TipoDeCozinha(1L, "Chinesa");

      Restaurante restaurante = new Restaurante();
      restaurante.setId(2L);
      restaurante.setCep("71500-000");
      restaurante.setTipoDeCozinha(tipoDeCozinha);

      distanciaClient.novoRestauranteAprovado(restaurante);
  }
}
```

_Observação: o teste anterior falhará quando for lançada uma exceção._

Seguem os imports:

```java
import org.junit.Before;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cloud.contract.stubrunner.spring.AutoConfigureStubRunner;
import org.springframework.cloud.contract.stubrunner.spring.StubRunnerProperties.StubsMode;
import org.springframework.test.context.junit4.SpringRunner;
import org.springframework.web.client.RestTemplate;

import br.com.caelum.eats.administrativo.TipoDeCozinha;
import br.com.caelum.eats.restaurante.DistanciaRestClient;
import br.com.caelum.eats.restaurante.Restaurante;
```

No módulo de restaurante do monólito, torne públicos a classe `DistanciaRestClient`, seu construtor e o método `novoRestauranteAprovado`:

####### fj33-eats-monolito-modular/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/DistanciaRestClient.java

```java
@Slf4j
@Service
public class DistanciaRestClient { // modificado

  // código omitido ...

  public DistanciaRestClient(RestTemplate restTemplate, // modificado
                                      @Value("${configuracao.distancia.service.url}") String distanciaServiceUrl) {
    this.distanciaServiceUrl = distanciaServiceUrl;
    this.restTemplate = restTemplate;
  }

  public void novoRestauranteAprovado(Restaurante restaurante) { // modificado
    // código omitido ...
  }

  // restante do código ...

}
```

Execute a classe `DistanciaRestClientWiremockTest` com o JUnit 4.

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
2019-07-03 17:41:37.689  INFO [monolito,,,] 32404 --- [tp1306763722-36] WireMock:
Request received:
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

## Exercício: Contract Test para comunicação síncrona

1. Abra um Terminal e vá até a branch `cap12-contrato-cliente-servidor` do projeto do serviço de distância:

  ```sh
  cd ~/Desktop/fj33-eats-distancia-service
  git checkout -f cap12-contrato-cliente-servidor
  ```

  Então, faça o build do serviço de distância, rode o Contract Test no próprio serviço, gere o JAR com os stubs do contrato e instale no repositório local do Maven. Basta executar o comando:

  ```sh
  mvn clean install
  ```

  Aguarde a execução do build. As mensagens finais devem conter:

  ```txt
  [INFO] Results:
  [INFO]
  [INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
  [INFO]
  [INFO]
  ...
  [INFO] Installing /home/<USUARIO-DO-CURSO>/Desktop/fj33-eats-distancia-service/target/eats-distancia-service-0.0.1-SNAPSHOT.jar to /home/<USUARIO-DO-CURSO>/.m2/repository/br/com/caelum/eats-distancia-service/0.0.1-SNAPSHOT/eats-distancia-service-0.0.1-SNAPSHOT.jar
  [INFO] Installing /home/<USUARIO-DO-CURSO>/Desktop/fj33-eats-distancia-service/pom.xml to /home/<USUARIO-DO-CURSO>/.m2/repository/br/com/caelum/eats-distancia-service/0.0.1-SNAPSHOT/eats-distancia-service-0.0.1-SNAPSHOT.pom
  [INFO] Installing /home/<USUARIO-DO-CURSO>/Desktop/fj33-eats-distancia-service/target/eats-distancia-service-0.0.1-SNAPSHOT-stubs.jar to /home/<USUARIO-DO-CURSO>/.m2/repository/br/com/caelum/eats-distancia-service/0.0.1-SNAPSHOT/eats-distancia-service-0.0.1-SNAPSHOT-stubs.jar
  [INFO] ------------------------------------------------------------------------
  [INFO] BUILD SUCCESS
  [INFO] ------------------------------------------------------------------------
  [INFO] Total time: 02:45 min
  [INFO] Finished at: 2019-07-03T16:45:17-03:00
  [INFO] ------------------------------------------------------------------------
  ```

  Observe, pelas mensagens anteriores, que o JAR com os stubs foi instalado no diretório `.m2`, o repositório local Maven, do usuário do curso.

2. No projeto do monólito modular, faça checkout da branch `cap12-contrato-cliente-servidor`:

  ```sh
  cd ~/Desktop/fj33-eats-monolito-modular
  git checkout -f cap12-contrato-cliente-servidor
  ```

  Faça o refresh do projeto no Eclipse.

  Clique com o botão direito na classe `DistanciaRestClientWiremockTest`, do módulo `eats-application` do monólito, e, então, em _Run As... > Run Configurations..._. Clique com o botão direito em _JUnit_ e, a seguir, em _New Configuration_. Em _Test runner_, escolha o JUnit 4. Então, clique em _Run_.

  Aguarde a execução dos testes. Sucesso!

## Exercício: definindo um contrato no publisher

1. Adicione, ao `pom.xml` do serviço de pagamentos, as dependências ao starter do Spring Cloud Contract Verifier e à biblioteca de suporte a testes do Spring Cloud Stream:

  ####### fj33-eats-pagamento-service/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-verifier</artifactId>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-test-support</artifactId>
    <scope>test</scope>
  </dependency>
  ```

  Adicione também o plugin Maven do Spring Cloud Contract, configurando `br.com.caelum.eats.pagamento.base` como pacote das classes base a serem usadas nos testes gerados a partir dos contratos.

  ####### fj33-eats-pagamento-service/pom.xml

  ```xml
  <plugin>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-contract-maven-plugin</artifactId>
    <version>2.1.2.RELEASE</version>
    <extensions>true</extensions>
    <configuration>
      <packageWithBaseClasses>br.com.caelum.eats.pagamento.base</packageWithBaseClasses>
    </configuration>
  </plugin>
  ```

2. Dentro do Eclipse, clique com o botão direito no projeto `eats-pagamento-service`, acessando _New > Folder..._ e definindo o caminho `src/test/resources/contracts/pagamentos/confirmados` em _Folder name_.

  Crie o arquivo `deveAdicionarNovoRestaurante.groovy` nesse diretório, definindo o contrato por meio da DSL Groovy:

  ####### fj33-eats-pagamento-service/src/test/resources/contracts/pagamentos/confirmados/deveNotificarPagamentosConfirmados.groovy

  ```groovy
  import org.springframework.cloud.contract.spec.Contract
  Contract.make {
    description "deve notificar pagamentos confirmados"
    label 'pagamento_confirmado'
    input {
      triggeredBy('novoPagamentoConfirmado()')
    }
    outputMessage {
      sentTo 'pagamentosConfirmados'
      body([
        pagamentoId: 2,
        pedidoId: 3
      ])
      headers {
        messagingContentType(applicationJson())
      }
    }
  }
  ```

  Definimos `pagamento_confirmado` como `label`, que será usado nos testes do subscriber. Em `input`, invocamos o método `novoPagamentoConfirmado` da classe base. Já em `outputMessage`, definimos `pagamentosConfirmados` como _destination_ esperado, o corpo da mensagem e o _Content Type_ nos cabeçalhos.

3. No pacote `br.com.caelum.eats.pagamento.base`,  do source folder `src/test/java`, crie a classe `PagamentosConfirmadosBase`. Anote essa classe com `@AutoConfigureMessageVerifier`, além das anotações de testes do Spring Boot. Na anotação `@SpringBootTest`, configure o `webEnvironment` para `NONE`.

  Peça ao Spring para injetar uma instância da classe `NotificadorPagamentoConfirmado`.

  Defina um método `novoPagamentoConfirmado`, que usa a instância injetada para chamar o método `notificaPagamentoConfirmado` passando como parâmetro um `Pagamento` com dados compatíveis com o contrato definido anteriormente.

  _Observação: o nome da classe `PagamentosConfirmadosBase` usa como prefixo o diretório de nosso contrato (`pagamentos/confirmados`) com o sufixo `Base`._

  ####### fj33-eats-pagamento-service/src/test/java/br/com/caelum/eats/pagamento/base/PagamentosConfirmadosBase.java

  ```java
  @RunWith(SpringRunner.class)
  @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.NONE)
  @AutoConfigureMessageVerifier
  public class PagamentosConfirmadosBase {

    @Autowired
    private NotificadorPagamentoConfirmado notificador;

    public void novoPagamentoConfirmado() {
      Pagamento pagamento = new Pagamento();
      pagamento.setId(2L);
      pagamento.setPedidoId(3L);
      notificador.notificaPagamentoConfirmado(pagamento);
    }

  }
  ```

  Confira os imports:

  ```java
  import org.junit.runner.RunWith;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.boot.test.context.SpringBootTest;
  import org.springframework.cloud.contract.verifier.messaging.boot.AutoConfigureMessageVerifier;
  import org.springframework.test.context.junit4.SpringRunner;

  import br.com.caelum.eats.pagamento.NotificadorPagamentoConfirmado;
  ```

4. Deve acontecer um erro de compilação no uso de `Pagamento` na classe `PagamentosConfirmadosBase`.

  Corrija esse erro, fazendo com que a classe `Pagamento` seja pública:

  ####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/Pagamento.java

  ```java
  public class Pagamento { // modificado

    // código omitido ...

  }
  ```

5. Abra um Terminal e vá ao diretório do serviço de pagamentos. Execute os comandos:

  ```sh
  cd ~/Desktop/eats-pagamento-service
  mvn clean install
  ```

  Depois da execução do build, deve aparecer algo como:

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
  [INFO] Total time: 50.94 s
  [INFO] Finished at: 2019-07-03T16:45:17-03:00
  [INFO] ------------------------------------------------------------------------
  ```

  O Spring Cloud Contract deve ter gerado a seguinte classe: eats-pagamento-service/target/generated-test-sources/contracts/br/com/caelum/eats/pagamento/base/pagamentos/ConfirmadosTest.java

  O intuito dessa classe é verificar que o contrato é seguido pelo próprio publisher.

  Com o sucesso dos testes, é  gerado o arquivo `eats-pagamento-service-0.0.1-SNAPSHOT-stubs.jar` em `target`, contendo o contrato `deveAdicionarNovoRestaurante.groovy`. Esse JAR será usado na verificação do contrato do lado do subscriber.

## Exercício: verificando o contrato no subscriber

1. Adicione, no `pom.xml` do serviço de nota fiscal, as dependências ao starter do Spring Cloud Contract Stub Runner e à biblioteca de suporte a testes do Spring Cloud Stream:

  ####### fj33-eats-nota-fiscal-service/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-contract-stub-runner</artifactId>
    <scope>test</scope>
  </dependency>
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-stream-test-support</artifactId>
    <scope>test</scope>
  </dependency>
  ```

2. Crie a classe `ProcessadorDePagamentosTest`, dentro do pacote `br.com.caelum.notafiscal` do source folder `src/test/java`.

  Anote-a com as anotações de teste do Spring Boot, definindo em `@SpringBootTest` o valor `NONE` no atributo `webEnvironment`.

  Adicione também a anotação `@AutoConfigureStubRunner`. No atributo `ids`, aponte para o artefato que conterá os stubs, definindo `br.com.caelum` como `groupId` e `eats-pagamento-service` como `artifactId`. No atributo `stubsMode`, use o modo `LOCAL`.

  Faça com que o Spring injete uma instância de `StubTrigger`.

  Injete também mocks para `GeradorDeNotaFiscal` e `PedidoRestClient` e um spy para `ProcessadorDePagamentos`, a classe que recebe as mensagens.

  Defina um método `deveProcessarPagamentoConfirmado`, anotando-o com `@Test`.
  
  No método de teste, use as instâncias de `GeradorDeNotaFiscal` e `PedidoRestClient` como stubs, registrando respostas as chamadas dos métodos `detalhaPorId` e `geraNotaPara`, respectivamente. O valor dos parâmetros deve considerar os valores definidos no contrato.

  Dispare a mensagem usando o label `pagamento_confirmado` no método `trigger` do `StubTrigger`.

  Verifique a chamada ao `ProcessadorDePagamentos`, usando um `ArgumentCaptor` do Mockito. Os valores dos parâmetros devem corresponder aos definidos no contrato.

  ####### fj33-eats-nota-fiscal-service/src/test/java/br/com/caelum/notafiscal/ProcessadorDePagamentosTest.java

  ```java
  @RunWith(SpringRunner.class)
  @SpringBootTest(webEnvironment = WebEnvironment.NONE)
  @AutoConfigureStubRunner(ids = "br.com.caelum:eats-pagamento-service", stubsMode = StubRunnerProperties.StubsMode.LOCAL)
  public class ProcessadorDePagamentosTest {

    @Autowired
    private StubTrigger stubTrigger;

    @MockBean
    private GeradorDeNotaFiscal notaFiscal;

    @MockBean
    private PedidoRestClient pedidos;

    @SpyBean
    private ProcessadorDePagamentos processadorPagamentos;

    @Test
    public void deveProcessarPagamentoConfirmado() {

      PedidoDto pedidoDto = new PedidoDto();
      Mockito.when(pedidos.detalhaPorId(3L)).thenReturn(pedidoDto);
      Mockito.when(notaFiscal.geraNotaPara(pedidoDto)).thenReturn("<xml>...</xml>");

      stubTrigger.trigger("pagamento_confirmado");

      ArgumentCaptor<PagamentoConfirmado> pagamentoArg = ArgumentCaptor.forClass(PagamentoConfirmado.class);

      Mockito.verify(processadorPagamentos).processaPagamento(pagamentoArg.capture());

      PagamentoConfirmado pagamentoConfirmado = pagamentoArg.getValue();
      Assert.assertEquals(2L, pagamentoConfirmado.getPagamentoId().longValue());
      Assert.assertEquals(3L, pagamentoConfirmado.getPedidoId().longValue());
    }

  }
  ```

3. No Eclipse, clique com o botão direito na classe `ProcessadorDePagamentosTest` e, então, em _Run As... > JUnit Test_. Use o JUnit 4.

  Aguarde a execução dos testes. Sucesso!
