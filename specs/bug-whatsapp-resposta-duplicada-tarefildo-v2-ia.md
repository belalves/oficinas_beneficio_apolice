# Bug Spec — Resposta duplicada no WhatsApp (workflow `tarefildo_v2_ia`)

- **ID:** BUG-WA-001
- **Workflow afetado:** `tarefildo_v2_ia` (workflow principal, n8n)
- **Integração:** WAHA (WhatsApp HTTP API — não oficial)
- **Severidade:** Alta — afeta 100% das mensagens recebidas
- **Status:** Aberto
- **Autor:** Bel Alves
- **Data:** 2026-07-15

---

## 1. Resumo

Ao enviar **uma** mensagem pelo WhatsApp, o workflow `tarefildo_v2_ia` é acionado
**duas vezes** no n8n (duas execuções distintas no histórico de *Executions*),
resultando em **duas respostas** enviadas ao usuário para uma única mensagem recebida.

## 2. Comportamento esperado

- 1 mensagem recebida → **1** execução do workflow → **1** resposta enviada.

## 3. Comportamento atual

- 1 mensagem recebida → **2** execuções do workflow → **2** respostas enviadas.

## 4. Passos para reproduzir

1. Garantir que o workflow `tarefildo_v2_ia` está **ativo** (produção).
2. Enviar **uma única** mensagem de texto do número de teste para o número conectado ao WAHA.
3. Abrir n8n → **Executions** do workflow.
4. **Observado:** aparecem **2 execuções** disparadas com poucos milissegundos/segundos
   de diferença, ambas concluídas com sucesso, e o usuário recebe **2 respostas**.

> Anexar ao ticket: print das duas execuções, o **payload** de cada uma (nó de trigger →
> aba *Input/JSON*) e a config de webhook do WAHA. Isso confirma qual das hipóteses abaixo é a real.

## 5. Causa raiz (hipóteses priorizadas)

O WAHA é conhecido por gerar **múltiplos eventos por mensagem**. As causas mais prováveis,
em ordem de probabilidade:

### H1 — Webhook inscrito em `message` **e** `message.any` (mais provável)
O WAHA emite eventos separados por tipo. `message` dispara para mensagens recebidas e
`message.any` dispara para **todas** as mensagens (recebidas **e** enviadas, incluindo as
do próprio bot). Se a configuração de webhook do WAHA lista **os dois** eventos apontando
para a **mesma** URL do n8n, cada mensagem recebida gera **2 chamadas** → 2 execuções.
> Como confirmar: nos 2 payloads, o campo de evento (`event`) virá diferente —
> um `message` e outro `message.any` — para o **mesmo** `payload.id`.

### H2 — Echo da própria resposta (`fromMe: true`) re-disparando o workflow
Se o webhook escuta `message.any` (ou `message` num engine que inclui `fromMe`), a
**resposta enviada pelo bot** volta como um novo evento. Sem filtro de `fromMe`, isso gera
uma execução extra a cada resposta — podendo virar **loop**.
> Como confirmar: uma das execuções tem `payload.fromMe === true` (ou `key.fromMe`),
> e o corpo é o texto que **o bot respondeu**, não o que o usuário enviou.

### H3 — Webhook registrado duas vezes (global + por sessão)
O WAHA permite configurar webhook **global** (`WHATSAPP_HOOK_URL` / config do servidor) e
webhook **por sessão** (no `POST /api/sessions`). Se **ambos** apontam para a mesma URL do
n8n, cada evento é entregue **2 vezes**.
> Como confirmar: os 2 payloads são **idênticos** (mesmo `event`, mesmo `id`, mesmo corpo).

### H4 — Retry do WAHA por resposta lenta/não-2xx do webhook
Se o nó **Webhook** do n8n está com *Respond* = *When Last Node Finishes* e o workflow
demora (chamada de IA, etc.), o WAHA pode **estourar timeout** e **reentregar** o evento,
disparando a segunda execução.
> Como confirmar: 2 payloads idênticos, com **gap de tempo** compatível com o timeout,
> e o workflow leva alguns segundos para concluir.

## 6. Correção proposta

Aplicar **defesa em camadas** — corrigir a origem (WAHA) **e** blindar o n8n:

### Passo 1 — Reduzir a inscrição de eventos no WAHA
Configurar o webhook para escutar **somente** o evento de mensagem recebida
(`message`). **Remover** `message.any` e eventos de status (`message.ack`) da lista, salvo
se forem realmente usados. Garantir **um único** ponto de configuração (global **ou** por
sessão, não os dois) apontando para a URL do n8n.

### Passo 2 — Filtro logo após o Trigger (blindagem no n8n)
Adicionar um nó **IF/Filter** imediatamente após o Webhook/Trigger, deixando passar apenas
mensagens legítimas de entrada. Descartar (encerrar o ramo) quando **qualquer** condição falhar:

- `{{ $json.body.event }}` **é** `message` (ignora `message.any`, `message.ack`, etc.)
- `{{ $json.body.payload.fromMe }}` **é diferente de** `true` (ignora echo do próprio bot)
- Existe conteúdo de texto/mídia válido (ignora eventos de status/entrega)

> Ajustar os caminhos (`body.payload...`) ao formato real do payload do WAHA em uso —
> confirmar com o JSON de entrada capturado no passo 4 da reprodução.

### Passo 3 — Deduplicação por `id` da mensagem (idempotência)
Como o WAHA pode reentregar, adicionar dedup por `payload.id`:

- Manter um registro dos **últimos IDs processados** (nó de dados estáticos do workflow,
  Redis, Data Store, ou uma planilha/coluna de controle).
- No início do fluxo, **checar** se `payload.id` já foi processado; se sim, **encerrar** sem responder.

### Passo 4 — Responder rápido para evitar retry (contra H4)
No nó **Webhook**, configurar *Respond* = **Immediately** (retorna 200 na hora) e processar
o restante do fluxo de forma assíncrona. Isso impede que o WAHA reentregue por timeout.

## 7. Critérios de aceite

- [ ] Enviar 1 mensagem gera **exatamente 1** execução em *Executions*.
- [ ] O usuário recebe **exatamente 1** resposta por mensagem.
- [ ] A resposta enviada pelo bot **não** dispara nova execução (sem loop de `fromMe`).
- [ ] Eventos de status/entrega (`message.ack`) **não** disparam resposta.
- [ ] Reenvio do mesmo evento (mesmo `payload.id`) **não** gera resposta duplicada.
- [ ] Teste com 5 mensagens seguidas → 5 execuções, 5 respostas, 0 duplicatas.

## 8. Riscos e observações

- **Falso-negativo do filtro:** se o caminho do campo (`fromMe`, `event`) estiver errado,
  o filtro pode barrar mensagens legítimas. Validar com payload real antes de ativar em produção.
- **Variação por engine do WAHA:** o formato do payload muda entre engines
  (`WEBJS`, `NOWEB`, `GOWS`). Confirmar os nomes de campo do engine em uso.
- **Estado de dedup:** dados estáticos do workflow n8n são limitados; para volume alto,
  preferir Redis/Data Store com TTL nos IDs.

## 9. Checklist de validação após o fix

1. [ ] Aplicar Passos 1–4.
2. [ ] Enviar 1 mensagem → conferir 1 execução / 1 resposta.
3. [ ] Enviar mensagem que gere resposta longa (IA) → conferir que não há retry/duplicata.
4. [ ] Disparar um evento de status manualmente (se possível) → conferir que é ignorado.
5. [ ] Monitorar *Executions* por 24h em produção → 0 duplicatas.
