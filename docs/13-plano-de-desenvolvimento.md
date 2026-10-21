# 13 — Plano de desenvolvimento do Luminaria

Data: 21/10/2026 · Situação: planejamento inicial; implementação ainda não iniciada.

## 1. Objetivo e base do plano

Entregar o SaaS multi-imobiliária descrito em [00 — Visão geral](00-visao-geral.md): importar a carteira XML, gerar vídeos, permitir revisão e agendamento e publicar por rede com rastreabilidade. Este plano transforma os documentos 00 a 12 e as ADRs em uma sequência executável.

Na revisão inicial, o repositório contém documentação e instruções; as pastas de aplicação descritas ainda precisam ser criadas. Listagens de comandos e exemplos nos documentos são especificações, não funcionalidades já entregues.

**Confirmado pelo responsável em 21/10/2026:** implantação no **EasyPanel** e banco **PostgreSQL**. Adotar PostgreSQL 16 como versão de referência, conforme a ADR-13. SQLite não fará parte da aplicação nova. Se houver SQLite no pipeline anterior, será apenas uma fonte da migração de dados.

### Decisões que orientam a execução

| Área | Base de implementação |
|---|---|
| Backend | Java 21, Spring Boot 3, monólito modular |
| Frontend | Next.js, TypeScript, TanStack Query, identidade Lumina |
| Fábrica | Python 3.12; ingestor, gerador e publicador |
| Integração | Somente PostgreSQL entre API e fábrica; filas com SKIP LOCKED, sinais com LISTEN/NOTIFY e polling de recuperação |
| Schema | Flyway como único responsável; schema luminaria; migrations compatíveis com a aplicação anterior |
| Mídia | MinIO privado; URLs assinadas após autorização |
| Isolamento | Tenant da sessão, filtro na aplicação e RLS no PostgreSQL |
| Automação | Fluxos JSON imutáveis e versionados, executados por etapas Python |
| Publicação | Instagram, TikTok e YouTube independentes |
| Operação | EasyPanel; imagens com a mesma tag de commit; um deploy atende todas as imobiliárias |

Há **cinco processos próprios**: API, web, ingestor, gerador e publicador. Isso não significa cinco serviços no ambiente inteiro: PostgreSQL, MinIO e compositor também precisam ser provisionados ou conectados; o proxy é fornecido/configurado na infraestrutura. São três imagens próprias, pois os três workers usam a mesma imagem Python.

A discussão sobre transformar Python em n8n não equivale a uma decisão arquitetural. O plano segue as ADRs 02, 03, 10, 11 e 13. Uma substituição futura exige avaliação e ADR própria, preservando transações, isolamento, retomada e matriz de escrita.

## 2. Como executar e acompanhar

O plano passa a ter **32 fases menores**, cada uma com objetivo, dependências, blocos de execução e aceite próprios. Cada bloco deve originar uma tarefa rastreável e um PR pequeno sempre que possível. A ordem abaixo privilegia entregas que podem ser demonstradas de ponta a ponta. Backend, interface, testes e documentação da funcionalidade são concluídos no mesmo bloco; não deixar toda a integração para o final.

- **P0:** necessário antes de operar o piloto real.
- **P1:** necessário para completar a v1 documentada.
- **P2:** evolução posterior, fora do compromisso da v1.
- **Estado inicial de todos os blocos:** a fazer. Este documento não registra funcionalidades como concluídas.
- **Responsáveis:** a atribuir antes de iniciar cada fase; papéis sugeridos são produto, backend, frontend, fábrica e operação.
- **Estimativas:** definir após o inventário da F01 e recalibrar ao concluir a primeira entrega integrada. Não há prazo fechado nem capacidade de equipe confirmada.

Um bloco termina com evidência verificável: PR, resultado de teste, demonstração ou registro de homologação. A fase só termina quando seu critério de saída é atendido. Se uma decisão pendente mudar um contrato, atualizar a especificação e a ADR antes de implementar a área.

## 3. Mapa das 32 fases

Os identificadores B00–B31 foram preservados para manter a rastreabilidade da versão anterior. Agora cada um pertence a uma fase própria (F01–F32), com tarefas menores numeradas. Concluir uma fase exige cumprir seu aceite, além de marcar tarefas. As dependências indicam entregas necessárias; a numeração não impede preparação antecipada de atividades independentes.

| Fase | Entrega | Prioridade | Depende de |
|---|---|---|---|
| F01 | Inventário do pipeline anterior | P0 | — |
| F02 | Resolver lacunas técnicas antes do schema | P0 | F01 |
| F03 | Preparar provedores, contas e infraestrutura | P0 | F01; iniciar junto de F02 |
| F04 | Estrutura e ferramentas | P0 | F01–F02 |
| F05 | Contrato compartilhado | P0 | F04 |
| F06 | Ambiente local e CI inicial | P0 | F04–F05 |
| F07 | Schema e migrations | P0 | F05–F06 |
| F08 | Isolamento multi-imobiliária | P0 | F07 |
| F09 | Franquia, concorrência e tempo | P0 | F07–F08 |
| F10 | Identidade e sessão | P0 | F08 |
| F11 | Base visual e navegação | P0 | F05, F10 |
| F12 | Cadastro operacional mínimo | P0 | F09–F11 |
| F13 | Ingestor XML | P0 | F12 |
| F14 | Catálogo e detalhe | P0 | F13 |
| F15 | Motor da fábrica e fila | P0 | F05, F09, F14 |
| F16 | Geração Essencial | P0 | F15 |
| F17 | Revisão inicial no painel | P0 | F11, F14, F16 |
| F18 | Aprovação e agendamento | P0 | F12, F17 |
| F19 | Rejeição, refação e chamado | P0 | F18 |
| F20 | Descarte, saída da carteira e eventos | P0 | F13, F18–F19 |
| F21 | OAuth e proteção dos tokens | P0 | F03, F08, F10, F12 |
| F22 | Publicador por rede | P0 | F18, F20–F21 |
| F23 | Entrega de homologação no EasyPanel | P0 | F06, F17–F18 |
| F24 | Piloto controlado | P0 | F19–F23; F22 na primeira rede liberada |
| F25 | Pro, Max e acessórios | P1 | F16, F24; provedores de F03 |
| F26 | Gestão comercial e fluxos | P1 | F09, F12, F15, F25 |
| F27 | Produtividade e monitores | P1 | F19–F20, F25–F26 |
| F28 | Dashboard, avisos e configurações | P1 | F17, F20, F22, F26–F27 |
| F29 | Administração completa | P1 | F12, F19, F21–F23, F26–F28 |
| F30 | Migração do pipeline anterior | P0/P1 | F01, F07–F09, F24 |
| F31 | Homologação funcional e resiliência | P0/P1 | F25–F30; três redes liberadas |
| F32 | Corte e operação assistida | P0/P1 | F31 |

### Marcos de entrega

| Marco | Fases necessárias | Demonstração |
|---|---|---|
| Base executável | F01–F06 | Clone limpo, projetos executando e CI ativo |
| Acesso isolado | F07–F12 | Duas imobiliárias cadastradas sem acesso cruzado |
| Primeira carteira | F13–F14 | XML importado e imóveis consultáveis |
| Primeiro vídeo revisável | F15–F17 | Vídeo Essencial, player e legenda editável |
| Controle editorial | F18–F20 | Aprovação, grade, refação e chamado |
| Piloto real | F21–F24 | Publicação em conta de teste da primeira rede liberada |
| v1 funcional completa | F25–F29, com fases anteriores | Planos, monitores, Dashboard e Admin completos |
| Produção estabilizada | F30–F32, com fases anteriores | Migração conciliada, homologação e operação assistida |

A F03 começa cedo e continua acompanhando permissões externas. A F23 pode ser preparada antes de todos os adaptadores sociais ficarem prontos. O piloto exige ao menos uma rede real liberada; a homologação da v1 exige as três. Nenhuma fase concluída é reaberta apenas por essa reorganização: o estado inicial continua a fazer porque ainda não houve implementação.

## 4. Execução fase a fase

### F01 — Inventário do pipeline anterior [P0]

**Objetivo:** Mapear o que existe e o que precisa ser migrado.

**Entrada:** —. **Referência anterior:** B00. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F01.01** — Localizar código Python existente, compositor, modelos visuais, origem dos XMLs e armazenamento atual.
- [ ] **F01.02** — Identificar banco anterior, tabelas, estados, IDs, credenciais necessárias e histórico que precisa ser preservado, sem copiar segredos para o repositório.
- [ ] **F01.03** — Mapear o que será reaproveitado, adaptado ou refeito; registrar volume e qualidade dos dados.
- [ ] **F01.04** — Produzir inventário e matriz origem → destino para a migração da ADR-14.

**Aceite:** origem dos dados e dependências acessíveis e documentadas; itens indisponíveis possuem responsável e impacto explícitos. Não presumir que o pipeline anterior está neste repositório.

### F02 — Resolver lacunas técnicas antes do schema [P0]

**Objetivo:** Fechar decisões que afetam contratos e consistência.

**Entrada:** F01. **Referência anterior:** B01. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F02.01** — Revisar a matriz de escrita: o Admin pede retry por rede, embora a tabela resumida atribua publicação ao Python; definir permissões de coluna e transições exatas.
- [ ] **F02.02** — Definir consumo de franquia em falha, retomada e reprocessamento, garantindo uma única cobrança lógica por post.
- [ ] **F02.03** — Definir confirmação de lease por tentativa para impedir conclusão de um processamento antigo após nova reserva.
- [ ] **F02.04** — Resolver entrega de eventos com commits fora de ordem: a janela fixa de 2 segundos descrita no documento 02 precisa de teste com transação longa; não aceitá-la como garantia de não perder eventos.
- [ ] **F02.05** — Confirmar RLS com roles reais, contexto administrativo e escopo dos workers; testar reset do contexto no pool.
- [ ] **F02.06** — Revisar exemplos SQL antes de convertê-los em migrations: ordem das FKs, extensões necessárias (incluindo btree_gist), expressões indexáveis e comparação de versões Flyway.
- [ ] **F02.07** — Conciliar checagem estrita de versão do worker com a janela expand/contract e o procedimento de reinício/rollback.
- [ ] **F02.08** — Revisar a máquina de estados para indisponibilidade durante geração/publicação e aprovação automática.

**Aceite:** decisões registradas nos documentos afetados e, quando estruturais, em ADR; cenários críticos convertidos em casos de teste. Este bloco não altera silenciosamente decisões existentes.

### F03 — Preparar provedores, contas e infraestrutura [P0]

**Objetivo:** Disponibilizar acessos e acompanhar liberações externas.

**Entrada:** F01; iniciar junto de F02. **Referência anterior:** B02. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F03.01** — Confirmar VPS, domínio, EasyPanel, registry, SMTP, MinIO e disponibilidade do compositor.
- [ ] **F03.02** — Definir provedores reais de clipes Pro/Max, LLM e locução, com orçamento inicial de homologação.
- [ ] **F03.03** — Iniciar cadastro e obtenção das permissões dos aplicativos nas três redes; confirmar requisitos atuais nos documentos oficiais durante a implementação.
- [ ] **F03.04** — Disponibilizar XML anonimizado e contas de teste; estabelecer uma imobiliária piloto.
- [ ] **F03.05** — Atribuir responsáveis e dimensionar a primeira fase.

**Aceite:** checklist de acessos por responsável, dependências externas e limites de gasto definidos. Uma rede sem permissão de publicação real permanece bloqueada para lançamento público, mesmo se funcionar com adaptador falso.

### F04 — Estrutura e ferramentas [P0]

**Objetivo:** Permitir compilar e executar os três projetos.

**Entrada:** F01–F02. **Referência anterior:** B03. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F04.01** — Criar api/, web/, fabrica/, contrato/ e infra/ conforme documentos 10 e 12.
- [ ] **F04.02** — Inicializar projetos e versões compatíveis com a stack documentada; registrar dependências em arquivos de lock.
- [ ] **F04.03** — Criar configuração por ambiente, arquivos de exemplo sem segredos e instruções reproduzíveis.
- [ ] **F04.04** — Implementar estrutura modular, verificações de fronteira e endpoints básicos de saúde.

**Aceite:** um clone limpo permite instalar, compilar e executar os três projetos; nenhum comando documentado depende de arquivo inexistente.

### F05 — Contrato compartilhado [P0]

**Objetivo:** Manter estados e tipos consistentes entre linguagens.

**Entrada:** F04. **Referência anterior:** B04. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F05.01** — Criar estados.yaml, fluxo.schema.json e gerador de enums, transições e cores de estado.
- [ ] **F05.02** — Gerar artefatos Java, Python, TypeScript e SQL; versionar os resultados.
- [ ] **F05.03** — Definir contrato inicial OpenAPI e geração dos tipos consumidos pelo frontend.
- [ ] **F05.04** — Validar o fluxo padrão e detectar divergência de artefatos no CI.

**Aceite:** regenerar não produz diferenças inesperadas; uma mudança de contrato inconsistente falha no CI.

### F06 — Ambiente local e CI inicial [P0]

**Objetivo:** Reproduzir o ambiente e validar mudanças automaticamente.

**Entrada:** F04–F05. **Referência anterior:** B05. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F06.01** — Criar Compose de desenvolvimento com PostgreSQL, MinIO, compositor ou substituto local de teste e workers.
- [ ] **F06.02** — Implementar provedores falsos de geração/publicação, desligando chamadas pagas por padrão.
- [ ] **F06.03** — Preparar testes Java, web e Python; CI deve validar também a fábrica quando migrations mudarem.
- [ ] **F06.04** — Criar dados sintéticos de duas imobiliárias apenas em desenvolvimento/testes.

**Aceite:** ambiente local sobe de forma reproduzível e o CI executa as verificações pertinentes.

### F07 — Schema e migrations [P0]

**Objetivo:** Criar o modelo relacional com integridade no PostgreSQL.

**Entrada:** F05–F06. **Referência anterior:** B06. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F07.01** — Implementar progressivamente entidades dos documentos 04 e 11, infraestrutura de sessão/eventos/filas e fluxo padrão.
- [ ] **F07.02** — Criar FKs compostas, unicidades, restrições de plano, agendamento e produção por imóvel.
- [ ] **F07.03** — Implementar transições por ator, updates condicionais, eventos transacionais e sinais após commit.
- [ ] **F07.04** — Separar usuário dono do schema, API e workers; Python nunca executa DDL.
- [ ] **F07.05** — Não copiar exemplos SQL sem executar a sequência completa de migrations.

**Aceite:** banco vazio migra com sucesso; reaplicar não altera migrations aplicadas; transições e vínculos inválidos são recusados.

### F08 — Isolamento multi-imobiliária [P0]

**Objetivo:** Impedir acesso cruzado entre imobiliárias.

**Entrada:** F07. **Referência anterior:** B07. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F08.01** — Implementar RLS, TenantContext, contexto transacional e permissões conforme matriz revisada.
- [ ] **F08.02** — Provar acesso fechado sem tenant e isolamento inclusive em tabelas filhas, consultas nativas e mídia.
- [ ] **F08.03** — Tratar acesso administrativo explicitamente e registrar ações sensíveis.
- [ ] **F08.04** — Criar suíte de isolamento extensível a cada endpoint novo.

**Aceite:** role da API sem contexto vê zero dados de cliente; imobiliária A não lê nem altera recursos de B; pool não reaproveita tenant anterior.

### F09 — Franquia, concorrência e tempo [P0]

**Objetivo:** Garantir saldo e reservas corretos sob concorrência.

**Entrada:** F07–F08. **Referência anterior:** B08. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F09.01** — Implementar fn_saldo_franquia, reserva com trava por imobiliária e consumo idempotente.
- [ ] **F09.02** — Cobrir plano ilimitado, sobras, virada do mês, fechamento repetido e mudança de plano.
- [ ] **F09.03** — Uniformizar UTC no armazenamento e America/Sao_Paulo nos cálculos comerciais.
- [ ] **F09.04** — Implementar primitivas de lease, reserva de fila e proteção contra worker atrasado.

**Aceite:** dois geradores não consomem a última vaga simultaneamente; retry não duplica consumo; resultados da API e da fábrica concordam.

### F10 — Identidade e sessão [P0]

**Objetivo:** Autenticar e revogar acesso com segurança.

**Entrada:** F08. **Referência anterior:** B09. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F10.01** — Login/logout, sessão JDBC, cookie seguro, CSRF e proteção contra abuso.
- [ ] **F10.02** — Convite e recuperação de senha por token de uso único; SMTP em homologação.
- [ ] **F10.03** — Desativação revoga sessões; Admin e cliente recebem permissões distintas.
- [ ] **F10.04** — Implementar acesso inicial administrativo por procedimento controlado, sem senha fixa em seed de produção.

**Aceite:** login, convite, expiração, redefinição, revogação e autorização testados; tenant vem exclusivamente da sessão.

### F11 — Base visual e navegação [P0]

**Objetivo:** Entregar navegação utilizável no celular e desktop.

**Entrada:** F05, F10. **Referência anterior:** B10. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F11.01** — Aplicar UI Kit descrito nos documentos 07 e 08; casca cliente/Admin, login e estados de erro.
- [ ] **F11.02** — Cliente HTTP com CSRF e ProblemDetail; cache limpo ao sair/trocar conta.
- [ ] **F11.03** — Layout móvel de 360 px e desktop, navegação por teclado e foco visível.
- [ ] **F11.04** — Gerar tipos de API; servidor continua sendo responsável por autorizar.

**Aceite:** login leva à área correta; estados carregando/vazio/erro e expiração de sessão funcionam nos dois tamanhos.

### F12 — Cadastro operacional mínimo [P0]

**Objetivo:** Permitir preparar um cliente sem acesso manual ao banco.

**Entrada:** F09–F11. **Referência anterior:** B11. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F12.01** — Admin cadastra imobiliária, plano Essencial, XML, fluxo padrão, grade e usuários.
- [ ] **F12.02** — Cliente consulta plano e altera dados/configurações autorizados.
- [ ] **F12.03** — Auditabilidade de mudanças e upload seguro de logo.
- [ ] **F12.04** — Campos de conexão das redes são preparados para F21–F22.

**Aceite:** cadastrar uma nova imobiliária não exige deploy nem edição direta no banco.

### F13 — Ingestor XML [P0]

**Objetivo:** Transformar XML em carteira normalizada sem perdas indevidas.

**Entrada:** F12. **Referência anterior:** B12. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F13.01** — Reaproveitar/adaptar parser VivaReal após inventário; normalizar dados, fotos e hash.
- [ ] **F13.02** — Detectar novos/alterados/ausentes, executar leitura periódica e comando Ler agora.
- [ ] **F13.03** — Proteger XML vazio, incompleto, falha de download e remoção em massa; validar URLs externas e impor limites de download.
- [ ] **F13.04** — Registrar ingestões e eventos; impedir duplicação e alteração indevida de vídeos já gerados.

**Aceite:** XML repetido não duplica imóveis; falha/vazio não remove carteira; alterações e saídas válidas ficam rastreáveis.

### F14 — Catálogo e detalhe [P0]

**Objetivo:** Permitir localizar e consultar imóveis e histórico.

**Entrada:** F13. **Referência anterior:** B13. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F14.01** — API e telas de catálogo, filtros/ordenação na URL, paginação, contagens e busca tolerante.
- [ ] **F14.02** — Detalhe com fotos, estados, versões e linha do tempo.
- [ ] **F14.03** — MinIO privado e URLs assinadas após autorização; carregar mídia sob demanda.
- [ ] **F14.04** — Preparar ações de geração e seleção de fotos.

**Aceite:** usuário consulta apenas a própria carteira e compartilha URL de filtro; recurso de outra imobiliária retorna 404.

### F15 — Motor da fábrica e fila [P0]

**Objetivo:** Executar trabalho em fila com retomada segura.

**Entrada:** F05, F09, F14. **Referência anterior:** B14. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F15.01** — Implementar registro de etapas, validação Pydantic e fluxo padrão versionado.
- [ ] **F15.02** — Reservar post, congelar fluxo/plano, validar franquia e concluir por transações curtas.
- [ ] **F15.03** — Implementar retries classificados, heartbeat, reaper, desligamento e retomada de artefatos.
- [ ] **F15.04** — Nunca manter transação aberta durante rede, IA, FFmpeg ou MinIO.

**Aceite:** queda do worker e perda de NOTIFY não perdem trabalho; réplicas não processam a mesma reserva; resposta antiga não sobrescreve a tentativa atual.

### F16 — Geração Essencial [P0]

**Objetivo:** Produzir vídeo Essencial e legenda a partir de um imóvel.

**Entrada:** F15. **Referência anterior:** B15. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F16.01** — Selecionar fotos válidas e respeitar escolha do cliente.
- [ ] **F16.02** — Gerar zoom FFmpeg, integrar compositor, produzir vídeo vertical/capa e legenda por adaptador LLM.
- [ ] **F16.03** — Salvar artefatos e mídia; distinguir mensagem técnica e mensagem para o usuário.
- [ ] **F16.04** — Implementar Gerar agora e prioridade; limites diários e bloqueio por franquia.

**Aceite:** imóvel de teste produz vídeo reproduzível, capa e legenda; arquivo é reproduzível no painel e pertence ao tenant correto.

### F17 — Revisão inicial no painel [P0]

**Objetivo:** Permitir assistir e editar o conteúdo gerado.

**Entrada:** F11, F14, F16. **Referência anterior:** B16. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F17.01** — Lista de pendentes, detalhe/player, status e estimativa, atualização automática condicional.
- [ ] **F17.02** — Edição de legenda/hashtags separada da IA; limites por rede vindos da API.
- [ ] **F17.03** — Mostrar franquia e erros acionáveis.
- [ ] **F17.04** — Acrescentar notificações mínimas de vídeo pronto e falha.

**Aceite:** demonstrar XML → imóvel → gerar → vídeo pendente → edição persistida após recarregar, sem publicação automática indevida.

### F18 — Aprovação e agendamento [P0]

**Objetivo:** Transformar aprovação em agendamento válido.

**Entrada:** F12, F17. **Referência anterior:** B17. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F18.01** — Aprovar ao momento atual ou conforme grade, com/sem encaixe automático.
- [ ] **F18.02** — Alterar/cancelar agendamento, horários recomendados e visualização dos próximos sete dias.
- [ ] **F18.03** — Conflito de horário responde 409; uma única alocação ativa por post.
- [ ] **F18.04** — Aprovação desligada usa aprovado_auto e alocação pelo Java, nunca pelo Python.

**Aceite:** duas aprovações concorrentes não ocupam o mesmo horário; grade vazia e mudança de estado possuem respostas claras; nenhum envio começa antes de agendamento válido.

### F19 — Rejeição, refação e chamado [P0]

**Objetivo:** Tratar insatisfação com refação e suporte rastreáveis.

**Entrada:** F18. **Referência anterior:** B18. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F19.01** — Primeira rejeição cria versão prioritária sem franquia adicional; reaproveitar etapas conforme categoria.
- [ ] **F19.02** — Segunda rejeição abre chamado e bloqueia geração automática do imóvel.
- [ ] **F19.03** — Admin compara versões e resolve por refazer, devolver ou descartar; registrar nota para cliente e evento.
- [ ] **F19.04** — Validar categorias pelo nível/acessórios usados no post, não pelo plano atual.

**Aceite:** duas rejeições seguem a máquina de estados, sem terceira geração automática; edição manual não é sobrescrita.

### F20 — Descarte, saída da carteira e eventos [P0]

**Objetivo:** Registrar decisões sobre imóveis e entregar avisos sem perdas.

**Entrada:** F13, F18–F19. **Referência anterior:** B19. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F20.01** — Não postar, desfazer descarte, confirmar venda/locação e post Vendido somente após confirmação.
- [ ] **F20.02** — Processar eventos com retomada e deduplicação conforme decisão do B01; notificações e lembrete de 48 h.
- [ ] **F20.03** — Cobrir concorrência entre ingestão, aprovação e geração.
- [ ] **F20.04** — Documentar seções da v1 que ainda estarão indisponíveis no piloto.

**Aceite:** replay não duplica efeitos; eventos não são perdidos por commits fora de ordem; saída do XML não publica Vendido sozinha.

### F21 — OAuth e proteção dos tokens [P0]

**Objetivo:** Conectar contas sociais e proteger suas credenciais.

**Entrada:** F03, F08, F10, F12. **Referência anterior:** B20. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F21.01** — Conectar/reconectar redes, validar callbacks e vincular conta ao tenant autorizado.
- [ ] **F21.02** — Implementar AES-256-GCM compatível Java/Python, renovação e expiração.
- [ ] **F21.03** — Exibir rede desconectada e manter publicação aguardando conexão.
- [ ] **F21.04** — Validar permissões reais, escopos e limites atuais usando documentação oficial.

**Aceite:** teste cruzado de criptografia passa; tokens não aparecem no navegador/logs; reconexão retoma apenas itens autorizados.

### F22 — Publicador por rede [P0]

**Objetivo:** Publicar cada rede independentemente e recuperar falhas.

**Entrada:** F18, F20–F21. **Referência anterior:** B21. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F22.01** — Implementar adaptadores Instagram, TikTok e YouTube, com tentativas e resultado por rede.
- [ ] **F22.02** — Publicar agendamento vencido, preservar legenda editada e registrar atraso.
- [ ] **F22.03** — Reconciliar confirmação externa em timeout/queda antes de repetir; não prometer exactly-once sem suporte do provedor.
- [ ] **F22.04** — Permitir retry administrativo somente da rede que falhou; uma confirmação marca post publicado sem cancelar outras redes.

**Aceite:** sucesso em uma rede e falha em outra ficam independentes; retry não repete publicação já confirmada; conta desconectada aguarda.

### F23 — Entrega de homologação no EasyPanel [P0]

**Objetivo:** Executar a stack em homologação no EasyPanel.

**Entrada:** F06, F17–F18. **Referência anterior:** B22. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F23.01** — Criar imagens API/web/fábrica com tag SHA e configurar os cinco processos.
- [ ] **F23.02** — Provisionar/conectar PostgreSQL, MinIO e compositor; volumes, rede interna, HTTPS e proxy na mesma origem.
- [ ] **F23.03** — Separar homologação e produção; healthchecks, limites, logs e segredos por papel.
- [ ] **F23.04** — Automatizar CI e publicação de imagens; implantar API/migrations → workers → web e testar rollback de aplicação.
- [ ] **F23.05** — Configurar backup PostgreSQL e preservação de mídia/chaves; ensaiar restauração.

**Aceite:** stack de homologação funciona após reinício; banco não está público; mídia assinada é acessível ao provedor pelo tempo necessário; restauração validada.

### F24 — Piloto controlado [P0]

**Objetivo:** Validar o fluxo real com escopo restrito.

**Entrada:** F19–F23; F22 na primeira rede liberada. **Referência anterior:** B23. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F24.01** — Ativar uma imobiliária de teste autorizada com Essencial e conta de teste da primeira rede liberada.
- [ ] **F24.02** — Executar XML → vídeo real → aprovação → horário → publicação → link e histórico.
- [ ] **F24.03** — Medir custo, duração de geração, memória, filas e falhas; ajustar limites.
- [ ] **F24.04** — Liberar cada rede somente depois de teste real e permissões confirmadas.
- [ ] **F24.05** — Registrar feedback e corrigir bloqueadores antes de ampliar.

**Aceite:** demonstração completa no EasyPanel, evidência de publicação, suporte e recuperação disponíveis. Rede ainda não liberada é explicitamente indisponível no piloto.

### F25 — Pro, Max e acessórios [P1]

**Objetivo:** Entregar os níveis de IA e acessórios contratados.

**Entrada:** F16, F24; provedores de F03. **Referência anterior:** B24. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F25.01** — Integrar geradores reais Pro/Max, jobs assíncronos e retomada por identificador externo.
- [ ] **F25.02** — Implementar Motion Graphics e locução, duração contratada e restrições por plano.
- [ ] **F25.03** — Aplicar limites de custo, timeout e controle de chamadas pagas.
- [ ] **F25.04** — Homologar qualidade visual e refação por categoria em cada nível.

**Aceite:** matriz Essencial/Pro/Max e acessórios permitidos testada; falha/retry não cria pedido pago desnecessário; artefatos identificam provedor e versão utilizados.

### F26 — Gestão comercial e fluxos [P1]

**Objetivo:** Completar regras comerciais e seleção de fluxos versionados.

**Entrada:** F09, F12, F15, F25. **Referência anterior:** B25. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F26.01** — Completar troca de plano imediata/programada, sobras e fechamento mensal idempotente.
- [ ] **F26.02** — Admin seleciona versões ativas do fluxo; versões anteriores permanecem imutáveis.
- [ ] **F26.03** — Validar todos os fluxos de migrations/seeds no registro Python; congelar versão por post.
- [ ] **F26.04** — Testar pelo menos uma variação de fluxo por configuração, sem condicional por cliente.
- [ ] **F26.05** — Manter editor visual/JSON de fluxos fora da v1.

**Aceite:** troca do fluxo não muda post em andamento; painel e fábrica mostram o mesmo saldo nas viradas comerciais.

### F27 — Produtividade e monitores [P1]

**Objetivo:** Completar ações em lote e o destino exclusivo de monitores.

**Entrada:** F19–F20, F25–F26. **Referência anterior:** B26. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F27.01** — Aprovação em lote com resultado por item e seleção de fotos completa.
- [ ] **F27.02** — Geração só para monitores, playlist segregada por token revogável e tela da lista de vídeos.
- [ ] **F27.03** — Garantir que monitores nunca entrem na aprovação, grade ou redes.
- [ ] **F27.04** — Concluir post Vendido com modelo visual e validação em homologação.
- [ ] **F27.05** — Página completa dos monitores permanece fora da v1, conforme documento 00.

**Aceite:** lote relata conflitos individualmente; vídeo exclusivo de monitor não é publicado em rede e obedece à regra de franquia.

### F28 — Dashboard, avisos e configurações [P1]

**Objetivo:** Entregar indicadores e configurações da experiência cliente.

**Entrada:** F17, F20, F22, F26–F27. **Referência anterior:** B27. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F28.01** — Números por período, gráficos por rede, carteira por estado, últimos publicados e tempo economizado.
- [ ] **F28.02** — Concluir sino, leitura de avisos, faixas de conexão e extrato de franquia.
- [ ] **F28.03** — Autosave de configurações, instruções de legenda, dados da imobiliária e plano somente leitura.
- [ ] **F28.04** — Validar mobile, acessibilidade e carregamento de mídia/gráficos sob demanda.

**Aceite:** relatórios conciliam com dados do banco e respeitam tenant/período/fuso; filtros sobrevivem ao recarregamento.

### F29 — Administração completa [P1]

**Objetivo:** Permitir operar o serviço pelo Admin.

**Entrada:** F12, F19, F21–F23, F26–F28. **Referência anterior:** B28. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F29.01** — Completar imobiliárias, usuários/convites, chamados, consumo/CSV, horários recomendados e filtros por fluxo.
- [ ] **F29.02** — Falhas por etapa/rede, retry seguro, leitura XML sob demanda e saúde por heartbeat/fila.
- [ ] **F29.03** — Entrar como imobiliária somente leitura, com trilha de auditoria e bloqueio de escrita no servidor.
- [ ] **F29.04** — Expor ações necessárias para operação sem SQL manual.

**Aceite:** equipe resolve chamado, identifica worker parado e trata falha isolada pelo painel, sem violar a máquina de estados.

### F30 — Migração do pipeline anterior [P0/P1]

**Objetivo:** Transferir dados e histórico com reconciliação.

**Entrada:** F01, F07–F09, F24. **Referência anterior:** B29. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F30.01** — Implementar importador versionado com modo de simulação, mapeamento de IDs, estados, tenants, mídia e histórico.
- [ ] **F30.02** — Se houver SQLite, extrair de cópia controlada; não manter escrita paralela sem estratégia explícita.
- [ ] **F30.03** — Ensaiar com dados anonimizados e conciliar contagens, relações, links de mídia e franquia.
- [ ] **F30.04** — Definir fonte responsável por publicações já iniciadas, janela de pausa e prevenção de duplicação durante o corte.
- [ ] **F30.05** — Registrar plano de retorno: após novas escritas/publicações, voltar ao pipeline anterior exige reconciliação, não apenas restaurar banco antigo.

**Aceite:** relatório de reconciliação sem divergência crítica; execução repetida não duplica entidades; plano de corte e retorno ensaiado.

### F31 — Homologação funcional e resiliência [P0/P1]

**Objetivo:** Comprovar a v1 completa sob condições reais e falhas.

**Entrada:** F25–F30; três redes liberadas. **Referência anterior:** B30. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F31.01** — Executar matriz completa de RF/RN/RNF com duas imobiliárias e roles de produção.
- [ ] **F31.02** — Testar réplicas, última vaga de franquia, clique duplo, cancelamento concorrente, retomada, perda de sinal e falha parcial.
- [ ] **F31.03** — Confirmar que parar API mantém trabalho possível da fábrica e parar fábrica preserva painel; aprovado_auto aguarda o Java sem perder intenção.
- [ ] **F31.04** — Validar isolamento de APIs, mídia, CSV, avisos, monitores e modo Entrar como.
- [ ] **F31.05** — Medir abertura < 2 s em perfil 4G documentado e carga de referência ~10 imobiliárias × 2.000 imóveis; registrar método e resultado.
- [ ] **F31.06** — Concluir publicação real nas três redes e teste de restauração de banco/mídia.

**Aceite:** bloqueadores resolvidos; cobertura da v1 registrada; limitações aceitas explicitamente pelo responsável, sem marcar requisito não entregue como concluído.

### F32 — Corte e operação assistida [P0/P1]

**Objetivo:** Liberar produção e estabilizar a operação.

**Entrada:** F31. **Referência anterior:** B31. **Estado:** a fazer.

**Blocos de execução:**

- [ ] **F32.01** — Fazer backup, pausar produtores anteriores, executar migração final e conferir saldos/agendamentos.
- [ ] **F32.02** — Publicar no EasyPanel na ordem prevista, rodar smoke e liberar clientes gradualmente.
- [ ] **F32.03** — Entregar runbook de deploy, rollback, restauração, rotação de segredos, onboarding e incidentes.
- [ ] **F32.04** — Acompanhar filas, custos, uso de recursos, conexões e publicações diariamente durante a estabilização.
- [ ] **F32.05** — Encerrar estabilização quando não houver incidente crítico aberto e as rotinas de operação tiverem evidência.

**Aceite:** dados conciliados, fluxo real validado, responsáveis de suporte definidos e operação aprovada pelo responsável do projeto.

## 5. Rastreabilidade dos requisitos

Os códigos completos e critérios de comportamento permanecem no [catálogo de requisitos](00-visao-geral.md#6-catálogo-de-requisitos); esta tabela indica onde entregar e verificar cada grupo.

| Requisitos | Referências B da seção 4 |
|---|---|
| RF 01–03: identidade e isolamento | B07, B09, B10 |
| RF 10–18: aprovação, legenda, rejeição, saída | B16–B19, B26 |
| RF 20–27: carteira, busca, gerar e fotos | B12–B16, B19, B26 |
| RF 30–31: monitores | B26 |
| RF 40–43, 45–46: Dashboard | B27 |
| RF 50–55: configurações, redes e plano | B11, B17, B20, B27 |
| RF 60–67: Admin | B11, B18, B22, B25, B28 |
| RF 70: avisos | B16, B19, B27 |
| RF 80–83: franquia e mudança de plano | B08, B16, B25 |
| RN 01–02: rejeição/chamado | B18 |
| RN 03–04: aprovação e grade | B17, B19 |
| RN 05–07: atraso, irreversibilidade e independência | B21 |
| RN 08–12: legenda, vendido, monitores, auditoria e XML | B06, B12, B16, B19, B26 |
| RN 13–15: franquia e acessórios | B08, B24, B25 |
| RNF 01–03: desempenho, responsividade e escala | B10, B13, B27, B30 |
| RNF 04–05: desacoplamento e segurança | B07, B14, B20–B22, B30 |
| RNF 06–09: identidade, localização, evolução, domínio único | B04, B10, B22, B25, B27 |

## 6. Critério comum de conclusão de um bloco

- Comportamento demonstrado e critérios específicos atendidos.
- Testes apropriados: domínio; integração PostgreSQL/Flyway e roles; isolamento de endpoints; contrato entre linguagens; interface nos dois tamanhos.
- CI pertinente aprovado e PR revisado; nenhuma credencial ou dado real inserido no repositório.
- Schema alterado somente por migration nova; contrato regenerado quando necessário.
- Mudanças de estado condicionais e eventos na mesma transação; chamadas externas fora da transação.
- Documentação afetada atualizada; limitações e dependências registradas.
- Evidência anexada à tarefa e estado atualizado somente depois da verificação.

## 7. Riscos e dependências de maior impacto

| Risco | Ação e ponto de controle |
|---|---|
| Aprovação de apps/permissões nas redes | Começar B02 imediatamente; validar em B20/B21; não liberar rede sem teste real |
| Código/dados/compositor anteriores indisponíveis | Inventariar em B00; reestimar B15/B29 se reaproveitamento não for possível |
| Custo e duração de IA | Provedores falsos em dev; orçamento em B02; retomada e limites em B14/B24 |
| Duplicação de publicação após timeout | Reconciliação externa e idempotência por adaptador em B21 |
| Vazamento entre imobiliárias | RLS e roles reais em B07; testes obrigatórios por endpoint e B30 |
| Consumo de franquia divergente | Função única e concorrência em B08; conciliação B25/B29 |
| Perda de eventos por ordem de commit | Resolver B01 e provar em B19, incluindo transação longa |
| Memória da VPS compartilhada | Medir vídeo real no piloto; limites por processo em B22/B23 |
| Migração e corte publicarem conteúdo duas vezes | Pausar pipeline anterior e reconciliar em B29/B31 |
| Escopo crescer antes da primeira entrega | Priorizar Essencial integrado; preservar evolução P1 em vez de bloquear piloto |

## 8. Evoluções posteriores [P2]

Permanecem fora da v1: vídeos próprios externos, página completa de monitores, editor visual do rodapé, métricas de desempenho das redes, busca em linguagem natural, avisos WhatsApp/e-mail, app nativo e calendário com arrastar. Editor de fluxos e eventual adoção de n8n exigem planejamento próprio.

**Próxima execução recomendada:** executar F01 (inventário), F02 (decisões técnicas) e iniciar F03 (acessos); depois avançar em F04–F06 (estrutura, contrato e ambiente). Não começar por todas as telas nem por provedores pagos antes de fechar o contrato de dados.
