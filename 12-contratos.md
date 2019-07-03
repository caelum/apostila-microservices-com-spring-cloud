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

