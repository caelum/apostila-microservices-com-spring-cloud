# Service Registry, Self Registration e Client Side Discovery

## Implementando um Service Registry com o Eureka

Pelo navegador, abra `https://start.spring.io/`.
Em _Project_, mantenha _Maven Project_.
Em _Language_, mantenha _Java_.
Em _Spring Boot_, mantenha a versão padrão.
No trecho de _Project Metadata_, defina:

- `br.com.caelum` em _Group_
- `service-registry` em _Artifact_

Mantenha os valores em _More options_.

Mantenha o _Packaging_ como `Jar`.
Mantenha a _Java Version_ em `8`.

Em _Dependencies_, adicione:

- Eureka Server

Clique em _Generate Project_.

Extraia o `service-registry.zip` e copie a pasta para seu Desktop.

Adicione a anotação `@EnableEurekaServer` à classe `ServiceRegistryApplication`:

####### service-registry/src/main/java/br/com/caelum/serviceregistry/ServiceRegistryApplication.java

```java
@EnableEurekaServer
@SpringBootApplication
public class ServiceRegistryApplication {

  public static void main(String[] args) {
    SpringApplication.run(ServiceRegistryApplication.class, args);
  }

}
```

Adicione o import:

```java
import org.springframework.cloud.netflix.eureka.server.EnableEurekaServer;
```

No arquivo `application.properties`, modifique a porta para `8761`, a porta padrão do Eureka Server, e adicione algumas configurações para que o próprio _service registry_ não se registre nele mesmo.

####### service-registry/src/main/resources/application.properties

```properties
server.port=8761

eureka.client.register-with-eureka=false
eureka.client.fetch-registry=false
logging.level.com.netflix.eureka=OFF
logging.level.com.netflix.discovery=OFF
```

## Exercício: executando o Service Registry

1. Em um Terminal, clone o repositório `fj33-service-registry` para seu Desktop:

  ```sh
  cd ~/Desktop
  git clone https://gitlab.com/aovs/projetos-cursos/fj33-service-registry.git
  ```

2. No Eclipse, no workspace de microservices, importe o projeto `fj33-service-registry`, usando o menu _File > Import > Existing Maven Projects_.

  Execute a classe `ServiceRegistryApplication`.

  Acesse, por um navegador, a URL `http://localhost:8761`. Esse é o Eureka!

  Por enquanto, a seção  _Instances currently registered with Eureka_, que mostra quais serviços estão registrados, está vazia.

## Exercício: self registration do serviço de distância no Eureka Server

1. No `pom.xml` do `eats-distancia-service`, adicione uma dependência ao _Spring Cloud_ na versão `Greenwich.SR2`, em `dependencyManagement`:

  ####### eats-distancia-service/pom.xml

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

2. Adicione o _starter_ do Eureka Client como dependência:
  
  ####### eats-distancia-service/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  ```

3. Adicione a anotação `@EnableDiscoveryClient` à classe `EatsDistanciaApplication`:

  ```java
  @EnableDiscoveryClient // adicionado
  @SpringBootApplication
  public class EatsDistanciaApplication {

    // código omitido ...

  }
  ```

  Adicione o import:

  ```java
  import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
  ```

4. É preciso identificar o serviço de distância para o Eureka Server. Para isso, adicione a propriedade `spring.application.name` ao `application.properties`:

  ####### eats-distancia-service/src/main/resources/application.properties

  ```properties
  spring.application.name=distancia
  ```

5. Pare as instâncias do serviço de distância.

  Execute a _run configuration_ `EatsDistanciaApplication`.

  Acesse o Eureka Server pelo navegador, na URL `http://localhost:8761/`. Observe que a aplicação _DISTANCIA_ aparece entre as instâncias registradas com Eureka.

  Então, execute a segunda instância do serviço de distância, usando a _run configuration_ `EatsDistanciaApplication (1)`.

  Recarregue a página do Eureka Server e note que são indicadas duas instâncias, com suas respectivas portas. Em _Status_, deve aparecer algo como `UP (2) - 192.168.0.90:distancia:9092 , 192.168.0.90:distancia:8082`. 

6. A URL padrão usada pelo Eureka Client é `http://localhost:8761/`.

  Porém, um problema é que não há uma configuração para a URL do Eureka Server que seja customizada nos clientes para ambientes como de testes, homologação e produção.

  É preciso definir essa configuração customizável no `application.properties`:

  ####### eats-distancia-service/src/main/resources/application.properties

  ```properties
  eureka.client.serviceUrl.defaultZone=${EUREKA_URI:http://localhost:8761/eureka/}
  ```

  Dessa maneira, caso seja necessário modificar a URL padrão do Eureka Server, basta definir a variável de ambiente `EUREKA_URI`.

## Exercício: self registration do serviço de pagamento no Eureka Server

1. No `pom.xml` do `eats-pagamento-service`, adicione como dependência o _starter_ do Eureka Client:

  ####### eats-pagamento-service/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  ```

2. Anote a classe `EatsPagamentoServiceApplication` com `@EnableDiscoveryClient`:

  ####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/EatsPagamentoServiceApplication.java

  ```java
  @EnableDiscoveryClient // adicionado
  @EnableFeignClients
  @SpringBootApplication
  public class EatsPagamentoServiceApplication {

  }
  ```

  Lembrando que o import é:

  ```java
  import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
  ```

3. Defina, no `application.properties`, um nome para aplicação, que será usado no Eureka Server. Além disso, adicione a configuração customizável para a URL do Eureka Server:

  ####### eats-pagamento-service/src/main/resources/application.properties

  ```properties
  spring.application.name=pagamentos

  eureka.client.serviceUrl.defaultZone=${EUREKA_URI:http://localhost:8761/eureka/}
  ```

4. Pare o serviço de pagamento.

  Em seguida, execute novamente a classe `EatsPagamentoServiceApplication`.

  Com o serviço em execução, vá até a página do Eureka Server e veja que _PAGAMENTOS_ está entre as instâncias registradas.

## Exercício: self registration do monólito no Eureka Server

1. No `pom.xml` do módulo `eats-application` do monólito, adicione como dependência o _starter_ do Eureka Client:

  ####### fj33-eats-monolito-modular/eats/eats-application/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  ```

2. Anote a classe `EatsApplication` com `@EnableDiscoveryClient`:

  ####### fj33-eats-monolito-modular/eats/eats-application/src/main/java/br/com/caelum/eats/EatsApplication.java

  ```java
  @EnableDiscoveryClient // adicionado
  @SpringBootApplication
  public class EatsApplication {

    // código omitido ...

  }
  ```

  Novamente, lembrando que o import correto:

  ```java
  import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
  ```

3. Defina, no `application.properties`, um nome para aplicação e a URL do Eureka Server:

  ####### fj33-eats-monolito-modular/eats/eats-application/src/main/resources/application.properties

  ```properties
  spring.application.name=monolito

  eureka.client.serviceUrl.defaultZone=${EUREKA_URI:http://localhost:8761/eureka/}
  ```

4. Pare as duas instâncias do monólito.

  A seguir, execute novamente a _run configuration_ `EatsApplication`.

  Observe _MONOLITO_ como instância registrada no Eureka Server.

  Execute a segunda instância do monólito com a _run configuration_ `EatsApplication (1)`.

  Note o registro da segunda instância no Eureka Server, também em _MONOLITO_.

## Exercício: self registration do API Gateway no Eureka Server

1. Adicione como dependência o _starter_ do Eureka Client, No `pom.xml` do `api-gateway`:

  ####### api-gateway/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
  </dependency>
  ```

2. Anote a classe `ApiGatewayApplication` com `@EnableDiscoveryClient`:

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/ApiGatewayApplication.java

  ```java
  @EnableDiscoveryClient // adicionado
  @EnableFeignClients
  @EnableZuulProxy
  @SpringBootApplication
  public class ApiGatewayApplication {
  
    // código omitido ...

  }
  ```

  Lembre do novo import:

  ```java
  import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
  ```

3. No `application.properties`, defina `apigateway` como nome da aplicação. Defina também a URL do Eureka Server:

  ####### api-gateway/src/main/resources/application.properties

  ```properties
  spring.application.name=apigateway

  eureka.client.serviceUrl.defaultZone=${EUREKA_URI:http://localhost:8761/eureka/}
  ```

4. Pare o API Gateway.

  Logo após, execute novamente `ApiGatewayApplication`.

  Note, no Eureka Server, o registro da instância _APIGATEWAY_.

## Exercício: client side discovery no serviço de pagamentos

1. No `application.properties` de `eats-pagamento-service`, apague a lista de servidores de distância do Ribbon, para que seja obtida do Eureka Server e, também, a configuração que desabilita o Eureka Client no Ribbon, que é habilitado por padrão:

  ####### eats-pagamento-service/src/main/resources/application.properties

  ```properties
  m̶o̶n̶o̶l̶i̶t̶o̶.̶r̶i̶b̶b̶o̶n̶.̶l̶i̶s̶t̶O̶f̶S̶e̶r̶v̶e̶r̶s̶=̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶0̶,̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶9̶0̶9̶0̶
  ̶r̶i̶b̶b̶o̶n̶.̶e̶u̶r̶e̶k̶a̶.̶e̶n̶a̶b̶l̶e̶d̶=̶f̶a̶l̶s̶e̶
  ```

2. Com as duas instâncias do monólito no ar, use um cliente REST como o cURL para confirmar um pagamento:

  ```txt
  curl -X PUT -i http://localhost:8081/pagamentos/1
  ```

  Note que os logs são alternados entre `EatsApplication` e `EatsApplication (1)`, quando testamos o comando acima várias vezes.

## Exercício: client side discovery no API Gateway

1. Modifique o `application.properties` do API Gateway, para que o Eureka Client seja habilitado e que não haja mais listas de servidores do Ribbon.

  Limpe as configurações, já que boa parte delas serão obtidas pelas próprias URLs requisitadas e os nomes no Eureka Server.

  Mantenha as que fazem sentido e modifique ligeiramente algumas delas.

  ####### api-gateway/src/main/resources/application.properties

  ```properties
  r̶i̶b̶b̶o̶n̶.̶e̶u̶r̶e̶k̶a̶.̶e̶n̶a̶b̶l̶e̶d̶=̶f̶a̶l̶s̶e̶

  z̶u̶u̶l̶.̶r̶o̶u̶t̶e̶s̶.̶p̶a̶g̶a̶m̶e̶n̶t̶o̶s̶.̶u̶r̶l̶=̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶1̶
  zuul.routes.pagamentos.stripPrefix=false

  z̶u̶u̶l̶.̶r̶o̶u̶t̶e̶s̶.̶d̶i̶s̶t̶a̶n̶c̶i̶a̶.̶p̶a̶t̶h̶=̶/̶d̶i̶s̶t̶a̶n̶c̶i̶a̶/̶*̶*̶
  d̶i̶s̶t̶a̶n̶c̶i̶a̶.̶r̶i̶b̶b̶o̶n̶.̶l̶i̶s̶t̶O̶f̶S̶e̶r̶v̶e̶r̶s̶=̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶2̶,̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶9̶0̶9̶2̶
  configuracao.distancia.service.url=http://distancia

  zuul.routes.local.path=/restaurantes-com-distancia/**
  zuul.routes.local.url=forward:/restaurantes-com-distancia

  z̶u̶u̶l̶.̶r̶o̶u̶t̶e̶s̶.̶m̶o̶n̶o̶l̶i̶t̶o̶.̶p̶a̶t̶h̶=̶/̶*̶*̶
  zuul.routes.monolito=/**
  
  m̶o̶n̶o̶l̶i̶t̶o̶.̶r̶i̶b̶b̶o̶n̶.̶l̶i̶s̶t̶O̶f̶S̶e̶r̶v̶e̶r̶s̶=̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶0̶,̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶9̶0̶9̶0̶
  ```

2. Teste, pelo navegador ou por um cliente REST, as seguintes URLs:

  - `http://localhost:9999/restaurantes/1`, observando se os logs são alternados entre as instâncias do monólito
  - `http://localhost:9999/distancia/restaurantes/mais-proximos/71503510`, e note a alternância entre logs das instâncias do serviço de distância
  - `http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1`, que alterna tanto entre instâncias do monólito como do serviço de distância

## Exercício: client side discovery no monólito

1. Remova, do `application.properties` do módulo `eats-application` do monólito, a lista de servidores de distância do Ribbon e a configuração que desabilita o Eureka Client:

  ####### fj33-eats-monolito-modular/eats/eats-application/src/main/resources/application.properties

  ```properties
  d̶i̶s̶t̶a̶n̶c̶i̶a̶.̶r̶i̶b̶b̶o̶n̶.̶l̶i̶s̶t̶O̶f̶S̶e̶r̶v̶e̶r̶s̶=̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶0̶8̶2̶,̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶9̶0̶9̶2̶
  r̶i̶b̶b̶o̶n̶.̶e̶u̶r̶e̶k̶a̶.̶e̶n̶a̶b̶l̶e̶d̶=̶f̶a̶l̶s̶e̶
  ```

2. Com a UI, os serviços e o monólito no ar, faça login em um restaurante e modifique o tipo de cozinha ou o CEP. Realize essa operação mais de uma vez.

  Perceba que as instâncias do serviço de distância são chamadas alternadamente.
