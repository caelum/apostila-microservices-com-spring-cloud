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

  A classe que gerencia a emissão das notas fiscais é `ProcessadorDePagamentos` que, dados os ids de um pagamento e de um pedido, obtém os detalhes do pedido do monólito usando o Feign.

  Então, é gerado um XML da nota fiscal usando a biblioteca FreeMarker.

## Exercício: configurando o RabbitMQ no Docker

1. Adicione ao `docker-compose.yaml` a configuração de um RabbitMQ na versão 3. Mantenha as portas padrão 5672 para o MOM propriamente dito e 15672 para a UI Web de gerenciamento. Defina o usuário `eats` com a senha `caelum123`:

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

  ####### eats-pagamento-service/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
  </dependency>
  ```

2. Adicione o usuário e senha do RabbitMQ no `application.properties` do serviço de pagamento:

  ####### eats-pagamento-service/src/main/resources/application.properties

  ```properties
  spring.rabbitmq.username=eats
  spring.rabbitmq.password=caelum123
  ```

3. Crie uma classe `AmqpConfig` no pacote `br.com.caelum.eats.pagamento` do serviço de pagamento, anotando-a com `@Configuration`.

  Dentro dessa classe, crie uma interface `PagamentoSource`, que define um método `pagamentosConfirmados`. Esse método deve retornar um `MessageChannel` e tem a anotação `@Output`, indicando que o utilizaremos para enviar mensagens ao MOM.

  Defina também uma constante `PAGAMENTOS_CONFIRMADOS`, que conterá o nome do _exchange_ no RabbitMQ.

  A classe `AmqpConfig` também deve ser anotada com `@EnableBinding`, passando como parâmetro a interface `PagamentoSource`:

  ####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/AmqpConfig.java

  ```java
  @EnableBinding(PagamentoSource.class)
  @Configuration
  public class AmqpConfig {

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

  import br.com.caelum.eats.pagamento.AmqpConfig.PagamentoSource;
  ```

4. Crie uma classe `PagamentoConfirmado`, que representará o payload da mensagem, no pacote `br.com.caelum.eats.pagamento` do serviço de pagamento. Essa classe deverá conter o id do pagamento e o id do pedido:

  ####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PagamentoConfirmado.java

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

  ####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/NotificadorPagamentoConfirmado.java

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
  import org.springframework.integration.support.MessageBuilder;
  import org.springframework.stereotype.Service;

  import br.com.caelum.eats.pagamento.AmqpConfig.PagamentoSource;
  import lombok.AllArgsConstructor;
  ```

6. Em `PagamentoController`, adicione um atributo `NotificadorPagamentoConfirmado` e, no método `confirma`, invoque o método `notificaPagamentoConfirmado`, passando o pagamento que acabou de ser confirmado:

  ####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PagamentoController.java

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

  ####### eats-nota-fiscal-service/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
  </dependency>
  ```

2. No `application.properties` do serviço de nota fiscal, defina o usuário e senha do RabbitMQ :

  ####### eats-nota-fiscal-service/src/main/resources/application.properties

  ```properties
  spring.rabbitmq.username=eats
  spring.rabbitmq.password=caelum123
  ```

3. No pacote `br.com.caelum.eats.notafiscal` do serviço de nota fiscal, crie uma classe `AmqpConfig` , anotando-a com `@Configuration`.

  Defina a interface `PagamentoSink`, que será para configuração do consumo de mensagens do MOM. Dentro dessa interface, defina o método `pagamentosConfirmados`, com a anotação `@Input` e com `SubscribableChannel` como tipo de retorno.

  O nome do _exchange_ no , que é o mesmo do _source_ do serviço de pagamentos, deve ser definido na constante `PAGAMENTOS_CONFIRMADOS`.

  Não deixe de anotar a classe `AmqpConfig` com `@EnableBinding`, tendo como parâmetro a interface `PagamentoSink`:

  ####### eats-nota-fiscal-service/src/main/java/br/com/caelum/eats/notafiscal/AmqpConfig.java

  ```java
  @EnableBinding(PagamentoSink.class)
  @Configuration
  public class AmqpConfig {

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

  import br.com.caelum.notafiscal.AmqpConfig.PagamentoSink;
  ```

4. Use a anotação `@StreamListener` no método `processaPagamento` da classe `ProcessadorDePagamentos`, passando a constante `PAGAMENTOS_CONFIRMADOS` de `PagamentoSink`:

  ####### eats-nota-fiscal-service/src/main/java/br/com/caelum/notafiscal/services/ProcessadorDePagamentos.java

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
  import br.com.caelum.notafiscal.AmqpConfig.PagamentoSink;
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

  ####### eats-nota-fiscal-service/src/main/resources/application.properties

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
