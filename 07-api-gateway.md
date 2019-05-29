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

-->
