# Monitoramento e Observabilidade

## Monitorando um sistema complexo 

À medida que fomos avançando, a Arquitetura do Caelum Eats foi aumentando progressivamente de complexidade. Temos cada vez mais componentes e, consequentemente, cada vez mais coisas podem apresentar erros.

![Componentes do back-end do Caelum Eats, com BDs omitidos {w=79}](imagens/13-monitoramento-e-observabilidade/backend-caelum-eats-todos-os-componentes-menos-bd.png)

Se houver uma demora no sistema, como descobrir a causa? Um request passa por quais componentes?

Não é possível debugar um Sistema Distribuído de maneira fácil, auxiliado por uma IDE, como é comum com um Monólito.

A _ilidade_ associada à capacidade de conhecer o estado de um sistema é chamada de **Observabilidade**. No artigo [Observability — A 3-Year Retrospective](https://thenewstack.io/observability-a-3-year-retrospective/) (MAJORS, 2019), Charity Majors, a CTO da Honeycomb, indica que o termo passou a ser utilizado por volta de 2016 por times de operações de empresas como Twitter, Stripe e Google.

Veremos algumas ideias e ferramentas que auxiliam a aumentar a Observabilidade de um sistema.

## Health Check API

É possível que uma instância de um serviço esteja no ar, recebendo requests, mas em um estado em que não consegue completá-las. Por exemplo, o BD pode estar momentaneamente fora do ar.

Para auxiliar na Observabilidade, uma instância de um serviço deve informar o seu estado interno, se consegue ou não lidar com requests. Uma solução comum é que essa informação seja parte da API do serviço.

> **Pattern: Health check API**
>
> Um serviço deve expor um endpoint em sua API que descreve sua saúde.
>
> Chris Richardson, no livro [Microservices Patterns](https://www.manning.com/books/microservices-patterns) (RICHARDSON, 2018a) 

Um Service Registry como o Eureka pode ser configurado para invocar a Health Check API de um serviço, de maneira a descobrir quais instâncias estão realmente disponíveis. Orquestradores de containers como o Kubernetes podem usar essa informação de um serviço para matar instâncias pouco saudáveis, criando novas. Ferramentas de monitoramento podem prover um dashboard com a saúde das instâncias e até disparar alertas.

No ecossistema Spring, o projeto [Spring Boot Actuator](https://docs.spring.io/spring-boot/docs/current/reference/html/production-ready-features.html) provê essa API na URL `/actuator/health`. Se o serviço estiver disponível, é retornado `{ "status": "UP" }` no corpo do response com um status `200 OK`. Já se houver indisponibilidade, é retornado `{ "status": "DOWN" }`, com um status `503 Service Unavailable`.

O Spring Boot Actuator executa uma série de checagens baseadas na infraestrutura usada pelo serviço. Se for usado um BD relacional, haverá checagem que executa um consulta de teste. Se for usado um Message Broker, é testada uma conexão. É possível adicionar checagens customizadas implementando a interface `HealthIndicator`.

## Métricas

No livro [Microservices Patterns](https://www.manning.com/books/microservices-patterns) (RICHARDSON, 2018a), Chris Richardson indica que um sistema de monitoramento deve agregar métricas, de maneira a prover informações críticas sobre a saúde de uma aplicação. Essa métricas podem ser de infraestrutura, como informações sobre o uso de CPU, de memória e de disco. Mas também podem ser métricas da própria aplicação, como o número de pedidos realizados ou de pagamentos confirmados.

> **Pattern: Application metrics**
>
> Os serviços enviam métricas a um servidor central que provê agregação, visualização e alertas.
>
> Chris Richardson, no livro [Microservices Patterns](https://www.manning.com/books/microservices-patterns) (RICHARDSON, 2018a) 

## Expondo endpoints do Spring Boot Actuator

Adicione uma dependência ao _starter_ do Spring Boot Actuator:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

O módulo `eats-application` do monólito já tem essa dependência.

Essa dependência deve ser adicionada ao `pom.xml` dos projetos:

- `fj33-eats-pagamento-service`
- `fj33-eats-distancia-service`
- `fj33-eats-nota-fiscal-service`
- `fj33-api-gateway`
- `fj33-service-registry`
- `fj33-config-server`

Adicione, ao `application.properties` do `config-repo`, que será aplicado aos clientes do Config Server, uma configuração para expôr todos os _endpoints_ disponíveis no Actuator:

####### config-repo/application.properties

```properties
management.endpoints.web.exposure.include=*
```

Observação: como estamos usando um repositório Git local, não há a necessidade de comitar as mudanças no arquivo anterior.

A configuração anterior será aplicada aos clientes do Config Server, que são os seguintes:

- monólito
- serviço de pagamentos
- serviço de distância
- serviço de nota fiscal
- API Gateway

Para impedir que as requisições a endereços do Actuator no API Gateway acabem enviadas para o monólito, faça a configuração a seguir:

####### fj33-api-gateway/src/main/resources/application.properties

```properties
zuul.routes.actuator.path=/actuator/**
zuul.routes.actuator.url=forward:/actuator
```

A configuração anterior deve ser feita antes da rota "coringa", que redirecionar tudo para o monólito.

Exponha também todos os endpoints do Actuator no `config-server` e no `service-registry`:

####### fj33-config-server/src/main/resources/application.properties

```properties
management.endpoints.web.exposure.include=*
```

e

####### fj33-service-registry/src/main/resources/application.properties

```properties
management.endpoints.web.exposure.include=*
```

Reinicie os serviços e explore os endpoints do Actuator.

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

## Exercício: Health Check API com Spring Boot Actuator

1. Faça checkout da branch `cap13-health-check-api-com-spring-boot-actuator` dos seguintes projetos:

  ```sh
  cd ~/Desktop/fj33-eats-pagamento-service
  git checkout -f cap13-health-check-api-com-spring-boot-actuator

  cd ~/Desktop/fj33-eats-distancia-service
  git checkout -f cap13-health-check-api-com-spring-boot-actuator

  cd ~/Desktop/fj33-eats-nota-fiscal-service
  git checkout -f cap13-health-check-api-com-spring-boot-actuator

  cd ~/Desktop/fj33-api-gateway
  git checkout -f cap13-health-check-api-com-spring-boot-actuator

  cd ~/Desktop/fj33-service-registry
  git checkout -f cap13-health-check-api-com-spring-boot-actuator

  cd ~/Desktop/fj33-config-server
  git checkout -f cap13-health-check-api-com-spring-boot-actuator
  ```

  Dê refresh nos projetos no Eclipse e os reinicie.

2. Explore os endpoints do Spring Boot Actuator, baseando-se nos exemplos da seção anterior.

  Por exemplo, teste a seguinte URL para visualizar um _stream_ (fluxo de dados) com as informações dos circuit breakers do API Gateway:

  http://localhost:9999/actuator/hystrix.stream

  Dica: a URL anterior funciona melhor no Google Chrome.

  Também é possível explorar os links retornados pela seguinte URL, trocando `{porta}` pelas portas dos serviços:

  http://localhost:{porta}/actuator


## Monitorando Circuit Breakers com Hystrix Dashboard

Uma maneira interessante de monitorar o que acontece em uma Arquitetura de Microservices é verificar o seu ponto de entrada: o API Gateway. Nosso API Gateway, o Zuul, usa um Circuit Breaker, o Hystrix, para cada roteamento. Além disso, a API Composition que implementamos no Zuul, que invoca o módulo de Restaurante do Monólito e o serviço de Distância, tem o Hystrix para cada chamada remota.

Como diz Michael Nygard, em seu livro [Release It! Second Edition](https://pragprog.com/book/mnee2/release-it-second-edition) (NYGARD, 2018), o estado dos Circuit Breaker de um sistema é importante para o pessoal de operações e deve ser exposto para monitoramento. A partir das estatísticas de um Circuit Breaker, podemos ter uma ideia do volume de chamadas e dos tempos de resposta.

Muitas informações úteis para monitoramento estão na URL `http://localhost:9999/actuator/hystrix.stream`. Mas não é algo fácil de entender por um humano. Por isso, o time da Netflix criou um projeto que serve como uma visualização do fluxo de estatísticas do Hystrix: o [Hystrix Dashboard](https://github.com/Netflix-Skunkworks/hystrix-dashboard).

![Diagrama anotado do estado de um Circuit Breaker. Da documentação do Hystrix Dashboard: https://github.com/Netflix-Skunkworks/hystrix-dashboard/wiki {w=84}](imagens/13-monitoramento-e-observabilidade/dashboard-annoted-circuit-640.png)

A representação gráfica do Hystrix Dashboard traz as seguintes informações:

- o status do Circuit Breaker, se está Fechado ou Aberto
- o tamanho do círculo representa o volume de chamadas por segundo. Quanto maior, mais chamadas.
- a cor do círculo mostra o estado do Circuit Breaker. Verde significa que o circuito está fechado e operando normalmente. Vermelho significa que o circuito está aberto.
- o gráfico atrás do círculo exibe 2 minutos da evolução no volume de chamadas
- o número de hosts do cluster e a volume de chamadas por segundo de um host específico e do cluster em geral
- a porcentagem de falhas na janela de tempo de 10 segundos
- estatísticas descritivas da latência do último minuto
- em verde, o número de chamadas bem sucedidas
- em laranja, o número de timeouts
- em roxo, o número de chamadas rejeitadas por não haver threads disponíveis
- em vermelho, o número de falhas do serviço remoto (erros 500)
- em azul, o número de chamadas que falharam rápido,  feitas enquanto o circuito estava aberto

<!--@note
  Ben Christensen e outros engenheiros da Netflix tem palestras e vídeos em que mostram o Hystrix Dashboard na prática:

  https://speakerdeck.com/benjchristensen/performance-and-fault-tolerance-for-the-netflix-api-august-2012?slide=39
  https://www.youtube.com/watch?v=zWM7oAbVL4g
-->

## Implementando um visualizador com Hystrix Dashboard

Pelo navegador, abra `https://start.spring.io/`.
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

Extraia o `hystrix-dashboard.zip` e copie a pasta para seu Desktop.

Adicione a anotação `@EnableHystrixDashboard` à classe `HystrixDashboardApplication`:

####### fj33-hystrix-dashboard/src/main/java/br/com/caelum/hystrixdashboard/HystrixDashboardApplication.java

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

No arquivo `application.properties`, modifique a porta para `7777`:

####### fj33-hystrix-dashboard/src/main/resources/application.properties

```properties
server.port=7777
```

## Exercício: Visualizando circuit breakers com o Hystrix Dashboard

1. Abra um Terminal e clone o projeto `fj33-hystrix-dashboard` para o seu Desktop:

  ```sh
  cd ~/Desktop
  git clone https://gitlab.com/aovs/projetos-cursos/fj33-hystrix-dashboard.git
  ```

  No Eclipse, no workspace de microservices, importe o projeto `hystrix-dashboard`, usando o menu _File > Import > Existing Maven Projects_.

  Execute a classe `HystrixDashboardApplication`.

2. Acesse o Hystrix Dashboard, pelo navegador, com a seguinte URL:

  http://localhost:7777/hystrix

  Coloque, na URL, o endpoint de Hystrix Stream Actuator do API Gateway:

  http://localhost:9999/actuator/hystrix.stream

  Clique em _Monitor Stream_.

  Em outra aba, acesse URLs do API Gateway como as que seguem:

  - http://localhost:9999/restaurantes/1, que exibirá o circuit breaker do monolito
  - http://localhost:9999/pagamentos/1, que exibirá o circuit breaker do serviço de pagamentos
  - http://localhost:9999/distancia/restaurantes/mais-proximos/71503510, que exibirá o circuit breaker do serviço de distância
  - http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1, que exibirá os circuit breakers relacionados a composição de chamadas feita no API Gateway

  Veja as informações dos circuit breakers do API Gateway no Hystrix Dashboard.

## Agregando dados dos circuit-breakers com Turbine

Há cenários em que queremos monitorar as métricas dos Circuit Breakers de diversos serviços. Precisamos de um agregador dos diversos streams. É o que faz o projeto [Turbine](https://github.com/Netflix/Turbine), da iniciativa open-source da Netflix.

Segundo a [documentação do Turbine](https://github.com/Netflix/Turbine/wiki), é feito o monitoramento de vários servidores agrupados em _clusters_. O cluster padrão é o default.

Para descobrir os serviços disponíveis, o Turbine pode usar as informações de instâncias disponíveis do Eureka, entre outras estratégias. No nosso caso, a configuração da URL do Eureka deve ser obtida do Config Server.

Então, é feita a coleta dos dados consultando as streams de todos os serviços configurados.

![Turbine como agregador de estatísticas de Circuit Breakers {w=34}](imagens/13-monitoramento-e-observabilidade/agregador-com-turbine.png)

<!--@note
  Ben Christensen e outros engenheiros da Netflix tem um post interessante sobre o lançamento do Turbine como ferramenta open-source:

  https://netflixtechblog.com/hystrix-dashboard-turbine-stream-aggregator-60985a2e51df
-->

## Implementando um agregador com Turbine

Pelo navegador, abra `https://start.spring.io/`.
Em _Project_, mantenha _Maven Project_.
Em _Language_, mantenha _Java_.
Em _Spring Boot_, mantenha a versão padrão.
No trecho de _Project Metadata_, defina:

- `br.com.caelum` em _Group_
- `turbine` em _Artifact_

Mantenha os valores em _More options_.

Mantenha o _Packaging_ como `Jar`.
Mantenha a _Java Version_ em `8`.

Em _Dependencies_, adicione:

- Turbine
- Eureka Client
- Config Client

Clique em _Generate Project_.

Extraia o `turbine.zip` e copie a pasta para seu Desktop.

Adicione as anotações `@EnableDiscoveryClient` e `@EnableTurbine` à classe `TurbineApplication`:

####### fj33-turbine/src/main/java/br/com/caelum/turbine/TurbineApplication.java

```java
@EnableTurbine
@EnableDiscoveryClient
@SpringBootApplication
public class TurbineApplication {

  public static void main(String[] args) {
    SpringApplication.run(TurbineApplication.class, args);
  }

}
```

Não esqueça de ajustar os imports:

```java
import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
import org.springframework.cloud.netflix.turbine.EnableTurbine;
```

No arquivo `application.properties`, modifique a porta para `7776`.

Adicione configurações que aponta para os nomes das aplicações e para o cluster `default`:

####### fj33-turbine/src/main/resources/application.properties

```properties
server.port=7776

turbine.appConfig=apigateway
turbine.clusterNameExpression='default'
```

Defina um arquivo `bootstrap.properties` no diretório de _resources_, configurando o endereço do Config Server:

```properties
spring.application.name=turbine
spring.cloud.config.uri=http://localhost:8888
```

## Agregando baseado em eventos com Turbine Stream

No caso de muitas instâncias de cada serviço, a agregação pode sobrecarregar o Turbine, já que seriam diversos streams de métricas do Hystrix. 

O Turbine Stream inverte a dependência usando um Message Broker: ao invés do Turbine depender de vários serviços, cada serviço envia seus dados para um Message Channel que é lido pelo Turbine Stream no ritmo adequado.

Para isso, cada serviço cujo stream será agregado deve usar a biblioteca `spring-cloud-netflix-hystrix-stream`, que enviará os dados ao Message Channel `springCloudHystrixStream`.

Como o Turbine Stream não precisa saber o endereço das instâncias disponíveis, podemos remover a dependência ao Service Registry.

![Turbine Stream agregando métricas com Mensageria {w=49}](imagens/13-monitoramento-e-observabilidade/agregador-com-turbine-stream.png)

## Turbine Stream

No `pom.xml` do projeto `turbine`, troque a dependência ao starter do Turbine pela do Turbine Stream. Remova a dependência ao starter do Eureka Client. Além disso, adicione o binder do Spring Cloud Stream ao RabbitMQ:

####### fj33-turbine/pom.xml

```xml
<̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
  <̶g̶r̶o̶u̶p̶I̶d̶>̶o̶r̶g̶.̶s̶p̶r̶i̶n̶g̶f̶r̶a̶m̶e̶w̶o̶r̶k̶.̶c̶l̶o̶u̶d̶<̶/̶g̶r̶o̶u̶p̶I̶d̶>̶
  <̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶s̶p̶r̶i̶n̶g̶-̶c̶l̶o̶u̶d̶-̶s̶t̶a̶r̶t̶e̶r̶-̶n̶e̶t̶f̶l̶i̶x̶-̶e̶u̶r̶e̶k̶a̶-̶c̶l̶i̶e̶n̶t̶<̶/̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶
<̶/̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
<̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
  <̶g̶r̶o̶u̶p̶I̶d̶>̶o̶r̶g̶.̶s̶p̶r̶i̶n̶g̶f̶r̶a̶m̶e̶w̶o̶r̶k̶.̶c̶l̶o̶u̶d̶<̶/̶g̶r̶o̶u̶p̶I̶d̶>̶
  <̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶s̶p̶r̶i̶n̶g̶-̶c̶l̶o̶u̶d̶-̶s̶t̶a̶r̶t̶e̶r̶-̶n̶e̶t̶f̶l̶i̶x̶-̶t̶u̶r̶b̶i̶n̶e̶<̶/̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶
<̶/̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-turbine-stream</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-stream-binder-rabbit</artifactId>
</dependency>
```

O status de cada circuit breaker será obtido por meio de eventos no Exchange `springCloudHystrixStream` do RabbitMQ.

Por isso, remova as configurações de aplicações do `application.properties`:

####### fj33-turbine/src/main/resources/application.properties

```properties
t̶u̶r̶b̶i̶n̶e̶.̶a̶p̶p̶C̶o̶n̶f̶i̶g̶=̶a̶p̶i̶g̶a̶t̶e̶w̶a̶y̶
t̶u̶r̶b̶i̶n̶e̶.̶c̶l̶u̶s̶t̶e̶r̶N̶a̶m̶e̶E̶x̶p̶r̶e̶s̶s̶i̶o̶n̶=̶'̶d̶e̶f̶a̶u̶l̶t̶'̶
```

Remove a anotação `@EnableDiscoveryClient` e troque a anotação `@EnableTurbine` por `@EnableTurbineStream` na classe `TurbineApplication`:

####### fj33-turbine/src/main/java/br/com/caelum/turbine/TurbineApplication.java

```java
@EnableTurbineStream
@̶E̶n̶a̶b̶l̶e̶T̶u̶r̶b̶i̶n̶e̶
@̶E̶n̶a̶b̶l̶e̶D̶i̶s̶c̶o̶v̶e̶r̶y̶C̶l̶i̶e̶n̶t̶
@SpringBootApplication
public class TurbineApplication {

  public static void main(String[] args) {
    SpringApplication.run(TurbineApplication.class, args);
  }

}
```

Ajuste os imports da seguinte maneira:

```java
i̶m̶p̶o̶r̶t̶ ̶o̶r̶g̶.̶s̶p̶r̶i̶n̶g̶f̶r̶a̶m̶e̶w̶o̶r̶k̶.̶c̶l̶o̶u̶d̶.̶c̶l̶i̶e̶n̶t̶.̶d̶i̶s̶c̶o̶v̶e̶r̶y̶.̶E̶n̶a̶b̶l̶e̶D̶i̶s̶c̶o̶v̶e̶r̶y̶C̶l̶i̶e̶n̶t̶;̶
i̶m̶p̶o̶r̶t̶ ̶o̶r̶g̶.̶s̶p̶r̶i̶n̶g̶f̶r̶a̶m̶e̶w̶o̶r̶k̶.̶c̶l̶o̶u̶d̶.̶n̶e̶t̶f̶l̶i̶x̶.̶t̶u̶r̶b̶i̶n̶e̶.̶E̶n̶a̶b̶l̶e̶T̶u̶r̶b̶i̶n̶e̶;̶
import org.springframework.cloud.netflix.turbine.stream.EnableTurbineStream;
```

Adicione a dependência ao Hystrix Stream no `pom.xml` do API Gateway:

####### fj33-api-gateway/pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-netflix-hystrix-stream</artifactId>
</dependency>
```

Ajuste o _destination_ do _channel_ `hystrixStreamOutput`, no `application.properties` do API Gateway, por meio da propriedade:

####### fj33-api-gateway/src/main/resources/application.properties

```properties
spring.cloud.stream.bindings.hystrixStreamOutput.destination=springCloudHystrixStream
```

## Exercício: Agregando Circuit Breakers com Turbine Stream

1. Faça o clone do projeto `fj33-turbine` para o seu Desktop:

  ```sh
  cd ~/Desktop/fj33-turbine
  git clone https://gitlab.com/aovs/projetos-cursos/fj33-turbine.git
  ```

  No Eclipse, no workspace de microservices, importe o projeto `turbine`, usando o menu _File > Import > Existing Maven Projects_.

  Execute a classe `TurbineApplication`.

2. Faça o checkout da branch `cap13-turbine-stream` do projeto `fj33-api-gateway`:

  ```sh
  cd ~/Desktop/fj33-api-gateway
  git checkout -f cap13-turbine-stream
  ```

  Dê o refresh do API Gateway no Eclipse e o reinicie.

3. Acesse o Turbine pela URL a seguir:

  http://localhost:7776/turbine.stream

  Em outra janela do navegador, faça algumas chamadas ao API Gateway, como as do exercício anterior.

  Observe, na página do Turbine, um fluxo de dados parecido com:

  ```txt
  : ping
  data: {"reportingHostsLast10Seconds":0,"name":"meta","type":"meta","timestamp":1562070789955}

  : ping
  data: {"reportingHostsLast10Seconds":0,"name":"meta","type":"meta","timestamp":1562070792956}

  : ping
  data: {"rollingCountFallbackSuccess":0,"rollingCountFallbackFailure":0,"propertyValue_circuitBreakerRequestVolumeThreshold":20,"propertyValue_circuitBreakerForceOpen":false,"propertyValue_metricsRollingStatisticalWindowInMilliseconds":10000,"latencyTotal_mean":0,"rollingMaxConcurrentExecutionCount":0,"type":"HystrixCommand","rollingCountResponsesFromCache":0,"rollingCountBadRequests":0,"rollingCountTimeout":0,"propertyValue_executionIsolationStrategy":"THREAD","rollingCountFailure":0,"rollingCountExceptionsThrown":0,"rollingCountFallbackMissing":0,"threadPool":"monolito","latencyExecute_mean":0,"isCircuitBreakerOpen":false,"errorCount":0,"rollingCountSemaphoreRejected":0,"group":"monolito","latencyTotal":{"0":0,"99":0,"100":0,"25":0,"90":0,"50":0,"95":0,"99.5":0,"75":0},"requestCount":0,"rollingCountCollapsedRequests":0,"rollingCountShortCircuited":0,"propertyValue_circuitBreakerSleepWindowInMilliseconds":5000,"latencyExecute":{"0":0,"99":0,"100":0,"25":0,"90":0,"50":0,"95":0,"99.5":0,"75":0},"rollingCountEmit":0,"currentConcurrentExecutionCount":1,"propertyValue_executionIsolationSemaphoreMaxConcurrentRequests":10,"errorPercentage":0,"rollingCountThreadPoolRejected":0,"propertyValue_circuitBreakerEnabled":true,"propertyValue_executionIsolationThreadInterruptOnTimeout":true,"propertyValue_requestCacheEnabled":true,"rollingCountFallbackRejection":0,"propertyValue_requestLogEnabled":true,"rollingCountFallbackEmit":0,"rollingCountSuccess":0,"propertyValue_fallbackIsolationSemaphoreMaxConcurrentRequests":10,"propertyValue_circuitBreakerErrorThresholdPercentage":50,"propertyValue_circuitBreakerForceClosed":false,"name":"RestauranteRestClient#porId(Long)","reportingHosts":1,"propertyValue_executionIsolationThreadPoolKeyOverride":"null","propertyValue_executionIsolationThreadTimeoutInMilliseconds":1000,"propertyValue_executionTimeoutInMilliseconds":1000}
  ```

  Garanta que o Hystrix Dashboard esteja rodando e vá a URL:

  http://localhost:7777/hystrix

  Na URL, use o endereço da stream do Turbine:

  http://localhost:7776/turbine.stream

  Faça algumas chamadas ao API Gateway. Veja os status dos circuit breakers.
  
  Pare os serviços e o monólito e faça mais chamadas ao API Gateway. Veja o resultado no Hystrix Dashboard.

## Rastreamento Distribuído

Digamos que, em uma Arquitetura de Microservices, um request passa a apresentar alta latência. A hipótese de um problema na infraestrutura é rapidamente descartada. O vilão é um de nosso serviços. Mas qual deles? Pode ser o API Gateway ou qualquer um dos serviços pelos quais o request passa.

Não é possível usar um debugger ou profiler para um serviço específico, como é comum em um Monólito. Para investigar e resolver o problema, precisamos de uma maneira de rastrear todas as etapas por onde passa um request, identificando a principal culpada pela demora.

Há ferramentas que permitem esse rastreamento, que implementam um Rastreamento Distribuído (em inglês, **Distributed Tracing**).

Há três conceitos importantes:

- _Trace_: o rastreamento de um request, que pode conter vários spans.
- _Span_: uma etapa do rastreamento, parte de um trace. As informações principais são o nome da operação, o timestamp de início e o de fim.
- _Correlation ID_: uma ID único de um trace que é propagado em cada span.

![Traces e spans {w=42}](imagens/13-monitoramento-e-observabilidade/trace-e-spans.png)

> **Pattern: Distributed tracing**
>
> Atribua a cada request externo um ID único e registre, em um servidor centralizado, como esse request flui pelo sistema de um serviço a outro. O servidor central fornece visualização e análise.
>
> Chris Richardson, no livro [Microservices Patterns](https://www.manning.com/books/microservices-patterns) (RICHARDSON, 2018a) 

Cada serviço tem suas chamadas instrumentadas por alguma biblioteca e as informações são enviadas para um servidor central. Esse servidor central faz análises e permite visualizações e consultas.

Entre os servidores open-source, estão:

- [Zipkin](https://zipkin.io/), lançado em 2012 pelos engenheiros do Twitter
- [Jaeger](https://www.jaegertracing.io/), lançado em 2017 pelos engenheiros do Uber, implementa a API [OpenTracing](https://opentracing.io/), é parte da [Cloud Native Computing Foundation](https://www.cncf.io/) e tem o orquestrador de containers Kubernetes como plataforma 

No ecossistema Spring, a biblioteca de instrumentação é o [Spring Cloud Sleuth](https://spring.io/projects/spring-cloud-sleuth). São instrumentados pontos comuns de entrada e saída de aplicações Spring como Zuul Filters, RestTemplate, clientes Feign e Message Channels.

Para enviar os traces para o Zipkin, basta usar a biblioteca `spring-cloud-starter-zipkin`. Por padrão, os traces são enviados via HTTP para `http://localhost:9411`. Esse endereço pode ser modificado pela configuração `spring.zipkin.baseUrl`. Se o RabbitMQ estiver no Classpath, é possível usá-lo no envio das informações com a configuração `spring.zipkin.sender.type`.

## Exercício: configurando o Zipkin no Docker Compose

1. Para provisionar uma instância do Zipkin, adicione as seguintes configurações ao `docker-compose.yml` do seu Desktop:

  ####### docker-compose.yml

  ```yml
  zipkin:
    image: openzipkin/zipkin
    ports:
      - "9410:9410"
      - "9411:9411"
    depends_on:
      - rabbitmq
    environment:
      RABBIT_URI: "amqp://eats:caelum123@rabbitmq:5672"
  ```

  O `docker-compose.yml` completo, com a configuração do Zipkin, pode ser encontrado em: https://gitlab.com/snippets/1888247

2. Execute o servidor do Zipkin pelo Docker Compose com o comando:

  ```sh
  docker-compose up
  ```

3. Acesse a UI Web do Zipkin pelo navegador através da URL:

  http://localhost:9411/zipkin/

## Enviando informações para o Zipkin com Spring Cloud Sleuth

Adicione uma dependência ao starter do Spring Cloud Zipkin no `pom.xml` do API Gateway:

####### fj33-api-gateway/pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-zipkin</artifactId>
</dependency>
```

Faça o mesmo no `pom.xml` do:

- módulo `eats-application` do monólito
- serviço de pagamentos
- serviço de distância
- serviço de nota fiscal

## Exercício: Distributed Tracing com Spring Cloud Sleuth e Zipkin

1. Vá até a branch `cap13-spring-cloud-sleuth` dos projetos do API Gateway, do Monólito Modular e dos serviços de distância, pagamentos e notas fiscais:

  ```sh
  cd ~/Desktop/fj33-api-gateway
  git checkout -f cap13-spring-cloud-sleuth

  cd ~/Desktop/fj33-eats-monolito-modular
  git checkout -f cap13-spring-cloud-sleuth

  cd ~/Desktop/fj33-eats-pagamento-service
  git checkout -f cap13-spring-cloud-sleuth

  cd ~/Desktop/fj33-eats-distancia-service
  git checkout -f cap13-spring-cloud-sleuth

  cd ~/Desktop/fj33-eats-nota-fiscal-service
  git checkout -f cap13-spring-cloud-sleuth
  ```

  Faça refresh no Eclipse e reinicie os projetos.

2. Por padrão, o Spring Cloud Sleuth faz rastreamento por amostragem de 10% das chamadas. É um bom valor, mas inviável pelo pouco volume de nossos requests.

  Por isso, altere a porcentagem de amostragem para 100%, modificando o arquivo `application.properties` do `config-repo`:

  ####### config-repo/application.properties

  ```properties
  spring.sleuth.sampler.probability=1.0
  ```

3. Reinicie os serviços que foram modificados no passo anterior. Garanta que a UI esteja no ar.

  Faça um novo pedido, até a confirmação do pagamento. Faça o login como dono do restaurante e aprove o pedido. Edite o tipo de cozinha e/ou CEP de um restaurante.

  Vá até a interface Web do Zipkin acessando: http://localhost:9411/zipkin/

  Selecione um serviço em _Service Name_. Então, clique em _Find traces_ e veja os rastreamentos. Clique para ver os detalhes.

  Na aba _Dependencies_, veja um gráfico com as dependências entre os serviços baseadas no uso real (e não apenas em diagramas arquiteturais).

## Spring Boot Admin

Pelo navegador, abra `https://start.spring.io/`.
Em _Project_, mantenha _Maven Project_.
Em _Language_, mantenha _Java_.
Em _Spring Boot_, mantenha a versão padrão.
No trecho de _Project Metadata_, defina:

- `br.com.caelum` em _Group_
- `admin-server` em _Artifact_

Mantenha os valores em _More options_.

Mantenha o _Packaging_ como `Jar`.
Mantenha a _Java Version_ em `8`.

Em _Dependencies_, adicione:

- Spring Boot Admin (Server)
- Config Client
- Eureka Discovery Client

Clique em _Generate Project_.

Extraia o `admin-server.zip` e copie a pasta para seu Desktop.

Adicione a anotação `@EnableAdminServer` à classe `AdminServerApplication`:

####### fj33-admin-server/src/main/java/br/com/caelum/adminserver/AdminServerApplication.java

```java
@EnableAdminServer
@SpringBootApplication
public class AdminServerApplication {

  public static void main(String[] args) {
    SpringApplication.run(AdminServerApplication.class, args);
  }

}
```

Adicione o import:

```java
import de.codecentric.boot.admin.server.config.EnableAdminServer;
```

No arquivo `application.properties`, modifique a porta para `6666`:

####### fj33-admin-server/src/main/resources/application.properties

```properties
server.port=6666
```

Crie um arquivo `bootstrap.properties` no diretório `src/main/resources` do Admin Server, definindo o nome da aplicação e o endereço do Config Server:

```properties
spring.application.name=adminserver
spring.cloud.config.uri=http://localhost:8888
```

## Spring Boot Admin

O [Spring Boot Admin](https://github.com/codecentric/spring-boot-admin) é um projeto open-source da consultoria alemã [codecentric AG](https://www.codecentric.de/), que tem integração com diversas ferramentas do ecossistema Spring, incluindo Spring Boot Actuator e Spring Cloud Netflix Eureka. Dessa forma, é possível exibir todas as instâncias de serviços disponíveis no Eureka Server e visualizar a saúde de cada instância e suas métricas, das técnicas às customizadas.

## Exercício: Visualizando os microservices com Spring Boot Admin

1. Faça clone do projeto `fj33-admin-server`:

  ```sh
  git clone https://gitlab.com/aovs/projetos-cursos/fj33-admin-server.git
  ```

  No Eclipse, no workspace de microservices, importe o projeto `fj33-admin-server`, usando o menu _File > Import > Existing Maven Projects_.

  Execute a classe `AdminServerApplication`.

2. Pelo navegador, acesse a URL:

  http://localhost:6666

  Veja informações sobre as aplicações e instâncias.

  Em _Wallboard_, há uma visualização interessante do status dos serviços.

## Monitoramento x Observabilidade
 
No livro [Distributed Systems Observability](https://learning.oreilly.com/library/view/distributed-systems-observability/9781492033431/) (SRIDHARAN, 2018), Cindy Sridharan indica que Monitoramento trata de conhecer a saúde do sistema e prover alertas, a partir de falhas previsíveis.

Para a autora, Observabilidade é uma propriedade de um sistema que foi projetado, construído, testado e é operado considerando a infinidade de falhas parciais imprevisíveis que podem acontecer em um Sistema Distribuído. Sridharan recomenda que o software seja implementado e testado considerando falhas e facilidade de ser debugado.

São listados três pilares da Observabilidade:

- Event Logs: logs devem ser implementados como um fluxo de eventos que aconteceram no passado. É uma estratégia semelhante à descrita nos [12 fatores](https://12factor.net/) do Heroku.
- Métricas: dados numéricos agregados em intervalos de tempo, que podem ser usados para identificar tendências.
- Traces: ajudam a rastrear a série de etapas pelas quais um request passa em um Sistema Distribuído.

No livro [Microservices Patterns](https://www.manning.com/books/microservices-patterns) (RICHARDSON, 2018a), Chris Richardson lista _patterns_ que auxiliam na Observabilidade de um sistema. Alguns desses estudamos nesse capítulo:

- API de Health Check: expor um recurso que retorna a saúde de um sistema
- Métricas de aplicação: cada serviço provê métricas que são expostas a um servidor de métricas.
- Rastreamento Distribuído: cada request externo passa a ter um ID único que é usado no rastreamento das chamadas entre os serviços.

Mas Richardson lista outros patterns relacionados:

- Agregação de Logs: a atividade de cada serviço é agregada em um servidor central, que provê buscas e alertas.
- Log de auditoria: é mantido um log das ações do usuário.
- Rastreamento de exceções: exceções são centralizadas em um serviço central que remove duplicações e alerta os desenvolvedores.
