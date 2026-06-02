# AGENDAD3 — Sistema de Agendamento LINI & PANDOLFI

Sistema de agendamento de reuniões comerciais de um escritório de advocacia trabalhista. Substitui o fluxo manual via grupo de WhatsApp onde SDRs postavam leads e closers negociavam horários publicamente.

**Usuários:** SDRs (preenchem o formulário), Closers/Advogados (recebem os agendamentos), Leads (clientes potenciais que recebem o convite).

---

## Stack

- **Frontend:** `index.html` — único arquivo HTML + CSS + JS (sem framework)
- **Hospedagem:** GitHub Pages — `https://importadorcomjud-max.github.io/agenda/`
- **Repositório:** `https://github.com/importadorcomjud-max/agenda` — público, branch main
- **Planilha:** Google Sheets "Agenda LPD3" via Google Apps Script
- **Automação:** n8n cloud em `itd3.app.n8n.cloud`
- **Calendário:** Google Calendar — agenda compartilhada "Agenda LPD3"

---

## Arquitetura e fluxo de dados

```
SDR preenche formulário (index.html)
        │
        ├──► gravarNaPlanilha() ──► Apps Script (doPost) ──► Google Sheets aba "Página1"
        │
        └──► enviarParaN8N() ──► n8n webhook ──► Google Calendar (cria evento + Meet)
                                             └──► Gmail (e-mail HTML de confirmação)
```

---

## Constantes críticas no index.html

```javascript
const APPS_SCRIPT_URL = 'https://script.google.com/macros/s/AKfycbz3syvz6kquQa_xhPzP2ePJtDnh_fgHoXwBtnWxR4FCp00UMn2Pk4spoYEYX94SFq-u/exec';
const N8N_WEBHOOK_URL = 'https://itd3.app.n8n.cloud/webhook/agendamento';
```

- `APPS_SCRIPT_URL` — muda a cada novo deploy do Apps Script. Quando mudar, atualizar aqui e fazer push.
- `N8N_WEBHOOK_URL` — webhook de produção do n8n. Não mudar sem atualizar o workflow.

---

## Closers — gerenciamento dinâmico

Os closers **não estão hardcoded** no código. São carregados na inicialização via:

```javascript
fetch(APPS_SCRIPT_URL + '?acao=listarClosers')
```

O Apps Script lê a aba **"Closers"** da planilha e retorna `[{nome, email}]`. Para adicionar, remover ou trocar e-mail de um closer: **editar a planilha**, não o código.

**Closers atuais:**
| Nome | E-mail |
|------|--------|
| Guilherme | guilherme@d3prime.com |
| Cinara | cinaracavalheiro@lpdb.com.br |
| Emmanuel | emmanuelferreira@lpdb.com.br |
| Erina | erina@d3prime.com |
| Andreia | andreia@d3prime.com |
| Mariana | marianalini@lpdb.com.br |

---

## Apps Script — estrutura

Dois endpoints no mesmo script:

```javascript
// POST { acao: 'inserir', linha: [...] } → grava linha na aba Página1
function doPost(e) { ... }

// GET ?acao=listarClosers → retorna [{nome, email}] da aba Closers
function doGet(e) { ... }
```

**Atenção:** Cada novo deploy gera uma URL nova. Sempre atualizar `APPS_SCRIPT_URL` no `index.html` após republicar.

---

## n8n — workflow

**Workflow:** "Agendamento LPD3 — MVP Guilherme"  
**3 nós em sequência:**
1. Webhook (POST `/agendamento`, Respond: When last node finishes)
2. Criar Evento Google Calendar (credencial `importadorcomjud@gmail.com`)
3. Enviar E-mail de Convite (Gmail HTML, CC para closerEmail)

**Payload recebido pelo webhook:**
```json
{
  "nome": "...", "email": "...", "telefone": "...",
  "closer": "Guilherme", "closerEmail": "guilherme@d3prime.com",
  "sdr": "...", "dataHoraInicio": "2026-05-25T14:00:00",
  "dataHoraFim": "2026-05-25T15:00:00",
  "data": "2026-05-25", "horario": "14:00",
  "modalidade": "Videochamada", "origem": "...", "obs": "..."
}
```

**Expressões validadas no nó Google Calendar:**
- Summary: `{{ "Reunião - " + $json.body.nome }}`
- Start: `{{ $json.body.dataHoraInicio }}`
- End: `{{ $json.body.dataHoraFim }}`

---

## Autenticação do site

Tela de senha antes de acessar o sistema. Senha: `D3@2026`.  
Armazenada como SHA-256 no código (`b1a65f2002a4295ae9e078bea4579bf4a9fbd48298b29b22ab3d0bdffdd5d290`).  
Sessão mantida via `sessionStorage` — pede a senha novamente ao fechar o browser.

---

## Pendências conhecidas

### 🔴 Sala de espera no Google Meet
O evento é criado pela conta `importadorcomjud@gmail.com` (host da reunião). Como essa conta nunca entra na chamada, closers de domínios diferentes ficam presos na sala de espera aguardando aprovação.

**Solução planejada:** Switch node no n8n roteando por `closer`, cada ramo com a credencial Google do respectivo closer. Assim o closer é o host e entra direto.

**Passo a passo:**
1. No n8n → Credentials → Add → Google Calendar OAuth2 API para cada closer
2. Nomear "Google Calendar - [Nome]", autenticar com a conta do closer
3. Adicionar Switch node após o webhook, condição: `{{ $json.body.closer }}`
4. Cada ramo usa a credencial correspondente
5. Merge antes do nó de Gmail

**Atenção:** Closers com Microsoft 365 precisarão de nó Microsoft Outlook em vez de Google Calendar.

### 🟡 Fase 2 — Lembretes WhatsApp
Enviar mensagem automática via Z-API 24h, 1h e 15min antes da reunião. A conta Z-API já existe. Implementar como nós adicionais no workflow n8n.

---

## Como publicar alterações

```bash
git add index.html
git commit -m "descrição da mudança"
git push origin main
# aguardar ~2 minutos para GitHub Pages atualizar
```

---

## Contas e acessos

| Sistema | Conta | Observação |
|---------|-------|------------|
| GitHub | `importadorcomjud-max` | Conta nova — conta antiga LPDB-D3 perdeu acesso ao e-mail |
| n8n | `itd3.app.n8n.cloud` | Conta nova, nome neutro (antiga expunha lpdbadvogados) |
| Google (sistema) | `importadorcomjud@gmail.com` | Conta de serviço — Calendar, Gmail e Apps Script |
| Google Calendar compartilhado | ID: `17b0df867214a18e60b1aee5b4d955f1c5fa1e3e4a96cb4809237b5f0e4f990f@group.calendar.google.com` | Todos os closers têm acesso |
