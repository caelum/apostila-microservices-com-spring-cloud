# Client Side Load Balancing com Ribbon

## Exercício: executando uma segunda instância do serviço de distância

1. Para que todas as requisições do serviço de distância sejam logadas (e com mais informações), vamos configurar um `CommonsRequestLoggingFilter`.

  Para isso, crie a classe `RequestLogConfig` no pacote `br.com.caelum.eats.distancia`:

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/RequestLogConfig.java

  ```java
  @Configuration
  public class RequestLogConfig {

    @Bean
    public CommonsRequestLoggingFilter requestLoggingFilter() {
      CommonsRequestLoggingFilter loggingFilter = new CommonsRequestLoggingFilter();
      loggingFilter.setIncludeClientInfo(true);
      loggingFilter.setIncludeQueryString(true);
      loggingFilter.setIncludePayload(true);
      return loggingFilter;
    }

  }
  ```

  Os imports são os seguintes:

  ```java
  import org.springframework.context.annotation.Bean;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.web.filter.CommonsRequestLoggingFilter;
  ```

2. Mude o nível de log do `CommonsRequestLoggingFilter` para `DEBUG` no `application.properties`:

  ####### eats-distancia-service/src/main/resources/application.properties

  ```properties
  logging.level.org.springframework.web.filter.CommonsRequestLoggingFilter=DEBUG
  ```

3. Configure a segunda instância para que seja executada na porta `8282`.

  No Eclipse, acesse o menu _Run > Run Configurations..._.

  Clique com o botão direito na configuração `EatsDistanciaServiceApplication` e, então, na opção _Duplicate_.

  Deve ser criada a configuração `EatsDistanciaServiceApplication (1)`.

  Na aba _Arguments_, defina `8282` como a porta dessa segunda instância, em _VM Arguments_:

  ```txt
  -Dserver.port=8282
  ```

  Clique em _Run_. Nova instância do serviço de distância no ar!

4. Acesse uma URL do serviço de distância que está sendo executado na porta `8082` como, por exemplo, a URL `http://localhost:8082/restaurantes/mais-proximos/71503510`. Verifique os logs no Console do Eclipse, na configuração `EatsDistanciaServiceApplication`.

  Use a porta para `8282`, por meio de uma URL como `http://localhost:8282/restaurantes/mais-proximos/71503510`. Note que os logs do Console do Eclipse agora são da configuração `EatsDistanciaServiceApplication (1)`.

## Exercício: client side load balancing com Ribbon no API Gateway

1. Modifique o `application.properties` do `api-gateway`, para que use o Ribbon como _load balancer_.

  Troque a configuração do Zuul do serviço de distância para fazer um _matching_ pelo `path`. Em seguida, configure a lista de servidores do Ribbon com as instâncias do serviço de distância.

  Adicione a propriedade `configuracao.distancia.service.url`, usando a URL `http://distancia` do Ribbon. Essa propriedade será usada no `DistanciaRestClient`.

  ####### api-gateway/src/main/resources/application.properties

  ```properties
  z̶u̶u̶l̶.̶r̶o̶u̶t̶e̶s̶.̶d̶i̶s̶t̶a̶n̶c̶i̶a̶.̶u̶r̶l̶=̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶2̶

  zuul.routes.distancia.path=/distancia/**
  distancia.ribbon.listOfServers=http://localhost:8082,http://localhost:8282

  configuracao.distancia.service.url=http://distancia
  ```

2. Modifique a anotação `@Value` do construtor de `DistanciaRestClient` para que use a propriedade `configuracao.distancia.service.url`:

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/DistanciaRestClient.java

  ```java
  public class DistanciaRestClient {

    // código omitido ...

    public DistanciaRestClient(RestTemplate restTemplate,
          @̶V̶a̶l̶u̶e̶(̶"̶$̶{̶z̶u̶u̶l̶.̶r̶o̶u̶t̶e̶s̶.̶d̶i̶s̶t̶a̶n̶c̶i̶a̶.̶u̶r̶l̶}̶"̶)̶ ̶S̶t̶r̶i̶n̶g̶ ̶d̶i̶s̶t̶a̶n̶c̶i̶a̶S̶e̶r̶v̶i̶c̶e̶U̶r̶l̶)̶ ̶{̶
          @Value("${configuracao.distancia.service.url}") String distanciaServiceUrl) {

      this.restTemplate = restTemplate;
      this.distanciaServiceUrl = distanciaServiceUrl;

    }

    // restante do código ...

  }
  ```

3. Para que a instância do `RestTemplate` use o Ribbon, anote o método `restTemplate` de `RestClientConfig` com `@LoadBalanced`:

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/RestClientConfig.java

  ```java
  @Configuration
  public class RestClientConfig {

    @LoadBalanced // adicionado
    @Bean
    public RestTemplate restTemplate() {
      return new RestTemplate();
    }

  }
  ```

  O import correto é:

  ```java
  import org.springframework.cloud.client.loadbalancer.LoadBalanced;
  ```

4. Veja que cada instância do serviço de distância é chamada alternadamente ao acessar a URL `http://localhost:9999/distancia/restaurantes/mais-proximos/71503510`, que efetua o proxy do Zuul.

  Observe, pelos logs, que a URL `http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1`, que compões as chamadas ao monólito e ao serviço de distância (esse, através do `RestTemplate` com `@LoadBalanced`), também alterna entre as instâncias.
