# 12 — Padrões de projeto e convenções

Padrões adotados no Luminaria, onde cada um se aplica e o motivo. Código novo segue este documento; desvio exige justificativa no PR (e ADR, se for estrutural).

## 1. Princípios

1. **Simples antes de genérico.** Abstração só quando há duas implementações reais ou um ponto de extensão previsto ([11](11-fluxos-de-automacao.md)). Nada de interface com uma implementação "para o futuro".
2. **A regra mora em um lugar.** Regra de negócio no domínio (Java) ou na etapa (Python); validade de estado e invariantes no banco; o front só exibe.
3. **O banco garante o que não pode falhar.** Unicidade, transições, isolamento e franquia têm proteção no PostgreSQL, além do código.
4. **Falhar fechado.** Na dúvida, não mostra dado, não publica, não marca imóvel como removido.
5. **Tudo deixa rastro.** Toda ação grava `evento` na mesma transação da mudança (RN-LUM-11).

## 2. Mapa de padrões

| Padrão | Onde | Por quê |
|---|---|---|
| Monólito modular | `api` (módulos com Spring Modulith) | Consistência transacional com fronteiras claras ([ADR-01](decisoes.md#adr-01--monólito-modular)) |
| Banco como ponto de integração | `api` ↔ `fabrica` | Desacoplamento sem broker ([ADR-02](decisoes.md#adr-02--integração-pelo-banco)) |
| Competing Consumers + Lease | Filas de geração e publicação | Várias réplicas, nenhum item duplicado, retomada após queda |
| Máquina de estados com tabela de transições | `imovel`, `post`, `publicacao` | Uma fonte de verdade para transições válidas ([03](03-ciclo-de-vida.md)) |
| Caso de uso (Application Service) | `api/*/aplicacao` | Uma classe por ação do usuário, fácil de rastrear até o requisito |
| Modelo de domínio rico | `api/*/dominio` | Entidades com comportamento (`post.aprovar()`), não só getters/setters |
| Serviço de domínio | `PoliticaDeRejeicao`, `AlocadorDeGrade` | Regras puras, testáveis sem banco |
| Repository | `api/*/infra` | Acesso a dados atrás de interface do Spring Data |
| DTO + Mapper | `api/*/api` | A API não expõe entidades; contrato estável |
| Value Object | `Dinheiro`, `CodigoImovel`, `HorarioGrade` | Validação e formatação num tipo só |
| Registro de eventos (log de domínio) | Tabela `evento` | Linha do tempo, Dashboard, Consumo e avisos a partir de um único registro |
| Strategy | Geradores de clipes, publicadores de rede, fontes | Trocar implementação por configuração |
| Registry / Plugin | Etapas da fábrica | Etapa nova sem alterar código existente |
| Pipeline | Execução do fluxo no gerador | Etapas sequenciais com artefatos por etapa |
| Adapter | Clientes de IA, redes sociais, XML | Isola APIs externas atrás de interfaces próprias |
| Specification | Regras de elegibilidade | Regras compostas por configuração |
| Idempotency Key | Chamadas pagas de IA, publicação | Retomar sem pagar ou publicar duas vezes |
| Retry com backoff exponencial | Erros transitórios na fábrica | Resiliência sem sobrecarregar provedores |
| Optimistic Locking | Edições no painel (`@Version`) | Concorrência sem travas longas |
| Update condicional | Toda mudança de estado | `WHERE estado = <esperado>` evita corrida entre painel e fábrica |
| Problem Details (RFC 9457) | Erros da API | Formato único de erro para o front |
| Server state no cliente | Front (TanStack Query) | Cache, polling e invalidação declarativos |
| URL como estado | Filtros, abas, item selecionado | Links compartilháveis, voltar do navegador funciona |

## 3. Backend (Java / Spring)

### 3.1 Estrutura de módulo

```
br.com.lumina.luminaria.<modulo>/
├── api/          Controllers + DTOs (records) + mapeadores. Sem regra de negócio.
├── aplicacao/    Casos de uso (@Service, @Transactional). Orquestram domínio e repositórios.
├── dominio/      Entidades, value objects, serviços de domínio, exceções de domínio. Sem Spring Web.
└── infra/        Repositórios, consultas nativas, clientes externos.
```

Dependências permitidas: `api → aplicacao → dominio`; `infra → dominio`. O domínio não conhece controller nem JDBC. Um módulo acessa outro **apenas** pela API pública do módulo (`<modulo>/aplicacao` ou eventos), nunca pelas entidades internas.

### 3.2 Caso de uso

```java
@Service
@RequiredArgsConstructor
public class AprovarPost {
    private final PostRepository posts;
    private final AgendarPost agendar;
    private final RegistroDeEventos eventos;

    @Transactional
    public AprovacaoResultado executar(UUID postId, Optional<OffsetDateTime> horarioEscolhido) {
        Post post = posts.buscar(postId).orElseThrow(NaoEncontrado::new);   // filtrado por tenant
        Agendamento ag = agendar.agendar(post, horarioEscolhido);
        post.aprovar();                                                     // regra no domínio
        eventos.registrar(TipoEvento.APROVADO, post, Map.of("agendadoPara", ag.agendadoPara()));
        return AprovacaoResultado.de(ag);
    }
}
```

Regras:

- Um caso de uso por classe, nome no infinitivo (`AprovarPost`, `RejeitarPost`, `GerarAgora`).
- `@Transactional` no caso de uso, nunca no controller nem no repositório.
- Entrada: tipos de domínio ou um `record` de comando; saída: `record` de resultado.
- Evento gravado **dentro** da mesma transação.

### 3.3 Domínio

- Entidades protegem invariantes: mudar estado só por método (`post.aprovar()`, `post.rejeitar(...)`), nunca por `setEstado`.
- Serviços de domínio são classes puras (sem Spring, sem I/O), testadas com tabelas de casos.
- Exceções de domínio herdam de `RegraDeNegocioException` e são mapeadas para 409/422 num único `@ControllerAdvice`.

### 3.4 API

- Recursos no plural, em português, kebab-case: `/api/v1/imoveis/{id}/linha-do-tempo`.
- Ações que não são CRUD como sub-recurso verbal: `POST /posts/{id}/aprovar`.
- DTOs são `record`s com Bean Validation; nunca entidades JPA na resposta.
- Paginação: `?pagina=&tamanho=`, resposta com `itens`, `total`, `pagina`.
- Datas em ISO-8601 com offset; dinheiro como string decimal.
- Erros em `ProblemDetail` com `type` estável (`/erros/franquia-esgotada`) e `detail` legível.

### 3.5 Persistência

- Spring Data JPA para escrita e leituras simples; SQL nativo (com `@Query` ou `JdbcClient`) para Dashboard, busca e relatórios.
- Toda leitura também em transação (`readOnly = true`), para o RLS receber o tenant.
- `@Version` em entidades editáveis pelo usuário.
- IDs UUID v7 gerados na aplicação.

### 3.6 Convenções Java

- Java 21: `record`, `sealed`, pattern matching e `Optional` no retorno de busca.
- Injeção por construtor, campos `final`.
- Logs com SLF4J e contexto MDC (`requestId`, `usuarioId`, `imobiliariaId`).
- Sem Lombok em entidades de domínio; permitido `@RequiredArgsConstructor` em serviços.

## 4. Fábrica (Python)

### 4.1 Estrutura

```
fabrica/
├── __main__.py        roteia o papel: ingestor | gerador | publicador
├── config.py          pydantic-settings
├── db/                pool, checagem de schema, consultas SQL por assunto (fila.py, imoveis.py…)
├── dominio/           tipos (dataclasses/pydantic), enums gerados, erros classificados
├── etapas/            registro + implementações por ponto de extensão
│   └── clientes/      etapas específicas de cliente, isoladas por pasta
├── ingestao/  geracao/  publicacao/  jobs/
└── tests/
```

### 4.2 Padrões

- **Strategy via `Protocol`** para cada ponto de extensão; implementações registradas com `@etapa("tipo")`.
- **Pipeline**: o gerador executa as etapas do fluxo em ordem, gravando `post.artefatos[etapa]` após cada uma.
- **Adapter** por provedor externo (IA, LLM, TTS, redes) com cliente HTTP próprio, timeouts explícitos e erros traduzidos para a hierarquia `ErroTransitorio` / `ErroPermanente` / `ErroConfiguracao`.
- **Consultas SQL** agrupadas por assunto em `db/`, parametrizadas, sem SQL montado por concatenação.
- **Lease + reaper** para itens em processamento; **chave de idempotência** em toda chamada paga.

### 4.3 Convenções Python

- Python 3.12, tipagem completa, `mypy --strict`, `ruff` (lint + format).
- Parâmetros de etapa validados com Pydantic ao carregar o fluxo.
- Sem estado global mutável; dependências passadas pelo construtor.
- Nenhuma transação aberta durante chamada externa.
- Logs com `structlog`, sempre com `post_id`/`imobiliaria_id` quando houver.

## 5. Frontend (Next.js)

- **Pastas por funcionalidade** (`features/<area>/{api.ts, hooks.ts, components/}`); `app/` só compõe páginas.
- **Hooks de dados**: todo acesso à API passa por `useXxx` / `useMutacaoXxx` com chaves centralizadas em `query-keys.ts`.
- **Componentes de apresentação** recebem dados por props e não chamam a API; componentes de página conectam hooks e apresentação.
- **Estado na URL** (`nuqs`) para filtros, abas e item selecionado.
- **Formulários** com `react-hook-form` + `zod`; o schema de validação espelha o DTO.
- **Tokens de design** no Tailwind (cores de estado, raios, sombras); nada de cor literal em componente.
- Texto da interface em arquivos `textos.ts` por funcionalidade.
- TypeScript `strict`; tipos da API gerados do OpenAPI, nunca escritos à mão.

## 6. Banco de dados

- Nomes em português, `snake_case`, tabelas no singular (`imovel`, `post`).
- Toda tabela de cliente tem `imobiliaria_id`, RLS habilitado e FK composta com o tenant quando referencia outra tabela de cliente.
- Estados como `text` + `CHECK`, gerados de `contrato/estados.yaml`.
- Regra que não pode falhar vira restrição: índice único parcial, `EXCLUDE`, `CHECK`.
- Migrations: `V<NNN>__<descricao>.sql`, uma preocupação por arquivo, nunca editadas depois de aplicadas, sempre compatíveis com a versão anterior da aplicação (*expand → contract*).

## 7. Testes

| Camada | Ferramenta | Obrigatório para |
|---|---|---|
| Domínio Java | JUnit 5 + AssertJ (tabelas de casos) | Toda regra de domínio |
| Caso de uso / API | Spring Boot Test + Testcontainers | Todo endpoint, incluindo teste de isolamento entre imobiliárias |
| Arquitetura | ArchUnit + Spring Modulith | Fronteiras de módulo e camadas |
| Fábrica | pytest + Testcontainers com migrations Flyway reais | Filas, franquia, transições, cada etapa |
| Contrato | Testes cruzados no CI | Enums, transições, criptografia de tokens, fluxos carregáveis |
| Front | Vitest + Testing Library + MSW; Playwright (360 px e 1440 px) | Fluxo principal de cada tela |

Nome de teste descreve o comportamento: `deveAbrirChamadoNaSegundaRejeicao`, `test_nao_marca_saida_quando_xml_vem_vazio`.

## 8. Git e revisão

- Branches: `main` (protegida) e `feat/…`, `fix/…`, `docs/…`, `chore/…`.
- Commits no padrão **Conventional Commits**, em português: `feat(aprovacao): encaixe automático na grade`.
- PR pequeno, com checklist:
  - [ ] Isolamento por imobiliária preservado (teste de isolamento passando)
  - [ ] Mudança de schema em migration nova e compatível com a versão anterior
  - [ ] Mudança de estado com update condicional e evento na mesma transação
  - [ ] Contrato (`contrato/`) regenerado se enums ou transições mudaram
  - [ ] Documentação em `docs/` atualizada se a mudança altera arquitetura, contrato ou comportamento
- Mudança estrutural (nova dependência de infraestrutura, novo estilo de integração, mudança de estado) exige ADR em [decisoes.md](decisoes.md).

## 9. Anti-padrões proibidos

| Proibido | Use em vez disso |
|---|---|
| `if imobiliaria_id == X` ou qualquer condicional por cliente | Configuração da imobiliária, plano ou etapa de fluxo |
| Receber `imobiliariaId` do cliente em endpoint não-Admin | Tenant da sessão (`TenantContext`) |
| DDL ou criação de tabela fora do Flyway | Migration nova |
| `UPDATE ... SET estado = ...` sem `WHERE estado = <esperado>` | Update condicional + verificação de linhas afetadas |
| Chamada HTTP entre `api` e `fabrica` | Escrita no banco + `NOTIFY` |
| Transação aberta durante chamada externa | Transações curtas antes e depois |
| Fila ou cache externo (Redis, RabbitMQ) sem ADR | Fila no PostgreSQL |
| Entidade JPA serializada na resposta da API | DTO `record` |
| Token, senha ou segredo em log, URL ou código | Variável de ambiente / campo cifrado |
| Python escrevendo `legenda_editada`, `decisao` ou `agendamento` | Apenas o painel escreve essas colunas |
| Editar uma versão de fluxo existente | Criar nova versão |
