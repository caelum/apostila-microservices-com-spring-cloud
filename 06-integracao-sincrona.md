# Integração síncrona (e RESTful)

## Exercício: cliente REST com RestTemplate do Spring

### Objetivo

Faça com que o monólito avise ao serviço de distância que novos restaurantes foram aprovados e que houve atualização de dados relevantes. A implementação deve usar `RestTemplate`.

### Passo a passo

1. No `eats-distancia-service`, crie um Controller chamado `RestaurantesController` no pacote `br.com.caelum.eats.distancia` com um método que insere um novo restaurante e outro que atualiza um restaurante existente. Defina mensagens de log em cada método.

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/RestaurantesController.java

  ```java
  @RestController
  @AllArgsConstructor
  @Slf4j
  public class RestaurantesController {

    private RestauranteMongoRepository repo;

    @PostMapping("/restaurantes")
    public RestauranteMongo adiciona(@RequestBody RestauranteMongo restaurante) {
      log.info("Insere novo restaurante: " + restaurante);
      RestauranteMongo salvo = repo.insert(restaurante);
      UriComponents uriComponents = uriBuilder.path("/restaurantes/{id}").buildAndExpand(salvo.getId());
      URI uri = uriComponents.toUri();
      return ResponseEntity.created(uri).contentType(MediaType.APPLICATION_JSON).body(salvo);
    }

    @PutMapping("/restaurantes/{id}")
    public RestauranteMongo atualiza(@PathVariable Long id, @RequestBody RestauranteMongo restaurante) {
      if (!repo.existsById(id)) {
        throw new ResourceNotFoundException();
      }
      log.info("Atualiza restaurante: " + restaurante);
      return repo.save(restaurante);
    }

  }
  ```

  Certifique-se que os imports estão corretos:

  ```java
  import org.springframework.web.bind.annotation.PathVariable;
  import org.springframework.web.bind.annotation.PostMapping;
  import org.springframework.web.bind.annotation.PutMapping;
  import org.springframework.web.bind.annotation.RequestBody;
  import org.springframework.web.bind.annotation.RestController;

  import br.com.caelum.eats.distancia.mongo.RestauranteMongo;
  import br.com.caelum.eats.distancia.mongo.RestauranteMongoRepository;
  import br.com.caelum.eats.exception.ResourceNotFoundException;
  import lombok.AllArgsConstructor;
  import lombok.extern.slf4j.Slf4j;
  ```

2. No `application.properties` do módulo `eats-application` do monólito, crie uma propriedade `configuracao.distancia.service.url` para indicar a URL do serviço de distância:

  ####### eats-monolito-modular/eats-application/src/main/resources/application.properties

  ```properties
  configuracao.distancia.service.url=http://localhost:8082
  ```

3. No módulo `eats-common` do monólito, crie uma classe `RestClientConfig` no pacote `br.com.caelum.eats`, que fornece um `RestTemplate` do Spring:

  ####### eats-monolito-modular/eats-common/src/main/java/br/com/caelum/eats/RestClientConfig.java

  ```java
  @Configuration
  public class RestClientConfig {

    @Bean
    public RestTemplate restTemplate() {
      return new RestTemplate();
    }

  }
  ```

  Faça os imports adequados:

  ```java
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.web.client.RestTemplate;
  ```

4. No módulo `eats-restaurante` do monólito, crie uma classe `RestauranteParaServicoDeDistancia` no pacote `br.com.caelum.eats.restaurante` que contém apenas as informações adequadas para o serviço de distância. Crie um construtor que recebe um `Restaurante` e popula os dados necessários:

  ####### eats-monolito-modular/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/RestauranteParaServicoDeDistancia.java

  ```java
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  class RestauranteParaServicoDeDistancia {

    private Long id;
    private String cep;
    private Long tipoDeCozinhaId;

    RestauranteParaServicoDeDistancia(Restaurante restaurante){
      this(restaurante.getId(), restaurante.getCep(), restaurante.getTipoDeCozinha().getId());
    }

  }
  ```

  Observação: a anotação `@Data` do Lombok define um Java Bean com getters, setters para campos mutáveis, `equals` e `hashcode` e `toString`.

5. Crie uma classe `DistanciaRestClient` no pacote `br.com.caelum.eats.restaurante` do módulo `eats-restaurante` do monólito. Defina como dependências um `RestTemplate` e uma `String` para armazenar a propriedade `configuracao.distancia.service.url`.

  Anote a classe com `@Service` do Spring.

  Defina métodos que chamam o serviço de distância para:

  - inserir um novo restaurante aprovado, enviando um POST para `/restaurantes` com o `RestauranteParaServicoDeDistancia` como corpo da requisição
  - atualizar um restaurante já existente, enviando um PUT para `/restaurantes/{id}`, com o `id` adequado e um `RestauranteParaServicoDeDistancia` no corpo da requisição

  ####### eats-monolito-modular/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/DistanciaRestClient.java

  ```java
  @Service
  public class DistanciaRestClient {

    private String distanciaServiceUrl;
    private RestTemplate restTemplate;

    public DistanciaRestClient(RestTemplate restTemplate,
                                        @Value("${configuracao.distancia.service.url}") String distanciaServiceUrl) {
      this.distanciaServiceUrl = distanciaServiceUrl;
      this.restTemplate = restTemplate;
    }

    public void novoRestauranteAprovado(Restaurante restaurante) {
      RestauranteParaServicoDeDistancia restauranteParaDistancia = new RestauranteParaServicoDeDistancia(restaurante);
      String url = distanciaServiceUrl+"/restaurantes";
      ResponseEntity<RestauranteParaServicoDeDistancia> responseEntity =
          restTemplate.postForEntity(url, restauranteParaDistancia, RestauranteParaServicoDeDistancia.class);
      HttpStatus statusCode = responseEntity.getStatusCode();
      if (!HttpStatus.CREATED.equals(statusCode)) {
        throw new RuntimeException("Status diferente do esperado: " + statusCode);
      }
    }

    public void restauranteAtualizado(Restaurante restaurante) {
      RestauranteParaServicoDeDistancia restauranteParaDistancia = new RestauranteParaServicoDeDistancia(restaurante);
      String url = distanciaServiceUrl+"/restaurantes/" + restaurante.getId();
      restTemplate.put(url, restauranteParaDistancia, RestauranteParaServicoDeDistancia.class);
    }

  }
  ```

  Os imports corretos são:

  ```java
  import org.springframework.beans.factory.annotation.Value;
  import org.springframework.stereotype.Service;
  import org.springframework.web.client.RestTemplate;
  ```

6. Altere a classe `RestauranteController` do módulo `eats-restaurante` do monólito para que:

  - tenha um `DistanciaRestClient` como dependência
  - no caso de aprovação de um restaurante, invoque o método `novoRestauranteAprovado` de `DistanciaRestClient`
  - no caso de atualização do CEP ou tipo de cozinha de um restaurante já aprovado, invoque o método `restauranteAtualizado` de `DistanciaRestClient`

  ####### eats-monolito-modular/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/RestauranteController.java

  ```java
  // anotações ...
  class RestauranteController {

    private RestauranteRepository restauranteRepo;
    private CardapioRepository cardapioRepo;

    private DistanciaRestClient distanciaRestClient; // adicionado

    // métodos omitidos ...

    @PutMapping("/parceiros/restaurantes/{id}")
    public Restaurante atualiza(@RequestBody Restaurante restaurante) {
      Restaurante doBD = restauranteRepo.getOne(restaurante.getId());
      restaurante.setUser(doBD.getUser());
      restaurante.setAprovado(doBD.getAprovado());

      // adicionado
      if (doBD.getAprovado() &&
              (cepDiferente(restaurante, doBD) || tipoDeCozinhaDiferente(restaurante, doBD))) {
        distanciaRestClient.restauranteAtualizado(restaurante);
      }

      return restauranteRepo.save(restaurante);
    }

    // adicionado
    private boolean tipoDeCozinhaDiferente(Restaurante restaurante, Restaurante doBD) {
      return !doBD.getTipoDeCozinha().getId().equals(restaurante.getTipoDeCozinha().getId());
    }

    // adicionado
    private boolean cepDiferente(Restaurante restaurante, Restaurante doBD) {
      return !doBD.getCep().equals(restaurante.getCep());
    }

    // método omitido ...

    @Transactional
    @PatchMapping("/admin/restaurantes/{id}")
    public void aprova(@PathVariable("id") Long id) {
      restauranteRepo.aprovaPorId(id);

      // adicionado
      Restaurante restaurante = restauranteRepo.getOne(id);
      distanciaRestClient.novoRestauranteAprovado(restaurante);
    }

  }
  ```

7. Teste a aprovação de novos restaurantes e a mudança de CEP ou tipo de cozinha de um restaurante existente. Verifique nos logs que o serviço de distância foi chamado.

8 (opcional) Será que os métodos auxiliares `tipoDeCozinhaDiferente` e `cepDiferente` deveriam ficar em `RestauranteController` mesmo?

## Exercício: cliente REST declarativo com Feign

### Objetivo

Implemente, usando Feign, uma maneira do serviço de pagamento avisar ao monólito que um pedido foi pago, após a confirmação do pagamento.

### Passo a passo

1. Adicione ao `PedidoController`, do módulo `eats-pedido` do monólito, um método que muda o status do pedido para _PAGO_:

  ####### eats-monolito-modular/eats-pedido/src/main/java/br/com/caelum/eats/pedido/PedidoController.java

  ```java
  @PutMapping("/pedidos/{id}/pago")
  public void pago(@PathVariable("id") Long id) {
    Pedido pedido = repo.porIdComItens(id);
    if (pedido == null) {
      throw new ResourceNotFoundException();
    }
    pedido.setStatus(Pedido.Status.PAGO);
    repo.atualizaStatus(Pedido.Status.PAGO, pedido);
  }
  ```

2. No arquivo `application.properties` de `eats-pagamento-service`, adicione uma propriedade `configuracao.pedido.service.url` que contém a URL do monólito:

  ####### eats-pagamento-service/src/main/resources/application.properties

  ```properties
  configuracao.pedido.service.url=http://localhost:8080
  ```

3. No `pom.xml` de `eats-pagamento-service`, adicione uma dependência ao _Spring Cloud_ na versão `Greenwich.RELEASE`, em `dependencyManagement`:

  ####### eats-pagamento-service/pom.xml

  ```xml
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>Greenwich.RELEASE</version>
        <type>pom</type>
        <scope>import</scope>
      </dependency>
    </dependencies>
  </dependencyManagement>
  ```

  Feito isso, adicione o _starter_ do OpenFeign como dependência:
  
  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
  </dependency>
  ```

4. Anote a classe `EatsPagamentoServiceApplication` com `@EnableFeignClients` para habilitar o Feign:

  ####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/EatsPagamentoServiceApplication.java

  ```java
  @EnableFeignClients // adicionado
  @SpringBootApplication	@SpringBootApplication
  public class EatsPagamentoServiceApplication {

    // código omitido ...

  }
  ```

  O import correto é o seguinte:

  ```java
  import org.springframework.cloud.openfeign.EnableFeignClients;
  ```

4. Defina, no pacote `br.com.caelum.eats.pagamento` de `eats-pagamento-service`, uma interface `PedidoRestClient` com um método `avisaQueFoiPago`, anotados da seguinte maneira:

  ```java
  @FeignClient(url="${configuracao.pedido.service.url}", name="pedido")
  public interface PedidoRestClient {

    @PutMapping("/pedidos/{pedidoId}/pago")
    public void avisaQueFoiPago(@PathVariable("pedidoId") Long pedidoId);

  }
  ```

  Ajuste os imports:

  ```java
  import org.springframework.cloud.openfeign.FeignClient;
  import org.springframework.web.bind.annotation.PathVariable;
  import org.springframework.web.bind.annotation.PutMapping;
  ```

5. Em `PagamentoController`, do serviço de pagamento, defina um `PedidoRestClient` como atributo e use o método `avisaQueFoiPago` passando o id do pedido:

  ```java
  // anotações ...
  public class PagamentoController {

    private PagamentoRepository pagamentoRepo;
    private PedidoRestClient pedidoClient; // adicionado

    // código omitido ...

    @PutMapping("/{id}")
    public PagamentoDto confirma(@PathVariable Long id) {
      Pagamento pagamento = pagamentoRepo.findById(id).orElseThrow(() -> new ResourceNotFoundException());
      pagamento.setStatus(Pagamento.Status.CONFIRMADO);
      pagamentoRepo.save(pagamento);

      // adicionado
      Long pedidoId = pagamento.getPedidoId();
      pedidoClient.avisaQueFoiPago(pedidoId);

      return new PagamentoDto(pagamento);
    }

    // restante do código ...

  }
  ```

6. Certifique-se que o serviço de pagamento foi reiniciado e que os demais serviços e o front-end estão no ar. Faça um novo pedido, realizando o pagamento. Veja que o status do pedido fica como _PAGO_.

## Exercício opcional: Spring HATEOAS

1. 
