# Registro de decisões de arquitetura (ADR)

Cada decisão registra o contexto, o que foi decidido e as consequências. Uma decisão só muda com uma nova ADR que a substitua.

| ADR | Decisão | Estado |
|---|---|---|
| [ADR-01](#adr-01--monólito-modular) | Monólito modular, sem microsserviços | Aceita |
| [ADR-02](#adr-02--integração-pelo-banco) | Painel e fábrica integram pelo PostgreSQL, sem HTTP entre eles | Aceita |
| [ADR-03](#adr-03--filas-no-postgresql) | Filas no próprio PostgreSQL com `SKIP LOCKED` + `LISTEN/NOTIFY` | Aceita |
| [ADR-04](#adr-04--flyway-é-o-dono-do-schema) | Flyway é o único dono do schema | Aceita |
| [ADR-05](#adr-05--post-independente-de-imóvel) | O que é aprovado e publicado é um *post*, com `imovel_id` opcional | Aceita |
| [ADR-06](#adr-06--isolamento-de-tenant-em-três-camadas) | Isolamento de tenant em três camadas | Aceita |
| [ADR-07](#adr-07--sessão-em-cookie-sem-jwt) | Sessão em cookie HttpOnly, sem JWT no navegador | Aceita |
| [ADR-08](#adr-08--transições-validadas-no-banco) | Transições de estado validadas por trigger | Aceita |
| [ADR-09](#adr-09--franquia-numa-função-sql-única) | Franquia calculada por uma função SQL única | Aceita |
| [ADR-10](#adr-10--monorepo-e-um-container-por-processo) | Monorepo; uma imagem por aplicação; um container por processo | Aceita |
| [ADR-11](#adr-11--fluxos-de-automação-versionados) | Automação variável por imobiliária via fluxos versionados | Aceita |
| [ADR-12](#adr-12--um-deploy-multi-tenant) | Um deploy multi-tenant para todos os clientes | Aceita |
| [ADR-13](#adr-13--stack-java--nextjs--python) | Java/Spring no painel, Next.js no front, Python na fábrica | Aceita |
| [ADR-14](#adr-14--schema-próprio-e-migração-dos-dados-existentes) | Schema próprio `luminaria`, com migração dos dados existentes | Aceita |

---

## ADR-01 — Monólito modular

**Contexto.** Escala modesta (~10 imobiliárias, ~2.000 imóveis cada), time pequeno, VPS compartilhada. As regras centrais (um vídeo por horário, franquia, segunda rejeição) dependem de consistência transacional.

**Decisão.** Um backend Spring Boot organizado em módulos com fronteiras verificadas (Spring Modulith). A fábrica Python roda como processos de fundo da mesma aplicação, compartilhando o banco. O estilo é uma *arquitetura baseada em serviços*, não microsserviços: não há banco por serviço nem comunicação síncrona entre serviços.

**Consequências.** Regras garantidas por transação e índice único, sem sagas. Um módulo pode virar serviço próprio no futuro sem reescrever a regra, se surgir motivo concreto (vários times, carga muito diferente, consumo externo).

## ADR-02 — Integração pelo banco

**Contexto.** O painel precisa continuar no ar com a fábrica parada, e vice-versa (RNF-LUM-04). Geração leva minutos.

**Decisão.** Java e Python não se chamam. O painel grava intenções (posts em `na_fila`, agendamentos, comandos); a fábrica consome e grava resultados. Detalhes no [02](02-integracao-java-python.md).

**Consequências.** Estado e "mensagem" são a mesma linha, na mesma transação. Exige matriz de escrita clara, updates condicionais e `GRANT`s por coluna.

## ADR-03 — Filas no PostgreSQL

**Contexto.** Volume baixo; operar Redis ou RabbitMQ adiciona serviço e risco de divergência entre fila e estado.

**Decisão.** Itens de fila são as próprias linhas de estado, consumidas com `FOR UPDATE SKIP LOCKED`. `NOTIFY` disparado por trigger acorda os consumidores; polling curto garante a entrega.

**Consequências.** Nenhum serviço extra. Escala horizontal da fábrica por réplicas.

## ADR-04 — Flyway é o dono do schema

**Decisão.** Toda mudança de schema é uma migration Flyway no módulo `api`. A fábrica nunca executa DDL e confere a versão do schema ao iniciar. Migrations seguem *expand → contract*.

**Consequências.** Um único lugar muda o contrato. Ordem de deploy: `api` → `fabrica` → `web`.

## ADR-05 — Post independente de imóvel

**Decisão.** Aprovação, grade e publicação operam sobre `post` (vídeo + legenda + hashtags). `post.imovel_id` é opcional.

**Consequências.** A futura publicação de vídeos próprios (sem imóvel) entra sem reescrever aprovação, grade nem publicadores.

## ADR-06 — Isolamento de tenant em três camadas

**Decisão.** O tenant vem só da sessão; o Hibernate filtra por `@TenantId`; o PostgreSQL aplica Row Level Security com a variável de transação `luminaria.imobiliaria_id`. Detalhes no [05](05-seguranca.md).

**Consequências.** Esquecer o filtro resulta em "nenhuma linha", nunca em "todas as linhas". Testes de isolamento obrigatórios no CI.

## ADR-07 — Sessão em cookie, sem JWT

**Decisão.** Spring Session JDBC com cookie `HttpOnly; Secure; SameSite=Lax`, 30 dias, CSRF ligado.

**Consequências.** Revogação imediata (desativar usuário apaga as sessões). Nenhum token acessível ao JavaScript.

## ADR-08 — Transições validadas no banco

**Decisão.** A tabela `transicao_estado` e um trigger recusam qualquer mudança de estado não prevista para o ator (`cliente`, `admin`, `sistema`). A decisão de *qual* transição fazer fica no código; a *validade*, no banco.

**Consequências.** Duas linguagens escrevem estado sem poder pular etapas. Violação vira erro `P0001`, mapeado para HTTP 409.

## ADR-09 — Franquia numa função SQL única

**Decisão.** `fn_saldo_franquia` é a única conta, usada pelo painel (contador, bloqueio) e pela fábrica (antes de gerar), a partir dos posts que consumiram franquia e do fechamento mensal.

**Consequências.** Sem contador solto que possa divergir. A conta é auditável pelo registro de posts.

## ADR-10 — Monorepo e um container por processo

**Decisão.** Um repositório com `api/`, `web/`, `fabrica/`, `contrato/`, `infra/` e `docs/`. Uma imagem por aplicação; a imagem da fábrica roda em três containers (ingestor, gerador, publicador). Detalhes no [10](10-deploy-e-repositorio.md).

**Consequências.** Mudança de contrato em um único PR. Falha, deploy, escala e limites de memória isolados por processo.

## ADR-11 — Fluxos de automação versionados

**Decisão.** A automação da fábrica é montada a partir de um fluxo JSON versionado, atribuído por imobiliária, executado por um registro de etapas no Python. Estados e painel são fixos. Detalhes no [11](11-fluxos-de-automacao.md).

**Consequências.** Variações por cliente sem condicionais por cliente e sem deploy do painel. Código específico, quando necessário, fica isolado em etapas próprias.

## ADR-12 — Um deploy multi-tenant

**Contexto.** Todas as imobiliárias entram pelo mesmo domínio (RNF-LUM-09).

**Decisão.** Uma única instalação atende todas as imobiliárias. Cliente novo é cadastro no Admin (imobiliária, plano, XML, fluxo, usuários), não deploy.

**Consequências.** Correções e migrations aplicadas uma vez para todos. Instalação dedicada só seria considerada por exigência contratual de isolamento físico, e reutilizaria as mesmas imagens.

## ADR-13 — Stack Java + Next.js + Python

**Decisão.** Painel em Java 21 + Spring Boot 3; frontend em Next.js (App Router) + TypeScript; fábrica em Python 3.12; PostgreSQL 16 com Flyway; MinIO para mídia.

**Consequências.** Domínio maduro do time em Java/Spring. Tipos compartilhados entre linguagens são gerados a partir de `contrato/` (estados, transições, schema de fluxo) e do OpenAPI. Orçamento de memória da JVM controlado no container.

## ADR-14 — Schema próprio e migração dos dados existentes

**Decisão.** O Luminaria usa o schema `luminaria`, criado do zero pelo Flyway. Os dados do pipeline anterior são migrados por um script único, versionado, executado antes da entrada em produção.

**Consequências.** Modelo limpo, sem herdar estruturas do pipeline anterior. A migração é testada em homologação com cópia dos dados reais.
