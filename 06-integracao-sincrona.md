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
  @SpringBootApplication
  public class EatsPagamentoServiceApplication {

    // código omitido ...

  }
  ```

  O import correto é o seguinte:

  ```java
  import org.springframework.cloud.openfeign.EnableFeignClients;
  ```

4. Defina, no pacote `br.com.caelum.eats.pagamento` de `eats-pagamento-service`, uma interface `PedidoRestClient` com um método `avisaQueFoiPago`, anotados da seguinte maneira:

  ####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PedidoRestClient.java

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

  ####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PagamentoController.java

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

## Exercício opcional: Spring HATEOAS e HAL

1. Adicione o Spring HATEOAS como dependência no `pom.xml` de `eats-pagamento-service`:

  ####### eats-pagamento-service/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
  </dependency>
  ```

2. Nos métodos de `PagamentoController`, retorne um `Resource` com uma lista de `Link` do Spring HATEOAS.

  - Em todos os métodos, defina um _link relation_ `self` que aponta para o próprio recurso, através da URL do método `detalha`
  - Nos métodos `detalha` e `cria`, defina _link relations_ `confirma` e `cancela`, apontando para as URLS associadas aos respectivos métodos de `PagamentoController`.

  Para criar os _links_, utilize os métodos estáticos `methodOn` e `linkTo` de `ControllerLinkBuilder`.

  O código de `PagamentoController` ficará semelhante a:

  ####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PagamentoController.java

  ```java
  @RestController
  @RequestMapping("/pagamentos")
  @AllArgsConstructor
  class PagamentoController {

    private PagamentoRepository pagamentoRepo;
    private PedidoRestClient pedidoClient;

    @GetMapping("/{id}")
    public Resource<PagamentoDto> detalha(@PathVariable Long id) {
      Pagamento pagamento = pagamentoRepo.findById(id).orElseThrow(() -> new ResourceNotFoundException());

      List<Link> links = new ArrayList<>();

      Link self = linkTo(methodOn(PagamentoController.class).detalha(id)).withSelfRel();
      links.add(self);

      if (Pagamento.Status.CRIADO.equals(pagamento.getStatus())) {
        Link confirma = linkTo(methodOn(PagamentoController.class).confirma(id)).withRel("confirma");
        links.add(confirma);

        Link cancela = linkTo(methodOn(PagamentoController.class).cancela(id)).withRel("cancela");
        links.add(cancela);
      }

      PagamentoDto dto = new PagamentoDto(pagamento);
      Resource<PagamentoDto> resource = new Resource<PagamentoDto>(dto, links);

      return resource;
    }

    @PostMapping
    public ResponseEntity<Resource<PagamentoDto>> cria(@RequestBody Pagamento pagamento,
        UriComponentsBuilder uriBuilder) {
      pagamento.setStatus(Pagamento.Status.CRIADO);
      Pagamento salvo = pagamentoRepo.save(pagamento);
      URI path = uriBuilder.path("/pagamentos/{id}").buildAndExpand(salvo.getId()).toUri();
      PagamentoDto dto = new PagamentoDto(salvo);

      Long id = salvo.getId();

      List<Link> links = new ArrayList<>();

      Link self = linkTo(methodOn(PagamentoController.class).detalha(id)).withSelfRel();
      links.add(self);

      Link confirma = linkTo(methodOn(PagamentoController.class).confirma(id)).withRel("confirma");
      links.add(confirma);

      Link cancela = linkTo(methodOn(PagamentoController.class).cancela(id)).withRel("cancela");
      links.add(cancela);

      Resource<PagamentoDto> resource = new Resource<PagamentoDto>(dto, links);
      return ResponseEntity.created(path).body(resource);
    }

    @PutMapping("/{id}")
    public Resource<PagamentoDto> confirma(@PathVariable Long id) {
      Pagamento pagamento = pagamentoRepo.findById(id).orElseThrow(() -> new ResourceNotFoundException());
      pagamento.setStatus(Pagamento.Status.CONFIRMADO);
      pagamentoRepo.save(pagamento);

      Long pedidoId = pagamento.getPedidoId();
      pedidoClient.avisaQueFoiPago(pedidoId);

      List<Link> links = new ArrayList<>();

      Link self = linkTo(methodOn(PagamentoController.class).detalha(id)).withSelfRel();
      links.add(self);

      PagamentoDto dto = new PagamentoDto(pagamento);
      Resource<PagamentoDto> resource = new Resource<PagamentoDto>(dto, links);

      return resource;
    }

    @DeleteMapping("/{id}")
    public Resource<PagamentoDto> cancela(@PathVariable Long id) {
      Pagamento pagamento = pagamentoRepo.findById(id).orElseThrow(() -> new ResourceNotFoundException());
      pagamento.setStatus(Pagamento.Status.CANCELADO);
      pagamentoRepo.save(pagamento);

      List<Link> links = new ArrayList<>();

      Link self = linkTo(methodOn(PagamentoController.class).detalha(id)).withSelfRel();
      links.add(self);

      PagamentoDto dto = new PagamentoDto(pagamento);
      Resource<PagamentoDto> resource = new Resource<PagamentoDto>(dto, links);

      return resource;
    }

  }
  ```

3. Reinicie o serviço de pagamentos e obtenha o pagamento de um `id` já cadastrado:

  ```sh
   curl -i http://localhost:8081/pagamentos/1
  ```

  A resposta será algo como:

  ```text
  HTTP/1.1 200 
  Content-Type: application/hal+json;charset=UTF-8
  Transfer-Encoding: chunked
  Date: Tue, 28 May 2019 19:04:43 GMT
  ```

  <!--  -->

  ```json
  {
    "id":1,
    "valor":51.80,
    "nome":"ANDERSON DA SILVA",
    "numero":"1111 2222 3333 4444",
    "expiracao":"2022-07",
    "codigo":"123",
    "status":"CRIADO",
    "formaDePagamentoId":2,
    "pedidoId":1,
    "_links":{
      "self":{
        "href":"http://localhost:8081/pagamentos/1"
      },
      "confirma":{
        "href":"http://localhost:8081/pagamentos/1"
      },
      "cancela":{
        "href":"http://localhost:8081/pagamentos/1"
      }
    }
  }
  ```
  
  Teste também a criação, confirmação e cancelamento de novos pagamentos.

4. Altere o código do front-end para usar os _link relations_ apropriados ao confirmar ou cancelar um pagamento:

  ####### fj33-eats-ui/src/app/services/pagamento.service.ts

  ```typescript
  confirma(pagamento): Observable<any> {
    t̶h̶i̶s̶.̶a̶j̶u̶s̶t̶a̶I̶d̶s̶(̶p̶a̶g̶a̶m̶e̶n̶t̶o̶)̶;̶

    const url = pagamento._links.confirma.href; // adicionado

    r̶e̶t̶u̶r̶n̶ ̶t̶h̶i̶s̶.̶h̶t̶t̶p̶.̶p̶u̶t̶(̶`̶$̶{̶t̶h̶i̶s̶.̶A̶P̶I̶}̶/̶$̶{̶p̶a̶g̶a̶m̶e̶n̶t̶o̶.̶i̶d̶}̶`̶,̶ ̶n̶u̶l̶l̶)̶;̶
    return this.http.put(url, null); // modificado
  }

  cancela(pagamento): Observable<any> {
    t̶h̶i̶s̶.̶a̶j̶u̶s̶t̶a̶I̶d̶s̶(̶p̶a̶g̶a̶m̶e̶n̶t̶o̶)̶;̶

    const url = pagamento._links.cancela.href; // adicionado

    r̶e̶t̶u̶r̶n̶ ̶t̶h̶i̶s̶.̶h̶t̶t̶p̶.̶d̶e̶l̶e̶t̶e̶(̶`̶$̶{̶t̶h̶i̶s̶.̶A̶P̶I̶}̶/̶$̶{̶p̶a̶g̶a̶m̶e̶n̶t̶o̶.̶i̶d̶}̶`̶)̶;̶
    return this.http.delete(url); // modificado
  }
  ```

  _Observação: o método auxiliar `ajustaIds` não é mais necessário ao confirmar e cancelar um pagamento, já que o `id` do pagamento não é mais usado para montar a URL. Porém, o método ainda é usado ao criar um pagamento._

5. Faça um novo pedido e efetue um pagamento. Deve continuar funcionando!

## Exercício opcional: Estendendo o Spring HATEOAS

### Objetivo

Faça com que o método HTTP seja incluído como informações dos links retornados pelo Spring HATEOAS.

### Passo a passo

1. Crie uma classe `LinkWithMethod` que estende o `Link` do Spring HATEOAS e define um atributo adicional chamado `method`, que armazenará o método HTTP dos links. Defina um construtor que recebe um `Link` e uma `String` com o método HTTP:

  ####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/LinkWithMethod.java

  ```java
  @Getter
  public class LinkWithMethod extends Link {

    private static final long serialVersionUID = 1L;

    private String method;

    public LinkWithMethod(Link link, String method) {
      super(link.getHref(), link.getRel());
      this.method = method;
    }
  }
  ```

  Os imports são os seguintes:

  ```java
  import org.springframework.hateoas.Link;
  import lombok.Getter;
  ```

2. Na classe `PagamentoController`, adicione um `LinkWithMethod` na lista para os links de confirmação e cancelamento, passando o método HTTP adequado.

  Use o trecho abaixo nos métodos `detalha` e `cria` de `PagamentoController`:

  ####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PagamentoController.java

  ```java
  Link confirma = linkTo(methodOn(PagamentoController.class).confirma(id)).withRel("confirma");
  l̶i̶n̶k̶s̶.̶a̶d̶d̶(̶c̶o̶n̶f̶i̶r̶m̶a̶)̶;̶
  links.add(new LinkWithMethod(confirma, "PUT")); // modificado

  Link cancela = linkTo(methodOn(PagamentoController.class).cancela(id)).withRel("cancela");
  l̶i̶n̶k̶s̶.̶a̶d̶d̶(̶c̶a̶n̶c̶e̶l̶a̶)̶;̶
  links.add(new LinkWithMethod(cancela, "DELETE")); // modificado
  ```

3. Ajuste o código do front-end para usar o `method` de cada _link relation_:

  ####### fj33-eats-ui/src/app/services/pagamento.service.ts

  ```typescript
  confirma(pagamento): Observable<any> {
    const url = pagamento._links.confirma.href;

    r̶e̶t̶u̶r̶n̶ ̶t̶h̶i̶s̶.̶h̶t̶t̶p̶.̶p̶u̶t̶(̶u̶r̶l̶,̶ ̶n̶u̶l̶l̶)̶;̶

    const method = pagamento._links.confirma.method;
    return this.http.request(method, url);
  }

  cancela(pagamento): Observable<any> {
    const url = pagamento._links.cancela.href;

    r̶e̶t̶u̶r̶n̶ ̶t̶h̶i̶s̶.̶h̶t̶t̶p̶.̶d̶e̶l̶e̶t̶e̶(̶u̶r̶l̶)̶;̶

    const method = pagamento._links.cancela.method;
    return this.http.request(method, url);
  }
  ```

4. (desafio) Modifique o `PagamentoController` para usar HAL-FORMS, disponível nas últimas versões do Spring HATEOAS.

<!--@note

---------------
Alexandre (BSB)
---------------

HAL-FORMS é uma das ideias de especificação de Hypermedia de Mike Amundsen, autor de diversos livros sobre REST na editor O'Reilly (aquela dos bichos na capa).

Usei o HAL-FORMS na versão milestone 2.2.0.M2 do Spring Boot no commit abaixo:

https://github.com/alexandreaquiles/eats/commit/f8ef33b88cd3d96c62627a13b4e8470c9f09ada0#diff-5f414af558500eda821060272d84b8d8

-->
