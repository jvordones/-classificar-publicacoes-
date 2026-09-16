---
name: classificar-publicacoes
description: Classifica publicações judiciais das 5 verticais da Resolvvi (Auxílio Acidente, INSS, Voos, BRS, NNI) na estrutura de 3 camadas — tipo do ato, conteúdo/assunto, qualificação — usando o histórico de classificações do próprio pedido como contexto, e resolve o status do Order correspondente. Use sempre que precisar classificar uma publicação judicial (individualmente ou em lote, inclusive via /schedule) para qualquer uma dessas 5 verticais.
---

# Classificação de publicações judiciais (3 camadas) — Resolvvi

Esta skill encapsula toda a lógica de classificação de publicações desenvolvida e validada em 17.817 casos reais de Auxílio Acidente. Aplique-a sempre que for classificar uma publicação — não reclassifique do zero com raciocínio próprio, siga os passos abaixo.

**Princípio central desta versão**: uma publicação não é um evento isolado. Todo processo tem um fluxo, e esse fluxo caminha predominantemente para frente. O histórico de classificações do pedido é **contexto obrigatório** — ele desempata leituras ambíguas, preenche elipses do texto e barra transições de status incoerentes. O que ele **não** faz é substituir o texto (ver "Regra do texto soberano").

⚠️ **Regra crítica de escopo**: o match classificação → status **só pode ser aplicado a pedidos de origem externa** (qualquer origem que não seja Resolvvi/marketplace). Antes de gravar um status resolvido, confirme a origem do pedido; se for Resolvvi/marketplace, classifique `{tipo, conteudo, qualificacao}` normalmente mas **não** dispare o match de status.

⚠️ **Estado atual das sugestões automáticas**: hoje só a Camada 1 (tipo do ato) tem sugestão automática. Conteúdo e qualificação são 100% manuais (analista) até o modelo em camadas (1 modelo de tipo + 5 modelos de conteúdo, um por tipo) ser reconstruído e entrar em produção. Ao rodar esta skill de forma autônoma/scheduled, trate isso como o fluxo real: você está fazendo o papel da sugestão automática ainda não reativada — sinalize sempre os casos incertos para revisão humana em vez de forçar uma classificação.

## Visão geral do pipeline

```
publicação (texto) + order_id
      │
      ▼
Passo 0 — carregar histórico do pedido e posicionar a fase processual
      │
      ▼
Camada 0 — é publicação de fato, ou é andamento puro?  (passo 1)
      │ sim
      ▼
Camada 1 — tipo do ato                                  (passo 2)
      │
      ▼
Camada 2 — conteúdo/assunto                             (passo 3)   ← histórico desempata
      │
      ▼
Camada 3 — qualificação                                 (passo 4)   ← histórico desempata
      │
      ▼
Passo 5 — checagem de coerência com o histórico
      │
      ▼
Passo 6 — match de status (por vertical) + regra de não-regressão
      │
      ▼
{ tipo, conteudo, qualificacao, parte, status, fase, coerencia, usou_historico }
```

---

## Passo 0 — Carregar o histórico do pedido

Antes de ler o texto da publicação, monte o **contexto processual** do pedido:

| Item | Para quê |
|---|---|
| `problem_kind` (vertical) | escolher a tabela de match do passo 6 |
| Origem do pedido (externa vs. Resolvvi) | aplicar a regra crítica de escopo |
| **Status atual do Order** | âncora de fase e base da regra de não-regressão |
| **Classificações anteriores do mesmo processo**, ordenadas por data de publicação: `{data, tipo, conteudo, qualificacao, parte, status_resultante, revisado_por_humano}` | prior de fase e desempate |
| Instância atual, se disponível | distinguir 1ª instância de tribunal |

**Janela**: as **10 classificações mais recentes**, mais **todos os marcos** do processo mesmo que fora da janela — sentença, acórdão, trânsito em julgado, início de cumprimento de sentença, extinção, suspensão. Marco nunca sai do contexto.

**Peso por procedência**: classificação **revisada por humano** vale mais que sugestão automática não revisada. Se o histórico recente for todo automático e não revisado, trate o prior como fraco — o risco de propagar um erro em cadeia é real.

**Sem histórico** (primeira publicação do processo): prior uniforme, comporte-se como classificador de texto isolado. Grave `usou_historico: false`.

⚠️ **Ausência de histórico não é evidência de fase inicial.** Pedido importado de origem externa entra na base com o processo já em curso. Se não há classificações anteriores mas o **status atual do Order** já indica fase avançada, use o status como âncora. Se não houver nem histórico nem status informativo, não presuma fase — classifique só pelo texto.

**Deduplicação**: o mesmo ato costuma ser publicado em mais de um diário. Se já existir no histórico uma classificação com o mesmo `{tipo, conteudo, qualificacao}` dentro de **10 dias** e o teor for equivalente, marque `duplicata: true`, grave a classificação normalmente e **não redispare o match de status**.

**Ordem de processamento em lote**: sempre por **data de publicação ascendente**, reconstruindo o histórico incrementalmente a cada item. Nunca classifique um lote em paralelo assumindo o mesmo histórico inicial para todos.

---

## Como o histórico pesa — regras de precedência

### Regra do texto soberano

O histórico é **prior**, não evidência. Ele nunca cria classificação que o texto não sustenta.

| Situação do texto | Quem decide |
|---|---|
| **Explícito e inequívoco** ("Julgo procedente o pedido") | **texto**, sozinho. Histórico é ignorado na classificação (mas ainda checa coerência no passo 5) |
| **Ambíguo entre 2+ leituras válidas** | **histórico desempata** — escolha a leitura coerente com a fase atual |
| **Elíptico / incompleto** ("Intime-se a parte autora para manifestar-se, prazo 15 dias") | **histórico completa a lacuna** até o nível de conteúdo. Qualificação sem suporte textual = `sem_qualificacao` |
| **Contradiz marco irreversível do histórico** | classifique pelo **texto**, mas não dispare status e sinalize (passo 5) |

Nunca infira qualificação só do histórico. Conteúdo pode ser inferido por contexto; resultado de julgamento (favorável/desfavorável/etc.) exige suporte no texto.

### Catálogo de usos do histórico

1. **Modalidade recursal** — resolve o maior ponto de ambiguidade do esquema:
   - último ato decisório no histórico = `decisao_interlocutoria` → **Agravo de Instrumento**
   - último = `sentenca` → **Apelação**
   - último = `acordao` colegiado → **Recurso Especial/Extraordinário**
   - último = decisão monocrática de relator → **Agravo Interno**
   - Se o texto nomear a modalidade, o **texto vence**.
   - Sem histórico e sem nome no texto: **não presuma Apelação** — grave `sem_qualificacao` e sinalize. Chute nunca pode atravessar para o passo 6.

2. **Parte recorrente (autor × réu)**, quando o texto não diz:
   - sentença desfavorável ao autor no histórico → recorrente provável = **autor**
   - sentença favorável ao autor → recorrente provável = **réu**
   - sentença **parcialmente** favorável → ambos podem recorrer, **não infira**; use `parte: indefinida` e sinalize
   - Marque sempre `parte_inferida: true` quando vier do histórico.

3. **"Manifestar-se" genérico** — sobre o quê? O último ato pendente no histórico responde: laudo pericial apresentado há 3 dias → `Perícia judicial :: manifestação sobre laudo`; cálculo apresentado pelo réu → `Cálculo/liquidação :: intimado a se manifestar`.

4. **Distribuição** — "certifico que os autos foram distribuídos" depois de uma interposição de recurso no histórico é **distribuição do recurso no tribunal**, não distribuição da ação (Camada 0 e conteúdo mudam).

5. **Perícia** — se o histórico já tem nomeação e agendamento, a próxima publicação de perícia tende a ser laudo/manifestação, não nova nomeação.

6. **Valores** — depois de `Cálculo/liquidação :: homologado`, publicação sobre valores tende a ser execução/pagamento (RPV, MLE, depósito), não liquidação inicial.

7. **`extinct`** — a regra de acumulação do passo 6 só é executável com histórico. É o caso de uso original.

8. **Não-regressão de status** — ver passo 6.

9. **Detecção de duplicata e de publicação fora de ordem** — ver passo 0.

---

## Mapa de fases processuais

Use para posicionar o processo e calcular a expectativa do próximo ato.

| # | Fase | Conteúdos típicos |
|---|---|---|
| 1 | `distribuicao` | Citação do réu, Despacho de pendência, Tutela de urgência, Redistribuição |
| 2 | `instrucao` | Réplica, Especificar provas, Saneamento, Perícia judicial, Audiência marcada, Alegações finais, Acordo (proposta) |
| 3 | `sentenca` | Mérito da causa, Extinção do processo, Acordo homologado |
| 4 | `recurso_2a` | Apelação/Agravo (interposição, distribuição, sessão), acórdão |
| 5 | `recurso_superior` | Recurso Especial/Extraordinário, Agravo Interno |
| 6 | `transito` | Trânsito em julgado |
| 7 | `liquidacao` | Cumprimento de sentença, Cálculo/liquidação, Embargos de execução, Honorários periciais |
| 8 | `pagamento` | Expedição de precatório/RPV, MLE, Depósito em juízo, Implantação do benefício, Custas |
| 9 | `encerrado` | Pago/arquivado/extinto definitivo |

**Trilhas paralelas — não movem a fase principal**: Agravo de Instrumento (ataca interlocutória durante a instrução), Embargos de declaração (em qualquer fase), Cessão de crédito, Prazo decorrido/silêncio, Processo suspenso (congela, não regride).

**Regressões legítimas** — só estas:

| Gatilho | Efeito |
|---|---|
| Acórdão anulando sentença | volta para `instrucao` ou `sentenca` |
| Nova perícia deferida | volta para `instrucao` |
| Execução infrutífera | volta para `liquidacao` |
| Impugnação ao cumprimento de sentença | permanece em `liquidacao` |
| Redistribuição de autos | mantém a fase |

Qualquer outro movimento para trás é incoerência — trate pelo passo 5.

---

## Passo 1 — Camada 0: filtro publicação × andamento

Pergunta: **esse conteúdo dá ciência formal à parte e/ou abre prazo processual?**

- **Sim** → é publicação, segue pro passo 2.
- **Não** → é andamento processual puro (autos recebidos, remetidos à conclusão, juntada de petição interna, distribuição/redistribuição sem intimação associada). **Não classifique** — descarte ou marque como `andamento`, sem gerar `{tipo, conteudo, qualificacao}`.

Casos excluídos do escopo mesmo se parecerem publicação:
- **Acordo pré-judicial/extrajudicial com a empresa** (Voos/BRS/NNI) — negociação rastreada por outro canal, antes da distribuição da ação.
- **Fase administrativa/pré-judicial** (ex.: INSS manifestar-se sobre pedido administrativo, antes da ação judicial).

Se o texto contiver "certifico que os autos foram distribuídos/redistribuídos" **sem** menção a intimação das partes sobre isso, trate como andamento. Se houver intimação associada ("ficam as partes intimadas da distribuição..."), é publicação (tipo `intimacao`, conteúdo `Apelação`/`Agravo de Instrumento`/etc. com qualificação `distribuição`, dependendo do recurso em curso).

**Com histórico**: se o histórico registra interposição de recurso recente, "distribuído" refere-se ao recurso no tribunal — publicação, não andamento (uso 4 do catálogo).

---

## Passo 2 — Camada 1: tipo do ato (5 valores, universal, nunca varia por vertical)

| Valor | Quando usar |
|---|---|
| `citacao` | O texto convoca o réu/executado a integrar a lide pela primeira vez |
| `intimacao` | Dá ciência de qualquer conteúdo, ordena providência, ou é mero expediente/despacho/certidão/ofício. **Use este como padrão** quando não houver carga decisória própria |
| `decisao_interlocutoria` | O texto contém o teor de uma decisão sobre questão incidental controvertida (defere/indefere um pedido, resolve controvérsia) — comporta agravo |
| `sentenca` | Põe fim à fase de conhecimento em 1ª instância — mérito ou extinção |
| `acordao` | Decisão colegiada de 2ª instância/tribunal, ou decisão monocrática de relator em agravo interno |

Regra prática: se o texto **apenas informa/ordena** sem decidir uma controvérsia, é `intimacao`. Se o texto **decide** algo contestável (defere, indefere, homologa incidentalmente), é `decisao_interlocutoria`. Sentença/acórdão só quando o texto é, ele mesmo, o teor da sentença/acórdão.

**Com histórico**: se o processo está em `recurso_2a`/`recurso_superior`, decisão colegiada é `acordao`, não `sentenca`. Se está em 1ª instância, `acordao` é implausível — exige texto explícito de julgamento colegiado.

⚠️ Viés conhecido: `intimacao` é simultaneamente o padrão e a classe mais frequente. Ao auditar, monitore especificamente o recall de `decisao_interlocutoria`.

---

## Passo 3 — Camada 2: conteúdo/assunto (34 valores)

Filtre pelos conteúdos compatíveis com o tipo escolhido no passo 2, **depois** pelos plausíveis na fase atual (mapa de fases). Conteúdo implausível para a fase exige texto explícito.

### Sob `decisao_interlocutoria`
- **Despacho de pendência** — juiz decide sobre pendência documental (procuração, comprovante de endereço, documento pessoal, CAT, outros)
- **Perícia judicial** — SÓ se resolver controvérsia (ex. indeferir pedido de nova perícia, determinar substituição do perito); mero agendamento vai em `intimacao`
- **Audiência marcada** — SÓ se for decisão sobre remarcação (deferida/indeferida); mero aviso vai em `intimacao`
- **Redistribuição de autos**
- **Tutela de urgência** — concessão ou negativa de liminar
- **Saneamento** — despacho saneador
- **Processo suspenso** — determinação ou retirada da suspensão
- **Cessão de crédito** — homologação ou não
- **Bloqueio / penhora de valores** — deferimento/indeferimento do pedido de penhora *(⚠️ hipótese não confirmada — ver Questões abertas)*
- **Genérico** — qualquer decisão interlocutória residual sem bucket específico. **Sem vocabulário fechado de qualificação** — registre um resumo livre no campo de qualificação

### Sob `intimacao` (mais frequente — é o veículo padrão)
- **Perícia judicial** — nomeação, agendamento, remarcação, laudo apresentado, esclarecimentos, manifestação sobre laudo, justificar ausência
- **Audiência marcada** — aviso simples ou remarcação
- **Réplica** — intimação do autor pra responder a contestação
- **Alegações finais**
- **Especificar provas** — intimação pra especificar provas que pretende produzir
- **Agravo de Instrumento / Apelação / Agravo Interno / Recurso Especial-Extraordinário** — interposição (identifique a parte!), distribuição no tribunal, agendamento de sessão de julgamento. **A modalidade importa**: Agravo de Instrumento ataca decisão interlocutória; Apelação ataca sentença de mérito ou extinção; Agravo Interno ataca decisão monocrática do relator; Recurso Especial/Extraordinário ataca acórdão (STJ/STF). Se o texto não deixar clara a modalidade, **derive do histórico** (uso 1 do catálogo); sem histórico, `sem_qualificacao` + revisão humana, **sem match de status**.
- **Embargos de declaração** — só a interposição (autor ou réu); resultado do julgamento vai em `sentenca`/`acordao`
- **Acordo** — intimação pra manifestar sobre proposta de acordo (não use "proposta"/"aceito" — esses valores foram descontinuados)
- **Cumprimento de sentença** — iniciado, determinado, necessidade de emenda
- **Cálculo/liquidação de valores** — intimado a apresentar, intimado a se manifestar, homologado, impugnado
- **Depósito em juízo** — efetuado
- **Expedição de precatório/RPV** — requerido, expedido *(só se aplica a devedor público — Aux/INSS)*
- **MLE (levantamento eletrônico)** — deferido, realizado *(só Aux/INSS)*
- **Honorários periciais** — arbitrados, pagos, impugnados, MLE deferido, RPV perito requerido/expedido *(só Aux/INSS)*
- **Embargos de execução** — interposição
- **Execução infrutífera**
- **Implantação do benefício** — ofício, determinação, a confirmar se efetivada, efetivada *(nunca via sentença — sempre intimação; só Aux/INSS)*
- **Trânsito em julgado**
- **Prazo decorrido/silêncio**
- **Custas processuais** — pagamento, complemento
- **Genérico** — intimação residual sem bucket específico; vocabulário fechado: favorável, desfavorável, parcialmente favorável, deferido, indeferido, homologado, "intimação para apresentar manifestação"

### Sob `sentenca`
- **Mérito da causa** — favorável, desfavorável, parcialmente favorável (só 1ª instância — nunca use este conteúdo sob acórdão)
- **Extinção do processo** — pendência de documentos, desistência, outro motivo *(ver regra especial de acumulação no passo 6 — nenhuma classificação isolada aqui gera `extinct`)*
- **Acordo** — homologado, não homologado
- **Embargos de declaração** — mantida, alterada (julgamento do embargo)

### Sob `acordao`
- **Agravo de Instrumento / Apelação / Agravo Interno / Recurso Especial-Extraordinário** — favorável, desfavorável, parcialmente favorável, anulada
- **Embargos de declaração** — mantida, alterada

### Sob `citacao`
- **Citação do réu** — sem qualificação específica

**Não existem mais** (removidos em rodadas de validação anteriores, não use): `Recurso de extinção` (recurso contra sentença de extinção é Apelação como qualquer outra) e `Mérito da causa` sob `acordao` (resultado de recurso vive em Apelação/Agravo/Especial).

---

## Passo 4 — Camada 3: qualificação

Use o vocabulário específico listado junto de cada conteúdo no passo 3. **Toda qualificação, em qualquer conteúdo, também aceita `sem_qualificacao`** ("Sem qualificação — campo aberto para escrever") — use quando o vocabulário fechado não cobre o caso; nesse caso registre um resumo livre do que a publicação diz, não force um valor que não reflete o texto.

Regra dura de consistência: **resultado de julgamento** (favorável/desfavorável/parcialmente favorável/anulada/mantida/alterada) só existe sob `sentenca`/`acordao`. **Estágio processual** (interposição/distribuição/agendamento) só existe sob `intimacao`. Nunca misture os dois no mesmo tipo.

**Campo `parte`** (`autor` | `reu` | `ambos` | `juizo` | `indefinida`): registre separadamente, não dentro da qualificação. É o que distingue `inss_appealed` de `judicial_appeal_awaiting_tribunal_decision`, e `company_appealed` nas verticais de empresa ré. Quando vier do histórico, marque `parte_inferida: true`.

**Limite de uso do histórico nesta camada**: resultado de julgamento nunca é inferido de fase ou expectativa. Só do texto.

---

## Passo 5 — Checagem de coerência com o histórico

Compare a classificação obtida com a fase apurada no passo 0 e classifique a coerência:

| Estado | Condição | Ação |
|---|---|---|
| `coerente` | Conteúdo esperado para a fase, ou avanço de 1 fase | grava normal, segue pro passo 6 |
| `neutro` | Trilha paralela (AI, ED, cessão, suspensão, prazo decorrido) ou conteúdo possível mas não esperado | grava normal, segue pro passo 6 |
| `salto` | Pula 2+ fases para frente | só grava status se o texto for explícito e inequívoco; caso contrário grava classificação e sinaliza |
| `regressao` | Move para trás **sem** gatilho da lista de regressões legítimas | grava classificação, **não dispara status**, sinaliza |
| `contradicao` | Incompatível com marco irreversível (ex.: citação do réu após sentença; designação de perícia após trânsito em julgado) | grava classificação, **não dispara status**, sinaliza em prioridade alta — suspeite de publicação de outro processo, erro de captura ou número de processo trocado |

Grave sempre o campo `coerencia` no resultado. É o principal instrumento de auditoria do uso do histórico: se a taxa de `contradicao` subir, o problema está na captura, não no classificador.

---

## Passo 6 — Match de status por vertical

Depois de ter `{tipo, conteudo, qualificacao, parte}`, resolva o status do `Order` usando a tabela da vertical certa (identificada pelo `problem_kind`) — **e somente se**:

1. o pedido for de **origem externa** (regra crítica no topo);
2. a `coerencia` for `coerente`, `neutro`, ou `salto` com texto explícito;
3. nenhum componente da classificação tiver sido **presumido sem suporte textual**;
4. a publicação não for `duplicata`;
5. a transição passar na regra de não-regressão abaixo.

### Regra de não-regressão

Transição para status de fase **anterior** à fase do status atual do Order só é permitida quando o gatilho está na tabela de regressões legítimas (mapa de fases). Fora disso: mantenha o status atual, grave a classificação e sinalize.

Motivo prático: publicação atrasada, republicação em outro diário ou backfill fora de ordem são comuns, e sem essa guarda um `intimacao > Cálculo/liquidação > intimado a apresentar` derruba um Order que já está em fase de pagamento.

### Regra especial — extinção e o status `extinct` (definitivo)

Esta regra **não é um par tipo/conteúdo/qualificação isolado** — é uma regra de acumulação de sinais ao longo do processo:

- Uma publicação isolada de **extinção em 1ª instância** (sentença de extinção) resolve para `extinction_court_decision`. **Nunca** resolve direto para `extinct`.
- `extinct` só é atingido quando, **além** da extinção em 1ª instância, o processo também acumula uma confirmação de que a extinção ficou definitiva **contra o autor** — ou seja:
  - trânsito em julgado **sem** recurso do autor contra a extinção, ou
  - decisão de 2ª instância/tribunal **mantendo** a extinção (desfavorável ao autor em sede de Apelação contra a sentença de extinção).
- Se o trânsito em julgado ou a decisão de 2ª instância forem **favoráveis ao autor** (extinção revertida), não aplique `extinct` — trate pela trilha normal de Apelação (favorável/desfavorável/parcial) do passo anterior.
- Com o histórico carregado no passo 0, a verificação é direta: procure no histórico do processo uma `sentenca > Extinção do processo` anterior. Se existir **e** a publicação atual for o trânsito em julgado sem recurso do autor, ou acórdão desfavorável ao autor em apelação contra essa extinção, os dois sinais estão acumulados.
- Mesmo com os dois sinais presentes, `extinct` permanece **promoção manual**: a skill nunca grava `extinct` sozinha. Grave o status da trilha normal e sinalize `pronto_para_promocao_extinct: true` para o analista confirmar.
- Se o histórico estiver ausente ou truncado, **não infira** — grave `extinction_court_decision` e sinalize.

### Auxílio Acidente (`accident_benefit_problem`)

| Tipo | Conteúdo | Qualificação / parte | Status |
|---|---|---|---|
| intimacao | Apelação/Agravo/Especial | interposição, parte=autor | `judicial_appeal_awaiting_tribunal_decision` |
| intimacao | Apelação/Agravo/Especial | interposição, parte=réu | `inss_appealed` |
| intimacao | Apelação/Agravo/Especial | distribuição/agendamento | `judicial_appeal_awaiting_tribunal_decision` |
| acordao | Apelação/Agravo/Especial | favorável/parcial | `favorable_tribunal_decision` |
| acordao | Apelação/Agravo/Especial | desfavorável | `unfavorable_tribunal_decision` |
| acordao | Apelação/Agravo/Especial | anulada | `awaiting_court_decision` |
| intimacao | Acordo | manifestar sobre proposta | `settlement_proposed_by_inss` |
| intimacao | Embargos de execução | interposição | `values_liquidation` |
| intimacao | Cumprimento de sentença | necessidade de emenda | `values_liquidation` |
| intimacao | Cálculo/liquidação | intimado a apresentar / a se manifestar / impugnado | `values_liquidation` |
| intimacao | Cálculo/liquidação | **homologado** | `calculation_approved` |
| intimacao | Audiência marcada | — | `scheduled_hearing` |
| intimacao | Implantação do benefício | qualquer qualificação | `benefit_implementation` |
| intimacao | Expedição de RPV | expedido | `rpv_precatory_expedition` |
| intimacao | MLE | deferido | `rpv_precatory_expedition` |
| intimacao | Perícia judicial | agendamento | `judicial_expert_examination_scheduled` |
| sentenca | Extinção do processo | qualquer (sinal isolado) | `extinction_court_decision` |
| sentenca/acordao | Extinção + trânsito ou 2ª instância contra o autor (acumulado) | — | `extinct` *(promoção manual — ver regra especial)* |
| decisao_interlocutoria | Processo suspenso | suspensão determinada | `lawsuit_suspended` |
| intimacao | Réplica | — | `awaiting_court_decision` |
| sentenca | Mérito da causa | favorável | `favorable_court_decision` |
| sentenca | Acordo | homologado | `favorable_court_decision` |
| sentenca | Mérito da causa | desfavorável | `unfavorable_court_decision` |
| sentenca | Mérito da causa | parcialmente favorável | `partially_favorable_court_decision` |
| intimacao | Perícia judicial | manifestação sobre laudo / laudo apresentado | `awaiting_court_decision` |
| intimacao | Custas processuais | pagamento | *sem transição* |
| intimacao | Honorários periciais | MLE deferido | *sem transição* |

Status confirmados como **não existentes** para match de publicação (não implemente, não invente roteamento):
- **"Pagamento parcial realizado"** — não existe esse status para dar match.
- **"Aguardando reprotocolo"** — é um status puramente financeiro/operacional, sem publicação judicial associada. Não considerar no matcher.

`defense_presented_by_inss` existe no enum mas não tem match de classificação conhecido.

### INSS (`inss_restitution_problem`)

Mesmo rito de Aux, troca "benefício" por "restituição". Diferenças: `essential_documents_analysis` (Despacho de pendência::outros), `favorable_court_ruling_reduction` (parcialmente favorável = redução de valor), `awaiting_restitution_calculation`, `awaiting_rpv_expedition`/`awaiting_rpv_payment` (mais granular que Aux), `received_restitution_value`. Perícia judicial e Implantação do benefício não se aplicam. A mesma regra especial de acumulação de extinção se aplica aqui: `lawyer_extinction_appeal_presented`/`awaiting_extinction_appeal_decision`/`favorable_extinction_appeal_decision` cobrem a trilha recorrível; o equivalente a `extinct` também exige o sinal acumulado, nunca uma publicação isolada.

### Voos (`flight_problem`), BRS (`social_network_problem`), NNI (`negativation_problem`)

⚠️ **Cobertura provisória**: a validação empírica completa (17.817 casos) existe só para Auxílio Acidente. Para estas três verticais as regras abaixo são as documentadas; tudo que não estiver aqui vai para revisão humana.

Rito "empresa ré" — réu é a empresa/plataforma/credor, não o Estado. RPV/precatório e MLE **não se aplicam** (mecanismos exclusivos de devedor público). Recurso com `parte=reu` sempre vira `company_appealed`. Padrões:
- Citação do réu → `waiting_for_company_position`
- Réplica → `awaiting_court_decision`
- Acordo::manifestar sobre proposta → `agreement_proposal_received` (BRS/NNI) ou `offer_of_anticipation` (Voos)
- Acordo::homologado (sentença) → `reached_agreement_and_awaiting_company_payment` (BRS) ou `awaiting_judicial_payment` (Voos)
- Mérito favorável → `definitive_account_unblock` (BRS), `defined_cleaning_decision` (NNI), `favorable_court_decision` (Voos)
- Mérito desfavorável → `unfavorable_judicial_decision` (BRS), `rejected_on_register` (NNI), `unfavorable_court_decision` (Voos)
- Tutela concedida → `preliminary_injunction_account_unblock` (BRS) ou `clear_name_authorized_by_judge` (NNI); Voos não tem status dedicado
- Bloqueio/penhora::efetivado (BRS) → `requested_lock`
- Despacho de pendência::documento pessoal (NNI) → `missing_personal_documents`
- Cumprimento de sentença (NNI) → `awaiting_debt_payment_time` ⚠️ ver Questões abertas

Se a combinação específica não estiver documentada aqui, **não invente o status** — marque para revisão humana em vez de adivinhar.

---

## Resultado — contrato de saída

```json
{
  "publicacao_id": "...",
  "order_id": "...",
  "camada_0": "publicacao | andamento",
  "tipo": "citacao | intimacao | decisao_interlocutoria | sentenca | acordao | null",
  "conteudo": "<valor da lista do passo 3> | null",
  "qualificacao": "<vocabulário do conteúdo> | sem_qualificacao | null",
  "qualificacao_texto_livre": "resumo, quando sem_qualificacao ou Genérico",
  "parte": "autor | reu | ambos | juizo | indefinida | null",
  "parte_inferida": false,
  "fase_apurada": "distribuicao | instrucao | ... | encerrado | desconhecida",
  "coerencia": "coerente | neutro | salto | regressao | contradicao",
  "usou_historico": true,
  "duplicata": false,
  "status_sugerido": "<status> | null",
  "status_bloqueado_por": "origem_resolvvi | nao_regressao | coerencia | duplicata | presuncao | null",
  "revisao_humana": false,
  "motivo_revisao": ["..."],
  "pronto_para_promocao_extinct": false
}
```

`status_sugerido: null` com `revisao_humana: true` é um resultado **válido e desejável** — é melhor que um status errado.

---

## Casos que exigem revisão humana

Marque a publicação para revisão manual quando:
- A Camada 0 for ambígua (não fica claro se é publicação ou andamento).
- A modalidade de recurso não for identificável no texto **nem derivável do histórico**.
- A parte recorrente não for identificável e a sentença anterior tiver sido parcialmente favorável.
- O conteúdo não se encaixar em nenhum dos 34 valores nem no `Genérico` de forma satisfatória.
- A `coerencia` for `regressao` ou `contradicao` (sempre), ou `salto` sem texto explícito.
- Envolver os 3 itens documentados como "não implementados por falta de origem confirmada": *Manifestação da União — Dossiê previdenciário*, *Encaminhado para pagamento*, *RPV pago* (Aux) — não classifique, sinalize.
- A vertical for Voos/BRS/NNI/INSS e a combinação não constar explicitamente na tabela de match.
- O caso envolver possível `extinct`: nunca grave `extinct` automaticamente, mesmo com os dois sinais acumulados.
- O pedido não for claramente de origem externa — classifique `{tipo, conteudo, qualificacao}` mas não dispare o match de status.
- O histórico do processo estiver **indisponível** e a classificação depender dele para desambiguar.

---

## Questões abertas

| # | Questão | Impacto |
|---|---|---|
| 1 | `Bloqueio / penhora de valores` sob `decisao_interlocutoria` — hipótese não confirmada. Se o texto só avisa que a penhora ocorreu, usar `intimacao` | conteúdo + status BRS |
| 2 | `awaiting_debt_payment_time` (NNI, cumprimento de sentença) — significado não confirmado | status NNI |
| 3 | Janela de dedupe de 10 dias — validar contra dados reais de republicação em diários | falsos positivos de duplicata |
| 4 | Publicação composta (ato múltiplo num só texto: "homologo os cálculos, intimem-se e designo perícia") — o esquema é single-label e a regra ainda não está definida | volume desconhecido, potencialmente relevante |

---

## Migração retroativa — combinações antigas sem destino direto

Ao migrar dados históricos do formato antigo (classificação + sub-classificação) para `{tipo, conteudo, qualificacao}`, 12 combinações antigas (45 registros de 29.207, 0,15% da base) não têm destino óbvio:

| Combinação antiga | Destino no formato novo |
|---|---|
| Julgamento dos embargos de declaração :: Favorável | Embargos de declaração > alterada (sentenca **ou** acordao — ver linha abaixo) |
| Julgamento dos embargos de declaração :: Desfavorável | Embargos de declaração > mantida |
| Julgamento dos embargos de declaração :: Parcial | Embargos de declaração > alterada |
| *(tipo do ato das 3 linhas acima, quando não registrado)* | assumir `sentenca` |
| Cessão de crédito :: Comunicação | intimacao > Cessão de crédito > sem_qualificacao |
| Acordo :: Acordo fechado | deixar em branco (sub não existe na lista oficial) |
| Cumprimento de sentença :: Proposto pelo Autor | deixar em branco (sub é de "Recurso", entrou por engano na base antiga) |
| Perícia Judicial :: Realizada | deixar em branco (sub não existe na lista oficial) |
| Citação realizada | deixar em branco (nem é classificação da lista oficial) |
| INSS manifestar-se sobre o pedido administrativo | deixar em branco (fase pré-judicial, fora da taxonomia — Camada 0) |
| INSS manifestar-se sobre pedido de desistência da ação | deixar em branco (fase pré-judicial, fora da taxonomia — Camada 0) |
| Intimação do réu sobre os cálculos da execução | intimacao > Cálculo/liquidação de valores > intimado a se manifestar |
| Expedido RPV / Precatório :: Advogado | intimacao > Expedição de precatório/RPV > expedido |

Isso é só para o **backfill histórico** — não afeta classificação de publicações novas. Importante: o backfill **não dispara match de status** e **não usa** a lógica de histórico deste documento; ele apenas traduz rótulos antigos.

---

## Integração de dados (a preencher antes de rodar via /schedule)

Esta skill descreve **a lógica de classificação**, não a integração com o banco/API. Antes de agendar uma execução autônoma, defina e documente aqui:

1. **Origem**: de onde vêm as publicações pendentes de classificação (tabela, endpoint, fila)?
2. **Campo de texto**: qual coluna/campo contém o texto da publicação a classificar?
3. **`problem_kind`**: como a publicação vem associada à vertical, ou isso precisa ser inferido?
4. **Origem externa vs. Resolvvi**: qual campo indica a origem do pedido?
5. **Histórico de classificações** (passo 0): qual query traz as últimas 10 classificações + marcos do mesmo processo? Join por `order_id` ou por número de processo? O campo `revisado_por_humano` existe?
6. **Status atual do Order**: de onde ler, e onde ficam registradas as transições anteriores (para a regra de não-regressão)?
7. **Escrita**: onde gravar o contrato de saída completo, incluindo `coerencia`, `usou_historico` e `motivo_revisao`? Fila separada para revisão humana ou flag numa coluna?
8. **Volume e ordem por execução**: quantas publicações por rodada, e como garantir o processamento em ordem ascendente de data dentro do lote?

*(Sem os itens 5 e 6, a lógica de histórico deste documento não roda — a skill degrada para classificação de texto isolado, o que é aceitável mas perde a maior parte do ganho de precisão.)*
