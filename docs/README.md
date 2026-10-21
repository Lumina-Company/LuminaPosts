# Documentação do Luminaria

Arquitetura e especificação técnica do **Luminaria**, o painel web multi-imobiliária do **LuminaImob**.

## Por onde começar

| Perfil | Leia primeiro |
|---|---|
| Quem chega ao projeto | [00 Visão geral](00-visao-geral.md) → [01 Arquitetura](01-arquitetura.md) → [decisões](decisoes.md) |
| Backend Java | [03](03-ciclo-de-vida.md) → [04](04-banco-de-dados.md) → [05](05-seguranca.md) → [06](06-backend-spring.md) |
| Fábrica Python | [02](02-integracao-java-python.md) → [09](09-fabrica-python.md) → [11](11-fluxos-de-automacao.md) |
| Frontend | [07](07-frontend-nextjs.md) → [08](08-especificacao-de-telas.md) |
| Operação | [10](10-deploy-e-repositorio.md) |
| Todos | [12 Padrões de projeto](12-padroes-de-projeto.md) |

## Índice

| # | Documento | Conteúdo |
|---|---|---|
| 00 | [Visão geral](00-visao-geral.md) | Produto, papéis, planos, escopo, mapa de telas, catálogo de requisitos, glossário |
| 01 | [Arquitetura](01-arquitetura.md) | Contexto, containers, responsabilidades, comunicação front ↔ API |
| 02 | [Integração painel ↔ fábrica](02-integracao-java-python.md) | Banco como contrato, filas, sinais, concorrência, sequências |
| 03 | [Ciclo de vida](03-ciclo-de-vida.md) | Máquinas de estado do imóvel e do post, transições por ator, regras de rejeição |
| 04 | [Banco de dados](04-banco-de-dados.md) | Modelo ER, restrições, busca, franquia, convenção Flyway |
| 05 | [Segurança e multi-tenant](05-seguranca.md) | Isolamento em três camadas, autenticação, autorização, mídia, segredos |
| 06 | [Backend Spring](06-backend-spring.md) | Módulos, diagramas de classes, casos de uso, API REST |
| 07 | [Frontend Next.js](07-frontend-nextjs.md) | Rotas, estrutura, camada de dados, componentes, identidade visual |
| 08 | [Especificação de telas](08-especificacao-de-telas.md) | Comportamento de cada tela do cliente e do Admin |
| 09 | [Fábrica Python](09-fabrica-python.md) | Ingestor, gerador, publicador, jobs, erros, operação, testes |
| 10 | [Repositório, containers e deploy](10-deploy-e-repositorio.md) | Monorepo, imagens, serviços, ordem de deploy, CI, ambientes, entrada de clientes |
| 11 | [Fluxos de automação](11-fluxos-de-automacao.md) | Fluxos versionados por imobiliária e catálogo de etapas |
| 12 | [Padrões de projeto](12-padroes-de-projeto.md) | Padrões adotados, convenções por camada, testes, Git, anti-padrões |
| — | [Decisões (ADR)](decisoes.md) | Registro das decisões de arquitetura |

## Stack

| Camada | Tecnologia |
|---|---|
| Painel (backend) | Java 21 · Spring Boot 3 · Spring Security · Spring Data JPA (Hibernate 6) · Spring Session JDBC |
| Banco | PostgreSQL 16 · Flyway |
| Frontend | Next.js (App Router) · TypeScript · TanStack Query · Tailwind · shadcn/ui |
| Fábrica | Python 3.12 · psycopg 3 · Pydantic · FFmpeg |
| Mídia | MinIO (S3) |
| Infraestrutura | Docker · EasyPanel · Traefik/Caddy · GitHub Actions |

## Diagramas

Todos os diagramas estão em [Mermaid](https://mermaid.js.org/) dentro dos próprios arquivos e são renderizados pelo GitHub. Para editar, altere o bloco ` ```mermaid ` correspondente.

## Manutenção desta documentação

- A documentação acompanha o código: um PR que muda arquitetura, contrato ou comportamento atualiza o documento correspondente.
- Decisões estruturais novas entram em [decisoes.md](decisoes.md) como nova ADR.

## Plano de desenvolvimento

Consulte [13 — Plano de desenvolvimento](13-plano-de-desenvolvimento.md) para as fases, blocos, dependências e critérios de conclusão.
