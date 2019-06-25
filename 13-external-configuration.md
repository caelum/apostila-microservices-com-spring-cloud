# External Configuration

## Exercício: implementando um Config Server

1. Pelo navegador, abra `https://start.spring.io/`.
  Em _Project_, mantenha _Maven Project_.
  Em _Language_, mantenha _Java_.
  Em _Spring Boot_, mantenha a versão padrão.
  No trecho de _Project Metadata_, defina:

  - `br.com.caelum` em _Group_
  - `config-server` em _Artifact_

  Mantenha os valores em _More options_.
 
  Mantenha o _Packaging_ como `Jar`.
  Mantenha a _Java Version_ em `8`.

  Em _Dependencies_, adicione:

  - Config Server

  Clique em _Generate Project_.
2. Extraia o `config-server.zip` e copie a pasta para seu Desktop.
3. No Eclipse, no workspace de microservices, importe o projeto `config-server`, usando o menu _File > Import > Existing Maven Projects_.
4. Adicione a anotação `@EnableConfigServer` à classe `ConfigServerApplication`:

  ####### config-server/src/main/java/br/com/caelum/configserver/ConfigServerApplication.java

  ```java
  @EnableConfigServer
  @SpringBootApplication
  public class ConfigServerApplication {

    public static void main(String[] args) {
      SpringApplication.run(ServiceRegistryApplication.class, args);
    }

  }
  ```

  Adicione o import:

  ```java
  import org.springframework.cloud.config.server.EnableConfigServer;
  ```

5. No arquivo `application.properties`, modifique a porta para `8888`, defina `configserver` como _application name_ e configure o _profile_ para `native`, que obtém os arquivos de configuração de um sistema de arquivos ou do próprio classpath.

  Nossos arquivos de configuração ficarão no diretório `src/main/resources/configs`, sendo copiados para a raiz do JAR e, em _runtime_, disponível pelo classpath. Portanto, configure a propriedade `spring.cloud.config.server.native.searchLocations` para apontar para esse diretório.

  ####### config-server/src/main/resources/application.properties

  ```properties
  server.port=8888
  spring.application.name=configserver

  spring.profiles.active=native
  spring.cloud.config.server.native.searchLocations=classpath:/configs
  ```

6. Crie o _Folder_ `configs` dentro de `src/main/resources/configs`. Dentro desse diretório, defina um `application.properties` contendo propriedades comuns à maioria dos serviços, como a URL do Eureka e as credencias do RabbitMQ:

  ```properties
  spring.rabbitmq.username=eats
  spring.rabbitmq.password=caelum123

  eureka.client.serviceUrl.defaultZone=${EUREKA_URI:http://localhost:8761/eureka/}
  ```

7. Execute a classe `ConfigServerApplication`.
