# 03 — Ciclo de vida (máquinas de estado)

Há **duas** máquinas, ligadas mas separadas:

- **Imóvel** — o que o cliente vê no selo do card, na pizza do Dashboard e no Admin.
- **Post** — cada versão de vídeo (v1, v2…), inclusive o post "Vendido" e, no futuro, o post próprio sem imóvel (ADR-05).

Por que separar: o imóvel `vendido` continua `vendido` enquanto o **post** "Vendido" passa por fila → aprovação → grade. E um vídeo "só para os monitores" tem post próprio sem mexer no estado do imóvel (RN-LUM-10).

## 1. Imóvel

```mermaid
stateDiagram-v2
    direction LR
    [*] --> nao_gerado: ingestor detecta no XML
    [*] --> inelegivel: menos de 4 fotos etc.
    inelegivel --> nao_gerado: XML corrigido

    nao_gerado --> na_fila: fila automática / Gerar agora
    na_fila --> gerando: gerador reserva
    gerando --> aguardando_aprovacao: vídeo pronto (aprovação ligada)
    gerando --> agendado: vídeo pronto (aprovação desligada)
    gerando --> falhou: erro após tentativas

    aguardando_aprovacao --> agendado: Aprovar
    aguardando_aprovacao --> na_fila: Rejeitar (1ª vez)
    aguardando_aprovacao --> em_analise: Rejeitar (2ª vez)
    aguardando_aprovacao --> descartado: Não postar

    agendado --> aguardando_aprovacao: Cancelar agendamento
    agendado --> publicado: publicador (≥1 rede ok)

    em_analise --> aguardando_aprovacao: Admin devolve
    em_analise --> na_fila: Admin refaz com ajuste
    em_analise --> descartado: Admin descarta

    falhou --> na_fila: Admin "tentar de novo"

    publicado --> na_fila: Gerar de novo (nova versão)
    publicado --> saiu_da_carteira: sumiu do XML
    saiu_da_carteira --> vendido: cliente confirma venda
    saiu_da_carteira --> indisponivel: cliente diz "só retirado"

    nao_gerado --> indisponivel: sumiu do XML
    na_fila --> indisponivel: sumiu do XML
    aguardando_aprovacao --> indisponivel: sumiu do XML
    agendado --> indisponivel: sumiu do XML

    descartado --> nao_gerado: Desfazer descarte
    indisponivel --> nao_gerado: voltou ao XML
```

Marcador **Refeito (vN)**: não é estado; é derivado de `post.numero_versao > 1` do post atual.

## 2. Post (versão de vídeo)

```mermaid
stateDiagram-v2
    direction LR
    [*] --> na_fila: criado
    na_fila --> gerando: gerador reserva (franquia ok)
    gerando --> na_fila: reaper (worker morreu)
    gerando --> falhou: 3 tentativas

    gerando --> aguardando_aprovacao: pronto · destino redes · aprovação ligada
    gerando --> aprovado_auto: pronto · aprovação desligada
    gerando --> nos_monitores: pronto · destino monitores

    aprovado_auto --> agendado: API aloca na grade
    aguardando_aprovacao --> agendado: Aprovar
    aguardando_aprovacao --> rejeitado: Rejeitar
    aguardando_aprovacao --> descartado: Não postar
    agendado --> aguardando_aprovacao: Cancelar agendamento

    agendado --> publicando: horário venceu
    publicando --> publicado: ≥1 rede ok
    publicando --> falhou: todas as redes falharam

    rejeitado --> aguardando_aprovacao: Admin devolve (chamado)
    falhou --> na_fila: Admin tenta de novo

    na_fila --> cancelado: imóvel indisponível
    aguardando_aprovacao --> cancelado: imóvel indisponível
    agendado --> cancelado: imóvel indisponível

    publicado --> [*]
    nos_monitores --> [*]
    descartado --> [*]
    cancelado --> [*]
```

`publicado` é final (RN-LUM-06): aprovar ou editar de novo não republica.

## 3. Quem pode fazer cada transição

Ator gravado na transação: `cliente`, `admin` (vindo do Java) ou `sistema` (vindo do Python ou de um job do Java).

### Imóvel

| De → Para | Ator | Requisito |
|---|---|---|
| `nao_gerado` → `na_fila` | sistema, cliente | fila automática, RF-LUM-24 |
| `na_fila` → `gerando` | sistema | — |
| `gerando` → `aguardando_aprovacao` / `agendado` / `falhou` | sistema | RF-LUM-52 |
| `aguardando_aprovacao` → `agendado` | cliente (ou sistema, se aprovação desligada) | RF-LUM-11 |
| `aguardando_aprovacao` → `na_fila` | cliente | RN-LUM-01 |
| `aguardando_aprovacao` → `em_analise` | cliente | RN-LUM-02 |
| `aguardando_aprovacao` → `descartado` | cliente | RF-LUM-16 |
| `agendado` → `aguardando_aprovacao` | cliente | RF-LUM-12 |
| `agendado` → `publicado` | sistema | — |
| `em_analise` → qualquer | **admin** | RN-LUM-02 |
| `falhou` → `na_fila` | admin | RF-LUM-63 |
| `publicado` → `na_fila` | cliente | RF-LUM-25 |
| `*` → `saiu_da_carteira` / `indisponivel` | sistema | RN-LUM-09 |
| `saiu_da_carteira` → `vendido` / `indisponivel` | cliente | RF-LUM-18 |
| `descartado` → `nao_gerado` | cliente | RF-LUM-27 |

## 4. O banco recusa transição inválida

Duas linguagens escrevem estado. A regra de **decidir** fica no código (Java ou Python), mas a regra de **validade** fica no banco, para nenhum dos dois conseguir pular uma etapa:

```sql
CREATE TABLE luminaria.transicao_estado (
  entidade text NOT NULL CHECK (entidade IN ('imovel','post')),
  de       text NOT NULL,
  para     text NOT NULL,
  atores   text[] NOT NULL,          -- {cliente,admin,sistema}
  PRIMARY KEY (entidade, de, para)
);

CREATE FUNCTION luminaria.trg_validar_transicao() RETURNS trigger AS $$
DECLARE
  v_ator text := coalesce(current_setting('luminaria.ator', true), 'sistema');
BEGIN
  IF NEW.estado IS DISTINCT FROM OLD.estado THEN
    PERFORM 1 FROM luminaria.transicao_estado
     WHERE entidade = TG_ARGV[0] AND de = OLD.estado AND para = NEW.estado
       AND v_ator = ANY(atores);
    IF NOT FOUND THEN
      RAISE EXCEPTION 'Transição inválida em %: % -> % (ator %)',
        TG_ARGV[0], OLD.estado, NEW.estado, v_ator
        USING ERRCODE = 'P0001';
    END IF;
    NEW.estado_desde := now();
  END IF;
  RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER imovel_transicao BEFORE UPDATE OF estado ON luminaria.imovel
  FOR EACH ROW EXECUTE FUNCTION luminaria.trg_validar_transicao('imovel');
CREATE TRIGGER post_transicao BEFORE UPDATE OF estado ON luminaria.post
  FOR EACH ROW EXECUTE FUNCTION luminaria.trg_validar_transicao('post');
```

- O Java faz `SELECT set_config('luminaria.ator', 'cliente', true)` no início de cada transação (junto do tenant; ver [05](05-seguranca.md)). O Python não precisa: o padrão é `sistema`.
- A tabela `transicao_estado` é populada por uma migration (`R__transicoes.sql`, repeatable) gerada de `contrato/estados.yaml`.
- No Java, `P0001` vira **409 Conflict**.

## 5. Regras de rejeição (RN-LUM-01 / 02)

```mermaid
flowchart TD
    r["Cliente clica Rejeitar<br/>(categorias + descrição + observação)"] --> v{"imovel.rejeicoes_no_ciclo"}
    v -->|"0 → 1ª rejeição"| f["post → rejeitado<br/>novo post vN+1 na_fila<br/>prioridade ALTA<br/>gasta_franquia = false"]
    v -->|"1 → 2ª rejeição"| c["post → rejeitado<br/>imóvel → em_analise<br/>INSERT chamado<br/>mensagem RF-LUM-17"]
    f --> e["escopo do refazer =<br/>maior escopo entre as categorias"]
    e --> e1["render → remontar"]
    e --> e2["dados → remontar lendo XML atual"]
    e --> e3["foto → novas fotos + gerar"]
    e --> e4["ia → novos clipes"]
    e --> e5["outro → refazer tudo"]
```

- `imovel.rejeicoes_no_ciclo` volta a 0 quando um post do imóvel é aprovado ou quando o Admin resolve o chamado.
- Escopo, do menor para o maior: `remontar` < `novas_fotos` = `novos_clipes` < `tudo`. Com `novas_fotos` + `novos_clipes`, vale `tudo`. O escopo vai para `post.escopo_refazer` e o gerador obedece.
- "Erro da IA" só aparece em Pro/Max; "Locução com erro" só com o acessório Locução. A API valida contra o **nível usado no post**, não o plano de hoje.
