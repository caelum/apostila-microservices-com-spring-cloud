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

6. Inicie o serviço de pagamentos, executando a classe `EatsPagamentoServiceApplication`.

## Exercício: extraindo código de pagamentos do monólito

1. Copie do módulo `eats-pagamento` do monólito, as seguintes classes, colando-as no pacote `br.com.caelum.eats.pagamento` do `eats-pagamento-service`:

  - `Pagamento`
  - `PagamentoController`
  - `PagamentoDto`
  - `PagamentoRepository`

  Também copie, do pacote `br.com.caelum.eats.exception` do módulo `eats-common` do monólito, a seguinte classe:

  - `ResourceNotFoundException`

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
      L̶o̶n̶g̶ ̶p̶e̶d̶i̶d̶o̶I̶d̶ ̶=̶ ̶p̶a̶g̶a̶m̶e̶n̶t̶o̶.̶g̶e̶t̶P̶e̶d̶i̶d̶o̶(̶)̶.̶g̶e̶t̶I̶d̶(̶)̶;̶
      ̶P̶e̶d̶i̶d̶o̶ ̶p̶e̶d̶i̶d̶o̶ ̶=̶ ̶p̶e̶d̶i̶d̶o̶s̶.̶p̶o̶r̶I̶d̶C̶o̶m̶I̶t̶e̶n̶s̶(̶p̶e̶d̶i̶d̶o̶I̶d̶)̶;̶
      ̶p̶e̶d̶i̶d̶o̶.̶s̶e̶t̶S̶t̶a̶t̶u̶s̶(̶P̶e̶d̶i̶d̶o̶.̶S̶t̶a̶t̶u̶s̶.̶P̶A̶G̶O̶)̶;̶
      ̶p̶e̶d̶i̶d̶o̶s̶.̶a̶t̶u̶a̶l̶i̶z̶a̶S̶t̶a̶t̶u̶s̶(̶P̶e̶d̶i̶d̶o̶.̶S̶t̶a̶t̶u̶s̶.̶P̶A̶G̶O̶,̶ ̶p̶e̶d̶i̶d̶o̶)̶;̶
      w̶e̶b̶s̶o̶c̶k̶e̶t̶.̶c̶o̶n̶v̶e̶r̶t̶A̶n̶d̶S̶e̶n̶d̶(̶"̶/̶p̶a̶r̶c̶e̶i̶r̶o̶s̶/̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶/̶"̶+̶p̶e̶d̶i̶d̶o̶.̶g̶e̶t̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶(̶)̶.̶g̶e̶t̶I̶d̶(̶)̶+̶"̶/̶p̶e̶d̶i̶d̶o̶s̶/̶p̶e̶n̶d̶e̶n̶t̶e̶s̶"̶,̶ ̶n̶e̶w̶ ̶P̶e̶d̶i̶d̶o̶D̶t̶o̶(̶p̶e̶d̶i̶d̶o̶)̶)̶;̶
      return new PagamentoDto(pagamento);
    }
  }
  ```

  Ah! Limpe os imports:

  ```java
  i̶m̶p̶o̶r̶t̶ ̶o̶r̶g̶.̶s̶p̶r̶i̶n̶g̶f̶r̶a̶m̶e̶w̶o̶r̶k̶.̶m̶e̶s̶s̶a̶g̶i̶n̶g̶.̶s̶i̶m̶p̶.̶S̶i̶m̶p̶M̶e̶s̶s̶a̶g̶i̶n̶g̶T̶e̶m̶p̶l̶a̶t̶e̶;̶

  i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶p̶e̶d̶i̶d̶o̶.̶P̶e̶d̶i̶d̶o̶;̶
  i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶p̶e̶d̶i̶d̶o̶.̶P̶e̶d̶i̶d̶o̶D̶t̶o̶;̶
  i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶p̶e̶d̶i̶d̶o̶.̶P̶e̶d̶i̶d̶o̶S̶e̶r̶v̶i̶c̶e̶;̶
  ```

5. Certifique-se que a classe `EatsPagamentoServiceApplication` esteja sendo executada.

  Abra um Terminal e teste a criação de um pagamento com o cURL:

  ```sh
  curl -X POST
    -i
    -H 'Content-Type: application/json'
    -d '{ "valor": 51.8, "nome": "JOÃO DA SILVA", "numero": "1111 2222 3333 4444", "expiracao": "2022-07", "codigo": "123", "formaDePagamentoId": 2, "pedidoId": 1 }'
    http://localhost:8081/pagamentos
  ```

  Para que você não precise digitar muito, o comando acima está disponível em: https://gitlab.com/snippets/1859389

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

  <!--  -->

  ```json
  { "id":7, "valor":51.8, "nome":"JOÃO DA SILVA",
    "numero":"1111 2222 3333 4444", "expiracao":"2022-07", "codigo":"123",
    "status":"CRIADO", "formaDePagamentoId":2, "pedidoId":1}
  ```

  Observação: há outros clientes para testar APIs RESTful, como o Postman. Fique à vontade para usá-los. Peça ajuda ao instrutor para instalá-los.

6. Usando o id retornado no passo anterior, teste a confirmação do pagamento pelo cURL, com o seguinte comando:

  ```sh
  curl -X PUT -i http://localhost:8081/pagamentos/7
  ```

  Você deve obter uma resposta semelhante a:

  ```txt
  HTTP/1.1 200 
  Content-Type: application/json;charset=UTF-8
  Transfer-Encoding: chunked
  Date: Tue, 21 May 2019 20:31:08 GMT
  ```

  <!--  -->

  ```json
  { "id":7, "valor":51.80, "nome":"JOÃO DA SILVA",
    "numero":"1111 2222 3333 4444", "expiracao":"2022-07", "codigo":"123",
    "status":"CONFIRMADO","formaDePagamentoId":2,"pedidoId":1}
  ```

  Observe que o status foi modificado para _CONFIRMADO_.

## Exercício: fazendo UI chamar novo serviço de pagamentos

1. Abra o Visual Studio Code e acesse o menu _File > Open Folder..._. Abra o projeto `fj33-eats-ui` do seu Desktop.
2. Adicione uma propriedade `pagamentoUrl`, que aponta para o endereço do novo serviço de pagamentos, no arquivo `environment.ts`:

  ####### fj33-eats-ui/src/environments/environment.ts

  ```typescript
  export const environment = {
    production: false,
    baseUrl: '//localhost:8080'
    , pagamentoUrl: '//localhost:8081' //adicionado
  };
  ```

3. Use a nova propriedade `pagamentoUrl` na classe `PagamentoService`:

  ####### fj33-eats-ui/src/app/services/pagamento.service.ts

  ```typescript
  export class PagamentoService {

    p̶r̶i̶v̶a̶t̶e̶ ̶A̶P̶I̶ ̶=̶ ̶e̶n̶v̶i̶r̶o̶n̶m̶e̶n̶t̶.̶b̶a̶s̶e̶U̶r̶l̶ ̶+̶ ̶'̶/̶p̶a̶g̶a̶m̶e̶n̶t̶o̶s̶'̶;̶
    private API = environment.pagamentoUrl + '/pagamentos';

    // restante do código ...
  }
  ```

4. No `eats-pagamento-service`, trocamos referências às entidades `Pedido` e `FormaDePagamento` pelos respectivos ids. Essa mudança afeta o código do front-end. Faça o ajuste dos ids na classe `PagamentoService`:

  ####### fj33-eats-ui/src/app/services/pagamento.service.ts

  ```typescript
  export class PagamentoService {

    // código omitido ...

    cria(pagamento): Observable<any> {
      this.ajustaIds(pagamento); // adicionado
      return this.http.post(`${this.API}`, pagamento);
    }

    confirma(pagamento): Observable<any> {
      this.ajustaIds(pagamento); // adicionado
      return this.http.put(`${this.API}/${pagamento.id}`, null);
    }

    cancela(pagamento): Observable<any> {
      this.ajustaIds(pagamento); // adicionado
      return this.http.delete(`${this.API}/${pagamento.id}`);
    }

    // adicionado
    private ajustaIds(pagamento) {
      pagamento.formaDePagamentoId = pagamento.formaDePagamentoId || pagamento.formaDePagamento.id;
      pagamento.pedidoId = pagamento.pedidoId || pagamento.pedido.id;
    }

  }
  ```

  O código do método privado `ajustaIds` define as propriedades `formaDePagamentoId` e `pedidoId`, caso ainda não estejam presentes.

5. No componente `PagamentoPedidoComponent`, precisamos fazer ajustes para usar o atributo `pedidoId` do pagamento:

  ####### fj33-eats-ui/src/app/pedido/pagamento/pagamento-pedido.component.ts

  ```typescript
  export class PagamentoPedidoComponent implements OnInit {

    // código omitido ...

    confirmaPagamento() {
      this.pagamentoService.confirma(this.pagamento)
        .̶s̶u̶b̶s̶c̶r̶i̶b̶e̶(̶p̶a̶g̶a̶m̶e̶n̶t̶o̶ ̶=̶>̶ ̶t̶h̶i̶s̶.̶r̶o̶u̶t̶e̶r̶.̶n̶a̶v̶i̶g̶a̶t̶e̶B̶y̶U̶r̶l̶(̶`̶p̶e̶d̶i̶d̶o̶s̶/̶$̶{̶p̶a̶g̶a̶m̶e̶n̶t̶o̶.̶p̶e̶d̶i̶d̶o̶.̶i̶d̶}̶/̶s̶t̶a̶t̶u̶s̶`̶)̶)̶;̶
        .subscribe(pagamento => this.router.navigateByUrl(`pedidos/${pagamento.pedidoId}/status`));
    }

    // restante do código ...

  }
  ```

6. Certifique-se que tanto o monólito como o serviço de pagamentos estejam rodando. Para subir o monólito, a classe `EatsApplication` do módulo `eats-application` deve ser executada. Já para o `eats-pagamento-service`, deve ser executada a classe `EatsPagamentoServiceApplication`.

  Veja também se o front-end está sendo executado. Caso não esteja, abra um Terminal e, na pasta `fj33-eats-ui` do Desktop, execute o comando `ng serve`.

  Acesse `http://localhost:4200` e realize um pedido. Tente criar um pagamento.

  Deve ocorrer um _Erro no Servidor_. O Console do navegador, acessível com F12, deve ter um erro parecido com:

  _Access to XMLHttpRequest at 'http://localhost:8081/pagamentos' from origin 'http://localhost:4200' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource._

  Isso acontece porque precisamos habilitar o CORS no serviço de pagamentos, que está sendo invocado diretamente pelo navegador.

## Exercício: habilitando CORS no serviço de pagamentos

1. Copie a classe `CorsConfig` do módulo `eats-common` do monólito para `eats-pagamento-service`. Ajuste o pacote da classe copiada para `br.com.caelum.eats.pagamento`.

2. Com o monólito, o serviço de pagamentos e o front-end rodando, acesse `http://localhost:4200`.

  Faça um novo pedido, crie e confirme um pagamento. Deve funcionar!

  Note apenas um detalhe: o status do pedido, exibido na tela após a confirmação do pagamento, está _REALIZADO_ e não _PAGO_. Iremos corrigir isso posteriormente.

## Exercício: apagando código de pagamentos do monólito

1. Remova a dependência a `eats-pagamento` do `pom.xml` do módulo `eats-application` do monólito:

  ####### fj33-eats-monolito-modular/eats/eats-application/pom.xml

  ```xml
  <̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
  ̶ ̶ ̶<̶g̶r̶o̶u̶p̶I̶d̶>̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶<̶/̶g̶r̶o̶u̶p̶I̶d̶>̶
  ̶ ̶ ̶<̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶e̶a̶t̶s̶-̶p̶a̶g̶a̶m̶e̶n̶t̶o̶<̶/̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶
  ̶ ̶ ̶<̶v̶e̶r̶s̶i̶o̶n̶>̶$̶{̶p̶r̶o̶j̶e̶c̶t̶.̶v̶e̶r̶s̶i̶o̶n̶}̶<̶/̶v̶e̶r̶s̶i̶o̶n̶>̶
  ̶<̶/̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
  ```

2. No projeto pai dos módulos, o projeto `eats`, remova o módulo `eats-pagamento`  do `pom.xml`:

  ####### fj33-eats-monolito-modular/eats/pom.xml

  ```xml
  <modules>
    <module>eats-admin</module>
    <̶m̶o̶d̶u̶l̶e̶>̶e̶a̶t̶s̶-̶p̶a̶g̶a̶m̶e̶n̶t̶o̶<̶/̶m̶o̶d̶u̶l̶e̶>̶
    <module>eats-restaurante</module>
    <module>eats-pedido</module>
    <module>eats-distancia</module>
    <module>eats-seguranca</module>
    <module>eats-common</module>
    <module>eats-application</module>
  </modules>
  ```

3. Apague o módulo `eats-pagamento` do monólito. Pelo Eclipse, tecle _Delete_ em cima do módulo, selecione a opção _Delete project contents on disk (cannot be undone)_ e clique em _OK_.

## Exercício: criando um microservice de distância

1. Abra `https://start.spring.io/` no navegador.
  Em _Project_, mantenha _Maven Project_.
  Em _Language_, mantenha _Java_.
  Em _Spring Boot_, mantenha a versão padrão.
  No trecho de _Project Metadata_, defina:

  - `br.com.caelum` em _Group_
  - `eats-distancia-service` em _Artifact_

  Clique em _More options_.
  Mantenha o valor em _Name_.
  Apague a _Description_, deixando-a em branco.
  Em _Package Name_, mude para `br.com.caelum.eats.distancia`.

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
2. Descompacte o `eats-distancia-service.zip` para seu Desktop.
4. Importe o projeto `eats-distancia-service`, por meio do menu _File > Import > Existing Maven Projects_ do Ecllipse, no workspace de microservices.
5. Edite o arquivo `src/main/resources/application.properties`, modificando a porta para 8082, apontando para o BD do monólito, além de definir configurações do JPA e de serialização de JSON:

  ####### eats-distancia-service/src/main/resources/application.properties

  ```properties
  server.port = 8082

  #DATASOURCE CONFIGS
  spring.datasource.url=jdbc:mysql://localhost/eats?createDatabaseIfNotExist=true
  spring.datasource.username=<SEU USUARIO>
  spring.datasource.password=<SUA SENHA>

  #JPA CONFIGS
  spring.jpa.hibernate.ddl-auto=validate
  spring.jpa.show-sql=true

  spring.jackson.serialization.fail-on-empty-beans=false
  ```

  Troque `<SEU USUARIO>` e `<SUA SENHA>` pelos valores informados pelo instrutor.

6. Execute a classe `EatsDistanciaServiceApplication`.

## Exercício: extraindo código de distância do monólito

1. Copie para o pacote `br.com.caelum.eats.distancia` do serviço `eats-distancia-service`, as seguintes classes do módulo `eats-distancia` do monólito:

  - `DistanciaService`
  - `RestauranteComDistanciaDto`
  - `RestaurantesMaisProximosController`

  Copie, do pacote `br.com.caelum.eats.exception` do módulo `eats-common` do monólito, a classe:

  - `ResourceNotFoundException`

  Além disso, já antecipando problemas com CORS no front-end, copie do módulo `eats-common` do monólito, para o pacote `br.com.caelum.eats.distancia` de `eats-distancia-service`, a classe:

  - `CorsConfig`

  Há alguns erros de compilação na classe `DistanciaService`, que corrigiremos nos passos seguintes.

2. O motivo de um dos erros de compilação é uma referência à classe `Restaurante` do módulo `eats-restaurante` do monólito.
  Copie essa classe para o pacote `br.com.caelum.eats.distancia` do serviço `eats-distancia-service`. Ajuste o pacote, caso seja necessário.

  Remova, na classe `Restaurante` copiada, a referência à entidade `TipoDeCozinha`, trocando-a pelo id. Remova também a referência à classe `User`.

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/Restaurante.java

  ```java
  // anotações
  public class Restaurante {

    // código omitido ...

    @̶M̶a̶n̶y̶T̶o̶O̶n̶e̶(̶o̶p̶t̶i̶o̶n̶a̶l̶=̶f̶a̶l̶s̶e̶)̶
    ̶p̶r̶i̶v̶a̶t̶e̶ ̶T̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶ ̶t̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶;̶
    private Long tipoDeCozinhaId;

    ̶@̶O̶n̶e̶T̶o̶O̶n̶e̶
    ̶p̶r̶i̶v̶a̶t̶e̶ ̶U̶s̶e̶r̶ ̶u̶s̶e̶r̶;̶

  }
  ```

  Remova os imports não utilizados:

  ```java
  i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶M̶a̶n̶y̶T̶o̶O̶n̶e̶;̶
  ̶i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶O̶n̶e̶T̶o̶O̶n̶e̶;̶

  ̶i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶a̶d̶m̶i̶n̶.̶T̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶;̶
  ̶i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶s̶e̶g̶u̶r̶a̶n̶c̶a̶.̶U̶s̶e̶r̶;̶
  ```

3. Na classe `DistanciaService` de `eats-distancia-service`, remova os imports que referenciam as classes `Restaurante` e `TipoDeCozinha`:

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/DistanciaService.java

  ```java
  ̶i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶a̶d̶m̶i̶n̶.̶T̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶;̶
  i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶.̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶;̶
  ```

  Como a classe `Restaurante` foi copiada para o mesmo pacote de `DistanciaService`, não há a necessidade de importá-la.

  Mas e para `TipoDeCozinha`? Utilizaremos apenas o id. Por isso, modifique o método `restaurantesDoTipoDeCozinhaMaisProximosAoCep` de `DistanciaService`:

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/DistanciaService.java

  ```java
  public List<RestauranteComDistanciaDto> restaurantesDoTipoDeCozinhaMaisProximosAoCep(Long tipoDeCozinhaId, String cep) {
    T̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶ ̶t̶i̶p̶o̶ ̶=̶ ̶n̶e̶w̶ ̶T̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶(̶)̶;̶
    t̶i̶p̶o̶.̶s̶e̶t̶I̶d̶(̶t̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶I̶d̶)̶;̶

    L̶i̶s̶t̶<̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶>̶ ̶a̶p̶r̶o̶v̶a̶d̶o̶s̶D̶o̶T̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶ ̶=̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶.̶f̶i̶n̶d̶A̶l̶l̶B̶y̶A̶p̶r̶o̶v̶a̶d̶o̶A̶n̶d̶T̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶(̶t̶r̶u̶e̶,̶ ̶t̶i̶p̶o̶,̶ ̶L̶I̶M̶I̶T̶)̶.̶g̶e̶t̶C̶o̶n̶t̶e̶n̶t̶(̶)̶;̶
    List<Restaurante> aprovadosDoTipoDeCozinha = restaurantes.findAllByAprovadoAndTipoDeCozinhaId(true, tipoDeCozinhaId, LIMIT).getContent(); // modificado ...

    return calculaDistanciaParaOsRestaurantes(aprovadosDoTipoDeCozinha, cep);
  }
  ```

4. Ainda resta um erro de compilação na classe `DistanciaService`: o uso da classe `RestauranteService`. Poderíamos fazer uma chamada remota, por meio de um cliente REST, ao monólito para obter os dados necessários. Porém, para esse serviço, acessaremos diretamente o BD.

  Por isso, crie uma interface `RestauranteRepository` no pacote `br.com.caelum.eats.distancia` de `eats-distancia-service`, que estende `JpaRepository` do Spring Data Jpa e possui os métodos usados por `DistanciaService`:

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/RestauranteRepository.java

  ```java
  package br.com.caelum.eats.distancia;

  import org.springframework.data.domain.Page;
  import org.springframework.data.domain.Pageable;
  import org.springframework.data.jpa.repository.JpaRepository;

  interface RestauranteRepository extends JpaRepository<Restaurante, Long> {

    Page<Restaurante> findAllByAprovadoAndTipoDeCozinhaId(boolean aprovado, Long tipoDeCozinhaId, Pageable limit);

    Page<Restaurante> findAllByAprovado(boolean aprovado, Pageable limit);

  }
  ```

  Em `DistanciaService`, use `RestauranteRepository` ao invés de `RestauranteService`:

  ####### eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/DistanciaService.java

  ```java
  // anotações ...
  class DistanciaService {

    // código omitido ...

    p̶r̶i̶v̶a̶t̶e̶ ̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶S̶e̶r̶v̶i̶c̶e̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶;̶
    private RestauranteRepository restaurantes;

    // restante do código ...

  }
  ```

  Limpe o import:

  ```java
  i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶.̶R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶S̶e̶r̶v̶i̶c̶e̶;̶
  ```

5. Verifique se a classe `EatsDistanciaServiceApplication` está sendo executada.

  Abra um Terminal e use o cURL para disparar uma chamada ao serviço de distância.

  Para buscar os restaurantes mais próximos ao CEP `71503-510`:

  ```sh
  curl -i http://localhost:8082/restaurantes/mais-proximos/71503510
  ```

  A resposta será algo como:

  ```txt
  HTTP/1.1 200 
  Content-Type: application/json;charset=UTF-8
  Transfer-Encoding: chunked
  Date: Wed, 22 May 2019 18:44:13 GMT
  ```

  <!--  -->

  ```json
  [
    { "restauranteId": 1, "distancia":8.357388557756333824499961338005959987640380859375},
    { "restauranteId": 2, "distancia":8.17018321127992663832628750242292881011962890625}
  ]
  ```

6. (opcional) Use o cURL para buscar os restaurantes mais próximos ao CEP `71503-510` com o tipo de cozinha _Chinesa_ (que tem o id `1`):

  ```sh
  curl -i http://localhost:8082/restaurantes/mais-proximos/71503510/tipos-de-cozinha/1
  ```

7. (opcional) Descubra a distância de uma dado CEP a um restaurante usando o cURL:

  ```sh
  curl -i http://localhost:8082/restaurantes/71503510/restaurante/1
  ```

## Exercício: fazendo UI chamar serviço de distância

1. Abra o projeto `fj33-eats-ui` no Visual Studio Code e defina uma propriedade `distanciaUrl` no arquivo `environment.ts`:

  ####### fj33-eats-ui/src/environments/environment.ts

  ```typescript
  export const environment = {
    production: false,
    baseUrl: '//localhost:8080'
    , pagamentoUrl: '//localhost:8081'
    , distanciaUrl: '//localhost:8082'
  };
  ```

2. Modifique a classe `RestauranteService` para que use `distanciaUrl` nos métodos `maisProximosPorCep`, `maisProximosPorCepETipoDeCozinha`  e `distanciaPorCepEId`:

  ####### fj33-eats-ui/src/app/services/restaurante.service.ts

  ```typescript
  export class RestauranteService {

    private API = environment.baseUrl;
    private DISTANCIA_API = environment.distanciaUrl; // adicionado

    // código omitido ...

    maisProximosPorCep(cep: string): Observable<any> {
      r̶e̶t̶u̶r̶n̶ ̶t̶h̶i̶s̶.̶h̶t̶t̶p̶.̶g̶e̶t̶(̶`̶$̶{̶t̶h̶i̶s̶.̶A̶P̶I̶}̶/̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶/̶m̶a̶i̶s̶-̶p̶r̶o̶x̶i̶m̶o̶s̶/̶$̶{̶c̶e̶p̶}̶`̶)̶;̶
      return this.http.get(`${this.DISTANCIA_API}/restaurantes/mais-proximos/${cep}`); // modificado
    }

    maisProximosPorCepETipoDeCozinha(cep: string, tipoDeCozinhaId: string): Observable<any> {
      r̶e̶t̶u̶r̶n̶ ̶t̶h̶i̶s̶.̶h̶t̶t̶p̶.̶g̶e̶t̶(̶`̶$̶{̶t̶h̶i̶s̶.̶A̶P̶I̶}̶/̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶/̶m̶a̶i̶s̶-̶p̶r̶o̶x̶i̶m̶o̶s̶/̶$̶{̶c̶e̶p̶}̶/̶t̶i̶p̶o̶s̶-̶d̶e̶-̶c̶o̶z̶i̶n̶h̶a̶/̶$̶{̶t̶i̶p̶o̶D̶e̶C̶o̶z̶i̶n̶h̶a̶I̶d̶}̶`̶)̶;̶
      return this.http.get(`${this.DISTANCIA_API}/restaurantes/mais-proximos/${cep}/tipos-de-cozinha/${tipoDeCozinhaId}`); // modificado
    }

    distanciaPorCepEId(cep: string, restauranteId: string): Observable<any> {
      r̶e̶t̶u̶r̶n̶ ̶t̶h̶i̶s̶.̶h̶t̶t̶p̶.̶g̶e̶t̶(̶`̶$̶{̶t̶h̶i̶s̶.̶A̶P̶I̶}̶/̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶/̶$̶{̶c̶e̶p̶}̶/̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶/̶$̶{̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶I̶d̶}̶`̶)̶;̶
      return this.http.get(`${this.DISTANCIA_API}/restaurantes/${cep}/restaurante/${restauranteId}`); // modificado
    }

    // restante do código ...

  }
  ```

3. Garanta que o front-end esteja rodando e acesse `http://localhost:4200`. Busque os restaurantes de um dado CEP, escolha um dos restaurantes retornados e, na tela de detalhes do restaurante, verifique que a distância aparece logo acima da descrição. Deve funcionar!

## Exercício: removendo código de distância do monólito

1. Remova a dependência a `eats-distancia` do `pom.xml` do módulo `eats-application`:

  ####### fj33-eats-monolito-modular/eats/eats-application/pom.xml

  ```xml
  <̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
  ̶ ̶ ̶<̶g̶r̶o̶u̶p̶I̶d̶>̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶<̶/̶g̶r̶o̶u̶p̶I̶d̶>̶
  ̶ ̶ ̶<̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶e̶a̶t̶s̶-̶d̶i̶s̶t̶a̶n̶c̶i̶a̶<̶/̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶
  ̶ ̶ ̶<̶v̶e̶r̶s̶i̶o̶n̶>̶$̶{̶p̶r̶o̶j̶e̶c̶t̶.̶v̶e̶r̶s̶i̶o̶n̶}̶<̶/̶v̶e̶r̶s̶i̶o̶n̶>̶
  ̶<̶/̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
  ```

2. No `pom.xml` do projeto `eats`, o módulo pai, remova a declaração do módulo `eats-distancia`:

  ####### fj33-eats-monolito-modular/eats/pom.xml

  ```xml
  <modules>
    <module>eats-admin</module>
    <module>eats-restaurante</module>
    <module>eats-pedido</module>
    <̶m̶o̶d̶u̶l̶e̶>̶e̶a̶t̶s̶-̶d̶i̶s̶t̶a̶n̶c̶i̶a̶<̶/̶m̶o̶d̶u̶l̶e̶>̶
    <module>eats-seguranca</module>
    <module>eats-common</module>
    <module>eats-application</module>
  </modules>
  ```

3. Apague o código do módulo `eats-distancia` do monólito. Pelo Eclipse, tecle _Delete_ em cima do módulo, selecione a opção _Delete project contents on disk (cannot be undone)_ e clique em _OK_.
