# ADR-006 – Guia de boas práticas e segurança do backend

- **Status:** aceito
- **Data:** 2026-11-10

> Numeração: os ADRs 001 a 005 já estão reservados no `arquitetura.md` (seção 10) para decisões
> previstas. Este recebe o próximo número livre.

## Contexto

O `arquitetura.md` define a **estrutura** do backend (domínios, camadas, entidades, autorização
jogador × Mestre), mas não dizia **como** tratar erros, validar entrada, guardar segredos, proteger
senhas, evitar acesso a recursos de outros usuários, lidar com concorrência e testar autorização.

Um projeto anterior tem um guia de boas práticas maduro para o backend
(Node.js + Fastify) e para o app (React Native). Muitos princípios valem para qualquer backend.

## Decisão

Criar `docs/arquitetura/boas-praticas.md`, adaptando para Java/Spring os princípios daquele guia que se
aplicam ao RPG Manager, com destaque para segurança.

**Adotado (traduzido para Spring):**
- camadas com dependência só para baixo e objetos HTTP restritos ao controller;
- domínio acessa outro domínio pelo service, não pelo repository;
- entidade nunca exposta: DTOs (`record`) de entrada e saída, um Response por visão;
- validação de entrada na borda (Bean Validation), rejeitando campos desconhecidos;
- exceções próprias + um handler central + `ProblemDetail`; nunca detalhes internos ao cliente;
- segredos só em variáveis de ambiente; configuração validada na inicialização;
- autorização por papel **e** por posse do recurso, no servidor, com identidade vinda do token;
- 404 para recurso de outra campanha; senha só com `PasswordEncoder`; login com erro genérico;
- banco: migrations, `ddl-auto: validate`, `open-in-view: false`, restrições no banco, consultas
  parametrizadas, `@Transactional`, lock otimista (`@Version`) nos estados da sessão;
- idempotência das ações da sessão (mecanismo a decidir);
- logs com SLF4J e sem dados sensíveis;
- teste de acesso negado obrigatório por rota; testes de caminho de erro.

**Não adotado (específico de outra stack ou de outro produto):**
- tudo de Expo/React Native (navegação, TanStack Query, acessibilidade mobile);
- Node.js: event loop, prototype pollution, `child_process`, ESM, TypeScript;
- push, mapas e GPS;
- versionamento `/v1`: o único cliente é o nosso desktop; reavaliar se surgir cliente externo;
- CORS: o cliente é JavaFX, não navegador.

**Aplicado já nesta decisão** (sem dependência nova):
- `.gitignore` passa a ignorar `.env`, chaves e dumps de banco;
- `application.yml` fixa explicitamente que erros não expõem stack trace, mensagem nem exceção.

O restante é regra marcada como ⏳ (vale quando o recurso existir) ou 🔶 (exige ADR próprio, como o
mecanismo de autenticação).

## Consequências

- Cada etapa da ordem de construção passa a ter regras claras de segurança desde o primeiro código.
- Algumas dependências já ficam previstas e justificadas para quando forem necessárias:
  `spring-boot-starter-validation` (primeiro DTO), Spring Security (autenticação),
  Data JPA + driver PostgreSQL + Flyway (banco).
- Ficam abertas e precisarão de ADR: mecanismo de autenticação (JWT × sessão), CSRF, limite de
  tentativas de login e mecanismo de idempotência das ações.
