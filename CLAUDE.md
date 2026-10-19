# CLAUDE.md

Instruções para o Claude trabalhar neste repositório. Leia este arquivo inteiro antes de qualquer tarefa.

## Projeto

**Luminaria** é o painel web SaaS multi-imobiliária do **LuminaImob**: a fábrica (Python) transforma o XML de imóveis de cada imobiliária em vídeos com legenda por IA; o painel (Spring Boot + Next.js) permite aprovar, agendar e acompanhar a publicação no Instagram, TikTok e YouTube Shorts. Um único deploy atende todas as imobiliárias.

## Documentação é a fonte da verdade

A arquitetura está em `docs/`. **Antes de alterar uma área, leia o documento dela.** Se o código e a documentação divergirem, pare e aponte a divergência em vez de escolher um lado em silêncio.

| Vai mexer em | Leia |
|---|---|
| Qualquer coisa (primeira vez) | `docs/00-visao-geral.md`, `docs/01-arquitetura.md`, `docs/decisoes.md` |
| Estados, aprovação, rejeição, chamados | `docs/03-ciclo-de-vida.md` |
| Schema, migrations, consultas, franquia | `docs/04-banco-de-dados.md` |
| Login, permissões, tenant, segredos | `docs/05-seguranca.md` |
| Backend Java | `docs/06-backend-spring.md` |
| Frontend | `docs/07-frontend-nextjs.md`, `docs/08-especificacao-de-telas.md` |
| Qualquer coisa que o painel e a fábrica compartilham | `docs/02-integracao-java-python.md` |
| Fábrica Python | `docs/09-fabrica-python.md`, `docs/11-fluxos-de-automacao.md` |
| Docker, CI, deploy | `docs/10-deploy-e-repositorio.md` |
| Estilo de código e padrões | `docs/12-padroes-de-projeto.md` |

## Estrutura do repositório

```
api/        Spring Boot (Java 21). Contém as migrations Flyway em src/main/resources/db/migration
web/        Next.js (App Router, TypeScript)
fabrica/    Python 3.12: ingestor, gerador, publicador (uma imagem, três papéis)
contrato/   Fonte única de enums, transições e schema de fluxo; gera código para as três linguagens
infra/      Docker Compose (dev e referência de produção), proxy
docs/       Arquitetura, especificação e decisões
```

## Regras invariantes

Estas regras não têm exceção. Se uma tarefa parecer exigir quebrar uma delas, pare e pergunte.

1. **Isolamento por imobiliária.** Toda tabela de dado de cliente tem `imobiliaria_id` e RLS. O tenant vem **só da sessão** (`TenantContext`); endpoints de cliente nunca recebem `imobiliariaId` por parâmetro, header ou corpo. Recurso de outra imobiliária responde **404**. Todo endpoint novo entra no teste de isolamento.
2. **Flyway é o único dono do schema.** Toda mudança de banco é uma migration nova em `api/src/main/resources/db/migration`, compatível com a versão anterior da aplicação (*expand → contract*). Nunca editar migration aplicada. Nunca DDL no Python. `ddl-auto=validate`.
3. **Painel e fábrica não se chamam.** Integração só pelo PostgreSQL (linhas de estado + `NOTIFY` por trigger). Nada de HTTP entre `api` e `fabrica`.
4. **Mudança de estado é condicional.** Todo `UPDATE` de `estado` tem `WHERE estado = <esperado>` e confere as linhas afetadas. A transição precisa existir em `transicao_estado` para o ator (`cliente`, `admin`, `sistema`).
5. **Toda ação grava `evento` na mesma transação** da mudança que ela registra.
6. **Fila no PostgreSQL** com `FOR UPDATE SKIP LOCKED`. Nenhuma transação aberta durante chamada externa (IA, redes, MinIO).
7. **Franquia só por `fn_saldo_franquia`**, verificada antes de gerar, com `pg_advisory_xact_lock` por imobiliária.
8. **A fábrica nunca escreve** `legenda_editada`, `hashtags_editadas`, `decisao` nem `agendamento`. Legenda editada pelo cliente sempre vence a IA.
9. **Sem condicional por cliente.** Nada de `if imobiliaria_id == ...`. Variação vai para configuração da imobiliária, plano ou etapa de fluxo (`fabrica/etapas/clientes/<cliente>/`). Versão de fluxo é imutável: muda-se criando outra.
10. **Contrato gerado.** Enums de estado, transições e schema de fluxo vêm de `contrato/`. Ao mudar, rode o gerador e commite o código gerado nas três linguagens.
11. **Segredos** só em variável de ambiente ou campo cifrado (AES-256-GCM). Nunca em código, log, URL ou mensagem de erro.
12. **Datas** em `timestamptz` (UTC) no banco; exibição e cálculo de grade no fuso `America/Sao_Paulo`.

## Convenções

- **Idioma:** domínio, tabelas, classes de domínio, endpoints e textos de interface em português (`AprovarPost`, `/imoveis/{id}/linha-do-tempo`, `post.legenda_editada`). Termos de framework permanecem em inglês (`Controller`, `Repository`, `hook`).
- **Java:** módulos `api/ aplicacao/ dominio/ infra/`; um caso de uso por classe; `@Transactional` no caso de uso; DTOs como `record`; erros em `ProblemDetail`; regras de estado em métodos da entidade.
- **Python:** tipagem completa (`mypy --strict`, `ruff`); SQL explícito com psycopg 3 em `fabrica/db/`; etapas registradas com `@etapa("tipo")` e parâmetros validados por Pydantic; erros classificados em transitório, permanente ou de configuração.
- **Frontend:** `features/<area>/{api.ts, hooks.ts, components/}`; dados só via TanStack Query; filtros e abas na URL; tipos da API gerados do OpenAPI; cores só por token do Tailwind.
- **Banco:** `snake_case`, tabelas no singular, estados como `text` + `CHECK`, regra crítica como restrição (índice único parcial, `EXCLUDE`, `CHECK`).
- **Commits:** Conventional Commits em português (`feat(aprovacao): ...`, `fix(fabrica): ...`, `docs: ...`).

Detalhes e exemplos em `docs/12-padroes-de-projeto.md`.

## Como trabalhar

1. Leia o documento da área antes de propor ou escrever código.
2. Prefira a solução mais simples que respeite as regras invariantes. Nada de abstração "para o futuro", Redis, broker ou microsserviço sem ADR.
3. Escreva os testes junto com o código:
   - regra de domínio → teste de tabela de casos;
   - endpoint → teste de integração (Testcontainers) **e** caso no teste de isolamento;
   - etapa da fábrica → teste unitário + integração com as migrations reais.
4. Mudou arquitetura, contrato, estado ou comportamento de tela? Atualize o documento correspondente em `docs/` no mesmo PR.
5. Decisão estrutural nova (dependência de infraestrutura, estilo de integração, nova máquina de estados) → nova ADR em `docs/decisoes.md`.
6. Diagramas sempre em Mermaid, dentro do Markdown.

## Comandos

| Ação | Comando |
|---|---|
| Ambiente local (Postgres, MinIO, compositor, fábrica com provedores falsos) | `docker compose -f infra/compose.dev.yml up -d` |
| Backend: testes completos | `cd api && ./mvnw verify` |
| Backend: rodar | `cd api && ./mvnw spring-boot:run -Dspring-boot.run.profiles=dev` |
| Frontend: dev / testes / build | `cd web && npm run dev` · `npm test` · `npm run build` |
| Fábrica: testes / lint / tipos | `cd fabrica && uv run pytest` · `uv run ruff check .` · `uv run mypy .` |
| Fábrica: rodar um papel | `cd fabrica && uv run python -m fabrica gerador` |
| Regenerar contrato | `python contrato/gerar.py` |

Provedores de IA e redes sociais ficam desligados em desenvolvimento (`LUM_GERADOR=falso`, `LUM_PUBLICADOR=falso`). Nunca use credenciais de produção localmente.

## Não faça

- Não exponha entidade JPA na API.
- Não leia nem grave dados sem transação (o RLS depende dela).
- Não mude estado com `setEstado` direto nem sem update condicional.
- Não adicione dependência de infraestrutura nova sem ADR.
- Não coloque nome de cliente, dado pessoal real ou credencial em código, teste, fixture ou documentação.
