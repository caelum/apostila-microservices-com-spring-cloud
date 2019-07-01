# Monitoramento e Observabilidade

## Exercício: expondo endpoints do Spring Boot Actuator

1. Adicione, a todos os projetos, uma dependência ao _starter_ do Spring Boot Actuator:

  ```xml
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
  </dependency>
  ```

  Essa dependência deve ser adicionada ao `pom.xml` do:

  - módulo `eats-application` do monólito
  - `eats-pagamento-service`
  - `eats-distancia-service`
  - `eats-nota-fiscal-service`
  - `api-gateway`
  - `service-registry`
  - `config-server`

2. Adicione, ao `application.properties` do `config-repo`, uma configuração para expôr todos os _endpoints_ do Actuator disponíveis:

  ####### config-repo/application.properties

  ```properties
  management.endpoints.web.exposure.include=*
  ```

  Não deixe de comitar a alteração:

  ```sh
  git commit -am "expõe endpoints do Spring Boot Actuator em todos os serviços"
  ```

  A configuração anterior será aplicada aos clientes do Config Server, que são os seguintes:

  - monólito
  - serviço de pagamentos
  - serviço de distância
  - serviço de nota fiscal
  - API Gateway

3. Para fazer com as requisições ao Actuator do API Gateway não acabem enviadas para o monólito, faça a configuração a seguir:

  ####### api-gateway/src/main/resources/application.properties

  ```properties
  zuul.routes.actuator.path=/actuator/**
  zuul.routes.actuator.url=forward:/actuator
  ```

4. Exponha todos os endpoints do Actuator no `config-server` e no `service-registry`:

  ####### config-server/src/main/resources/application.properties

  ```properties
  management.endpoints.web.exposure.include=*
  ```

  e

  ####### service-registry/src/main/resources/application.properties

  ```properties
  management.endpoints.web.exposure.include=*
  ```

5. Reinicie os serviços e explore os endpoints do Actuator.

  A seguinte URL contém links para os demais endpoints:

  http://localhost:{porta}/actuator

  É possível ver, de maneira detalhada, os valores das configurações:

  http://localhost:{porta}/actuator/configprops

  e

  http://localhost:{porta}/actuator/env

  Podemos verificar (e até modificar) os níveis de log:

  http://localhost:{porta}/actuator/loggers

  Com a URL a seguir, podemos ver uma lista de métricas disponíveis:

  http://localhost:{porta}/actuator/metrics

  Por exemplo, podemos obter o _uptime_ da JVM com a URL:

  http://localhost:{porta}/actuator/metrics/process.uptime

  Há uma lista dos `@RequestMapping` da aplicação:

  http://localhost:{porta}/actuator/mappings

  Podemos obter informações sobre os bindings, exchanges e channels do Spring Cloud Stream com as URLs:

  http://localhost:{porta}/actuator/bindings

  e

  http://localhost:{porta}/actuator/channels

  _Observação: troque `{porta}` pela porta de algum serviço._

  Há ainda endpoints específicos para o serviço que estamos acessando. Por exemplo, para o API Gateway temos com as rotas e _filters_:

  http://localhost:9999/actuator/routes

  e

  http://localhost:9999/actuator/filters

## Exercício: configurando o Hystrix Dashboard

1. Pelo navegador, abra `https://start.spring.io/`.
  Em _Project_, mantenha _Maven Project_.
  Em _Language_, mantenha _Java_.
  Em _Spring Boot_, mantenha a versão padrão.
  No trecho de _Project Metadata_, defina:

  - `br.com.caelum` em _Group_
  - `hystrix-dashboard` em _Artifact_

  Mantenha os valores em _More options_.
 
  Mantenha o _Packaging_ como `Jar`.
  Mantenha a _Java Version_ em `8`.

  Em _Dependencies_, adicione:

  - Hystrix Dashboard

  Clique em _Generate Project_.
2. Extraia o `hystrix-dashboard.zip` e copie a pasta para seu Desktop.
3. No Eclipse, no workspace de microservices, importe o projeto `hystrix-dashboard`, usando o menu _File > Import > Existing Maven Projects_.
4. Adicione a anotação `@EnableHystrixDashboard` à classe `HystrixDashboardApplication`:

  ####### hystrix-dashboard/src/main/java/br/com/caelum/hystrixdashboard/HystrixDashboardApplication.java

  ```java
  @EnableHystrixDashboard
  @SpringBootApplication
  public class HystrixDashboardApplication {

    public static void main(String[] args) {
      SpringApplication.run(HystrixDashboardApplication.class, args);
    }

  }
  ```

  Adicione o import:

  ```java
  import org.springframework.cloud.netflix.hystrix.dashboard.EnableHystrixDashboard;
  ```

5. No arquivo `application.properties`, modifique a porta para `7777`:

  ####### hystrix-dashboard/src/main/resources/application.properties

  ```properties
  server.port=7777
  ```

6. Execute a classe `HystrixDashboardApplication`.

