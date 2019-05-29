## Exercício: implementando um API Gateway com Zuul

1. Pelo navegador, abra `https://start.spring.io/`.
  Em _Project_, mantenha _Maven Project_.
  Em _Language_, mantenha _Java_.
  Em _Spring Boot_, mantenha a versão padrão.
  No trecho de _Project Metadata_, defina:

  - `br.com.caelum` em _Group_
  - `api-gateway` em _Artifact_

  Mantenha os valores em _More options_.
 
  Mantenha o _Packaging_ como `Jar`.
  Mantenha a _Java Version_ em `8`.

  Em _Dependencies_, adicione:

  - Zuul
  - DevTools

  Clique em _Generate Project_.
2. Extraia o `api-gateway.zip` e copie a pasta para seu Desktop.
3. No Eclipse, no workspace de microservices, importe o projeto `api-gateway`, usando o menu _File > Import > Existing Maven Projects_.
4. Adicione a anotação `@EnableZuulProxy` à classe `ApiGatewayApplication`:

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/ApiGatewayApplication.java

  ```java
  @EnableZuulProxy
  @SpringBootApplication
  public class ApiGatewayApplication {

    public static void main(String[] args) {
      SpringApplication.run(ApiGatewayApplication.class, args);
    }

  }
  ```

  Não deixe de adicionar o import:

  ```java
  import org.springframework.cloud.netflix.zuul.EnableZuulProxy;
  ```

5. No arquivo `src/main/resources/application.properties`:

  - modifique a porta para 9999
  - desabilite o Eureka, por enquanto (o abordaremos mais adiante)
  - para as URLs do serviço de pagamento, parecidas com `http://localhost:9999/pagamentos/algum-recurso`, redirecione para `http://localhost:8081`. Para manter o prefixo `/pagamentos`, desabilite a propriedade `stripPrefix`.
  - para as URLs do serviço de distância, algo como `http://localhost:9999/distancia/algum-recurso`, redirecione para `http://localhost:8082`. O prefixo `/distancia` será removido, já que esse é o comportamento padrão.
  - para as demais URLs, redirecione para `http://localhost:8080`, o monólito.

  O arquivo ficará semelhante a:

  ####### api-gateway/src/main/resources/application.properties

  ```properties
  server.port = 9999

  ribbon.eureka.enabled=false

  zuul.routes.pagamentos.url=http://localhost:8081
  zuul.routes.pagamentos.stripPrefix=false

  zuul.routes.distancia.url=http://localhost:8082

  zuul.routes.monolito.path=/**
  zuul.routes.monolito.url=http://localhost:8080
  ```

6. Execute a classe `ApiGatewayApplication`, certificando-se que os serviços de pagamento e distância estão no ar, assim como o monólito.

  Alguns exemplos de URLs:

  - `http://localhost:9999/pagamentos/1`
  - `http://localhost:9999/distancia/restaurantes/mais-proximos/71503510`
  - `http://localhost:9999/restaurantes/1`

  Note que as URLs anteriores, apesar de serem invocados no API Gateway, invocam o serviço de pagamento, o de distância e o monólito, respectivamente.

## Exercício: fazendo a UI usar o API Gateway

1. Remova as URLs específicas dos serviços de distância e pagamento, mantendo apenas a `baseUrl`, que deve apontar para o API Gateway:

  ####### fj33-eats-ui/src/environments/environment.ts

  ```typescript
  export const environment = {
    production: false,

    b̶a̶s̶e̶U̶r̶l̶:̶ ̶'̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶0̶'̶
    baseUrl: '//localhost:9999' // modificado

    ,̶ ̶p̶a̶g̶a̶m̶e̶n̶t̶o̶U̶r̶l̶:̶ ̶'̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶1̶'̶
    ,̶ ̶d̶i̶s̶t̶a̶n̶c̶i̶a̶U̶r̶l̶:̶ ̶'̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶2̶'̶
  };
  ```

2. Em `PagamentoService`, troque `pagamentoUrl` por `baseUrl`:

  ####### fj33-eats-ui/src/app/services/pagamento.service.ts

  ```typescript
  export class PagamentoService {

    p̶r̶i̶v̶a̶t̶e̶ ̶A̶P̶I̶ ̶=̶ ̶e̶n̶v̶i̶r̶o̶n̶m̶e̶n̶t̶.̶p̶a̶g̶a̶m̶e̶n̶t̶o̶U̶r̶l̶ ̶+̶ ̶'̶/̶p̶a̶g̶a̶m̶e̶n̶t̶o̶s̶'̶;̶
    private API = environment.baseUrl + '/pagamentos'; // modificado

    // restante do código ...

  }
  ```

3. Use apenas `baseUrl` em `RestauranteService`, alterando o atributo `DISTANCIA_API`:

  ####### fj33-eats-ui/src/app/services/restaurante.service.ts

  ```typescript
  export class RestauranteService {

    private API = environment.baseUrl;

    p̶r̶i̶v̶a̶t̶e̶ ̶D̶I̶S̶T̶A̶N̶C̶I̶A̶_̶A̶P̶I̶ ̶=̶ ̶e̶n̶v̶i̶r̶o̶n̶m̶e̶n̶t̶.̶d̶i̶s̶t̶a̶n̶c̶i̶a̶U̶r̶l̶;̶
    private DISTANCIA_API = environment.baseUrl + '/distancia'; // modificado

    // código omitido ...

  }
  ```

4. Faça um novo pedido e efetue o pagamento. Deve funcionar!

5. Teste fazer o login como administrador e acessar a página de restaurantes em aprovação. Deve ocorrer um erro _401 Unauthorized_, que não acontecia antes da UI passar pelo API Gateway. Porque será que acontece esse erro?

<!--@note

  O Zuul remove alguns headers sensíveis.

  A configuração é a seguinte:

    sensitiveHeaders: Cookie,Set-Cookie,Authorization

  https://cloud.spring.io/spring-cloud-netflix/multi/multi__router_and_filter_zuul.html#_cookies_and_sensitive_headers

  Observação: talvez o ideal seja fazer a Autenticação/Autorização no próprio API Gateway.

-->

## Exercício: desabilitando a remoção de cabeçalhos sensíveis no Zuul

1. Por padrão, o Zuul remove os cabeçalhos HTTP `Cookie`, `Set-Cookie`, `Authorization`. Vamos desabilitar essa remoção no `application.properties`:

  ####### api-gateway/src/main/resources/application.properties

  ```properties
  zuul.sensitiveHeaders=
  ```

2. Reinicie o `ApiGatewayApplication` e faça o login como administrador. Acesse a página de restaurantes em aprovação. Deve funcionar!

<!--@note
Ao acessarmos a página que detalha um restaurante, como com a URL `http://localhost:4200/pedidos/71503510/restaurante/1`, são feitas duas chamadas pela UI:

  - `http://localhost:9999/restaurantes/1` busca os detalhes do restaurante do monólito
  - `http://localhost:9999/distancia/restaurantes/71503510/restaurante/1` busca os km entre o CEP e o restaurante do serviço de distância

Faça com que a UI obtenha tanto os detalhes do restaurante como a quilometragem até o CEP informado com apenas uma chamada a uma URL como `http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1`.

Invoque o serviço de distância com o `RestTemplate` do Spring e o monólito com o Feign. Usaremos isso mais adiante.
-->

## Exercício: invocando serviço de distância a partir do API Gateway com RestTemplate

1. Crie uma classe `RestClientConfig` no pacote `br.com.caelum.apigateway`, que fornece um `RestTemplate` do Spring:

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/RestClientConfig.java

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

  _Observação: estamos usando o `RestTemplate` ao invés do Feign porque estudaremos a diferença entre os dois mais adiante._

2. Adicione o Lombok como dependência no `pom.xml` do projeto `api-gateway`:

  ####### api-gateway/pom.xml

  ```xml
  <dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
  </dependency>
  ```

3. Crie uma classe `RestauranteComDistanciaDto`, que deve conter os atributos `restauranteId` e `distancia`, um `Long` e um `BigDecimal`, respectivamente.

  Defina as anotações `@Data`, `@AllArgsConstructor` e `@NoArgsConstructor` do Lombok.

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/RestauranteComDistanciaDto.java

  ```java
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  public class RestauranteComDistanciaDto {

    private Long restauranteId;
    private BigDecimal distancia;

  }
  ```

  Certifique-se dos imports corretos:

  ```java
  import java.math.BigDecimal;

  import lombok.AllArgsConstructor;
  import lombok.Data;
  import lombok.NoArgsConstructor;
  ```

5. Ainda no pacote `br.com.caelum.apigateway`, crie um `@Service` chamado `DistanciaRestClient` que recebe um `RestTemplate` e o valor de `zuul.routes.distancia.url`, que contém a URL do serviço de distância.

  No método `comDistanciaPorCepEId`, dispare um `GET` à URL do serviço de distância que retorna a quilometragem de um restaurante a um dado CEP:

  ```java
  @Service
  public class DistanciaRestClient {

    private RestTemplate restTemplate;
    private String distanciaServiceUrl;

    public DistanciaRestClient(RestTemplate restTemplate,
        @Value("${zuul.routes.distancia.url}") String distanciaServiceUrl) {
      this.restTemplate = restTemplate;
      this.distanciaServiceUrl = distanciaServiceUrl;
    }

    public RestauranteComDistanciaDto porCepEId(String cep, Long restauranteId) {
      String url = distanciaServiceUrl + "/restaurantes/" + cep + "/restaurante/" + restauranteId;
      return restTemplate.getForObject(url, RestauranteComDistanciaDto.class);
    }

  }
  ```

<!--  

## Exercício: invocando monólito a partir do API Gateway com Feign

3. No pacote `br.com.caelum.apigateway`, crie uma classe `TipoDeCozinhaDto` que contém o `id` e nome do tipo de cozinha.

  Crie também uma classe `RestauranteDto`, que contém todos dos dados informados na tela de detalhes de um restaurante, incluindo o tipo de cozinha.

  Defina, em ambas, as anotações `@Data`, `@AllArgsConstructor` e `@NoArgsConstructor` do Lombok.

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/TipoDeCozinhaDto.java

  ```java
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  public class TipoDeCozinhaDto {

    private Long id;
    private String nome;

  }
  ```

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/RestauranteDto.java

  ```java
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  public class RestauranteDto {

    private Long id;

    private String nome;

    private String descricao;

    private String cep;

    private String endereco;

    private BigDecimal taxaDeEntregaEmReais;

    private Integer tempoDeEntregaMinimoEmMinutos;

    private Integer tempoDeEntregaMaximoEmMinutos;

    private TipoDeCozinhaDto tipoDeCozinha;

  }
  ```

  Os imports são os que seguem:

  ```java
  import lombok.AllArgsConstructor;
  import lombok.Data;
  import lombok.NoArgsConstructor;
  ```

  Defina também o atributo `restaurante` do tipo `RestauranteDto` e o coloque como um `@Delegate` do Lombok para incorporar todos os atributos do restaurante à classe `RestauranteComDistanciaDto`. Essa última propriedade não deve ser serializada no JSON resultante.

  Na classe, use as anotações usadas em passos anteriores.

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/RestauranteComDistanciaDto.java

  ```java
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  public class RestauranteComDistanciaDto {

    private Long restauranteId;
    private BigDecimal distancia;

    @Delegate @JsonIgnore
    private RestauranteDto restaurante;

  }
  ```

  Certifique-se dos imports corretos:

  ```java
  import java.math.BigDecimal;

  import com.fasterxml.jackson.annotation.JsonIgnore;

  import lombok.AllArgsConstructor;
  import lombok.Data;
  import lombok.experimental.Delegate;
  import lombok.NoArgsConstructor;
  ```

-->
