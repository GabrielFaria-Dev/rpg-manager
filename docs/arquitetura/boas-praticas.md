# Boas práticas – Backend RPG Manager (Spring Boot)

Guia a ser seguido por todo o código do backend. Complementa o [`arquitetura.md`](arquitetura.md),
que continua sendo o guia oficial de **estrutura**. Este arquivo trata de **como escrever** o código.

Origem: adaptado de um guia de boas práticas de outro projeto (Node.js), que se baseia no
[nodebestpractices](https://github.com/goldbergyoni/nodebestpractices) e no guia de segurança do
Node, traduzido para Java/Spring. Decisão registrada em
[`ADR-006`](../decisoes/ADR-006-boas-praticas-e-seguranca.md).

Legenda:
- ⏳ = **planejado**. A regra vale quando o recurso for construído; não crie nada antes da hora.
- 🔶 = **decisão em aberto**. Precisa de ADR antes de implementar.

---

## 1. Camadas e dependências entre domínios

A estrutura de pastas está no `arquitetura.md` (seção 5). As regras de uso:

- **A dependência só desce:** `controller → service → repository`. O repository nunca chama o
  service e o service nunca chama o controller.
- **Objetos HTTP não saem do controller.** `HttpServletRequest`, `ResponseEntity` e cabeçalhos ficam
  no controller. O service recebe dados simples (IDs, DTOs, o usuário autenticado).
  *Por quê:* o service pode ser testado e reaproveitado sem simular uma requisição.
- **Um domínio usa outro pelo service, nunca pelo repository alheio.** Exemplo: `sessao` precisa de um
  personagem → chama `PersonagemService`, não `PersonagemRepository`.
  *Por quê:* as regras de `personagem` (incluindo quem pode ver o quê) ficam num lugar só.
- **Injeção por construtor**, com campos `final`. Nada de `@Autowired` em campo.
  *Por quê:* deixa explícito do que a classe depende e facilita testar sem o Spring.
- `shared/` só recebe o que **dois ou mais domínios** usam.

## 2. DTO × Entidade

- **A entidade nunca sai da API.** O controller recebe um `...Request` e devolve um `...Response`.
  *Por quê:* evita vazar campos (senha, dados privados de outros jogadores) e permite mudar o banco
  sem quebrar o desktop.
- DTOs são `record`s do Java (imutáveis e curtos).
- Um Response por **visão**: o Mestre e o dono veem a ficha completa, os outros jogadores veem um
  Response reduzido (ver seção 7 do `arquitetura.md`). A privacidade é garantida pelo **tipo** que o
  servidor monta, não por esconder campos no desktop.

## 3. Validação de entrada ⏳

Entra junto com o primeiro DTO de entrada (domínio `usuario`), adicionando
`spring-boot-starter-validation`.

- **Valide toda entrada na borda** com Bean Validation (`@NotBlank`, `@Size`, `@Positive`, `@Email`…)
  e `@Valid` no controller. O service recebe dados já válidos no formato.
- Validação de **formato** fica no DTO. Validação de **regra** (ex.: "a sessão está aberta?") fica no
  service.
- Limite tamanhos de texto (`@Size(max = ...)`) em todo campo livre: nome, descrição etc.
- Rejeite campos desconhecidos no JSON (configuração do Jackson). *Por quê:* um cliente não consegue
  "contrabandear" campos como `tipo: MESTRE`.
- Valores de ação são **positivos e limitados** ("recebi 25 de dano" → `@Positive`); a direção (dano
  ou cura) é dada pelo endpoint, não por um número negativo.

## 4. Tratamento de erros ⏳

Entra quando o primeiro service lançar um erro de negócio.

- **Exceções próprias** em `shared/exception/`, com nome em português:
  `RecursoNaoEncontradoException` (404), `AcessoNegadoException` (403),
  `ConflitoException` (409), `RegraDeNegocioException` (422).
- **Um único handler central** (`@RestControllerAdvice`) traduz exceção → resposta HTTP.
  Services **lançam**; ninguém monta resposta de erro com `try/catch` espalhado.
- Formato único de erro: `ProblemDetail` (RFC 9457), já embutido no Spring, sem dependência nova.
- **Erro esperado** (validação, não encontrado, sem permissão) vira resposta HTTP com mensagem útil.
  **Bug** (qualquer outra exceção) é logado com detalhes e devolve **500 genérico**.
- **Nunca** devolva stack trace, SQL, nome de classe ou mensagem interna ao cliente
  (já fixado em `application.yml`, `server.error.*`).
- Nunca engula exceção em silêncio (`catch` vazio).

## 5. Design da API

- REST, recursos no plural e em português, sob `/api`: `/api/campanhas`, `/api/sessoes/{id}/eventos`.
- Status corretos: `201` criação, `204` sem corpo, `400` entrada inválida, `401` não autenticado,
  `403` sem permissão, `404` não encontrado, `409` conflito, `422` regra de negócio violada.
- Ações do jogo são endpoints de **ação** (`POST /api/sessoes/15/personagens/8/dano`), nunca um
  `PUT` com o valor final (regra 6.4 do `arquitetura.md`).
- ⏳ **Paginação** em toda listagem que cresce (eventos da sessão, histórico): `Pageable` com
  **tamanho máximo** fixo.
- ⏳ **Idempotência das ações:** se o desktop reenviar "recebi 25 de dano" por falha de rede, o dano
  não pode ser aplicado duas vezes. Mecanismo 🔶 a decidir quando os eventos forem construídos
  (sugestão: chave de idempotência enviada pelo cliente, com `UNIQUE` no `EventoSessao`).
- Versionamento (`/api/v1`) **não é adotado por enquanto**: o único cliente é o nosso desktop.
  Reavaliar se surgir um cliente externo.

## 6. Configuração e segredos

- Toda configuração em `application.yml`. **Segredos nunca no arquivo:** use variáveis de ambiente
  com placeholder, ex. `password: ${DB_PASSWORD}`.
- `.env`, chaves e dumps de banco **nunca** são versionados (já no `.gitignore`). Versione um
  `.env.example` com nomes e valores fictícios quando o primeiro segredo existir.
- ⏳ Configuração própria do projeto via `@ConfigurationProperties` + `@Validated`: se faltar um
  valor obrigatório, a aplicação **falha ao subir** com mensagem clara, em vez de quebrar depois.
- Diferenças por ambiente via *profiles* (`application-dev.yml`, `application-test.yml`), não via
  `if` no código.

## 7. Segurança

### 7.1 Autenticação ⏳ 🔶

Entra na etapa "autenticação" da ordem de construção, com Spring Security e um ADR próprio
(mecanismo: token JWT × sessão).

- **Senha:** só com `PasswordEncoder` do Spring Security (BCrypt ou Argon2). Nunca texto puro, nunca
  hash simples (MD5/SHA), nunca em log nem em Response.
- Login com mensagem de erro **genérica** ("e-mail ou senha inválidos"), sem dizer qual dos dois
  está errado. *Por quê:* não revela quais e-mails estão cadastrados.
- Token/sessão validado **no servidor em toda rota protegida**. Todas as rotas são protegidas por
  padrão; as públicas (login, cadastro, health) são liberadas uma a uma.
- 🔶 Limite de tentativas de login (rate limit / bloqueio temporário) decidido no mesmo ADR.
- 🔶 Revogação: logout precisa invalidar o token/sessão de verdade.

### 7.2 Autorização

Complementa a seção 7 do `arquitetura.md`.

- **O servidor decide, sempre.** O desktop esconder um botão é só conforto, não proteção.
- **Papel e posse do recurso**, verificados **no service**, em toda operação:
  - o jogador só altera o **próprio** personagem;
  - o jogador só vê dados privados (HP, barras, quedas) do **próprio** personagem;
  - o jogador só acessa sessões e campanhas **das quais participa**;
  - o Mestre só tem poder de Mestre **na campanha que mestra**, não em todas.
- **Quem é o usuário vem do token**, nunca do corpo ou da URL. Nunca aceite `usuarioId`, `tipo` ou
  `mestre: true` enviados pelo cliente.
- **Não confie em IDs da URL** sem checar a posse. Trocar `/personagens/8` por `/personagens/9` não
  pode dar acesso ao personagem de outra pessoa (falha conhecida como IDOR).
- Recurso de outra campanha → **404**, não 403. *Por quê:* não revela que o recurso existe.
- Correção direta de estado pelo Mestre é uma rota separada, só para o Mestre, e **também** gera
  `EventoSessao` (quem corrigiu, valor anterior, valor novo).

### 7.3 Proteções da aplicação

- Não desligue os cabeçalhos de segurança que o Spring Security já adiciona.
- **CORS desligado:** o cliente é um desktop JavaFX, não um navegador, então CORS não se aplica.
  Só liberar, com lista explícita de origens, se um cliente web surgir.
- 🔶 CSRF: depende do mecanismo de autenticação (com token no cabeçalho pode ser desligado; com
  cookie de sessão, fica ligado). Decidir no ADR de autenticação.
- Em produção, a API só é acessada por **HTTPS**. HTTP puro apenas em `localhost`.
- Nada de stack trace, versão do servidor ou detalhe interno nas respostas.

### 7.4 Dados pessoais (LGPD)

- Colete o mínimo: nome, e-mail e senha bastam para o usuário.
- **Nunca** registre em log: senha, token, e-mail completo.
- ⏳ Permitir exclusão da conta e dos dados do usuário.

## 8. Banco de dados ⏳

Entra na etapa "banco" da ordem de construção.

- Esquema **só por migrations versionadas** em `db/migration/` (Flyway, a justificar no `pom.xml`
  quando entrar). Nunca alterar o banco "na mão".
- `spring.jpa.hibernate.ddl-auto: validate`: o Hibernate **confere** o esquema, não o cria.
  *Por quê:* `update` muda o banco sem controle e sem histórico.
- `spring.jpa.open-in-view: false`: consultas ao banco ficam no service, dentro da transação, não
  escondidas na hora de montar o JSON.
- **Integridade também no banco:** `NOT NULL`, `UNIQUE`, `CHECK` e chaves estrangeiras. O código
  valida, mas o banco é a última garantia.
- **Consultas parametrizadas sempre.** Nunca concatene texto em `@Query` ou SQL nativo
  (`"... WHERE nome = '" + nome + "'"` permite injeção de SQL). Use `:parametro`.
- `@Transactional` no **service**, em toda operação que grava mais de uma tabela
  (ex.: alterar `EstadoPersonagemSessao` + criar `EventoSessao`: ou os dois gravam, ou nenhum).
- **Concorrência:** jogador e Mestre podem alterar o mesmo estado ao mesmo tempo. Use `@Version`
  (lock otimista) em `EstadoPersonagemSessao` e `EstadoCriaturaSessao`; conflito vira `409`.
- Índices nas colunas usadas em busca e junção (`sessao_id`, `personagem_id`, `campanha_id`).

## 9. Logs

- Logger SLF4J (`LoggerFactory.getLogger(...)`), já incluído no Spring. **Nunca** `System.out` ou
  `printStackTrace()`.
- Níveis coerentes: `debug` para detalhes de desenvolvimento, `info` para fatos relevantes, `warn`
  para situação estranha mas tratada, `error` para bugs.
- Sem dados sensíveis (seção 7.4).
- ⏳ Identificador por requisição (`requestId`) nos logs e nas respostas de erro.
- ⏳ `GET /ready` (verifica o banco) além do `/api/health` quando houver banco; avaliar o Actuator.

## 10. Código e estilo

- Indentação com tab; nomes de domínio em português; termos técnicos do Java/Spring em inglês.
- Métodos pequenos, **retorno antecipado** em vez de `if` aninhado.
- Comentários explicam o **porquê**, não o óbvio. Regras do RPG são citadas pela referência em
  `docs/regras/`.
- Sem código morto, sem `TODO` sem contexto, sem classes duplicadas.
- Dependência nova só com motivo concreto e justificativa (regra do `arquitetura.md`, seção 3).
  Versões ficam a cargo do `spring-boot-starter-parent` sempre que possível.

## 11. Testes

- Mesma estrutura de pacotes do código (`src/test/java/com/rpgmanager/<dominio>/...`).
- Nome do teste: **o que + em qual situação + resultado esperado**
  (`aplicarDano_quandoJogadorNaoEDono_retorna403`). Estrutura **Arrange–Act–Assert**.
- Cada teste cria **os próprios dados**; nada de dados compartilhados entre testes.
- **Teste obrigatório de acesso negado:** toda rota protegida tem pelo menos um caso
  "usuário sem permissão recebe 403/404" (jogador alterando personagem alheio, jogador vendo HP de
  outro, Mestre de outra campanha).
- Teste os **caminhos de erro**, não só o caminho feliz.
- Regras de negócio (cálculo do novo estado, geração de evento) têm teste unitário no service.
- ⏳ Testes de integração com PostgreSQL real (avaliar Testcontainers quando o banco entrar).
- Bug corrigido = teste que o reproduz.
- **Não exclua nem enfraqueça testes** para fazer algo passar.

## 12. Git e revisão

- Branch principal: `main`, **protegida**: ninguém faz push direto nela.
- Todo trabalho em branch curta (`feat/…`, `fix/…`, `docs/…`, `chore/…`) → **pull request** → CI
  verde → revisão → *squash merge*.
- PRs pequenos: uma etapa ou uma parte de etapa por PR. O template (`.github/pull_request_template.md`)
  traz o checklist da seção 13.
- *Conventional Commits* em português com escopo do domínio:
  `feat(personagem): cria ficha permanente`, `fix(sessao): impede dano duplicado`.
- Nunca commitar: `.env`, senhas, chaves (`*.jks`, `*.p12`, `*.pem`, `*.key`), dumps de banco.
- Fim de linha LF garantido pelo `.gitattributes` (exceto `*.cmd`). Não remova.
- CI (`.github/workflows/ci.yml`): `./mvnw verify` em todo PR.
- Dependabot (`.github/dependabot.yml`) abre PRs semanais de atualização das dependências e das
  actions; eles passam pela mesma revisão.

## 13. Checklist de entrega

- [ ] Código no domínio e na camada certos; dependência só desce
- [ ] Entidade não exposta; entrada validada; Response certo para cada visão
- [ ] Autorização por papel **e** por posse, verificada no service, com teste de acesso negado
- [ ] Erros usam as exceções próprias; nada de detalhe interno na resposta
- [ ] Nenhum segredo no código ou no `application.yml`; nada sensível em log
- [ ] Consultas parametrizadas; migration incluída; `@Transactional` onde grava mais de uma tabela
- [ ] Toda alteração de estado da sessão gera `EventoSessao`
- [ ] `./mvnw test` passando
