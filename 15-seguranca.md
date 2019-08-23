# Segurança

## Autenticação e Autorização

Grande parte das aplicações tem diferentes perfis de usuário, que tem permissão de acesso a diferentes funcionalidades. Isso é o que chamamos de **Autorização**.

No caso do Caelum Eats, qualquer usuário pode acessar a parte de pedidos. Porém, a parte de administração do sistema, que permite cadastrar tipos de cozinha, formas de pagamento e aprovar restaurantes, só é acessível pelo perfil de administrador. Já os dados de um restaurante e o gerenciamento dos pedidos pendentes só são acessíveis pelo dono de cada restaurante.

Um usuário precisa identificar-se, ou seja, dizer quem está acessando a aplicação. Isso é o que chamamos de **Autenticação**.

Uma vez que o usuário está autenticado e sua identidade é conhecida, a aplicação pode reforçar as permissões de acesso.

Existem algumas maneiras mais comuns de um sistema confirmar a identidade de um usuário:

- algo que o usuário sabe (um segredo), como uma senha
- algo que o usuário tem, como um token físico ou por uma app mobile
- algo que o usuário é, biometria das digitais, íris ou reconhecimento facial

A maneira mais comum de autenticação é por senha

> Two-factor authentication (2FA), ou autenticação de dois fatores, é a associação de duas formas de autenticação para minimizar as chances de alguém mal intencionado identificar-se como outro usuário, no caso de apoderar-se de um dos fatores de autenticação.

## Sessões e escalabilidade

Uma aplicação Web tradicional armazena a identidade do usuário, depois da autenticação, em uma **sessão**, que comumente é armazenada em memória, mas pode ser armazenada em disco ou em um BD.

O cliente da aplicação, comumente um navegador, deve armazenar um id da sessão. Em toda requisição, o cliente passa esse id para identificar o usuário.

O que acontece quando há um aumento drástico no número de usuários em momento de pico de uso, como na Black Friday?

Uma aplicação que aguenta esse aumento na carga possui a característica arquitetural da **Escalabilidade**. Quando a escalabilidade é atingida aumentando o número de máquinas, dizemos que é a escalabilidade horizontal.

Onde fica armazenada a sessão se temos mais de uma máquina como servidor Web? Uma estratégia são as _sticky sessions_, em que cada usuário tem sua sessão em uma máquina específica.

Mas quando alguma máquina falhar, o usuário seria deslogado e não teria mais acesso às funcionalidades. Para que a experiência do usuário seja transparente, de maneira que ele não perceba a falha em uma máquina, há a técnica da **replicação de sessão**, em que cada servidor compartilha, pela rede, suas sessões com outros servidores. Isso traz uma sobrecarga de processamento, armazenamento e tráfego na rede.

## REST, stateless sessions e self-contained tokens

Em sua tese de doutorado _Architectural Styles and the Design of Network-based Software Architectures_, Roy Fielding descreve o estilo arquitetural da Web e o chama de **Representational State Transfer (REST)**. Uma das características do REST é que a comunicação deve ser **Stateless**: toda informação deve estar contida na requisição do cliente ao servidor, sem a necessidade de nenhum contexto armazenado no servidor.

Manter uma sessão no(s) servidor(es) é manter estado. Portanto, podemos dizem que utilizar sessões não é RESTful porque não segue a característica do REST de ser stateless.

Mas então como fazer um mecanismo de autenticação que seja stateless e, por consequência, mais próximo do REST?

Usando tokens! Há tokens opacos, que são apenas um texto randômico e que não carregam nenhuma informação. Porém, há os **self-contained tokens**, que contém informações sobre o usuário e/ou sobre o sistema cliente. Cada requisição teria um self-contained token em seu cabeçalho, com todas as informações necessárias para a aplicação. Assim, tiramos a necessidade de armazenamento da sessão no lado do servidor.

A grande questão é como ter um token que contém informações e, ao mesmo tempo, garantir sua integridade, confirmando que os dados do token não foram manipulados?

## JWT e JWS

JWT (JSON Web Token) é um formato de token compacto e self-contained que serve propagar informações de identidade, permissões de um usuário em uma aplicação de maneira segura. Foi definido na RFC 7519 da Internet Engineering Task Force (IETF), em Maio de 2015.

O Working Group da IETF chamado Javascript Object Signing and Encryption (JOSE), definiu duas outras RFCs relacionadas:

- JSON Web Signature (JWS), definido na RFC 7515, que representa em JSON conteúdo assinado digitalmente
- JSON Web Encryption (JWE), definido na RFC 7516, que representa em JSON conteúdo criptografado

Para garantir a integridade dos dados de um token, é suficiente usarmo o JWS.

Um JWS consiste de três partes, separadas por `.`:

```txt
BASE64URL(UTF8(JWS Protected Header)) || '.' ||
BASE64URL(JWS Payload) || '.' ||
BASE64URL(JWS Signature)
```

Todas as partes do JWS são codificadas em Base64 URL encoded. Base64 é uma representação em texto de dados binários. URL encoded significa que caracteres especiais são codificados com `%`, de maneira que possa ser passado como parâmetro em URLs.

Um exemplo de um JWS usado no Caelum Eats seria o seguinte:

```txt
eyJhbGciOiJIUzI1NiJ9.
eyJpc3MiOiJDYWVsdW0gRWF0cyIsInN1YiI6IjIiLCJyb2xlcyI6WyJQQVJDRUlSTyJdLCJ1c2VybmFtZSI6ImxvbmdmdSIsImlhdCI6MTU2NjQ5ODA5MSwiZXhwIjoxNTY3MTAyODkxfQ.
GOwiEeJMP9t0tV2lQpNiDU211WKL6h5Z6OkNcA-f4EY
```

Os trechos anteriores podem ser descodificados de base64 para texto normal usando um site como: http://www.base64url.com/

O primeiro trecho, `eyJhbGciOiJIUzI1NiJ9`, é o cabeçalho do JWS e quando desconvertido, é:

```json
{"alg":"HS256"}
```

O valor de `alg` indica que foi utilizado o algoritmo HMAC (hash-based message authentication code) com SHA-256 como função de hash. Nesse algoritmo, há uma chave secreta (um texto) simétrica, que deve ser conhecida tanto pela parte que cria o token como pela parte que o validará. Se essa chave secreta for descoberta por um agente mal intencionado, pode ser usada para gerar deliberadamente tokens válidos.

O segundo trecho, `eyJpc3MiOiJDYWVsdW0gRWF0cyIsInN1YiI6IjIiLCJyb2xlcyI6WyJQQVJDRUlSTyJdLCJ1c2VybmFtZSI6ImxvbmdmdSIsImlhdCI6MTU2NjQ5ODA5MSwiZXhwIjoxNTY3MTAyODkxfQ`, contém os dados (payload) do JWS:

```json
{
  "iss": "Caelum Eats",
  "sub": "2",
  "roles": [
    "PARCEIRO"
  ],
  "username": "longfu",
  "iat": 1566498091,
  "exp": 1567102891
}
```

O valor de `iss` é o issuer, definido pela aplicação que gerou o token.
O valor de `sub` é o subject, que contém informações do usuário.
Os valores de `iat` e `exp`, são as datas de geração e expiração do token, respectivamente.
Os demais valores são _claims_ customizadas, que declaram informações adicionais do usuário.

O terceiro trecho, `GOwiEeJMP9t0tV2lQpNiDU211WKL6h5Z6OkNcA-f4EY`, é a assinatura do JWS e não pode ser decodificada para um texto. O que importam são os bytes.

> No site https://jwt.io/ conseguimos obter o algoritmo utilizado, os dados do payload e até validar um JWT.

Se soubermos a chave secreta, podemos verificar se a assinatura bate com o payload do JWS. Se sim, o token é válido. Dessa maneira, conseguimos garantir que não houve manipulação dos dados e, portanto, sua integridade.

Um detalhe importante é que um JWS não garante a confidencialidade dos dados. Se houver algum software bisbilhotando os dados trafegados na rede, o payload do JWS pode ser lido, já que é apenas codificado em base64 URL encoded. A confidencialidade pode ser reforçada por meio de TLS no canal de comunicação ou por meio de JWE.

> Uma grande desvantagem de um JWT é que o token é irrevogável antes de sua expiração. Isso implica que, enquanto o token não estiver espirado será válido. Por isso, implementar um mecanismo de logout pelo usuário passa a ser complicado. Poderíamos trabalhar com intervalos pequenos de expiração, mas isso afetaria a experiência do usuário, já que frequentemente a expiração levaria o usuário a efetuar novo login. Uma maneira comum de implementar logout é ter um cache com JWT invalidados. Porém, isso nos leva novamente a uma solução _stateful_.

## Stateless Sessions no Caelum Eats

Até o momento, um login de um dono de restaurante ou do administrador do sistema dispara a execução do `AuthenticationController` do módulo `eats-seguranca` do monólito. No método `authenticate`, é gerado e retornado um token JWS.

O token JWS é armazenado em um `localStorage` no front-end. Há um _interceptor_ do Angular que, antes de cada requisição AJAX, adiciona o cabeçalho `Authorization: Bearer` com o valor do token armazenado.

No back-end, a classe `JwtAuthenticationFilter` é executada a cada requisição e o token JWS é extraído dos cabeçalhos HTTP e validado. Caso seja válido, é recuperado o `sub` (Subject) e obtido o usuário do BD com seus ROLEs (`ADMIN` ou `PARCEIRO`), setando um `Authentication` no contexto de segurança.

A geração, validação e recuperação dos dados do token é feita por meio da classe `JwtTokenManager`, que utiliza a biblioteca `jjwt`:

####### fj33-eats-monolito-modular/eats/eats-seguranca/src/main/java/br/com/caelum/eats/seguranca/JwtTokenManager.java

```java
@Component
class JwtTokenManager {

  private String secret;
  private long expirationInMillis;

  public JwtTokenManager(	@Value("${jwt.secret}") String secret, 
              @Value("${jwt.expiration}") long expirationInMillis) {
    this.secret = secret;
    this.expirationInMillis = expirationInMillis;
  }

  public String generateToken(User user) {
    final Date now = new Date();
    final Date expiration = new Date(now.getTime() + this.expirationInMillis);
    return Jwts.builder()
              .setIssuer("Caelum Eats")
              .setSubject(Long.toString(user.getId()))
              .claim("roles", user.getRoles())
              .claim("username", user.getUsername())
              .setIssuedAt(now)
              .setExpiration(expiration)
              .signWith(SignatureAlgorithm.HS256, this.secret)
              .compact();
  }

  public boolean isValid(String jwt) {
    try {
      Jwts
        .parser()
        .setSigningKey(this.secret)
        .parseClaimsJws(jwt);
      return true;
    } catch (JwtException | IllegalArgumentException e) {
      return false;
    }
  }

  public Long getUserIdFromToken(String jwt) {
    Claims claims = Jwts.parser().setSigningKey(this.secret).parseClaimsJws(jwt).getBody();
    return Long.parseLong(claims.getSubject());
  }

}
```

As configurações de autorização estão definidas na classe `SecurityConfig` do módulo `eats-seguranca` do monólito.

Para URLs que começam com `/admin`, o ROLE do usuário deve ser `ADMIN` e terá acesso a tudo relativo a administração da aplicação. Esse tipo de autorização, em que um determinado ROLE tem acesso a qualquer endpoint relacionado é o que chamamos de _role-based authorization_.

No caso da URL começar com `/parceiros/restaurantes/{restauranteId}`, é necessária uma autorização mais elaborada, que verifica se o usuário tem permissão ao restaurante por meio da classe `AuthorizationService`. Esse tipo de autorização, em que um usuário ter permissão em apenas alguns objetos de negócio é o que chamamos de _ACL-based authorization_. A sigla ACL significa _Access Control List_.

## Autenticação com Microservices e Single Sign On

Poderíamos implementar a autenticação  numa Arquitetura de Microservices de duas maneiras:

- o usuário precisa autenticar novamente ao acessar cada serviço
- a autenticação é feita apenas uma vez e as informações de identidade do usuário são repassadas para os serviços

Autenticar várias vezes, a cada serviço, é algo que deixaria a experiência do usuário terrível. Além disso, todos os serviços teriam que ser _edge services_, expostos à rede externa.

Autenticar apenas uma vez e repassar as dados do usuário autenticado permite que os serviços não fiquem expostos, diminuindo a superfície de ataque. Além disso, a experiência para o usuário é transparente, como se todas as funcionalidades fossem parte da mesma aplicação. É esse tipo de solução que chamamos de **Single Sign On** (SSO).

## Autenticação no API Gateway e Autorização nos Serviços

No livro Microservice Patterns, Chris Richardson descreve uma maneira comum de lidar com autenticação em uma arquitetura de Microservices: implementá-la API Gateway, o edge service único que fica exposto para o mundo externo. Dessa maneira, as chamadas a URLs protegidas já seriam barradas antes de passar para a rede interna, no caso do usuário não estar autenticado.

E a autorização? Poderíamos fazê-la também no API Gateway. É algo razoável para role-based authorization, em que é preciso saber apenas o ROLE do usuário. Porém, implementar ACL-based authorization no API Gateway levaria a um alto acoplamento com os serviços, já que precisamos saber se um dado usuário tem permissão para um objeto de negócio específico. Então, provavelmente uma atualização em um serviço iria querer uma atualização sincronizada no API Gateway, diminuindo a independência de cada serviço. Portanto, uma ideia melhor é fazer a autorização, role-based ou ACL-based, em cada serviço.

## Access Token e JWT

Com a autenticação sendo feito no API Gateway e a autorização em cada _downstream service_, surge um problema: como passar a identidade de um usuário do API Gateway para cada serviço?

Há duas alternativas:

- um token opaco: simplesmente uma string ou UUID que precisaria ser validada por cada serviço no emissor do token através de uma chamada remota.
- um self-contained token: um token que contém as informações do usuário e que tem sua integridade protegida através de uma assinatura. Assim, o próprio recipiente do token pode validar as informações checando a assinatura. Tanto o emissor como o recipiente devem compartilhar chaves para que a emissão e a checagem do token possam ser realizadas.

> **Pattern: Acess Token**
> 
> O API Gateway passa um token contendo informações sobre o usuário, como sua identidade e seus roles, para os demais serviços.

Implementamos stateless sessions no monólito com um JWS, um tipo de JWT que é um token self-contained e assinado. Podemos usar o mesmo mecanismo, fazendo com que o API Gateway repasse o JWT para cada serviço. Cada serviço checaria a assinatura e extrairia, do payload do JWT, o subject, que contém o id do usuário, e os respectivos roles, usando essas informações para checar a permissão do usuário ao recurso solicitado.

## Segurança no Caelum Eats

A solução de stateless sessions com JWT do Caelum Eats, foi pensada e implementada visando uma aplicação monolítica. E o resto dos serviços?

Temos serviços de infra-estrutura como:

- API Gateway
- Service Registry
- Config Server
- Hystrix Dashboard
- Turbine
- Admin Server

Temos serviços alinhado a contextos delimitados da Caelum Eats, como:

- Distância
- Pagamento
- Nota Fiscal

Há ainda módulos do monólito relacionados a contextos delimitados:

- Admin
- Pedido
- Restaurante

Os únicos módulos cujos endpoints tem seu acesso protegido são os módulos de Admin e Restaurante do monólito.

O monólito possui também um módulo de Segurança, que trata desse requisito transversal e contém código de configuração do Spring Security.

O módulo Admin por meio de role-based authorization, bastando o usuário estar no role ADMIN para acessar os endpoints de administração de tipos de cozinha e formas de pagamento.

Já o módulo de Restaurante efetua ACL-based authorization, limitando o acesso do usuário com role PARCEIRO a um restaurante específico.

![Geração e validação de tokens no módulo de segurança do monólito {w=39}](imagens/15-seguranca/geracao-e-validacao-de-tokens-no-modulo-de-seguranca-do-monolito.png)

Vamos modificar esse cenário, passando a responsabilidade de geração de tokens JWS para o API Gateway, além do cadastro de novos usuários donos de restaurantes. A validação do token e autorização ainda ficará a cargo dos módulos Admin e Restaurante do monólito.

## Exercício: Autenticação no API Gateway

1. Poderíamos ter um BD específico para conter dados de usuários com as tabelas `user`, `role` e `user_authorities`. Porém, para simplificar, vamos manter o BD do API Gateway no próprio BD do monólito.

  Adicione, ao API Gateway, dependências ao JJWT, biblioteca que gera e valida tokens JWT, aos _starters_ do Spring Security e do Spring Data JPA e o driver do MySQL:

  ####### api-gateway/pom.xml

  ```xml
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt</artifactId>
    <version>0.9.1</version>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
  </dependency>

  <dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <scope>runtime</scope>
  </dependency>

  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  ```

2. No `config-repo`, adicione um arquivo `apigateway.properties` com o dados de conexão do BD do monólito, além das configurações de segredo e expiração do JWT, que são usadas na geração do token:

  ####### config-repo/apigateway.properties

  ```properties
  #DATASOURCE CONFIGS
  spring.datasource.url=jdbc:mysql://localhost/eats?createDatabaseIfNotExist=true
  spring.datasource.username=root
  spring.datasource.password=

  #JWT CONFIGS
  jwt.secret = rm'!@N=Ke!~p8VTA2ZRK~nMDQX5Uvm!m'D&]{@Vr?G;2?XhbC:Qa#9#eMLN\}x3?JR3.2zr~v)gYF^8\:8>:XfB:Ww75N/emt9Yj[bQMNCWwW\J?N,nvH.<2\.r~w]*e~vgak)X"v8H`MH/7"2E`,^k@n<vE-wD3g9JWPy;CrY*.Kd2_D])=><D?YhBaSua5hW%{2]_FVXzb9`8FH^b[X3jzVER&:jw2<=c38=>L/zBq`}C6tT*cCSVC^c]-L}&/
  jwt.expiration = 604800000
  ```

3. Mova as classes a seguir do módulo de segurança do monólito para o API Gateway, no pacote `br.com.caelum.apigateway`:

  - AuthenticationController.java
  - AuthenticationDto.java
  - UserInfoDto.java
  - UserRepository.java
  - UserService.java
  - PasswordEncoderConfig.java

  Além dessas, copie as seguintes classes, mantendo-as no módulo de segurança do monólito:

  - SecurityConfig.java
  - Role.java
  - User.java
  - JwtTokenManager.java

  Copie também a seguinte classe do módulo `eats-common` do monólito:

  - CorsConfig.java

  Não esqueça de ajustar o pacote das classes copiadas.

  Essas classes fazem geração e validação de tokens JWT, assim como o cadastro de novos donos de restaurante.

4. Altere a classe `SecurityConfig` para que permita toda e qualquer requisição, desabilitando a autorização, que será feita pelos serviços. A autenticação será _stateless_.

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/SecurityConfig.java

  ```java
  @Configuration
  @EnableWebSecurity
  @AllArgsConstructor
  class SecurityConfig extends WebSecurityConfigurerAdapter {

    private UserService userService;
    p̶r̶i̶v̶a̶t̶e̶ ̶J̶w̶t̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶F̶i̶l̶t̶e̶r̶ ̶j̶w̶t̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶F̶i̶l̶t̶e̶r̶;̶
    p̶r̶i̶v̶a̶t̶e̶ ̶J̶w̶t̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶E̶n̶t̶r̶y̶P̶o̶i̶n̶t̶ ̶j̶w̶t̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶E̶n̶t̶r̶y̶P̶o̶i̶n̶t̶;̶
    private BCryptPasswordEncoder bCryptPasswordEncoder;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
      http.authorizeRequests()
          .̶a̶n̶t̶M̶a̶t̶c̶h̶e̶r̶s̶(̶"̶/̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶/̶*̶*̶"̶,̶ ̶"̶/̶p̶e̶d̶i̶d̶o̶s̶/̶*̶*̶"̶,̶ ̶"̶/̶p̶a̶g̶a̶m̶e̶n̶t̶o̶s̶/̶*̶*̶"̶,̶ ̶"̶/̶t̶i̶p̶o̶s̶-̶d̶e̶-̶c̶o̶z̶i̶n̶h̶a̶/̶*̶*̶"̶,̶ ̶"̶/̶f̶o̶r̶m̶a̶s̶-̶d̶e̶-̶p̶a̶g̶a̶m̶e̶n̶t̶o̶/̶*̶*̶"̶)̶.̶p̶e̶r̶m̶i̶t̶A̶l̶l̶(̶)̶
          .̶a̶n̶t̶M̶a̶t̶c̶h̶e̶r̶s̶(̶"̶/̶s̶o̶c̶k̶e̶t̶/̶*̶*̶"̶)̶.̶p̶e̶r̶m̶i̶t̶A̶l̶l̶(̶)̶
          .̶a̶n̶t̶M̶a̶t̶c̶h̶e̶r̶s̶(̶"̶/̶a̶u̶t̶h̶/̶*̶*̶"̶)̶.̶p̶e̶r̶m̶i̶t̶A̶l̶l̶(̶)̶
          .̶a̶n̶t̶M̶a̶t̶c̶h̶e̶r̶s̶(̶"̶/̶a̶c̶t̶u̶a̶t̶o̶r̶/̶*̶*̶"̶)̶.̶p̶e̶r̶m̶i̶t̶A̶l̶l̶(̶)̶
          .̶a̶n̶t̶M̶a̶t̶c̶h̶e̶r̶s̶(̶"̶/̶a̶d̶m̶i̶n̶/̶*̶*̶"̶)̶.̶h̶a̶s̶R̶o̶l̶e̶(̶R̶o̶l̶e̶.̶R̶O̶L̶E̶S̶.̶A̶D̶M̶I̶N̶.̶n̶a̶m̶e̶(̶)̶)̶
          .̶a̶n̶t̶M̶a̶t̶c̶h̶e̶r̶s̶(̶H̶t̶t̶p̶M̶e̶t̶h̶o̶d̶.̶P̶O̶S̶T̶,̶ ̶"̶/̶p̶a̶r̶c̶e̶i̶r̶o̶s̶/̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶"̶)̶.̶p̶e̶r̶m̶i̶t̶A̶l̶l̶(̶)̶
          .̶a̶n̶t̶M̶a̶t̶c̶h̶e̶r̶s̶(̶"̶/̶p̶a̶r̶c̶e̶i̶r̶o̶s̶/̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶s̶/̶{̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶I̶d̶}̶/̶*̶*̶"̶)̶
              .̶a̶c̶c̶e̶s̶s̶(̶"̶@̶a̶u̶t̶h̶o̶r̶i̶z̶a̶t̶i̶o̶n̶S̶e̶r̶v̶i̶c̶e̶.̶c̶h̶e̶c̶a̶T̶a̶r̶g̶e̶t̶I̶d̶(̶a̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶,̶#̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶I̶d̶)̶"̶)̶
          .̶a̶n̶t̶M̶a̶t̶c̶h̶e̶r̶s̶(̶"̶/̶p̶a̶r̶c̶e̶i̶r̶o̶s̶/̶*̶*̶"̶)̶.̶h̶a̶s̶R̶o̶l̶e̶(̶R̶o̶l̶e̶.̶R̶O̶L̶E̶S̶.̶P̶A̶R̶C̶E̶I̶R̶O̶.̶n̶a̶m̶e̶(̶)̶)̶
          .̶a̶n̶y̶R̶e̶q̶u̶e̶s̶t̶(̶)̶.̶a̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶e̶d̶(̶)̶
          .anyRequest().permitAll() // modificado
          .and().cors()
          .and().csrf().disable()
          .formLogin().disable()
          .httpBasic().disable()
          .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
          .̶a̶n̶d̶(̶)̶
          .̶a̶d̶d̶F̶i̶l̶t̶e̶r̶B̶e̶f̶o̶r̶e̶(̶j̶w̶t̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶F̶i̶l̶t̶e̶r̶,̶ ̶U̶s̶e̶r̶n̶a̶m̶e̶P̶a̶s̶s̶w̶o̶r̶d̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶F̶i̶l̶t̶e̶r̶.̶c̶l̶a̶s̶s̶)̶
          .̶e̶x̶c̶e̶p̶t̶i̶o̶n̶H̶a̶n̶d̶l̶i̶n̶g̶(̶)̶.̶a̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶E̶n̶t̶r̶y̶P̶o̶i̶n̶t̶(̶j̶w̶t̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶E̶n̶t̶r̶y̶P̶o̶i̶n̶t̶)̶;
    }

    @Override
    protected void configure(final AuthenticationManagerBuilder auth) throws Exception {
      auth.userDetailsService(userService).passwordEncoder(bCryptPasswordEncoder);
    }

    @Override
    @Bean(BeanIds.AUTHENTICATION_MANAGER)
    public AuthenticationManager authenticationManagerBean() throws Exception {
      return super.authenticationManagerBean();
    }

  }
  ```

5. Na classe `JwtTokenManager` do API Gateway, podemos remover os métodos `isValid` e `getUserIdFromToken`. Ambos serão responsabilidade dos serviços, que farão a autorização.

  É importante adicionar aos _claims_ do JWT gerado, o username e os roles do usuário.

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/JwtTokenManager.java

  ```java
  @Component
  class JwtTokenManager {

    private String secret;
    private long expirationInMillis;

    public JwtTokenManager(@Value("${jwt.secret}") String secret,
                @Value("${jwt.expiration}") long expirationInMillis) {
      this.secret = secret;
      this.expirationInMillis = expirationInMillis;
    }

    public String generateToken(User user) {
      final Date now = new Date();
      final Date expiration = new Date(now.getTime() + this.expirationInMillis);
      return Jwts.builder()
          .setIssuer("Caelum Eats")
          .setSubject(Long.toString(user.getId()))
          .claim("roles", user.getRoles()) // adicionado
          .claim("username", user.getUsername()) // adicionado
          .setIssuedAt(now)
          .setExpiration(expiration)
          .signWith(SignatureAlgorithm.HS256, this.secret)
          .compact();
    }

    ̶p̶u̶b̶l̶i̶c̶ ̶b̶o̶o̶l̶e̶a̶n̶ ̶i̶s̶V̶a̶l̶i̶d̶(̶S̶t̶r̶i̶n̶g̶ ̶j̶w̶t̶)̶ ̶{̶
    }̶

    p̶u̶b̶l̶i̶c̶ ̶L̶o̶n̶g̶ ̶g̶e̶t̶U̶s̶e̶r̶I̶d̶F̶r̶o̶m̶T̶o̶k̶e̶n̶(̶S̶t̶r̶i̶n̶g̶ ̶j̶w̶t̶)̶ ̶{̶
    }̶

  }
  ```

6. Nas classes `AuthenticationDto` e `AuthenticationController`, vamos remover o `targetId`, que é usada para encontrar o restaurante de um usuário específico. Isso passará a ser responsabilidade de cada serviço:

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/AuthenticationDto.java

  ```java
  @Data
  @AllArgsConstructor
  class AuthenticationDto {
    private String username;
    private List<String> roles;
    private String token;
    private Long targetId;

    p̶u̶b̶l̶i̶c̶ ̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶D̶t̶o̶(̶U̶s̶e̶r̶ ̶u̶s̶e̶r̶,̶ ̶S̶t̶r̶i̶n̶g̶ ̶j̶w̶t̶T̶o̶k̶e̶n̶,̶ ̶L̶o̶n̶g̶ ̶t̶a̶r̶g̶e̶t̶I̶d̶)̶ ̶{̶
    public AuthenticationDto(User user, String jwtToken) {
      t̶h̶i̶s̶(̶u̶s̶e̶r̶.̶g̶e̶t̶N̶a̶m̶e̶(̶)̶,̶ ̶u̶s̶e̶r̶.̶g̶e̶t̶R̶o̶l̶e̶s̶(̶)̶,̶ ̶j̶w̶t̶T̶o̶k̶e̶n̶,̶ ̶t̶a̶r̶g̶e̶t̶I̶d̶)̶;̶
      this(user.getName(), user.getRoles(), jwtToken);
    }

  }
  ```

  ####### api-gateway/src/main/java/br/com/caelum/apigateway/AuthenticationController.java

  ```java
  @RestController
  @RequestMapping("/auth")
  @AllArgsConstructor
  class AuthenticationController {
    private AuthenticationManager authManager;
    private JwtTokenManager jwtTokenManager;
    private UserService userService;
    p̶r̶i̶v̶a̶t̶e̶ ̶L̶i̶s̶t̶<̶A̶u̶t̶h̶o̶r̶i̶z̶a̶t̶i̶o̶n̶T̶a̶r̶g̶e̶t̶S̶e̶r̶v̶i̶c̶e̶>̶ ̶a̶u̶t̶h̶o̶r̶i̶z̶a̶t̶i̶o̶n̶S̶e̶r̶v̶i̶c̶e̶s̶;̶

    @PostMapping
    public ResponseEntity<AuthenticationDto> authenticate(@RequestBody UserInfoDto login) {
      UsernamePasswordAuthenticationToken authenticationToken = new UsernamePasswordAuthenticationToken(
          login.getUsername(), login.getPassword());
      try {
        Authentication authentication = authManager.authenticate(authenticationToken);
        User user = (User) authentication.getPrincipal();
        String jwt = jwtTokenManager.generateToken(user);
        L̶o̶n̶g̶ ̶t̶a̶r̶g̶e̶t̶I̶d̶ ̶=̶ ̶g̶e̶t̶T̶a̶r̶g̶e̶t̶I̶d̶F̶o̶r̶(̶u̶s̶e̶r̶)̶;̶
        A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶D̶t̶o̶ ̶t̶o̶k̶e̶n̶R̶e̶s̶p̶o̶n̶s̶e̶ ̶=̶ ̶n̶e̶w̶ ̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶D̶t̶o̶(̶u̶s̶e̶r̶,̶ ̶j̶w̶t̶,̶ ̶t̶a̶r̶g̶e̶t̶I̶d̶)̶;̶
        AuthenticationDto tokenResponse = new AuthenticationDto(user, jwt); // modificado
        return ResponseEntity.ok(tokenResponse);
      } catch (AuthenticationException e) {
        return ResponseEntity.badRequest().build();
      }

    }

    @PostMapping("/parceiro")
    public Long register(@RequestBody UserInfoDto userInfo) {
      User user = userInfo.toUser();
      user.addRole(Role.ROLES.PARCEIRO);
      User salvo = userService.save(user);
      return salvo.getId();
    }

    p̶r̶i̶v̶a̶t̶e̶ ̶L̶o̶n̶g̶ ̶g̶e̶t̶T̶a̶r̶g̶e̶t̶I̶d̶F̶o̶r̶(̶U̶s̶e̶r̶ ̶u̶s̶e̶r̶)̶ ̶{̶
  ̶ ̶ ̶ ̶ ̶f̶o̶r̶ ̶(̶A̶u̶t̶h̶o̶r̶i̶z̶a̶t̶i̶o̶n̶T̶a̶r̶g̶e̶t̶S̶e̶r̶v̶i̶c̶e̶ ̶a̶u̶t̶h̶o̶r̶i̶z̶a̶t̶i̶o̶n̶T̶a̶r̶g̶e̶t̶S̶e̶r̶v̶i̶c̶e̶ ̶:̶ ̶a̶u̶t̶h̶o̶r̶i̶z̶a̶t̶i̶o̶n̶S̶e̶r̶v̶i̶c̶e̶s̶)̶ ̶{̶
  ̶ ̶ ̶ ̶ ̶ ̶ ̶L̶o̶n̶g̶ ̶t̶a̶r̶g̶e̶t̶I̶d̶ ̶=̶ ̶a̶u̶t̶h̶o̶r̶i̶z̶a̶t̶i̶o̶n̶T̶a̶r̶g̶e̶t̶S̶e̶r̶v̶i̶c̶e̶.̶g̶e̶t̶T̶a̶r̶g̶e̶t̶I̶d̶B̶y̶U̶s̶e̶r̶(̶u̶s̶e̶r̶)̶;̶
  ̶ ̶ ̶ ̶ ̶ ̶ ̶i̶f̶ ̶(̶t̶a̶r̶g̶e̶t̶I̶d̶ ̶!̶=̶ ̶n̶u̶l̶l̶)̶ ̶{̶
  ̶ ̶ ̶ ̶ ̶ ̶ ̶ ̶ ̶r̶e̶t̶u̶r̶n̶ ̶t̶a̶r̶g̶e̶t̶I̶d̶;̶
  ̶ ̶ ̶ ̶ ̶ ̶ ̶}̶
  ̶ ̶ ̶ ̶ ̶}̶
  ̶ ̶ ̶ ̶ ̶r̶e̶t̶u̶r̶n̶ ̶n̶u̶l̶l̶;̶
  ̶ ̶ ̶}̶

  }
  ```

7. Ainda no API Gateway, adicione um _forward_ para a URL do `AuthenticationController`, de maneira que o Zuul não tente fazer o proxy dessa chamada:

  ####### api-gateway/src/main/resources/application.properties

  ```properties
  zuul.routes.auth.path=/auth/**
  zuul.routes.auth.url=forward:/auth
  ```

  _Observação: essa configuração deve ficar antes da configuração da rota fallback que direciona as requisições para o monólito._

8. Execute `ApiGatewayApplication`, certificando-se que o Service Registry e o Config Server estão no ar.

  Então, abra o terminal e simule a autenticação do administrador:

  ```sh
  curl -i -X POST -H 'Content-type: application/json' -d '{"username":"admin", "password":"123456"}' http://localhost:9999/auth
  ```

  Use o seguinte snippet, para evitar digitação: https://gitlab.com/snippets/1888245

  Você deve obter um retorno parecido com:

  ```txt
  HTTP/1.1 200 
  X-Content-Type-Options: nosniff
  X-XSS-Protection: 1; mode=block
  Cache-Control: no-cache, no-store, max-age=0, must-revalidate
  Pragma: no-cache
  Expires: 0
  X-Frame-Options: DENY
  Content-Type: application/json;charset=UTF-8
  Transfer-Encoding: chunked
  Date: Fri, 23 Aug 2019 00:05:22 GMT

  {"username":"admin","roles":["ADMIN"],"token":"eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJDYWVsdW0gRWF0cyIsInN1YiI6IjEiLCJyb2xlcyI6WyJBRE1JTiJdLCJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTY2NTE4NzIyLCJleHAiOjE1NjcxMjM1MjJ9.FmH2QkryLBxWZjt2DMKHsCmjQNCmk3hrRAC0keam5_w"}
  ```

  São retornados, no corpo da resposta, informações sobre o usuário, seus roles e um token. Guarde esse token que o usaremos em breve!

## Exercício: Validando o token JWT e implementando autorização no Monólito

1. Modifique o `JwtTokenManager` do módulo de segurança do monólito, removendo o código de geração de token (e o atributo `expirationInMillis`) e deixando apenas a validação e extração de dados. Modifique o método `getUserIdFromToken` para que obtenha os dados do usuário somente a partir dos claims do token JWT.

  ####### fj33-eats-monolito-modular/eats/eats-seguranca/src/main/java/br/com/caelum/eats/seguranca/JwtTokenManager.java

  ```java
  @Component
  class JwtTokenManager {

    private String secret;
    p̶r̶i̶v̶a̶t̶e̶ ̶l̶o̶n̶g̶ ̶e̶x̶p̶i̶r̶a̶t̶i̶o̶n̶I̶n̶M̶i̶l̶l̶i̶s̶;̶

    public JwtTokenManager(@Value("${jwt.secret}") String secret,̶
                @̶V̶a̶l̶u̶e̶(̶"̶$̶{̶j̶w̶t̶.̶e̶x̶p̶i̶r̶a̶t̶i̶o̶n̶}̶"̶)̶ ̶l̶o̶n̶g̶ ̶e̶x̶p̶i̶r̶a̶t̶i̶o̶n̶I̶n̶M̶i̶l̶l̶i̶s̶) {
      this.secret = secret;
      t̶h̶i̶s̶.̶e̶x̶p̶i̶r̶a̶t̶i̶o̶n̶I̶n̶M̶i̶l̶l̶i̶s̶ ̶=̶ ̶e̶x̶p̶i̶r̶a̶t̶i̶o̶n̶I̶n̶M̶i̶l̶l̶i̶s̶;̶
    }

    p̶u̶b̶l̶i̶c̶ ̶S̶t̶r̶i̶n̶g̶ ̶g̶e̶n̶e̶r̶a̶t̶e̶T̶o̶k̶e̶n̶(̶U̶s̶e̶r̶ ̶u̶s̶e̶r̶)̶ ̶{̶
  ̶ ̶ ̶ ̶ ̶f̶i̶n̶a̶l̶ ̶D̶a̶t̶e̶ ̶n̶o̶w̶ ̶=̶ ̶n̶e̶w̶ ̶D̶a̶t̶e̶(̶)̶;̶
  ̶ ̶ ̶ ̶ ̶f̶i̶n̶a̶l̶ ̶D̶a̶t̶e̶ ̶e̶x̶p̶i̶r̶a̶t̶i̶o̶n̶ ̶=̶ ̶n̶e̶w̶ ̶D̶a̶t̶e̶(̶n̶o̶w̶.̶g̶e̶t̶T̶i̶m̶e̶(̶)̶ ̶+̶ ̶t̶h̶i̶s̶.̶e̶x̶p̶i̶r̶a̶t̶i̶o̶n̶I̶n̶M̶i̶l̶l̶i̶s̶)̶;̶
  ̶ ̶ ̶ ̶ ̶r̶e̶t̶u̶r̶n̶ ̶J̶w̶t̶s̶.̶b̶u̶i̶l̶d̶e̶r̶(̶)̶.̶s̶e̶t̶I̶s̶s̶u̶e̶r̶(̶"̶C̶a̶e̶l̶u̶m̶ ̶E̶a̶t̶s̶"̶)̶.̶s̶e̶t̶S̶u̶b̶j̶e̶c̶t̶(̶L̶o̶n̶g̶.̶t̶o̶S̶t̶r̶i̶n̶g̶(̶u̶s̶e̶r̶.̶g̶e̶t̶I̶d̶(̶)̶)̶)̶.̶s̶e̶t̶I̶s̶s̶u̶e̶d̶A̶t̶(̶n̶o̶w̶)̶
  ̶ ̶ ̶ ̶ ̶ ̶ ̶ ̶ ̶.̶s̶e̶t̶E̶x̶p̶i̶r̶a̶t̶i̶o̶n̶(̶e̶x̶p̶i̶r̶a̶t̶i̶o̶n̶)̶.̶s̶i̶g̶n̶W̶i̶t̶h̶(̶S̶i̶g̶n̶a̶t̶u̶r̶e̶A̶l̶g̶o̶r̶i̶t̶h̶m̶.̶H̶S̶2̶5̶6̶,̶ ̶t̶h̶i̶s̶.̶s̶e̶c̶r̶e̶t̶)̶.̶c̶o̶m̶p̶a̶c̶t̶(̶)̶;̶
  ̶ ̶ ̶}̶

    public boolean isValid(String jwt) {
      // não modificado
    }

    p̶u̶b̶l̶i̶c̶ ̶L̶o̶n̶g̶ ̶g̶e̶t̶U̶s̶e̶r̶I̶d̶F̶r̶o̶m̶T̶o̶k̶e̶n̶(̶S̶t̶r̶i̶n̶g̶ ̶j̶w̶t̶)̶ ̶{̶
    public User getUserFromToken(String jwt) {
      Claims claims = Jwts.parser().setSigningKey(this.secret).parseClaimsJws(jwt).getBody();
      r̶e̶t̶u̶r̶n̶ ̶L̶o̶n̶g̶.̶p̶a̶r̶s̶e̶L̶o̶n̶g̶(̶c̶l̶a̶i̶m̶s̶.̶g̶e̶t̶S̶u̶b̶j̶e̶c̶t̶(̶)̶)̶;̶
      User user = new User();
      user.setName(claims.get("username", String.class));
      user.setId(Long.parseLong(claims.getSubject()));
      List<String> roles = claims.get("roles", List.class);
      roles.stream().forEach(role -> user.addRole(ROLES.valueOf(role)));
      return user;
    }

  }
  ```

  Não deixe de importar:

  ```java
  import java.util.List;
  ```

2. Remova as anotações do JPA e Beans Validator das classes `User` e `Role` do módulo de segurança do monólito. O cadastro de usuários será feito pelo API Gateway.

  ####### fj33-eats-monolito-modular/eats/eats-seguranca/src/main/java/br/com/caelum/eats/seguranca/User.java

  ```java
  @̶E̶n̶t̶i̶t̶y̶
  @NoArgsConstructor
  @AllArgsConstructor
  @Data
  public class User implements UserDetails {

    private static final long serialVersionUID = 1L;

    @̶I̶d̶
    @̶G̶e̶n̶e̶r̶a̶t̶e̶d̶V̶a̶l̶u̶e̶(̶s̶t̶r̶a̶t̶e̶g̶y̶ ̶=̶ ̶G̶e̶n̶e̶r̶a̶t̶i̶o̶n̶T̶y̶p̶e̶.̶I̶D̶E̶N̶T̶I̶T̶Y̶)̶
    private Long id;

    @̶N̶o̶t̶B̶l̶a̶n̶k̶ ̶@̶J̶s̶o̶n̶I̶g̶n̶o̶r̶e̶
    private String name;

    @̶N̶o̶t̶B̶l̶a̶n̶k̶ ̶@̶J̶s̶o̶n̶I̶g̶n̶o̶r̶e̶
    private String password;

    @̶M̶a̶n̶y̶T̶o̶M̶a̶n̶y̶(̶f̶e̶t̶c̶h̶ ̶=̶ ̶F̶e̶t̶c̶h̶T̶y̶p̶e̶.̶E̶A̶G̶E̶R̶)̶ ̶@̶J̶s̶o̶n̶I̶g̶n̶o̶r̶e̶
    private List<Role> authorities = new ArrayList<>();

    // restante do código ...
  ```

  ####### fj33-eats-monolito-modular/eats/eats-seguranca/src/main/java/br/com/caelum/eats/seguranca/Role.java

  ```java
  @̶E̶n̶t̶i̶t̶y̶
  @NoArgsConstructor
  @AllArgsConstructor
  @Data
  public class Role implements GrantedAuthority {

    // código omitido...

    @̶I̶d̶
    private String authority;

    // restante do código...
  ```

  3. Altere a classe `SecurityConfig` do módulo de segurança do monólito, removendo código associado a autenticação e cadastro de novos usuários:

  ####### fj33-eats-monolito-modular/eats/eats-seguranca/src/main/java/br/com/caelum/eats/SecurityConfig.java

  ```java
  @Configuration
  @EnableWebSecurity
  @AllArgsConstructor
  class SecurityConfig extends WebSecurityConfigurerAdapter {

    p̶r̶i̶v̶a̶t̶e̶ ̶U̶s̶e̶r̶S̶e̶r̶v̶i̶c̶e̶ ̶u̶s̶e̶r̶S̶e̶r̶v̶i̶c̶e̶;̶
    private JwtAuthenticationFilter jwtAuthenticationFilter;
    private JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
    p̶r̶i̶v̶a̶t̶e̶ ̶B̶C̶r̶y̶p̶t̶P̶a̶s̶s̶w̶o̶r̶d̶E̶n̶c̶o̶d̶e̶r̶ ̶b̶C̶r̶y̶p̶t̶P̶a̶s̶s̶w̶o̶r̶d̶E̶n̶c̶o̶d̶e̶r̶;̶

    // código omitido...

    @̶O̶v̶e̶r̶r̶i̶d̶e̶
  ̶ ̶ ̶p̶r̶o̶t̶e̶c̶t̶e̶d̶ ̶v̶o̶i̶d̶ ̶c̶o̶n̶f̶i̶g̶u̶r̶e̶(̶f̶i̶n̶a̶l̶ ̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶M̶a̶n̶a̶g̶e̶r̶B̶u̶i̶l̶d̶e̶r̶ ̶a̶u̶t̶h̶)̶ ̶t̶h̶r̶o̶w̶s̶ ̶E̶x̶c̶e̶p̶t̶i̶o̶n̶ ̶{̶
  ̶ ̶ ̶ ̶ ̶a̶u̶t̶h̶.̶u̶s̶e̶r̶D̶e̶t̶a̶i̶l̶s̶S̶e̶r̶v̶i̶c̶e̶(̶u̶s̶e̶r̶S̶e̶r̶v̶i̶c̶e̶)̶.̶p̶a̶s̶s̶w̶o̶r̶d̶E̶n̶c̶o̶d̶e̶r̶(̶b̶C̶r̶y̶p̶t̶P̶a̶s̶s̶w̶o̶r̶d̶E̶n̶c̶o̶d̶e̶r̶)̶;̶
  ̶ ̶ ̶}̶

    @̶O̶v̶e̶r̶r̶i̶d̶e̶
  ̶ ̶ ̶@̶B̶e̶a̶n̶(̶B̶e̶a̶n̶I̶d̶s̶.̶A̶U̶T̶H̶E̶N̶T̶I̶C̶A̶T̶I̶O̶N̶_̶M̶A̶N̶A̶G̶E̶R̶)̶
  ̶ ̶ ̶p̶u̶b̶l̶i̶c̶ ̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶M̶a̶n̶a̶g̶e̶r̶ ̶a̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶M̶a̶n̶a̶g̶e̶r̶B̶e̶a̶n̶(̶)̶ ̶t̶h̶r̶o̶w̶s̶ ̶E̶x̶c̶e̶p̶t̶i̶o̶n̶ ̶{̶
  ̶ ̶ ̶ ̶ ̶r̶e̶t̶u̶r̶n̶ ̶s̶u̶p̶e̶r̶.̶a̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶M̶a̶n̶a̶g̶e̶r̶B̶e̶a̶n̶(̶)̶;̶
  ̶ ̶ ̶}̶

  }
  ```

4. Modifique a classe `JwtAuthenticationFilter` do módulo de segurança do monólito, para que os dados do usuário não sejam mais obtidos do BD mas do próprio token JWT:

  ####### fj33-eats-monolito-modular/eats/eats-seguranca/src/main/java/br/com/caelum/eats/seguranca/JwtAuthenticationFilter.java

  ```java
  public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private JwtTokenManager tokenManager;
    p̶r̶i̶v̶a̶t̶e̶ ̶U̶s̶e̶r̶S̶e̶r̶v̶i̶c̶e̶ ̶u̶s̶e̶r̶s̶S̶e̶r̶v̶i̶c̶e̶;̶

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
        throws ServletException, IOException {
      String jwt = getTokenFromRequest(request);
      if (tokenManager.isValid(jwt)) {
        L̶o̶n̶g̶ ̶u̶s̶e̶r̶I̶d̶ ̶=̶ ̶t̶o̶k̶e̶n̶M̶a̶n̶a̶g̶e̶r̶.̶g̶e̶t̶U̶s̶e̶r̶I̶d̶F̶r̶o̶m̶T̶o̶k̶e̶n̶(̶j̶w̶t̶)̶;̶
        User user = tokenManager.getUserFromToken(jwt);
        U̶s̶e̶r̶D̶e̶t̶a̶i̶l̶s̶ ̶u̶s̶e̶r̶D̶e̶t̶a̶i̶l̶s̶ ̶=̶ ̶u̶s̶e̶r̶s̶S̶e̶r̶v̶i̶c̶e̶.̶l̶o̶a̶d̶U̶s̶e̶r̶B̶y̶I̶d̶(̶u̶s̶e̶r̶I̶d̶)̶;̶
        UserDetails userDetails = user;
        UsernamePasswordAuthenticationToken authentication = 
            new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
        SecurityContextHolder.getContext().setAuthentication(authentication);
    // restante do código ...
  ```

5. Como a classe `User` não é mais uma entidade, devemos modificar seu relacionamento na classe `Restaurante` do módulo `eats-restaurante` do monólito:

  ####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/Restaurante.java

  ```java
  @Entity
  @NoArgsConstructor
  @AllArgsConstructor
  @Data
  public class Restaurante {

    // código omitido...

    @̶O̶n̶e̶T̶o̶O̶n̶e̶
    p̶r̶i̶v̶a̶t̶e̶ ̶U̶s̶e̶r̶ ̶u̶s̶e̶r̶;̶
    private Long userId; // modificado

  }
  ```

  Modifique também o uso do atributo `user` do `Restaurante` na classe `RestauranteController`:

  ####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/RestauranteController.java

  ```java
  @RestController
  @AllArgsConstructor
  class RestauranteController {

    // código omitido...

    @PutMapping("/parceiros/restaurantes/{id}")
    public Restaurante atualiza(@RequestBody Restaurante restaurante) {
      Restaurante doBD = restauranteRepo.getOne(restaurante.getId());

      r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶.̶s̶e̶t̶U̶s̶e̶r̶(̶d̶o̶B̶D̶.̶g̶e̶t̶U̶s̶e̶r̶(̶)̶)̶;̶
      restaurante.setUserId(doBD.getUserId()); // modificado

      restaurante.setAprovado(doBD.getAprovado());

      // código omitido...

      return restauranteRepo.save(restaurante);
    }
  
    // código omitido...

  }
  ```

  Ajuste a interface `RestauranteRepository`:

  ####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/RestauranteRepository.java

  ```java
  interface RestauranteRepository extends JpaRepository<Restaurante, Long> {

    // código omitido...

    R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶ ̶f̶i̶n̶d̶B̶y̶U̶s̶e̶r̶(̶U̶s̶e̶r̶ ̶u̶s̶e̶r̶)̶;̶
    Restaurante findByUserId(Long userId); // modificado

    // código omitido...

  }
  ```

  Faça com que a classe `RestauranteAuthorizationTargetService` use o novo método do repository:

  ####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/RestauranteAuthorizationTargetService.java

  ```java
  @Service
  @AllArgsConstructor
  class RestauranteAuthorizationTargetService implements AuthorizationTargetService {

    private RestauranteRepository restauranteRepo;

    @Override
    public Long getTargetIdByUser(User user) {
      if (user.isInRole(Role.ROLES.PARCEIRO)) {

        R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶ ̶=̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶R̶e̶p̶o̶.̶f̶i̶n̶d̶B̶y̶U̶s̶e̶r̶(̶u̶s̶e̶r̶)̶;̶
        Restaurante restaurante = restauranteRepo.findByUserId(user.getId()); // modificado

        if (restaurante != null) {
          return restaurante.getId();
        }
      }
      return null;
    }

  }
  ```

6. Mude o `monolito.properties` do `config-repo`, removendo a configuração de expiração do token JWT. Essa configuração será usada apenas pelo gerador de tokens, o API Gateway.

  ####### config-repo/monolito.properties

  ```properties
  j̶w̶t̶.̶e̶x̶p̶i̶r̶a̶t̶i̶o̶n̶ ̶=̶ ̶6̶0̶4̶8̶0̶0̶0̶0̶0̶
  ```

7. Execute o `EatsApplication` do módulo `eats-application` do monólito.  Certifique-se que o Service Registry, Config Server e API Gateway estejam sendo executados.

  As URLS que não tem acesso protegido continuam funcionando. Por exemplo, acesse, pelo navegador, a URL a seguir para obter todas as formas de pagamento:

  http://localhost:9999/formas-de-pagamento

  ou

  http://localhost:8080/formas-de-pagamento


  Deve funcionar e retornar algo como:

  ```json
  [{"id":4,"tipo":"VALE_REFEICAO","nome":"Alelo"},{"id":3,"tipo":"CARTAO_CREDITO","nome":"Amex"},{"id":2,"tipo":"CARTAO_CREDITO","nome":"MasterCard"},{"id":6,"tipo":"CARTAO_DEBITO","nome":"MasterCard Maestro"},{"id":5,"tipo":"VALE_REFEICAO","nome":"Ticket Restaurante"},{"id":1,"tipo":"CARTAO_CREDITO","nome":"Visa"},{"id":7,"tipo":"CARTAO_DEBITO","nome":"Visa Débito"}]
  ```

  Porém, URLs protegidas precisarão de um _access token_ válido e que foi emitido para um usuário que tenha permissão para fazer operações no recurso solicitado.

  Abra um Terminal e tente modificar o nome de uma forma de pagamento usando o cURL:

  ```sh
  curl -i -X PUT -H 'Content-type: application/json' -d '{"id": 3, "tipo": "CARTAO_CREDITO", "nome": "American Express"}' http://localhost:9999/admin/formas-de-pagamento/3
  ```

  O comando anterior pode ser encontrado em: https://gitlab.com/snippets/1888251

  A resposta será um erro HTTP 401 (Unauthorized), com uma mensagem de acesso negado. Algo como:

  ```txt
  HTTP/1.1 401 
  Date: Fri, 23 Aug 2019 00:50:11 GMT
  X-Content-Type-Options: nosniff
  X-XSS-Protection: 1; mode=block
  Cache-Control: no-cache, no-store, max-age=0, must-revalidate
  Pragma: no-cache
  Expires: 0
  X-Frame-Options: DENY
  Content-Type: application/json;charset=UTF-8
  Transfer-Encoding: chunked

  {"timestamp":"2019-08-23T00:50:11.652+0000","status":401,"error":"Unauthorized","message":"Você não está autorizado a acessar esse recurso.","path":"/admin/formas-de-pagamento/3"}
  ```

  Use o token obtido no exercício anterior, de autenticação no API Gateway, colocando-o no cabeçalho HTTP `Authorization`, depois do valor `Bearer`. Faça o seguinte comando cURL em um Terminal:

  ```sh
  curl -i -X PUT -H 'Content-type: application/json' -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJDYWVsdW0gRWF0cyIsInN1YiI6IjEiLCJyb2xlcyI6WyJBRE1JTiJdLCJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTY2NTE4NzIyLCJleHAiOjE1NjcxMjM1MjJ9.FmH2QkryLBxWZjt2DMKHsCmjQNCmk3hrRAC0keam5_w' -d '{"id": 3, "tipo": "CARTAO_CREDITO", "nome": "American Express"}' http://localhost:9999/admin/formas-de-pagamento/3
  ```

  Você pode encontrar o comando anterior em: https://gitlab.com/snippets/1888252

  Deverá ser obtida uma resposta bem sucedida, com os dados da forma de pagamento alterados!

  ```txt
  HTTP/1.1 200 
  Date: Fri, 23 Aug 2019 00:56:00 GMT
  X-Content-Type-Options: nosniff
  X-XSS-Protection: 1; mode=block
  Cache-Control: no-cache, no-store, max-age=0, must-revalidate
  Pragma: no-cache
  Expires: 0
  X-Frame-Options: DENY
  Content-Type: application/json;charset=UTF-8
  Transfer-Encoding: chunked

  {"id":3,"tipo":"CARTAO_CREDITO","nome":"American Express"}
  ```

  Isso indica que o módulo de segurança do monólito reconheceu o token como válido e extraiu a informação dos roles do usuário, reconhecendo-o no role ADMIN.

<!-- ## Extraindo um serviço Administrativo -->

<!-- ## OAuth -->

<!-- ## Protegendo o Config Server e o Service Registry -->

<!-- ## Mutual Authentication -->

<!-- ## Data at transit and data at rest -->

<!-- ## Rotação de credenciais e Vault -->

<!-- 
Referências:

https://blog.caelum.com.br/morte-a-sessao-entenda-esse-tal-de-stateless-session-com-tokens/

https://scotch.io/tutorials/the-ins-and-outs-of-token-based-authentication

https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm

https://microservices.io/patterns/security/access-token.html

Microservices Patterns
Chris Richardson - Manning
Capítulo 11 (Developing production-ready services) - Seção 11.1 (DEVELOPING SECURE SERVICES)
https://learning.oreilly.com/library/view/microservices-patterns/9781617294549/kindle_split_019.html


Microservices for the Enterprise: Designing, Developing, and Deploying
Kasun Indrasiri; Prabath Siriwardena - Apress 2018
Capítulo 11 (Microservices Security Fundamentals)
Capítulo 12 (Securing Microservices)
https://learning.oreilly.com/library/view/microservices-for-the/9781484238585/html/461146_1_En_11_Chapter.xhtml

Building Secure Microservices Architectures
Sam Newman - O'Reilly Media, Inc.
April 2018
https://learning.oreilly.com/learning-paths/learning-path-building/9781492041481/
-->

