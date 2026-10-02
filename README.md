# Checkpoint 5 — Bug Hunt PetFiap

> Copie este arquivo para a raiz do seu repositório com o nome **README.md**
> e preencha todas as seções.

## Identificação

**Grupo:** Arthur

| Integrante | RM | Turma |
|---|---|---|
|Arthur Reis|562181|2CCPW|

| Campo | |
|---|---|
| **Total de bugs corrigidos** | 12 / 12 |
| **Total de ajustes de Clean Code** | 6 / 6 |
| **Total de testes novos escritos** | 6 / 6 |
| **Suíte final (Run As → JUnit Test)** | ___ testes, ___ falhas |

---

## Parte 1 — Bugs encontrados

> Uma linha por bug, na ordem em que você os encontrou. Use a numeração dos seus
> commits (`fix: bug01 ...`). Preencha TODAS as colunas — metade da nota está aqui.

| # | Sintoma observado (o que fiz/vi) | Causa raiz (arquivo e linha aproximada) | Correção aplicada | Conceito da disciplina |
|---|---|---|---|---|
| bug01 | `AtendimentoFactoryTest.deveCriarTosaQuandoTipoForTosa` vermelho: pedi um atendimento do tipo `"TOSA"` e veio um objeto `Banho` (esperado `Tosa`) | `AtendimentoFactory.java`, linha 17: o `case "TOSA"` instanciava `new Banho(...)` | Troquei para `new Tosa(...)` no `case "TOSA"` | Padrão Factory (Aula 14) e polimorfismo: a Factory é o único lugar que conhece as subclasses concretas, então um `case` errado faz todo o sistema tratar uma tosa como banho (preço, pontos e duração errados) |
| bug02 | `TosaTest.deveDurar60Minutos` (teste01) vermelho: `expected: <60> but was: <30>`, a Tosa durava os 30 min padrão em vez de 60 | `Tosa.java`, linha 40: o método era `getDuracaoMinutos(String porte)`, com um parâmetro a mais, então não sobrescrevia o `getDuracaoMinutos()` do `Atendimento` | Tirei o parâmetro `String porte` e acrescentei `@Override`, deixando a assinatura igual à da classe-mãe | Sobrescrita vs sobrecarga (Aula 7): com a assinatura diferente, o método virou uma sobrecarga e o polimorfismo chamava o da classe-mãe. O `@Override` faz o compilador acusar esse erro |
| bug03 | `BanhoTest.deveCustar60ReaisParaPortePequeno` (teste02) vermelho: `expected: <60.0> but was: <100.0>`, o banho de porte pequeno saía pelo preço do grande | `Banho.java`, linhas 28 e 32: no `calcularPreco()`, PEQUENO retornava 100.0 e o caso padrão (GRANDE) retornava 60.0, com os valores invertidos | Troquei os valores: PEQUENO → 60.0 e GRANDE → 100.0 (MEDIO continua 80.0), conforme a tabela do contrato | Regra de negócio no model e polimorfismo (cada subclasse calcula o próprio preço); testes unitários (Aula 15) como contrato: a regra não tinha cobertura e o erro só apareceu com o teste novo |
| bug04 | `AtendimentoBuilderTest.deveMontarAtendimentoCompleto` vermelho: `expected: <Rex> but was: <null>`, o atendimento montado pelo Builder saía sem o nome do pet | `AtendimentoBuilder.java`, linha 24: no `comPet`, `petNome = petNome;` atribuía o parâmetro a ele mesmo, sem o `this`, então o atributo da classe nunca era preenchido | Troquei para `this.petNome = petNome;` | Encapsulamento e o uso do `this` (POO): quando o parâmetro tem o mesmo nome do atributo, ele "esconde" o atributo (shadowing), e só o `this.` acessa o campo do objeto |
| bug05 | `AgendaServiceTest.deveRecusarAgendamentoComHorarioJaOcupado` vermelho: o Rex já tinha banho AGENDADO no mesmo horário, mas o segundo agendamento passava sem `HorarioOcupadoException` | `AgendaService.java`, linha 23: a verificação de conflito comparava `getPetNome()` (String) e `getDataHora()` (LocalDateTime) com `==`, que compara referências e não valores | Troquei os dois `==` por `.equals()` | `==` vs `.equals()` (Aula 7): `==` em objetos compara se são o mesmo objeto na memória. Com `"Rex"` funcionava por sorte (literais de String ficam no pool e viram a mesma referência), mas a data vinda de outro objeto (`LocalDateTime.parse`) tinha o mesmo valor em outra referência |
| bug06 | `AtendimentoBuilderTest.deveRecusarMontagemSemNomeDoPet` e `deveRecusarMontagemSemPorte` vermelhos: os dois esperavam `IllegalArgumentException`, mas o Builder montava o atendimento normalmente com nome ou porte `null` | `AtendimentoBuilder.java`, linha 41: o `construir` repassava os campos direto para a `AtendimentoFactory`, sem nenhuma validação | No começo do `construir`, verifico se `petNome` ou `petPorte` são `null` e, se forem, lanço `IllegalArgumentException` antes de chamar a Factory | Padrão Builder (Aula 14) e exceções: o `construir` é o último passo da montagem, então é ali que o objeto precisa ser validado para só nascer válido, mesmo que o `comPet` nunca seja chamado. |
| bug07 | `AgendaServiceTest.deveRecusarCancelamentoDeAtendimentoJaConcluido` (teste04) vermelho: esperava `StatusInvalidoException` ao cancelar um atendimento já CONCLUIDO, mas nada era lançado e o atendimento era salvo como CANCELADO | `Atendimento.java`, linha 63: o `cancelar()` só fazia `status = "CANCELADO"`, sem olhar o status atual (o `concluir()`, logo acima, já fazia essa verificação) | No começo do `cancelar()`, se o status não for `"AGENDADO"`, lanço `StatusInvalidoException`; só depois troco o status para `"CANCELADO"` | Encapsulamento e exceções personalizadas: a regra de transição de status fica dentro do próprio `Atendimento`, que protege o seu estado, em vez de depender de quem chama. O service só repassa a exceção e, por isso, não chega a salvar |
| bug08 | `AgendaServiceTest.deveRecusarAgendamentoComHorarioNoPassado` (teste06) vermelho: esperava `IllegalArgumentException` ao agendar um banho para ontem, mas o `agendar` consultava o banco e seguia até o `save` (no teste, estourava `NullPointerException` no recibo, porque o mock devolve `null`) | `AgendaService.java`, linha 20: o `agendar` começava direto no `repository.findByPetNome(...)`, sem nenhuma verificação da data do atendimento novo | Na primeira linha do `agendar`, antes de tocar no repository, comparo `LocalDateTime.now()` com `novo.getDataHora()` usando `isAfter`; se a data já passou, lanço `IllegalArgumentException` | Validação de regra de negócio na camada de service e exceções (falhar cedo): a entrada é validada antes de qualquer acesso ao banco, e por isso o teste consegue verificar com `verify(repository, never())` que nada foi consultado nem salvo (mock, Aula 15). Datas são objetos, então a comparação é com `isAfter`/`isBefore`, não com `<` ou `>` |
| bug09 | `AtendimentoFactoryTest.devePreencherOsDadosDoPetNaConsulta` vermelho: `expected: <Mimi> but was: <null>`, a consulta criada pela Factory vinha sem nome do pet, porte e tutor | `ConsultaVeterinaria.java`, linha 17: o construtor com argumentos recebia os cinco dados mas chamava `super()` vazio, então o construtor sem argumentos do `Atendimento` rodava e nada era guardado (nem o status inicial `"AGENDADO"`) | Troquei `super()` por `super(protocolo, petNome, petPorte, tutorNome, dataHora)`, igual ao que `Banho` e `Tosa` já faziam | Herança e construtores: os atributos são `private` no `Atendimento`, então a subclasse só consegue preenchê-los repassando os argumentos ao construtor da classe-mãe com `super(...)`. O `super()` vazio compila sem erro porque a classe-mãe também tem um construtor sem argumentos |
| bug10 | `GeradorProtocoloTest.deveManterUmaUnicaInstancia` e `deveGerarProtocolosSequenciais` vermelhos: duas chamadas a `getInstancia()` devolviam objetos diferentes, e o protocolo vinha sempre 1 (`expected: <2> but was: <1>`) em vez de 1, 2, 3 | `GeradorProtocolo.java`, linha 19: dentro do `if (instancia == null)`, o método fazia `return new GeradorProtocolo();` sem guardar o objeto no atributo estático `instancia`, que continuava `null` para sempre | Troquei o `return new GeradorProtocolo();` por `instancia = new GeradorProtocolo();`, deixando o `return instancia;` do fim do método devolver sempre o mesmo objeto | Padrão Singleton (Aula 14) e atributo `static`: a instância única só existe se for guardada no atributo da classe. Sem isso, cada chamada criava um gerador novo com o contador zerado, e todos os atendimentos sairiam com protocolo 1 |
| bug11 | `AgendaServiceTest.deveLancarExcecaoQuandoAtendimentoNaoExiste` vermelho: `Expected AtendimentoNaoEncontradoException to be thrown, but nothing was thrown`, a busca por um id inexistente devolvia `null` em vez de lançar a exceção | `AgendaService.java`, linhas 37 a 43: no `buscarPorId`, o `orElseThrow` lançava a exceção certa, mas estava dentro de um `try` cujo `catch (Exception e)` capturava essa mesma exceção e fazia `return null` | Removi o `try/catch`, deixando só o `return repository.findById(id).orElseThrow(...)`, para a exceção subir até quem chamou | Tratamento de exceções: um `catch (Exception e)` genérico que devolve `null` "engole" o erro e esconde o problema. Sem a exceção, o controller nunca chegava no `catch` que responde 404, e o `concluir`/`cancelar` quebravam com `NullPointerException` ao usar o atendimento nulo |
| bug12 | Com a suíte toda verde, subi a API e fiz o `POST /api/atendimentos` do enunciado com curl: em vez de 201 com `"id": 1`, a resposta era erro 500, e o console mostrava a falha do Hibernate ao salvar um atendimento sem identificador. Nenhum teste acusava, porque todos usam mock no lugar do banco | `Atendimento.java`, linha 14: o atributo `id` tinha só `@Id`, sem nenhuma estratégia de geração, então o JPA esperava que o próprio código preenchesse o id antes do `save`, e ninguém preenchia | Acrescentei `@GeneratedValue(strategy = GenerationType.IDENTITY)` no `id`, para o banco gerar a chave a cada novo atendimento (e recriei a tabela, que tinha sido criada sem a coluna de identidade) | Mapeamento de entidades com JPA/Hibernate (Aulas 12 e 13): `@Id` só marca a chave primária; quem define como ela é gerada é o `@GeneratedValue`. Mostra também o limite do teste unitário com mock (Aula 15): ele valida a regra do service, mas não o mapeamento com o banco, que só aparece rodando a aplicação |

## Parte 2 — Ajustes de Clean Code

| # | Onde estava | Qual princípio/boas práticas era violado | O que eu mudei |
|---|---|---|---|
| clean01 | `AtendimentoController` — método privado `calcularDescontoFidelidade` e comentário "Fidelidade (futuro)" no fim da classe | Código morto / YAGNI: método nunca chamado, escrito "para o futuro", que só polui a classe e confunde quem lê | Removi o método e o comentário; se a regra de fidelidade for aprovada, ela é implementada quando for necessária (e o histórico do git guarda a versão antiga) |
| clean02 | `AtendimentoFactory.criar` — parâmetros `p, t, n, po, tu, d` | Nomes significativos: abreviações de 1–2 letras não dizem o que guardam e obrigam quem lê a decifrar cada uma (`po` é porte? `tu` é tutor?) | Renomeei para `protocolo, tipo, petNome, petPorte, tutorNome, dataHora`, os mesmos nomes usados no resto do projeto |
| clean03 | `AgendaService.agendar` — `System.out.println("Recibo: atendimento ...")` logo depois do `repository.save` | Responsabilidade única e saída no lugar errado: o service cuida da regra de agendamento, não de imprimir recibo. Um `System.out.println` em código de produção só escreve no console do servidor, onde o cliente da API nunca vê, e ainda expõe nome do pet e do tutor | Removi o `println`; o `agendar` agora só salva e devolve o atendimento, e quem chama a API recebe esses mesmos dados no corpo da resposta |
| clean04 | `GeradorProtocolo` — `System.out.println("GeradorProtocolo criado!")` dentro do construtor privado | Rastro de depuração esquecido no código: a linha só servia para o programador ver no console que o objeto tinha nascido. Um construtor deve apenas inicializar o objeto, sem efeito colateral de escrever na saída padrão | Removi o `println`; o construtor ficou só com `contador = 0`. A garantia de instância única é verificada pelos testes do `GeradorProtocoloTest`, não por uma mensagem no console |
| clean05 | `Atendimento` (construtor, `concluir` e `cancelar`) e `AgendaService.agendar` — os textos `"AGENDADO"`, `"CONCLUIDO"` e `"CANCELADO"` escritos direto no código, seis vezes nos dois arquivos | Strings mágicas / DRY: o mesmo literal repetido em vários lugares. Um erro de digitação em um deles (`"AGENDAD0"`) compila normalmente e só quebra a regra de status em tempo de execução, sem nenhum aviso | Criei as constantes `STATUS_AGENDADO`, `STATUS_CONCLUIDO` e `STATUS_CANCELADO` no `Atendimento` (no mesmo estilo do `TIPO` que `Banho`, `Tosa` e `ConsultaVeterinaria` já usavam) e troquei todos os literais por elas; agora um nome errado vira erro de compilação |
| clean06 | `AtendimentoBuilder` — comentário acima do `construir` dizendo que "a validação dos campos obrigatórios fica por conta do controller"; `GeradorProtocolo` — comentário "Thread-safe para o uso concorrente do pet shop" no topo da classe | Comentários enganosos: um comentário que não bate com o código é pior do que nenhum, porque quem lê confia nele. O controller nunca validou nada (e a validação agora está no próprio `construir`), e o Singleton não tem `synchronized` nem outro mecanismo que o torne seguro para threads | Removi os dois comentários. O que o código faz já fica claro pelo próprio código (o `if` no começo do `construir`), e não deixei nenhuma promessa que a classe não cumpre |

## Parte 3 — Testes novos (regras que estavam sem cobertura)

> Uma linha por teste novo (`test: ...`). "Regra coberta" é o comportamento do
> contrato (seção 3 do enunciado) que o teste protege. Em "Resultado", diga se o
> teste ficou vermelho ao ser escrito (revelou bug — qual?) ou verde de cara
> (regra já estava correta).

| # | Teste escrito (classe.método) | Regra coberta | Resultado ao escrever (vermelho/verde) |
|---|---|---|---|
| teste01 | `TosaTest.deveDurar60Minutos` | Tosa dura 60 minutos (tabela de preço, pontos e duração) | **Vermelho** - revelou que o `getDuracaoMinutos(String porte)` da Tosa é uma sobrecarga, não uma sobrescrita, então valia a duração padrão do `Atendimento` |
| teste02 | `BanhoTest.deveCustar60ReaisParaPortePequeno` | Banho de porte pequeno custa R$ 60,00 (tabela de preço, pontos e duração) | **Vermelho** - revelou que, no Banho, os valores de porte pequeno e grande estão invertidos |
| teste03 | `ConsultaVeterinariaTest.deveCustar150Reais` | Consulta custa R$ 150,00 fixo, independente do porte (tabela de preço, pontos e duração) | **Verde** |
| teste04 | `AgendaServiceTest.deveRecusarCancelamentoDeAtendimentoJaConcluido` | Cancelar um atendimento já CONCLUIDO é recusado com `StatusInvalidoException` e nada é salvo (regras de status) | **Vermelho** - revelou que o `cancelar()` do `Atendimento` não valida o status |
| teste05 | `AgendaServiceTest.deveCancelarAtendimentoAgendado` | Cancelar um atendimento AGENDADO muda o status para CANCELADO e salva (regras de status) | **Verde** |
| teste06 | `AgendaServiceTest.deveRecusarAgendamentoComHorarioNoPassado` | Agendar com data/hora no passado é recusado com `IllegalArgumentException` e o banco nem é consultado (regras de agendamento) | **Vermelho** - revelou que o `agendar()` do `AgendaService` não valida a data: consulta o banco e tenta salvar um atendimento no passado |

---

## Parte 4 — Perguntas de reflexão

> Responda com suas palavras, 5 a 10 linhas cada, **usando o código real do
> projeto como exemplo**. Respostas genéricas de tutorial não pontuam.

### 1. A suíte como contrato (Aula 15)
O projeto chegou com 20 testes, 9 vermelhos. Descreva como você usou as
mensagens de falha (ex.: `expected: <Rex> but was: <null>`) para caçar os bugs.
O que a suíte de testes tem de melhor do que testar tudo na mão com curl?

### 2. Mock e injeção de dependência (Aulas 13 a 15)
No `AgendaServiceTest`, o `@Mock` cria um `AtendimentoRepository` falso e o
`@InjectMocks` o injeta no service. Explique a relação disso com o `@Autowired`
que o Spring faz em produção — quem "injeta" em cada mundo, e por que o teste
consegue rodar sem banco e sem subir o Spring?

### 3. `==` vs `.equals()` (Aula 7)
Um dos bugs fazia o agendamento duplicado passar pela verificação de conflito.
Explique por que `==` entre Strings e `LocalDateTime` falhou aqui, por que ele
"funciona por sorte" com literais como `"Rex"`, e o que a sua correção mudou.

### 4. Sobrescrita vs sobrecarga (Aula 7)
Um dos bugs compilava sem nenhum erro: um método parecia sobrescrever
`getDuracaoMinutos`, mas na verdade criava uma assinatura nova. Explique a
diferença entre override e overload nesse caso e por que a anotação `@Override`
teria impedido o bug.

### 5. Singleton manual vs bean do Spring (Aula 14)
O `GeradorProtocolo` é um Singleton escrito à mão e causou um dos bugs.
Explique o que ele garante, qual foi o bug, e por que o `AgendaService`
(`@Service`) não corre o mesmo risco no container do Spring.

### 6. Cobertura de testes: onde parar? (Aula 15)
Dos 6 testes novos que você escreveu, alguns ficaram vermelhos (revelaram
bugs) e outros verdes de cara (regras já corretas). Vale a pena manter os que
ficaram verdes? Em um projeto real com prazo, o que você priorizaria testar:
caminho feliz, caminhos de erro, ou 100% de cobertura? Justifique.

---

## Parte 5 — Espaço livre (opcional)

Alguma dificuldade, dúvida ou comentário sobre o checkpoint?

```

```
