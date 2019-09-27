# Segurança

## Extraindo um serviço Administrativo do monólito

Primeiramente, vamos extrair o módulo `eats-administrativo` do monólito para um serviço `eats-administrativo-service`.

Para isso, criamos um novo projeto Spring Boot com as seguintes dependências:

- Spring Boot DevTools
- Spring Boot Actuator
- Spring Data JPA
- Spring Web Starter
- Config Client
- Eureka Discovery Client
- Zipkin Client

Então, movemos as seguintes classes do módulo Administrativo do monólito para o novo serviço:

- FormaDePagamento
- FormaDePagamentoController
- FormaDePagamentoRepository
- TipoDeCozinha
- TipoDeCozinhaController
- TipoDeCozinhaRepository

O serviço administrativo deve apontar para o Config Server, definindo um `bootstrap.properties` com `administrativo` como _application name_. No arquivo `administrativo.properties` do `config-repo`, definiremos as configurações de data source.

Inicialmente, o serviço administrativo pode apontar para o mesmo BD do monólito. Aos poucos, deve ser feita a migração das tabelas `forma_de_pagamento` e `tipo_de_cozinha` para um BD próprio.

No `application.properties`, deve ser definida `8084` na porta a ser utilizada.

Então, o módulo `eats-administrativo` do monólito pode ser removido, assim como suas autorizações no módulo `eats-seguranca`.

Remova a dependência a `eats-administrativo` do `pom.xml` do módulo `eats-application` do monólito:

####### fj33-eats-monolito-modular/eats/eats-application/pom.xml

```xml
<̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
̶ ̶ ̶<̶g̶r̶o̶u̶p̶I̶d̶>̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶<̶/̶g̶r̶o̶u̶p̶I̶d̶>̶
̶ ̶<̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶e̶a̶t̶s̶-̶a̶d̶m̶i̶n̶i̶s̶t̶r̶a̶t̶i̶v̶o̶<̶/̶a̶r̶t̶i̶f̶a̶c̶t̶I̶d̶>̶
̶ ̶ ̶<̶v̶e̶r̶s̶i̶o̶n̶>̶$̶{̶p̶r̶o̶j̶e̶c̶t̶.̶v̶e̶r̶s̶i̶o̶n̶}̶<̶/̶v̶e̶r̶s̶i̶o̶n̶>̶
̶<̶/̶d̶e̶p̶e̶n̶d̶e̶n̶c̶y̶>̶
```

Faço o mesmo nos arquivos `pom.xml` dos módulos `eats-restaurante` e `eats-pedido` do monólito.

No projeto pai dos módulos, o projeto `eats`, remova o módulo `eats-administrativo`  do `pom.xml`:

####### fj33-eats-monolito-modular/eats/pom.xml

```xml
<modules>
  <̶m̶o̶d̶u̶l̶e̶>̶e̶a̶t̶s̶-̶a̶d̶m̶i̶n̶i̶s̶t̶r̶a̶t̶i̶v̶o̶<̶/̶m̶o̶d̶u̶l̶e̶>̶
  <module>eats-restaurante</module>
  <module>eats-pedido</module>
  <module>eats-seguranca</module>
  <module>eats-application</module>
</modules>
```

Apague o módulo `eats-administrativo` do monólito. Pelo Eclipse, tecle _Delete_ em cima do módulo, selecione a opção _Delete project contents on disk (cannot be undone)_ e clique em _OK_.

Remova, da classe `SecurityConfig` do módulo `eats-seguranca` do monólito,  as configurações de autorização dos endpoints que foram movidos:

####### fj33-eats-monolito-modular/eats/eats-seguranca/src/main/java/br/com/caelum/eats/SecurityConfig.java

```java
class SecurityConfig extends WebSecurityConfigurerAdapter {

  // código omitido ...

  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
      .antMatchers("/restaurantes/**", "/pedidos/**",̶ ̶"̶/̶t̶i̶p̶o̶s̶-̶d̶e̶-̶c̶o̶z̶i̶n̶h̶a̶/̶*̶*̶"̶,̶ ̶"̶/̶f̶o̶r̶m̶a̶s̶-̶d̶e̶-̶p̶a̶g̶a̶m̶e̶n̶t̶o̶/̶*̶*̶"̶).permitAll()
      .antMatchers("/actuator/**").permitAll()
      .̶a̶n̶t̶M̶a̶t̶c̶h̶e̶r̶s̶(̶"̶/̶a̶d̶m̶i̶n̶/̶*̶*̶"̶)̶.̶h̶a̶s̶R̶o̶l̶e̶(̶R̶o̶l̶e̶.̶R̶O̶L̶E̶S̶.̶A̶D̶M̶I̶N̶.̶n̶a̶m̶e̶(̶)̶)̶
      // código omitido ...
  }

}
```

Também é necessário alterar as referências às classes `TipoDeCozinha` e `FormaDePagamento` no `DistanciaRestClientWiremockTest` do módulo `eats-application`.

Já no módulo de restaurantes do monólito, é preciso alterar referências às classes migradas para o serviço Administrativo para apenas utilizarem os ids dos agregados `TipoDeCozinha` e `FormaDePagamento`. Isso afeta diversas classes do módulo `eats-restaurante`:

- Restaurante
- RestauranteController
- RestauranteDto
- RestauranteFormaDePagamento
- RestauranteFormaDePagamentoController
- RestauranteFormaDePagamentoRepository
- RestauranteParaServicoDeDistancia
- RestauranteRepository
- RestauranteService

A UI também será afetada. Uma das mudanças é que chamadas relativas a tipos de cozinha e formas de pagamento devem ser direcionadas para o serviço Administrativo. Esse serviço registra-se no Eureka Server com o nome `administrativo`, o seu application name. O API Gateway faz o roteamento dinâmico baseado nas instâncias disponíveis no Service Registry. Por isso, podemos trocar chamadas como a seguinte para utilizarem o prefixo `administrativo`:

####### fj33-eats-ui/src/app/services/tipo-de-cozinha.service.ts

```typescript
export class TipoDeCozinhaService {

  p̶r̶i̶v̶a̶t̶e̶ ̶A̶P̶I̶ ̶=̶ ̶e̶n̶v̶i̶r̶o̶n̶m̶e̶n̶t̶.̶b̶a̶s̶e̶U̶r̶l̶;̶
  private API = environment.baseUrl + '/administrativo';

  // código omitido ...

}
```

O mesmo deve ser feito para a classe `FormaDePagamentoService`.

As diversas mudanças no módulo de restaurantes do monólito também afetam a UI.

![Serviço administrativo extraído do monólito {w=93}](imagens/15-seguranca/administrativo-extraido-do-monolito.png)

## Exercício: um serviço Administrativo

1. Crie um arquivo `administrativo.properties` no `config-repo`, definindo um data source que aponta para o mesmo BD do monólito:

  ####### config-repo/administrativo.properties

  ```properties
  spring.datasource.url=jdbc:mysql://localhost/eats?createDatabaseIfNotExist=true
  spring.datasource.username=root
  spring.datasource.password=
  ```

2. Clone o projeto `fj33-eats-administrativo-service` para o seu Desktop:

  ```sh
  cd ~/Desktop
  git clone https://gitlab.com/aovs/projetos-cursos/fj33-eats-administrativo-service.git
  ```

  No Eclipse, no workspace de microservices, importe o projeto `fj33-eats-administrativo-service`, usando o menu _File > Import > Existing Maven Projects_.

  Com o Service Registry e o Config Server no ar, suba o serviço Administrativo executando a classe `EatsAdministrativoServiceApplication`.

3. Faça checkout da branch `cap15-extrai-administrativo-service` do monólito modular e da UI:

  ```sh
  cd ~/Desktop/fj33-eats-monolito-modular
  git checkout -f cap15-extrai-administrativo-service

  cd ~/Desktop/fj33-eats-ui
  git checkout -f cap15-extrai-administrativo-service
  ```

  Faça refresh do monólito modular no Eclipse.

  Suba os serviços e o front-end. Teste o Caelum Eats. Deve funcionar!

## Autenticação e Autorização

Grande parte das aplicações tem diferentes perfis de usuário, que têm permissão de acesso a diferentes funcionalidades. Isso é o que chamamos de **Autorização**.

No caso do Caelum Eats, qualquer usuário pode acessar fazer pedidos e acompanhá-los. Porém, a administração do sistema, que permite cadastrar tipos de cozinha, formas de pagamento e aprovar restaurantes, só é acessível pelo perfil de administrador. Já os dados de um restaurante e o gerenciamento dos pedidos pendentes só são acessíveis pelo dono de cada restaurante.

Um usuário precisa identificar-se, ou seja, dizer quem está acessando a aplicação. Isso é o que chamamos de **Autenticação**.

Uma vez que o usuário está autenticado e sua identidade é conhecida, é possível que a aplicação reforce as permissões de acesso.

Existem algumas maneiras mais comuns de um sistema confirmar a identidade de um usuário:

- algo que o usuário sabe, um segredo, como uma senha
- algo que o usuário tem, como um token físico ou por uma app mobile
- algo que o usuário é, como biometria das digitais, íris ou reconhecimento facial

> Two-factor authentication (2FA), ou autenticação de dois fatores, é a associação de duas formas de autenticação para minimizar as chances de alguém mal intencionado identificar-se como outro usuário, no caso de apoderar-se de um dos fatores de autenticação.

## Sessões e escalabilidade

Após a autenticação, uma aplicação Web tradicional guarda a identidade do usuário em uma **sessão**, que comumente é armazenada em memória, mas pode ser armazenada em disco ou em um BD.

O cliente da aplicação, em geral um navegador, deve armazenar um id da sessão. Em toda requisição, o cliente passa esse id para identificar o usuário.

O que acontece quando há um aumento drástico no número de usuários em momento de pico de uso, como na Black Friday?

Se a aplicação suportar esse aumento na carga, podemos dizer que possui a característica arquitetural da **Escalabilidade**. Quando a escalabilidade é atingida aumentando o número de máquinas, dizemos que é a escalabilidade horizontal.

Mas, se escalarmos horizontalmente a aplicação, onde fica armazenada a sessão se temos mais de uma máquina como servidor Web? Uma estratégia são as _sticky sessions_, em que cada usuário tem sua sessão em uma máquina específica.

Mas quando alguma máquina falhar, o usuário seria deslogado e não teria mais acesso às funcionalidades. Para que a experiência do usuário seja transparente, de maneira que ele não perceba a falha em uma máquina, há a técnica da **replicação de sessão**, em que cada servidor compartilha, pela rede, suas sessões com outros servidores. Isso traz uma sobrecarga de processamento, armazenamento e tráfego na rede.

## REST, stateless sessions e self-contained tokens

Em sua tese de doutorado _Architectural Styles and the Design of Network-based Software Architectures_, Roy Fielding descreve o estilo arquitetural da Web e o chama de **Representational State Transfer (REST)**. Uma das características do REST é que a comunicação deve ser **Stateless**: toda informação deve estar contida na requisição do cliente ao servidor, sem a necessidade de nenhum contexto armazenado no servidor.

Manter sessões nos servidores é manter estado. Portanto, podemos dizem que utilizar sessões não é RESTful porque não segue a característica do REST de ser stateless.

Mas então como fazer um mecanismo de autenticação que seja stateless e, por consequência, mais próximo do REST?

Usando tokens! Há tokens opacos, que são apenas um texto randômico e que não carregam nenhuma informação. Porém, há os **self-contained tokens**, que contém informações sobre o usuário e/ou sobre o sistema cliente. Cada requisição teria um self-contained token em seu cabeçalho, com todas as informações necessárias para a aplicação. Assim, tiramos a necessidade de armazenamento da sessão no lado do servidor.

A grande questão é como ter um token que contém informações e, ao mesmo tempo, garantir sua integridade, confirmando que os dados do token não foram manipulados?

## JWT e JWS

JWT (JSON Web Token) é um formato de token compacto e self-contained que serve propagar informações de identidade, permissões de um usuário em uma aplicação de maneira segura. Foi definido na RFC 7519 da Internet Engineering Task Force (IETF), em Maio de 2015.

O Working Group da IETF chamado Javascript Object Signing and Encryption (JOSE), definiu duas outras RFCs relacionadas:

- JSON Web Signature (JWS), definido na RFC 7515, que representa em JSON conteúdo assinado digitalmente
- JSON Web Encryption (JWE), definido na RFC 7516, que representa em JSON conteúdo criptografado

Para garantir a integridade dos dados de um token, é suficiente usarmos o JWS.

Um JWS consiste de três partes, separadas por `.`:

```txt
BASE64URL(UTF8(Cabeçalho)) || '.' ||
BASE64URL(Payload) || '.' ||
BASE64URL(Assinatura JWS)
```

Todas as partes do JWS são codificadas em _Base64 URL encoded_. Base64 é uma representação em texto de dados binários. URL encoded significa que caracteres especiais são codificados com `%`, da mesma maneira como são passados via parâmetros de URLs.

Um exemplo de um JWS usado no Caelum Eats seria o seguinte:

```txt
eyJhbGciOiJIUzI1NiJ9.
eyJpc3MiOiJDYWVsdW0gRWF0cyIsInN1YiI6IjIiLCJyb2xlcyI6WyJQQVJDRUlSTyJdLCJ1c2VybmFtZSI6ImxvbmdmdSIsImlhdCI6MTU2NjQ5ODA5MSwiZXhwIjoxNTY3MTAyODkxfQ.
GOwiEeJMP9t0tV2lQpNiDU211WKL6h5Z6OkNcA-f4EY
```

Os trechos anteriores podem ser descodificados de Base64 para texto normal usando um site como: http://www.base64url.com/

O primeiro trecho, `eyJhbGciOiJIUzI1NiJ9`, é o cabeçalho do JWS. Quando descodificado, é:

```json
{"alg":"HS256"}
```

O valor de `alg` indica que foi utilizado o algoritmo HMAC (hash-based message authentication code) com SHA-256 como função de hash. Nesse algoritmo, há uma chave secreta (um texto) simétrica, que deve ser conhecida tanto pela parte que cria o token como pela parte que o validará. Se essa chave secreta for descoberta por um agente mal intencionado, pode ser usada para gerar tokens válidos deliberadamente.

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

O valor de `iss` é o issuer, a aplicação que gerou o token.
O valor de `sub` é o subject, que contém informações do usuário.
Os valores de `iat` e `exp`, são as datas de geração e expiração do token, respectivamente.
Os demais valores são _claims_ customizadas, que declaram informações adicionais do usuário.

O terceiro trecho, `GOwiEeJMP9t0tV2lQpNiDU211WKL6h5Z6OkNcA-f4EY`, é a assinatura do JWS e não pode ser decodificada para um texto. O que importam são os bytes.

> No site https://jwt.io/ conseguimos obter o algoritmo utilizado, os dados do payload e até validar um JWT.

Se soubermos a chave secreta, podemos verificar se a assinatura bate com o payload do JWS. Se bater, o token é válido. Dessa maneira, conseguimos garantir que não houve manipulação dos dados e, portanto, sua integridade.

Um detalhe importante é que um JWS não garante a confidencialidade dos dados. Se houver algum software bisbilhotando os dados trafegados na rede, o payload do JWS pode ser lido, já que é apenas codificado em Base64 URL encoded. A confidencialidade pode ser reforçada por meio de TLS no canal de comunicação ou por meio de JWE.

> Uma grande desvantagem de um JWT é que o token é irrevogável antes de sua expiração. Isso implica que, enquanto o token não estiver espirado será válido. Por isso, implementar um mecanismo de logout pelo usuário passa a ser complicado. Poderíamos trabalhar com intervalos pequenos de expiração, mas isso afetaria a experiência do usuário, já que frequentemente a expiração levaria o usuário a efetuar novo login. Uma maneira comum de implementar logout é ter um cache com JWT invalidados. Porém, isso nos leva novamente a uma solução _stateful_.

## Stateless Sessions no Caelum Eats

Até o momento, um login de um dono de restaurante ou do administrador do sistema dispara a execução do `AuthenticationController` do módulo `eats-seguranca` do monólito. No método `authenticate`, é gerado e retornado um token JWS.

O token JWS é armazenado em um `localStorage` no front-end. Há um _interceptor_ do Angular que, antes de cada requisição AJAX, adiciona o cabeçalho `Authorization: Bearer` com o valor do token armazenado.

No back-end, a classe `JwtAuthenticationFilter` é executada a cada requisição e o token JWS é extraído dos cabeçalhos HTTP e validado. Caso seja válido, é recuperado o `sub` (Subject) e obtido o usuário do BD com seus ROLEs (`ADMIN` ou `PARCEIRO`), setando um `Authentication` no contexto de segurança:

####### fj33-eats-monolito-modular/eats/eats-seguranca/src/main/java/br/com/caelum/eats/seguranca/JwtAuthenticationFilter.java

```java
@Component
@AllArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

  private JwtTokenManager tokenManager;
  private UserService usersService;

  @Override
  protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain chain)
      throws ServletException, IOException {
    String jwt = getTokenFromRequest(request);
    if (tokenManager.isValid(jwt)) {
      Long userId = tokenManager.getUserIdFromToken(jwt);
      UserDetails userDetails = usersService.loadUserById(userId);
      UsernamePasswordAuthenticationToken authentication = new UsernamePasswordAuthenticationToken(userDetails,
          null, userDetails.getAuthorities());
      SecurityContextHolder.getContext().setAuthentication(authentication);
    }

    chain.doFilter(request, response);
  }

  private String getTokenFromRequest(HttpServletRequest request) {
   // código omitido ...
  }

}
```

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
              .claim("username", user.getName())
              .claim("roles", user.getRoles())
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

Antes da extração do serviço administrativo, para as URLs que começavam com `/admin`, o ROLE do usuário deveria ser `ADMIN` e teria acesso a tudo relativo à administração da aplicação. Esse tipo de autorização, em que um determinado ROLE tem acesso a qualquer endpoint relacionado é o que chamamos de _role-based authorization_.

Porém, ao extrairmos o serviço administrativo, perdemos a autorização feita no módulo de segurança do monólito. Ainda não implementamos autorização no novo serviço.

No caso da URL começar com `/parceiros/restaurantes/do-usuario/{username}` ou `/parceiros/restaurantes/{restauranteId}`, é necessária uma autorização mais elaborada, que verifica se o usuário tem permissão a um restaurante específico, por meio da classe `RestauranteAuthorizationService`. Esse tipo de autorização, em que um usuário ter permissão em apenas alguns objetos de negócio é o que chamamos de _ACL-based authorization_. A sigla ACL significa _Access Control List_.

![Geração e validação de tokens no módulo de segurança do monólito {w=39}](imagens/15-seguranca/geracao-e-validacao-de-tokens-no-modulo-de-seguranca-do-monolito.png)

## Autenticação com Microservices e Single Sign On

Poderíamos implementar a autenticação numa Arquitetura de Microservices de duas maneiras:

- o usuário precisa autenticar novamente ao acessar cada serviço
- a autenticação é feita apenas uma vez e as informações de identidade do usuário são repassadas para os serviços

Autenticar várias vezes, a cada serviço, é algo que deixaria a experiência do usuário terrível. Além disso, todos os serviços teriam que ser _edge services_, expostos à rede externa.

Autenticar apenas uma vez e repassar as dados do usuário autenticado permite que os serviços não fiquem expostos, diminuindo a superfície de ataque. Além disso, a experiência para o usuário é transparente, como se todas as funcionalidades fossem parte da mesma aplicação. É esse tipo de solução que chamamos de **Single Sign On** (SSO).

## Autenticação no API Gateway e Autorização nos Serviços

No livro Microservice Patterns, Chris Richardson descreve uma maneira comum de lidar com autenticação em uma arquitetura de Microservices: implementá-la API Gateway, o único edge service que fica exposto para o mundo externo. Dessa maneira, as chamadas a URLs protegidas já seriam barradas antes de passar para a rede interna, no caso do usuário não estar autenticado.

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

## Autenticação e Autorização nos Microservices do Caelum Eats

A solução de stateless sessions com JWT do Caelum Eats, foi pensada e implementada visando uma aplicação monolítica.

E o resto dos serviços?

Temos serviços de infraestrutura como:

- API Gateway
- Service Registry
- Config Server
- Hystrix Dashboard
- Turbine
- Admin Server

Como estamos tratando de autorização relacionada a um determinado usuário, deixaremos para um outro momento a discussão da autenticação e autorização desses serviços de infraestrutura.

Temos serviços alinhados a contextos delimitados (bounded contexts) da Caelum Eats, como:

- Distância
- Pagamento
- Nota Fiscal
- Administrativo, um novo serviço que acabamos de extrair

Há ainda módulos do monólito relacionados a contextos delimitados:

- Pedido
- Restaurante

O único módulo cujos endpoints tem seu acesso protegido é o módulo de Restaurante do monólito. O módulo Administrativo foi extraído para um serviço próprio e não implementamos a autorização.

O monólito possui também um módulo de Segurança, que trata desse requisito transversal e contém o código de configuração do Spring Security.

O módulo Administrativo do monólito era protegido por meio de role-based authorization, bastando o usuário estar no role ADMIN para acessar os endpoints de administração de tipos de cozinha e formas de pagamento. Esse tipo de autorização não está sendo feito no `eats-administrativo-service`.

Já o módulo de Restaurante efetua ACL-based authorization, limitando o acesso do usuário com role PARCEIRO a um restaurante específico.

Vamos modificar esse cenário, passando a responsabilidade de geração de tokens JWT/JWS para o API Gateway, que também será responsável pelo cadastro de novos usuários. A validação do token e autorização dos recursos ficará a cargo do módulo Restaurante do monólito e do serviço Administrativo.

![Geração de tokens no API Gateway e validação no módulo de segurança do monólito e no serviço Administrativo {w=39}](imagens/15-seguranca/geracao-de-tokens-no-api-gateway-e-validacao-no-modulo-de-seguranca-do-monolito.png)

## Autenticação no API Gateway

Poderíamos ter um BD específico para conter dados de usuários nas tabelas `user`, `role` e `user_authorities`. Porém, para simplificar, vamos manter os dados de usuários no BD do próprio monólito.

No `config-repo`, adicione um arquivo `apigateway.properties` com o dados de conexão do BD do monólito, além das configurações da chave e expiração do JWT, que são usadas na geração do token:

####### config-repo/apigateway.properties

```properties
#DATASOURCE CONFIGS
spring.datasource.url=jdbc:mysql://localhost/eats?createDatabaseIfNotExist=true
spring.datasource.username=root
spring.datasource.password=

#JWT CONFIGS
jwt.secret = um-segredo-bem-secreto
jwt.expiration = 604800000
```

Adicione, ao API Gateway, dependências ao starter do Spring Data JPA e ao driver do MySQL. Adicione também o JJWT, biblioteca que gera e valida tokens JWT, e ao starter do Spring Security.

####### fj33-api-gateway/pom.xml

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

Copie as classes a seguir do módulo `eats-seguranca` do monólito para o pacote `br.com.caelum.apigateway.seguranca` do API Gateway:

- AuthenticationController
- AuthenticationDto
- Role
- User
- UserInfoDto
- UserRepository
- UserService

Copie a seguinte classe do módulo de segurança do monólito para o pacote `br.com.caelum.apigateway` do API Gateway:

- PasswordEncoderConfig

Não esqueça de ajustar o pacote das classes copiadas.

Essas classes fazem a autenticação de usuários, assim como o cadastro de novos donos de restaurante.

Defina uma classe `SecurityConfig` no pacote `br.com.caelum.apigateway` para que permita toda e qualquer requisição, desabilitando a autorização, que será feita pelos serviços. A autenticação será _stateless_.

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/SecurityConfig.java

```java
@Configuration
@EnableWebSecurity
@AllArgsConstructor
class SecurityConfig extends WebSecurityConfigurerAdapter {

  private UserService userService;
  private PasswordEncoder passwordEncoder;

  @Override
  protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
        .anyRequest().permitAll()
        .and().cors()
        .and().csrf().disable()
        .formLogin().disable()
        .httpBasic().disable()
        .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
  }

  @Override
  protected void configure(final AuthenticationManagerBuilder auth) throws Exception {
    auth.userDetailsService(userService).passwordEncoder(passwordEncoder);
  }

  @Override
  @Bean(BeanIds.AUTHENTICATION_MANAGER)
  public AuthenticationManager authenticationManagerBean() throws Exception {
    return super.authenticationManagerBean();
  }

}
```

Não deixe de fazer os imports corretos:

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/SecurityConfig.java

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.config.BeanIds;
import org.springframework.security.config.annotation.authentication.builders.AuthenticationManagerBuilder;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configuration.WebSecurityConfigurerAdapter;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.crypto.password.PasswordEncoder;

import br.com.caelum.apigateway.seguranca.UserService;
import lombok.AllArgsConstructor;
```

Defina, no pacote `br.com.caelum.apigateway.seguranca` do API Gateway, uma classe `JwtTokenManager`, responsável pela geração dos tokens. A validação e extração de informações de um token serão responsabilidade de cada serviço.

É importante adicionar o username e os roles do usuário aos _claims_ do JWT.

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/seguranca/JwtTokenManager.java

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
        .claim("roles", user.getRoles())
        .claim("username", user.getUsername())
        .setIssuedAt(now)
        .setExpiration(expiration)
        .signWith(SignatureAlgorithm.HS256, this.secret)
        .compact();
  }

}
```

Cheque os imports:

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/seguranca/JwtTokenManager.java

```java
import java.util.Date;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
```

Ainda no API Gateway, adicione um _forward_ para a URL do `AuthenticationController`, de maneira que o Zuul não tente fazer o proxy dessa chamada:

####### fj33-api-gateway/src/main/resources/application.properties

```properties
zuul.routes.auth.path=/auth/**
zuul.routes.auth.url=forward:/auth
```

_Observação: essa configuração deve ficar antes da rota "coringa", que direciona todas as requisições para o monólito._

Podemos fazer uma chamada como a seguinte, que autentica o dono do restaurante Long Fu:

```sh
curl -i -X POST -H 'Content-type: application/json' -d '{"username":"longfu", "password":"123456"}' http://localhost:9999/auth
```

O retorno obtido será algo como:

```txt
{"userId":2,"username":"longfu","roles":["PARCEIRO"],"token":"eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJDYWVsdW0gRWF0cyIsInN1YiI6IjIiLCJyb2xlcyI6WyJQQVJDRUlSTyJdLCJ1c2VybmFtZSI6ImxvbmdmdSIsImlhdCI6MTU2OTU1MDYyOCwiZXhwIjoxNTcwMTU1NDI4fQ.S1V7aBNN206NpxPEEaIibtluJD9Bd-gHPK-MaUHcgxs"}
```

São retornados, no corpo da resposta, informações sobre o usuário, seus roles e um token. O token, no formato JWS, contém as mesmas informações do corpo da resposta, mas em base 64, e uma assinatura.

## Validando o token JWT e implementando autorização no Monólito

Remova as seguintes classes do módulo `eats-seguranca` do monólito, cujas responsabilidades foram passadas para o API Gateway. Elas estão no pacote `br.com.caelum.eats.seguranca`:

- AuthenticationController
- AuthenticationDto
- UserInfoDto
- UserRepository
- UserService

Remova também a classe a seguir, do pacote `br.com.caelum.eats`:

- PasswordEncoderConfig

A classe `SecurityConfig` deve apresentar um erro de compilação.

Altere a classe `SecurityConfig` do módulo de segurança do monólito, removendo o código associado a autenticação e cadastro de novos usuários:

####### fj33-eats-monolito-modular/eats/eats-seguranca/src/main/java/br/com/caelum/eats/SecurityConfig.java

```java
@Configuration
@EnableWebSecurity
@AllArgsConstructor
class SecurityConfig extends WebSecurityConfigurerAdapter {

  p̶r̶i̶v̶a̶t̶e̶ ̶U̶s̶e̶r̶S̶e̶r̶v̶i̶c̶e̶ ̶u̶s̶e̶r̶S̶e̶r̶v̶i̶c̶e̶;̶
  private JwtAuthenticationFilter jwtAuthenticationFilter;
  private JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint;
  p̶r̶i̶v̶a̶t̶e̶ ̶P̶a̶s̶s̶w̶o̶r̶d̶E̶n̶c̶o̶d̶e̶r̶ ̶p̶a̶s̶s̶w̶o̶r̶d̶E̶n̶c̶o̶d̶e̶r̶;̶

  // código omitido...

  @̶O̶v̶e̶r̶r̶i̶d̶e̶
  p̶r̶o̶t̶e̶c̶t̶e̶d̶ ̶v̶o̶i̶d̶ ̶c̶o̶n̶f̶i̶g̶u̶r̶e̶(̶f̶i̶n̶a̶l̶ ̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶M̶a̶n̶a̶g̶e̶r̶B̶u̶i̶l̶d̶e̶r̶ ̶a̶u̶t̶h̶)̶ ̶t̶h̶r̶o̶w̶s̶ ̶E̶x̶c̶e̶p̶t̶i̶o̶n̶ ̶{̶
    a̶u̶t̶h̶.̶u̶s̶e̶r̶D̶e̶t̶a̶i̶l̶s̶S̶e̶r̶v̶i̶c̶e̶(̶u̶s̶e̶r̶S̶e̶r̶v̶i̶c̶e̶)̶.̶p̶a̶s̶s̶w̶o̶r̶d̶E̶n̶c̶o̶d̶e̶r̶(̶t̶h̶i̶s̶.̶p̶a̶s̶s̶w̶o̶r̶d̶E̶n̶c̶o̶d̶e̶r̶)̶;̶
  }̶

  @̶O̶v̶e̶r̶r̶i̶d̶e̶
  @̶B̶e̶a̶n̶(̶B̶e̶a̶n̶I̶d̶s̶.̶A̶U̶T̶H̶E̶N̶T̶I̶C̶A̶T̶I̶O̶N̶_̶M̶A̶N̶A̶G̶E̶R̶)̶
  p̶u̶b̶l̶i̶c̶ ̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶M̶a̶n̶a̶g̶e̶r̶ ̶a̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶M̶a̶n̶a̶g̶e̶r̶B̶e̶a̶n̶(̶)̶ ̶t̶h̶r̶o̶w̶s̶ ̶E̶x̶c̶e̶p̶t̶i̶o̶n̶ ̶{̶
    r̶e̶t̶u̶r̶n̶ ̶s̶u̶p̶e̶r̶.̶a̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶M̶a̶n̶a̶g̶e̶r̶B̶e̶a̶n̶(̶)̶;̶
  }̶


}
```

Remove os seguintes imports:

####### fj33-eats-monolito-modular/eats/eats-seguranca/src/main/java/br/com/caelum/eats/SecurityConfig.java

```java
i̶m̶p̶o̶r̶t̶ ̶o̶r̶g̶.̶s̶p̶r̶i̶n̶g̶f̶r̶a̶m̶e̶w̶o̶r̶k̶.̶c̶o̶n̶t̶e̶x̶t̶.̶a̶n̶n̶o̶t̶a̶t̶i̶o̶n̶.̶B̶e̶a̶n̶;̶
̶i̶m̶p̶o̶r̶t̶ ̶o̶r̶g̶.̶s̶p̶r̶i̶n̶g̶f̶r̶a̶m̶e̶w̶o̶r̶k̶.̶s̶e̶c̶u̶r̶i̶t̶y̶.̶a̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶.̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶M̶a̶n̶a̶g̶e̶r̶;̶
̶i̶m̶p̶o̶r̶t̶ ̶o̶r̶g̶.̶s̶p̶r̶i̶n̶g̶f̶r̶a̶m̶e̶w̶o̶r̶k̶.̶s̶e̶c̶u̶r̶i̶t̶y̶.̶c̶o̶n̶f̶i̶g̶.̶B̶e̶a̶n̶I̶d̶s̶;̶
̶i̶m̶p̶o̶r̶t̶ ̶o̶r̶g̶.̶s̶p̶r̶i̶n̶g̶f̶r̶a̶m̶e̶w̶o̶r̶k̶.̶s̶e̶c̶u̶r̶i̶t̶y̶.̶c̶o̶n̶f̶i̶g̶.̶a̶n̶n̶o̶t̶a̶t̶i̶o̶n̶.̶a̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶.̶b̶u̶i̶l̶d̶e̶r̶s̶.̶A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶M̶a̶n̶a̶g̶e̶r̶B̶u̶i̶l̶d̶e̶r̶;̶
̶i̶m̶p̶o̶r̶t̶ ̶o̶r̶g̶.̶s̶p̶r̶i̶n̶g̶f̶r̶a̶m̶e̶w̶o̶r̶k̶.̶s̶e̶c̶u̶r̶i̶t̶y̶.̶c̶r̶y̶p̶t̶o̶.̶p̶a̶s̶s̶w̶o̶r̶d̶.̶P̶a̶s̶s̶w̶o̶r̶d̶E̶n̶c̶o̶d̶e̶r̶;̶
̶i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶s̶e̶g̶u̶r̶a̶n̶c̶a̶.̶U̶s̶e̶r̶S̶e̶r̶v̶i̶c̶e̶;̶
```

Modifique o `JwtTokenManager` do módulo de segurança do monólito, removendo o código de geração de token e o atributo `expirationInMillis`, deixando apenas a validação e extração de dados do token.

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

  ̶ ̶ ̶}̶

  public boolean isValid(String jwt) {
    // não modificado ...
  }

  @SuppressWarnings("unchecked")
  public User getUserFromToken(String jwt) {
    // não modificado ...
  }

}
```

Remova os imports desnecessários:

####### fj33-eats-monolito-modular/eats/eats-seguranca/src/main/java/br/com/caelum/eats/seguranca/JwtTokenManager.java

```java
i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶.̶u̶t̶i̶l̶.̶D̶a̶t̶e̶;̶
̶i̶m̶p̶o̶r̶t̶ ̶i̶o̶.̶j̶s̶o̶n̶w̶e̶b̶t̶o̶k̶e̶n̶.̶S̶i̶g̶n̶a̶t̶u̶r̶e̶A̶l̶g̶o̶r̶i̶t̶h̶m̶;̶
```

Remova as anotações do JPA e Beans Validator das classes `User` e `Role` do módulo de segurança do monólito. O cadastro de usuários será feito pelo API Gateway.

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

Limpe os imports da classe `User`:

####### fj33-eats-monolito-modular/eats/eats-seguranca/src/main/java/br/com/caelum/eats/seguranca/User.java

```java
i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶E̶n̶t̶i̶t̶y̶;̶
̶i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶F̶e̶t̶c̶h̶T̶y̶p̶e̶;̶
̶i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶G̶e̶n̶e̶r̶a̶t̶e̶d̶V̶a̶l̶u̶e̶;̶
̶i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶G̶e̶n̶e̶r̶a̶t̶i̶o̶n̶T̶y̶p̶e̶;̶
̶i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶I̶d̶;̶
̶i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶M̶a̶n̶y̶T̶o̶M̶a̶n̶y̶;̶
̶i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶v̶a̶l̶i̶d̶a̶t̶i̶o̶n̶.̶c̶o̶n̶s̶t̶r̶a̶i̶n̶t̶s̶.̶N̶o̶t̶B̶l̶a̶n̶k̶;̶

i̶m̶p̶o̶r̶t̶ ̶c̶o̶m̶.̶f̶a̶s̶t̶e̶r̶x̶m̶l̶.̶j̶a̶c̶k̶s̶o̶n̶.̶a̶n̶n̶o̶t̶a̶t̶i̶o̶n̶.̶J̶s̶o̶n̶I̶g̶n̶o̶r̶e̶;̶
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

Limpe também os imports da classe `Role`:

####### fj33-eats-monolito-modular/eats/eats-seguranca/src/main/java/br/com/caelum/eats/seguranca/Role.java

```java
i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶E̶n̶t̶i̶t̶y̶;̶
̶i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶I̶d̶;̶
```

Como a classe `User` não é mais uma entidade, devemos modificar seu relacionamento na classe `Restaurante` do módulo `eats-restaurante` do monólito:

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

Os imports devem ser ajustados:

####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/Restaurante.java

```java
i̶m̶p̶o̶r̶t̶ ̶j̶a̶v̶a̶x̶.̶p̶e̶r̶s̶i̶s̶t̶e̶n̶c̶e̶.̶O̶n̶e̶T̶o̶O̶n̶e̶;̶
̶i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶s̶e̶g̶u̶r̶a̶n̶c̶a̶.̶U̶s̶e̶r̶;̶
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

Limpe o import:

####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/RestauranteRepository.java

```java
i̶m̶p̶o̶r̶t̶ ̶b̶r̶.̶c̶o̶m̶.̶c̶a̶e̶l̶u̶m̶.̶e̶a̶t̶s̶.̶s̶e̶g̶u̶r̶a̶n̶c̶a̶.̶U̶s̶e̶r̶;̶
```

Faça com que a classe `RestauranteAuthorizationService` use o novo método do repository:

####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/RestauranteAuthorizationService.java

```java
@Service
@AllArgsConstructor
class RestauranteAuthorizationService  {

  private RestauranteRepository restauranteRepo;

  public boolean checaId(Authentication authentication, long id) {
    User user = (User) authentication.getPrincipal();
    if (user.isInRole(Role.ROLES.PARCEIRO)) {
      R̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶ ̶=̶ ̶r̶e̶s̶t̶a̶u̶r̶a̶n̶t̶e̶R̶e̶p̶o̶.̶f̶i̶n̶d̶B̶y̶U̶s̶e̶r̶(̶u̶s̶e̶r̶)̶;̶
      Restaurante restaurante = restauranteRepo.findByUserId(user.getId());
      if (restaurante != null) {
        return id == restaurante.getId();
      }
    }
    return false;
  }

  // código omitido...

}
```

Mude o `monolito.properties` do `config-repo`, removendo a configuração de expiração do token JWT. Essa configuração será usada apenas pelo gerador de tokens, o API Gateway. A chave privada, presente na propriedade `jwt.secret` ainda deve ser mantida, pois é usada na validação do token HS256.

####### config-repo/monolito.properties

```properties
j̶w̶t̶.̶e̶x̶p̶i̶r̶a̶t̶i̶o̶n̶ ̶=̶ ̶6̶0̶4̶8̶0̶0̶0̶0̶0̶
```

As URLS que não tem acesso protegido continuam funcionando sem a necessidade de um token. Por exemplo:

http://localhost:9999/restaurantes/1

ou

http://localhost:8080/restaurantes/1

Porém, URLs protegidas precisarão de um _access token_ válido e que foi emitido para um usuário que tenha permissão para fazer operações no recurso solicitado.

Se tentarmos acessar uma URL como a seguir, teremos o acesso negado:

http://localhost:8080/parceiros/restaurantes/1

A resposta será um erro HTTP 401 (Unauthorized).

Devemos usar um token obtido na autenticação como o API Gateway, colocando-o no cabeçalho HTTP `Authorization`, com `Bearer` como prefixo. O comando cURL em um Terminal, parece com:

```sh
curl -i -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJDYWVsdW0gRWF0cyIsInN1YiI6IjIiLCJyb2xlcyI6WyJQQVJDRUlSTyJdLCJ1c2VybmFtZSI6ImxvbmdmdSIsImlhdCI6MTU2OTU1MDYyOCwiZXhwIjoxNTcwMTU1NDI4fQ.S1V7aBNN206NpxPEEaIibtluJD9Bd-gHPK-MaUHcgxs' http://localhost:8080/parceiros/restaurantes/1
```

Deverá ser obtida uma resposta bem sucedida!

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

{"id":1,"cnpj":"98444252000104","nome":"Long Fu","descricao":"O melhor da China aqui do seu lado.","cep":"70238500","endereco":"ShC/SUL COMERCIO LOCAL QD 404-BL D LJ 17-ASA SUL","taxaDeEntregaEmReais":6.00,"tempoDeEntregaMinimoEmMinutos":40,"tempoDeEntregaMaximoEmMinutos":25,"aprovado":true,"tipoDeCozinhaId":1}
```

Isso indica que o módulo de segurança do monólito reconheceu o token como válido e extraiu a informação dos roles do usuário, reconhecendo-o no role PARCEIRO.

## Exercício: Autenticação no API Gateway e Autorização no monólito

1. Faça checkout da branch `cap15-autenticacao-no-api-gateway-e-autorizacao-nos-servicos` nos projeto do monólito modular, API Gateway e UI:

  ```sh
  cd ~/Desktop/fj33-eats-monolito-modular
  git checkout -f cap15-autenticacao-no-api-gateway-e-autorizacao-nos-servicos

  cd ~/Desktop/fj33-api-gateway
  git checkout -f cap15-autenticacao-no-api-gateway-e-autorizacao-nos-servicos

  cd ~/Desktop/fj33-eats-ui
  git checkout -f cap15-autenticacao-no-api-gateway-e-autorizacao-nos-servicos
  ```

  Faça refresh no Eclipse nos projetos do monólito modular e do API Gateway.

2. Poderíamos ter um BD específico para conter dados de usuários nas tabelas `user`, `role` e `user_authorities`. Porém, para simplificar, vamos manter os dados de usuários no BD do próprio monólito.

  No `config-repo`, adicione um arquivo `apigateway.properties` com o dados de conexão do BD do monólito, além das configurações da chave e expiração do JWT, que são usadas na geração do token:

  ####### config-repo/apigateway.properties

  ```properties
  #DATASOURCE CONFIGS
  spring.datasource.url=jdbc:mysql://localhost/eats?createDatabaseIfNotExist=true
  spring.datasource.username=root
  spring.datasource.password=

  #JWT CONFIGS
  jwt.secret = um-segredo-bem-secreto
  jwt.expiration = 604800000
  ```

3. Execute `ApiGatewayApplication`, certificando-se que o Service Registry e o Config Server estão no ar.

  Então, abra o terminal e simule a autenticação do dono do restaurante Long Fu:

  ```sh
  curl -i -X POST -H 'Content-type: application/json' -d '{"username":"longfu", "password":"123456"}' http://localhost:9999/auth
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

  {"userId":2,"username":"longfu","roles":["PARCEIRO"],"token":"eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJDYWVsdW0gRWF0cyIsInN1YiI6IjIiLCJyb2xlcyI6WyJQQVJDRUlSTyJdLCJ1c2VybmFtZSI6ImxvbmdmdSIsImlhdCI6MTU2OTU1MDYyOCwiZXhwIjoxNTcwMTU1NDI4fQ.S1V7aBNN206NpxPEEaIibtluJD9Bd-gHPK-MaUHcgxs"}
  ```

  São retornados, no corpo da resposta, informações sobre o usuário, seus roles e um token. Guarde esse token: o usaremos em breve!

4. Execute o `EatsApplication` do módulo `eats-application` do monólito.  Certifique-se que o Service Registry, Config Server e API Gateway estejam sendo executados.

  As URLS que não tem acesso protegido continuam funcionando. Por exemplo, acesse, pelo navegador, a URL a seguir para obter todas as formas de pagamento:

  http://localhost:9999/restaurantes/1

  ou

  http://localhost:8080/restaurantes/1

  Deve funcionar e retornar algo como:

  ```json
  {"id":1,"cnpj":"98444252000104","nome":"Long Fu","descricao":"O melhor da China aqui do seu lado.","cep":"70238500","endereco":"ShC/SUL COMERCIO LOCAL QD 404-BL D LJ 17-ASA SUL","taxaDeEntregaEmReais":6.00,"tempoDeEntregaMinimoEmMinutos":40,"tempoDeEntregaMaximoEmMinutos":25,"aprovado":true,"tipoDeCozinhaId":1}
  ```

  Porém, URLs protegidas precisarão de um _access token_ válido e que foi emitido para um usuário que tenha permissão para fazer operações no recurso solicitado.

  Tente acessar uma variação da URL anterior que só é acessível para usuários com o role PARCEIRO:

  http://localhost:8080/parceiros/restaurantes/1

  A resposta será um erro HTTP 401 (Unauthorized), com uma mensagem de acesso negado.

  Use o token obtido no exercício anterior, de autenticação no API Gateway, colocando-o no cabeçalho HTTP `Authorization`, depois do valor `Bearer`. Faça o seguinte comando cURL em um Terminal:

  ```sh
  curl -i -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJDYWVsdW0gRWF0cyIsInN1YiI6IjIiLCJyb2xlcyI6WyJQQVJDRUlSTyJdLCJ1c2VybmFtZSI6ImxvbmdmdSIsImlhdCI6MTU2OTU1MDYyOCwiZXhwIjoxNTcwMTU1NDI4fQ.S1V7aBNN206NpxPEEaIibtluJD9Bd-gHPK-MaUHcgxs' http://localhost:8080/parceiros/restaurantes/1
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

  {"id":1,"cnpj":"98444252000104","nome":"Long Fu","descricao":"O melhor da China aqui do seu lado.","cep":"70238500","endereco":"ShC/SUL COMERCIO LOCAL QD 404-BL D LJ 17-ASA SUL","taxaDeEntregaEmReais":6.00,"tempoDeEntregaMinimoEmMinutos":40,"tempoDeEntregaMaximoEmMinutos":25,"aprovado":true,"tipoDeCozinhaId":1}
  ```

  Isso indica que o módulo de segurança do monólito reconheceu o token como válido e extraiu a informação dos roles do usuário, reconhecendo-o no role PARCEIRO.

5. Altere o payload do JWT, definindo um valor diferente para o `sub`, o Subject, que indica o id do usuário assim como para o ROLE.

  Para isso, vá até um site como o http://www.base64url.com/ e defina no campo _Base 64 URL Encoding_ o payload do token JWT recebido do API Gateway.

  Altere o `sub` para `1`, simulando um usuário malicioso tentando forjar um token para roubar a identidade de outro usuário, de id diferente. Mude também o role para `ADMIN` e o nome do usuário para `admin`.

  O texto codificado em Base 64 URL Encoding será algo como:

  ```txt
  eyJpc3MiOiJDYWVsdW0gRWF0cyIsInN1YiI6IjEiLCJyb2xlcyI6WyJBRE1JTiJdLCJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTY2NTE4NzIyLCJleHAiOjE1NjcxMjM1MjJ9
  ```

  Através de um Terminal, use o cURL para tentar alterar uma forma de pagamento utilizando o payload modificado do JWT:

  ```sh
  curl -i -H 'Authorization: Bearer eyJhbGciOiJIUzI1NiJ9.eyJpc3MiOiJDYWVsdW0gRWF0cyIsInN1YiI6IjEiLCJyb2xlcyI6WyJBRE1JTiJdLCJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNTY2NTE4NzIyLCJleHAiOjE1NjcxMjM1MjJ9.S1V7aBNN206NpxPEEaIibtluJD9Bd-gHPK-MaUHcgxs' http://localhost:8080/parceiros/restaurantes/1
  ```

  Obtenha o comando anterior na seguinte URL: https://gitlab.com/snippets/1888416

  Como a assinatura do JWT não bate com o payload, o acesso deverá ser negado:

  ```txt
  HTTP/1.1 401 
  Date: Fri, 23 Aug 2019 12:47:07 GMT
  X-Content-Type-Options: nosniff
  X-XSS-Protection: 1; mode=block
  Cache-Control: no-cache, no-store, max-age=0, must-revalidate
  Pragma: no-cache
  Expires: 0
  X-Frame-Options: DENY
  Content-Type: application/json;charset=UTF-8
  Transfer-Encoding: chunked

  {"timestamp":"2019-08-23T12:47:07.785+0000","status":401,"error":"Unauthorized","message":"Você não está autorizado a acessar esse recurso.","path":"/parceiros/restaurantes/1"}
  ```

  Teste também com o cURL o acesso direto ao monólito, usando a porta `8080`. O acesso deve ser negado, da mesma maneira.

6. (desafio) Faça com que o novo serviço Administrativo também tenha autorização, fazendo com que apenas usuários no role ADMIN tenham acesso a URLS que começam com `/admin`.

## Deixando de reinventar a roda com OAuth 2.0

Da maneira como implementamos a autenticação anteriormente, acabamos definindo mais uma responsabilidade para o API Gateway: além de proxy e API Composer, passou a servir como autenticador e gerador de tokens. E, para isso, o API Gateway precisou conhecer tabelas dos usuários e seus respectivos roles. E mais: implementamos a geração e verificação de tokens manualmente.

Autenticação, autorização, tokens, usuário e roles são necessidades comuns e poderiam ser implementadas de maneira genérica. Melhor ainda se houvesse um padrão aberto, que permitisse implementação por diferentes fornecedores. Assim, os desenvolvedores poderiam focar mais em código de negócio e menos em código de segurança.

Há um framework de autorização baseado em tokens que permite que não nos preocupemos com detalhes de implementação de autenticação e autorização: o padrão **OAuth 2.0**. Foi definido na RFC 6749 da Internet Engineering Task Force (IETF), em Outubro de 2012.

Há extensões do OAuth 2.0 como o OpenID Connect (OIDC), que fornece uma camada de autenticação baseada em tokens JWT em cima do OAuth 2.0.

O foco original do OAuth 2.0, na verdade, é permitir que aplicações de terceiros usem informações de usuários em serviços como Google, Facebook e GitHub. Quando efetuamos login em uma aplicação com uma conta do Facebook ou quando permitimos que um serviço de Integração Contínua como o Travis CI acesse nosso repositório no GitHub, estamos usando OAuth 2.0.

Um padrão como o OAuth 2.0 nos permite instalar softwares como KeyCloak, WSO2 Identity Server, OpenAM ou Gluu e até usar soluções prontas de _identity as a service_ (IDaaS) como Auth0 ou Okta.

E, claro, podemos usar as soluções do Spring: **Spring Security OAuth**, que estende o Spring Security fornecendo implementações para OAuth 1 e OAuth 2.0. Há ainda o **Spring Cloud Security**, que traz soluções compatíveis com outros projetos do Spring Cloud.

## Roles

O OAuth 2.0 define quatro componentes, chamados de roles na especificação:

- **Resource Owner**: em geral, o usuário que tem algum recurso protegido como sua conta no Facebook, suas fotos no Flickr, seus repositórios no GitHub ou seu restaurante no Caelum Eats.
- **Resource Server**: provê o recurso protegido e permite o acesso mediante o uso de access tokens válidos.
- **Client**: a aplicação, Web, Single Page Application (SPA), Desktop ou Mobile, que deseja acessar os recursos do _Resource Owner_. Um _Client_ precisa estar registrado no _Authorization Server_, sendo identificado por um _client id_ e um _client secret_.
- **Authorization Server**: provê uma API para autenticar usuário e gerar access tokens. Pode estar na mesma aplicação do _Resource Server_.

O padrão OAuth 2.0 não especifica um formato para o access token. Se for usado um **access token opaco**, como uma String randômica ou UUID, a validação feita pelo Resource Server deve invocar o Authorization Server. Já no caso de um **self-contained access token** como um JWT/JWS, o próprio token contém informações para sua validação.

## Grant Types

O padrão OAuth 2.0 é bastante flexível e especifica diferentes maneiras de um _Client_ obter um access token, chamadas de _grant types_:

- **Password**: usada quando há uma forte relação de confiança entre o Client e o Authorization Server, como quando ambos são da mesma organização. O usuário informa suas credenciais (username e senha) diretamente para o Client, que repassa essas credenciais do usuário para o Authorization Server, junto com seu client id e client secret.
- **Client credentials**: usada quando não há um usuário envolvido, apenas um sistema chamando um recurso protegido de outro sistema. Apenas as credenciais do Client são informadas para o Authorization Server.
- **Authorization Code**: usada quando aplicações de terceiros desejam acessar informações de um recurso protegido sem que o Client conheça explicitamente as credenciais do usuário. Por exemplo, quando um usuário (Resource Owner) permite que o Travis CI (Client) acesse os seus repositórios do GitHub (Authorization Server e Resource Server). No momento em que o usuário cadastra seu GitHub no Travis CI, é redirecionado para uma tela de login do GitHub. Depois de efetuar o login no GitHub e escolher as permissões (ou _scopes_ nos termos do OAuth), é redirecionado para um servidor do Travis CI com um _authorization code_ como parâmetro da URL. Então, o Travis CI invoca o GitHub passando esse authorization code para obter um access token. As aplicações de terceiro que utilizam um authorization code são, em geral, aplicações Web clássicas com renderização das páginas no _serve-side_.
- **Implicit**: o usuário é direcionado a uma página de login do Authorization Server, mas o redirect é feito diretamente para o user-agent (o navegador, no caso da Web) já enviando o access token. Dessa forma, o Client SPA ou Mobile conhece diretamente o access token. Isso traz uma maior eficiência porém traz vulnerabilidades.

> A RFC 8252 (OAuth 2.0 for Native Apps), de Outubro de 2017, traz indicações de como fazer autenticação e autorização com OAuth 2.0 para aplicações mobile nativas.

No OAuth 2.0, um access token deve ter um tempo de expiração. Um token expirado levaria à necessidade de nova autenticação pelo usuário. Um Authorization Server pode emitir um _refresh token_, de expiração mais longa, que seria utilizado para obter um novo access token, sem a necessidade de nova autenticação. De acordo com a especificação, o grant type Implicit não deve permitir um refresh token, já que o token é conhecido e armazenado no próprio user-agent.

## OAuth no Caelum Eats

Podemos dizer que o API Gateway, que conhece os dados de usuário e seus roles, gera tokens e faz autenticação, é análogo a um Authorization Server do OAuth. O monólito, com a implementação de autorização para os módulos de Restaurante e Admin, serve como um Resource Server do OAuth. O front-end em Angular seria o Client do OAuth.

A autenticação no API Gateway é feita usando o nome do usuário e a respectiva senha que são informadas na própria aplicação do Angular. Ou seja, o Client conhece as credenciais do usuário e as repassa para o Authorization Server para autenticá-lo. Isso é análogo a um **Password grant type** do OAuth.

Poderíamos reimplementar a autenticação e autorização com OAuth usando código já pronto das bibliotecas Spring Security OAuth 2 e Spring Cloud Security, diminuindo o código que precisamos manter e cujas vulnerabilidades temos que sanar. Para isso, podemos definir um Authorization Server separado do API Gateway, responsável apenas pela autenticação e gerenciamento de tokens.

![Roles OAuth no Caelum Eats {w=73}](imagens/15-seguranca/roles-oauth-no-caelum-eats.png)

## Authorization Server com Spring Security OAuth 2

Para implementarmos um Authorization Server compatível com OAuth 2.0, devemos criar um novo projeto Spring Boot e adicionar como dependência o starter do Spring Cloud OAuth2:

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-oauth2</artifactId>
</dependency>
```

Com a dependência ao `spring-cloud-starter-oauth2` definida, devemos anotar a Application com `@EnableAuthorizationServer`.

No `application.properties`, devemos definir um client id e seu respectivo client secret:

```properties
security.oauth2.client.client-id=eats
security.oauth2.client.client-secret=eats123
```

> A configuração anterior define apenas um Client. Se tivermos registro de diferentes clients, podemos fornecer uma implementação da interface `ClientDetailsService`, que define o método `loadClientByClientId`. Nesse método, recebemos uma String com o client id e devemos retornar um objeto que implementa a interface `ClientDetails`.

Com essas configurações mínimas, teremos um Authorization Server que dá suporte a todos os grant types do OAuth 2.0 mencionados acima.

Se quisermos usar o Password grant type, devemos fornecer uma implementação da interface `UserDetailsService`, usada pelo Spring Security para obter os detalhes dos usuários. Essa implementação é exatamente igual ao que implementamos no API Gateway, nas classes `UserService`, `User` e `Role`, `UserRepository` e `SecurityConfig`. Para obter o registro dos usuários, o Authorization Server deve ter um data source que aponte para as tabelas de usuários e seus roles.

Ao executar o Authorization Server, podemos gerar um token enviando uma requisição POST ao endpoint `/oauth/token`. As credenciais do Client devem ser autenticadas com HTTP Basic. Devem ser definidos como parâmetros o grant type e o scope. Como não definimos nenhum scope, devemos usar `any`. No caso do Password grant type, devemos informar também as credenciais do usuário.

```sh
curl -i -X POST
  --basic -u eats:eats123
  -H 'Content-Type: application/x-www-form-urlencoded'
  -d 'grant_type=password&username=admin&password=123456&scope=any'
  http://localhost:8085/oauth/token
```

Como resposta, obteremos um access token e um refresh token, ambos opacos.

```txt
HTTP/1.1 200 
Pragma: no-cache
Cache-Control: no-store
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
X-Frame-Options: DENY
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Wed, 28 Aug 2019 13:54:22 GMT

{"access_token":"bdb22855-5705-4533-b925-f1091d576db7","token_type":"bearer","refresh_token":"0780c97f-f1d1-4a6f-82cb-c17ba5624caa","expires_in":43199,"scope":"any"}
```

Podemos checar um token opaco por meio de uma requisição GET ao endpoint `/oauth/check_token`, passando o access token obtido no parâmetro `token`:

```sh
curl -i localhost:8080/oauth/check_token/?token=bdb22855-5705-4533-b925-f1091d576db7
```

O corpo da resposta deve conter o username e os roles do usuário, entre outras informações:

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
Date: Wed, 28 Aug 2019 14:56:32 GMT

{"active":true,"exp":1567046599,"user_name":"admin","authorities":["ROLE_ADMIN"],"client_id":"eats","scope":["any"]}
```

### Erros comuns

Se as credenciais do Client estiverem incorretas

```sh
curl -i -X POST --basic -u eats:SENHA_ERRADA -H 'Content-Type: application/x-www-form-urlencoded' -k -d 'grant_type=password&username=admin&password=123456&scope=any' http://localhost:8085/oauth/token
```

receberemos um status 401 (Unauthorized):

```txt
HTTP/1.1 401
...
{"timestamp":"2019-08-28T14:39:58.413+0000","status":401,"error":"Unauthorized","message":"Unauthorized","path":"/oauth/token"}
```

Se as credenciais do usuário estiverem incorretas, no caso de um Password grant type

```sh
curl -i -X POST --basic -u eats:eats123 -H 'Content-Type: application/x-www-form-urlencoded' -k -d 'grant_type=password&username=admin&password=SENHA_ERRADA&scope=any' http://localhost:8085/oauth/token
```

receberemos um status 400 (Bad Request), com _Bad credentials_ como mensagem de erro

```txt
HTTP/1.1 400
...
{"error":"invalid_grant","error_description":"Bad credentials"}
```

Se omitirmos o scope

```sh
curl -i -X POST --basic -u eats:eats123 -H 'Content-Type: application/x-www-form-urlencoded' -d 'grant_type=password&username=admin&password=123456' http://localhost:8085/oauth/token
```

receberemos um status 400 (Bad Request), com _Empty scope_ como mensagem de erro

```txt
HTTP/1.1 400
...
{"error":"invalid_scope","error_description":"Empty scope (either the client or the user is not allowed the requested scopes)"}
```

Se omitirmos o grant type

```sh
curl -i -X POST --basic -u eats:eats123 -H 'Content-Type: application/x-www-form-urlencoded' -d 'username=admin&password=123456&scope=any' http://localhost:8085/oauth/token
```

receberemos um status 400 (Bad Request), com _Missing grant type_ como mensagem de erro

```txt
HTTP/1.1 400
...
{"error":"invalid_request","error_description":"Missing grant type"}
```

Se informarmos um grant type incorreto

```sh
curl -i -X POST --basic -u eats:eats123 -H 'Content-Type: application/x-www-form-urlencoded' -d 'grant_type=NAO_EXISTE&username=admin&password=123456&scope=any' http://localhost:8085/oauth/token
```

receberemos um status 400 (Bad Request), com _Unsupported grant type_ como mensagem de erro

```txt
HTTP/1.1 400
...
{"error":"unsupported_grant_type","error_description":"Unsupported grant type: NAO_EXISTE"}
```

Se, ao checarmos um token, passarmos um token expirado ou inválido

```sh
curl -i localhost:8085/oauth/check_token/?token=TOKEN_INVALIDO
```

receberemos um status 400 (Bad Request), com _Token was not recognised_ como mensagem de erro

```txt
HTTP/1.1 400
...
{"error":"invalid_token","error_description":"Token was not recognised"}
```

<!--

TODO: parei aqui

LEMBRETES no caderno

-->

## JWT como formato de token no Spring Security OAuth 2

A dependência `spring-cloud-starter-oauth2` já tem como dependência transitiva a biblioteca `spring-security-jwt`, que provê suporte a JWT no Spring Security.

Precisamos fazer algumas configurações para que o token gerado seja um JWT. Para isso, devemos definir uma implementação para a interface `AuthorizationServerConfigurer`. Podemos usar a classe `AuthorizationServerConfigurerAdapter` como auxílio.

As configurações são as seguintes:

- um objeto da classe `JwtTokenStore`, que implementa a interface `TokenStore`
- um objeto da classe `JwtAccessTokenConverter`, que implementa a interface `AccessTokenConverter`. A classe `JwtAccessTokenConverter` gera, por padrão, um chave privada de assinatura (`signingKey`) randômica. É interessante definir uma propriedade `jwt.secret`, como havíamos feito anteriormente.
- uma implementação de `ClientDetailsService` para que as propriedades `security.oauth2.client.client-id` e `security.oauth2.client.client-secret` funcionem e definam o id e a senha do Client com sucesso. Podemos usar a classe `ClientDetailsServiceConfigurer`. Os valores das propriedades de Client id e secret podem ser obtidas usando `OAuth2ClientProperties`.
- devemos definir o `AuthenticationManager` configurado na classe `SecurityConfig` por meio da classe `AuthorizationServerEndpointsConfigurer`

Fazemos todas essas configurações na classe `OAuthServerConfig` a seguir:

```java
@Configuration
public class OAuthServerConfig extends AuthorizationServerConfigurerAdapter {

  private final AuthenticationManager authenticationManager;
  private final OAuth2ClientProperties clientProperties;
  private final String jwtSecret;

  public OAuthServerConfiguration(AuthenticationManager authenticationManager,
                OAuth2ClientProperties clientProperties, 
                @Value("${jwt.secret}") String jwtSecret) {
    this.authenticationManager = authenticationManager;
    this.clientProperties = clientProperties;
    this.jwtSecret = jwtSecret;
  }

  @Override
  public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
    clients.inMemory()
      .withClient(clientProperties.getClientId())
      .secret(clientProperties.getClientSecret());
  }

  @Override
  public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
    endpoints.tokenStore(tokenStore())
        .accessTokenConverter(accessTokenConverter())
        .authenticationManager(authenticationManager);
  }

  @Bean
  public TokenStore tokenStore() {
    return new JwtTokenStore(accessTokenConverter());
  }

  @Bean
  public JwtAccessTokenConverter accessTokenConverter() {
    JwtAccessTokenConverter converter = new JwtAccessTokenConverter();
    converter.setSigningKey(this.jwtSecret);
    return converter;
  }

}
```

A configuração padrão habilitada pela anotação `@EnableAuthorizationServer` usa um `NoOpsPasswordEncoder`, que faz com que as senhas sejam lidas em texto puro. Porém, como definimos o `BCryptPasswordEncoder` no nosso `SecurityConfig`, precisaremos modificar a propriedade `security.oauth2.client.client-secret` no arquivo `application.properties`:

```properties
security.oauth2.client.client-secret=$2a$10$1YJxJHAbtsSCeyqgN7S1gurPZ8NSmTVA33dgPq6NqElU6qjzlpkOa
```

Ao executar novamente o Authorization Server, os tokens serão gerados no formato JWT/JWS.

Podemos testar novamente com o cURL:

```sh
curl -i -X POST --basic -u eats:eats123 -H 'Content-Type: application/x-www-form-urlencoded' -d 'grant_type=password&username=admin&password=123456&scope=any' http://localhost:8085/oauth/token
```

Teremos uma resposta bem sucedida, com um access token no formato JWT:

```txt
HTTP/1.1 200
Pragma: no-cache
Cache-Control: no-store
X-Content-Type-Options: nosniff
X-XSS-Protection: 1; mode=block
X-Frame-Options: DENY
Content-Type: application/json;charset=UTF-8
Transfer-Encoding: chunked
Date: Wed, 28 Aug 2019 18:11:25 GMT

{"access_token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NjcwNTkwODUsInVzZXJfbmFtZSI6ImFkbWluIiwiYXV0aG9yaXRpZXMiOlsiUk9MRV9BRE1JTiJdLCJqdGkiOiI2ODlkMGE0ZS0xZjRmLTQ5OGMtOGMzMS05YjVlYjMyZWYxYjgiLCJjbGllbnRfaWQiOiJlYXRzIiwic2NvcGUiOlsiYW55Il19.ZtYpX3GJPYU8UNhHRtmEtQ7SLiiZdZOrdCRJt64ovF4","token_type":"bearer","expires_in":43199,"scope":"any","jti":"689d0a4e-1f4f-498c-8c31-9b5eb32ef1b8"}
```

O access token anterior contém, como todo JWS, 3 partes.

O cabeçalho:

```txt
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9
```

Que pode ser decodificado, usando um Base 64 URL Decoder, para:

```json
{"alg":"HS256","typ":"JWT"}
```

Já a segunda parte é o payload, que contém os claims do JWT:

```txt
eyJleHAiOjE1NjcwNTkwODUsInVzZXJfbmFtZSI6ImFkbWluIiwiYXV0aG9yaXRpZXMiOlsiUk9MRV9BRE1JTiJdLCJqdGkiOiI2ODlkMGE0ZS0xZjRmLTQ5OGMtOGMzMS05YjVlYjMyZWYxYjgiLCJjbGllbnRfaWQiOiJlYXRzIiwic2NvcGUiOlsiYW55Il19
```

Após a decodificação Base64, teremos:

```json
{
  "exp":1567059085,
  "user_name":"admin",
  "authorities":["ROLE_ADMIN"],
  "jti":"689d0a4e-1f4f-498c-8c31-9b5eb32ef1b8",
  "client_id":"eats",
  "scope":["any"]}
```

Perceba que temos o `user_name` e os respectivos roles em `authorities`.

Há também uma propriedade `jti` (JWT ID), uma String randômica (UUID) que serve como um _nonce_: um valor é diferente a cada request e previne o sistema contra _replay attacks_.

A terceira parte é a assinatura:

```txt
ZtYpX3GJPYU8UNhHRtmEtQ7SLiiZdZOrdCRJt64ovF4
```

Como usamos o algoritmo `HS256`, um algoritmo de chaves simétricas, a chave privada setada em `signingKey` precisa ser conhecida para validar a assinatura.

## Exercício: um Authorization Server com Spring Security OAuth 2

1. Abra um Terminal e baixe o projeto `fj33-authorization-server` para o seu Desktop usando o Git:

  ```sh
  cd ~/Desktop
  git clone https://gitlab.com/aovs/projetos-cursos/fj33-authorization-server.git
  ```

2. No workspace de microservices do Eclipse, acesse _File > Import > Existing Maven Projects_ e clique em _Next_. Em _Root Directory_, aponte para o diretório clonado anteriormente.

  Veja o código das classes `AuthorizationServerApplication` e `OAuthServerConfig`, além dos arquivos `bootstrap.properties` e `application.properties`.

  Note que o `spring.application.name` é `authorizationserver`. A porta definida para o Authorization Server é `8085`.

3. Crie o arquivo `authorizationserver.properties` no `config-repo`, com o seguinte conteúdo:

  ```properties
  #DATASOURCE CONFIGS
  spring.datasource.url=jdbc:mysql://localhost/eats?createDatabaseIfNotExist=true
  spring.datasource.username=root
  spring.datasource.password=

  jwt.secret = rm'!@N=Ke!~p8VTA2ZRK~nMDQX5Uvm!m'D&]{@Vr?G;2?XhbC:Qa#9#eMLN\}x3?JR3.2zr~v)gYF^8\:8>:XfB:Ww75N/emt9Yj[bQMNCWwW\J?N,nvH.<2\.r~w]*e~vgak)X"v8H`MH/7"2E`,^k@n<vE-wD3g9JWPy;CrY*.Kd2_D])=><D?YhBaSua5hW%{2]_FVXzb9`8FH^b[X3jzVER&:jw2<=c38=>L/zBq`}C6tT*cCSVC^c]-L}&/

  security.oauth2.client.client-id=eats
  security.oauth2.client.client-secret=$2a$10$1YJxJHAbtsSCeyqgN7S1gurPZ8NSmTVA33dgPq6NqElU6qjzlpkOa
  ```

  O código anterior pode ser encontrado em: https://gitlab.com/snippets/1890756

  Note que copiamos o `jwt.secret` e os dados do BD do monólito. Isso indica que o BD será mantido de maneira monolítica. Eventualmente, seria possível fazer a migração de dados de usuário para um BD específico.

  Além disso, definimos as propriedades de Client id e secret do Spring Security OAuth 2.

  Não deixe de comitar o novo arquivo no repositório Git.

4. Execute a classe `AuthorizationServerApplication`.

  Então, abra um terminal e execute o seguinte comando:

  ```sh
  curl -i -X POST --basic -u eats:eats123 -H 'Content-Type: application/x-www-form-urlencoded' -d 'grant_type=password&username=admin&password=123456&scope=any' http://localhost:8085/oauth/token
  ```

  Se não quiser digitar, é possível encontrar o comando anterior no seguinte link: https://gitlab.com/snippets/1890014

  Como resposta, deverá ser exibido algo como:

  ```txt
  HTTP/1.1 200

  ...
  {
  "access_token":
    "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.
    eyJleHAiOjE1NjcwNTkwODUsInVzZXJfbmFtZSI6ImFkbWluIiwiYXV0aG9yaXRpZXMiOlsiUk9MRV9BRE1JTiJdLCJqdGkiOiI2ODlkMGE0ZS0xZjRmLTQ5OGMtOGMzMS05YjVlYjMyZWYxYjgiLCJjbGllbnRfaWQiOiJlYXRzIiwic2NvcGUiOlsiYW55Il19.
    ZtYpX3GJPYU8UNhHRtmEtQ7SLiiZdZOrdCRJt64ovF4",
  "token_type": "bearer",
  "expires_in": 43199,
  "scope": "any",
  "jti": "689d0a4e-1f4f-498c-8c31-9b5eb32ef1b8"
  }
  ```

  Pegue o conteúdo da propriedade `access_token` e analise o cabeçalho e o payload em: https://jwt.io

  O payload deverá conter algo semelhante a:

  ```json
  {
    "exp": 1567059085,
    "user_name": "admin",
    "authorities": [
      "ROLE_ADMIN"
    ],
    "jti": "689d0a4e-1f4f-498c-8c31-9b5eb32ef1b8",
    "client_id": "eats",
    "scope": [
      "any"
    ]
  }
  ```

5. Remova o código de autenticação do API Gateway.

  Para isso, delete as seguintes classes  do API Gateway:

  - A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶C̶o̶n̶t̶r̶o̶l̶l̶e̶r̶
  - A̶u̶t̶h̶e̶n̶t̶i̶c̶a̶t̶i̶o̶n̶D̶t̶o̶
  - J̶w̶t̶T̶o̶k̶e̶n̶M̶a̶n̶a̶g̶e̶r̶
  - P̶a̶s̶s̶w̶o̶r̶d̶E̶n̶c̶o̶d̶e̶r̶C̶o̶n̶f̶i̶g̶
  - R̶o̶l̶e̶
  - S̶e̶c̶u̶r̶i̶t̶y̶C̶o̶n̶f̶i̶g̶
  - U̶s̶e̶r̶
  - U̶s̶e̶r̶I̶n̶f̶o̶D̶t̶o̶
  - U̶s̶e̶r̶R̶e̶p̶o̶s̶i̶t̶o̶r̶y̶
  - U̶s̶e̶r̶S̶e̶r̶v̶i̶c̶e̶

  Remova as seguintes dependências do `pom.xml` do API Gateway:

  - j̶j̶w̶t̶
  - m̶y̶s̶q̶l̶-̶c̶o̶n̶n̶e̶c̶t̶o̶r̶-̶j̶a̶v̶a̶
  - s̶p̶r̶i̶n̶g̶-̶b̶o̶o̶t̶-̶s̶t̶a̶r̶t̶e̶r̶-̶d̶a̶t̶a̶-̶j̶p̶a̶
  - s̶p̶r̶i̶n̶g̶-̶b̶o̶o̶t̶-̶s̶t̶a̶r̶t̶e̶r̶-̶s̶e̶c̶u̶r̶i̶t̶y̶

  Apague a seguinte rota do `application.properties` do API Gateway:

  ```properties
  z̶u̶u̶l̶.̶r̶o̶u̶t̶e̶s̶.̶a̶u̶t̶h̶.̶p̶a̶t̶h̶=̶/̶a̶u̶t̶h̶/̶*̶*̶
  z̶u̶u̶l̶.̶r̶o̶u̶t̶e̶s̶.̶a̶u̶t̶h̶.̶u̶r̶l̶=̶f̶o̶r̶w̶a̶r̶d̶:̶/̶a̶u̶t̶h̶
  ```

  Delete o arquivo `apigateway.properties` do `config-repo`.

6. (desafio - trabalhoso) Aplique uma estratégia de migração de dados de usuário do monólito para um BD específico para o Authorization Server.

## Resource Server com Spring Security OAuth 2

Para definir um Resource Server com o Spring Security OAuth 2, que consiga validar e decodificar os tokens (opacos ou JWT) emitidos pelo Authorization Server, basta anotar a aplicação ou uma configuração com `@EnableResourceServer`.

Podemos definir, na configuração `security.oauth2.resource.token-info-uri`, a URI de validação de tokens opacos.

No caso de token self-contained JWT, devemos definir a propriedade `security.oauth2.resource.jwt.key-value`. Pode ser a chave simétrica, no caso de algoritmos como o HS256, ou a chave pública, como no RS256. A chave pública em um algoritmo assimétrico pode ser baixada do servidor quando definida a propriedade `security.oauth2.resource.jwt.key-uri`.

Por padrão, todos os endereços requerem autenticação. Porém, é possível customizar esse e outros detalhes fornecendo uma implementação da interface `ResourceServerConfigurer`. É possível herdar da classe `ResourceServerConfigurerAdapter` para facilitar as configurações.

## Protegendo o serviço Administrativo

Adicione os starters do Spring Security OAuth 2 e Spring Cloud Security ao `pom.xml` do `eats-administrativo-service`:

####### fj33-eats-administrativo-service/pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-oauth2</artifactId>
</dependency>
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-security</artifactId>
</dependency>
```

Anote a  classe `EatsAdministrativoServiceApplication` com `@EnableResourceServer`:

####### fj33-eats-administrativo-service/src/main/java/br/com/caelum/eats/administrativo/EatsAdministrativoServiceApplication.java

```java
@EnableResourceServer // adicionado
@EnableDiscoveryClient
@SpringBootApplication
public class EatsAdministrativoServiceApplication {

  public static void main(String[] args) {
    SpringApplication.run(EatsAdministrativoServiceApplication.class, args);
  }

}
```

O import correto é o seguinte:

####### fj33-eats-administrativo-service/src/main/java/br/com/caelum/eats/administrativo/EatsAdministrativoServiceApplication.java

```java
import org.springframework.security.oauth2.config.annotation.web.configuration.EnableResourceServer;
```

Adicione ao `administrativo.properties` do `config-repo`, a mesma chave usada no Authorization Server, porém na propriedade `security.oauth2.resource.jwt.key-value`:

####### config-repo/administrativo.properties

```properties
security.oauth2.resource.jwt.key-value = um-segredo-bem-secreto
```

Crie uma classe `OAuthResourceServerConfig`. Herde da classe `ResourceServerConfigurerAdapter` e permita que todos acessem a listagem de tipos de cozinha e formas de pagamento, assim como os endpoints do Spring Boot Actuator. As URLs que começam com `/admin` devem ser restritas a usuário que tem o role `ADMIN`.

####### fj33-eats-administrativo-service/src/main/java/br/com/caelum/eats/administrativo/OAuthResourceServerConfig.java

```java
@Configuration
class OAuthResourceServerConfig extends ResourceServerConfigurerAdapter {

  @Override
  public void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
    .antMatchers("/tipos-de-cozinha/**", "/formas-de-pagamento/**").permitAll()
    .antMatchers("/actuator/**").permitAll()
    .antMatchers("/admin/**").hasRole("ADMIN")
    .anyRequest().authenticated()
    .and().cors()
    .and().csrf().disable()
    .formLogin().disable()
    .httpBasic().disable();
  }

}
```

Certifique-se que os imports estão corretos:

####### fj33-eats-administrativo-service/src/main/java/br/com/caelum/eats/administrativo/OAuthResourceServerConfig.java

```java
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.oauth2.config.annotation.web.configuration.ResourceServerConfigurerAdapter;
```

URLs abertas, como a que lista todas as formas de pagamento, terão sucesso sem nenhum token ser passado:

http://localhost:8084/formas-de-pagamento

Já URLs como a que permite a alteração dos dados de uma forma de pagamento estarão protegidas:

```sh
curl -i -X PUT -H 'Content-type: application/json' -d '{"id": 3, "tipo": "CARTAO_CREDITO", "nome": "American Express"}' http://localhost:9999/admin/formas-de-pagamento/3
```

Como resposta, teríamos um erro `401 (Unauthorized)`:

```txt
HTTP/1.1 401
{"error":"unauthorized","error_description":"Full authentication is required to access this resource"}
```

Será necessário passar um token obtido do Authorization Server que contém o role `ADMIN` para que a chamada anterior seja bem sucedida.

## Exercício: Protegendo o serviço Administrativo com Spring Security OAuth 2

1. Faça checkout da branch `cap15-resource-server-com-spring-security-oauth-2` do serviço Administrativo:

  ```sh
  cd ~/Desktop/fj33-eats-administrativo-service
  git checkout -f cap15-resource-server-com-spring-security-oauth-2
  ```

  Faça refresh do projeto no Eclipse e o reinicie.

2. Abra um terminal e tente listas todas as formas de pagamento sem passar nenhum token:

  ```sh
  curl http://localhost:8084/formas-de-pagamento
  ```

  A resposta deve ser bem sucedida, contendo algo como:

  ```txt
  [{"id":4,"tipo":"VALE_REFEICAO","nome":"Alelo"},{"id":3,"tipo":"CARTAO_CREDITO","nome":"Amex Express"},{"id":2,"tipo":"CARTAO_CREDITO","nome":"MasterCard"},{"id":6,"tipo":"CARTAO_DEBITO","nome":"MasterCard Maestro"},{"id":5,"tipo":"VALE_REFEICAO","nome":"Ticket Restaurante"},{"id":1,"tipo":"CARTAO_CREDITO","nome":"Visa"},{"id":7,"tipo":"CARTAO_DEBITO","nome":"Visa Débito"}]
  ```

  Vamos tentar editar uma forma de pagamento, chamando um endpoint que começa com `/admin`, sem um token:

  ```sh
  curl -i -X PUT -H 'Content-type: application/json' -d '{"id": 3, "tipo": "CARTAO_CREDITO", "nome": "American Express"}' http://localhost:9999/admin/formas-de-pagamento/3
  ```

  O comando anterior pode ser encontrado em: https://gitlab.com/snippets/1888251

  Deve ser retornado um erro `401 (Unauthorized)`, com a descrição _Full authentication is required to access this resource_, indicando que o acesso ao recurso depende de autenticação:

  ```txt
  HTTP/1.1 401 
  Pragma: no-cache
  WWW-Authenticate: Bearer realm="oauth2-resource", error="unauthorized", error_description="Full authentication is required to access this resource"
  Cache-Control: no-store
  X-Content-Type-Options: nosniff
  X-XSS-Protection: 1; mode=block
  X-Frame-Options: DENY
  Content-Type: application/json;charset=UTF-8
  Transfer-Encoding: chunked
  Date: Thu, 29 Aug 2019 20:12:57 GMT

  {"error":"unauthorized","error_description":"Full authentication is required to access this resource"}
  ```

  Devemos incluir, no cabeçalho `Authorization`, o token JWT obtido anteriormente:

  ```sh
  curl -i -X PUT -H 'Content-type: application/json' -H 'Authorization: Bearer TOKEN-JWT-AQUI' -d '{"id": 3, "tipo": "CARTAO_CREDITO", "nome": "Amex Express"}' http://localhost:8084/admin/formas-de-pagamento/3
  ```

  O comando acima pode ser encontrado em: https://gitlab.com/snippets/1890417

  Observação: troque `TOKEN-JWT-AQUI` pelo token obtido do Authorization Server em exercícios anteriores.

  A resposta será um sucesso!

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
  Date: Thu, 29 Aug 2019 20:13:02 GMT

  {"id":3,"tipo":"CARTAO_CREDITO","nome":"Amex Express"}
  ```

## Protegendo serviços de infraestrutura

Temos serviços de infraestrutura como:

- API Gateway
- Service Registry
- Config Server
- Hystrix Dashboard
- Turbine
- Admin Server

O API Gateway é _edge service_ do Caelum Eats, que fica na fronteira da infra-estrutura. Serve como um proxy que repassa requisições e as respectivas respostas. Podemos fazer um _rate limiting_, cortando requisições de um mesmo cliente a partir de uma certa taxa de requisições por segundo, afim de evitar um ataque de DDoS (Distributed Denial of Service), que visa deixar um sistema fora do ar. O API Gateway também serve como um API Composer, que dispara requisições a vários serviços, agregando as respostas. Nesse caso, é preciso avaliar se a composição requer algum tipo de autorização. No caso implementado, a composição que agrega dados de um restaurante com sua distância a um determinado CEP, é feita por meio de dados públicas. Portanto, não há a necessidade de autorização. Nesse cenário de composição, a avaliação da necessidade de autorização, deve ser feita caso a caso. Uma ideia simples é repassar erros de autorização dos serviços invocados.

Uma vulnerabilidade da nossa aplicação é que uma vez que o endereço do Service Registry é conhecido, é possível descobrir nomes, hosts e portas de todos os serviços. A partir dos nomes dos serviços, podemos consultar o Config Server e observar detalhes de configuração de cada serviço.

Podemos, de maneira bem fácil, proteger o Config Server, o Service Registry e demais serviços de infraestrutura que criamos.

Basta adicionarmos, às dependências do Maven, o Spring Security:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

Para que o Spring Security não use uma senha randômica, devemos definir usuário e senha como propriedades no `application.properties`. Por exemplo, para o Config Server:

```properties
security.user.name=configUser
security.user.password=configPassword
```

Nos demais serviços, devemos adicionar ao `bootstrap.properties`:

```properties
spring.cloud.config.uri=http://localhost:8888
spring.cloud.config.username=configUser
spring.cloud.config.password=configPassword
```

No caso do Service Registry, faríamos o mesmo processo.

Definiríamos o endereço nos clientes do Eureka da seguinte maneira:

```properties
eureka.client.serviceUrl.defaultZone=http://eurekaUser:eurekaPassword@localhost:8761/eureka/
```

## Confidencialidade, Integridade e Autenticidade com HTTPS

O protocolo HTTP é baseado em texto e, sem uma estratégia de confidencialidade, as informações serão trafegadas como texto puro da UI para o seu sistema e nas chamadas entre os serviços. Dados como usuário, senha, cartões de crédito estariam totalmente expostos.

HTTPS é uma extensão ao HTTP que usa TLS (Transport Layer Security) para prover confidencialidade aos dados por meio de criptografia. O protocolo SSL (Security Sockets Layer) é o predecessor do TLS e está _deprecated_.

Além disso, o HTTPS provê a integridade dos dados, evitando que sejam manipulados no meio do caminho, bem como a autenticidade do servidor, garantindo que o servidor é exatamente o que o cliente espera.

A confidencialidade, integridade e autenticidade do servidor no HTTPS é atingida por meio de criptografia assimétrica (public-key cryptography). O servidor tem um par de chaves (key pair): uma pública e uma privada. Algo criptografado com a chave pública só pode ser descriptografado com a chave privada, garantindo confidencialidade. Algo criptografado com a chave privada pode ser verificado com a chave pública, validando a autenticidade.

A chave pública faz parte de um certificado digital, que é emitido por uma Autoridade Certificadora (Certificate Authority) como Comodo, Symantec, Verizon ou Let's Encrypt. Toda a infraestrutura dos certificados digitais é baseada na confiança de ambas as partes, cliente e servidor, nessas Autoridades Certificadoras.

Mas o HTTPS não é um mar de rosas: os certificados tem validade e precisam ser gerenciados. A automação do gerenciamento de certificados ainda deixa a desejar, mas tem melhorado progressivamente. Let's Encrypt sendo uma referência nessa automação.

Certificados gerados sem uma autoridade certificadora (self-signed certificates) não são confiáveis e apresentam erros em navegadores e outros sistemas.

Com o comando `keytool`, que vem com a JDK, podemos gerar um self-signed certificate:

```sh
keytool -genkey -alias eats -storetype JKS -keyalg RSA -keysize 2048 -keystore eats-keystore.jks -validity 3650
```

Será solicitada uma senha e uma série de outras informações e gerado o arquivo `eats-keystore.jks`.

Podemos configurar o `application.properties` de uma aplicação Spring Boot da seguinte maneira:

```properties
server.port=8443
server.ssl.key-store=eats-keystore.jks
server.ssl.key-store-password=a-senha-escolhida
server.ssl.keyAlias=eats
```

## Mutual Authentication

Um outro detalhe do HTTPS é que não há garantias da autenticidade do cliente, apenas do servidor.

Para garantir a autenticidade do cliente _e_ do servidor, podemos fazer com que ambos tenham certificados digitais. Quando o cliente é um navegador, isso não é possível porque é inviável exigir a cada um dos usuários a instalação de um certificado. Por isso, o uso mútuo de certificados é comumente usado na comunicação servidor-servidor.

Cada serviço deve ter dois _stores_ com chaves criptográficas, que possuem a extensão `.jks` na plataforma Java:

- uma _key store_, que contém a chave privada de um determinado serviço, além de um certificado com a respectiva chave pública
- uma _trust store_, que contém os certificados com chaves públicas dos clientes e servidores ou de Autoridades Certificadoras considerados confiáveis

O `application.properties` deve ter configurações tanto do key store como do trust store, além da propriedade `server.ssl.client-auth` que indica o uso de autenticação mútua e pode ter os valores `none`, `want` (não obrigatório) e `need` (obrigatório).

```properties
server.ssl.key-store=eats-keystore.jks
server.ssl.key-store-password=a-senha-escolhida
server.ssl.keyAlias=eats

server.ssl.trust-store=eats-truststore.jks
server.ssl.trust-store-password=senha-do-trust-store

server.ssl.client-auth=need
```

## Protegendo dados armazenados

Mesmo investindo esforço em proteger a rede, a comunicação entre os serviços (_data at transit_) e os serviços em si, é preciso preparar nosso ambiente para uma possível invasão.

Uma vulnerabilidade está nos dados armazenados (_data at rest_) em BDs, arquivos de configuração e backups. Em especial, devemos proteger dados sensíveis como cartões de crédito, senhas e chaves criptográficas. Muitos ataques importantes exploraram a falta de criptografia de dados armazenados ou falhas nos algoritmos criptográficos utilizados.

Em seu livro _Building Microservices_, Sam Newman indica algumas medidas que devem ser tomadas para proteger os dados armazenados:

- use implementações padrão de algoritmos criptográficos conhecidos, ficando atento a possíveis vulnerabilidades e aplicando _patches_ regularmente. Não tente criar o seu algoritmo. Para senhas, use Strings randômicas (salts) que minimizam ataques baseados em tabelas de hashes.
- limite a encriptação a tabelas dos BDs e a arquivos que realmente são sensíveis para evitar impactos negativos na performance da aplicação
- criptografe os dados sensíveis logo que entrarem no sistema, descriptografe sob demanda e assegure que os dados não são armazenados em outros lugares
- assegure que os backups estejam criptografados
- armazene as chaves criptográficas em um software ou appliance (hardware) específico para gerenciamento de chaves.

## Rotação de credenciais

Em Junho de 2014, a Code Spaces, uma concorrente do GitHub que fornecia Git e SVN na nuvem, sofreu um ataque em que o invasor, após chantagem, apagou quase todos os dados, configurações de máquinas e backups da empresa. O ataque levou a empresa à falência! Isso aconteceu porque o invasor teve acesso ao painel de controle do AWS e conseguiu apagar quase todos os artefatos, incluindo os backups.

Não se sabe ao certo como o invasor conseguiu o acesso indevido ao painel de controle do AWS, mas há a hipótese de que obteve as credenciais de acesso de um antigo funcionário da empresa.

É imprescindível que as credenciais tenham acesso limitado, minimizando o potencial de destruição de um possível invasor.

Outra coisa importante é que as senhas dos usuários, chaves criptográficas, API keys e outras credenciais sejam modificadas de tempos em tempos. Assim, ataques feitos com a ajuda funcionários desonestos terão efeito limitado. Se possível, essa **rotação de credenciais** deve ser feita de maneira automatizada.

Há alguns softwares que automatizam o gerenciamento de credenciais:

- Vault, da HashiCorp
- AWS Secrets Manager
- KeyWiz, da Square
- CredHyb, da Cloud Foundry

> Um outro aspecto do caso da Code Spaces é que os backups eram feitos no próprio AWS. É importante que tenhamos offsite backups, em caso de comprometimento de um provedor de cloud computing.

### Vault

Vault é uma solução de gerenciamento de credenciais da HashiCorp, a mesma empresa que mantém o Vagrant, Consul, Terraform, entre outros.

O Vault armazena de maneira segura e controla o acesso de tokens, senhas, API Keys, chaves criptográficas, e certificados digitais. Provê uma CLI, uma API HTTP e uma UI Web para gerenciamento. É possível criar, revogar e rotacionar credenciais de maneira automatizada.

Para que a senha, por exemplo, de um BD seja alterada pelo Vault, é necessário que seja configurado um usuário do BD que possa criar e remover outros usuários.

Segue um exemplo dos comandos da CLI do Vault para criação de credenciais com duração de 1 hora no MySQL:

```sh
vault secrets enable mysql
vault write mysql/config/connection connection_url="root:root-password@tcp(192.168.33.10:3306)/"
vault write mysql/config/lease lease=1h lease_max=24h
vault write mysql/roles/readonly sql="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';"
```

As credenciais dos backends precisam ser conhecidas pelo Vault. No caso do MySQL, o usuário `root` e a respectiva senha precisam ser conhecidos. Essas configurações são armazenadas de maneira criptografada na representação interna do Vault. O Vault pode usar para armazenamento Consul, Etcd, o sistema de arquivos, entre diversos outros mecanismos.

Os dados do Vault são criptografados com uma chave simétrica. Essa chave simétrica é criptografada com uma _master key_. E a _master key_ é criptografada usando o algoritmo _Shamir's secret sharing_, em que mais de uma chave é necessária para descriptografar os dados. Por padrão, o Vault usa 5 chaves ao todo, sendo 3 delas necessárias para a descriptografia.

O Spring Cloud Config Server permite o uso do Vault como repositório de configurações: https://cloud.spring.io/spring-cloud-config/reference/html/#vault-backend

Há ainda o Spring Cloud Vault, que provê um cliente Vault para aplicações Spring Boot: https://cloud.spring.io/spring-cloud-vault/reference/html/

## Segurança em um Service Mesh

Conforme discutimos em capítulos anteriores, um Service Mesh como Istio ou Linkerd cuidam de várias necessidades de infraestrutura em uma Arquitetura de Microservices como resiliência, monitoramento, load balancing e service discovery.

Além dessas, um Service Mesh pode cuidar de necessidades de segurança como Confidencialidade, Autenticidade, Autenticação, Autorização e Auditoria. Assim, removemos a responsabilidade da segurança dos serviços e passaríamos para a infraestrutura que os conecta.

O Istio, por exemplo, provê de maneira descomplicada:

- Mutual Authentication com TLS 
- gerenciamento de chaves e rotação de credenciais com o componente Citadel
- whitelists e blacklists para restringir o acesso de certos serviços
- configuração de rate limiting, afim de evitar ataques DDoS (Distributed Denial of Service)

<!-- 
Referências:

https://blog.caelum.com.br/morte-a-sessao-entenda-esse-tal-de-stateless-session-com-tokens/

https://scotch.io/tutorials/the-ins-and-outs-of-token-based-authentication

https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm

https://microservices.io/patterns/security/access-token.html

Spring Microservices in Action
John Carnell; Kalpit Patel - Manning 2017
Capítulo 7 - Securing Microservices
Apêndice B - OAuth2 grant types
https://learning.oreilly.com/library/view/spring-microservices-in/9781617293986/OEBPS/Text/B.html

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

Building Microservices
Sam Newman  - O'Reilly Media, Inc.
February 2015
https://learning.oreilly.com/library/view/building-microservices/9781491950340/

CodeSpaces
https://www.infoworld.com/article/2608076/murder-in-the-amazon-cloud.html

https://pt.slideshare.net/AWSAktuell/secret-management-with-hashicorps-vault

https://istio.io/docs/concepts/security/
-->
