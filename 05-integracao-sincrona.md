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

### Cabeçalhos

Tanto um request como um response HTTP podem ter, além de um corpo, metadados nos **Cabeçalhos** HTTP. Os cabeçalhos possíveis são especificados por RFCs na IETF e atualizados pela IANA. Alguns dos cabeçalhos mais utilizados:

- `Accept`: usado no request para indicar qual representação (Media Type) é aceito no response
- `Access-Control-Allow-Origin`: usado no response por chamadas CORS para indicar quais origins podem acessar um recurso
- `Authorization`: usado no request para passar credenciais de autenticação
- `Allow`: usado no response para indicar os métodos HTTP válidos para o recurso
- `Content-type`: a representação (Media Type) usada no request ou no response
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
- `409 Conflict`: o request não pode ser processado por causa de um conflito no estado atual do recurso, como múltiplas atualizações simultâneas.
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

## REST, o estilo arquitetural da Web

Roy Fielding, um dos autores das especificações do protocolo HTTP e cofundador do Apache HTTP Server, estudou diferentes estilos arquiteturais de Sistemas Distribuídos em sua tese de PhD: [Architectural Styles and the Design of Network-based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm) (FIELDING, 2000).

As restrições e princípios que fundamentam o estilo arquitetural da Web são descrita por Fielding da seguinte maneira:

- _Cliente/Servidor_: a UI é separada do armazenamento dos dados, permitindo a portabilidade para diferentes plataformas e simplificando o Servidor.
- _Stateless_: cada request do cliente deve conter todos os dados necessários, sem tomar vantagem de nenhum contexto armazenado no servidor. Sessões de usuário devem ser mantidas no cliente. Essa característica melhor: a Escalabilidade, já que não há uso de recursos entre requests diferentes; Confiabilidade, já que torna mais fácil a recuperação de falhas parciais; Visibilidade, já que não há a necessidade de monitorar além de um único request. Como desvantagem, há uma piora na Performance da rede, um aumento de dados repetitivos entre requests e uma dependência da implementação correta dos múltiplos clientes.
- _Cache_: os requests podem ser classificados como cacheáveis, fazendo com que o cliente possa reusar o response para requests equivalentes. Assim, a Latência é reduzida de maneira a melhorar a Eficiência, Escalabilidade e a Performance percebida pelo usuário. Porém, a Confiabilidade pode ser afetada caso haja aumento significante de dados desatualizados.
- _Interface Uniforme_: URLs, representações, métodos padronizados e links são restrições da Web que simplificam a Arquitetura e aumentam a Visibilidade das interações, encorajando a evolução independente de cada parte. Por outro lado, são menos eficientes que protocolos específicos.
- _Sistema em Camadas_: cada componente só conhece a camada com a qual interage imediatamente, minimizando a complexidade, aumentando a independência e permitindo intermediários como _load balancers_, _firewalls_ e _caches_. Porém, pode ser adicionada latência.
- _Code-On-Demand_: os clientes podem estender as funcionalidades por meio da execução de _applets_ e _scripts_, aumentando a Extensibilidade. Por outro lado, a Visibilidade do sistema diminui.

Fielding chama esse estilo arquitetural da Web de Representational State Transfer, ou simplesmente **REST**. Adicionando o sufixo _-ful_, que denota "que possui a característica de" em inglês, podemos chamar serviços que seguem esse estilo arquitetural de _RESTful_.

<!--@note

  Alexandre: costumo a lista palavras terminadas com -ful, ressaltando que não é -full (com dois L):

    Beauty - Beatiful
    Pain - Painful
    REST - RESTful

-->

<!-- separador -->

> Um excelente resumo de boas práticas e princípios de uma API RESTful podem ser encontrado no blog da Caelum, no post [REST: Princípios e boas práticas](https://blog.caelum.com.br/rest-principios-e-boas-praticas) (FERREIRA, 2017), disponível em: https://blog.caelum.com.br/rest-principios-e-boas-praticas

Leonard Richardson e Sam Ruby, no livro [RESTful Web Services](https://learning.oreilly.com/library/view/restful-web-services/9780596529260/) (RICHARDSON; RUBY, 2007), contrastam serviços no estilo _Remote Procedure Call_ (RPC) com serviços _Resource-Oriented_.

Um Web Service no estilo RPC expõe operações de uma aplicação. Não há recursos, representações nem métodos. O SOAP é um exemplo de um protocolo nesse estilo RPC: há apenas um recurso com só uma URL (a do Web Service), há apenas uma representação (XML), há apenas um método (POST, em geral). Cada `operation` do Web Service SOAP é exposta num WSDL.

Richardson e Ruby mostram, no livro, uma API de busca do Google implementada com SOAP. Para buscar sobre "REST" deveríamos efetuar o seguinte request:

```txt
POST http://api.google.com/search/beta2
Content-Type: application/soap+xml
```

<!-- separador -->

```xml
<?xml version="1.0" encoding="UTF-8"?>
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <gs:doGoogleSearch xmlns:gs="urn:GoogleSearch">
      <q>REST</q>
      ...
    </gs:doGoogleSearch>
  </soap:Body>
</soap:Envelope>
```

A mesma consulta pode ser feita pela API mais moderna do Google, que expõe um recurso `search` que recebe um parâmetro `q` com o termo a ser consultado:

```txt
GET http://www.google.com/search?q=REST
Content-Type: text/html
```

## O Modelo de Maturidade de Richardson

No post [Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html) (FOWLER, 2010), Martin Fowler explica uma heurística para a maturidade da adoção de REST descrita por Leonard Richardson no "ato 3" de sua palestra [Justice Will Take Us Millions Of Intricate Moves](https://www.crummy.com/writing/speaking/2008-QCon/act3.html) (RICHARDSON, 2008).

O Modelo de Maturidade descrito por Richardson indica uma progressão na adoção das tecnologias da Web.

### Nível 0 - O Pântano do POX

No Nível 0 de Maturidade, o HTTP é usado apenas como um mecanismo de transporte para interações remotas no estilo RPC, sem a filosofia da Web.

Fowler usa o termo Plain Old XML (POX) para descrever APIs que provêem só um endpoint, todas as chamadas usam POST, a única representação é XML. As mensagens de erro contém um status code de sucesso (200) com os detalhes do erro no corpo do response. É o caso de tecnologias como SOAP e XML-RPC.

É importante ressaltar que o uso de XML é apenas um exemplo. Poderiam ser utilizados JSON, YAML ou qualquer outra representação. Esse nível de maturidade trata de uma única representação.

Para ilustrar esse tipo de API, Fowler usa um exemplo de consultas médicas. Por exemplo, para agendar uma consulta:

```txt
POST /agendamentoService HTTP/1.1
```
<!-- separador -->

```xml
<horariosDisponiveisRequest data="2010-01-04" doutor="huberman"/>
```

O retorno seria algo como:

```txt
HTTP/1.1 200 OK
```
<!-- separador -->

```xml
<listaDeHorariosDisponiveis>
  <horario inicio="14:00" fim="14:50">
    <doutor id="huberman"/>
  </horario>
  <horario inicio="16:00" fim="16:50">
    <doutor id="huberman"/>
  </horario>
</listaDeHorariosDisponiveis>
```

Para marcar uma consulta:

```txt
POST /agendamentoService HTTP/1.1
```
<!-- separador -->

```xml
<agendamentoConsultaRequest>
  <horario doutor="huberman" inicio="14:00" fim="14:50"/>
  <paciente id="alexandre"/>
</agendamentoConsultaRequest>
```

A resposta de sucesso, com a confirmação da consulta, seria algo como:

```txt
HTTP/1.1 200 OK
```
<!-- separador -->

```xml
<consulta>
  <horario doutor="huberman" inicio="14:00" fim="14:50"/>
  <paciente id="alexandre"/>
</consulta>
```

Em caso de erro, teríamos uma resposta ainda com o código de status `200`, mas com os detalhes do erro no corpo do response:

```txt
HTTP/1.1 200 OK
```
<!-- separador -->

```xml
<agendamentoConsultaRequestFailure>
  <horario doutor="huberman" inicio="14:00" fim="14:50"/>
  <paciente id="alexandre"/>
  <motivo>Horário não disponível</motivo>
</agendamentoConsultaRequestFailure>
```

Perceba que, no exemplo de Fowler, não foi utilizado SOAP mas uma API que usa HTTP, só que sem os conceitos do protocolo.

### Nível 1 - Recursos

Um primeiro passo, o Nível 1 de Maturidade, é ter diferentes recursos, cada um com sua URL, ao invés de um único endpoint para toda a API.

No exemplo de consultas médicas de Fowler, poderíamos ter um recurso específico para um doutor:

```txt
POST /doutores/huberman HTTP/1.1
```
<!-- separador -->

```xml
<horariosDisponiveisRequest data="2010-01-04"/>
```

O response seria semelhante ao anterior, mas com uma maneira de endereçar individualmente cada horário disponível para um doutor específico:

```txt
HTTP/1.1 200 OK
```
<!-- separador -->

```xml
<listaDeHorariosDisponiveis>
  <horario id="1234" doutor="huberman" inicio="14:00" fim="14:50"/>
  <horario id="5678" doutor="huberman" inicio="16:00" fim="16:50"/>
</listaDeHorariosDisponiveis>
```

Com um endereço para cada horário, marcar uma consulta seria fazer um `POST` para um recurso específico:

```txt
POST /horarios/1234 HTTP/1.1
```
<!-- separador -->

```xml
<agendamentoConsulta>
  <paciente id="alexandre"/>
</agendamentoConsulta>
```

O response seria semelhante ao anterior:

```txt
HTTP/1.1 200 OK
```
<!-- separador -->

```xml
<consulta>
  <horario id="1234" doutor="huberman" inicio="14:00" fim="14:50"/>
  <paciente id="alexandre"/>
</consulta>
```

### Nível 2 - Verbos HTTP

O segundo passo, o Nível 2 de Maturidade, é utilizar os verbos (ou métodos) HTTP o mais perto o possível de seu intuito original.

Para obter a lista de horários disponíveis, poderíamos usar um `GET`:

```txt
GET /doutores/huberman/horarios?data=2010-01-04&status=disponivel HTTP/1.1
```

A resposta seria a mesma de antes:

```txt
HTTP/1.1 200 OK
```
<!-- separador -->

```xml
<listaDeHorariosDisponiveis>
  <horario id="1234" doutor="huberman" inicio="14:00" fim="14:50"/>
  <horario id="5678" doutor="huberman" inicio="16:00" fim="16:50"/>
</listaDeHorariosDisponiveis>
```

No Nível 2 de Maturidade, Fowler diz que o uso de `GET` para consultas é crucial. Como o HTTP define o `GET` como uma operação _safe_, isso significa que não há mudanças significativas no estado de nenhum dos dados. Assim, é seguro invocar um `GET` diversas vezes seguidas e obter os mesmos resultados. Uma consequência importante é que é possível fazer _cache_ dos resultados, melhorando a Performance.

Para marcar uma consulta, é necessário um verbo HTTP que permite a mudança de estado. Poderíamos usar um `POST`, da mesma maneira anterior:

```txt
POST /horarios/1234 HTTP/1.1
```
<!-- separador -->

```xml
<agendamentoConsulta>
  <paciente id="alexandre"/>
</agendamentoConsulta>
```

Uma API de Nível 2 de Maturidade, deve usar os códigos de status e cabeçalhos a seu favor. Para indicar que uma nova consulta foi criada, com o agendamento do paciente naquele horário podemos usar o status `201 Created`. Esse status deve incluir, no response, um cabeçalho `Location` com a URL do novo recurso. Essa nova URL pode ser usada pelo cliente para obter, com um `GET`, mais detalhes sobre o recurso que acabou de ser criado. Portanto, a resposta teria alguns detalhes diferentes da anterior:

```txt
HTTP/1.1 201 Created
Location: horarios/1234/consulta
```
<!-- separador -->

```xml
<consulta>
  <horario id="1234" doutor="huberman" inicio="14:00" fim="14:50"/>
  <paciente id="alexandre"/>
</consulta>
```

No caso de um erro, devem ser usados códigos 4XX ou 5XX. Por exemplo, para indicar que houve uma atualização do recurso por outro cliente, pode ser usado o status `409 Conflict` com uma nova lista de horários disponíveis no corpo do response:

```txt
HTTP/1.1 409 Conflict
```

<!-- separador -->

```xml
<listaDeHorariosDisponiveis>
  <horario id="5678" doutor="huberman" inicio="16:00" fim="16:50"/>
</listaDeHorariosDisponiveis>
```

### Nível 3 - Controles de Hypermedia

Um dos conceitos importantes do HTTP, o protocolo da Web, é o uso de hypertext.

O Nível 3, o nível final, de Maturidade de uma API RESTful é atingido quando são utilizados links. Mas falaremos sobre isso mais adiante.

## Cliente REST com RestTemplate do Spring

Precisamos de que o módulo de Restaurante do Monólito avise ao serviço de Distância que novos restaurantes foram aprovados e que houve atualização no cep e/ou tipo de cozinha de restaurantes já cadastrados no BD de Distância.

Para isso, vamos expandir a API RESTful do serviço de Distância para que receba novos restaurantes aprovados e os insira no BD de Distância e atualize dados alterados de restaurantes existentes.

O serviço de Distância será o servidor e o módulo de Restaurante do Monólito terá o código do cliente REST. Para implementarmos esse cliente, usaremos a classe `RestTemplate` do Spring.

![Módulo de Restaurante faz chamada HTTP ao serviço de Distância {w=49}](imagens/05-integracao-sincrona/modulo-de-restaurante-chama-distancia.png)

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

## Exercício: Integrando o módulo de restaurantes ao serviço de distância com RestTemplate

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

Depois de confirmar um pagamento, o status do pedido ainda permanece como _REALIZADO_. Precisamos implementar uma maneira do serviço de Pagamentos avisar que um determinado pedido foi pago.

Para isso, vamos acrescentar à API RESTful do módulo de Pedido do Monólito um recurso para notificar o pagamento de um pedido.

O módulo de Pedido será o servidor, enquanto o cliente REST será o serviço de Pagamentos. Faremos a implementação de maneira declarativa com o Feign.

![Serviço de Pagamentos chama módulo de Pedido do Monólito {w=49}](imagens/05-integracao-sincrona/pagamentos-chamando-modulo-de-pedido.png)

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

## Exercício: Integrando o serviço de pagamentos e o módulo de pedidos com Feign

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

## O poder dos Links

Ao descrevermos as boas ideias do procolo HTTP, mencionamos a importância dos links. Quando falamos sobre REST e sobre o Nível 3, o máximo, do Modelo de Maturidade de Leonard Richardson, falamos mais uma vez de links.

Roy Fielding, o criador do termo REST, diz no post [REST APIs must be hypertext-driven](https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven) (FIELDING, 2008) que toda API, para ser considerada RESTful, deve necessariamente usar links.

### Hypertext? Hypermedia?

HyperText é um conceito tão importante para a Web que está nas iniciais de seu protocolo, o HTTP, e de seu formato de documentos original, o HTML.

O termo hypertext, foi cunhado por Ted Nelson e publicado no artigo [Complex Information Processing](https://elmcip.net/node/7367) (NELSON, 1965) para denotar um material escrito ou pictórico interconectado de maneira tão complexa que não pode ser representado convenientemente em papel. A ideia original é que seria algo que iria além do texto e deveria ser representado em uma tela interativa.

Nos comentários do post [REST APIs must be hypertext-driven](https://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven) (FIELDING, 2008), Fielding dá a sua definição:

_Quando eu digo hypertext, quero dizer a apresentação simultânea de informações e controles, de forma que as informações se tornem o meio pelo qual o usuário (ou autômato) obtém escolhas e seleciona ações._

_A hypermedia é apenas uma expansão sobre o que o texto significa para incluir âncoras temporais em um fluxo de mídia; a maioria dos pesquisadores abandonou a distinção._

_Hypertext não precisa ser HTML num navegador. Máquinas podem seguir links quando entendem o formato de dados e os tipos de relacionamento._

### Monte a sua histório seguindo links

No final da década de 1970 e começo da década de 1980, surgiu a série de livros-jogo "Escolha a sua aventura" (em inglês, _Choose Your Own Adventure_), lançada pela editora americana Bantam Books e editada pela Ediouro no Brasil.

Títulos como "A Gruta do Tempo" (em inglês, _The Cave of Time_), de Edward Packard, permitiam que o leitor fizesse escolhas e determinasse o rumo da história.

Na página 1 do livro, a história era contada linearmente, até que o personagem chegava a um ponto de decisão, em que eram oferecidas algumas opções para o leitor determinar o rumo da narrativa. Por exemplo:

- Se quiser seguir o velho camponês para ver aonde ele vai, vá para a página 3.
- Se preferir voltar para casa, vá para a página 15.
- Se quiser sentar e esperar, vá para a página 71.

O leitor poderia coletar itens no decorrer da história que seriam determinadas em trechos posteriores. Por exemplo:

- Se você tiver o item mágico "Olho do Falcão", vá para a página 210.
- Se você não tiver o item, mas possui a perícia "Rastreamento", vá para a página 19.
- Caso contrário, vá para a página 101.

Essa ideia de montar a sua história seguindo links parece muito com o uso de hypertext para APIs RESTful.

Em sua palestra [Getting Things Done with REST](https://www.infoq.com/presentations/Getting-Things-Done-with-REST) (ROBINSON, 2011), Ian Robinson cita talvez o mais famoso desses livros-jogo, o de título "O Feiticeiro da Montanha de Fogo" (em inglês, _The Warlock of Firetop Mountain_), escrito por Ian Livingstone e Steve Jackson e publicado em 1982 pela Puffin Books.

### Links para a transição de estados

Algumas entidades de negócio mudam de estados no decorrer do seu uso em uma aplicação.

No Caelum, um restaurante é cadastrado e, para entrar nos resultados de buscas feitas pelos usuário, precisa ser aprovado pelo setor Administrativo. Podemos dizer que um restaurante começa com o estado _CADASTRADO_ e então pode passar para o estado de _APROVADO_.

![A transição de estados de um restaurante {w=28}](imagens/05-integracao-sincrona/maquina-de-estados-de-restaurantes.png)

Para o restaurante, não há nenhum que indica os estados possíveis. Há apenas um atributo `aprovado` na classe `Restaurante` do módulo de Restaurante do Monólito.

Já para um pedido, há mais estados possíveis. Os valores desses estados estão representados na enum `Status` da classe `Pedido` do módulo de Pedido do Monólito.

![A transição de estados de um pedido {w=80}](imagens/05-integracao-sincrona/maquina-de-estados-de-pedidos.png)

Um pagamento tem uma transição mais interessante: depois de _CRIADO_, pode ser _CONFIRMADO_ ou _CANCELADO_.

![A transição de estados de um pagamento {w=26}](imagens/05-integracao-sincrona/maquina-de-estados-de-pagamentos.png)

Se observarmos a implementação da API de pagamentos do `eats-pagamento-service`, 

- um `POST` em `/pagamentos` adiciona um novo pagamento com o estado `CRIADO` e retorna um código `201` com a URL do novo recurso no cabeçalho `Location`. Por exemplo, `/pagamentos/15`
- um `PUT` em `/pagamentos/15` confirma o pagamento de `id` 15, fazendo sua transição para o estado `CONFIRMADO`
- um `DELETE` em `/pagamentos/15` cancela o pagamento de `id` 15, deixando-o no estado `CANCELADO`

Ao invocar a URL de um pagamento com diferentes métodos HTTP, fazemos a transição dos estados de um pagamento.

Para um cliente HTTP fazer essa transição de estados, seu programador deve saber previamente quais os estados possíveis e quais URLs devem ser chamadas. No caso de uma mudança de URL, o programador teria que corrigir o código.

Poderíamos tornar o cliente HTTP mais flexível se representássemos as transições de estados possíveis por meio de links. Essa ideia é comumente chamada de **Hypermedia As The Engine Of Application State (HATEOAS)**.

Para a transição de estados de um pagamento, poderíamos ter um link de confirmação e um link de cancelamento.

Ainda teríamos que saber a utilidade de cada link. Para isso, temos o _link relation_, uma descrição definida em um atributo `rel` associado ao link.

Há um [padrão de link relations](https://www.iana.org/assignments/link-relations/link-relations.xhtml) mantido pela IANA. Alguns deles:

- `self`: um link para o próprio recurso
- `search`: um link para o um recurso de busca.
- `next`: um link para o próximo recurso de uma série. Comumente usado em paginação.
- `previous`: um link para o recurso anterior de uma série. Comumente usado em paginação.
- `first`: um link para o primeiro recurso de uma série. Comumente usado em paginação.
- `last`:um link para o último recurso de uma série. Comumente usado em paginação.

Podemos criar os nossos próprios link relations. Podemos associar o link de confirmação ao link relation `confirma` e o de cancelamento ao `cancela`.

### Representando Links

Como representar esse links? 

Em um XML, podemos usar o elemento `<link>`, que é usado em um HTML para incluir um CSS em uma página. Por exemplo, para o pagamento:

```xml
<pagamento>
  <id>1</id>
  <valor>51.8</valor>
  <nome>ANDERSON DA SILVA</nome>
  <numero>1111 2222 3333 4444</numero>
  <expiracao>2022-07</expiracao>
  <codigo>123</codigo>
  <status>CRIADO</status>
  <formaDePagamentoId>2</formaDePagamentoId>
  <pedidoId>1</pedidoId>
  <link rel="self" href="http://localhost:8081/pagamentos/1" />
  <link rel="confirma" href="http://localhost:8081/pagamentos/1" />
  <link rel="cancela" href="http://localhost:8081/pagamentos/1" />
</pagamento>
```

E para JSON? Poderíamos criar a nossa própria representação de links, mas já existe o HAL, ou _JSON Hypertext Application Language_, descrita por Mike Kelly na especificação preliminar (Internet-Draft) [draft-kelly-json-hal-00](https://tools.ietf.org/html/draft-kelly-json-hal-00) (KELLY, 2012) da IETF. Apesar do status preliminar, a especificação é usada em diferentes tecnologias e frameworks. O media type associado ao HAL é `application/hal+json`. Em teoria, seria possível definir um HAL expressado numa representação XML.

No HAL, é adicionado ao JSON da aplicação uma propriedade `_links` que é um objeto contendo uma propriedade para cada link relation. Os links relations são definidos como objetos cujo link está na propriedade `href`. Por exemplo, para um pagamento:

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

A representação HAL de um recurso pode ter outros recursos embutidos, descritos na propriedade `_embedded`. Por exemplo, um pedido poderia ter uma lista de itens embutida, com representações e links para cada item.

Na [RFC 5988](https://tools.ietf.org/html/rfc5988) (NOTTINGHAM, 2010) da IETF é definido um cabeçalho `Link` e uma série de link relations padronizados. A paginação da API do GitHub segue esse padrão:

```txt
Link: <https://api.github.com/repositories/237159/pulls?page=2>; rel="next", <https://api.github.com/repositories/237159/pulls?page=2>; rel="last"
```

A classe [Link](https://docs.oracle.com/javaee/7/api/javax/ws/rs/core/Link.html), disponível a partir da especificação JAX-RS 2.0 do Java EE 7, usa o cabeçalho `Link` como formato de hypermedia.

### Revisitando o Modelo de Maturidade de Richardson

Quando falamos sobre o Nível 3, o máximo, do Modelo de Maturidade de Leonard Richardson descrito por Martin Fowler no post [Richardson Maturity Model](https://martinfowler.com/articles/richardsonMaturityModel.html) (FOWLER, 2010), apenas mencionamos que links são importantes.

Agora, sabemos que links podem ser usados para descrever a transição de estados da aplicação, o que chamamos de HATEOAS.

Voltando ao exemplo de consultas médicas, cada horário da lista de horários disponíveis pode conter um link de agendamento:

```txt
GET /doutores/huberman/horarios?data=2010-01-04&status=disponivel HTTP/1.1
```

A resposta seria a mesma de antes:

```txt
HTTP/1.1 200 OK
```
<!-- separador -->

```xml
<listaDeHorariosDisponiveis>
  <horario id="1234" doutor="huberman" inicio="14:00" fim="14:50">
    <link rel="agendamento" href="/horarios/1234">
  </horario>
  <horario id="5678" doutor="huberman" inicio="16:00" fim="16:50">
    <link rel="agendamento" href="/horarios/5678">
  </horario>
</listaDeHorariosDisponiveis>
```

Os links descrevem o que pode ser feito a seguir e as URLs que precisam ser manipuladas. 

Seguindo o link de agendamento, o `POST` para marcar uma consulta seria feito da mesma maneira anterior:

```txt
POST /horarios/1234 HTTP/1.1
```
<!-- separador -->

```xml
<agendamentoConsulta>
  <paciente id="alexandre"/>
</agendamentoConsulta>
```

O resultado poderia conter links para diferentes possibilidades:

```txt
HTTP/1.1 201 Created
Location: horarios/1234/consulta
```
<!-- separador -->

```xml
<consulta>
  <horario id="1234" doutor="huberman" inicio="14:00" fim="14:50"/>
  <paciente id="alexandre"/>
  <link rel="agendamentoExame" href="/horarios/1234/consulta/exames">
  <link rel="cancelamento" href="/horarios/1234/consulta">
  <link rel="mudancaHorario" href="/doutor/huberman/horarios?data=2010-01-04&status=disponivel">
</consulta>
```

Assim, as URLs podem ser modificadas sem que o código dos clientes quebre. Um benefício adicional é que os clientes podem conhecer e explorar os próximos passos.

> Idealmente, um cliente deveria depender apenas da URL raiz de uma API e, a partir dela, navegar pelos links, descobrindo o que pode ser feito com a API.

Fowler menciona a ideia de Ian Robinson de que o Modelo de Maturidade de Richardson está relacionado com técnicas comuns de design: 

- O Nível 1 trata de como lidar com a complexidade usando "dividir e conquistar", dividindo um grande endpoint para o serviço todo em vários recursos. 
- O Nível 2 adiciona os métodos HTTP como um padrão, de maneira que possamos lidar com situações semelhantes da mesma maneira, evitando variações desnecessárias.
- O Nível 3 introduz a capacidade de descoberta (em inglês, _Discoverability_), fornecendo uma maneira de tornar o protocolo auto-documentado.

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

## HATEOAS e Métodos HTTP

Observe o JSON com a representação de um pagamento retornado pelo serviço de Pagamentos:

```json
  {
    "id":1,
    "valor":51.80,
    ...
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

Há links, cada um com seu link relation distinto: `self`, `confirma` e `cancela`.

Mas um detalhe importante é que todos os links são iguais! Tanto para confirmar como para cancelar o pagamento do JSON anterior, o link é: http://localhost:8081/pagamentos/1

Além de saber sobre o significado de cada link relation (em outros termos, sua semântica), um cliente dessa API deve saber qual método HTTP utilizar para efetuar a confirmação ou cancelamento do pagamento.

Essa necessidade de um conhecimento prévio do cliente sobre o método HTTP a ser utilizado diminui a Discoverability da API.

Há [bastante](https://stackoverflow.com/questions/25513781/adding-http-method-to-spring-hateoas-links) [discussão](https://stackoverflow.com/questions/19959284/where-in-a-hateoas-architecture-do-you-specify-the-http-verbs) sobre o fato de se o método HTTP deve ser associado a um link relation.

Uma visão mais purista diria que não devemos associar link relations a métodos HTTP.

Poderíamos utilizar uma chamada `OPTIONS` na URL do `href` do link relation e descobrir pelo cabeçalho `Allow` quais os métodos permitidos. Isso levaria a mais uma chamada pela rede entre o cliente e o servidor, impactando negativamente a Performance da aplicação.

Uma outra ideia é que poderíamos usar sempre o método `PUT`, passando o `status` desejado (_CONFIRMADO_ ou _CANCELADO_) no corpo da requisição.

Uma visão mais pragmática é usada na [API do PayPal](https://developer.paypal.com/docs/api/payments/v2/), em que um atribudo `method` é associado ao link relation:

```json
{
  "id": "8AA831015G517922L",
  "status": "CREATED",
  "links": [
    {
      "rel": "self",
      "method": "GET",
      "href": "https://api.paypal.com/v2/payments/authorizations/8AA831015G517922L"
    },
    {
      "rel": "capture",
      "method": "POST",
      "href": "https://api.paypal.com/v2/payments/authorizations/8AA831015G517922L/capture"
    },
    {
      "rel": "void",
      "method": "POST",
      "href": "https://api.paypal.com/v2/payments/authorizations/8AA831015G517922L/void"
    },
    {
      "rel": "reauthorize",
      "method": "POST",
      "href": "https://api.paypal.com/v2/payments/authorizations/8AA831015G517922L/reauthorize"
    }
  ]
}
```

_Note que a API do PayPal não usa HAL: os links ficam no atributo `links`, e não `_links`, que é um array, e não um objeto._

Podemos nos inspirar na API do PayPal e adicionar um atributo `method` em cada link.

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

## Para saber mais: HAL-FORMS

[HAL-FORMS](https://rwcbook.github.io/hal-forms/) (AMUNDSEN, 2016) é uma extensão do HAL especificada por Mike Amundsen que adiciona um atributo `_templates`, permitindo atribuir um método HTTP e outras propriedades a um link. O media type proposto é `application/prs.hal-forms+json`.

Um exemplo de response HAL-FORMS com os dados de uma pessoa:

```json
{
  "id" : 1,
  "firstName" : "Frodo",
  "lastName" : "Baggins",
  "role" : "ring bearer",
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/employees/1"
    },
    "employees" : {
      "href" : "http://localhost:8080/employees"
    }
  },
  "_templates" : {
    "default" : {
      "title" : null,
      "method" : "put",
      "contentType" : "",
      "properties" : [ {
        "name" : "firstName",
        "required" : true
      }, {
        "name" : "id",
        "required" : true
      }, {
        "name" : "lastName",
        "required" : true
      }, {
        "name" : "role",
        "required" : true
      } ]
    },
    "deleteEmployee" : {
      "title" : null,
      "method" : "delete",
      "contentType" : "",
      "properties" : [ ]
    }
  }
}
```

O template `default` do HAL-FORMS presume que o recurso será editado por meio de um `PUT` na URL do link relation `self` e as propriedades associadas: `id`, `firstName`, `lastName` e `role`, todas obrigatórias nesse exemplo. Os dados desse template pode ser usados, por exemplo, para construir um `<form>` no front-end.

Já o template `deleteEmployee` tem associado o método `DELETE`, sem nenhuma propriedade.

O Spring HATEOAS na versão `1.0.0.RELEASE` contém suporte a HAL-FORMS, que pode ser habilitado com a anotação `@EnableHypermediaSupport(type = HypermediaType.HAL_FORMS)`.

Além disso, há suporte a [Uniform Basis for Exchanging Representations](https://rawgit.com/uber-hypermedia/specification/master/uber-hypermedia.html) (UBER), [Collection+JSON](http://amundsen.com/media-types/collection/) e [Application-Level Profile Semantics](https://tools.ietf.org/html/draft-amundsen-richardson-foster-alps-01) (ALPS), todos trabalhos experimentais de Mike Amundsen focados em hypermedia e que compõe a Affordance API do Spring HATEOAS.



<!--@note

Usei o HAL-FORMS na versão milestone 2.2.0.M2 do Spring Boot no commit abaixo:

https://github.com/alexandreaquiles/eats/commit/f8ef33b88cd3d96c62627a13b4e8470c9f09ada0#diff-5f414af558500eda821060272d84b8d8

-->

## Para saber mais: Spring Data REST

O Spring Data REST parte do Spring Data para expor entidades e repositórios como recursos REST, utilizando hypermedia  e HAL como representação. Os mecanismos de persistência suportados são BDs relacionais com JPA, MongoDB, Neo4j e Gemfire.

Para utilizá-lo, basta incluir o starter no `pom.xml` da aplicação:

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-rest</artifactId>
</dependency>
```

Se utilizar MySQL como BD, também precisamos incluir o Spring Data JPA e o driver do MySQL. Além disso, são necessárias as configurações de data sources no `application.properties`.

Vamos definir, como exemplo, uma entidade `Pessoa`:

```java
@Entity
@Data
public class Pessoa {

  @Id @GeneratedValue(strategy=GenerationType.IDENTITY)
  private Long id;
  private String nome;
  private String sobrenome;

}
```

O `@Data` do Lombok provê getters e setters para cada atributo, além de implementações para `equals`, `hashcode` e `toString`.

Também deve ser definido um repository:

```java
public interface PessoaRepository extends JpaRepository<Pessoa, Long> {

  List<Pessoa> findBySobrenome(@Param("sobrenome") String sobrenome);

}
```

E pronto! Ao subirmos a aplicação já temos uma API RESTful com hypermedia!

A raiz da API provê links para as entidades expostas:

```txt
GET http://localhost:8080
```

A resposta é:

```
200 OK
Content-Type: application/hal+json
```

<!-- separador -->

```json
{
  "_links" : {
    "pessoas" : {
      "href" : "http://localhost:8080/pessoas{?page,size,sort}",
      "templated" : true
    },
    "profile" : {
      "href" : "http://localhost:8080/profile"
    }
  }
}
```

Note que o response é um HAL com o link relation `pessoas`.

O link relation `profile` está associada a especificação ALPS, que provê uma maneira de descrever os links, classificando-os em safe, idempotente, entre outros.

O recurso `pessoas` já é paginado, já que um `JpaRepository` extende a interface `PagingAndSortingRepository` do Spring Data Core.

Podemos criar uma nova pessoa fazendo o seguinte request ao recurso `pessoas`:

```txt
POST http://localhost:8080/pessoas
Content-Type: application/json
```

Com o payload:

```json
{
  "nome": "Alexandre",
  "sobrenome": "Aquiles"
}
```

Como response, teremos:

```txt
201 Created
Content-Type: application/json
Location: http://localhost:8080/pessoas/3
```

<!-- separador -->

```json
{
  "nome": "Alexandre",
  "sobrenome": "Aquiles",
  "_links": {
    "self": {
      "href": "http://localhost:8080/pessoas/3"
    },
    "pessoa": {
      "href": "http://localhost:8080/pessoas/3"
    }
  }
}
```

Perceba que é retornado o status `201` com a URL do novo recurso no cabeçalho `Location`.

No corpo do response, são retornados os dados do recurso criado junto aos links.

Se dispararmos um `GET` ao `href` do link relation `self`, a URL `http://localhost:8080/pessoas/3`, teremos um payload semelhante ao anterior.

Podemos editar um recurso com um `PUT`, que sobreescreeve os dados do recurso com a representação passada no request. Os atributos omitidos ficarão como nulos.

Se quisermos passar apenas um subconjunto dos dados, podemos usar um `PATCH`.

Para remover um recurso, podemos usar um `DELETE`.

Para listarmos todas as pessoas, podemos consultar o recurso do link relation `pessoas` da raiz da API:

```txt
GET http://localhost:8080/pessoas
```

O response será paginado, com

```json
{
  "_embedded" : {
    "pessoas" : [ {
      "nome" : "Anderson",
      "sobrenome" : "da Silva",
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/pessoas/2"
        },
        "pessoa" : {
          "href" : "http://localhost:8080/pessoas/2"
        }
      }
    }, {
      "nome" : "Alexandre",
      "sobrenome" : "Aquiles",
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/pessoas/3"
        },
        "pessoa" : {
          "href" : "http://localhost:8080/pessoas/3"
        }
      }
    } ]
  },
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/pessoas{?page,size,sort}",
      "templated" : true
    },
    "profile" : {
      "href" : "http://localhost:8080/profile/pessoas"
    },
    "search" : {
      "href" : "http://localhost:8080/pessoas/search"
    }
  },
  "page" : {
    "size" : 20,
    "totalElements" : 2,
    "totalPages" : 1,
    "number" : 0
  }
}
```

Os itens da lista ficam no atributo `_embedded`. Cada item contém seus dados e links.

Há dados de paginação: o `size` indica o tamanho máximo de elementos de uma página, 20 é o padrão mas pode ser alterado com o parâmetro `size`; `number` indica o número da página atual; `totalPages`, o número de páginas; e `totalElements`, o número total de elementos cadastrados.

Há os links da própria lista. Se houver mais de uma página, teremos os link relations `next` e `last`.

Ao seguirmos o link relation `search`, são exibidas as consultas possíveis:

```txt
GET http://localhost:8080/pessoas/search
```

<!-- separador -->

```json
{
  "_links" : {
    "findBySobrenome" : {
      "href" : "http://localhost:8080/pessoas/search/findBySobrenome{?sobrenome}",
      "templated" : true
    },
    "self" : {
      "href" : "http://localhost:8080/pessoas/search"
    }
  }
}
```

Podemos usar o link relation `findBySobrenome`, para buscar todas as pessoas com sobrenome `Aquiles`:

```txt
GET http://localhost:8080/pessoas/search/findBySobrenome?sobrenome=Aquiles
```

Obteremos no response:

```json
{
  "_embedded" : {
    "pessoas" : [ {
      "nome" : "Alexandre",
      "sobrenome" : "Aquiles",
      "_links" : {
        "self" : {
          "href" : "http://localhost:8080/pessoas/3"
        },
        "pessoa" : {
          "href" : "http://localhost:8080/pessoas/3"
        }
      }
    } ]
  },
  "_links" : {
    "self" : {
      "href" : "http://localhost:8080/pessoas/search/findBySobrenome?sobrenome=Aquiles"
    }
  }
}
```

## Formatos e Protocolos Binários

Na palestra [PB vs. Thrift vs. Avro](https://pt.slideshare.net/IgorAnishchenko/pb-vs-thrift-vs-avro) (ANISHCHENKO, 2012), Igor Anishchenko demonstra que o protocolo HTTP/1.1 com representações como XML e JSON pode ser ineficiente se comparado com alternativas binárias como:

- RMI
- Apache Thrift, usado no Facebook e em projetos com o Hadoop
- Protocol Buffers, usado na Google

Anishchenko, ao serializar um objeto Curso com 5 objetos Pessoa e um Telefone associados, mostra o tamanho:

- Protocol Buffers: 250
- Thrift TCompactProcotol: 278
- Thrift TBinaryProtocol: 460
- HTTP/JSON: 559
- HTTP/XML: 836
- RMI: 905

O protocolo Thrift TBinaryProtocol é otimizado em termos de processamento e o Thrift TCompactProcotol, em tamanho.

É interessante notar que a serialização RMI foi a menos eficiente em termos de tamanho. Não é um formato otimizado.

Os formatos binários Thrift TCompactProcotol e Protocol Buffers apresentam um tamanho de cerca de metade do HTTP/JSON para esse caso simples.

Então, Anishchenko compara 10000 chamadas de uma busca por uma listagem de códigos de Curso e, em seguida, os dados do Curso associado a esse código. São avaliados o tempo de resposta, porcentagem de uso de CPU no servidor e no cliente.

Os resultados para o tempo de resposta:

- Thrift TCompactProcotol: 01 min 05 s
- Thrift TBinaryProtocol: 01 min 13 s
- Protocol Buffers: 01 min 19 s
- RMI: 02 min 14 s
- HTTP/JSON: 04 min 44 s
- HTTP/XML: 05 min 27 s

Os resultados para porcentagem de uso de CPU do servidor:

- Thrift TBinaryProtocol: 33 %
- Thrift TCompactProcotol: 30 %
- Protocol Buffers: 30 %
- HTTP/JSON: 20 %
- RMI: 16 %
- HTTP/XML: 12 %

Os resultados para CPU do cliente:

- Thrift TCompactProcotol: 22.5 %
- Thrift TBinaryProtocol: 21 %
- Protocol Buffers: 37.75 %
- RMI: 46.5 %
- HTTP/JSON: 75 %
- HTTP/XML: 80.75 %

Pelos dados de Anishchenko, Thrift toma mais processamento do servidor, suavizando o processamento no cliente.

Outra alternativa mencionada por Anishchenko é o Apache Avro, usado pelo Apache Kafka, entre outros projetos.

Em alguns cenários de aplicação Mobile, um formato de serialização mais compacto e com menos processamento no cliente pode ser interessante já que há limitações de CPU, bateria e banda de rede.

Nada impede que Protocol Buffers ou Apache Thrift sejam usados apenas como formato de serialização de dados, usando um transporte HTTP. Mas o protocolo HTTP/1.1 em si é ineficiente por ser baseado em texto com diversos cabeçalhos a cada request e response.

O protocolo HTTP/2 aproveita melhor as conexões TCP sobre as quais é construído, codificando e comprimindo os dados em _frames_ binários. Há ainda a possibilidade de _streams_ múltiplas, iniciadas pelo cliente ou pelo servidor.

<!--

## gRPC

CORBA

Stubby -> RPC

proto buffers
-->

## Para saber mais: Todo o poder emana do cliente - explorando uma API GraphQL

> O texto dessa seção é baseado no post [Todo o poder emana do cliente: explorando uma API GraphQL](https://blog.caelum.com.br/todo-o-poder-emana-do-cliente-explorando-uma-api-graphql) (AQUILES, 2017) do blog da Caelum, disponível em: https://blog.caelum.com.br/todo-o-poder-emana-do-cliente-explorando-uma-api-graphql

### Quais as limitações de uma API REST?

Para ilustrar o que pode ser melhorado em uma API REST, vamos utilizar a [versão 3](https://developer.github.com/v3/) da API do GitHub, considerada muito consistente e aderente aos princípios REST.

Queremos uma maneira de avaliar bibliotecas open-source. Para isso, dado um repositório do GitHub, desejamos descobrir:

- o número de stars
- o número de pull requests abertos

Como exemplo, vamos usar o repositório de uma biblioteca NodeJS muito usada: o framework Web minimalista Express.

### Obtendo detalhes de um repositório

Lendo a documentação da API do GitHub, descobrimos que para [obter detalhes sobre um repositório](https://developer.github.com/v3/repos/#get), devemos enviar uma requisição GET para `/repos/:owner/:repo`. Então, para o repositório do Express, devemos fazer:

```txt
GET https://api.github.com/repos/expressjs/express
```

Como resposta, obtemos:

- 2.2 KB _gzipados_ transferidos, incluindo cabeçalhos
- 6.1 KB de JSON em 110 linhas, quando descompactado

```txtx
200 OK
Content-type: application/json; charset=utf-8
```

<!-- separador -->

```json
{
    "id": 237159,
    "name": "express",
    "full_name": "expressjs/express",
    "private": false,
    "html_url": "https://github.com/expressjs/express",
    "description": "Fast, unopinionated, minimalist web framework for node.",
    "fork": false,
    "issues_url": "https://api.github.com/repos/expressjs/express/issues{/number}",
    "pulls_url": "https://api.github.com/repos/expressjs/express/pulls{/number}",
    "stargazers_count": 33508,
    ... 
}
```

O JSON retornado tem diversas informações sobre o repositório do Express. Por meio da propriedade `stargazers_count`, descobrimos que há mais de 33 mil stars.

Porém, **não** temos o número de pull requests abertos.

### Obtendo os pull requests de um repositório

Na propriedade `pulls_url`, temos apenas uma URL: https://api.github.com/repos/expressjs/express/pulls{/number}.

Um bom palpite é que sem esse `{/number}` teremos a lista de todos os pull requests, o que pode ser confirmado na [seção de pull requests](https://developer.github.com/v3/pulls/#list-pull-requests) da documentação da API REST do GitHub.

> O `{/number}` da URL segue o modelo proposto pela [RFC 6570](https://tools.ietf.org/html/rfc6570) (URI Template).

Mas como filtrar apenas pelos pull requests abertos?

Na mesma documentação, verificamos que podemos usar a URL `/repos/:owner/:repo/pulls?state=open` ou simplesmente `/repos/:owner/:repo/pulls`, já que o filtro por pull requests abertos é aplicado por padrão. Em outras palavras, precisamos de outra requisição:

```txt
GET https://api.github.com/repos/expressjs/express/pulls
```

A resposta é:

- 54.1 KB _gzipados_ transferidos, incluindo cabeçalhos
- 514 KB de JSON em 9150 linhas, quando descompactado

```txt
200 OK
Content-type: application/json; charset=utf-8
Link: <https://api.github.com/repositories/237159/pulls?page=2>; rel="next",
      <https://api.github.com/repositories/237159/pulls?page=2>; rel="last"
```

<!-- separador -->

```json
[
    {
        //um pull request...
        "url": "https://api.github.com/repos/expressjs/express/pulls/3391",
        "id": 134639441,
        "html_url": "https://github.com/expressjs/express/pull/3391",
        "diff_url": "https://github.com/expressjs/express/pull/3391.diff",
        "patch_url": "https://github.com/expressjs/express/pull/3391.patch",
        "issue_url": "https://api.github.com/repos/expressjs/express/issues/3391",
        "number": 3391,
        "state": "open",
        "locked": false,
        "title": "Update guide to ES6",
        "user": {
            "login": "jevtovich",
            "id": 13847095,
            "avatar_url": "https://avatars3.githubusercontent.com/u/13847095?v=4",
            ...
        },
        "body": "",
        "created_at": "2017-08-08T11:40:32Z",
        "updated_at": "2017-08-08T17:28:01Z",
        ...
    },
    {
        //outro pull request...
        "url": "https://api.github.com/repos/expressjs/express/pulls/3390",
        "id": 134634529,
        ...
    },
    ...
]
```

É retornado um array de 30 objetos que representam os pull requests. Cada objeto ocupa uma média de 300 linhas, com informações sobre status, descrição, autores, commits e _diversas_ URLs relacionadas.

Disso tudo, só queremos saber a contagem: 30 pull requests. Não precisamos de **nenhuma** outra informação.

Mas há outra questão: o resultado é paginado com 30 resultados por página, por padrão, conforme descrito na [seção de paginação](https://developer.github.com/v3/#pagination) da documentação da API REST do GitHub.

As URLs das próximas páginas devem ser obtidas a partir do cabeçalho de resposta `Link`, extraindo o `rel` (_link relation_) `next`.

> Os links para as próximas páginas seguem o conceito de hipermídia do REST e foram implementados usando o cabeçalho Link e o formato descrito na [RFC 5988](https://tools.ietf.org/html/rfc5988) (Web Linking). Essa RFC sugere um punhado de link relations padronizados.

Então, a partir do `next`, seguimos para a próxima página:

```txt
GET https://api.github.com/repositories/237159/pulls?page=2
```

Temos como resposta:

- 26.9 KB _gzipados_ transferidos, incluindo cabeçalhos
- 248 KB de JSON em 4394 linhas, quando descompactado

```txt
200 OK
Content-type: application/json; charset=utf-8
Link: <https://api.github.com/repositories/237159/pulls?page=1>; rel="first",
      <https://api.github.com/repositories/237159/pulls?page=1>; rel="prev"
```

<!-- separador -->

```json
[
  {
    //um pull request...
    "url": "https://api.github.com/repos/expressjs/express/pulls/2730",
    "id": 41965836,
    ...
  },
  {
    //outro pull request...
    "url": "https://api.github.com/repos/expressjs/express/pulls/2703",
    "id": 39735937,
     ...
  },
  ...
]
```

O array retornado contabiliza mais 14 objetos representando os pull requests. Dessa vez, não há o link relation next, indicando que é a última página.

Então, sabemos que há 44 (30 + 14) pull requests abertos no repositório do Express.

### Resumindo a consulta REST

No momento da escrita desse artigo, o número de stars do Express no GitHub é 33508 e o de pull requests abertos é 44. Para descobrir isso, tivemos que:

- disparar 3 requisições ao servidor
-  baixar 83.2 KB de informações gzipadas e cabeçalhos
- fazer parse de 768.1 KB de JSON ou 13654 linhas
O
 que daria pra melhorar? Ir menos vezes ao servidor, baixando menos dados!

Não é um problema com o REST em si, mas uma discrepância entre a modelagem atual da API e as nossas necessidades.

Poderíamos pedir para o GitHub implementar um recurso específico que retornasse somente as informações, tudo em apenas um request.

Mas será que o pessoal do GitHub vai nos atender?

### Mais flexibilidade e eficiência com GraphQL

Numa API GraphQL, o cliente diz exatamente os dados que quer da API, tornando a requisição muito **flexível**.

A API, por sua vez, retorna apenas os dados que o cliente pediu, fazendo com que a transferência da resposta seja bastante **eficiente**.

Mas afinal de contas, o que é GraphQL?

GraphQL _não_ é um banco de dados, _não_ é um substituto do SQL, _não_ é uma ferramenta do lado do servidor e _não_ é específico para React (apesar de muito usado por essa comunidade).

Um servidor que aceita requisições GraphQL poderia ser implementado em _qualquer_ linguagem usando qualquer banco de dados. Há várias [bibliotecas](https://graphql.org/code/#server-libraries) de diferentes plataformas que ajudam a implementar esse servidor.

Clientes que enviam requisições GraphQL também poderiam ser implementados em qualquer tecnologia: web, mobile, desktop, etc. Diversas [bibliotecas](https://graphql.org/code/#graphql-clients) auxiliam nessa tarefa.

GraphQL é uma **query language para APIs** que foi [especificada](https://graphql.github.io/graphql-spec/) pelo Facebook em 2012 para uso interno e aberta ao público em 2015.

A _query language_ do GraphQL é **fortemente tipada** e descreve, através de um _schema_, o modelo de dados oferecido pelo serviço. Esse schema pode ser usado para verificar se uma dada requisição é válida e, caso seja, executar as tarefas no back-end e estruturar os dados da resposta.

Um cliente pode enviar 3 tipos de requisições GraphQL, os _root types_:

- _query_, para consultas;
- _mutation_, para enviar dados;
- _subscription_, para comunicação baseada em eventos.

### Montando uma consulta GraphQL

A [versão 4](https://developer.github.com/v4/) da API do GitHub, a mais recente, dá suporte a requisições GraphQL.

Para fazer nossa consulta às stars e aos pull requests abertos do repositório do Express usando a API GraphQL do GitHub, devemos começar com a query:

```graphql
query {
}
```

Vamos usar o campo `repository` da query, que recebe os argumentos `owner` e `name`, ambos obrigatórios e do tipo String. Para buscar pelo Express, devemos fazer:

```graphql
query {
  repository (owner: "expressjs", name: "express") {
  }
}
```

A partir do objeto `repository`, podemos descobrir o número de stars por meio do campo `stargazers`, que é uma connection do tipo `StargazerConnection`. Como queremos apenas a quantidade de itens, só precisamos obter propriedade `totalCount` dessa connection.

```graphql
query {
  repository (owner: "expressjs", name: "express") {
    stargazers {
      totalCount
    }
  }
}
```

Para encontrarmos o número de pull requests abertos, basta usarmos o campo `pullRequests` do `repository`, uma connection do tipo `PullRequestConnection`. O filtro por pull requests abertos não é aplicado por padrão. Por isso, usaremos o argumento `states`. Da connection, obteremos apenas o `totalCount`.

```graphql
query { 
  repository(owner: "expressjs", name: "express") {
    stargazers {
      totalCount
    }
    pullRequests(states: OPEN) {
      totalCount
    }
  } 
}
```

Basicamente, é essa a nossa consulta! Bacana, não?

Uma maneira de “rascunhar” consultas GraphQL é usar a ferramenta [GraphiQL](https://github.com/graphql/graphiql), que permite explorar APIs pelo navegador. Há até code completion! Boa parte das APIs GraphQL dá suporte, incluindo [a do GitHub](https://developer.github.com/v4/explorer/).

### Tá, mas como enviar a consulta para a API?

A maneira mais comum de publicar APIs GraphQL é usar a boa e velha Web, com seu protocolo HTTP.

> Apesar do HTTP ser o mais usado para publicar APIs GraphQL, teoricamente não há limitações em usar outros protocolos.

Uma API GraphQL possui apenas um _endpoint_ e, consequentemente, só uma URL.

É possível enviar requisições GraphQL usando o método `GET` do HTTP, com a consulta como um parâmetro na URL. Porém, como as consultas são relativamente grandes e requisições `GET` tem um limite de tamanho, o método mais utilizado pelas APIs GraphQL é o `POST`, com a consulta no corpo da requisição.

No caso do GitHub a URL do endpoint GraphQL é: https://api.github.com/graphql

O GitHub só dá suporte ao método `POST` e o corpo da requisição deve ser um JSON cuja propriedade `query` conterá uma String com a nossa consulta.

Mesmo para consultas a repositórios públicos, a API GraphQL do GitHub precisa de um token de autorização.

```txt
POST https://api.github.com/graphql
Content-type: application/json
Authorization: bearer f023615deb415e...
```

<!-- separador -->

```json
{
"query":    "query {
                repository(owner: \"expressjs\", name: \"express\") { 
                    stargazers {
                        totalCount
                    } 
                    pullRequests(states: OPEN) {
                        totalCount
                    }
                }
            }"
}
```

O retorno será um JSON em que os dados estarão na propriedade `data`:

```json
{
    "data": {
        "repository": {
            "stargazers": {
                "totalCount": 33508
            },
            "pullRequests": {
                "totalCount": 44
            }
        }
    }
}
```

> Na verdade, os JSONs de requisição e resposta ficam em apenas 1 linha. Formatamos o código anterior em várias linhas para melhor legibilidade.

Repare que os campos da consulta, dentro da `query`, tem exatamente a mesma estrutura do retorno da API. É como se a resposta fosse a própria consulta, mas com os valores preenchidos. Por isso, montar consultas com GraphQL é razoavelmente intuitivo.

### Resumindo a consulta GraphQL

Obtivemos os mesmos resultados: 33508 stars e 44 pull requests. Para isso, tivemos que:

- disparar apenas 1 requisição ao servidor
- baixar somente 996 bytes de informações _gzipadas_, incluindo cabeçalhos
- fazer parse só de 93 bytes de JSON

São 66,67% requisições a menos, 98,82% menos dados e cabeçalhos trafegados e 99,99% menos JSON a ser “parseado”. Ou seja, **MUITO mais rápido**.

### Considerações finais

O GraphQL dá bastante poder ao cliente. Isso é especialmente útil quando a equipe que implementa o cliente é totalmente separada da que implementa o servidor. Mas há casos mais simples, em que as equipes do cliente e servidor trabalham juntas. Então, não haveria tanta dificuldade em manter uma API RESTful customizada para o cliente.

Poderíamos buscar outros dados da API do GitHub: o número de issues abertas, a data da última release, informações sobre o último commit, etc.

Uma coisa é certa: com uma consulta GraphQL, eu faria menos requisições e receberia menos dados desnecessários. Mais flexibilidade e mais eficiência.

Considerando o Modelo de Maturidade de Richardson, podemos considerar que o GraphQL está no Nível 0:

- não diferentes recursos, apenas uma URI como ponto de entrada para toda a API GraphQL
- não há a ideia de diferentes verbos HTTP, só é usado POST
- não há diferentes representações, apenas um JSON que contém uma estrutura GraphQL

Existem várias outras questões que surgem ao estudar o GraphQL:

- como fazer um servidor que atenda a toda essa flexibilidade?
- é possível gerar uma documentação a partir do código para a minha API?
- vale a pena migrar minha API pra GraphQL?
- posso fazer uma “casca” GraphQL para uma API REST já existente?
- como implementar um cliente sem muito trabalho?
- quais os pontos ruins dessa tecnologia e desafios na implementação?

## Para saber mais: Field Selectors

Um maneira de otimizar uma API REST já existente é implementar um mecanismo de obter um subconjunto das representações de um recurso.

A API do Linkedin, por exemplo, implementa [field projections](https://docs.microsoft.com/en-us/linkedin/shared/api-guide/concepts/projections?context=linkedin/context), que permitem selecionar os campos retornados. Por exemplo:

```txt
GET https://api.linkedin.com/v2/people/id=-f_Ut43FoQ?projection=(id,localizedFirstName,localizedLastName)
```

```json
{
  "id": "-f_Ut43FoQ",
  "localizedFirstName": "Dwight",
  "localizedLastName": "Schrute"
}
```

A Graph API do Facebook é uma API REST (não GraphQL!) que permite que programadores interajam com a plataforma do Facebook para ler ou enviar dados de usuário, páginas, fotos, entre outros. Essa API implementa field expansions uma maneira de retornar [apenas os campos ](https://developers.facebook.com/docs/graph-api/using-graph-api?locale=en_US#field-expansion):

```txt
https://graph.facebook.com/{your-user-id}?fields=birthday,email,hometown&access_token={your-user-access-token}
```

```json
{
  "hometown": "Your, Hometown",
  "birthday": "01/01/1985",
  "email": "your-email@email.addresss.com",
  "id": "{your-user-id}"
}
```

Diversas APIs do Google permitem [partial responses](https://developers.google.com/url-shortener/v1/performance#partial-response), em que apenas os campos necessários são retornados:

```txt
GET https://www.googleapis.com/demo/v1?fields=kind,items(title,characteristics/length)
```

```json
{
  "kind": "demo",
  "items": [{
    "title": "First title",
    "characteristics": {
      "length": "short"
    }
  }, {
    "title": "Second title",
    "characteristics": {
      "length": "long"
    }
  },
  ]
}
```
