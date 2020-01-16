# Resiliência

## Revisitando o conceito de Disponibilidade

Em capítulos anteriores, definimos Disponibilidade como a proporção de tempo, em um determinado período, em que o sistema opera normalmente.

Mas o que acontece se um sistema estiver extremamente lento?

Conforme mencionamos nos primeiros capítulos, ao falar sobre Consistência em um Sistema Distribuído, no paper [Consistency Tradeoffs in Modern Distributed Database System Design](http://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf) (ABADI, 2012), Daniel Abadi amplia o Teorema CAP, cunhando o Teorema PACELC: no caso de uma Partição na rede, um Sistema Distribuído precisa escolher entre Disponibilidade (em inglês, _Availability_) e Consistência;  se não houver Partição na rede, a escolha é entre Latência e Consistência. Simplificando, podemos incluir alta latência de rede como uma forma de indisponibilidade.

> _Observe que Disponibilidade e Latência são sem dúvida a mesma coisa: um sistema indisponível fornece essencialmente uma Latência extremamente alta. Para os fins desta discussão, considero como indisponíveis sistemas com latências maiores que um timeout típico para uma requisição, como alguns segundos; e como "Alta Latência", sistemas com latências menores que um timeout típico, mas ainda se aproximando de centenas de milissegundos._
>
> Daniel Abadi, no paper [Consistency Tradeoffs in Modern Distributed Database System Design](http://www.cs.umd.edu/~abadi/papers/abadi-pacelc.pdf) (ABADI, 2012)

<!--@note
  Alexandre - Podemos lembrar aos alunos dos números de Latência do Jeffrey Dean da Google: Cache L1 é 0.5 ns enquanto round Round trip dentro do mesmo data center é 500 000 ns. Ou seja, uma latência 1 milhão de vezes maior. Podemos lembrar que a Luz, a coisa mais rápida do Universo, demoraria 133 ms pra ir e voltar do Japão.
-->

## Falhas em Cascata

Em um Sistema Distribuído implementado de maneira ingênua, uma lentidão em um serviço pode levar o sistema todo a ficar indisponível. Isso é comumente conhecido como falhas em cascata (em inglês, _cascading failures_).

No livro [Microservices in Action](https://www.manning.com/books/microservices-in-action) (BRUCE; PEREIRA, 2018), os autores ligam esse tipo de comportamento em cascata com um comportamento emergente de Sistemas Complexos: _um evento perturba um Sistema, levando a algum efeito que, por sua vez, aumenta a magnitude do distúrbio inicial. [...] Considere uma debandada em um rebanho de animais: o pânico faz um animal correr que, por sua vez, espalha o pânico para outros animais, o que faz com que eles corram, e assim por diante. Em Microservices, uma sobrecarga pode causar um efeito dominó: uma falha em um serviço aumenta falhas nos serviços que o chamam e, por sua vez, nos serviços que chamam esses. No pior caso, o resultado é uma indisponibilidade generalizada._

No livro [Building Microservices](https://learning.oreilly.com/library/view/building-microservices/9781491950340/) (NEWMAN, 2015), Sam Newman descreve o caso em um sistema de agregador de anúncios, que unia resultados de diferentes sistemas legados. Um dos legados mais antigos e pouco usados, que correspondia a menos de 5% do faturamento, passou a responder de maneira muito lenta.  A demora fez com que o único pool de conexões fosse preenchido e, como as threads ficavam esperando indefinidamente, o sistema todo veio abaixo.

> _Responder muito lentamente é um dos piores modos de falha que você pode experimentar. Se um sistema não está no ar, você descobre rapidamente. Quando apenas está lento, você acaba esperando um pouco antes de desistir. [...] descobrimos da maneira mais difícil que os sistemas lentos são muito mais difíceis de lidar do que os sistemas que falham rapidamente. Em um Sistema Distribuído, a Latência mata._
>
> Sam Newman, no livro [Building Microservices](https://learning.oreilly.com/library/view/building-microservices/9781491950340/) (NEWMAN, 2015)

## Patterns de Estabilidade

Desejamos que nosso Sistema Distribuído apresente como característica a **Resiliência**. Segundo o Dicionário Michaelis, um significado figurativo da palavra [Resiliência](https://michaelis.uol.com.br/moderno-portugues/busca/portugues-brasileiro/resili%C3%AAncia/) é a _capacidade de rápida adaptação ou recuperação_. A qual _ilidade_ a Resiliência estaria ligada? À **Estabilidade** que, também segundo o Dicionário Michaelis, é _característica daquilo que é estável; solidez._

Michael Nygard, em seu livro [Release It! Second Edition](https://pragprog.com/book/mnee2/release-it-second-edition) (NYGARD, 2018), define Estabilidade como a característica observável em _um sistema resiliente que se mantém processando transações [de negócio], mesmo quando há impulsos transientes, estresses persistentes ou falhas de componentes que interrompem o processamento normal._

<!--@note
  Nygard define uma série de termos de usados quando fala em Estabilidade:
  transaction: abstract unit of work processed by the system
  system: I mean the complete, interdependent set of hardware, applications, and services required to process transactions for users

  (impulse and stress come from mechanical engineering)

  impulse: a rapid shock to the system
  stress: a force applied to the system over an extended period
-->

Nygard lista uma série de patterns de Estabilidade. Alguns deles serão descritos a seguir.

## Timeouts

No exemplo do agregador de anúncios mencionado por Sam Newman, no livro [Building Microservices](https://learning.oreilly.com/library/view/building-microservices/9781491950340/) (NEWMAN, 2015), a lentidão em um dos legados ocasionou uma interrupção no sistema todo. O motivo mencionado por Newman é que o pool de conexões único utilizado esperava "para sempre" e, com uma alta demanda, o pool foi exaurido e o serviço de anúncios não poderia chamar nenhum outro legado. A biblioteca de pool de conexões dava suporte a _timeouts_, mas estava desabilitada por padrão!

Newman, recomenda que todas as chamadas remotas tenham timeouts configurados. E qual valor definir? Se for longo demais, ainda causará lentidão no sistema. Se for rápido demais, uma chamada bem sucedida por ser considerado como falha. Uma boa solução é usar valores default de bibliotecas, ajustando valores para cenários específicos.

Em seu livro [Release It! Second Edition](https://pragprog.com/book/mnee2/release-it-second-edition) (NYGARD, 2018), Michael Nygard argumenta que timeouts provêem _isolamento de falhas_, já que um problema em outro serviço, dispositivo ou sistema não deve ser um problema de quem o invoca. Nygard relata que, infelizmente, muitas APIs e bibliotecas de clientes de sistemas como Bancos de Dados não provêem maneiras de setar timeouts. Para Nygard, qualquer pool de recursos que bloqueia threads deve ter timeouts.

## Fail Fast

Michael Nygard, ainda no livro [Release It! Second Edition](https://pragprog.com/book/mnee2/release-it-second-edition) (NYGARD, 2018), diz: _se respostas lentas são piores que nenhuma resposta, o pior dos mundos certamente são falhas lentas._ Se um sistema puder detectar que vai falhar, é melhor que retorne o mais rápido possível um response de erro para seus clientes. Como predizer uma falha? Nygard afirma que um load balancer, por exemplo, deve recusar novas requisições se não houver nenhum servidor para balanceamento no ar, evitando enfileirar requisições.

Para Nygard, no código de uma aplicação, parâmetros devem ser validados e recursos obtidos, como conexões a BDs ou a outros sistemas, assim que possível. Assim, se houver uma falha de validação ou na obtenção de algum recurso, é possível falhar rapidamente. É importante notar que isso serve como um príncipio, porém não é aplicável em todas as situações.

<!--@note
Nygard cita um projeto para um estúdio fotográfico em que, caso fontes, cores, imagens ou planos de fundo não estivessem disponíveis, uma imagem preta era enviada a uma impressora de alta resolução. Eram gastos recursos caríssimos como papéis especiais, produtos químicos, tempo de impressão e profissionais para separar as imagens defeituosas, diagnosticando e ajudando desenvolvedores a debugar o software. Então, o time de Nygard aplicou o pattern _Fail Fast_, checando fontes, cores, imagens, planos de fundo, memória disponível. Dessa forma, os erros induzidos pelo software foram a zero.
-->

Para Nygard, tanto o Timeout como Fail Fast são patterns que tratam de problemas de latência, sendo dois lados da mesma moeda. O Timeout é útil para proteger o seu sistema de falhas dos outros. O Fail Fast é útil quando você precisa avisar o porquê não será possível processar alguma requisição. Timeouts são para saídas e Fail Fast para entradas.

## Bulkheads

Ainda no livro [Release It! Second Edition](https://pragprog.com/book/mnee2/release-it-second-edition) (NYGARD, 2018), Michael Nygard empresta um conceito da Engenharia Naval: as anteparas (em inglês, _bulkheads_): _em um navio, bulkheads são divisórias metálicas que podem ser seladas para dividir o navio em compartimentos separados. Quando as escotilhas são fechadas, uma bulkhead impede que a água se mova de uma seção para outra. Dessa maneira, um único dano no casco não afunda irreversivelmente o navio. A bulkhead aplica um princípio de contenção de dados._

![Navios sem e com bulkheads {w=80}](imagens/09-resiliencia/bulkheads-navais.png)

_Observação: a fonte das imagens anteriores é a [documentação do OpenLiberty](https://openliberty.io/guides/bulkhead.html), um microservice chassis baseado no IBM WebSphere._

Nygard afirma que, em TI, redundância física é a maneira mais comum de aplicar a ideia de bulkheads: uma falha no hardware de um servidor não afetaria os outros. O mesmo princípio pode ser atingido executando múltiplas instâncias de um serviço em um mesmo servidor.

Para Nygard, há maneiras mais granulares de aplicar bulkheads. Por exemplo, é possível apartar em pools diferentes threads de um processo que tem responsabilidades distintas, separando threads que tratam requests de threads administrativas. Assim, se as threads da aplicação travarem, é possível usar as threads administrativas para obter um dump ou fazer um _shut down_. Nygard menciona ainda que um Sistema Operacional pode alocar um processo a um core específico (ou a um grupo de cores), o que é conhecido como _CPU Binding_. Dessa maneira, a sobrecarga em um processo não degrada a performance da máquina toda, já outros cores estarão liberados para outros processo.

Há também maneiras menos granulares. No caso de _cloud computing_, podem ser exploradas diferentes topologias como zonas e regiões da AWS.

> _Você pode particionar pools de threads em uma aplicação, CPUs em um servidor ou servidores em um cluster._
>
> Michael Nygard, no livro [Release It! Second Edition](https://pragprog.com/book/mnee2/release-it-second-edition) (NYGARD, 2018)

Em seu livro [Building Microservices](https://learning.oreilly.com/library/view/building-microservices/9781491950340/) (NEWMAN, 2015), Sam Newman afirma que as barreiras arquiteturais entre microservices são, no fim das contas, bulkheads: uma falha em um serviço pode degradar certas funcionalidades, mas não pára tudo.

No exemplo do agregador de anúncios de Newman, todas as conexões aos sistemas legados compartilhavam um mesmo pool de conexões. A falha em um legado derrubou o único pool de conexões e, consequentemente, impediu que chamadas fossem feitas aos outros legados, tornando o sistema inutilizável. Usando a ideia de bulkheads, cada sistema legado deveria ter seu próprio pool de conexões, de maneira a isolar possíveis falhas.

<!-- 
## Circuit Breaker
 -->

## Exercício: simulando demora no serviço de distância

1. Altere o método `calculaDistancia` da classe `DistanciaService` do serviço de distância, para que invoque o método que simula uma demora de 10 a 20 segundos:

  ####### fj33-eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/DistanciaService.java

  ```java
  class DistanciaService {

    // código omitido ...

    private BigDecimal calculaDistancia() {
      simulaDemora(); // modificado
      return new BigDecimal(Math.random() * 15);
    }

  }
  ```

2. Em um Terminal, use o ApacheBench para simular a consulta da distância entre um CEP e um restaurante específico, cujos dados são compostos no API Gateway, com 100 requisições ao todo e 10 requisições concorrentes.

  O comando será parecido com o seguinte:

  ```sh
  ab -n 100 -c 10 http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1
  ```

  A opção `-n` define o número total de requisições. A opção `-c`, o número de requisições concorrentes.

  Entre os resultados aparecerá algo como:

  ```txt
  Connection Times (ms)
              min  mean[+/-sd] median   max
  Connect:        0    0   0.1      0       1
  Processing: 10097 15133 2817.3  14917   19635
  Waiting:    10096 15131 2817.4  14917   19632
  Total:      10097 15133 2817.3  14917   19636
  ```

  A requisição mais demorada, no exemplo anterior, foi de 19,6 segundos. Inviável!

## Circuit Breaker com Hystrix

No `pom.xml` do API Gateway, adicione o _starter_ do Spring Cloud Netflix Hystrix:

####### fj33-api-gateway/pom.xml

```xml
<dependency>
  <groupId>org.springframework.cloud</groupId>
  <artifactId>spring-cloud-starter-netflix-hystrix</artifactId>
</dependency>
```

Adicione a anotação `@EnableCircuitBreaker` à classe `ApiGatewayApplication`:

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/ApiGatewayApplication.java

```java
@EnableCircuitBreaker // adicionado
// demais anotações...
public class ApiGatewayApplication {
  // código omitido...
}
```

Não deixe de adicionar o import correto:

```java
import org.springframework.cloud.client.circuitbreaker.EnableCircuitBreaker;
```

Na classe `DistanciaRestClient` do API Gateway, habilite o _circuit breaker_ no método `porCepEId`, com a anotação `@HystrixCommand`:

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/DistanciaRestClient.java

```java
@Service
class DistanciaRestClient {

  // código omitido...

  @HystrixCommand // adicionado
  Map<String, Object>  porCepEId(String cep, Long restauranteId) {
    String url = distanciaServiceUrl + "/restaurantes/" + cep + "/restaurante/" + restauranteId;
    return restTemplate.getForObject(url, Map.class);
  }

}
```

O import é o seguinte:

```java
import com.netflix.hystrix.contrib.javanica.annotation.HystrixCommand;
```

<!--@note

O timeout padrão do Hystrix é de 1s. Modificável pela propriedade:

hystrix.command.default.execution.isolation.thread.timeoutInMilliseconds

https://github.com/Netflix/Hystrix/wiki/Configuration#CommandProperties

Porém, há problemas na interação entre Zuul, Ribbon e Hystrix:
https://github.com/spring-cloud/spring-cloud-netflix/issues/2606

-->

## Exercício: Circuit Breaker com Hystrix

1. Mude para a branch `cap10-circuit-breaker-com-hystrix` do projeto `fj33-api-gateway`:

  ```sh
  cd ~/Desktop/fj33-api-gateway
  git checkout -f cap10-circuit-breaker-com-hystrix
  ```

2. Reinicie o API Gateway e execute novamente a simulação com o ApacheBench, com o comando:

  ```sh
  ab -n 100 -c 10 http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1
  ```

  Observe nos resultados uma diminuição no tempo máximo de request de 19,6 para 1,5 segundos:

  ```txt
  Connection Times (ms)
                min  mean[+/-sd] median   max
  Connect:        0    0   0.7      0       7
  Processing:    75  381 360.8    275    1557
  Waiting:       67  375 359.0    270    1527
  Total:         75  382 360.9    275    1558
  ```

## Fallback no @HystrixCommand

Se acessarmos repetidas vezes, em um navegador, a URL a seguir:

http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1

Deve ocorrer, em algumas das vezes, uma exceção semelhante a:

```txt
There was an unexpected error (type=Internal Server Error, status=500).
route:SendForwardFilter
com.netflix.hystrix.exception.HystrixRuntimeException: porCepEId timed-out and fallback failed.
  at com.netflix.hystrix.AbstractCommand$22.call(AbstractCommand.java:832)
  at com.netflix.hystrix.AbstractCommand$22.call(AbstractCommand.java:807)
  at rx.internal.operators.OperatorOnErrorResumeNextViaFunction$4.onError(OperatorOnErrorResumeNextViaFunction.java:140)
  ...
```

A mensagem da exceção (_porCepEId timed-out and fallback failed_) ,indica que houve um erro de timeout.

Em outras tentativas, teremos uma exceção semelhante, mas cuja mensagem indica que o Circuit Breaker está aberto e a resposta foi _short-circuited_, não chegando a invocar o serviço de destino da requisição:

```txtx
There was an unexpected error (type=Internal Server Error, status=500).
route:SendForwardFilter
com.netflix.hystrix.exception.HystrixRuntimeException: porCepEId short-circuited and fallback failed.
	at com.netflix.hystrix.AbstractCommand$22.call(AbstractCommand.java:832)
	at com.netflix.hystrix.AbstractCommand$22.call(AbstractCommand.java:807)
	at rx.internal.operators.OperatorOnErrorResumeNextViaFunction$4.onError(OperatorOnErrorResumeNextViaFunction.java:140)
```

É possível fornecer um _fallback_, passando o nome de um método na propriedade `fallbackMethod` da anotação `@HystrixCommand`.

Defina o método `restauranteSemDistanciaNemDetalhes`, que retorna apenas o restaurante com o id. Se a outra parte da API Composition, a interface `RestauranteRestClient` não der erro e retornar os dados do restaurante, teríamos todos os detalhes do restaurante menos a distância.

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/DistanciaRestClient.java

```java
@Service
class DistanciaRestClient {

  // código omitido...

  @̶H̶y̶s̶t̶r̶i̶x̶C̶o̶m̶m̶a̶n̶d̶
  @HystrixCommand(fallbackMethod="restauranteSemDistanciaNemDetalhes") // modificado
  Map<String, Object>  porCepEId(String cep, Long restauranteId) {
    String url = distanciaServiceUrl+"/restaurantes/"+cep+"/restaurante/"+restauranteId;
    return restTemplate.getForObject(url, Map.class);
  }

  // método adicionado
  Map<String, Object> restauranteSemDistanciaNemDetalhes(String cep, Long restauranteId) {
    Map<String, Object> resultado = new HashMap<>();
    resultado.put("restauranteId", restauranteId);
    resultado.put("cep", cep);
    return resultado;
  }

}
```

O seguinte import deve ser adicionado:

```java
import java.util.HashMap;
```

Observação: uma solução interessante seria manter um cache das distâncias entre CEPs e restaurantes e usá-lo como fallback, se possível. Porém, a _hit ratio_, a taxa de sucesso das consultas ao cache, deve ser baixa, já que os CEPs dos clientes mudam bastante.

## Exercício: Fallback com Hystrix

1. Acesse repetidas vezes, em um navegador, a URL a seguir:

  http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1

  Deve ocorrer, em algumas das vezes, uma exceção `HystrixRuntimeException` com as mensagens:

  - _porCepEId timed-out and fallback failed._
  - _porCepEId short-circuited and fallback failed._

2. No projeto `fj33-api-gateway`, obtenha o código da branch `cap10-fallback-no-hystrix-command`:

  ```sh
  cd ~/Desktop/fj33-api-gateway
  git checkout -f cap10-fallback-no-hystrix-command
  ```
  
  Reinicie o API Gateway.

3. Tente acessar várias vezes a URL testada anteriormente:

  http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1

  Observe que não ocorre mais uma exceção, mas não há a informação de distância. Apenas os detalhes do restaurante são retornados.

  Algo semelhante a:
  
  ```json
  {
    "id": 1,
    "cnpj": "98444252000104",
    "nome": "Long Fu",
    "descricao": "O melhor da China aqui do seu lado.",
    "cep": "71503510",
    "endereco": "ShC/SUL COMERCIO LOCAL QD 404-BL D LJ 17-ASA SUL",
    "taxaDeEntregaEmReais": 6,
    "tempoDeEntregaMinimoEmMinutos": 40,
    "tempoDeEntregaMaximoEmMinutos": 25,
    "aprovado": true,
    "tipoDeCozinha": {
      "id": 1,
      "nome": "Chinesa"
    },
    "restauranteId": 1
  }
  ```

## Exercício: Removendo simulação de demora do serviço de distância

1. Comente a chamada ao método que simula a demora em `DistanciaService` do `eats-distancia-service`. Veja se, quando não há demora, a distância volta a ser incluída na resposta.

  ####### fj33-eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/DistanciaService.java

  ```java
  class DistanciaService {

    // código omitido ...

    private BigDecimal calculaDistancia() {
      //simulaDemora(); // modificado
      return new BigDecimal(Math.random() * 15);
    }

  }
  ```

## Exercício: Simulando demora no monólito

1. Altere o método `detalha` da classe `RestauranteController` do monólito para que tenha uma espera de 20 segundos:

  ####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/RestauranteController.java

  ```java
  // anotações ...
  class RestauranteController {

    // código omitido ...

    @GetMapping("/restaurantes/{id}")
    RestauranteDto detalha(@PathVariable("id") Long id) {

      // trecho de código adicionado ...
      try {
        Thread.sleep(20000);
      } catch (InterruptedException e) {
        throw new RuntimeException(e);
      }

      Restaurante restaurante = restauranteRepo.findById(id).orElseThrow(() -> new ResourceNotFoundException());
      return new RestauranteDto(restaurante);
    }
  
  // restante do código ...

  }
  ```

## Circuit Breaker com Hystrix no Feign

No `application.properties` do API Gateway, é preciso adicionar a seguinte linha:

####### fj33-api-gateway/src/main/resources/application.properties

```properties
feign.hystrix.enabled=true
```

A integração entre o Feign e o Hystrix vem desabilitada por padrão, nas versões mais recentes. Por isso, é necessário habilitá-la.

## Exercício: Integração entre Hystrix e Feign

1. Faça o checkout da branch `cap10-circuit-breaker-com-hystrix-no-feign` do projeto `fj33-api-gateway`:

  ```sh
  cd ~/Desktop/fj33-api-gateway
  git checkout -f cap10-circuit-breaker-com-hystrix-no-feign
  ```

  Reinicie o API Gateway.

2. Tente acessar novamente a URL:

  http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1

  _Observação: a URL anterior, além de obter a distância do serviço apropriado, obtém os detalhes do restaurante do monólito utilizando o Feign na implementação do cliente REST._

  Deve ocorrer a seguinte exceção:

  ```java
  There was an unexpected error (type=Internal Server Error, status=500).
    route:SendForwardFilter
    com.netflix.hystrix.exception.HystrixRuntimeException: RestauranteRestClient#porId(Long) timed-out and no fallback available.
    ...
  ```

  Quando o Circuit Breaker estiver aberto, a mensagem da exceção `HystrixRuntimeException` será um pouco diferente: _RestauranteRestClient#porId(Long) short-circuited and no fallback available_.

## Fallback com Feign

No Feign, definimos de maneira declarativa o cliente REST, por meio de uma interface.

A estratégia de Fallback na integração entre Hystrix e Feign é fornecer uma implementação para essa interface. Engenhoso!

No `api-gateway`, crie uma classe `RestauranteRestClientFallback`, que implementa a interface `RestauranteRestClient`. No método `porId`, deve ser fornecida uma lógica de fallback para o detalhamento de um restaurante. Anote essa nova classe com `@Component`, para que seja gerenciada pelo Spring.

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/RestauranteRestClientFallback.java

```java
@Component
class RestauranteRestClientFallback implements RestauranteRestClient {

  @Override
  public Map<String,Object> porId(Long id) {
    Map<String,Object> resultado = new HashMap<>();
    resultado.put("id", id);
    return resultado;
  }

}
```

A seguir, estão os imports corretos:

```java
import java.util.HashMap;
import java.util.Map;

import org.springframework.stereotype.Component;
```

Observação: Uma solução mais interessante seria manter um cache dos dados dos restaurantes, com o `id` como chave, que seria usado em caso de fallback. Nesse caso, a _hit ratio_, a taxa de sucesso das consultas ao cache, seria bem alta: há um número limitado de restaurantes, que são escolhidos repetidas vezes, e os dados são raramente alterados.

Altere a anotação `@FeignClient` de `RestauranteRestClient`, passando na propriedade `fallback` a classe criada no passo anterior.

####### fj33-api-gateway/src/main/java/br/com/caelum/apigateway/RestauranteRestClient.java

```java
@̶F̶e̶i̶g̶n̶C̶l̶i̶e̶n̶t̶(̶"̶m̶o̶n̶o̶l̶i̶t̶o̶"̶)̶
@FeignClient(name = "monolito", fallback=RestauranteRestClientFallback.class) // modificado
interface RestauranteRestClient {

  @GetMapping("/restaurantes/{id}")
  Map<String,Object> porId(@PathVariable("id") Long id);

}
```

## Exercício: Fallback com Feign

1. Vá até a branch `cap10-fallback-com-feign` do projeto `fj33-api-gateway`:

  ```sh
  cd ~/Desktop/fj33-api-gateway
  git checkout -f cap10-fallback-com-feign
  ```

  Certifique-se que o API Gateway foi reiniciado.

2. Por mais algumas vezes, tente acessar a URL:

    http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1

  Veja que são mostrados apenas o id e a distância do restaurante. Os demais campos não são exibidos.

## Exercício: Removendo simulação de demora do monólito

1. Remova da classe `RestauranteController` do monólito, a simulação de demora.
  
  ####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/RestauranteController.java

  ```java
  // anotações ...
  class RestauranteController {

    // código omitido ...

    @GetMapping("/restaurantes/{id}")
    public RestauranteDto detalha(@PathVariable("id") Long id) {

      t̶r̶y̶ ̶{̶
        T̶h̶r̶e̶a̶d̶.̶s̶l̶e̶e̶p̶(̶2̶0̶0̶0̶0̶)̶;̶
      }̶ ̶c̶a̶t̶c̶h̶ ̶(̶I̶n̶t̶e̶r̶r̶u̶p̶t̶e̶d̶E̶x̶c̶e̶p̶t̶i̶o̶n̶ ̶e̶)̶ ̶{̶
        t̶h̶r̶o̶w̶ ̶n̶e̶w̶ ̶R̶u̶n̶t̶i̶m̶e̶E̶x̶c̶e̶p̶t̶i̶o̶n̶(̶e̶)̶;̶
      }̶

      Restaurante restaurante = restauranteRepo.findById(id).orElseThrow(() -> new ResourceNotFoundException());
      return new RestauranteDto(restaurante);
    }
  ```

  Teste novamente a URL: http://localhost:9999/restaurantes-com-distancia/71503510/restaurante/1

  Os detalhes do restaurante devem voltar a ser exibidos!

## Exercício: Forçando uma exceção no serviço de distância

1. No serviço de distância, force o lançamento de uma exceção no método `atualiza` da classe `RestaurantesController`.

  Comente o código que está depois da exceção.

  ####### fj33-eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/RestaurantesController.java

  ```java
  // anotações ...
  class RestaurantesController {

    // código omitido ...

    @PutMapping("/restaurantes/{id}")
    Restaurante atualiza(@PathVariable("id") Long id, @RequestBody Restaurante restaurante) {

      throw new RuntimeException();

      // código comentado ...

      //if (!repo.existsById(id)) {
      //  throw new ResourceNotFoundException();
      //}
      //log.info("Atualiza restaurante: " + restaurante);
      //return repo.save(restaurante);
    }

  }
  ```

## Tentando novamente com Spring Retry

No módulo `eats-restaurante` do monólito, adicione o Spring Retry como dependência:

####### fj33-eats-monolito-modular/eats/eats-restaurante/pom.xml

```xml
<dependency>
  <groupId>org.springframework.retry</groupId>
  <artifactId>spring-retry</artifactId>
</dependency>
```

Adicione a anotação `@EnableRetry` na classe `EatsApplication` do módulo `eats-application` do monólito:

####### fj33-eats-monolito-modular/eats/eats-application/src/main/java/br/com/caelum/eats/EatsApplication.java

```java
@EnableRetry // adicionado
// outra anotações
public class EatsApplication {

  // código omitido...

}
```

Faça o import adequado:

```java
import org.springframework.retry.annotation.EnableRetry;
```

Adicione a anotação `@Slf4j` à classe `DistanciaRestClient`, do módulo `eats-restaurante` do monólito, para configurar um logger que usaremos a seguir:

####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/DistanciaRestClient.java

```java
@Slf4j // adicionado
@Service
public class DistanciaRestClient {

  // código omitido...

}
```

O import é o seguinte:

```java
import lombok.extern.slf4j.Slf4j;
```

Em seguida, anote o método `restauranteAtualizado` com `@Retryable` para que faça 5 tentativas, logando as tentativas de acesso:

####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/DistanciaRestClient.java

```java
// anotações ...
public class DistanciaRestClient {

  // código omitido ...

  @Retryable(maxAttempts=5) // adicionado
  public void restauranteAtualizado(Restaurante restaurante) {
    log.info("monólito tentando chamar distancia-service");

    // código omitido ...
  }

}
```

Certifique-se que o import correto foi realizado:

```java
import org.springframework.retry.annotation.Retryable;
```

## Exercício: Spring Retry

1. Faça o checkout da branch `cap10-retry` do monólito:

  ```sh
  cd ~/Desktop/fj33-eats-monolito-modular
  git checkout -f cap10-retry
  ```

  Reinicie o monólito.

2. Garanta que o monólito, o serviço de distância e que a UI estejam no ar.

  Faça login como dono de um restaurante (por exemplo, `longfu`/`123456`) e mude o CEP ou tipo de cozinha.

  Perceba que nos logs que foram feitas 5 tentativas de chamada ao serviço de distância. Algo como o que segue:

  ```txt
  2019-06-18 17:30:42.943  INFO 12547 --- [nio-8080-exec-9] b.c.c.e.restaurante.DistanciaRestClient  : monólito tentando chamar distancia-service
  2019-06-18 17:30:43.967  INFO 12547 --- [nio-8080-exec-9] b.c.c.e.restaurante.DistanciaRestClient  : monólito tentando chamar distancia-service
  2019-06-18 17:30:44.990  INFO 12547 --- [nio-8080-exec-9] b.c.c.e.restaurante.DistanciaRestClient  : monólito tentando chamar distancia-service
  2019-06-18 17:30:46.034  INFO 12547 --- [nio-8080-exec-9] b.c.c.e.restaurante.DistanciaRestClient  : monólito tentando chamar distancia-service
  2019-06-18 17:30:46.085 ERROR 12547 --- [nio-8080-exec-9] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is org.springframework.web.client.HttpServerErrorException$InternalServerError: 500 null] with root cause

  org.springframework.web.client.HttpServerErrorException$InternalServerError: 500 null
    at org.springframework.web.client.HttpServerErrorException.create(HttpServerErrorException.java:79) ~[spring-web-5.1.4.RELEASE.jar:5.1.4.RELEASE]
    ...
  ```

## Exponential Backoff

Vamos configurar um backoff para ter um tempo progressivo entre as tentativas de 2, 4, 8 e 16 segundos:

####### fj33-eats-monolito-modular/eats/eats-restaurante/src/main/java/br/com/caelum/eats/restaurante/DistanciaRestClient.java

```java
// anotações ...
public class DistanciaRestClient {

  // código omitido ...

  @̶R̶e̶t̶r̶y̶a̶b̶l̶e̶(̶m̶a̶x̶A̶t̶t̶e̶m̶p̶t̶s̶=̶5̶)̶
  @Retryable(maxAttempts=5, backoff=@Backoff(delay=2000,multiplier=2))
  public void restauranteAtualizado(Restaurante restaurante) {
    // código omitido ...
  }

}
```

O import a seguir deve ser adicionado:

```java
import org.springframework.retry.annotation.Backoff;
```

## Exercício: Exponential Backoff com Spring Retry

1. Vá até a branch `cap10-backoff` do projeto `fj33-eats-monolito-modular`:

  ```sh
  cd ~/Desktop/fj33-eats-monolito-modular
  git checkout -f cap10-backoff
  ```

2. Pela UI, faça novamente o login como dono de um restaurante (por exemplo, com `longfu`/`123456`) e modifique o CEP ou tipo de cozinha.

  Note o tempo progressivo nos logs. Será alguma coisa semelhante a:

  ```txt
  2019-06-18 18:00:18.367  INFO 15044 --- [nio-8080-exec-8] b.c.c.e.restaurante.DistanciaRestClient  : monólito tentando chamar distancia-service
  ...
  2019-06-18 18:00:20.973  INFO 15044 --- [nio-8080-exec-8] b.c.c.e.restaurante.DistanciaRestClient  : monólito tentando chamar distancia-service
  2019-06-18 18:00:24.994  INFO 15044 --- [nio-8080-exec-8] b.c.c.e.restaurante.DistanciaRestClient  : monólito tentando chamar distancia-service
  2019-06-18 18:00:33.047  INFO 15044 --- [nio-8080-exec-8] b.c.c.e.restaurante.DistanciaRestClient  : monólito tentando chamar distancia-service
  2019-06-18 18:00:49.079  INFO 15044 --- [nio-8080-exec-8] b.c.c.e.restaurante.DistanciaRestClient  : monólito tentando chamar distancia-service
  2019-06-18 18:00:49.127 ERROR 15044 --- [nio-8080-exec-8] o.a.c.c.C.[.[.[/].[dispatcherServlet]    : Servlet.service() for servlet [dispatcherServlet] in context with path [] threw exception [Request processing failed; nested exception is org.springframework.web.client.HttpServerErrorException$InternalServerError: 500 null] with root cause
  ...
  ```

## Exercício: Removendo exceção forçada do serviço de distância

1. Agora que testamos o retry e o backoff, vamos remover a exceção que forçamos anteriormente na classe `RestaurantesController` do serviço de distância:

  ####### fj33-eats-distancia-service/src/main/java/br/com/caelum/eats/distancia/RestaurantesController.java

  ```java
  // anotações ...
  class RestaurantesController {

    // código omitido ...

    @PutMapping("/restaurantes/{id}")
    Restaurante atualiza(@PathVariable("id") Long id, @RequestBody Restaurante restaurante) {

      t̶h̶r̶o̶w̶ ̶n̶e̶w̶ ̶R̶u̶n̶t̶i̶m̶e̶E̶x̶c̶e̶p̶t̶i̶o̶n̶(̶)̶;̶

      // descomente o código abaixo ...

      if (!repo.existsById(id)) {
        throw new ResourceNotFoundException();
      }
      log.info("Atualiza restaurante: " + restaurante);
      return repo.save(restaurante);
    }

  }
  ```
