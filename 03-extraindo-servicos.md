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

