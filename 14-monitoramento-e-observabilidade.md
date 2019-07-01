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

  _Observação: troque `{porta}` pela porta de um serviço qualquer._

  Há ainda endpoints específicos para o serviço que estamos acessando. Por exemplo, para o API Gateway temos com as rotas e _filters_:

  http://localhost:9999/actuator/routes

  e

  http://localhost:9999/actuator/filters
