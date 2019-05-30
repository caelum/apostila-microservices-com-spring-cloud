# Service Registry e Self Registration

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
