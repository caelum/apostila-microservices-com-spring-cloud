# Extraindo serviços

## Exercício: criando um microservice de pagamentos

1. Vamos criar um workspace do Eclipse separado para os microservices, mantendo aberto o workspace com o monólito. Para isso, clique no ícone do Eclipse da área de trabalho. Em _Workspace_, defina `/home/<usuario-do-curso>/workspace-microservices`, onde `<usuario-do-curso>` é o login do curso.
2. Pelo navegador, abra `https://start.spring.io/`.
  Em _Project_, mantenha _Maven Project_.
  Em _Language_, mantenha _Java_.
  Em _Spring Boot_, mantenha a versão padrão.
  No trecho de _Project Metadata_, defina:

  - `br.com.caelum` em _Group_
  - `eats-pagamento-service` em _Artifact_

  Clique em _More options_.
  Mantenha o valor em _Name_.
  Apague a _Description_, deixando-a em branco.
  Em _Package Name_, mude para `br.com.caelum.eats.pagamento`.

  Mantenha o _Packaging_ como `Jar`.
  Mantenha a _Java Version_ em `8`.

  Em _Dependencies_, adicione:

  - Web
  - Validation
  - DevTools
  - Lombok
  - JPA
  - MySQL

  Clique em _Generate Project_.
3. Extraia o `eats-pagamento-service.zip` e copie a pasta para seu Desktop.
4. No Eclipse, no workspace de microservices, importe o projeto `eats-pagamento-service`, usando o menu _File > Import > Existing Maven Projects_.
5. No arquivo `src/main/resources/application.properties`, modifique a porta para 8081 e, por enquanto, aponte para o mesmo BD do monólito. Defina também algumas outras configurações do JPA e de serialização de JSON.

  ####### eats-pagamento-service/src/main/resources/application.properties

  ```properties
  server.port = 8081

  #DATASOURCE CONFIGS
  spring.datasource.url=jdbc:mysql://localhost/eats?createDatabaseIfNotExist=true
  spring.datasource.username=<SEU USUARIO>
  spring.datasource.password=<SUA SENHA>

  #JPA CONFIGS
  spring.jpa.hibernate.ddl-auto=validate
  spring.jpa.show-sql=true

  spring.jackson.serialization.fail-on-empty-beans=false
  ```

  Não deixe de trocar `<SEU USUARIO>` e `<SUA SENHA>` pelos valores informados pelo instrutor.

6. Inicie o serviço de pagamentos, executando  .

## Exercício: extraindo código de pagamentos do monólito

1. Copie do módulo `eats-pagamento` do monólito, as seguintes classes, colando-as no pacote `br.com.caelum.eats.pagamento` do `eats-pagamento-service`:

  - `Pagamento`
  - `PagamentoController`
  - `PagamentoDto`
  - `PagamentoRepository`

  Também copie a classe `ResourceNotFoundException` do pacote `br.com.caelum.eats.exception`, que está no módulo `eats-common` do monólito.

  Dica: você pode copiar e colar pelo próprio Eclipse.

  Há alguns erros de compilação. Os corrigiremos nos próximos passos.

2. Na classe `Pagamento`, há erros de compilação nas referências às classes `Pedido` e `FormaDePagamento` que são, respectivamente, dos módulos `eats-pedido` e `eats-admin` do monólito.
  Será que devemos colocar dependências Maven a esses módulos? Não parece uma boa, não é mesmo?
  Vamos, então, trocar as referências a essas classes pelos respectivos ids, de maneira a referenciar as raízes dos agregados `Pedido` e `FormaDePagamento`:

  ####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/Pagamento.java

  ```java
  // anotações ...
  class Pagamento {

    // código omitido...

    @̶M̶a̶n̶y̶T̶o̶O̶n̶e̶(̶o̶p̶t̶i̶o̶n̶a̶l̶=̶f̶a̶l̶s̶e̶)̶
    p̶r̶i̶v̶a̶t̶e̶ ̶P̶e̶d̶i̶d̶o̶ ̶p̶e̶d̶i̶d̶o̶;̶

    @Column(nullable=false)
    private Long pedidoId;

    @̶M̶a̶n̶y̶T̶o̶O̶n̶e̶(̶o̶p̶t̶i̶o̶n̶a̶l̶=̶f̶a̶l̶s̶e̶)̶
    p̶r̶i̶v̶a̶t̶e̶ ̶F̶o̶r̶m̶a̶D̶e̶P̶a̶g̶a̶m̶e̶n̶t̶o̶ ̶f̶o̶r̶m̶a̶D̶e̶P̶a̶g̶a̶m̶e̶n̶t̶o̶;̶

    @Column(nullable=false)
    private Long formaDePagamentoId;

  }
  ```

  Ajuste os imports, removendo os desnecessários e adicionando novos:

  ```java
  i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶a̶d̶m̶i̶n̶.̶F̶o̶r̶m̶a̶D̶e̶P̶a̶g̶a̶m̶e̶n̶t̶o̶;̶
  i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶p̶e̶d̶i̶d̶o̶.̶P̶e̶d̶i̶d̶o̶;̶
  i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶M̶a̶n̶y̶T̶o̶O̶n̶e̶;̶

  import javax.persistence.Column; // adicionado ...

  // outros imports ...
  ```

3. Faça algo semelhante ao passo anterior para a classe `PagamentoDto`, referenciando apenas os ids das classes `PedidoDto` e `FormaDePagamento`:

  ####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PagamentoDto.java

  ```java
  // anotações ...
  class PagamentoDto {

    // outros atributos...
    p̶r̶i̶v̶a̶t̶e̶ ̶F̶o̶r̶m̶a̶D̶e̶P̶a̶g̶a̶m̶e̶n̶t̶o̶ ̶f̶o̶r̶m̶a̶D̶e̶P̶a̶g̶a̶m̶e̶n̶t̶o̶;̶
    private Long formaDePagamentoId;

    p̶r̶i̶v̶a̶t̶e̶ ̶P̶e̶d̶i̶d̶o̶D̶t̶o̶ ̶p̶e̶d̶i̶d̶o̶;̶
    private Long pedidoId;

    public PagamentoDto(Pagamento p) {
      this(p.getId(), p.getValor(), p.getNome(), p.getNumero(), p.getExpiracao(), p.getCodigo(), p.getStatus(),
        p̶.̶g̶e̶t̶F̶o̶r̶m̶a̶D̶e̶P̶a̶g̶a̶m̶e̶n̶t̶o̶(̶)̶,̶ ̶n̶e̶w̶ ̶P̶e̶d̶i̶d̶o̶D̶t̶o̶(̶p̶.̶g̶e̶t̶P̶e̶d̶i̶d̶o̶(̶)̶)̶)̶;̶
        p.getFormaDePagamentoId(), p.getPedidoId());
    }

  }
  ```

  Remova os imports desnecessários:

  ```java
  i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶a̶d̶m̶i̶n̶.̶F̶o̶r̶m̶a̶D̶e̶P̶a̶g̶a̶m̶e̶n̶t̶o̶;̶
  i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶p̶e̶d̶i̶d̶o̶.̶P̶e̶d̶i̶d̶o̶D̶t̶o̶;̶
  ```

4. Ao confirmar um pagamento, a classe `PagamentoController` atualiza o status do pedido e envia o pedido atualizado para um WebSocket. Por enquanto, vamos simplificar a confirmação de pagamento, que ficará semelhante a criação e cancelamento: apenas o status do pagamento será atualizado. Depois voltaremos com a atualização do pedido.

  ####### eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PagamentoController.java

  ```java
  // anotações ...
  class PagamentoController {

    private PagamentoRepository pagamentoRepo;
    p̶r̶i̶v̶a̶t̶e̶ ̶P̶e̶d̶i̶d̶o̶S̶e̶r̶v̶i̶c̶e̶ ̶p̶e̶d̶i̶d̶o̶s̶;̶
    p̶r̶i̶v̶a̶t̶e̶ ̶S̶i̶m̶p̶M̶e̶s̶s̶a̶g̶i̶n̶g̶T̶e̶m̶p̶l̶a̶t̶e̶ ̶w̶e̶b̶s̶o̶c̶k̶e̶t̶;̶

    // demais métodos...

    @PutMapping("/{id}")
    public PagamentoDto confirma(@RequestBody Pagamento pagamento) {
      pagamento.setStatus(Pagamento.Status.CONFIRMADO);
      pagamentoRepo.save(pagamento);
      P̶e̶d̶i̶d̶o̶ ̶p̶e̶d̶i̶d̶o̶ ̶=̶ ̶p̶a̶g̶a̶m̶e̶n̶t̶o̶.̶g̶e̶t̶P̶e̶d̶i̶d̶o̶(̶)̶;̶
      p̶e̶d̶i̶d̶o̶.̶s̶e̶t̶S̶t̶a̶t̶u̶s̶(̶P̶e̶d̶i̶d̶o̶.̶S̶t̶a̶t̶u̶s̶.̶P̶A̶G̶O̶)̶;̶
      p̶e̶d̶i̶d̶o̶s̶.̶a̶t̶u̶a̶l̶i̶z̶a̶S̶t̶a̶t̶u̶s̶(̶P̶e̶d̶i̶d̶o̶.̶S̶t̶a̶t̶u̶s̶.̶P̶A̶G̶O̶,̶ ̶p̶e̶d̶i̶d̶o̶)̶;̶
      w̶e̶b̶s̶o̶c̶k̶e̶t̶.̶c̶o̶n̶v̶e̶r̶t̶A̶n̶d̶S̶e̶n̶d̶(̶"̶/̶p̶a̶r̶c̶e̶i̶r̶o̶s̶/̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶/̶"̶+̶p̶e̶d̶i̶d̶o̶.̶g̶e̶t̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶(̶)̶.̶g̶e̶t̶I̶d̶(̶)̶+̶"̶/̶p̶e̶d̶i̶d̶o̶s̶/̶p̶e̶n̶d̶e̶n̶t̶e̶s̶"̶,̶ ̶p̶e̶d̶i̶d̶o̶)̶;̶
      return new PagamentoDto(pagamento);
    }
  }
  ```

  Ah! Limpe os imports:

  ```java
  i̶m̶p̶o̶r̶t̶ ̶o̶r̶g̶.̶s̶p̶r̶i̶n̶g̶f̶r̶a̶m̶e̶w̶o̶r̶k̶.̶m̶e̶s̶s̶a̶g̶i̶n̶g̶.̶s̶i̶m̶p̶.̶S̶i̶m̶p̶M̶e̶s̶s̶a̶g̶i̶n̶g̶T̶e̶m̶p̶l̶a̶t̶e̶;̶

  i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶p̶e̶d̶i̶d̶o̶.̶P̶e̶d̶i̶d̶o̶;̶
  i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶p̶e̶d̶i̶d̶o̶.̶P̶e̶d̶i̶d̶o̶S̶e̶r̶v̶i̶c̶e̶;̶
  ```

5. Certifique-se que a classe `EatsPagamentoServiceApplication` esteja sendo executada.

  Abra um Terminal e teste a criação de um pagamento com o cURL:

  ```sh
  curl -X POST
    -i
    -H 'Content-Type: application/json'
    -d '{ "valor": 51.8, "nome": "ALEXANDRE DA SILVA", "numero": "1111 2222 3333 4444", "expiracao": "2022-07", "codigo": "123", "formaDePagamentoId": 2, "pedidoId": 1 }'
    http://localhost:8081/pagamentos
  ```

  No comando acima, usamos as seguintes opções do cURL:

  - `-X` define o método HTTP a ser utilizado
  - `-i` inclui informações detalhadas da resposta
  - `-H` define um cabeçalho HTTP
  - `-d` define uma representação do recurso a ser enviado ao serviço

  A resposta deve ser algo parecido com:

  ```txt
  HTTP/1.1 200
  Content-Type: application/json;charset=UTF-8
  Transfer-Encoding: chunked
  Date: Tue, 21 May 2019 20:27:10 GMT
  ```

  ```json
  { "id":7, "valor":51.8, "nome":"ALEXANDRE DA SILVA",
    "numero":"1111 2222 3333 4444", "expiracao":"2022-07", "codigo":"123",
    "status":"CRIADO", "formaDePagamentoId":2, "pedidoId":1}
  ```

  Observação: há outros clientes para testar APIs RESTful, como o Postman. Se for o caso, peça ajuda ao instrutor para instalá-los e fique à vontade para usá-los.

6. Usando o id retornado no passo anterior, teste a confirmação do pagamento pelo cURL, com o seguinte comando:

  ```sh
  curl -X PUT http://localhost:8081/pagamentos/7
  ```

  Você deve obter uma resposta semelhante a:

  ```txt
  HTTP/1.1 200 
  Content-Type: application/json;charset=UTF-8
  Transfer-Encoding: chunked
  Date: Tue, 21 May 2019 20:31:08 GMT
  ```

  ```json
  { "id":7, "valor":51.80, "nome":"ALEXANDRE DA SILVA",
    "numero":"1111 2222 3333 4444", "expiracao":"2022-07", "codigo":"123",
    "status":"CONFIRMADO","formaDePagamentoId":2,"pedidoId":1}
  ```

  Observe que o status foi modificado para _CONFIRMADO_.
