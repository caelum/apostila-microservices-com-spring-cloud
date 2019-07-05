# API Gateway

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

5. Teste fazer o login como administrador e acessar a página de restaurantes em aprovação. Deve ocorrer um erro _401 Unauthorized_, que não acontecia antes da UI passar pelo API Gateway. Por que será que acontece esse erro?

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

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/DistanciaRestClient.java

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

## Exercício: invocando monólito a partir do API Gateway com Feign

1. Crie uma classe `TipoDeCozinhaDto` que contém o `id` e nome do tipo de cozinha.

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

2. Adicione o Feign como dependência no `pom.xml` do projeto `api-gateway`:

  ####### api-gateway/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
  </dependency>
  ```

3. Na classe `ApiGatewayApplication`, adicione a anotação `@EnableFeignClients`:

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/ApiGatewayApplication.java

  ```java
  @EnableFeignClients // adicionado
  @EnableZuulProxy
  @SpringBootApplication
  public class ApiGatewayApplication {

    public static void main(String[] args) {
      SpringApplication.run(ApiGatewayApplication.class, args);
    }

  }
  ```

  O import a ser adicionado está a seguir:

  ```java
  import org.springframework.cloud.openfeign.EnableFeignClients;
  ```

5. Crie uma interface `RestauranteRestClient`, que define um método `porId` que recebe um `id` e retorna um `RestauranteDto`. Anote esse método com as anotações do Spring Web, para que dispare um GET à URL do monólito que detalha um restaurante.

  A interface deve ser anotada com `@FeignClient`, apontando para a configuração do monólito no Zuul. 

  ```java
  @FeignClient("monolito")
  public interface RestauranteRestClient {

    @GetMapping("/restaurantes/{id}")
    public RestauranteDto porId(@PathVariable("id") Long id);

  }
  ```

  Ajuste os imports:

  ```java
  import org.springframework.cloud.openfeign.FeignClient;
  import org.springframework.web.bind.annotation.GetMapping;
  import org.springframework.web.bind.annotation.PathVariable;
  ```

6. A configuração do monólito no Zuul precisa ser ligeiramente alterada para que o Feign funcione:

  ####### api-gateway/src/main/resources/application.properties

  ```properties
  z̶u̶u̶l̶.̶r̶o̶u̶t̶e̶s̶.̶m̶o̶n̶o̶l̶i̶t̶o̶.̶u̶r̶l̶=̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶0̶
  monolito.ribbon.listOfServers=http://localhost:8080
  ```

  Mais adiante estudaremos cuidadosamente o Ribbon.

## Exercício: compondo chamadas no API Gateway

1. Adicione, à classe `RestauranteComDistanciaDto`, um atributo `restaurante` do tipo `RestauranteDto`.

  Para incorporar todos os atributos do restaurante na própria classe `RestauranteComDistanciaDto`, coloque o restaurante como um `@Delegate` do Lombok.
  
  Esse novo atributo não deve ser serializado no JSON resultante.

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/RestauranteComDistanciaDto.java

  ```java
  // anotações ...
  public class RestauranteComDistanciaDto {

    // outros atributos ...

    // adicionado
    @Delegate @JsonIgnore
    private RestauranteDto restaurante;

  }
  ```

  Adicione os imports corretos:

  ```java
  import com.fasterxml.jackson.annotation.JsonIgnore;

  import lombok.experimental.Delegate;
  ```

2. No `api-gateway`, crie um `RestauranteComDistanciaController`, que invoca dado um CEP e um id de restaurante obtém:

  - os detalhes do restaurante usando `RestauranteRestClient`
  - a quilometragem entre o restaurante e o CEP usando `DistanciaRestClient`

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/RestauranteComDistanciaController.java

  ```java
  @RestController
  @AllArgsConstructor
  public class RestauranteComDistanciaController {

    private RestauranteRestClient restauranteRestClient;
    private DistanciaRestClient distanciaRestClient;

    @GetMapping("/restaurantes-com-distancia/{cep}/restaurante/{restauranteId}")
    public RestauranteComDistanciaDto porCepEIdComDistancia(@PathVariable("cep") String cep, @PathVariable("restauranteId") Long restauranteId) {
      RestauranteDto restaurante = restauranteRestClient.porId(restauranteId);
      RestauranteComDistanciaDto restauranteComDistancia = distanciaRestClient.porCepEId(cep, restauranteId);
      restauranteComDistancia.setRestaurante(restaurante);
      return restauranteComDistancia;
    }

  }
  ```

  Não esqueça dos imports:

  ```java
  import org.springframework.web.bind.annotation.GetMapping;
  import org.springframework.web.bind.annotation.PathVariable;
  import org.springframework.web.bind.annotation.RestController;

  import lombok.AllArgsConstructor;
  ```

3. Tente acessar, pelo navegador ou pelo cURL, a URL: `http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1`

  Note que o status da resposta é `401 Unauthorized`.

  Isso ocorre porque, como o prefixo não é `pagamentos` nem `distancia`, a requisição é repassada ao monólito pelo Zuul.

  Devemos configurar uma rota no Zuul, usando o `forward` para o endereço local:

  ####### api-gateway/src/main/resources/application.properties

  ```properties
  zuul.routes.local.path=/restaurantes-com-distancia/**
  zuul.routes.local.url=forward:/restaurantes-com-distancia
  ```

  A rota acima deve ficar logo antes da rota do monólito, porque esta última é `/**`,  um "coringa" que corresponde a qualquer URL solicitada.

4. Tente novamente acessar a URL `http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1`

  Deve ser retornado algo parecido com:

  ```json
  {
    "restauranteId":1,
    "distancia":11.393642891403121808480136678554117679595947265625,
    "cep":"70238500",
    "descricao":"O melhor da China aqui do seu lado.",
    "endereco":"ShC/SUL COMERCIO LOCAL QD 404-BL D LJ 17-ASA SUL",
    "nome":"Long Fu",
    "taxaDeEntregaEmReais":6.00,
    "tempoDeEntregaMaximoEmMinutos":25,
    "tempoDeEntregaMinimoEmMinutos":40,
    "tipoDeCozinha":{
      "id":1,
      "nome":"Chinesa"
    },
    "id":1
  }
  ```

## Exercício: chamando a composição do API Gateway a partir da UI

1. No projeto `eats-ui`, adicione um método que chama a nova URL do API Gateway em `RestauranteService`:

  ####### fj33-eats-ui/src/app/services/restaurante.service.ts

  ```typescript
  export class RestauranteService {

    // código omitido ...

    porCepEIdComDistancia(cep: string, restauranteId: string): Observable<any> {
      return this.http.get(`${this.API}/restaurantes-com-distancia/${cep}/restaurante/${restauranteId}`);
    }

  }
  ```

2. Altere ao `RestauranteComponent` para que chame o novo método `porCepEIdComDistancia`.

  Não será mais necessário invocar o método `distanciaPorCepEId`, porque o restaurante já terá a distância.

  ####### fj33-eats-ui/src/app/pedido/restaurante/restaurante.component.ts

  ```java
  export class RestauranteComponent implements OnInit {

    // código omitido ...

    ngOnInit() {

      t̶h̶i̶s̶.̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶S̶e̶r̶v̶i̶c̶e̶.̶p̶o̶r̶I̶d̶(̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶I̶d̶)̶
      this.restaurantesService.porCepEIdComDistancia(this.cep, restauranteId) // modificado
        .subscribe(restaurante => {

          this.restaurante = restaurante;
          this.pedido.restaurante = restaurante;

          t̶h̶i̶s̶.̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶S̶e̶r̶v̶i̶c̶e̶.̶d̶i̶s̶t̶a̶n̶c̶i̶a̶P̶o̶r̶C̶e̶p̶E̶I̶d̶(̶t̶h̶i̶s̶.̶c̶e̶p̶,̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶I̶d̶)̶
            .̶s̶u̶b̶s̶c̶r̶i̶b̶e̶(̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶C̶o̶m̶D̶i̶s̶t̶a̶n̶c̶i̶a̶ ̶=̶>̶ ̶{̶
              t̶h̶i̶s̶.̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶.̶d̶i̶s̶t̶a̶n̶c̶i̶a̶ ̶=̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶C̶o̶m̶D̶i̶s̶t̶a̶n̶c̶i̶a̶.̶d̶i̶s̶t̶a̶n̶c̶i̶a̶;̶
          }̶)̶;̶

          // código omitido ...

        });

    }

    // restante do código ...

  }
  ```

3. Busque os restaurantes a partir de um CEP e escolha um deles. Você também pode acessar, diretamente pelo navegador, uma URL como: `http://localhost:4200/pedidos/71503510/restaurante/1`.

  Deve ocorrer um _Erro no servidor_.

  No Console do navegador, podemos perceber que o erro é relacionado a CORS:

  _Access to XMLHttpRequest at 'http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1' from origin 'http://localhost:4200' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource._

4. Adicione ao método `porCepEIdComDistancia` de `RestauranteComDistanciaController` a anotação `@CrossOrigin`:

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/RestauranteComDistanciaController.java

  ```java
  // anotações ...
  public class RestauranteComDistanciaController {

    // atributos ...

    @CrossOrigin // adicionado
    @GetMapping("/restaurantes-com-distancia/{cep}/restaurante/{restauranteId}")
    public RestauranteComDistanciaDto porCepEIdComDistancia(@PathVariable("cep") String cep, @PathVariable("restauranteId") Long restauranteId) {
      // código omitido ...
    }

  }
  ```

5. Teste novamente. Perceba que os detalhes do restaurante são exibidos, assim como a distância ao CEP.

## Exercício: LocationRewriteFilter no Zuul

1. Através de um cliente REST, tente adicionar um pagamento passando pelo API Gateway. Para isso, utilize a porta `9999`.

  Com o cURL é algo como:

  ```sh
  curl -X POST
    -i
    -H 'Content-Type: application/json'
    -d '{ "valor": 51.8, "nome": "JOÃO DA SILVA", "numero": "1111 2222 3333 4444", "expiracao": "2022-07", "codigo": "123", "formaDePagamentoId": 2, "pedidoId": 1 }'
    http://localhost:9999/pagamentos
  ```

  Lembrando que um comando semalhante ao anterio, mas com a porta `8081`, está disponível em: https://gitlab.com/snippets/1859389

  Note no cabeçalho `Location` do response que, mesmo utilizando a porta `9999` na requisição, a porta da resposta é a `8081`.

  ```txt
  Location: http://localhost:8081/pagamentos/40
  ```

2. O `LocationRewriteFilter` padrão do Spring Cloud Zuul só funciona para chamadas com status de resposta `3XX`, de redirecionamentos. Vamos customizá-lo, para que funcione com respostas bem sucedidas, de status `2XX`.

  Para isso, crie uma classe `LocationRewriteConfig` no pacote `br.com.caelum.apigateway`, definindo uma subclasse anônima  de `LocationRewriteFilter`, modificando alguns detalhes.

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/LocationRewriteConfig.java

  ```java
  @Configuration
  public class LocationRewriteConfig {

    @Bean
    public LocationRewriteFilter locationRewriteFilter() {
      return new LocationRewriteFilter() {
        @Override
        public boolean shouldFilter() {
          int statusCode = RequestContext.getCurrentContext().getResponseStatusCode();
          return HttpStatus.valueOf(statusCode).is3xxRedirection() || HttpStatus.valueOf(statusCode).is2xxSuccessful();
        }
      };
    }

  }
  ```

  Tome bastante cuidado com os imports:

  ```java
  import org.springframework.cloud.netflix.zuul.filters.post.LocationRewriteFilter;
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.http.HttpStatus;

  import com.netflix.zuul.context.RequestContext;
  ```

3. Teste novamente a criação de um pagamento com um cliente REST. Perceba que o cabeçalho `Location` agora tem a porta `9999`, do API Gateway.

4. (desafio - opcional) Se você fez os exercícios opcionais de Spring HATEOAS, note que as URLs dos links ainda contém a porta `8081`. Implemente um Filter do Zuul que modifique as URLs do corpo de um _response_ para que apontem para a porta `9999`, do API Gateway.

## Exercício opcional: um ZuulFilter de Rate Limiting

1. Adicione, no `pom.xml` de `api-gateway`, uma dependência a biblioteca Google Guava:

  ####### api-gateway/pom.xml

  ```xml
  <dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>15.0</version>
  </dependency>
  ```

  A biblioteca Google Guava possui uma implementação de _rate limiter_, que restringe o acesso a recurso em uma determinada taxa configurável.

2. Crie um `ZuulFilter` que retorna uma falha com status `429 TOO MANY REQUESTS` se a taxa de acesso ultrapassar 1 requisição a cada 30 segundos:

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/RateLimitingZuulFilter.java

  ```java
  @Component
  public class RateLimitingZuulFilter extends ZuulFilter {

    private final RateLimiter rateLimiter = RateLimiter.create(1.0 / 30.0); //queries per second

    @Override
    public String filterType() {
      return FilterConstants.PRE_TYPE;
    }

    @Override
    public int filterOrder() {
      return Ordered.HIGHEST_PRECEDENCE + 100;
    }

    @Override
    public boolean shouldFilter() {
      return true;
    }

    @Override
    public Object run() {
      try {
        RequestContext currentContext = RequestContext.getCurrentContext();
        HttpServletResponse response = currentContext.getResponse();

        if (!this.rateLimiter.tryAcquire()) {
          response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
          response.getWriter().append(HttpStatus.TOO_MANY_REQUESTS.getReasonPhrase());
          currentContext.setSendZuulResponse(false);
        }
      } catch (IOException e) {
        ReflectionUtils.rethrowRuntimeException(e);
      }
      return null;
    }
  }
  ```

  Os imports são os seguintes:

  ```java
  import java.io.IOException;

  import javax.servlet.http.HttpServletResponse;

  import org.springframework.cloud.netflix.zuul.filters.support.FilterConstants;
  import org.springframework.core.Ordered;
  import org.springframework.http.HttpStatus;
  import org.springframework.stereotype.Component;
  import org.springframework.util.ReflectionUtils;

  import com.google.common.util.concurrent.RateLimiter;
  import com.netflix.zuul.ZuulFilter;
  import com.netflix.zuul.context.RequestContext;
  ```

3. Garanta que `ApiGatewayApplication` foi reiniciado e acesse alguma várias vezes seguidas pelo navegador, uma URL como `http://localhost:9999/restaurantes/1`.

  Deve ocorrer um erro `429 Too Many Requests`.

4. Apague (ou desabilite comentando a anotação `@Component`) a classe `RateLimitingZuulFilter` para que não cause erros na aplicação no restante do curso.
