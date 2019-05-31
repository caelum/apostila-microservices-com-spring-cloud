# Service Registry, Self Registration e Client Side Discovery

## Exercício: implementando um Service Registry com o Eureka

1. Pelo navegador, abra `https://start.spring.io/`.
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
2. Extraia o `service-registry.zip` e copie a pasta para seu Desktop.
3. No Eclipse, no workspace de microservices, importe o projeto `service-registry`, usando o menu _File > Import > Existing Maven Projects_.
4. Adicione a anotação `@EnableEurekaServer` à classe `ServiceRegistryApplication`:

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

5. No arquivo `application.properties`, modifique a porta para `8761`, a porta padrão do Eureka Server, e adicione algumas configurações para que o próprio _service registry_ não se registre nele mesmo.

  ####### service-registry/src/main/resources/application.properties

  ```properties
  server.port=8761

  eureka.client.register-with-eureka=false
  eureka.client.fetch-registry=false
  logging.level.com.netflix.eureka=OFF
  logging.level.com.netflix.discovery=OFF
  ```

6. Execute a classe `ServiceRegistryApplication`.

  Acesse, por um navegador, a URL `http://localhost:8761`. Esse é o Eureka!

  Por enquanto, a seção  _Instances currently registered with Eureka_, que mostra quais serviços estão registrados, está vazia.

## Exercício: self registration do serviço de distância no Eureka Server

1. No `pom.xml` do `eats-distancia-service`, adicione uma dependência ao _Spring Cloud_ na versão `Greenwich.RELEASE`, em `dependencyManagement`:

  ####### eats-distancia-service/pom.xml

  ```xml
  <dependencyManagement>
    <dependencies>
      <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-dependencies</artifactId>
        <version>Greenwich.RELEASE</version>
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

3. Defina, no `application.properties`, um nome para aplicação, que será usado no Eureka Server:

  ####### eats-pagamento-service/src/main/resources/application.properties

  ```properties
  spring.application.name=pagamentos
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

  }
  ```

  Novamente, lembrando que o import correto:

  ```java
  import org.springframework.cloud.client.discovery.EnableDiscoveryClient;
  ```

3. Defina, no `application.properties`, um nome para aplicação, que será usado no Eureka Server:

  ####### fj33-eats-monolito-modular/eats/eats-application/src/main/resources/application.properties

  ```properties
  spring.application.name=monolito
  ```

4. Pare as duas instâncias do monólito.

  A seguir, execute novamente a _run configuration_ `EatsApplication`.

  Observe _MONOLITO_ como instância registrada no Eureka Server.

  Execute a segunda instância do monólito com a _run configuration_ `EatsApplication (1)`.

  Note o registro da segunda instância no Eureka Server, também em _MONOLITO_.
