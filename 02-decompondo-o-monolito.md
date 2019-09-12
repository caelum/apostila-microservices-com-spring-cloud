# Decompondo o monólito

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
