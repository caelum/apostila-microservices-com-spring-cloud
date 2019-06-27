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

## Exercício: configurando Config Clients nos serviços

1. Vamos usar como exemplo a configuração do Config Client no serviço de pagamento. Os passos para os demais serviços serão semelhantes.

  No `pom.xml` de `eats-pagamento-service`, adicione a dependência ao _starter_ do Spring Cloud Config Client:

  ####### eats-pagamento-service/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
  </dependency>
  ```

2. Retire do `application.properties` do serviço de pagamentos as configurações comuns que foram definidas no Config Server. Remova também o nome da aplicação:

  ####### eats-pagamento-service/src/main/resources/application.properties

  ```properties
  
  e̶u̶r̶e̶k̶a̶.̶c̶l̶i̶e̶n̶t̶.̶s̶e̶r̶v̶i̶c̶e̶U̶r̶l̶.̶d̶e̶f̶a̶u̶l̶t̶Z̶o̶n̶e̶=̶$̶{̶E̶U̶R̶E̶K̶A̶_̶U̶R̶I̶:̶h̶t̶t̶p̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶8̶7̶6̶1̶/̶e̶u̶r̶e̶k̶a̶/̶}̶

  s̶p̶r̶i̶n̶g̶.̶r̶a̶b̶b̶i̶t̶m̶q̶.̶u̶s̶e̶r̶n̶a̶m̶e̶=̶e̶a̶t̶s̶
  s̶p̶r̶i̶n̶g̶.̶r̶a̶b̶b̶i̶t̶m̶q̶.̶p̶a̶s̶s̶w̶o̶r̶d̶=̶c̶a̶e̶l̶u̶m̶1̶2̶3̶
  ```

3. Crie o arquivo `bootstrap.properties` no diretório `src/main/resources` do serviço de pagamentos. Nesse arquivo, defina o nome da aplicação e a URL do Config Server:

  ####### eats-pagamento-service/src/main/resources/bootstrap.properties

  ```properties
  spring.application.name=pagamentos

  spring.cloud.config.uri=http://localhost:8888
  ```

4. Faça o mesmo para:

  - o API Gateway
  - o monólito
  - o serviço de nota fiscal
  - o serviço de distância

  _Observação: no monólito, as configurações devem ser feitas no módulo `eats-application`._

5. Reinicie todos os serviços. Garanta que a UI esteja no ar. Teste a aplicação, por exemplo, fazendo um pedido até o final e confirmando-o no restaurante. Deve funcionar!
