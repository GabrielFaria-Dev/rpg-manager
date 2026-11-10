## O que muda

<!-- Resumo em 1–3 linhas. -->

## Por quê

<!-- Motivo / etapa da ordem de construção / ADR relacionado. -->

## Como testar

<!-- Testes adicionados e, se fizer sentido, a requisição para testar à mão. -->

## Checklist (docs/arquitetura/boas-praticas.md, seção 13)

- [ ] Código no domínio e na camada certos; dependência só desce
- [ ] Entidade não exposta; entrada validada; Response certo para cada visão
- [ ] Autorização por papel **e** por posse, verificada no service, com teste de acesso negado
- [ ] Erros usam as exceções próprias; nada de detalhe interno na resposta
- [ ] Nenhum segredo no código ou no `application.yml`; nada sensível em log
- [ ] Consultas parametrizadas; migration incluída; `@Transactional` onde grava mais de uma tabela
- [ ] Toda alteração de estado da sessão gera `EventoSessao`
- [ ] Dependência nova justificada (ou nenhuma)
- [ ] Mudança de arquitetura registrada em ADR (ou nenhuma)
- [ ] `./mvnw test` passando
