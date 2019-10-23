# Integração síncrona (e RESTful)

## Em busca das funcionalidades perdidas

Ao extrairmos os serviços de Pagamentos e de Distância do Monólito, o Caelum Eats perdeu algumas funcionalidades.

Depois de termos extraído o serviço de Pagamentos, depois de confirmar um pagamento, o status do pedido é mostrado como _REALIZADO_ e não como _PAGO_. Isso acontece porque a confirmação é feita no serviço de Pagamentos e removemos o código que atualizava o status do pedido, que é parte do módulo de Pedido do Monólito.

Já no caso do serviço de Distância, a extração em si não fez com que nenhuma funcionalidade fosse perdida. Porém, ao migramos os dados para um BD próprio, copiamos apenas os restaurantes do momento da migração. Mas dados de restaurantes podem ser modificados e novos restaurantes podem ser aprovados. E esses dados de restaurantes não estão sendo replicados para o BD de Distância.

Para que essas funcionalidades perdidas voltem a funcionar, temos que fazer uma integração entre os serviços de Pagamento e Distância e o Monólito.

## Integrando sistemas com o protocolo da Web

Vamos implementar essa integração entre sistemas usando o protocolo da Web, o HTTP (Hyper Text Transfer Protocol).

### A história do HTTP

Mas da onde vem o HTTP?

No década de 80, (o agora Sir) Tim Berners-Lee trabalhava no CERN, a Organização Europeia para a Pesquisa Nuclear. Em 1989, Berners-Lee criou uma aplicação que provia uma UI para diferentes documentos como relatórios, notas, documentação, etc. Para isso, baseou-se no conceito de _hypertext_, em que nós de informação são ligados a outros nós formando uma teia em que o usuário pode navegar. E com o nascimento de Internet, a rede mundial de computadores, essa navegação poderia expor informações de diferentes servidores dentro e fora do CERN. Berners-Lee chamou essa teia mundial de documentos ligados uns aos outros de _World Wide Web_.

<!--@note

Alexandre: costumo a falar do CERN e de que os artigos dos pesquisadores faziam referências a outros artigos, que referenciavam outros, formando uma teia de artigos. Uma Web.

-->

Então, a equipe de Tim Berners-Lee criou alguns softwares para essa aplicação:

- um servidor Web que provia documentos pela Internet
- um cliente Web, o navegador, que permitia aos usuários visualizar e seguir os links entre os documentos
- o HTML, um formato para os documentos
- o HTTP, o protocolo de comunicação entre o navegador e o servidor Web

O protocolo HTTP foi inicialmente especificado, em sua versão 1.0, pela Internet Engineering Task Force (IETF) na [RFC 1945](https://tools.ietf.org/html/rfc1945) (BERNES-LEE et al., 1996), em um grupo de trabalho liderado por Tim Berners-Lee, Roy Fielding e Henrik Frystyk. Desde então, diversas atualizações foram feitas no protocolo, em diferentes RFCs.

O HTTP é um protocolo do tipo request/response, em que um cliente como um navegador faz uma chamada ao servidor Web e fica aguardando os dados de resposta. Tanto o cliente como o servidor precisam estar no ar ao mesmo tempo para que a chamada seja feita com sucesso. Portanto, podemos dizer que o HTTP é um protocolo _síncrono_.

<!--@note

Alexandre: costumo a falar de marcar consulta no médico. Antigamente, tínhamos que ficar no telefone, tentando ser atendidos e ficava TU TU TU. Aí, ao sermos atendidos, já perguntaríamos sobre os horários disponíveis, sobre o preço e já teríamos a resposta na hora.
Hoje em dia, podemos marcar consultas pelo Whatsapp. Você manda uma pergunta para um consultório e pode fazer outras coisas, enquanto espera. Quando vier a resposta, você não precisa parar o que está fazendo imediatamente. Quando puder, você responde, fazendo novas perguntas. E por aí vai...

-->

HTTP é o protocolo do maior Sistema Distribuído do mundo, a Web, que usada diariamente por bilhões de pessoas. Podemos dizer que é um protocolo bem sucedido!

O HTTP tem algumas ideias interessantes. Vamos estudá-las a seguir.

### Recursos

Um **recurso** é um substantivo, uma coisa que está em um servidor Web e pode ser acessado por diferentes clientes. Pode ser um livro, uma lista de restaurantes, um post em um blog.

Todo recurso tem um endereço, uma **URL** (_Uniform Resource Locator_). Por exemplo, a URL dos tópicos mais recentes de Java no fórum da Alura:

https://cursos.alura.com.br/forum/subcategoria-java/todos/1

> URL é um conceito da Internet e não só da Web. A especificação inicial foi feita na [RFC 1738](https://tools.ietf.org/html/rfc1738) (BERNES-LEE et al., 1994) pela IETF. Podemos ter URLs para recursos disponíveis por FTP, SMTP ou AMQP. Por exemplo, uma URL de conexão com o RabbitMQ que usaremos mais adiante no curso:
>
> `amqp://eats:caelum123@rabbitmq:5672`
>
> Um _URI_ (_Uniform Resource Identifier_) é uma generalização de URLs que identifica um recurso que não necessariamente está exposto em uma rede. Foi especificado inicialmente pela IETF na [RFC 2396](https://tools.ietf.org/html/rfc2396) (BERNES-LEE et al., 1998). Por exemplo, um Data URI que representa uma imagem PNG de um pequeno ponto vermelho:
>
> `data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAUAAAAFCAYAAACNbyblAAAAHElEQVQI12P4//8/w38GIAXDIBKE0DHxgljNBAAO9TXL0Y4OHwAAAABJRU5ErkJggg==`

### Representações

No HTTP, um recurso pode ter diferentes **representações**. Por exemplo, os dados de um livro da [Casa do Código](https://www.casadocodigo.com.br/), disponível na URL https://www.casadocodigo.com.br/products/livro-git-github, pode ser representado em XML:

```xml
<livro>
  <nome>Controlando versões com Git e GitHub</nome>
  <autores>
    <autor>Alexandre Aquiles</autor>
    <autor>Rodrigo Caneppele</autor>
  </autores>
  <paginas>220</paginas>
  <ISBN>978-85-66250-53-4</ISBN>
</livro>
```

O mesmo recurso, da URL https://www.casadocodigo.com.br/products/livro-git-github, pode ser representado em JSON:

```json
{
  "nome": "Controlando versões com Git e GitHub",
  "autores": [
    { "autor": "Alexandre Aquiles" },
    { "autor": "Rodrigo Caneppele" }
  ],
  "paginas": 220,
  "ISBN": "978-85-66250-53-4"
}
```

E é possível ter representações do mesmo recurso, o livro da [Casa do Código](https://www.casadocodigo.com.br/), em formatos de ebook como PDF, EPUB e MOBI.

As representações de um recurso devem seguir um **Media Type**. Media Types são padronizados pela Internet Assigned Numbers Authority (IANA), a mesma organização que mantém endereços IP, time zones, os top level domains do DNS, etc.

_Curiosidade: os Media Types eram originalmente definidos como MIME (Multipurpose Internet Mail Extensions) Types, em uma especificação que definia o conteúdo de emails e seus anexos._

Entre os Media Types comuns, estão:

- `text/html` para HTML
- `text/plain` para texto puro (mas cuidado com a codificação!)
- `image/png` para imagens no formato PNG
- `application/json`, para JSON
- `application/xml` para XML
- `application/pdf` para PDF
- `application/epub+zip` para ebooks EPUB
- `application/vnd.amazon.mobi8-ebook` para ebooks MOBI
- `application/vnd.ms-excel` para arquivos `.xls` do Microsoft Excel

### Cabeçalhos

Tanto um request como um response HTTP podem ter, além de um corpo, metadados nos **Cabeçalhos** HTTP. Os cabeçalhos possíveis são especificados por RFCs na IETF e atualizados pela IANA. Alguns dos cabeçalhos mais utilizados:

- `Accept`: usado no request para indicar qual representação (Media Type) é aceito no response
- `Access-Control-Allow-Origin`: usado no response por chamadas CORS para indicar quais origins podem acessar um recurso
- `Authorization`: usado no request para passar credenciais de autenticação
- `Content-type`: a representação (Media Type) usado no request ou no response
- `ETag`: usado no response para indicar a versão de um recurso
- `If-None-Match`: usado no request com um ETag de um recurso, permitindo _caching_
- `Location`: usado no response para indicar uma URL de redirecionamento ou o endereço de um novo recurso

Os cabeçalhos HTTP `Accept` e `Content-type` permitem a **Content negotiation** (negociação de conteúdo), em que um cliente pode negociar com um servidor Web representações aceitáveis.

Por exemplo, um cliente pode indicar no request que aceita JSON e XML como representações, com seguinte cabeçalho:

```txt
Accept: application/json, application/xml
```

O servidor Web pode escolher entre essas duas representações. Se entre os formatos suportados pelo servidor não estiver JSON mas apenas XML, o response teria o cabeçalho:

```txt
Content-type: application/xml
```

No corpo do response, estaria um XML representado os dados do recurso.

> Um navegador sempre usa o cabeçalho `Accept` em seus requests. Por exemplo, no Mozilla Firefox:
>
> ```txt
> Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
> ```
>
> O cabeçalho anterior indica que o navegador Mozilla Firefox aceita do servidor as representações HTML, XHTML ou XML, nessa ordem. Se nenhuma dessas estiver disponível, pode ser enviada qualquer representação, indicada pelo `*/*`.
>
> O parâmetro `q` utilizado no cabeçalho anterior é um _relative quality factor_, ou fator relativo de qualidade, que indica a preferência por uma representação. O valor varia entre `0`, indicando menor preferência, e `1`, o valor padrão que indica uma maior preferência. No cabeçalho `Accept` anterior, o HTML e XHTML tem valor `1`, XML tem valor `0.9` e qualquer outra representação com valor `0.8`.

### Métodos

Para indicar uma ação a ser efetuada em um determinado recurso, o HTTP define os **métodos**. Se os recursos são os substantivos, os métodos são os verbos, como são comumente chamados.

O HTTP define apenas 9 métodos, cada um com o seu significado e uso diferentes:

- `GET`: usado para obter uma representação de um recurso em uma determinada URL.
- `HEAD`: usado para obter os metadados (cabeçalhos) de um recurso sem a sua representação. É um GET sem o corpo do response.
- `POST`: a representação do recurso passada no request é usada para criar um novo recurso subordinado no servidor, com sua própria URL.
- `PUT`: o request contém uma representação do recurso que será utilizada para atualizar ou criar um recurso na URL informada.
- `PATCH`: o request contém uma representação parcial de um recurso, que será utilizada para atualizá-lo. É uma adição tardia ao protocolo, especificada na [RFC 5789](https://tools.ietf.org/html/rfc5789) (DUSSEAULT; SNELL, 2010).
- `DELETE`: o recurso da URL informada é removido do servidor.
- `OPTIONS`: retorna os métodos HTTP suportados por uma URL.
- `TRACE`: repete o request, para que o cliente saiba se há alguma alteração feita por servidores intermediários.
- `CONNECT`: transforma o request em um túnel TCP/IP para permitir comunicação encriptada através de um proxy.

<!--@note

  Uma coisa a ser notada é que PUT não é necessariamente só pra atualizar um recurso. No RFC, a diferença entre POST e PUT é que o POST cria um recurso com uma nova URI.

  The PUT method requests that the enclosed entity be stored under the supplied Request-URI.

  If a new resource is created, the origin server MUST inform the user agent via the 201 (Created) response.

  The fundamental difference between the POST and PUT requests is reflected in the different meaning of the Request-URI. The URI in a POST request identifies the resource that will handle the enclosed entity. That resource might be a data-accepting process, a gateway to some other protocol, or a separate entity that accepts annotations. In contrast, the URI in a PUT request identifies the entity enclosed with the request -- the user agent knows what URI is intended and the server MUST NOT attempt to apply the request to some other resource.

  https://www.w3.org/Protocols/rfc2616/rfc2616-sec9.html

-->

Os métodos que não causam efeitos colaterais e cuja intenção é recuperação de dados são classificados como **safe**. São eles: GET, HEAD, OPTIONS e TRACE. Já os métodos que mudam os recursos ou causam efeitos em sistemas externos, como transações financeiras ou transmissão de emails, não são considerados safe.

Já os métodos em que múltiplos requests idênticos tem o mesmo efeito de apenas um request são classificados de **idempotent**, podendo ter ou não efeitos colaterais. Todos os métodos safe são idempotentes e também os métodos PUT e DELETE. Os métodos POST e CONNECT não são idempotentes.

O método PATCH não é considerado nem safe nem idempotente pela [RFC 5789](https://tools.ietf.org/html/rfc5789) (DUSSEAULT; SNELL, 2010), já que parte de um estado específico do recurso no servidor e só contém aquilo que deve ser alterado. Dessa maneira, múltiplos PATCHs podem ter efeitos distintos no servidor, pois o estado do recurso pode ser diferente entre os requests.

A ausência de efeitos colaterais e idempotências dos métodos assim classificados fica a cargo do desenvolvedor, não sendo garantidas pelo protocolo nem pelos servidores Web.

Resumindo:

- métodos safe: GET, HEAD, OPTIONS e TRACE
- métodos idempotentes: os métodos safe, PUT e DELETE
- métodos nem safe nem idempotentes: POST, CONNECT e PATCH

É importante notar que apenas esses 9 métodos, ou até um subconjunto deles, são suficientes para a maioria das aplicações distribuídas. Geralmente são descritos como uma **interface uniforme**.

### Códigos de Status

Um response HTTP pode ter diferentes códigos de status, especificados em RFCs da IETF e mantidos pela IANA.

O primeiro dígito indica a categoria do response:

- `1XX (Informational)`: o request foi recebido e o processamento continua
- `2XX (Success)`: o request foi recebido, entendido e aceito com sucesso
- `3XX (Redirection)`: mais ações são necessárias para completar o request
- `4XX (Client Error)`: o request é inválido e contém erros causados pelo cliente
- `5XX (Server Error)`: o servidor falhou em completar um request válido

Alguns dos códigos de status mais comuns:

- `101 Switching Protocols`: o cliente solicitou a troca de protocolos e o servidor aceitou, trocando para o protocolo indicado no cabeçalho `Upgrade`. Usado para iniciar uma conexão a um WebSocket.

- `200 OK`: código padrão de sucesso.
- `201 Created`: indica que um novo recurso foi criado. Em geral, o response contém a URL do novo recurso no cabeçalho `Location`.
- `202 Accepted`: o request foi aceito para processamento mas ainda não foi completado.
- `204 No Content`: o request foi processado com sucesso mas não há corpo no response.

- `301 Moved Permanently`: todos os requests futuros devem ser redirecionados para a URL indicada no cabeçalho `Location`.
- `302 Found`: o cliente deve redirecionar para a URL indicada no cabeçalho `Location`. Navegadores, frameworks e aplicações a implementam como um _redirect_, que seria o intuito do código `303`.
- `303 See Other`: o request foi completado com sucesso mas o response deve ser encontrado na URL indicada no cabeçalho `Location` por meio de um `GET`. O intuito era ser utilizado para um _redirect_, de maneira a implementar o pattern _POST/redirect/GET_. Criado a partir do `HTTP 1.1`.
- `304 Not Modified`: indica que a versão (`ETag`) do recurso não foi modificada em relação à informada no cabeçalho `If-None-Match` do request. Portanto, não há a necessidade de transmitir uma representação do recurso. O cliente tem a última versão em seu cache. 

- `400 Bad Request`: o cliente enviou um request inválido.
- `401 Unauthorized`: o cliente tentou acessar um recurso protegido em que não tem as permissões necessárias. O request pode ser refeito se passado um cabeçalho `Authorization` que contenha credenciais de um usuário com permissão para acessar o recurso.
- `403 Forbidden`: o cliente tentou uma ação proibida ou o usuário indicado no cabeçalho `Authorization` não tem acesso ao recurso solicitado.
- `404 Not Found`: o recurso não existe no servidor.
- `405 Method Not Allowed`: o cliente usou um método HTTP não suportado pelo recurso solicitado.
- `406 Not Acceptable`: o servidor não consegue gerar uma representação compatível com nenhum valor do cabeçalho `Accept` do request.
- `415 Unsupported Media Type`: o request foi enviado com uma representação, indicada no `Content-type`, não suportada pelo servidor.
- `429 Too Many Requests`: o cliente enviou requests excessivos em uma determinada fatia de tempo. Usado ao implementar _rate limiting_ com o intuito de previnir contra ataques Denial of Service (DoS).

- `500 Internal Server Error`: um erro inesperado aconteceu no servidor. O erro do _"Bad, bad server. No donut for your._ do Orkut e da baleia do Twitter.
- `503 Service Unavailable`: servidor em manutenção ou sobrecarregado temporariamente.
- `504 Gateway Timeout`: o servidor está servindo como um proxy para um request mas não recebeu a tempo um response do servidor de destino.

<!--@note

  Alexandre: tenho o costume de mostrar os sites:
  - http.cat, que tem memes de gatos
  - httpstatusdogs.com, que tem memes de cachorros.

-->

### Links

A Web tem esse nome por ser uma teia de documentos ligados entre si. Links, ou hypertext, são conceitos muito importantes na Web e podem ser usado na integração de sistemas. Veremos como mais adiante.

## Cliente REST com RestTemplate do Spring

No `eats-distancia-service`, crie um Controller chamado `RestaurantesController` no pacote `br.com.caelum.eats.distancia` com um método que insere um novo restaurante e outro que atualiza um restaurante existente. Defina mensagens de log em cada método.

####### fj33-eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/RestaurantesController.java

```java
@RestController
@AllArgsConstructor
@Slf4j
class RestaurantesController {

  private RestauranteRepository repo;

  @PostMapping("/restaurantes")
  ResponseEntity<Restaurante> adiciona(@RequestBody Restaurante restaurante, UriComponentsBuilder uriBuilder) {
    log.info("Insere novo restaurante: " + restaurante);
    Restaurante salvo = repo.insert(restaurante);
    UriComponents uriComponents = uriBuilder.path("/restaurantes/{id}").buildAndExpand(salvo.getId());
    URI uri = uriComponents.toUri();
    return ResponseEntity.created(uri).contentType(MediaType.APPLICATION_JSON).body(salvo);
  }

  @PutMapping("/restaurantes/{id}")
  Restaurante atualiza(@PathVariable("id") Long id, @RequestBody Restaurante restaurante) {
    if (!repo.existsById(id)) {
      throw new ResourceNotFoundException();
    }
    log.info("Atualiza restaurante: " + restaurante);
    return repo.save(restaurante);
  }

}
```

Certifique-se que os imports estão corretos:

```java
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.PutMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import lombok.AllArgsConstructor;
import lombok.extern.slf4j.Slf4j;
```

No `application.properties` do módulo `eats-application` do monólito, crie uma propriedade `configuracao.distancia.service.url` para indicar a URL do serviço de distância:

####### fj33-eats-monolito-modular/eats/eats-application/src/main/resources/application.properties

```properties
configuracao.distancia.service.url=http://localhost:8082
```

No módulo `eats-application` do monólito, crie uma classe `RestClientConfig` no pacote `br.com.caelum.eats`, que fornece um `RestTemplate` do Spring:

####### fj33-eats-monolito-modular/eats/eats-application/src/main/java/br/com/caelum/eats/RestClientConfig.java

```java
@Configuration
class RestClientConfig {

  @Bean
  RestTemplate restTemplate() {
    return new RestTemplate();
  }

}
```

Faça os imports adequados:

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.client.RestTemplate;
```

No módulo `eats-restaurante` do monólito, crie uma classe `RestauranteParaServicoDeDistancia` no pacote `br.com.caelum.eats.restaurante` que contém apenas as informações adequadas para o serviço de distância. Crie um construtor que recebe um `Restaurante` e popula os dados necessários:

####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/RestauranteParaServicoDeDistancia.java

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
class RestauranteParaServicoDeDistancia {

  private Long id;
  private String cep;
  private Long tipoDeCozinhaId;

  RestauranteParaServicoDeDistancia(Restaurante restaurante){
    this(restaurante.getId(), restaurante.getCep(), restaurante.getTipoDeCozinha().getId());
  }

}
```

Não esqueça de definir os imports:

```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
```

Observação: a anotação `@Data` do Lombok define um Java Bean com getters, setters para campos mutáveis, `equals` e `hashcode` e `toString`.

Crie uma classe `DistanciaRestClient` no pacote `br.com.caelum.eats.restaurante` do módulo `eats-restaurante` do monólito. Defina como dependências um `RestTemplate` e uma `String` para armazenar a propriedade `configuracao.distancia.service.url`.

Anote a classe com `@Service` do Spring.

Defina métodos que chamam o serviço de distância para:

- inserir um novo restaurante aprovado, enviando um POST para `/restaurantes` com o `RestauranteParaServicoDeDistancia` como corpo da requisição
- atualizar um restaurante já existente, enviando um PUT para `/restaurantes/{id}`, com o `id` adequado e um `RestauranteParaServicoDeDistancia` no corpo da requisição

####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/DistanciaRestClient.java

```java
@Service
class DistanciaRestClient {

  private String distanciaServiceUrl;
  private RestTemplate restTemplate;

  DistanciaRestClient(RestTemplate restTemplate,
                                      @Value("${configuracao.distancia.service.url}") String distanciaServiceUrl) {
    this.distanciaServiceUrl = distanciaServiceUrl;
    this.restTemplate = restTemplate;
  }

  void novoRestauranteAprovado(Restaurante restaurante) {
    RestauranteParaServicoDeDistancia restauranteParaDistancia = new RestauranteParaServicoDeDistancia(restaurante);
    String url = distanciaServiceUrl+"/restaurantes";
    ResponseEntity<RestauranteParaServicoDeDistancia> responseEntity =
        restTemplate.postForEntity(url, restauranteParaDistancia, RestauranteParaServicoDeDistancia.class);
    HttpStatus statusCode = responseEntity.getStatusCode();
    if (!HttpStatus.CREATED.equals(statusCode)) {
      throw new RuntimeException("Status diferente do esperado: " + statusCode);
    }
  }

  void restauranteAtualizado(Restaurante restaurante) {
    RestauranteParaServicoDeDistancia restauranteParaDistancia = new RestauranteParaServicoDeDistancia(restaurante);
    String url = distanciaServiceUrl+"/restaurantes/" + restaurante.getId();
    restTemplate.put(url, restauranteParaDistancia, RestauranteParaServicoDeDistancia.class);
  }

}
```

Os imports corretos são:

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;
```

Altere a classe `RestauranteController` do módulo `eats-restaurante` do monólito para que:

- tenha um `DistanciaRestClient` como dependência
- no caso de aprovação de um restaurante, invoque o método `novoRestauranteAprovado` de `DistanciaRestClient`
- no caso de atualização do CEP ou tipo de cozinha de um restaurante já aprovado, invoque o método `restauranteAtualizado` de `DistanciaRestClient`

####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/RestauranteController.java

```java
// anotações ...
class RestauranteController {

  private RestauranteRepository restauranteRepo;
  private CardapioRepository cardapioRepo;
  private DistanciaRestClient distanciaRestClient; // adicionado

  // métodos omitidos ...

  @PutMapping("/parceiros/restaurantes/{id}")
  Restaurante atualiza(@RequestBody Restaurante restaurante) {
    Restaurante doBD = restauranteRepo.getOne(restaurante.getId());
    restaurante.setUser(doBD.getUser());
    restaurante.setAprovado(doBD.getAprovado());

    Restaurante salvo = restauranteRepo.save(restaurante);

    if (restaurante.getAprovado() &&
              (cepDiferente(restaurante, doBD) || tipoDeCozinhaDiferente(restaurante, doBD))) {

      distanciaRestClient.restauranteAtualizado(restaurante);

    }

    return salvo;
  }

  // método omitido ...

  @Transactional
  @PatchMapping("/admin/restaurantes/{id}")
  void aprova(@PathVariable("id") Long id) {
    restauranteRepo.aprovaPorId(id);

    // adicionado
    Restaurante restaurante = restauranteRepo.getOne(id);
    distanciaRestClient.novoRestauranteAprovado(restaurante);
  }

  private boolean tipoDeCozinhaDiferente(Restaurante restaurante, Restaurante doBD) {
    return !doBD.getTipoDeCozinha().getId().equals(restaurante.getTipoDeCozinha().getId());
  }

  private boolean cepDiferente(Restaurante restaurante, Restaurante doBD) {
    return !doBD.getCep().equals(restaurante.getCep());
  }

}
```

Observação: pensando em design de código, será que os métodos auxiliares `tipoDeCozinhaDiferente` e `cepDiferente` deveriam ficar em `RestauranteController` mesmo?

## Exercício: Testando a integração entre o módulo de restaurantes do monólito e o serviço de distância

1. Interrompa o monólito e o serviço de distância.

  Em um terminal, vá até a branch `cap6-integracao-monolito-distancia-com-rest-template` dos projetos `fj33-eats-monolito-modular` e `fj33-eats-distancia-service`:

  ```sh
  cd ~/Desktop/fj33-eats-monolito-modular
  git checkout -f cap6-integracao-monolito-distancia-com-rest-template
  
  cd ~/Desktop/fj33-eats-distancia-service
  git checkout -f cap6-integracao-monolito-distancia-com-rest-template
  ```

  Suba o monólito executando a classe `EatsApplication` e o serviço de distância por meio da classe `EatsDistanciaServiceApplication`.

2. Efetue login como um dono de restaurante.

  O restaurante Long Fu, que já vem pré-cadastrado, tem o usuário `longfu` e a senha `123456`.

  Faça uma mudança no tipo de cozinha ou CEP do restaurante.

  Verifique nos logs que o restaurante foi atualizado no serviço de distância.

  Se desejar, cadastre um novo restaurante. Então, faça login como Adminstrador do Caelum Eats: o usuário é `admin` e a senha é `123456`.
  
  Aprove o novo restaurante. O serviço de distância deve ter sido chamado. Veja nos logs.

  No diretório do `docker-compose.yml`, acesse o database de distância no MongoDB com o Mongo Shell:

  ```sh
  cd ~/Desktop
  docker-compose exec mongo.distancia mongo eats_distancia
  ```

  Então, veja o conteúdo da collection restaurantes com o comando:

  ```js
  db.restaurantes.find();
  ```

## Cliente REST declarativo com Feign

Adicione ao `PedidoController`, do módulo `eats-pedido` do monólito, um método que muda o status do pedido para _PAGO_:

####### fj33-eats-monolito-modular/eats/eats-pedido/src/main/java/br/com/caelum/eats/pedido/PedidoController.java

```java
@PutMapping("/pedidos/{id}/pago")
void pago(@PathVariable("id") Long id) {
  Pedido pedido = repo.porIdComItens(id);
  if (pedido == null) {
    throw new ResourceNotFoundException();
  }
  pedido.setStatus(Pedido.Status.PAGO);
  repo.atualizaStatus(Pedido.Status.PAGO, pedido);
}
```

No arquivo `application.properties` de `eats-pagamento-service`, adicione uma propriedade `configuracao.pedido.service.url` que contém a URL do monólito:

####### fj33-eats-pagamento-service/src/main/resources/application.properties

```properties
configuracao.pedido.service.url=http://localhost:8080
```

No `pom.xml` de `eats-pagamento-service`, adicione uma dependência ao _Spring Cloud_ na versão `Greenwich.SR2`, em `dependencyManagement`:

####### fj33-eats-pagamento-service/pom.xml

```xml
<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>org.springframework.cloud</groupId>
      <artifactId>spring-cloud-dependencies</artifactId>
      <version>Greenwich.SR2</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
  </dependencies>
</dependencyManagement>
```

Feito isso, adicione o _starter_ do OpenFeign como dependência:

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

Anote a classe `EatsPagamentoServiceApplication` com `@EnableFeignClients` para habilitar o Feign:

####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/EatsPagamentoServiceApplication.java

```java
@EnableFeignClients // adicionado
@SpringBootApplication
public class EatsPagamentoServiceApplication {

  // código omitido ...

}
```

O import correto é o seguinte:

```java
import org.springframework.cloud.openfeign.EnableFeignClients;
```

Defina, no pacote `br.com.caelum.eats.pagamento` de `eats-pagamento-service`, uma interface `PedidoRestClient` com um método `avisaQueFoiPago`, anotados da seguinte maneira:

####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PedidoRestClient.java

```java
@FeignClient(url="${configuracao.pedido.service.url}", name="pedido")
interface PedidoRestClient {

  @PutMapping("/pedidos/{pedidoId}/pago")
  void avisaQueFoiPago(@PathVariable("pedidoId") Long pedidoId);

}
```

Ajuste os imports:

```java
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.PutMapping;
```

Em `PagamentoController`, do serviço de pagamento, defina um `PedidoRestClient` como atributo e use o método `avisaQueFoiPago` passando o id do pedido:

####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PagamentoController.java

```java
// anotações ...
public class PagamentoController {

  private PagamentoRepository pagamentoRepo;
  private PedidoRestClient pedidoClient; // adicionado

  // código omitido ...

  @PutMapping("/{id}")
  public PagamentoDto confirma(@PathVariable Long id) {
    Pagamento pagamento = pagamentoRepo.findById(id).orElseThrow(() -> new ResourceNotFoundException());
    pagamento.setStatus(Pagamento.Status.CONFIRMADO);
    pagamentoRepo.save(pagamento);

    // adicionado
    Long pedidoId = pagamento.getPedidoId();
    pedidoClient.avisaQueFoiPago(pedidoId);

    return new PagamentoDto(pagamento);
  }

  // restante do código ...

}
```

## Exercício: Testando a integração entre o serviço de pagamento e o módulo de pedidos do monólito

1. Interrompa o monólito e o serviço de pagamentos.

  Em um terminal, vá até a branch `cap6-integracao-pagamento-monolito-com-feign` dos projetos `fj33-eats-monolito-modular` e `fj33-eats-pagamento-service`:

  ```sh
  cd ~/Desktop/fj33-eats-monolito-modular
  git checkout -f cap6-integracao-pagamento-monolito-com-feign
  
  cd ~/Desktop/fj33-eats-pagamento-service
  git checkout -f cap6-integracao-pagamento-monolito-com-feign
  ```

  Suba o monólito executando a classe `EatsApplication` e o serviço de pagamentos por meio da classe `EatsPagamentoServiceApplication`.

2. Certifique-se que o serviço de pagamento foi reiniciado e que os demais serviços e o front-end estão no ar.

  Faça um novo pedido, realizando e confirmando um pagamento.
  
  Veja que, depois dessa mudança, o status do pedido fica como **_PAGO_** e não apenas como _REALIZADO_.

## Exercício opcional: Spring HATEOAS e HAL

1. Adicione o Spring HATEOAS como dependência no `pom.xml` de `eats-pagamento-service`:

  ####### fj33-eats-pagamento-service/pom.xml

  ```xml
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-hateoas</artifactId>
  </dependency>
  ```

2. Nos métodos de `PagamentoController`, retorne um `Resource` com uma lista de `Link` do Spring HATEOAS.

  - Em todos os métodos, defina um _link relation_ `self` que aponta para o próprio recurso, através da URL do método `detalha`
  - Nos métodos `detalha` e `cria`, defina _link relations_ `confirma` e `cancela`, apontando para as URLS associadas aos respectivos métodos de `PagamentoController`.

  Para criar os _links_, utilize os métodos estáticos `methodOn` e `linkTo` de `ControllerLinkBuilder`.

  O código de `PagamentoController` ficará semelhante a:

  ####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PagamentoController.java

  ```java
  @RestController
  @RequestMapping("/pagamentos")
  @AllArgsConstructor
  class PagamentoController {

    private PagamentoRepository pagamentoRepo;
    private PedidoRestClient pedidoClient;

    @GetMapping("/{id}")
    public Resource<PagamentoDto> detalha(@PathVariable Long id) {
      Pagamento pagamento = pagamentoRepo.findById(id).orElseThrow(() -> new ResourceNotFoundException());

      List<Link> links = new ArrayList<>();

      Link self = linkTo(methodOn(PagamentoController.class).detalha(id)).withSelfRel();
      links.add(self);

      if (Pagamento.Status.CRIADO.equals(pagamento.getStatus())) {
        Link confirma = linkTo(methodOn(PagamentoController.class).confirma(id)).withRel("confirma");
        links.add(confirma);

        Link cancela = linkTo(methodOn(PagamentoController.class).cancela(id)).withRel("cancela");
        links.add(cancela);
      }

      PagamentoDto dto = new PagamentoDto(pagamento);
      Resource<PagamentoDto> resource = new Resource<PagamentoDto>(dto, links);

      return resource;
    }

    @PostMapping
    public ResponseEntity<Resource<PagamentoDto>> cria(@RequestBody Pagamento pagamento,
        UriComponentsBuilder uriBuilder) {
      pagamento.setStatus(Pagamento.Status.CRIADO);
      Pagamento salvo = pagamentoRepo.save(pagamento);
      URI path = uriBuilder.path("/pagamentos/{id}").buildAndExpand(salvo.getId()).toUri();
      PagamentoDto dto = new PagamentoDto(salvo);

      Long id = salvo.getId();

      List<Link> links = new ArrayList<>();

      Link self = linkTo(methodOn(PagamentoController.class).detalha(id)).withSelfRel();
      links.add(self);

      Link confirma = linkTo(methodOn(PagamentoController.class).confirma(id)).withRel("confirma");
      links.add(confirma);

      Link cancela = linkTo(methodOn(PagamentoController.class).cancela(id)).withRel("cancela");
      links.add(cancela);

      Resource<PagamentoDto> resource = new Resource<PagamentoDto>(dto, links);
      return ResponseEntity.created(path).body(resource);
    }

    @PutMapping("/{id}")
    public Resource<PagamentoDto> confirma(@PathVariable Long id) {
      Pagamento pagamento = pagamentoRepo.findById(id).orElseThrow(() -> new ResourceNotFoundException());
      pagamento.setStatus(Pagamento.Status.CONFIRMADO);
      pagamentoRepo.save(pagamento);

      Long pedidoId = pagamento.getPedidoId();
      pedidoClient.avisaQueFoiPago(pedidoId);

      List<Link> links = new ArrayList<>();

      Link self = linkTo(methodOn(PagamentoController.class).detalha(id)).withSelfRel();
      links.add(self);

      PagamentoDto dto = new PagamentoDto(pagamento);
      Resource<PagamentoDto> resource = new Resource<PagamentoDto>(dto, links);

      return resource;
    }

    @DeleteMapping("/{id}")
    public Resource<PagamentoDto> cancela(@PathVariable Long id) {
      Pagamento pagamento = pagamentoRepo.findById(id).orElseThrow(() -> new ResourceNotFoundException());
      pagamento.setStatus(Pagamento.Status.CANCELADO);
      pagamentoRepo.save(pagamento);

      List<Link> links = new ArrayList<>();

      Link self = linkTo(methodOn(PagamentoController.class).detalha(id)).withSelfRel();
      links.add(self);

      PagamentoDto dto = new PagamentoDto(pagamento);
      Resource<PagamentoDto> resource = new Resource<PagamentoDto>(dto, links);

      return resource;
    }

  }
  ```

3. Reinicie o serviço de pagamentos e obtenha o pagamento de um `id` já cadastrado:

  ```sh
   curl -i http://localhost:8081/pagamentos/1
  ```

  A resposta será algo como:

  ```text
  HTTP/1.1 200 
  Content-Type: application/hal+json;charset=UTF-8
  Transfer-Encoding: chunked
  Date: Tue, 28 May 2019 19:04:43 GMT
  ```

  <!--  -->

  ```json
  {
    "id":1,
    "valor":51.80,
    "nome":"ANDERSON DA SILVA",
    "numero":"1111 2222 3333 4444",
    "expiracao":"2022-07",
    "codigo":"123",
    "status":"CRIADO",
    "formaDePagamentoId":2,
    "pedidoId":1,
    "_links":{
      "self":{
        "href":"http://localhost:8081/pagamentos/1"
      },
      "confirma":{
        "href":"http://localhost:8081/pagamentos/1"
      },
      "cancela":{
        "href":"http://localhost:8081/pagamentos/1"
      }
    }
  }
  ```
  
  Teste também a criação, confirmação e cancelamento de novos pagamentos.

4. Altere o código do front-end para usar os _link relations_ apropriados ao confirmar ou cancelar um pagamento:

  ####### fj33-eats-ui/src/app/services/pagamento.service.ts

  ```typescript
  confirma(pagamento): Observable<any> {
    t̶h̶i̶s̶.̶a̶j̶u̶s̶t̶a̶I̶d̶s̶(̶p̶a̶g̶a̶m̶e̶n̶t̶o̶)̶;̶

    const url = pagamento._links.confirma.href; // adicionado

    r̶e̶t̶u̶r̶n̶ ̶t̶h̶i̶s̶.̶h̶t̶t̶p̶.̶p̶u̶t̶(̶`̶$̶{̶t̶h̶i̶s̶.̶A̶P̶I̶}̶/̶$̶{̶p̶a̶g̶a̶m̶e̶n̶t̶o̶.̶i̶d̶}̶`̶,̶ ̶n̶u̶l̶l̶)̶;̶
    return this.http.put(url, null); // modificado
  }

  cancela(pagamento): Observable<any> {
    t̶h̶i̶s̶.̶a̶j̶u̶s̶t̶a̶I̶d̶s̶(̶p̶a̶g̶a̶m̶e̶n̶t̶o̶)̶;̶

    const url = pagamento._links.cancela.href; // adicionado

    r̶e̶t̶u̶r̶n̶ ̶t̶h̶i̶s̶.̶h̶t̶t̶p̶.̶d̶e̶l̶e̶t̶e̶(̶`̶$̶{̶t̶h̶i̶s̶.̶A̶P̶I̶}̶/̶$̶{̶p̶a̶g̶a̶m̶e̶n̶t̶o̶.̶i̶d̶}̶`̶)̶;̶
    return this.http.delete(url); // modificado
  }
  ```

  _Observação: o método auxiliar `ajustaIds` não é mais necessário ao confirmar e cancelar um pagamento, já que o `id` do pagamento não é mais usado para montar a URL. Porém, o método ainda é usado ao criar um pagamento._

5. Faça um novo pedido e efetue um pagamento. Deve continuar funcionando!

## Exercício opcional: Estendendo o Spring HATEOAS

1. Crie uma classe `LinkWithMethod` que estende o `Link` do Spring HATEOAS e define um atributo adicional chamado `method`, que armazenará o método HTTP dos links. Defina um construtor que recebe um `Link` e uma `String` com o método HTTP:

  ####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/LinkWithMethod.java

  ```java
  @Getter
  public class LinkWithMethod extends Link {

    private static final long serialVersionUID = 1L;

    private String method;

    public LinkWithMethod(Link link, String method) {
      super(link.getHref(), link.getRel());
      this.method = method;
    }
  }
  ```

  Os imports são os seguintes:

  ```java
  import org.springframework.hateoas.Link;
  import lombok.Getter;
  ```

2. Na classe `PagamentoController`, adicione um `LinkWithMethod` na lista para os links de confirmação e cancelamento, passando o método HTTP adequado.

  Use o trecho abaixo nos métodos `detalha` e `cria` de `PagamentoController`:

  ####### fj33-eats-pagamento-service/src/main/java/br/com/caelum/eats/pagamento/PagamentoController.java

  ```java
  Link confirma = linkTo(methodOn(PagamentoController.class).confirma(id)).withRel("confirma");
  l̶i̶n̶k̶s̶.̶a̶d̶d̶(̶c̶o̶n̶f̶i̶r̶m̶a̶)̶;̶
  links.add(new LinkWithMethod(confirma, "PUT")); // modificado

  Link cancela = linkTo(methodOn(PagamentoController.class).cancela(id)).withRel("cancela");
  l̶i̶n̶k̶s̶.̶a̶d̶d̶(̶c̶a̶n̶c̶e̶l̶a̶)̶;̶
  links.add(new LinkWithMethod(cancela, "DELETE")); // modificado
  ```

3. Usando o cURL, obtenha novamente uma representação de um pagamento já cadastrado:

  ```sh
   curl -i http://localhost:8081/pagamentos/1
  ```

   Deve ser retornado algo parecido com:

  ```text
  HTTP/1.1 200 
  Content-Type: application/hal+json;charset=UTF-8
  Transfer-Encoding: chunked
  Date: Tue, 28 May 2019 19:04:43 GMT
  ```

  <!--  -->

  ```json
  {
    "id":1,
    "valor":51.80,
    "nome":"ANDERSON DA SILVA",
    "numero":"1111 2222 3333 4444",
    "expiracao":"2022-07",
    "codigo":"123",
    "status":"CRIADO",
    "formaDePagamentoId":2,
    "pedidoId":1,
    "_links":{
      "self":{
        "href":"http://localhost:8081/pagamentos/1"
      },
      "confirma":{
        "href":"http://localhost:8081/pagamentos/1",
        "method": "PUT"
      },
      "cancela":{
        "href":"http://localhost:8081/pagamentos/1",
        "method": "DELETE"
      }
    }
  }
  ```

  Observe os métodos HTTP na propriedade `method` dos _link relations_ `confirma` e `cancela`. 

4. Ajuste o código do front-end para usar o `method` de cada _link relation_:

  ####### fj33-eats-ui/src/app/services/pagamento.service.ts

  ```typescript
  confirma(pagamento): Observable<any> {
    const url = pagamento._links.confirma.href;

    r̶e̶t̶u̶r̶n̶ ̶t̶h̶i̶s̶.̶h̶t̶t̶p̶.̶p̶u̶t̶(̶u̶r̶l̶,̶ ̶n̶u̶l̶l̶)̶;̶

    const method = pagamento._links.confirma.method;
    return this.http.request(method, url);
  }

  cancela(pagamento): Observable<any> {
    const url = pagamento._links.cancela.href;

    r̶e̶t̶u̶r̶n̶ ̶t̶h̶i̶s̶.̶h̶t̶t̶p̶.̶d̶e̶l̶e̶t̶e̶(̶u̶r̶l̶)̶;̶

    const method = pagamento._links.cancela.method;
    return this.http.request(method, url);
  }
  ```

4. (desafio) Modifique o `PagamentoController` para usar HAL-FORMS, disponível nas últimas versões do Spring HATEOAS.

<!--@note

---------------
Alexandre (BSB)
---------------

HAL-FORMS é uma das ideias de especificação de Hypermedia de Mike Amundsen, autor de diversos livros sobre REST na editor O'Reilly (aquela dos bichos na capa).

Usei o HAL-FORMS na versão milestone 2.2.0.M2 do Spring Boot no commit abaixo:

https://github.com/alexandreaquiles/eats/commit/f8ef33b88cd3d96c62627a13b4e8470c9f09ada0#diff-5f414af558500eda821060272d84b8d8

-->
