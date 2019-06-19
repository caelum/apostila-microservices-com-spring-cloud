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

2. Adicione o usuário e senha do RabbitMQ no `application.propertis` do serviço de pagamento:

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

7. Execute o back-end e o front-end e, então, acesse `http://localhost:4200`.

  Faça um novo pedido, crie e confirme um pagamento.

8. Acesse a UI de gerenciamento do RabbitMQ, pela URL `http://localhost:15672`.

  Veja nos gráficos que algumas mensagens foram publicadas. Em _Exchange_, veja `pagamentosConfirmados`.

