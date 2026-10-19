# 06 — Backend Spring Boot

Monólito modular: um deploy, módulos com fronteiras claras (verificáveis com **Spring Modulith**). A regra de negócio fica em serviços de domínio pequenos e testáveis sem banco; o resto é CRUD bem feito.

## 1. Módulos

```mermaid
flowchart LR
    subgraph api["br.com.lumina.luminaria"]
        identidade["identidade<br/>login, usuários, senha"]
        imobiliaria["imobiliaria<br/>config, plano, franquia, grade"]
        carteira["carteira<br/>imóveis, fotos, busca, gerar"]
        aprovacao["aprovacao<br/>posts, decisões, rejeição"]
        agenda["agenda<br/>alocação, agendados"]
        publicacao["publicacao<br/>leitura, conexões de rede"]
        chamados["chamados"]
        avisos["avisos<br/>sino, ProcessadorDeEventos"]
        dashboard["dashboard<br/>consultas agregadas"]
        admin["admin<br/>visões globais"]
        compartilhado["compartilhado<br/>tenant, segurança, eventos, erros"]
    end

    carteira --> imobiliaria
    aprovacao --> agenda
    aprovacao --> chamados
    aprovacao --> carteira
    agenda --> imobiliaria
    avisos --> agenda
    admin --> imobiliaria & chamados & identidade
    dashboard -.->|"só leitura SQL"| compartilhado
```

Estrutura de cada módulo:

```
aprovacao/
├── api/            AprovacaoController, DTOs (records), mapeadores
├── aplicacao/      AprovarPost, RejeitarPost, NaoPostarImovel, EditarLegenda  (@Service @Transactional)
├── dominio/        Post, Decisao, PoliticaDeRejeicao, CategoriaRejeicao  (regras puras)
└── infra/          PostRepository, consultas nativas
```

Um caso de uso por classe (`AprovarPost.executar(...)`): fácil de achar, de testar e de mapear para o requisito.

## 2. Diagrama de classes — domínio

```mermaid
classDiagram
    direction TB

    class Imobiliaria {
        UUID id
        String nome
        boolean aprovacaoLigada
        ModoPublicacao modoPublicacao
        boolean encaixeAutomatico
        boolean monitoresHabilitados
        String xmlUrl
        +publicaAoAprovar() boolean
    }
    class PlanoImobiliaria {
        UUID id
        NivelVideo nivel
        Integer franquiaMensal
        int duracaoSegundos
        boolean motionGraphics
        boolean locucao
        LocalDate vigenteDe
        LocalDate vigenteAte
        +ilimitado() boolean
    }
    class Usuario {
        UUID id
        UUID imobiliariaId
        String nome
        String email
        Papel papel
        boolean ativo
    }
    class Imovel {
        UUID id
        UUID imobiliariaId
        String codigo
        Finalidade finalidade
        EstadoImovel estado
        int rejeicoesNoCiclo
        +podeGerar() boolean
        +registrarRejeicao() boolean
        +zerarCiclo()
    }
    class Post {
        UUID id
        UUID imobiliariaId
        UUID imovelId
        TipoPost tipo
        int numeroVersao
        Destino destino
        OrigemPost origem
        EstadoPost estado
        NivelVideo nivelVideo
        boolean gastaFranquia
        EscopoRefazer escopoRefazer
        String legendaIa
        String legendaEditada
        String hashtagsIa
        String hashtagsEditadas
        long versao
        +legendaFinal() String
        +editarLegenda(String, String)
        +voltarAoOriginal()
        +aprovar()
        +rejeitar()
        +proximaVersao(EscopoRefazer) Post
    }
    class Decisao {
        UUID id
        UUID postId
        UUID usuarioId
        AcaoDecisao acao
        Set~CategoriaRejeicao~ categorias
        String descricao
        String observacao
        Instant criadoEm
    }
    class GradeHorario {
        DayOfWeek diaSemana
        LocalTime hora
    }
    class Agendamento {
        UUID id
        UUID postId
        OffsetDateTime agendadoPara
        OrigemAgendamento origem
        Instant canceladoEm
        +cancelar()
    }
    class Publicacao {
        UUID id
        Rede rede
        EstadoPublicacao estado
        String urlPost
        String erro
    }
    class Chamado {
        UUID id
        UUID imovelId
        EstadoChamado estado
        ResolucaoChamado resolucao
        +resolver(ResolucaoChamado, String)
    }

    Imobiliaria "1" --> "*" PlanoImobiliaria
    Imobiliaria "1" --> "*" Usuario
    Imobiliaria "1" --> "*" Imovel
    Imobiliaria "1" --> "*" GradeHorario
    Imovel "0..1" --> "*" Post
    Post "1" --> "*" Decisao
    Post "1" --> "0..1" Agendamento
    Post "1" --> "*" Publicacao
    Imovel "1" --> "*" Chamado

    class PoliticaDeRejeicao {
        <<domain service>>
        +decidir(Imovel, Set~CategoriaRejeicao~) ResultadoRejeicao
        +escopo(Set~CategoriaRejeicao~) EscopoRefazer
        +categoriasPermitidas(Post) Set~CategoriaRejeicao~
    }
    class AlocadorDeGrade {
        <<domain service>>
        +proximoLivre(List~GradeHorario~, Set~OffsetDateTime~, ZonedDateTime agora) OffsetDateTime
        +proximosLivres(..., int n) List~OffsetDateTime~
    }
    class CalculadoraFranquia {
        <<interface>>
        +saldo(UUID imobiliariaId) SaldoFranquia
    }
    class SaldoFranquia {
        <<record>>
        Integer franquia
        int sobra
        int usados
        Integer disponivel
        boolean ilimitado
        +esgotada() boolean
        +percentualUsado() int
    }
    PoliticaDeRejeicao ..> Imovel
    AlocadorDeGrade ..> GradeHorario
    CalculadoraFranquia ..> SaldoFranquia

    class EstadoImovel {
        <<enumeration>>
        NAO_GERADO
        INELEGIVEL
        NA_FILA
        GERANDO
        AGUARDANDO_APROVACAO
        AGENDADO
        PUBLICADO
        EM_ANALISE
        FALHOU
        DESCARTADO
        SAIU_DA_CARTEIRA
        VENDIDO
        INDISPONIVEL
    }
    class EstadoPost {
        <<enumeration>>
        NA_FILA
        GERANDO
        AGUARDANDO_APROVACAO
        APROVADO_AUTO
        AGENDADO
        PUBLICANDO
        PUBLICADO
        REJEITADO
        DESCARTADO
        CANCELADO
        FALHOU
        NOS_MONITORES
    }
    class CategoriaRejeicao {
        <<enumeration>>
        VIDEO_MAL_RENDERIZADO
        ERRO_IA_VIDEO
        FOTO_MAL_ESCOLHIDA
        DADOS_ERRADOS
        LOCUCAO_COM_ERRO
        OUTRO
    }
```

Outros enums (gerados de `contrato/estados.yaml`): `NivelVideo {ESSENCIAL, PRO, MAX}`, `Destino {REDES, MONITORES}`, `TipoPost {IMOVEL, VENDIDO, PROPRIO}`, `Rede {INSTAGRAM, TIKTOK, YOUTUBE}`, `ModoPublicacao {PUBLICAR_AO_APROVAR, SEGUIR_GRADE}`, `EscopoRefazer {REMONTAR, NOVAS_FOTOS, NOVOS_CLIPES, TUDO}`.

## 3. Diagrama de classes — aplicação (aba Aprovação)

```mermaid
classDiagram
    direction LR
    class AprovacaoController {
        +pendentes(Pageable) Page~PostPendenteDto~
        +aprovar(UUID, AprovarRequest) AprovacaoResultadoDto
        +aprovarVarios(AprovarVariosRequest) List~AprovacaoResultadoDto~
        +rejeitar(UUID, RejeitarRequest) RejeicaoResultadoDto
        +naoPostar(UUID, NaoPostarRequest)
        +editarLegenda(UUID, EditarLegendaRequest) PostDto
        +horariosLivres(int n) List~OffsetDateTime~
    }
    class AprovarPost {
        +executar(UUID postId, OffsetDateTime horarioEscolhido) AprovacaoResultado
    }
    class RejeitarPost {
        +executar(UUID postId, RejeicaoCmd) RejeicaoResultado
    }
    class AgendarPost {
        +agendar(Post, Imobiliaria, OffsetDateTime escolhido) Agendamento
        +cancelar(UUID postId)
        +trocarHorario(UUID postId, OffsetDateTime novo)
    }
    class AbrirChamado {
        +executar(Imovel, Post, Decisao)
    }
    class RegistroDeEventos {
        +registrar(TipoEvento, UUID imovelId, UUID postId, Map dados)
    }
    class TenantContext {
        <<static>>
        +atual() Contexto
        +imobiliariaId() UUID
    }
    class PostRepository
    class ImovelRepository
    class AgendamentoRepository
    class DecisaoRepository
    class TravaImobiliaria {
        +travar(UUID imobiliariaId)
    }

    AprovacaoController --> AprovarPost
    AprovacaoController --> RejeitarPost
    AprovarPost --> AgendarPost
    AprovarPost --> PostRepository
    AprovarPost --> RegistroDeEventos
    AgendarPost --> AlocadorDeGrade
    AgendarPost --> TravaImobiliaria
    AgendarPost --> AgendamentoRepository
    RejeitarPost --> PoliticaDeRejeicao
    RejeitarPost --> AbrirChamado
    RejeitarPost --> ImovelRepository
    RejeitarPost --> DecisaoRepository
    RejeitarPost --> RegistroDeEventos
    AprovarPost ..> TenantContext
```

`TravaImobiliaria` = `SELECT pg_advisory_xact_lock(hashtextextended(?::text, 0))`: serializa alocação de grade e conta de franquia por imobiliária, sem travar as outras.

## 4. Casos de uso

### 4.1 Aprovar (RF-LUM-11)

```mermaid
sequenceDiagram
    autonumber
    participant C as AprovacaoController
    participant U as AprovarPost
    participant P as PostRepository
    participant G as AgendarPost
    participant L as AlocadorDeGrade
    participant DB as PostgreSQL

    C->>U: executar(postId, horarioEscolhido?)
    U->>P: buscar(postId)  [filtrado por @TenantId]
    alt não existe / outra imobiliária
        U-->>C: 404
    else estado ≠ aguardando_aprovacao
        U-->>C: 409 "este vídeo mudou de estado"
    end
    U->>G: agendar(post, imobiliaria, horarioEscolhido)
    G->>DB: pg_advisory_xact_lock(imobiliaria)
    alt publicar ao aprovar
        G->>G: horario = agora (origem imediato)
    else grade + encaixe automático
        G->>DB: grade + agendamentos ativos futuros
        G->>L: proximoLivre(grade, ocupados, agora SP)
    else grade sem encaixe
        G->>G: valida horarioEscolhido ∈ grade e livre
    end
    G->>DB: INSERT agendamento
    U->>DB: post.estado = agendado · imovel.estado = agendado · imovel.rejeicoes_no_ciclo = 0
    U->>DB: INSERT decisao(aprovar) · INSERT evento
    U-->>C: {agendadoPara, textoAmigavel: "qua 15/10 às 18h"}
```

Grade sem encaixe automático: o front primeiro chama `GET /aprovacao/horarios-livres?n=8` e mostra a lista com o mais próximo selecionado; o POST vem com o horário escolhido. Se outro usuário pegou o horário nesse meio-tempo, o índice único devolve conflito → **409** com a próxima sugestão.

### 4.2 Alocador de grade (RN-LUM-04)

```java
public OffsetDateTime proximoLivre(List<GradeHorario> grade, Set<OffsetDateTime> ocupados, ZonedDateTime agora) {
    if (grade.isEmpty()) throw new GradeVaziaException();
    var porDia = grade.stream().collect(groupingBy(GradeHorario::diaSemana,
                     mapping(GradeHorario::hora, toCollection(TreeSet::new))));
    LocalDate dia = agora.toLocalDate();
    for (int i = 0; i < 366; i++, dia = dia.plusDays(1)) {          // "sem limite", com teto de segurança
        for (LocalTime hora : porDia.getOrDefault(dia.getDayOfWeek(), new TreeSet<>())) {
            var candidato = ZonedDateTime.of(dia, hora, SAO_PAULO);
            if (candidato.isAfter(agora) && !ocupados.contains(candidato.toOffsetDateTime()))
                return candidato.toOffsetDateTime();
        }
    }
    throw new GradeLotadaException();
}
```

Função pura: testada com tabela de casos (fim de semana, virada de mês, horário passando agora, grade cheia).

### 4.3 Rejeitar (RF-LUM-15, RN-LUM-01/02)

```mermaid
sequenceDiagram
    autonumber
    participant C as AprovacaoController
    participant U as RejeitarPost
    participant R as PoliticaDeRejeicao
    participant DB as PostgreSQL

    C->>U: executar(postId, {categorias, descricao, observacao})
    U->>DB: post (estado = aguardando_aprovacao) + imóvel
    U->>R: categoriasPermitidas(post)
    alt categoria não permitida (ex.: ERRO_IA em post Essencial)
        U-->>C: 422
    end
    U->>R: decidir(imovel, categorias)
    U->>DB: post → rejeitado · INSERT decisao · evento rejeitado
    alt 1ª rejeição
        U->>DB: INSERT post vN+1 (na_fila, prioridade ALTA, gasta_franquia=false, escopo)
        U->>DB: imovel → na_fila · rejeicoes_no_ciclo = 1
        U-->>C: {resultado: REFAZENDO, versao: 2}
    else 2ª rejeição
        U->>DB: imovel → em_analise · INSERT chamado
        U-->>C: {resultado: EM_ANALISE, mensagem: RF-LUM-17}
    end
```

### 4.4 Processador de eventos (avisos)

Job `@Scheduled(fixedDelay = 5s)` + acordado por `NOTIFY lum_evento`. Roda com `TenantContext` de sistema (`admin = true`, `ator = sistema`).

| Evento lido | Ação |
|---|---|
| `video_gerado` (post `aguardando_aprovacao`) | aviso "vídeo novo para aprovar" para a imobiliária |
| `video_gerado` (post `aprovado_auto`) | `AgendarPost` com a mesma regra do botão Aprovar |
| `saiu_da_carteira` | aviso "o imóvel 669 saiu do seu sistema. Foi vendido?" |
| `publicacao_falhou` / `conexao_expirada` | aviso vermelho + alerta no Admin |
| `franquia_80` / `franquia_esgotada` | aviso (emitido uma vez por mês, chave única) |
| `chamado_resolvido` | aviso para quem rejeitou |

Mais um `@Scheduled` a cada hora: pendentes há 48 h → lembrete (RN-LUM-03), sem duplicar (chave única `tipo + post_id`).

## 5. API REST (visão geral)

Prefixo `/api/v1`, JSON, erros em `ProblemDetail`. Contrato completo virá em OpenAPI (springdoc) num documento próprio.

| Recurso | Endpoints principais | RF |
|---|---|---|
| Auth | `POST /auth/login`, `POST /auth/logout`, `GET /auth/eu`, `POST /auth/esqueci-senha`, `POST /auth/redefinir-senha` | 01, 03 |
| Aprovação | `GET /aprovacao/pendentes`, `GET /aprovacao/agendados`, `GET /aprovacao/horarios-livres`, `POST /posts/{id}/aprovar`, `POST /posts/aprovar-varios`, `POST /posts/{id}/rejeitar`, `PATCH /posts/{id}/legenda`, `GET /posts/{id}/midia` (URL assinada), `PATCH /agendamentos/{postId}`, `DELETE /agendamentos/{postId}` | 10–17 |
| Imóveis | `GET /imoveis?estado=&finalidade=&tipo=&bairro=&precoMin=&precoMax=&quartos=&q=&ordem=&pagina=`, `GET /imoveis/{id}`, `GET /imoveis/{id}/estado`, `GET /imoveis/{id}/linha-do-tempo`, `POST /imoveis/{id}/gerar`, `POST /imoveis/{id}/nao-postar`, `POST /imoveis/{id}/desfazer-descarte`, `POST /imoveis/{id}/confirmar-saida`, `PUT /imoveis/{id}/fotos-escolhidas` | 16, 18, 20–27, 30 |
| Dashboard | `GET /dashboard/resumo?de=&ate=`, `GET /dashboard/publicados-por-periodo`, `GET /dashboard/carteira-por-estado`, `GET /dashboard/ultimos-publicados` | 40–46 |
| Configurações | `GET/PUT /configuracoes/publicacao`, `GET/PUT /configuracoes/grade`, `GET /configuracoes/horarios-recomendados`, `GET/PUT /configuracoes/imobiliaria`, `GET /configuracoes/plano`, `GET /configuracoes/redes`, `GET /configuracoes/limites-legenda` | 50–55 |
| Franquia | `GET /franquia`, `GET /franquia/extrato` | 80–82 |
| Avisos | `GET /avisos`, `POST /avisos/{id}/lido`, `POST /avisos/lidos` | 70 |
| Monitores | `GET /monitores/videos` | 31 |
| Apoio às telas ([08](08-especificacao-de-telas.md)) | `GET /aprovacao/resumo`, `GET /aprovacao/agenda?dias=7`, `GET /imoveis/contagens`, `GET /posts/{id}/comparar`, `POST /monitores/videos`, `POST /auth/definir-senha` | — |
| Admin | `/admin/imobiliarias` (+ `POST` nova), `/admin/imobiliarias/{id}/ler-xml`, `/admin/publicacoes/{id}/tentar-de-novo`, `/admin/saude`, `/admin/consumo.csv`, `/admin/usuarios/{id}/reativar`, `/admin/imobiliarias/{id}/plano`, `/admin/chamados`, `/admin/chamados/{id}/resolver`, `/admin/falhas`, `/admin/falhas/{postId}/tentar-de-novo`, `/admin/consumo`, `/admin/usuarios`, `/admin/horarios-recomendados`, `/admin/imobiliarias/{id}/entrar-como` | 60–67 |

## 6. Bibliotecas

| Necessidade | Escolha |
|---|---|
| Web/validação | `spring-boot-starter-web`, `-validation` |
| Persistência | `spring-boot-starter-data-jpa` (Hibernate 6), `flyway-core` + `flyway-database-postgresql` |
| Segurança | `spring-boot-starter-security`, `spring-session-jdbc` |
| Rate limit | `bucket4j-core` |
| Módulos | `spring-modulith-starter-core` (+ teste de fronteiras) |
| Documentação da API | `springdoc-openapi-starter-webmvc-ui` (só fora de produção) |
| MinIO | `io.minio:minio` (URL assinada) |
| Observabilidade | `spring-boot-starter-actuator` (health para o EasyPanel), logs JSON |
| Testes | JUnit 5, AssertJ, Testcontainers (PostgreSQL), ArchUnit |

Sem Lombok nas entidades de domínio (records nos DTOs bastam); sem MapStruct na v1 (mapeadores manuais pequenos).
