# Decompondo o monólito

## Da bagunça a camadas

Qual é a emoção que a palavra **monólito** traz pra você?

Muitos desenvolvedores associam a palavra monólito a código mal feito, sem estrutura aparente, com muito código repetido e com dados compartilhados entre partes pouco relacionadas. É o que comumente chamado de código **Spaghetti** ou, [Big Ball of Mud](http://www.laputan.org/mud/mud.html) (FOOTE; YODER, 1999).

Porém, é possível criar monólitos bem estruturados. Uma maneira comum de organizar um monólito é usar camadas: cada camada provê um serviço para a camada de cima e é um cliente das camadas de baixo.

Usualmente, o código de uma aplicação é estruturado em 3 camadas:

- _Apresentação_: responsável por prover serviços ao front-end. Em alguns casos, é feita a renderização das telas da UI, como ao utilizar JSPs.
- _Negócio_: responsável pelos cálculos, fluxos e regras de negócio.
- _Persistência_: responsável pelo acesso aos dados armazenados, geralmente, em um Banco de Dados.

Uma maneira de representar essas camadas em um código Java é utilizar pacotes, o que é conhecido como _Package by Layer_.

> Na verdade, é preciso distinguir dois tipos de camadas que influenciam a arquitetura do software: as camadas físicas e as camadas lógicas.
> Uma camada física, ou _tier_ em inglê, descreve a estrutura de implantação do software, ou seja, as máquinas utilizadas.
> O que descrevemos no texto anterior é o conceito de camada lógica. Em inglês, a camada lógica é chamada de _layer_. Trata de agrupar o código que corresponde às camadas descritas anteriormente.

### Camadas no Caelum Eats

O código de back-end do Caelum Eats está organizado em camadas (layers). Podemos observar isso estudando a estrutura de pacotes do código Java:

<!--@note
É possível rodar o comando abaixo, na pasta do pacote base do código Java (src/main/java/br/com/caelum/eats/) do projeto fj33-eats-monolito, para mostrar a estrutura básica dos pacotes.

  tree -d -L 1

Claro, pela IDE também é possível ter uma "flat presentation" dos pacotes.
-->

```txt
eats
├── controller
├── dto
├── exception
├── model
├── repository
└── service
```

Bem organizado, não é mesmo?

Porém, quando a aplicação começa a crescer, o código passa a ficar difícil de entender. Centenas de Controllers juntos, centenas de Repositories misturados. Passa a ser difícil encontrar o código que precisa ser alterado.

## A Arquitetura que Grita

Reflita um pouco: dos projetos implementados com a plataforma Java em que você trabalhou, quantos seguiam uma variação da estrutura Package by Layer que estudamos anteriormente? Provavelmente, a maioria!

Veja a estrutura de diretórios abaixo:

```txt
.
├── assets
├── controllers
├── helpers
├── mailers
├── models
└── views
```

Os diretórios anteriores são a estrutura padrão para um framework aplicações Web muito influente: Ruby On Rails.

Perceba que interessante: a estrutura básica de diretórios é familiar; em alguns casos, até temos um palpite sobre qual o framework utilizado; mas não temos ideia do **domínio** da aplicação. Qual é o problema que está sendo resolvido?

No post [Screaming Architecture](https://blog.cleancoder.com/uncle-bob/2011/09/30/Screaming-Architecture.html) (MARTIN, 2011), Robert "Uncle Bob" Martin diz que a partir das plantas, conseguimos saber que trata-se de uma casa ou uma biblioteca: a arquitetura "grita" a finalidade da construção.

Para projetos de software, entretanto, é usual que a estrutura básica de diretórios indique qual o framework ou qual a ferramenta de build utilizados. Porém, para Uncle Bob, o framework é um detalhe; o Banco de Dados é um detalhe; a Web é um mecanismo de entrega da UI, um detalhe.

Uncle Bob cita a ideia de Ivar Jacobson, um dos criadores do UML, descrita no livro [Object Oriented Software Engineering: A Use-Case Driven Approach](https://www.amazon.com/Object-Oriented-Software-Engineering-Approach/dp/0201544350) (JACOBSON, 1992), de que a arquitetura de um software deveria ser _centrada nos casos de uso_ e não em detalhes técnicos.

## Clean Architecture

No livro [Clean Architecture](https://www.amazon.com/Clean-Architecture-Craftsmans-Software-Structure/dp/0134494164) (MARTIN, 2017), Uncle Bob define uma abordagem arquitetural que torna a aplicação:

- **independente de frameworks**: um framework não é sua aplicação. A estrutura de diretórios e as restrições do design do nosso código não deveriam ser determinadas por um framework. Frameworks deveriam ser usados apenas como ferramentas para que a aplicação cumpra suas necessidades.
- **independente da UI**: a UI muda mais frequentemente que o resto do sistema. Além disso, é possível termos diferentes UIs para as mesmas regras de negócio.
- **independente de BD**: as regras de negócio não devem depender de um Banco de Dados específico. Devemos possibilitar a troca, de maneira fácil, de Oracle ou SQL Server para MongoDB, CouchDB, Neo4J ou qualquer outro BD.
- **testável**: deve ser possível testar as regras de negócio diretamente, sem a necessidade de usar uma UI, BD ou servidor Web.

Uncle Bob ainda cita, no mesmo livro, outras arquiteturas semelhantes:

- Hexagonal Architecture, ou Ports & Adapters, descrita por Alistair Cockburn.
- DCI (Data, Context and Interaction), descrita por James Coplien, pioneiro dos Design Patterns, e Trygve Reenskaug, criador do MVC.
- BCE (Boundary-Control-Entity), introduzida por Ivar Jacobson no livro mencionado anteriormente.

## Explorando o domínio com DDD

Até um certo tamanho, uma aplicação estruturada em camadas no estilo Package by Layer, é fácil de entender. Novos desenvolvedores entram no projeto sem muitas dificuldades para entender a organização do código, já que é uma forma comum.

Mas há um momento em que é interessante reorganizar os pacotes ao redor do domínio, o que alguns chamam de _Package by Feature_. O intuito é que fique bem clara qual é a área de negócio de cade parte do código.

Mas qual critério usar para decompor o monólito? Uma funcionalidade é um pacote, como sugere o nome Package by Feature? Talvez isso seja muito granular.

Uma fonte de _insights_ interessantes em como explorar o domínio de uma aplicação é o livro [Domain Driven Design](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215), de Eric Evans (EVANS, 2003).

## Agregado

Todo domínio tem entidades importantes, que são um ponto de entrada para o negócio e tem várias outras entidades ligadas. Um objeto que representa um pedido, por exemplo, terá informações dos itens do pedido, do cliente, da entrega e do pagamento. É o que é chamado, nos termos do DDD, de _Agregado_.

> **AGREGADO**
>
> Agrupamento de objetos associados que são tratados como uma unidade para fiz de alterações nos dados. Referências externas são restritas a um único membro do AGREGADO, designado como _raiz_. Um conjunto de regras de consistência se aplicada dentro dos limites do AGREGADO.
>
> [Domain Driven Design](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215) (EVANS, 2003)

Referências a outros objetos, de fora desse Agregado, devem ser feitas por meio do objeto principal, a raiz do Agregado. Uma relação entre um pedido e um restaurante deve ser feita pelo pedido, e não pela entrega ou o pagamento.

Os dados de um Agregado são alterados em conjunto. Por isso, Banco de Dados Orientados a Documentos, como o MongoDB, em que um documento pode ter um grafo de outros objetos conectados, são interessantes para persistir Agregados.

E no Caelum Eats?

<!--@note
  Falar para o pessoal abrir o pacote `model` e ir explorando com eles os objetos principais.
  Explorar os outros pacotes, desenho o que o pessoal acha que são os agregados.
  Classes como `RestauranteComDistanciaDto` e `MediaAvaliacoesDto` são objetos interessante que estão no pacote `dto`.
  Uma classe especial é a DistanciaService, do pacote `service`. Faz parte do agregado de Restaurante ou não?
-->

Um objeto muito importante é o `Pedido`. Associado a um `Pedido` temos uma lista de `ItemDoPedido` e uma `Entrega`. Uma `Entrega` está associada a um `Cliente`. Cada `Pedido` também pode ter uma `Avaliacao`. Há, possivelmente, um `Pagamento` para cada `Pedido`. E um `Pagamento` tem uma `FormaDePagamento`. Um `Pedido` também está associado a um `Restaurante`.

Um `Restaurante` tem seu cadastro mantido por seu dono e é aprovado pelo administrador do Caelum Eats. Um `Restaurante` pode existir sem nenhum pedido, o que acontece logo que foi cadastrado. Um `Restaurante` está relacionado a um `Cardapio` que contém uma lista de `CategoriaDoCardapio` que, por sua vez, contém uma lista de `ItemDoCardapio`. Um `Restaurante` possui também uma lista de `HorarioDeFuncionamento` e das `FormaDePagamento` aceitas, assim como um `TipoDeCozinha`.

Um `Restaurante` também possui um `User`, que representa seu dono. O `User` também é utilizado pelo administrador do Caelum Eats. Um `User` tem um ou mais `Role`.

Uma `FormaDePagamento` existe independentemente de um `Restaurante` e os valores possíveis são mantidos pelo administrador. O mesmo ocorre para `TipoDeCozinha`.

Há algumas classes interessantes como `MediaAvaliacoesDto` e `RestauranteComDistanciaDto`, que associam um restaurante à média das notas das avaliações e a uma distância a um dado CEP, respectivamente.

![Agrupando agregados no Caelum Eats](imagens/02-decompondo-o-monolito/agregados-do-caelum-eats.png)

Poderíamos alinhar o código do Caelum Eats com o domínio, utilizando os agregados identificados anteriormente.

## Contextos Delimitados (Bounded Contexts)

Um foco importante do DDD é na linguagem. Os **especialistas de domínio** usam certos termos que devem ser representados nos requisitos, nos testes e, claro, no código de produção. A linguagem do negócio deve estar representada em código, no que chamamos de **Modelo de Domínio** (em inglês, _Domain Model_). Essa linguagem estruturada em torno do domínio e usada por todos os envolvidos no desenvolvimento do software é chamada pelo DDD de **Linguagem Onipresente** (em inglês, _Ubiquitous Language_).

Porém, para uma aplicação de grande porte as coisas não são tão simples. Por exemplo, em uma aplicação de e-commerce, o que é um Produto? Para diferentes especialistas de domínio e desenvolvedores, um Produto tem diferentes significados.

Para os da Loja Online, um Produto é algo que tem preço, altura, largura e peso.
Para os do Estoque, um Produto é algo que tem uma quantidade em um inventário.
Para os do Financeiro, um Produto é algo que tem um preço e descontos.
Para os de Entrega, um Produto é algo que tem uma altura, largura e peso.

Até atributos de um Produto tem diferentes significados, dependendo do contexto. Para a Loja Online, o que interessa é a altura, largura e peso de um produto fora da caixa, que servem para um cliente saber se o item caberá na pia da sua cozinha ou em seu armário. Já para a Entrega, esses atributos devem incluir a caixa e vão influenciar nos custos de transporte.

Para aplicações maiores, manter apenas um Modelo de Domínio é inviável. A origem do problema está na linguagem utilizada pelos especialistas de domínio: não há só uma Linguagem Onipresente. Nessa situação, tentar unificar o Modelo de Domínio o tornará inconsistente.

A linguagem utilizada pelos especialistas de domínio está atrelada a uma área de negócio. Há um contexto em que um Modelo de Domínio é consistente, porque representa a linguagem de uma área de negócio. No DDD, é importante identificarmos esses **Contextos Delimitados** (em inglês, _Bounded Contexts_), para que não haja degradação dos _vários_ Modelos de Domínio.

> **CONTEXTO DELIMITADO**
>
> Aplicabilidade delimitada de um determinado modelo. CONTEXTOS DELIMITADOS dão aos membros de uma equipe um entendimento claro e compartilhado do que deve ser consistente e o que pode se desenvolver independentemente.
>
> [Domain Driven Design](https://www.amazon.com/Domain-Driven-Design-Tackling-Complexity-Software/dp/0321125215) (EVANS, 2003)

No fim das contas, ao alinhar as linguagens utilizadas nas áreas de negócio aos modelos representados em código, estamos caminhando na direção de uma velha promessa: _alinhar TI com o Negócio_.

No Caelum Eats, podemos verificar como a empresa é organizada para encontrar os Contextos Delimitados e influenciar na organização do nosso código.

Digamos que a Caelum Eats tem, atualmente, as seguintes áreas de negócio:

- _Administrativo_, que mantém os cadastros básicos como tipos de cozinha e formas de pagamento aceitas, além de aprovar novos restaurantes
- _Pedido_, que acompanha e dá suporte aos pedidos dos clientes
- _Pagamento_, que cuida da parte Financeira e está explorando novos meios de pagamento como criptomoedas e QR Code
- _Distância_, que contém especialistas em geoprocessamento
- _Restaurante_, que lida com tudo que envolve os donos do restaurantes

Esses seriam os Contextos Delimitados, que definem uma fronteira para um Modelo de Domínio consistente.

Poderíamos reorganizar o código para que a estrutura básica de pacotes seja parecida com a seguinte:

```txt
eats
├── administrativo
├── distancia
├── pagamento
├── pedido
└── restaurante
```

![Reagrupando agregados inspirado por Contextos Delimitados](imagens/02-decompondo-o-monolito/reagrupando-agregados-do-caelum-eats-inspirado-por-contextos-delimitados.png)

Há ainda o código de Segurança, um domínio bastante técnico que cuida de um requisito transversal (ou não-funcional), utilizado por todas as outras partes do sistema.

Poderíamos incluir um pacote responsável por agrupar o código de segurança:

```txt
eats
├── administrativo
├── distancia
├── pagamento
├── pedido
├── restaurante
└── seguranca
```

<!--@note

Costumo a desenhar a arquietura em 3 camadas (layers) como uma maneira horizontal de "fatiar" o código.

Já agrupar por contextos delimitados seria uma maneira vertical.

Martin Fowler fala sobre isso no texto PresentationDomainDataLayering e descreve que a maneira vertical seria mais interessante pra aplicações maiores:
https://martinfowler.com/bliki/PresentationDomainDataLayering.html

-->


## Nem tudo precisa ser público

Em Java, uma classe pode ter dois modificadores de acesso: `public` ou _default_, o padrão, quando não há um modificador de acesso.

Uma classe `public` pode ser acessada por classes de qualquer outro pacote.

Já no caso de classes com modificador de acesso padrão, só podem ser acessadas por outras classes do mesmo pacote. É o que alguns chamam de _package-private_.

No caso de atributos, métodos e construtores, além de `public` e _default_, há os modificadores `private`, que restringe acesso a própria classe, e `protected`, que restringe ao mesmo pacote ou classes filhas mesmo se estiverem em outros pacotes.

Pense nos projetos Java em que você trabalhou: quantas vezes você criou uma classe que não é pública?

<!--@note

Essa ideia de agrupar classes por domínio e não colocar public é bem propagada por Simon Brown, que tem uma palestra interessante sobre o tema com a mesma pegada:

http://www.codingthearchitecture.com/presentations/sa2015-modular-monoliths

-->

Provavelmente, é uma implicação de organizarmos o código usando Package by Layer. Como o Controller está em um pacote, a entidade em outro e o Repository em outro ainda, precisamos tornar as classes públicas.

Uma outra coisa que nos "empurra" na direção de todas as classes serem públicas são as IDEs: o Eclipse, por exemplo, coloca o `public` por padrão.

Porém, se alinharmos os pacotes ao domínio, passamos a ter a opção de deixar as classes acessíveis apenas no pacote em que são definidas.

Por exemplo, podemos agrupar no pacote `br.com.caelum.eats.pagamento` as seguintes classes relacionadas ao contexto de Pagamento:

####### br.com.caelum.eats.pagamento

```txt
.
├── PagamentoController.java
├── PagamentoDto.java
├── Pagamento.java
└── PagamentoRepository.java
```

Algumas classes como `FormaDePagamento` e `Restaurante` são usadas por classes de outros pacotes. Essas devem ser deliberadamente tornadas públicas.

Há o caso da classe `DistanciaService`, que utiliza diretamente `RestauranteRepository`. Ou seja, temos uma classe de um contexto (distância) usando um detalhe de BD de outro contexto (restaurante). É interessante manter `RestauranteRepository` com o modificador padrão e criar uma classe `RestauranteService`, responsável por expôr funcionalidades para outros pacotes.

## Exercício opcional: decomposição em pacotes

1. Baixe, via Git, o projeto do monólito decomposto em pacotes:

  ```sh
  cd ~/Desktop
  git clone https://gitlab.com/aovs/projetos-cursos/fj33-eats-pacotes.git
  ```

2. Crie um novo workspace no Eclipse, clicando em _File > Switch Workspace > Other_. Defina o workspace `/home/<usuario-do-curso>/workspace-pacotes`, onde `<usuario-do-curso>` é o login do curso.
3. Acesse _File > Import > Existing Maven Projects_ e clique em _Next_. Em _Root Directory_, aponte para o diretório clonado no passo anterior.
4. Acesse a classe `EatsApplication` e a execute com _CTRL+F11_.
5. Certifique-se que o projeto `fj33-eats-ui` esteja sendo executado. Acesse `http://localhost:4200` e teste algumas das funcionalidades. Tudo deve funcionar como antes!
6. Analise o projeto. Veja quais classes e interfaces são públicas e quais são _package private_. Observe as dependências entre os pacotes.

## Módulos

No livro [Java Application Architecture: Modularity Patterns](https://www.amazon.com.br/Java-Application-Architecture-Modularity-Patterns/dp/0321247132) (KNOERNSCHILD, 2012), Kirk Knoernschild descreve módulos como artefatos que contém as seguintes características:

- **Implantáveis**: são entregáveis que podem ser executados em _runtime_
- **Reusáveis**: são nativamente reusáveis por diferentes aplicações, sem a necessidade de comunicação pela rede. As funcionalidade de um módulo são invocadas diretamente, dentro da mesma JVM e, portanto, do mesmo processo (no Windows, o mesmo `java.exe`).
- **Testáveis**: podem ser testados independentemente, com testes de unidade.
- **Gerenciáveis**: em um sistema de módulos mais elaborado, como OSGi, podem ser instalados, reinstalados e desinstalados.
- **Sem estado**: módulos não mantém estado, apenas suas classes.
- **Unidades de Composição**: podem se unir a outros módulos para compor uma aplicação.

![Características de módulos {w=47}](imagens/02-decompondo-o-monolito/caracteristicas-de-modulos.png)

Qual será o artefato Java que contém todas essas características?

É o JAR, ou **J**ava **AR**chive.

JARs são arquivos compactados no padrão ZIP que contém pacotes que, por sua vez, contém os `.class` compilados a partir do código fonte das classes.

Um JAR é implantável, reusável, testável, gerenciável, sem estado e é possível compô-lo com outros JARs para formar uma aplicação.  

### Porque modularizar?

No livro [Modular Java](https://pragprog.com/book/cwosg/modular-java) (WALLS, 2009), Craig Walls cita algumas vantagens de modularizar uma aplicação:

- capacidade de trocar um módulo por outro, com uma implementação diferente, desde que a interface pública seja mantida
- facilidade de compreensão de cada módulo individualmente
- possibilidade de desenvolvimento em paralelo, permitindo que tarefas sejam divididas entre diferentes times
- testabilidade melhorada, permitindo um outro nível de testes, que trata um módulo como uma unidade
- flexibilidade, permitindo o reuso de módulos em outras aplicações

## Módulos Maven

Com o Maven, é possível criarmos um **multi-module project**, que permite definir vários módulos em um mesmo projeto. O Maven ficaria responsável por obter as dependências necessárias e o fazer _build_ na ordem correta. Os artefatos gerados (JARs, WARs e/ou EARs) teriam a mesma versão.

Devemos definir um módulo pai, ou supermódulo, que contém um ou mais módulos filhos, ou submódulos.

No caso do Caelum Eats, teríamos um supermódulo `eats` e submódulos para cada contexto delimitado identificado anteriormente.

Além disso, precisaríamos de um submódulo que depende de todos os outros submódulos e conteria a classe principal, que possui o `main` e está anotada com `@SpringBootApplication`. Dentro desse submódulo, que chamaremos de `eats-application`, teríamos em `src/main/resources` o arquivo `eats-application` e as migrations do Flyway.

A estrutura de diretórios seria a seguinte:

```txt
eats
│
├── pom.xml
│
├── eats-application
│   ├── pom.xml
│   └── src
│
├── eats-administrativo
│   ├── pom.xml
│   └── src
│
├── eats-distancia
│   ├── pom.xml
│   └── src
│
│
├── eats-pagamento
│   ├── pom.xml
│   └── src
│
│
├── eats-pedido
│   ├── pom.xml
│   └── src
│
├── eats-restaurante
│   ├── pom.xml
│   └── src
│
└── eats-seguranca
    ├── pom.xml
    └── src
```

O supermódulo `eats` deve definir um `pom.xml`.  Nesse arquivo, a propriedade `packaging` deve ter o valor `pom`. Podem ser definidas propriedades, dependências, repositórios e outras configurações comuns a todos os submódulos. No nosso caso, definiríamos no supermódulo a dependência ao Lombok.

Os submódulos disponíveis devem ser declarados da seguinte maneira:

```xml
<modules>
  <module>eats-application</module>
  <module>eats-administrativo</module>
  <module>eats-distancia</module>
  <module>eats-pagamento</module>
  <module>eats-pedido</module>
  <module>eats-restaurante</module>
  <module>eats-seguranca</module>
</modules>
```

Já os submódulos não devem definir um `<groupId>` ou `<version>` próprios, apenas o `<artifactId>`. Devem declarar qual é o seu supermódulo com a tag `<parent>`. Segue o exemplo para o submódulo de restaurante:

```xml
<parent>
  <groupId>br.com.caelum</groupId>
  <artifactId>eats</artifactId>
  <version>0.0.1-SNAPSHOT</version>
</parent>

<artifactId>eats-restaurante</artifactId>
```

Os arquivos `pom.xml` dos submódulos podem definir em seus `pom.xml` suas próprias configurações, como propriedades, dependências e repositórios.

Se houver uma dependência a outros submódulos, é possível usar `${project.version}` como versão. Por exemplo, o submódulo `eats-restaurante` deve declarar a dependência ao submódulo `eats-administrativo` da seguinte maneira:

```xml
<dependency>
  <groupId>br.com.caelum</groupId>
  <artifactId>eats-administrativo</artifactId>
  <version>${project.version}</version>
</dependency>
```

É interessante notar que, nos arquivos `pom.xml`, há uma _materialização em código_ da estrutura de dependências entre os módulos do sistema. Observando as dependências declaradas entre os módulos, podemos montar um diagrama parecido com o seguinte:

![Dependências entre módulos do Caelum Eats](imagens/02-decompondo-o-monolito/dependencias-dentre-modulos-do-caelum-eats.png)

O submódulo `eats-application` depende de todos os outros e contém a classe principal, que contém o `main`. Ao executarmos o build, com o comando `mvn clean package` no diretório do supermódulo `eats`, o _Fat JAR_ do Spring Boot é gerado no diretório _target_ do `eats-application`, contendo o código compilado da aplicação e de todas as bibliotecas utilizadas.

<!-- TODO: 
  discutir sobre as dependências dos módulos spring-boot-starter-data-jpa, spring-boot-starter-web e spring-boot-starter-validation
  discutir duplicação, módulo common/util e ResourceNotFoundException
  mencionar ArchUnit? https://blogs.oracle.com/javamagazine/unit-test-your-architecture-with-archunit  
-->

## Exercício: o monólito modular

1. Clone o projeto com a decomposição do monólito em módulos Maven:

  ```sh
  cd ~/Desktop
  git clone https://gitlab.com/aovs/projetos-cursos/fj33-eats-monolito-modular.git
  ```

2. Crie o novo workspace `/home/<usuario-do-curso>/workspace-monolito-modular` no Eclipse, clicando em _File > Switch Workspace > Other_. Troque `<usuario-do-curso>` pelo login do curso.
3. Importe, pelo menu  _File > Import > Existing Maven Projects_ do Eclipse, o projeto `fj33-eats-monolito-modular`.
4. Para executar a aplicação, acesse o módulo `eats-application` e execute a classe `EatsApplication` com _CTRL+F11_. Certifique-se que as versões anteriores do projeto não estão sendo executadas.
5. Com o projeto `fj33-eats-ui` no ar, teste as funcionalidades por meio de `http://localhost:4200`. Deve funcionar!
6. Observe os diferentes módulos Maven. Note as dependências entre esses módulos, declaradas nos `pom.xml` de cada módulo.
