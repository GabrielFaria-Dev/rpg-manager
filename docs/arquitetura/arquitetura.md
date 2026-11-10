# Arquitetura do RPG Manager

> Documento-guia extraído da conversa de planejamento do projeto.
> Contém **somente** as decisões de arquitetura. Regras do RPG, metodologia de uso das IAs e
> manual do jogador ficam fora deste arquivo.

---

## 1. Princípio fundamental

> **O RPG Manager administra e registra o jogo. Ele não é o sistema de resolução de dados.**

O programa **não calcula** acerto, dano rolado, dificuldade, bônus ou D20. Ele apenas **registra o
resultado** decidido pelo jogador/Mestre e mantém o estado e o histórico da sessão.

---

## 2. Visão geral

```text
                    ┌─────────────────────┐
                    │     JavaFX / UI     │
                    │      Desktop        │
                    └──────────┬──────────┘
                               │  HTTP/REST + JSON
                               ▼
                    ┌─────────────────────┐
                    │      REST API       │
                    │ Java + Spring Boot  │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
       ┌────────────┐   ┌─────────────┐  ┌─────────────┐
       │  Services  │   │ Repositories│  │    Auth     │
       └─────┬──────┘   └──────┬──────┘  └─────────────┘
             └────────┬────────┘
                      ▼
               ┌──────────────┐
               │ PostgreSQL   │
               └──────────────┘
```

Fluxo em camadas:

```text
JavaFX → REST API → Controllers → Services → Repositories → PostgreSQL
```

**Regra:** o JavaFX **nunca** conversa diretamente com o PostgreSQL. O desktop conversa apenas com a API.

---

## 3. Tecnologias

| Camada      | Tecnologia                                              |
|-------------|---------------------------------------------------------|
| Linguagem   | Java 21 (LTS); **uma única versão de Java para todo o projeto** |
| Backend     | Spring Boot 4.1.1, Spring Web, Spring Security, JPA/Hibernate |
| Banco       | PostgreSQL (migrations em `db/migration`)               |
| Desktop     | JavaFX, FXML, JavaFX CSS, Scene Builder                 |
| Build       | Maven                                                   |
| Qualidade   | JUnit, Mockito, testes de integração, CI/CD             |
| Controle    | Git / GitHub                                            |
| IDE         | IntelliJ IDEA                                           |

**Regra de dependências:** a dependência só entra quando precisamos dela. No início, só **Spring Web**.
Security, Data JPA, driver do PostgreSQL, Validation etc. entram quando houver motivo concreto.

---

## 4. Estrutura do repositório

### 4.1 Agora (fase inicial)

Por enquanto, **o próprio projeto é o backend**. Ainda não usamos Maven multimódulo nem `backend/backend`.

```text
rpg-manager/
├── .mvn/
├── src/
│   ├── main/
│   │   ├── java/com/rpgmanager/
│   │   │   └── RpgManagerApplication.java
│   │   └── resources/
│   └── test/
├── .gitignore
├── mvnw / mvnw.cmd
└── pom.xml
```

Configuração do projeto no Spring Initializr (IntelliJ → New Project → Spring Boot):

```text
Name:          rpg-manager
Language:      Java
Type:          Maven
Group:         com.rpgmanager
Artifact:      rpg-manager
Package:       com.rpgmanager
JDK / Java:    21
Packaging:     Jar
Spring Boot:   4.1.1
Dependências:  Spring Web (apenas)
```

### 4.2 Destino final (Maven multimódulo)

Quando o JavaFX entrar, o projeto vira:

```text
RPG-MANAGER/
├── backend/            ← Spring Boot (API)
│   ├── pom.xml
│   └── src/{main/{java,resources}, test}
├── desktop/            ← JavaFX
│   ├── pom.xml
│   └── src/{main/{java,resources}, test}
├── docs/
│   ├── regras/         ← fonte oficial das regras do RPG
│   ├── arquitetura/    ← decisões técnicas (este arquivo)
│   ├── decisoes/       ← ADRs
│   └── tarefas/
├── infra/
├── README.md
├── CLAUDE.md
├── AGENTS.md
└── pom.xml             ← pom pai (multimódulo)
```

---

## 5. Backend: organização por funcionalidade (feature)

Organizamos **por domínio do RPG**, e não por tipo técnico. Tudo que pertence a um domínio fica junto.

❌ **Não fazer** (pastas por camada técnica no topo):

```text
controller/  service/  repository/  entity/  dto/
```

✅ **Fazer** (cada domínio tem suas próprias camadas):

```text
com.rpgmanager/
├── RpgManagerApplication.java   ← no pacote-raiz (component scan a partir daqui)
├── config/                      ← configurações técnicas do Spring
├── shared/                      ← apenas o que é realmente compartilhado
├── usuario/
├── campanha/
├── personagem/
├── sessao/
├── criatura/
├── marca/
├── habilidade/
├── item/
└── transcendente/
```

### 5.1 Camadas dentro de cada domínio

```text
personagem/
├── controller/   PersonagemController.java        → recebe a requisição
├── dto/          CriarPersonagemRequest.java      → entrada/saída da API
│                 PersonagemResponse.java
├── entity/       Personagem.java                  → representa os dados persistidos
├── repository/   PersonagemRepository.java        → acessa o banco
└── service/      PersonagemService.java           → contém a regra de negócio
```

### 5.2 Responsabilidade de cada domínio

| Pacote          | Conteúdo                                                                 |
|-----------------|--------------------------------------------------------------------------|
| `usuario`       | `Usuario`, login, `LoginRequest`, `UsuarioResponse`; depois autenticação/autorização |
| `campanha`      | Campanhas (Usuário → Campanha → Sessões → Personagens)                   |
| `personagem`    | Ficha **permanente** do personagem                                       |
| `sessao`        | `Sessao`, `EstadoPersonagemSessao`, `EventoSessao`, `SessaoService`      |
| `criatura`      | `Criatura` (cadastro) e `EstadoCriaturaSessao` (criatura dentro de uma sessão) |
| `marca`         | Tudo relativo às Marcas. É uma parte própria do RPG, **não** um atributo |
| `habilidade`    | Habilidades, relacionadas a Personagem e a Marca                         |
| `item`          | Armas, utensílios, armaduras, escudos; tipos: comum, material especial, tecnológico, misto, lendário |
| `transcendente` | Camada própria do sistema, separada de atributos, personagem e habilidades |
| `config`        | Ex.: `SecurityConfig`, `CorsConfig`, `JacksonConfig` (criar só quando necessário) |
| `shared`        | Ex.: `exception/`, `validation/`, `response/`, `util/`                   |

> ⚠️ **Regra do `shared`:** não pode virar uma "gaveta de bagunça". Só entra ali o que é usado por
> várias funcionalidades.

### 5.3 Resources

```text
src/main/resources/
├── application.yml      ← server, database, JPA, logging, security
└── db/
    └── migration/       ← migrations do banco
```

### 5.4 Testes

Os testes espelham a mesma organização:

```text
src/test/java/com/rpgmanager/
├── usuario/
├── campanha/
├── personagem/
│   └── service/PersonagemServiceTest.java
├── sessao/
└── ...
```

### 5.5 Regras práticas no IntelliJ

- Criar pacotes com **New → Package** (não com pastas do Windows), para manter o `package` coerente.
- **Não criar todos os pacotes de uma vez.** Primeiro `config`, `shared` e o domínio `usuario`;
  depois `campanha`, depois `personagem`, e assim por diante. Cada pasta só aparece quando tem um motivo.

---

## 6. Modelo de domínio

### 6.1 Decisão central: ficha permanente × estado da sessão

```text
PERSONAGEM              = estado permanente (ficha)
EstadoPersonagemSessao  = estado temporário (durante uma sessão)
```

A ficha permanente **não é alterada** toda vez que alguém toma dano. Essa separação deve existir
desde o primeiro código. (O mesmo vale para `Criatura` × `EstadoCriaturaSessao`.)

### 6.2 Entidades

```text
Usuario
 ├── id, nome, email, senha
 └── tipo: JOGADOR | MESTRE

Campanha
 ├── id, nome, descrição
 └── mestre

Sessao
 ├── id, campanha, nome, data, status
 └── participantes

Personagem
 ├── nome, raça, classe, nível, XP, HP máximo
 ├── atributos
 ├── Marca
 ├── habilidades
 └── equipamentos

EstadoPersonagemSessao
 ├── HP atual
 ├── barras
 ├── quedas
 ├── estados temporários
 └── outros efeitos da sessão

EventoSessao
 ├── personagem, jogador
 ├── tipo  (DANO, CURA, BARRA_ALTERADA, QUEDA, ...)
 ├── valor, descrição
 └── horário
```

Relações principais:

```text
USUARIO
   └── PERSONAGEM
          ├── ATRIBUTOS
          ├── RAÇA
          ├── CLASSE
          ├── HABILIDADES
          ├── EXECUÇÕES
          ├── CONDIÇÕES
          └── MARCA
                ├── Origem
                ├── Tipo
                ├── Manifestação
                ├── Condições de despertar
                └── Habilidades
```

> Não existe uma tabela `magias` tradicional: no universo do jogo não há Mana, e sim **Marcas**.

### 6.3 Eventos da sessão (histórico)

Toda alteração importante gera um `EventoSessao`, formando o histórico completo:

```text
DANO            Arkan  -20 HP        "Golpe do Guardião"  19:42
BARRA_ALTERADA  Arkan  Barra 1 +10%                       19:43
QUEDA           Arkan  Queda nº 2                         19:51
```

### 6.4 O cliente envia **ações**, não valores finais

```text
❌  "HP = 57"
✅  "Recebi 25 de dano"   → servidor: HP anterior 82 − 25 = 57
✅  "Recuperei 10 HP"
✅  "Carreguei a Barra 1 em 10%"
```

O servidor calcula o novo estado e registra o evento, o que mantém o histórico confiável.
**Exceção:** o Mestre tem ferramentas administrativas para corrigir um estado diretamente.

---

## 7. Autorização e privacidade

- Cada jogador **controla o estado do próprio personagem** (dano, cura, barras, quedas).
- HP, barras e número de quedas dos **outros** jogadores são **privados**; os outros veem apenas uma
  indicação visual/estado geral.
- O **Mestre** tem acesso completo e pode alterar qualquer personagem ou criatura.

Exemplo de verificação no backend:

```text
POST /sessoes/15/personagens/8/dano

Quem está fazendo a requisição?
        ↓
É jogador? → É o dono do personagem?
        ├─ SIM → permite
        └─ NÃO → bloqueia
```

| Perfil  | Próprio personagem | Outros jogadores        | Criaturas                    |
|---------|--------------------|-------------------------|------------------------------|
| JOGADOR | completo           | informações limitadas   | conforme configuração do Mestre |
| MESTRE  | completo           | completo / pode alterar | completo / pode alterar      |

---

## 8. Desktop (JavaFX)

```text
desktop/
└── src/main/
    ├── java/com/rpgmanager/      (*)
    │   ├── controller/
    │   ├── service/              ← chamadas à API REST
    │   ├── model/
    │   └── navigation/
    └── resources/
        ├── fxml/
        ├── css/
        ├── images/
        └── themes/
```

(*) Na conversa este trecho aparece como `br/com/rpgmanager`, mas o backend usa `com.rpgmanager`.
Recomenda-se padronizar em **`com.rpgmanager`** nos dois módulos.

Ordem das telas: **Login → Dashboard → Personagem → Sessão → Painel do Mestre.**

---

## 9. Ordem de construção do backend

```text
 1. Projeto Maven
 2. Spring Boot
 3. Estrutura de pacotes
 4. Health check  (GET /api/health → {"status": "ok"})
 5. Banco
 6. Usuario
 7. Autenticação
 8. Campanha
 9. Personagem
10. Sessão
11. Estado da sessão
12. Eventos
13. Combate
14. Marcas
15. Habilidades
16. Itens
17. Criaturas
18. Transcendentes
```

Primeiro marco: **IntelliJ → Java 21 → Maven → Spring Boot 4.1.1 → Spring Web →
`RpgManagerApplication` inicia (Tomcat na porta 8080).** Nada de banco, login, personagem ou combate ainda.

---

## 10. Documentação e decisões

```text
docs/regras/        ← fonte oficial das regras do RPG (nenhuma regra vive só na conversa)
docs/arquitetura/   ← arquitetura.md, boas-praticas.md, modelo-dominio.md, banco-de-dados.md,
                      api.md, autenticacao.md, autorizacao.md, javafx.md, testes.md
docs/decisoes/      ← ADRs
```

ADRs previstos:

```text
ADR-001-personagem-estado-sessao.md
ADR-002-api-rest.md
ADR-003-maven-multimodule.md
ADR-004-autorizacao-jogador-mestre.md
ADR-005-javafx-como-desktop.md
```

ADRs já registrados: `ADR-006-boas-praticas-e-seguranca.md`.

**Como escrever o código** (erros, validação, segredos, segurança, banco, logs, testes):
[`boas-praticas.md`](boas-praticas.md).

Formato:

```text
# ADR-001
## Decisão     Separar Personagem de EstadoPersonagemSessao.
## Contexto    HP atual e barras são temporários.
## Consequência O personagem permanente pode existir independentemente de uma sessão.
```
