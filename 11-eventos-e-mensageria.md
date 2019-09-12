# Eventos e mensageria

## Exercício: um serviço de nota fiscal

1. Baixe o projeto do serviço de nota fiscal para seu Desktop usando o Git, com os seguintes comandos:

  ```sh
  cd ~/Desktop
  git clone https://gitlab.com/aovs/projetos-cursos/fj33-eats-nota-fiscal-service.git
  ```

2. Abra o Eclipse, usando o workspace dos microservices.
3. No Eclipse, acesse _File > Import > Existing Maven Projects_ e clique em _Next_. Em _Root Directory_, aponte para o diretório clonado anteriormente.
4. Observe o projeto. Já há configurações para:
  - Clientes REST declarativos com Feign
  - Self registration com Eureka Client

  A classe que gerencia a emissão das notas fiscais é a `ProcessadorDePagamentos` que, dados os ids de um pagamento e de um pedido, obtém os detalhes do pedido do monólito usando o Feign.

  Então, é gerado um XML da nota fiscal usando a biblioteca FreeMarker.

## Exercício: configurando o RabbitMQ no Docker

1. Adicione ao `docker-compose.yml` a configuração de um RabbitMQ na versão 3. Mantenha as portas padrão 5672 para o MOM propriamente dito e 15672 para a UI Web de gerenciamento. Defina o usuário `eats` com a senha `caelum123`:

  ```yaml
  rabbitmq:
    image: "rabbitmq:3-management"
    restart: on-failure
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: eats
      RABBITMQ_DEFAULT_PASS: caelum123
  ```

  O `docker-compose.yml` completo, com a configuração do RabbitMQ, pode ser encontrado em: https://gitlab.com/snippets/1888246

2. Execute novamente o seguinte comando:

  ```sh
  docker-compose up -d
  ```

  Deve aparecer algo como:

  ```txt
  eats-microservices_mysql.pagamento_1 is up-to-date
  eats-microservices_mongo.distancia_1 is up-to-date
  Creating eats-microservices_rabbitmq_1 ... done
  ```

3. Para verificar se está tudo OK, acesse a  pelo navegador a UI de gerenciamento do RabbitMQ:

  http://localhost:15672/

  O username deve ser _eats_ e a senha _caelum123_.

## Exercício: publicando um evento de pagamento confirmado

1. Adicione, no `pom.xml` do serviço de pagamento, o starter do projeto Spring Cloud Stream Rabbit:

  ####### fj33-eats-pagamento-service/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
  </dependency>
  ```

2. Adicione o usuário e senha do RabbitMQ no `application.properties` do serviço de pagamento:

  ####### fj33-eats-pagamento-service/src/main/resources/application.properties

  ```properties
  spring.rabbitmq.username=eats
  spring.rabbitmq.password=caelum123
  ```

3. Crie uma classe `AmqpPagamentoConfig` no pacote `br.com.caelum.eats.pagamento` do serviço de pagamento, anotando-a com `@Configuration`.

  Dentro dessa classe, crie uma interface `PagamentoSource`, que define um método `pagamentosConfirmados`. Esse método deve retornar um `MessageChannel` e tem a anotação `@Output`, indicando que o utilizaremos para enviar mensagens ao MOM.

  Defina também uma constante `PAGAMENTOS_CONFIRMADOS`, que conterá o nome do _exchange_ no RabbitMQ.

  A classe `AmqpPagamentoConfig` também deve ser anotada com `@EnableBinding`, passando como parâmetro a interface `PagamentoSource`:

  ####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/AmqpPagamentoConfig.java

  ```java
  @EnableBinding(PagamentoSource.class)
  @Configuration
  public class AmqpPagamentoConfig {

    public static interface PagamentoSource {
      String PAGAMENTOS_CONFIRMADOS = "pagamentosConfirmados";

      @Output
      MessageChannel pagamentosConfirmados();
    }

  }
  ```

  Os imports são os seguintes:

  ```java
  import org.springframework.cloud.stream.annotation.EnableBinding;
  import org.springframework.cloud.stream.annotation.Output;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.messaging.MessageChannel;

  import br.com.caelum.eats.pagamento.AmqpPagamentoConfig.PagamentoSource;
  ```

4. Crie uma classe `PagamentoConfirmado`, que representará o payload da mensagem, no pacote `br.com.caelum.eats.pagamento` do serviço de pagamento. Essa classe deverá conter o id do pagamento e o id do pedido:

  ####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PagamentoConfirmado.java

  ```java
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  public class PagamentoConfirmado {

    private Long pagamentoId;
    private Long pedidoId;

  }
  ```

5. No mesmo pacote de `eats-pagamento-service`, crie uma classe `NotificadorPagamentoConfirmado`, anotando-a com `@Service`.

  Injete `PagamentoSource` na classe e adicione um método `notificaPagamentoConfirmado`, que recebe um `Pagamento`. Nesse método, crie um `PagamentoConfirmado` e use o `MessageChannel` de `PagamentoSource` para enviá-lo para o MOM:

  ####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/NotificadorPagamentoConfirmado.java

  ```java
  @Service
  @AllArgsConstructor
  public class NotificadorPagamentoConfirmado {

    private PagamentoSource source;

    public void notificaPagamentoConfirmado(Pagamento pagamento) {
      Long pagamentoId = pagamento.getId();
      Long pedidoId = pagamento.getPedidoId();
      PagamentoConfirmado confirmado = new PagamentoConfirmado(pagamentoId, pedidoId);
      source.pagamentosConfirmados().send(MessageBuilder.withPayload(confirmado).build());
    }

  }
  ```

  Faça os imports a seguir:

  ```java
  import org.springframework.messaging.support.MessageBuilder;
  import org.springframework.stereotype.Service;

  import br.com.caelum.eats.pagamento.AmqpPagamentoConfig.PagamentoSource;
  import lombok.AllArgsConstructor;
  ```

6. Em `PagamentoController`, adicione um atributo `NotificadorPagamentoConfirmado` e, no método `confirma`, invoque o método `notificaPagamentoConfirmado`, passando o pagamento que acabou de ser confirmado:

  ####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PagamentoController.java

  ```java
  // anotações ...
  class PagamentoController {

    // outros atributos ...
    private NotificadorPagamentoConfirmado pagamentoConfirmado; // adicionado

    // código omitido ...

    @PutMapping("/{id}")
    public Resource<PagamentoDto> confirma(@PathVariable Long id) {

      Pagamento pagamento = pagamentoRepo.findById(id).orElseThrow(() -> new ResourceNotFoundException());
      pagamento.setStatus(Pagamento.Status.CONFIRMADO);
      pagamentoRepo.save(pagamento);

      pagamentoConfirmado.notificaPagamentoConfirmado(pagamento);

      // código omitido ...
    }

    // código omitido ...

  }
  ```

7. Certifique-se que o serviço de pagamento e demais serviços estejam sendo executados.

  Confirme um pagamento já existente com o cURL:

  ```txt
  curl -X PUT -i http://localhost:9999/pagamentos/1
  ```

  _Observação: para facilitar testes durante o curso, a API de pagamentos permite reconfirmação de pagamentos. Talvez não seja o ideal..._

8. Acesse a UI de gerenciamento do RabbitMQ, pela URL `http://localhost:15672`.

  Veja nos gráficos que algumas mensagens foram publicadas. Veja `pagamentosConfirmados` listado em _Exchange_.

## Exercício: recebendo pagamentos confirmados no serviço de nota fiscal

1. Adicione ao `pom.xml` do `eats-nota-fiscal-service` uma dependência ao starter do projeto Spring Cloud Stream Rabbit:

  ####### fj33-eats-nota-fiscal-service/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
  </dependency>
  ```

2. No `application.properties` do serviço de nota fiscal, defina o usuário e senha do RabbitMQ :

  ####### fj33-eats-nota-fiscal-service/src/main/resources/application.properties

  ```properties
  spring.rabbitmq.username=eats
  spring.rabbitmq.password=caelum123
  ```

3. No pacote `br.com.caelum.eats.notafiscal` do serviço de nota fiscal, crie uma classe `AmqpNotaFiscalConfig` , anotando-a com `@Configuration`.

  Defina a interface `PagamentoSink`, que será para configuração do consumo de mensagens do MOM. Dentro dessa interface, defina o método `pagamentosConfirmados`, com a anotação `@Input` e com `SubscribableChannel` como tipo de retorno.

  O nome do _exchange_ no , que é o mesmo do _source_ do serviço de pagamentos, deve ser definido na constante `PAGAMENTOS_CONFIRMADOS`.

  Não deixe de anotar a classe `AmqpNotaFiscalConfig` com `@EnableBinding`, tendo como parâmetro a interface `PagamentoSink`:

  ####### fj33-eats-nota-fiscal-service/src/main/java/br/com/caelum/eats/notafiscal/AmqpNotaFiscalConfig.java

  ```java
  @EnableBinding(PagamentoSink.class)
  @Configuration
  public class AmqpNotaFiscalConfig {

    public static interface PagamentoSink {
      String PAGAMENTOS_CONFIRMADOS = "pagamentosConfirmados";

      @Input
      SubscribableChannel pagamentosConfirmados();
    }

  }
  ```

  Adicione os imports corretos:

  ```java
  import org.springframework.cloud.stream.annotation.EnableBinding;
  import org.springframework.cloud.stream.annotation.Input;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.messaging.SubscribableChannel;

  import br.com.caelum.notafiscal.AmqpNotaFiscalConfig.PagamentoSink;
  ```

4. Use a anotação `@StreamListener` no método `processaPagamento` da classe `ProcessadorDePagamentos`, passando a constante `PAGAMENTOS_CONFIRMADOS` de `PagamentoSink`:

  ####### fj33-eats-nota-fiscal-service/src/main/java/br/com/caelum/notafiscal/services/ProcessadorDePagamentos.java

  ```java
  // anotações ...
  public class ProcessadorDePagamentos {

    // código omitido ...

    @StreamListener(PagamentoSink.PAGAMENTOS_CONFIRMADOS) // adicionado
    public void processaPagamento(PagamentoConfirmado pagamento) {
      // código omitido ...
    }

  }
  ```

  Faça os imports adequados:

  ```java
  import org.springframework.cloud.stream.annotation.StreamListener;
  import br.com.caelum.notafiscal.AmqpNotaFiscalConfig.PagamentoSink;
  ```

5. Inicie o serviço de nota fiscal executando a classe `EatsNotaFiscalServiceApplication`. Certifique-se que os demais serviços estejam rodando.

  Novamente, use o cURL para :

  ```txt
  curl -X PUT -i http://localhost:9999/pagamentos/1
  ```

  Observe, nos logs do serviço de nota fiscal, o XML da nota emitida. Algo parecido com:

  ```xml
  <xml>
    <loja>314276853</loja>
    <nat_operacao>Almoços, Jantares, Refeições e Pizzas</nat_operacao>
    <pedido>
      <items>
        <item>
          <descricao>Yakimeshi</descricao>
          <un>un</un>
          <codigo>004</codigo>
          <qtde>1</qtde>
          <vlr_unit>21.90</vlr_unit>
          <tipo>P</tipo>
          <class_fiscal>21069090</class_fiscal>
        </item>
        <item>
          <descricao>Coca-Cola Zero Lata 310 ML</descricao>
          <un>un</un>
          <codigo>004</codigo>
          <qtde>2</qtde>
          <vlr_unit>5.90</vlr_unit>
          <tipo>P</tipo>
          <class_fiscal>21069090</class_fiscal>
        </item>
      </items>
    </pedido>
    <cliente>
      <nome>Isabela</nome>
      <tipoPessoa>F</tipoPessoa>
      <contribuinte>9</contribuinte>
      <cpf_cnpj>169.127.587-54</cpf_cnpj>
      <email>isa@gmail.com</email>
      <endereco>Rua dos Bobos, n 0</endereco>
      <complemento>-</numero>
      <cep>10001-202</cep>
    </cliente>
  </xml>
  ```

## Exercício: Competing Consumers e Durable Subscriber com Consumer Groups

1. Pare o serviço de nota fiscal e confirme alguns pagamentos pelo cURL.

  Note que, mesmo com o serviço consumidor parado, a mensagem é publicada no MOM.

  Suba novamente o serviço de nota fiscal e perceba que as mensagens publicadas enquanto o serviço estava fora do ar **não** foram recebidas. Essa é a característica de um _non-durable subscriber_.

2. Execute uma segunda instância do serviço de nota fiscal na porta `9093`.

  No workspace dos microservices, acesse o menu _Run > Run Configurations..._ do Eclipse e clique com o botão direito na configuração `EatsNotaFiscalServiceApplication` e depois clique em _Duplicate_.

  Na configuração `EatsNotaFiscalServiceApplication (1)` que foi criada, acesse a aba _Arguments_ e defina `9093` como a porta da segunda instância, em _VM Arguments_:

  ```txt
  -Dserver.port=9093
  ```

  Clique em _Run_. Nova instância do serviço de nota fiscal no ar!

3. Use o cURL para confirmar um pagamento. Algo como:

 ```txt
  curl -X PUT -i http://localhost:9999/pagamentos/1
  ```

  Note que o XML foi impresso nos logs das duas instâncias, `EatsNotaFiscalServiceApplication` e `EatsNotaFiscalServiceApplication (1)`. Ou seja, todas as instâncias recebem todas as mensagens publicadas no exchange `pagamentosConfirmados` do RabbitMQ.

4. Adicione um nome de grupo para as instâncias do serviço de nota fiscal, definindo a propriedade `spring.cloud.stream.bindings.pagamentosConfirmados.group` no `application.properties`:

  ####### fj33-eats-nota-fiscal-service/src/main/resources/application.properties

  ```properties
  spring.cloud.stream.bindings.pagamentosConfirmados.group=notafiscal
  ```

5. Novamente, confirme alguns pagamentos por meio do cURL.

  Note que o XML é impresso alternadamente nos logs das instâncias `EatsNotaFiscalServiceApplication` e `EatsNotaFiscalServiceApplication (1)`.

  Apenas uma instância do grupo recebe a mensagem, um pattern conhecido como _Competing Consumers_.

6. Pare ambas as instâncias do serviço de nota fiscal. Confirme novos pagamentos usando o cURL.

  Perceba que não ocorre nenhum erro.

  Acesse a UI de gerenciamento do RabbitMQ, na página que lista as _queues_ (filas):

  http://localhost:15672/#/queues

  Perceba que há uma queue para o consumer group chamada `pagamentosConfirmados.notafiscal`, com uma mensagem em _Ready_ para cada confirmação efetuada. Isso indica mensagem de pagamento confirmado foi armazenada na queue.

  Suba uma (ou ambas) as instâncias do `eats-nota-fiscal-service`. Perceba que os XMLs das notas fiscais foram impressos no log.

  Armazenar mensagens publicadas enquanto um subscriber está fora do ar, entregando-as quando sobem novamente, é um pattern conhecido como _Durable Subscriber_.

  Como vimos, os _Consumer Groups_ do Spring Cloud Stream / RabbitMQ implementam os patterns _Competing Consumers_ e  _Durable Subscriber_.


<!--

  Execute todos os serviços, o monólito e a UI e acessa a página de status de um pedido. Será algo como:

  http://localhost:4200/pedidos/6/status

  Clique em F12 para abrir o Dev Tools do navegador. Veja que, nessa página de status, há diversos erro relacionados com WebSocket:

  ```txt
  WebSocket connection to 'ws://localhost:9999/socket/097/ldjsnvcq/websocket' failed: Error during WebSocket handshake: Unexpected response code: 400
  
  Refused to display 'http://localhost:9999/socket/iframe.html#0pn4s3nd' in a frame because it set 'X-Frame-Options' to 'deny'.
  
  Failed to execute 'postMessage' on 'DOMWindow': The target origin provided ('http://localhost:9999') does not match the recipient window's origin ('null').
  
  Failed to load resource: the server responded with a status of 504 ()
  
  Access to XMLHttpRequest at 'http://localhost:9999/socket/097/r52mepqt/xhr?t=1561148554812' from origin 'http://localhost:4200' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.
  ```

  O que acontece é que não configuramos o API Gateway para fazer forwarding de WebSocket. Resolveremos esse problema usando o WebSocket no próprio API Gateway e, no serviço de pedido, publicando um evento de atualização de status.

-->

## Exercício Opcional: Configurações de WebSocket para o API Gateway

1. Adicione a dependência ao starter de WebSocket do Spring Boot no `pom.xml` do API Gateway:

  ####### fj33-api-gateway/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
  </dependency>
  ```

5. Defina a classe `WebSocketConfig` no pacote `br.com.caelum.apigateway` do API Gateway. O código será algo como:

  ####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/WebSocketConfig.java

  ```java
  @EnableWebSocketMessageBroker
  @Configuration
  public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
      registry.enableSimpleBroker("/pedidos", "/parceiros/restaurantes");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
      registry.addEndpoint("/socket").setAllowedOrigins("*").withSockJS();
    }

  }
  ```

  Não esqueça dos imports:

  ```java
  import org.springframework.context.annotation.Configuration;
  import org.springframework.messaging.simp.config.MessageBrokerRegistry;
  import org.springframework.web.socket.config.annotation.EnableWebSocketMessageBroker;
  import org.springframework.web.socket.config.annotation.StompEndpointRegistry;
  import org.springframework.web.socket.config.annotation.WebSocketMessageBrokerConfigurer;
  ```

6. No `application.properties` do API Gateway, defina uma rota local do Zuul, usando forwarding, para as URLs que contém o prefixo `/socket`:

  ####### fj33-api-gateway/src/main/resources/application.properties

  ```properties
  zuul.routes.websocket.path=/socket/**
  zuul.routes.websocket.url=forward:/socket
  ```

  _ATENÇÃO: essa rota deve vir antes da rota `zuul.routes.monolito`, que está definida como `/**`, um padrão que corresponde a qualquer URL._

  Ainda não utilizaremos o WebSocket no API Gateway. Mas está tudo preparado!

## Exercício Opcional: Publicando evento de atualização de pedido no monólito

1. Adicione ao `pom.xml` do módulo `eats-application` do monólito, a dependência ao starter do Spring Cloud Stream Rabbit:

  ####### fj33-eats-monolito-modular/eats/eats-application/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
  </dependency>
  ```

2. Configure usuário e senha do RabbitMQ no `application.properties` do módulo `eats-application` do monólito:

  ####### fj33-eats-monolito-modular/eats/eats-application/src/main/resources/application.properties

  ```properties
  spring.rabbitmq.username=eats
  spring.rabbitmq.password=caelum123
  ```

3. Crie a classe `AmqpPedidoConfig` no pacote `br.com.caelum.eats` do módulo de pedidos do monólito, anotada com `@Configuration`.

  _ATENÇÃO: o pacote deve ser o mencionado anteriormente, para que não sejam necessárias configurações extras no Spring Boot._

  Dentro dessa classe, defina uma interface `AtualizacaoPedidoSource` que define o método `pedidoComStatusAtualizado` que tem o tipo de retorno `MessageChannel` e é anotada com `@Output`.

  Defina o nome do exchange na constante `PEDIDO_COM_STATUS_ATUALIZADO`.

  Anote a classe `AmqpPedidoConfig` com `@EnableBinding`, passando a interface criada.

  ####### fj33-eats-monolito-modular/eats/eats-pedido/src/main/java/br/com/caelum/eats/AmqpPedidoConfig.java

  ```java
  @EnableBinding(AtualizacaoPedidoSource.class)
  @Configuration
  public class AmqpPedidoConfig {

    public static interface AtualizacaoPedidoSource {

      String PEDIDO_COM_STATUS_ATUALIZADO = "pedidoComStatusAtualizado";

      @Output
      MessageChannel pedidoComStatusAtualizado();
    }

  }
  ```

  Seguem os imports:

  ```java
  import org.springframework.cloud.stream.annotation.EnableBinding;
  import org.springframework.cloud.stream.annotation.Output;
  import org.springframework.context.annotation.Configuration;
  import org.springframework.messaging.MessageChannel;

  import br.com.caelum.eats.AmqpPedidoConfig.AtualizacaoPedidoSource;
  ```

4. Adicione um atributo do tipo `AtualizacaoPedidoSource` e utiliza logo depois de atualizar o status do pedido no BD, nos método `atualizaStatus` e `pago` da classe `PedidoController`, do módulo de pedido do monólito:

  ####### fj33-eats-monolito-modular/eats/eats-pedido/src/main/java/br/com/caelum/eats/pedido/PedidoController.java

  ```java
  // anotações ...
  class PedidoController {

    private PedidoRepository repo;
    private AtualizacaoPedidoSource atualizacaoPedido; // adicionado

    // código omitido ...

    @PutMapping("/pedidos/{id}/status")
    public PedidoDto atualizaStatus(@RequestBody Pedido pedido) {
      repo.atualizaStatus(pedido.getStatus(), pedido);

      r̶e̶t̶u̶r̶n̶ ̶n̶e̶w̶ ̶P̶e̶d̶i̶d̶o̶D̶t̶o̶(̶p̶e̶d̶i̶d̶o̶)̶;̶

      // adicionado
      PedidoDto dto = new PedidoDto(pedido);
      atualizacaoPedido.pedidoComStatusAtualizado().send(MessageBuilder.withPayload(dto).build());
      return dto;

    }

    // código omitido ...

    @PutMapping("/pedidos/{id}/pago")
    public void pago(@PathVariable("id") Long id) {
      // código omitido ...
      repo.atualizaStatus(Pedido.Status.PAGO, pedido);

      // adicionado
      PedidoDto dto = new PedidoDto(pedido);
      atualizacaoPedido.pedidoComStatusAtualizado().send(MessageBuilder.withPayload(dto).build());

    }

  }
  ```

  ```java
  import org.springframework.messaging.support.MessageBuilder;
  import br.com.caelum.eats.AmqpPedidoConfig.AtualizacaoPedidoSource;
  ```

5. Inicie todos os serviços, o monólito e a UI. Efetue login como dono de um restaurante e acesse _Pedidos pendentes_.

  Altere o status de um pedido e observe nos logs do monólito a atualização do status seguida por uma conexão com o RabbitMQ, usada para enviar a mensagem de atualização de status do pedido. Algo como:

  ```txt
  Hibernate: update pedido set status=? where id=?
  2019-06-22 08:27:39.793  INFO 12308 --- [nio-8080-exec-4] o.s.a.r.c.CachingConnectionFactory       : Attempting to connect to: [localhost:5672]
  2019-06-22 08:27:39.797  INFO 12308 --- [nio-8080-exec-4] o.s.a.r.c.CachingConnectionFactory       : Created new connection: rabbitConnectionFactory.publisher#679864d6:0/SimpleConnection@1ec86cef [delegate=amqp://eats@127.0.0.1:5672/, localPort= 54602]
  ```

## Exercício Opcional: Recebendo o evento de atualização de status do pedido no API Gateway

1. Adicione o starter do Spring Cloud Stream Rabbit como dependência no `pom.xml` do API Gateway:

  ####### fj33-api-gateway/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
  </dependency>
  ```

2. No pacote `br.com.caelum.apigateway` do API Gateway, defina uma classe que `AmqpApiGatewayConfig`, anotada com `@Configuration` e `@EnableBinding`.

  Dentro dessa classe, defina a interface `AtualizacaoPedidoSink` que deve conter o método `pedidoComStatusAtualizado`, anotado com `@Input` e retornando um `SubscribableChannel`. Essa interface deve conter também a constante `PEDIDO_COM_STATUS_ATUALIZADO`:

  ####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/AmqpApiGatewayConfig.java

  ```java
  @EnableBinding(AtualizacaoPedidoSink.class)
  @Configuration
  public class AmqpApiGatewayConfig {

    public static interface AtualizacaoPedidoSink {

      String PEDIDO_COM_STATUS_ATUALIZADO = "pedidoComStatusAtualizado";

      @Input
      SubscribableChannel pedidoComStatusAtualizado();
    }

  }
  ```

3. No `application.properties` do API Gateway, configure o usuário e senha do RabbitMQ. Defina também um Consumer Group para o exchange `pedidoComStatusAtualizado`:

  ####### fj33-api-gateway/src/main/resources/application.properties

  ```properties
  spring.rabbitmq.username=eats
  spring.rabbitmq.password=caelum123

  spring.cloud.stream.bindings.pedidoComStatusAtualizado.group=apigateway
  ```

  Dessa maneira, teremos um Durable Subscriber com uma queue para armazenar as mensagens, no caso do API Gateway estar fora do ar, e Competing Consumers, no caso de mais de uma instância.

4. Crie as classes a seguir no pacote `br.com.caelum.apigateway.pedido` do API Gateway, que serão utilizadas na desserialização da mensagem do MOM.

  _DICA: basei-se nas classes do módulo de pedido do monólito._

  ####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/pedido/ClienteDto.java

  ```java
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  public class ClienteDto {

    private String nome;

    private String cpf;

    private String email;

    private String telefone;

  }
  ```

  ####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/pedido/EntregaDto.java

  ```java
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  public class EntregaDto {

    private Long id;
    private ClienteDto cliente;
    private String cep;
    private String endereco;
    private String complemento;

  }
  ```

  ####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/pedido/ItemDoCardapioDto.java

  ```java
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  public class ItemDoCardapioDto {

    private Long id;
    private String nome;
    private String descricao;
    private BigDecimal preco;
    private BigDecimal precoPromocional;

  }
  ```

  ####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/pedido/ItemDoPedidoDto.java

  ```java
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  public class ItemDoPedidoDto {

    private Long id;
    private Integer quantidade;
    private String observacao;
    private ItemDoCardapioDto itemDoCardapio;

  }
  ```

  ####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/pedido/PedidoDto.java

  ```java
  @Data
  @AllArgsConstructor
  @NoArgsConstructor
  public class PedidoDto {

    private Long id;
    private LocalDateTime dataHora;
    private String status;
    private RestauranteDto restaurante;
    private EntregaDto entrega;
    private List<ItemDoPedidoDto> itens = new ArrayList<>();

    public BigDecimal getTotal() {
      BigDecimal taxaDeEntregaEmReais = restaurante.getTaxaDeEntregaEmReais();
      BigDecimal total = taxaDeEntregaEmReais != null ? taxaDeEntregaEmReais : BigDecimal.ZERO;
      for (ItemDoPedidoDto item : itens) {
        ItemDoCardapioDto itemDoCardapio = item.getItemDoCardapio();
        BigDecimal precoPromocional = itemDoCardapio.getPrecoPromocional();
        BigDecimal preco = precoPromocional != null ? precoPromocional : itemDoCardapio.getPreco() ;
        total = total.add(preco.multiply(new BigDecimal(item.getQuantidade())));
      }
      return total;
    }

  }
  ```

5. Crie uma classe para receber as mensagens de atualização de status do pedido chamada `StatusDoPedidoService`, no pacote `br.com.caelum.apigateway.pedido` do API Gateway.

  Anote-a com `@Service` e `@AllArgsConstructor`. Defina um atributo do tipo `SimpMessagingTemplate`, cuja instância será injetada pelo Spring.

  Crie um método `pedidoAtualizado`, que recebe um `PedidoDto` como parâmetro. Nesse método, use o  `SimpMessagingTemplate` para enviar o novo status do pedido para o front-end. Se o pedido for pago, envie para uma _destination_ específica para os pedidos pendentes do restaurante.

  Anote o método `pedidoAtualizado` com `@StreamListener`, passando como parâmetro a constante `PEDIDO_COM_STATUS_ATUALIZADO` de `AtualizacaoPedidoSink`.

  ####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/pedido/StatusDoPedidoService.java

  ```java
  @Service
  @AllArgsConstructor
  public class StatusDoPedidoService {

    private SimpMessagingTemplate websocket;

    @StreamListener(AtualizacaoPedidoSink.PEDIDO_COM_STATUS_ATUALIZADO)
    public void pedidoAtualizado(PedidoDto pedido) {

      websocket.convertAndSend("/pedidos/"+pedido.getId()+"/status", pedido);

      if ("PAGO".equals(pedido.getStatus())) {
        websocket.convertAndSend("/parceiros/restaurantes/"+pedido.getRestaurante().getId()+"/pedidos/pendentes", pedido);
      }
    }

  }
  ```

  Certifique-se que fez os imports adequados:

  ```java
  import org.springframework.cloud.stream.annotation.StreamListener;
  import org.springframework.messaging.simp.SimpMessagingTemplate;
  import org.springframework.stereotype.Service;

  import br.com.caelum.apigateway.AmqpApiGatewayConfig.AtualizacaoPedidoSink;

  import lombok.AllArgsConstructor;
  ```

6. Suba todos os serviços, o monólito e o front-end.

  Abra duas janelas de um navegador, de maneira que possa vê-las simultaneamente.

  Em uma das janelas, efetue login como dono de um restaurante e vá até a página de pedidos pendentes.

  Na outra janela do navegador, efetue um pedido no mesmo restaurante, até confirmar o pagamento.

  Perceba que o novo pedido aparece na tela de pedidos pendentes.

  Mude o status do pedido para _Confirmado_ ou _Pronto_ e veja a alteração na tela de acompanhamento do pedido.
  