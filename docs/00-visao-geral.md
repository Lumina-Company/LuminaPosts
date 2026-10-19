# 00 — Visão geral

## 1. Produto

O **LuminaImob** transforma a carteira de imóveis de uma imobiliária em vídeos para redes sociais. A partir do XML de imóveis da imobiliária (o mesmo enviado aos portais, formato VivaReal), o sistema escolhe as fotos, gera um vídeo vertical com os dados do imóvel, escreve a legenda com IA e publica no Instagram (Reels), no TikTok e no YouTube Shorts.

O **Luminaria** é o painel web do LuminaImob. Nele a imobiliária:

1. vê todos os imóveis da carteira e o estado do vídeo de cada um;
2. aprova, edita a legenda ou rejeita os vídeos antes de irem ao ar;
3. define quando os vídeos aprovados são publicados (grade de horários);
4. acompanha os resultados (gerados, publicados, rejeitados).

A equipe Lumina usa a área **Admin** para acompanhar todas as imobiliárias, tratar chamados e configurar planos.

Princípio de produto: **controle total com o mínimo de trabalho**. A imobiliária só aprova; se não gostar, o vídeo é refeito.

## 2. Papéis

| Papel | Quem | Pode |
|---|---|---|
| **Cliente** | Usuário da imobiliária (dono, corretor, marketing) | Tudo da própria imobiliária: aprovar, rejeitar, editar legenda, gerar, agendar, configurar |
| **Admin** | Equipe Lumina | Área Admin global: imobiliárias, planos, chamados, usuários, consumo, saúde |
| **Sistema** | Fábrica (workers Python) | Ingestão, geração e publicação |

Todo usuário tem login próprio, criado pela equipe Lumina (não há cadastro aberto). Uma imobiliária nunca vê dados de outra.

## 3. Planos

| Nível | Vídeo | Faixas |
|---|---|---|
| **Essencial** | Fotos com zoom e moldura com os dados do imóvel, sem IA | 30 · 60 · Ilimitado |
| **Pro** | Vídeo gerado por IA a partir das fotos | 30 · 50 · Personalizado |
| **Max** | Vídeo por IA de qualidade superior | 30 · 60 · Personalizado |

- Vídeos de 15 s; no Personalizado, duração (5 a 60 s) e quantidade são definidas na contratação.
- **Acessórios** (valem para todos os vídeos da imobiliária): **Motion Graphics** (dados animados no lugar da moldura) e **Locução por IA**. Não existem no Essencial Ilimitado.
- **Franquia mensal**: atingido o limite, a geração para até o mês seguinte; vídeos não usados passam só para o mês seguinte e são gastos primeiro.
- Um plano vale para um perfil (1 Instagram + 1 TikTok + 1 YouTube). Imobiliária com dois perfis = duas imobiliárias no sistema.

## 4. Escopo da v1

**Dentro:** login; Aprovação (pendentes e agendados); Imóveis (catálogo, busca, detalhe, gerar); Dashboard; Configurações (publicação, grade, redes, dados, plano); franquia; monitores da loja; post "Vendido"; avisos no painel; Admin (imobiliárias, chamados, falhas e saúde, consumo, usuários, horários recomendados); fluxos de automação por imobiliária.

**Fora (arquitetura já preparada):** publicação de vídeos próprios gerados por ferramenta externa de motion graphics; página completa dos monitores; editor visual do rodapé; métricas de desempenho dos posts; busca em linguagem natural; avisos por WhatsApp/e-mail; app nativo; calendário com arrastar para reagendar.

## 5. Mapa de telas

```
Login (único para Cliente e Admin)
│
├── Cliente (uma imobiliária)
│   ├── Topo: contador da franquia · sino · usuário
│   ├── Aprovação  ← tela inicial
│   │   ├── Pendentes
│   │   └── Agendados
│   ├── Imóveis    ← catálogo ("Vitrine de imóveis")
│   ├── Monitores da loja (se habilitado)
│   ├── Dashboard
│   └── Configurações (inclui o plano, somente leitura)
│
└── Admin (global)
    ├── Chamados   ← tela inicial
    ├── Imobiliárias
    ├── Falhas e saúde
    ├── Consumo
    └── Usuários
```

## 6. Catálogo de requisitos

Os documentos técnicos referenciam os requisitos pelos códigos abaixo.

### 6.1 Funcionais — Cliente

| Código | Requisito |
|---|---|
| RF-LUM-01 | Login por e-mail e senha; sessão de 30 dias; mesma tela para Cliente e Admin |
| RF-LUM-02 | Várias imobiliárias; cada uma vê só os próprios dados |
| RF-LUM-03 | Redefinição de senha |
| RF-LUM-10 | Lista de pendentes (vídeo, legenda, dados, "esperando desde", versão, redes), atualizada sozinha |
| RF-LUM-11 | Aprovar em um clique, conforme o modo de publicação |
| RF-LUM-12 | Agendados: trocar horário e cancelar |
| RF-LUM-13 | Aprovar vários |
| RF-LUM-14 | Editar legenda e hashtags, com contador e limite por rede |
| RF-LUM-15 | Rejeitar com categorias, descrição e observação |
| RF-LUM-16 | Não postar este imóvel |
| RF-LUM-17 | Mensagem na segunda rejeição |
| RF-LUM-18 | Saiu da carteira: confirmar venda/locação |
| RF-LUM-20 | Catálogo de imóveis em cards |
| RF-LUM-21 | Filtros e ordenação na URL |
| RF-LUM-22 | Busca tolerante a acento, erro de digitação e trecho de palavra |
| RF-LUM-23 | Detalhe do imóvel com versões, publicações e linha do tempo |
| RF-LUM-24 | Gerar agora |
| RF-LUM-25 | Gerar de novo (imóvel publicado) |
| RF-LUM-26 | Escolher as fotos do próximo vídeo |
| RF-LUM-27 | Desfazer descarte |
| RF-LUM-30 | Gerar só para os monitores |
| RF-LUM-31 | Lista de vídeos dos monitores |
| RF-LUM-40 | Filtro de período no Dashboard |
| RF-LUM-41 | Números do período |
| RF-LUM-42 | Publicados ao longo do tempo, por rede |
| RF-LUM-43 | Carteira por estado (clicável) |
| RF-LUM-45 | Últimos publicados |
| RF-LUM-46 | Tempo economizado |
| RF-LUM-50 | Modo de publicação (ao aprovar / seguir a grade, com ou sem encaixe automático) |
| RF-LUM-51 | Grade de horários com horários recomendados |
| RF-LUM-52 | Aprovação liga/desliga |
| RF-LUM-53 | Redes e estado da conexão |
| RF-LUM-54 | Dados da imobiliária |
| RF-LUM-55 | Plano atual (somente leitura) |
| RF-LUM-70 | Avisos no painel (sino) |
| RF-LUM-80 | Contador da franquia |
| RF-LUM-81 | Franquia esgotada |
| RF-LUM-82 | Sobra do mês |
| RF-LUM-83 | Troca de plano |

### 6.2 Funcionais — Admin

| Código | Requisito |
|---|---|
| RF-LUM-60 | Lista de imobiliárias com plano, configurações e estado |
| RF-LUM-61 | Configurações definidas pelo Admin (plano, monitores, XML, limites, horários recomendados) |
| RF-LUM-62 | Chamados |
| RF-LUM-63 | Falhas e saúde |
| RF-LUM-64 | Consumo |
| RF-LUM-65 | Usuários |
| RF-LUM-66 | Entrar como a imobiliária |
| RF-LUM-67 | Instruções da legenda |

### 6.3 Regras de negócio

| Código | Regra |
|---|---|
| RN-LUM-01 | Primeira rejeição volta para a fila com prioridade alta; o que é refeito depende da categoria |
| RN-LUM-02 | Segunda rejeição vira chamado; só o Admin tira o imóvel de análise |
| RN-LUM-03 | Com a aprovação ligada, nada é publicado sem aprovação; lembrete após 48 h |
| RN-LUM-04 | Um vídeo por horário da grade |
| RN-LUM-05 | Horário perdido: publica assim que possível e registra o atraso |
| RN-LUM-06 | Publicação não volta atrás |
| RN-LUM-07 | Redes independentes |
| RN-LUM-08 | Edição manual da legenda vence a IA |
| RN-LUM-09 | Post "Vendido" só com confirmação |
| RN-LUM-10 | Vídeo dos monitores nunca entra na aprovação, na grade nem nas redes |
| RN-LUM-11 | Toda ação é registrada (quem, quando, o quê) |
| RN-LUM-12 | Dados alterados no XML depois da geração não refazem o vídeo |
| RN-LUM-13 | Gastam franquia: vídeo novo (automático, Gerar agora, Gerar de novo, monitores). Não gastam: refeito por rejeição, refeito em chamado, post Vendido |
| RN-LUM-14 | Franquia conferida antes de gerar, a partir do registro de vídeos |
| RN-LUM-15 | Acessórios valem para todos os vídeos da imobiliária |

### 6.4 Não funcionais

| Código | Requisito |
|---|---|
| RNF-LUM-01 | Leve: abre em menos de 2 s no 4G; mídia só carrega sob demanda |
| RNF-LUM-02 | Celular (a partir de 360 px) e computador, com um layout adaptável |
| RNF-LUM-03 | Escala: ~10 imobiliárias com até ~2.000 imóveis cada |
| RNF-LUM-04 | Painel e fábrica desacoplados |
| RNF-LUM-05 | Segurança simples e bem feita; isolamento total entre imobiliárias |
| RNF-LUM-06 | Identidade visual Lumina (UI Kit v2 Light) |
| RNF-LUM-07 | Português, dd/mm/aaaa, reais, fuso America/Sao_Paulo |
| RNF-LUM-08 | Funcionalidades novas sem reescrever o painel |
| RNF-LUM-09 | Um endereço único para todas as imobiliárias |

## 7. Glossário

| Termo | Significado |
|---|---|
| **Imobiliária** | Cliente do SaaS e unidade de isolamento (tenant) |
| **Carteira** | Imóveis presentes no XML da imobiliária |
| **Post** | Uma versão de vídeo + legenda + hashtags, destinada às redes ou aos monitores |
| **Versão** | Número sequencial dos posts de um imóvel (v1, v2…) |
| **Refeito** | Post gerado após uma rejeição |
| **Chamado** | Imóvel rejeitado duas vezes, tratado pela equipe Lumina |
| **Grade** | Horários semanais de publicação da imobiliária |
| **Franquia** | Quantidade de vídeos novos por mês do plano |
| **Monitores da loja** | Telas dentro da loja da imobiliária que exibem vídeos que não vão para as redes |
| **Vitrine de imóveis** | Nome comercial do catálogo (aba Imóveis) |
| **Fábrica** | Workers Python que ingerem, geram e publicam |
| **Fluxo de automação** | Definição versionada das etapas da fábrica para uma imobiliária |
