# Decompondo o monólito

## Da bagunça a camadas

Muitos desenvolvedores associam a palavra **monólito** a código mal feito, sem estrutura aparente, com muito código repetido e com dados compartilhados entre partes pouco relacionadas. É o que comumente chamado de código **Spaghetti** ou, [Big Ball of Mud](http://www.laputan.org/mud/mud.html) (FOOTE; YODER, 1999).

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

### Clean Architecture

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

### Agregado

Todo domínio tem entidades importantes, que são um ponto de entrada para o negócio e tem várias outras entidades ligadas. Um objeto que representa um pedido, por exemplo, terá informações dos itens do pedido, do cliente, da entrega e do pagamento. É o que é chamado, nos termos do DDD, de _Agregado_.

> **AGREGADO**
>
> Agrupamento de objetos associados que são tratados como uma unidade para fiz de alterações nos dados. Referências externas são restritas a um único membro do AGREGADO, designado como _raiz_. Um conjunto de regras de consistência se aplicada dentro dos limites do AGREGADO.

Referências a outros objetos, de fora desse Agregado, devem ser feitas por meio do objeto principal, a raiz do Agregado. Uma relação entre um pedido e um restaurante deve ser feita pelo pedido, e não pela entrega ou o pagamento.

Os dados de um Agregado são alterados em conjunto. Por isso, Banco de Dados Orientados a Documentos, como o MongoDB, em que um documento pode ter um grafo de outros objetos conectados, são interessantes para persistir Agregados.

<!-- TODO: explorar agregados no Caelum Eats -->

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
