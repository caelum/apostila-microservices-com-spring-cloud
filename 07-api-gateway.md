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
