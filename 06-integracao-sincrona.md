# Integração síncrona

## Exercício: cliente REST com RestTemplate do Spring

### Objetivo

Faça com que o monólito avise ao serviço de distância que novos restaurantes foram aprovados e que houve atualização de dados relevantes.

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
      return repo.insert(restaurante);
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
      restTemplate.postForObject(url, restauranteParaDistancia, RestauranteParaServicoDeDistancia.class);
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

## Exercício: cliente declarativo com Feign
