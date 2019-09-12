# Client Side Load Balancing com Ribbon

## Detalhando o log de requests do serviço de distância

Para que todas as requisições do serviço de distância sejam logadas (e com mais informações), vamos configurar um `CommonsRequestLoggingFilter`.

Para isso, crie a classe `RequestLogConfig` no pacote `br.com.caelum.eats.distancia`:

####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/RequestLogConfig.java

```java
@Configuration
class RequestLogConfig {

  @Bean
  CommonsRequestLoggingFilter requestLoggingFilter() {
    CommonsRequestLoggingFilter loggingFilter = new CommonsRequestLoggingFilter();
    loggingFilter.setIncludeClientInfo(true);
    loggingFilter.setIncludePayload(true);
    loggingFilter.setIncludeHeaders(true);
    loggingFilter.setIncludeQueryString(true);
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

O nível de log do `CommonsRequestLoggingFilter` deve ser modificado para `DEBUG` no `application.properties`:

####### eats-distancia-service/src/main/resources/application.properties

```properties
logging.level.org.springframework.web.filter.CommonsRequestLoggingFilter=DEBUG
```

## Exercício: executando uma segunda instância do serviço de distância

1. Interrompa o serviço de distância.

  No projeto `fj33-eats-distancia-service`, vá até a branch `cap8-detalhando-o-log-de-resquests-do-servico-de-distancia`:

  ```sh
  cd ~/Desktop/fj33-eats-distancia-service
  git checkout -f cap8-detalhando-o-log-de-resquests-do-servico-de-distancia
  ```

  Execute a classe `EatsDistanciaServiceApplication`.

2. Configure a segunda instância do serviço de distância para que seja executada na porta `9092`.

  No Eclipse, acesse o menu _Run > Run Configurations..._.

  Clique com o botão direito na configuração `EatsDistanciaServiceApplication` e, então, na opção _Duplicate_.

  Deve ser criada a configuração `EatsDistanciaServiceApplication (1)`.

  Na aba _Arguments_, defina `9092` como a porta dessa segunda instância, em _VM Arguments_:

  ```txt
  -Dserver.port=9092
  ```

  Clique em _Run_. Nova instância do serviço de distância no ar!

3. Acesse uma URL do serviço de distância que está sendo executado na porta `8082` como, por exemplo, a URL `http://localhost:8082/restaurantes/mais-proximos/71503510`. Verifique os logs no Console do Eclipse, na configuração `EatsDistanciaServiceApplication`.

  Use a porta para `9092`, por meio de uma URL como `http://localhost:9092/restaurantes/mais-proximos/71503510`. Note que os logs do Console do Eclipse agora são da configuração `EatsDistanciaServiceApplication (1)`.

## Client side load balancing no RestTemplate do monólito com Ribbon

No `pom.xml` do módulo `eats`, o módulo pai do monólito, adicione uma dependência ao _Spring Cloud_ na versão `Greenwich.SR2`, em `dependencyManagement`:

####### fj33-eats-monolito-modular/eats/pom.xml

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>Greenwich.SR2</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

Adicione o _starter_ do Ribbon como dependência do módulo `eats-application` do monólito:
  
####### fj33-eats-monolito-modular/eats/eats-application/pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```

Para que a instância do `RestTemplate` configurada no módulo `eats-application` do monólito use o Ribbon, anote o método `restTemplate` de `RestClientConfig` com `@LoadBalanced`:

####### fj33-eats-monolito-modular/eats/eats-application/src/main/java/br/com/caelum/eats/RestClientConfig.java

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

Mude o arquivo `application.properties`, do módulo `eats-application` do monólito, para que seja configurado o _virtual host_ `distancia`, com uma lista de servidores cujas chamadas serão alternadas.

Faça com que a propriedade `configuracao.distancia.service.url` aponte para esse _virtual host_.

Por enquanto, desabilite o Eureka, que será abordado mais adiante.

####### fj33-eats-monolito-modular/eats/eats-application/src/main/resources/application.properties

```properties
c̶o̶n̶f̶i̶g̶u̶r̶a̶c̶a̶o̶.̶d̶i̶s̶t̶a̶n̶c̶i̶a̶.̶s̶e̶r̶v̶i̶c̶e̶.̶u̶r̶l̶=̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶2̶
configuracao.distancia.service.url=http://distancia

distancia.ribbon.listOfServers=http://localhost:8082,http://localhost:9092
ribbon.eureka.enabled=false
```

## Client side load balancing no RestTemplate do API Gateway com Ribbon

O Zuul já é integrado com o Ribbon e, por isso, não precisamos colocá-lo como dependência.

Modifique o `application.properties` do `api-gateway`, para que use o Ribbon como _load balancer_ nas chamadas ao serviço de distância.

Troque a configuração do Zuul do serviço de distância para fazer um _matching_ pelo `path`. Em seguida, configure a lista de servidores do Ribbon com as instâncias do serviço de distância.

Adicione a propriedade `configuracao.distancia.service.url`, usando a URL `http://distancia` do Ribbon. Essa propriedade será usada no `DistanciaRestClient`.

####### api-gateway/src/main/resources/application.properties

```properties
z̶u̶u̶l̶.̶r̶o̶u̶t̶e̶s̶.̶d̶i̶s̶t̶a̶n̶c̶i̶a̶.̶u̶r̶l̶=̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶2̶

zuul.routes.distancia.path=/distancia/**
distancia.ribbon.listOfServers=http://localhost:8082,http://localhost:9092

configuracao.distancia.service.url=http://distancia
```

Modifique a anotação `@Value` do construtor de `DistanciaRestClient` para que use a propriedade `configuracao.distancia.service.url`:

####### api-gateway/src/main/java/br/com/caelum/apigateway/DistanciaRestClient.java

```java
class DistanciaRestClient {

  // código omitido ...

  DistanciaRestClient(RestTemplate restTemplate,
        @̶V̶a̶l̶u̶e̶(̶"̶$̶{̶z̶u̶u̶l̶.̶r̶o̶u̶t̶e̶s̶.̶d̶i̶s̶t̶a̶n̶c̶i̶a̶.̶u̶r̶l̶}̶"̶)̶ ̶S̶t̶r̶i̶n̶g̶ ̶d̶i̶s̶t̶a̶n̶c̶i̶a̶S̶e̶r̶v̶i̶c̶e̶U̶r̶l̶)̶ ̶{̶
        @Value("${configuracao.distancia.service.url}") String distanciaServiceUrl) {

    this.restTemplate = restTemplate;
    this.distanciaServiceUrl = distanciaServiceUrl;

  }

  // restante do código ...

}
```

Na classe `RestClientConfig` do `api-gateway`, faça com que o `RestTemplate` seja `@LoadBalanced`:

####### api-gateway/src/main/java/br/com/caelum/apigateway/RestClientConfig.java

```java
@Configuration
class RestClientConfig {

  @LoadBalanced // adicionado
  @Bean
  RestTemplate restTemplate() {
    return new RestTemplate();
  }

}
```

Lembrando que o import correto é:

```java
import org.springframework.cloud.client.loadbalancer.LoadBalanced;
```

## Exercício: Testando o client side load balancing no RestTemplate do monólito com Ribbon

1. Interrompa o monólito e o API Gateway.

  Faça o checkout da branch `cap8-client-side-load-balancing-no-rest-template-com-ribbon` dos projetos `fj33-eats-monolito-modular` e `fj33-api-gateway`:

  ```sh
  cd ~/Desktop/fj33-eats-monolito-modular
  git checkout -f cap8-client-side-load-balancing-no-rest-template-com-ribbon

  cd ~/Desktop/fj33-api-gateway
  git checkout -f cap8-client-side-load-balancing-no-rest-template-com-ribbon
  ```

  Execute novamente o monólito e o API Gateway.

2. Certifique-se que o monólito, o serviço de distância, o API Gateway e a UI estejam no ar.

  Teste a alteração do CEP e/ou tipo de cozinha de um restaurante. Para isso, efetue o login como um dono de restaurante. Se desejar, use as credenciais pré-cadastradas (`longfu`/`123456`) do restaurante Long Fu.

  Observe qual instância do serviço de distância foi invocada.

  Tente alterar novamente o CEP e/ou tipo de cozinha do restaurante. Note que foi invocada a outra instância do serviço de distância.

  A cada alteração, as instâncias são invocadas alternadamente.

3. Teste também a API Composition do API Gateway, que invoca o serviço de distância usando um `RestTemplate` do Spring, agora com `@LoadBalanced`, na classe `DistanciaRestClient`.

  Observe, pelos logs, que a URL `http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1` também alterna entre as instâncias.

  O Zuul já está integrado com o Ribbon. Então, ao utilizarmos o Zuul como proxy, a alternância entre as instâncias já é efetuada. Teste isso acessando a URL `http://localhost:9999/distancia/restaurantes/mais-proximos/71503510`.

## Exercício: executando uma segunda instância do monólito

1. Faça com que uma segunda instância do monólito rode com a porta `9090`.

  No workspace do monólito, acesse o menu _Run > Run Configurations..._ do Eclipse e clique com o botão direito na configuração `EatsApplication` e depois clique em _Duplicate_.

  Na configuração `EatsApplication (1)` que foi criada, acesse a aba _Arguments_ e defina `9090` como a porta da segunda instância, em _VM Arguments_:

  ```txt
  -Dserver.port=9090
  ```

  Clique em _Run_. Nova instância do monólito no ar!

## Client side load balancing no Feign do serviço de pagamentos com Ribbon

Adicione como dependência o _starter_ do Ribbon no `pom.xml` do `eats-pagamento-service`:

####### eats-pagamento-service/pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-ribbon</artifactId>
</dependency>
```

Configure a URL do monólito para use uma lista de servidores do Ribbon e, por enquanto, desabilite o Eureka:

####### eats-pagamento-service/src/main/resources/application.properties

```properties
c̶o̶n̶f̶i̶g̶u̶r̶a̶c̶a̶o̶.̶p̶e̶d̶i̶d̶o̶.̶s̶e̶r̶v̶i̶c̶e̶.̶u̶r̶l̶=̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶0̶

monolito.ribbon.listOfServers=http://localhost:8080,http://localhost:9090
ribbon.eureka.enabled=false
```

Troque a anotação do Feign em `PedidoRestClient` para que aponte para a configuração `monolito` do Ribbon:

####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PedidoRestClient.java

```java
@̶F̶e̶i̶g̶n̶C̶l̶i̶e̶n̶t̶(̶u̶r̶l̶=̶"̶$̶{̶c̶o̶n̶f̶i̶g̶u̶r̶a̶c̶a̶o̶.̶p̶e̶d̶i̶d̶o̶.̶s̶e̶r̶v̶i̶c̶e̶.̶u̶r̶l̶}̶"̶,̶ ̶n̶a̶m̶e̶=̶"̶p̶e̶d̶i̶d̶o̶"̶)̶
@FeignClient("monolito") // modificado
public interface PedidoRestClient {

  // código omitido ...

}
```

## Client side load balancing no Feign do API Gateway com Ribbon

No `application.properties` do `api-gateway`, adicione da URL da segunda instância do monólito:

####### api-gateway/src/main/resources/application.properties

```properties
m̶o̶n̶o̶l̶i̶t̶o̶.̶r̶i̶b̶b̶o̶n̶.̶l̶i̶s̶t̶O̶f̶S̶e̶r̶v̶e̶r̶s̶=̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶0̶
monolito.ribbon.listOfServers=http://localhost:8080,http://localhost:9090
```

## Exercício: Client side load balancing no Feign com Ribbon

1. Pare o serviço de pagamentos e o API Gateway.

  Vá até a branch `cap8-client-side-load-balancing-no-feign-com-ribbon` nos projetos `fj33-eats-pagamento-service` e `fj33-api-gateway`:

  ```sh
  cd ~/Desktop/fj33-eats-pagamento-service
  git checkout -f cap8-client-side-load-balancing-no-feign-com-ribbon

  cd ~/Desktop/fj33-api-gateway
  git checkout -f cap8-client-side-load-balancing-no-feign-com-ribbon
  ```

  Execute novamente o serviço de pagamentos e o API Gateway.

2. Garanta que o serviço de pagamento foi reiniciado e que as duas instâncias do monólito estão no ar.

  Use um cliente REST como o cURL para confirmar um pagamento:

  ```txt
  curl -X PUT -i http://localhost:8081/pagamentos/1
  ```

  Teste várias vezes seguidas e note que os logs são alternados entre `EatsApplication` e `EatsApplication (1)`, as instâncias do monólito.

  _Observação: confirmar um pagamento já confirmado tem o mesmo efeito, incluindo o aviso de pagamento ao monólito._

3. Acesse pelo API Gateway, por duas vezes seguidas, uma URL do monólito como `http://localhost:9999/restaurantes/1`.

  Veja que os logs são alternados entre os Consoles de `EatsApplication` e `EatsApplication (1)`.
