# Mensageria e Eventos

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

## Publicando um evento de pagamento confirmado com Spring Cloud Stream

Adicione, no `pom.xml` do serviço de pagamento, o starter do projeto Spring Cloud Stream Rabbit:

####### fj33-eats-pagamento-service/pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```

Adicione o usuário e senha do RabbitMQ no `application.properties` do serviço de pagamento:

####### fj33-eats-pagamento-service/src/main/resources/application.properties

```properties
spring.rabbitmq.username=eats
spring.rabbitmq.password=caelum123
```

Crie uma classe `AmqpPagamentoConfig` no pacote `br.com.caelum.eats.pagamento` do serviço de pagamento, anotando-a com `@Configuration`.

Dentro dessa classe, crie uma interface `PagamentoSource`, que define um método `pagamentosConfirmados`, que tem o nome do _exchange_ no RabbitMQ. Esse método deve retornar um `MessageChannel` e tem a anotação `@Output`, indicando que o utilizaremos para enviar mensagens ao MOM.

A classe `AmqpPagamentoConfig` também deve ser anotada com `@EnableBinding`, passando como parâmetro a interface `PagamentoSource`:

####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/AmqpPagamentoConfig.java

```java
@EnableBinding(PagamentoSource.class)
@Configuration
class AmqpPagamentoConfig {

  static interface PagamentoSource {

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

Crie uma classe `PagamentoConfirmado`, que representará o payload da mensagem, no pacote `br.com.caelum.eats.pagamento` do serviço de pagamento. Essa classe deverá conter o id do pagamento e o id do pedido:

####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PagamentoConfirmado.java

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
class PagamentoConfirmado {

  private Long pagamentoId;
  private Long pedidoId;

}
```

Os imports são do Lombok:

```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
```

No mesmo pacote de `eats-pagamento-service`, crie uma classe `NotificadorPagamentoConfirmado`, anotando-a com `@Service`.

Injete `PagamentoSource` na classe e adicione um método `notificaPagamentoConfirmado`, que recebe um `Pagamento`. Nesse método, crie um `PagamentoConfirmado` e use o `MessageChannel` de `PagamentoSource` para enviá-lo para o MOM:

####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/NotificadorPagamentoConfirmado.java

```java
@Service
@AllArgsConstructor
class NotificadorPagamentoConfirmado {

  private PagamentoSource source;

  void notificaPagamentoConfirmado(Pagamento pagamento) {
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

Em `PagamentoController`, adicione um atributo `NotificadorPagamentoConfirmado` e, no método `confirma`, invoque o método `notificaPagamentoConfirmado`, passando o pagamento que acabou de ser confirmado:

####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PagamentoController.java

```java
// anotações ...
class PagamentoController {

  // outros atributos ...
  private NotificadorPagamentoConfirmado pagamentoConfirmado; // adicionado

  // código omitido ...

  @PutMapping("/{id}")
  Resource<PagamentoDto> confirma(@PathVariable Long id) {

    Pagamento pagamento = pagamentoRepo.findById(id).orElseThrow(() -> new ResourceNotFoundException());
    pagamento.setStatus(Pagamento.Status.CONFIRMADO);
    pagamentoRepo.save(pagamento);

    pagamentoConfirmado.notificaPagamentoConfirmado(pagamento); // adicionado

    Long pedidoId = pagamento.getPedidoId();
    pedidoClient.avisaQueFoiPago(pedidoId);

    return new PagamentoDto(pagamento);
  }

  // código omitido ...

}
```

## Recebendo eventos de pagamentos confirmados com Spring Cloud Stream

Adicione ao `pom.xml` do `eats-nota-fiscal-service` uma dependência ao starter do projeto Spring Cloud Stream Rabbit:

####### fj33-eats-nota-fiscal-service/pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```

No `application.properties` do serviço de nota fiscal, defina o usuário e senha do RabbitMQ :

####### fj33-eats-nota-fiscal-service/src/main/resources/application.properties

```properties
spring.rabbitmq.username=eats
spring.rabbitmq.password=caelum123
```

No pacote `br.com.caelum.eats.notafiscal` do serviço de nota fiscal, crie uma classe `AmqpNotaFiscalConfig` , anotando-a com `@Configuration`.

Defina a interface `PagamentoSink`, que será para configuração do consumo de mensagens do MOM. Dentro dessa interface, defina o método `pagamentosConfirmados`, com a anotação `@Input` e com `SubscribableChannel` como tipo de retorno.

O nome do _exchange_ no , que é o mesmo do _source_ do serviço de pagamentos, deve ser definido na constante `PAGAMENTOS_CONFIRMADOS`.

Não deixe de anotar a classe `AmqpNotaFiscalConfig` com `@EnableBinding`, tendo como parâmetro a interface `PagamentoSink`:

####### fj33-eats-nota-fiscal-service/src/main/java/br/com/caelum/eats/notafiscal/AmqpNotaFiscalConfig.java

```java
@EnableBinding(PagamentoSink.class)
@Configuration
class AmqpNotaFiscalConfig {

  static interface PagamentoSink {
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

Use a anotação `@StreamListener` no método `processaPagamento` da classe `ProcessadorDePagamentos`, passando a constante `PAGAMENTOS_CONFIRMADOS` de `PagamentoSink`:

####### fj33-eats-nota-fiscal-service/src/main/java/br/com/caelum/notafiscal/ProcessadorDePagamentos.java

```java
// anotações ...
class ProcessadorDePagamentos {

  // código omitido ...

  @StreamListener(PagamentoSink.PAGAMENTOS_CONFIRMADOS) // adicionado
  void processaPagamento(PagamentoConfirmado pagamento) {
    // código omitido ...
  }

}
```

Faça os imports adequados:

```java
import org.springframework.cloud.stream.annotation.StreamListener;
import br.com.caelum.notafiscal.AmqpNotaFiscalConfig.PagamentoSink;
```

## Exercício: Evento de Pagamento Confirmado com Spring Cloud Stream

1. Faça checkout da branch `cap11-evento-de-pagamento-confirmado-com-spring-cloud-stream` nos projetos do serviços de pagamentos e de nota fiscal:

  ```sh
  cd ~/Desktop/fj33-eats-pagamento-service
  git checkout -f cap11-evento-de-pagamento-confirmado-com-spring-cloud-stream

  cd ~/Desktop/fj33-eats-nota-fiscal-service
  git checkout -f cap11-evento-de-pagamento-confirmado-com-spring-cloud-stream
  ```

  Reinicie o serviço de pagamento.

  Inicie o serviço de nota fiscal executando a classe `EatsNotaFiscalServiceApplication`.

2. Certifique-se que o service registry, o serviço de pagamento, o serviço de nota fiscal e o monólito estejam sendo executados.

  Confirme um pagamento já existente com o cURL:

  ```txt
  curl -X PUT -i http://localhost:8081/pagamentos/1
  ```

  _Observação: para facilitar testes durante o curso, a API de pagamentos permite reconfirmação de pagamentos. Talvez não seja o ideal..._

  Acesse a UI de gerenciamento do RabbitMQ, pela URL `http://localhost:15672`.

  Veja nos gráficos que algumas mensagens foram publicadas. Veja `pagamentosConfirmados` listado em _Exchange_.

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

## Consumer Groups do Spring Cloud Stream

Adicione um nome de grupo para as instâncias do serviço de nota fiscal, definindo a propriedade `spring.cloud.stream.bindings.pagamentosConfirmados.group` no `application.properties`:

####### fj33-eats-nota-fiscal-service/src/main/resources/application.properties

```properties
spring.cloud.stream.bindings.pagamentosConfirmados.group=notafiscal
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

4. Em um Terminal, vá até a branch `cap11-consumer-groups` do serviço de nota fiscal:

  ```sh
  cd ~/Desktop/fj33-eats-nota-fiscal-service
  git checkout -f cap11-consumer-groups
  ```

  Reinicie ambas as instâncias do serviço de nota fiscal.

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

## Configurações de WebSocket para o API Gateway

Adicione a dependência ao starter de WebSocket do Spring Boot no `pom.xml` do API Gateway:

####### fj33-api-gateway/pom.xml

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

Defina a classe `WebSocketConfig` no pacote `br.com.caelum.apigateway` do API Gateway:

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/WebSocketConfig.java

```java
@EnableWebSocketMessageBroker
@Configuration
class WebSocketConfig implements WebSocketMessageBrokerConfigurer {

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

No `application.properties` do API Gateway, defina uma rota local do Zuul, usando forwarding, para as URLs que contém o prefixo `/socket`:

####### fj33-api-gateway/src/main/resources/application.properties

```properties
zuul.routes.websocket.path=/socket/**
zuul.routes.websocket.url=forward:/socket
```

_ATENÇÃO: essa rota deve vir antes da rota `zuul.routes.monolito`, que está definida como `/**`, um padrão que corresponde a qualquer URL._

Ainda não utilizaremos o WebSocket no API Gateway. Mas está tudo preparado!

## Publicando evento de atualização de pedido no monólito

Adicione ao `pom.xml` do módulo `eats-pedido` do monólito, a dependência ao starter do Spring Cloud Stream Rabbit:

####### fj33-eats-monolito-modular/eats/eats-pedido/pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```

Configure usuário e senha do RabbitMQ no `application.properties` do módulo `eats-application` do monólito:

####### fj33-eats-monolito-modular/eats/eats-application/src/main/resources/application.properties

```properties
spring.rabbitmq.username=eats
spring.rabbitmq.password=caelum123
```

Crie a classe `AmqpPedidoConfig` no pacote `br.com.caelum.eats` do módulo de pedidos do monólito, anotada com `@Configuration`.

_ATENÇÃO: o pacote deve ser o mencionado anteriormente, para que não sejam necessárias configurações extras no Spring Boot._

Dentro dessa classe, defina uma interface `AtualizacaoPedidoSource` que define o método `pedidoComStatusAtualizado`, com o nome da exchange no RabbitMQ e que tem o tipo de retorno `MessageChannel` e é anotado com `@Output`.

Anote a classe `AmqpPedidoConfig` com `@EnableBinding`, passando a interface criada.

####### fj33-eats-monolito-modular/eats/eats-pedido/src/main/java/br/com/caelum/eats/AmqpPedidoConfig.java

```java
@EnableBinding(AtualizacaoPedidoSource.class)
@Configuration
public class AmqpPedidoConfig {

  public static interface AtualizacaoPedidoSource {

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

Na classe `PedidoController` do módulo de pedido do monólito, adicione um atributo do tipo `AtualizacaoPedidoSource` e o utilize logo depois de atualizar o status do pedido no BD, nos método `atualizaStatus` e `pago`:

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

## Recebendo o evento de atualização de status do pedido no API Gateway

Adicione o starter do Spring Cloud Stream Rabbit como dependência no `pom.xml` do API Gateway:

####### fj33-api-gateway/pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```

No pacote `br.com.caelum.apigateway` do API Gateway, defina uma classe que `AmqpApiGatewayConfig`, anotada com `@Configuration` e `@EnableBinding`.

Dentro dessa classe, defina a interface `AtualizacaoPedidoSink` que deve conter o método `pedidoComStatusAtualizado`, anotado com `@Input` e retornando um `SubscribableChannel`. Essa interface deve conter também a constante `PEDIDO_COM_STATUS_ATUALIZADO`:

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/AmqpApiGatewayConfig.java

```java
@EnableBinding(AtualizacaoPedidoSink.class)
@Configuration
class AmqpApiGatewayConfig {

  static interface AtualizacaoPedidoSink {

    String PEDIDO_COM_STATUS_ATUALIZADO = "pedidoComStatusAtualizado";

    @Input
    SubscribableChannel pedidoComStatusAtualizado();
  }

}
```

No `application.properties` do API Gateway, configure o usuário e senha do RabbitMQ. Defina também um Consumer Group para o exchange `pedidoComStatusAtualizado`:

####### fj33-api-gateway/src/main/resources/application.properties

```properties
spring.rabbitmq.username=eats
spring.rabbitmq.password=caelum123

spring.cloud.stream.bindings.pedidoComStatusAtualizado.group=apigateway
```

Dessa maneira, teremos um Durable Subscriber com uma queue para armazenar as mensagens, no caso do API Gateway estar fora do ar, e Competing Consumers, no caso de mais de uma instância.

Crie uma classe para receber as mensagens de atualização de status do pedido chamada `StatusDoPedidoService`, no pacote `br.com.caelum.apigateway.pedido` do API Gateway.

Anote-a com `@Service` e `@AllArgsConstructor`. Defina um atributo do tipo `SimpMessagingTemplate`, cuja instância será injetada pelo Spring.

Crie um método `pedidoAtualizado`, que recebe um `Map<String,Object>` como parâmetro. Nesse método, use o  `SimpMessagingTemplate` para enviar o novo status do pedido para o front-end. Se o pedido for pago, envie para uma _destination_ específica para os pedidos pendentes do restaurante.

Anote o método `pedidoAtualizado` com `@StreamListener`, passando como parâmetro a constante `PEDIDO_COM_STATUS_ATUALIZADO` de `AtualizacaoPedidoSink`.

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/pedido/StatusDoPedidoService.java

```java
@Service
@AllArgsConstructor
class StatusDoPedidoService {

  private SimpMessagingTemplate websocket;

  @StreamListener(AtualizacaoPedidoSink.PEDIDO_COM_STATUS_ATUALIZADO)
  void pedidoAtualizado(Map<String, Object> pedido) {

    websocket.convertAndSend("/pedidos/"+pedido.get("id")+"/status", pedido);

    if ("PAGO".equals(pedido.get("status"))) {
      Map<String, Object> restaurante = (Map<String, Object>) pedido.get("restaurante");
      websocket.convertAndSend("/parceiros/restaurantes/"+restaurante.get("id")+"/pedidos/pendentes", pedido);
    }

  }

}  
```

Certifique-se que fez os imports adequados:

```java
import java.util.Map;

import org.springframework.cloud.stream.annotation.StreamListener;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Service;

import br.com.caelum.apigateway.AmqpApiGatewayConfig.AtualizacaoPedidoSink;
import lombok.AllArgsConstructor;
```

## Exercício: notificando novos pedidos e mudança de status do pedido com WebSocket e Eventos

1. Em um Terminal, faça um checkout da branch `cap11-websocket-e-eventos` do monólito, do API Gateway e da UI:

  ```sh
  cd ~/Desktop/fj33-eats-monolito-modular
  git checkout -f cap11-websocket-e-eventos

  cd ~/Desktop/fj33-api-gateway
  git checkout -f cap11-websocket-e-eventos

  cd ~/Desktop/fj33-eats-ui
  git checkout -f cap11-websocket-e-eventos
  ```

2. Rode o comando abaixo para baixar as bibliotecas SockJS e Stomp, que são usadas pela UI:

  ```sh
  cd ~/Desktop/fj33-eats-ui
  npm install
  ```

3. Suba todos os serviços, o monólito e o front-end.

  Abra duas janelas de um navegador, de maneira que possa vê-las simultaneamente.

  Em uma das janelas, efetue login como dono de um restaurante (por exemplo, `longfu`/ `123456`) e vá até a página de pedidos pendentes.

  Na outra janela do navegador, efetue um pedido no mesmo restaurante, até confirmar o pagamento.

  Perceba que o novo pedido aparece na tela de pedidos pendentes.

  Mude o status do pedido para _Confirmado_ ou _Pronto_ e veja a alteração na tela de acompanhamento do pedido.
