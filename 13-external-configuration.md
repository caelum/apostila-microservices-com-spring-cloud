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

## Exercício: repositório Git no Config Server

1. Crie um repositório Git no diretório `config-repo` do seu Desktop com os comandos:

  ```sh
  cd Desktop
  mkdir config-repo
  cd config-repo
  git init
  ```

2. Mova o arquivo `application.properties` do diretório `configs` do Config Server, que contém as configurações comuns aos vários serviços, para o repositório `config-repo` e o faça o commit:

  ```sh
  mv ~/Desktop/config-server/src/main/resources/configs/application.properties ~/Desktop/config-repo
  git add .
  git commit -m "versão inicial do application.properties"
  ```

3. Configure o `application.properties` do `config-server` para apontar para o repositório Git:

  ####### config-server/src/main/resources/application.properties

  ```properties
  s̶p̶r̶i̶n̶g̶.̶p̶r̶o̶f̶i̶l̶e̶s̶.̶a̶c̶t̶i̶v̶e̶=̶n̶a̶t̶i̶v̶e̶
  s̶p̶r̶i̶n̶g̶.̶c̶l̶o̶u̶d̶.̶c̶o̶n̶f̶i̶g̶.̶s̶e̶r̶v̶e̶r̶.̶n̶a̶t̶i̶v̶e̶.̶s̶e̶a̶r̶c̶h̶L̶o̶c̶a̶t̶i̶o̶n̶s̶=̶c̶l̶a̶s̶s̶p̶a̶t̶h̶:̶/̶c̶o̶n̶f̶i̶g̶s̶

  spring.cloud.config.server.git.uri: file://${user.home}/Desktop/config-repo
  ```

4. Reinicie os serviços e o monólito. Com a UI no ar, veja se a aplicação continua funcionando!

## Movendo configurações específicas dos serviços para o Config Server

1. Crie um arquivo `monolito.properties` no diretório `config-repo`, que é nosso repositório Git. Passe para esse novo arquivo as configurações de BD e chaves criptográficas, removendo-as do monólito:

  ####### fj33-eats-monolito-modular/eats/eats-application/src/main/resources/application.properties

  ```properties
  #̶D̶A̶T̶A̶S̶O̶U̶R̶C̶E̶ ̶C̶O̶N̶F̶I̶G̶S̶
  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶u̶r̶l̶=̶j̶d̶b̶c̶:̶m̶y̶s̶q̶l̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶/̶e̶a̶t̶s̶?̶c̶r̶e̶a̶t̶e̶D̶a̶t̶a̶b̶a̶s̶e̶I̶f̶N̶o̶t̶E̶x̶i̶s̶t̶=̶t̶r̶u̶e̶
  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶u̶s̶e̶r̶n̶a̶m̶e̶=̶r̶o̶o̶t̶
  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶p̶a̶s̶s̶w̶o̶r̶d̶=̶

  # código omitido ...
  
  #̶J̶W̶T̶ ̶C̶O̶N̶F̶I̶G̶S̶
  j̶w̶t̶.̶s̶e̶c̶r̶e̶t̶ ̶=̶ ̶r̶m̶'̶!̶@̶N̶=̶K̶e̶!̶~̶p̶8̶V̶T̶A̶2̶Z̶R̶K̶~̶n̶M̶D̶Q̶X̶5̶U̶v̶m̶!̶m̶'̶D̶&̶]̶{̶@̶V̶r̶?̶G̶;̶2̶?̶X̶h̶b̶C̶:̶Q̶a̶#̶9̶#̶e̶M̶L̶N̶\̶}̶x̶3̶?̶J̶R̶3̶.̶2̶z̶r̶~̶v̶)̶g̶Y̶F̶^̶8̶\̶:̶8̶>̶:̶X̶f̶B̶:̶W̶w̶7̶5̶N̶/̶e̶m̶t̶9̶Y̶j̶[̶b̶Q̶M̶N̶C̶W̶w̶W̶\̶J̶?̶N̶,̶n̶v̶H̶.̶<̶2̶\̶.̶r̶~̶w̶]̶*̶e̶~̶v̶g̶a̶k̶)̶X̶"̶v̶8̶H̶`̶M̶H̶/̶7̶"̶2̶E̶`̶,̶^̶k̶@̶n̶<̶v̶E̶-̶w̶D̶3̶g̶9̶J̶W̶P̶y̶;̶C̶r̶Y̶*̶.̶K̶d̶2̶_̶D̶]̶)̶=̶>̶<̶D̶?̶Y̶h̶B̶a̶S̶u̶a̶5̶h̶W̶%̶{̶2̶]̶_̶F̶V̶X̶z̶b̶9̶`̶8̶F̶H̶^̶b̶[̶X̶3̶j̶z̶V̶E̶R̶&̶:̶j̶w̶2̶<̶=̶c̶3̶8̶=̶>̶L̶/̶z̶B̶q̶`̶}̶C̶6̶t̶T̶*̶c̶C̶S̶V̶C̶^̶c̶]̶-̶L̶}̶&̶/̶
  j̶w̶t̶.̶e̶x̶p̶i̶r̶a̶t̶i̶o̶n̶ ̶=̶ ̶6̶0̶4̶8̶0̶0̶0̶0̶0̶
  ```

  ####### config-repo/monolito.properties

  ```properties
  # DATASOURCE CONFIGS
  spring.datasource.url=jdbc:mysql://localhost/eats?createDatabaseIfNotExist=true
  spring.datasource.username=root
  spring.datasource.password=
  
  #JWT CONFIGS
  jwt.secret = rm'!@N=Ke!~p8VTA2ZRK~nMDQX5Uvm!m'D&]{@Vr?G;2?XhbC:Qa#9#eMLN\}x3?JR3.2zr~v)gYF^8\:8>:XfB:Ww75N/emt9Yj[bQMNCWwW\J?N,nvH.<2\.r~w]*e~vgak)X"v8H`MH/7"2E`,^k@n<vE-wD3g9JWPy;CrY*.Kd2_D])=><D?YhBaSua5hW%{2]_FVXzb9`8FH^b[X3jzVER&:jw2<=c38=>L/zBq`}C6tT*cCSVC^c]-L}&/
  jwt.expiration = 604800000
  ```

  Não deixe de comitar o novo arquivo no repositório Git:

  ```sh
  git add monolito.properties
  git commit -m "adiciona configurações do monolito"
  ```

2. Mova as configurações do BD do serviço de pagamento para o arquivo `pagamentos.properties` do repositório de configurações:

  ###### eats-pagamento-service/src/main/resources/application.properties

  ```properties
  #̶D̶A̶T̶A̶S̶O̶U̶R̶C̶E̶ ̶C̶O̶N̶F̶I̶G̶S̶
  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶u̶r̶l̶=̶j̶d̶b̶c̶:̶m̶y̶s̶q̶l̶:̶/̶/̶l̶o̶c̶a̶l̶h̶o̶s̶t̶:̶3̶3̶0̶7̶/̶e̶a̶t̶s̶_̶p̶a̶g̶a̶m̶e̶n̶t̶o̶?̶c̶r̶e̶a̶t̶e̶D̶a̶t̶a̶b̶a̶s̶e̶I̶f̶N̶o̶t̶E̶x̶i̶s̶t̶=̶t̶r̶u̶e̶
  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶u̶s̶e̶r̶n̶a̶m̶e̶=̶p̶a̶g̶a̶m̶e̶n̶t̶o̶
  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶s̶o̶u̶r̶c̶e̶.̶p̶a̶s̶s̶w̶o̶r̶d̶=̶p̶a̶g̶a̶m̶e̶n̶t̶o̶1̶2̶3̶
  ```

  ####### config-repo/pagamentos.properties

  ```properties
  #DATASOURCE CONFIGS
  spring.datasource.url=jdbc:mysql://localhost:3307/eats_pagamento?createDatabaseIfNotExist=true
  spring.datasource.username=pagamento
  spring.datasource.password=pagamento123
  ```

  Faça o commit do novo arquivo:

  ```sh
  git add pagamentos.properties
  git commit -m "adiciona configurações do serviço de pagamento"
  ```

3. Transfira as configurações de DB do serviço de distância para um novo arquivo `distancia.properties` do `config-repo`:

  ###### eats-distancia-service/src/main/resources/application.properties

  ```properties
  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶.̶m̶o̶n̶g̶o̶d̶b̶.̶d̶a̶t̶a̶b̶a̶s̶e̶=̶e̶a̶t̶s̶_̶d̶i̶s̶t̶a̶n̶c̶i̶a̶
  s̶p̶r̶i̶n̶g̶.̶d̶a̶t̶a̶.̶m̶o̶n̶g̶o̶d̶b̶.̶p̶o̶r̶t̶=̶2̶7̶0̶1̶8̶
  ```

  ####### config-repo/distancia.properties

  ```properties
  spring.data.mongodb.database=eats_distancia
  spring.data.mongodb.port=27018
  ```

  Comite o novo arquivo no repositório Git:

  ```sh
  git add distancia.properties
  git commit -m "adiciona configurações do serviço de distância"
  ```

4. Faça com que os serviços sejam reiniciados. Acesse a UI e teste as funcionalidades.
