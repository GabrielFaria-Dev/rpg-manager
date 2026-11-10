# RPG Manager

Backend Spring Boot que administra e registra mesas de RPG: usuários, campanhas, sessões,
personagens, estado da sessão e histórico. No futuro terá um desktop em JavaFX que consome esta API.

**Regra principal:** o RPG Manager administra e registra o jogo. Ele **não** rola dados nem calcula
acerto, dano, dificuldade ou bônus: só registra o resultado decidido pelo jogador ou pelo Mestre.

## Antes de qualquer mudança

Leia `docs/arquitetura/arquitetura.md`. É o guia oficial de arquitetura do projeto. Não altere a
arquitetura sem registrar a decisão em `docs/decisoes/` (ADR).

Siga também `docs/arquitetura/boas-praticas.md` (erros, validação, segredos, segurança, banco,
logs e testes). Itens marcados ⏳ só valem quando o recurso existir; itens 🔶 exigem ADR antes.

Não invente regras do RPG. As regras oficiais ficarão em `docs/regras/`. Se uma regra não estiver
documentada, pergunte ao usuário.

## Stack e comandos

- Java 21 (`java.version=21`). A máquina tem só o JDK 26, que compila normalmente.
- Spring Boot 4.1.1, Maven. Use sempre o wrapper: o Maven não está instalado globalmente.
- O usuário roda a aplicação pelo IntelliJ (`RpgManagerApplication` → ▶). Ela sobe na porta 8080.

```bash
./mvnw test              # roda os testes
./mvnw spring-boot:run   # sobe a aplicação (porta 8080)
```

Health check: `GET http://localhost:8080/api/health` → `{"status":"ok"}`

## Regras de código

- **Pacotes por domínio**, nunca por camada técnica no topo:
  `com.rpgmanager.<dominio>/{controller,dto,entity,repository,service}`.
  Domínios previstos: `usuario`, `campanha`, `personagem`, `sessao`, `criatura`, `marca`,
  `habilidade`, `item`, `transcendente`.
- `config/` guarda a configuração técnica do Spring. `shared/` guarda só o que é usado por vários
  domínios; não pode virar gaveta de bagunça.
- Crie pacotes e domínios **de forma incremental**, só quando forem usados.
- **Dependência só entra com motivo concreto.** Justifique antes de adicionar ao `pom.xml`.
- `Personagem` (ficha permanente) fica separado de `EstadoPersonagemSessao` (estado temporário).
  Toda alteração na sessão gera um `EventoSessao`.
- O cliente envia **ações** ("recebi 25 de dano"), não valores finais. O servidor calcula o novo
  estado. Só o Mestre pode corrigir um estado diretamente.
- O jogador altera apenas o próprio personagem. HP, barras e quedas dos outros jogadores são privados.
- **Segurança:** a autorização (papel **e** posse do recurso) é checada no service; a identidade vem
  do token, nunca do corpo/URL. Entidade nunca sai da API (use DTOs). Segredos só em variáveis de
  ambiente. Toda rota protegida tem teste de acesso negado.
- A configuração fica em `application.yml` (não `.properties`). As migrations ficam em
  `src/main/resources/db/migration/`.
- Os testes seguem a mesma estrutura de pacotes do código (`src/test/java/com/rpgmanager/<dominio>/...`).
  No Spring Boot 4, `@WebMvcTest` fica em `org.springframework.boot.webmvc.test.autoconfigure`.
- Siga o estilo do código existente (indentação com tab, nomes de domínio em português).

## Como trabalhar com o usuário

- Responda em **português**.
- O usuário está aprendendo Java e Spring e quer entender **por que** cada pasta e cada classe
  existe, não só copiar. Explique os passos e guie no IntelliJ quando fizer sentido.
- Não exclua nem enfraqueça testes para fazer algo passar.
- Commits só quando o usuário pedir. Nunca commitar direto na `main`: crie uma branch
  (`feat/…`, `fix/…`, `docs/…`, `chore/…`) e abra PR. Commits em *Conventional Commits* em português.
- Ao terminar uma tarefa: rode `./mvnw test` e informe os arquivos modificados e qualquer decisão tomada.

## Ordem de construção

Banco (PostgreSQL) → `usuario` → autenticação → `campanha` → `personagem` → `sessao` →
estado da sessão → eventos → combate → marcas → habilidades → itens → criaturas → transcendentes.
