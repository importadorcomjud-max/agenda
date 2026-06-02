# AGENDAD3 — Handoff para André

**Data:** 27/05/2026  
**Responsável até aqui:** Matheus  
**Próximo responsável:** André

---

## O que é o sistema

Sistema de agendamento de reuniões comerciais da LINI & PANDOLFI (escritório de advocacia trabalhista). Substitui o fluxo manual via grupo de WhatsApp onde SDRs postavam dados de leads e closers negociavam horários publicamente.

**Fluxo:** SDR preenche formulário → dados salvos na planilha → n8n cria evento no Google Calendar e envia e-mail de confirmação para o lead e o closer.

---

## Acesso ao sistema

| Item | URL / Info |
|------|------------|
| **Site (produção)** | https://importadorcomjud-max.github.io/agenda/ |
| **Senha de acesso** | `D3@2026` |
| **Repositório GitHub** | https://github.com/importadorcomjud-max/agenda |
| **Conta GitHub** | `importadorcomjud-max` (nova conta — conta antiga LPDB-D3 perdeu acesso ao e-mail) |

---

## Arquitetura

```
[Formulário index.html]
        │
        ├──► Google Apps Script ──► Google Sheets "Agenda LPD3" (aba Página1)
        │
        └──► n8n Webhook ──► Google Calendar (Agenda LPD3 compartilhada)
                         └──► Gmail (e-mail HTML de confirmação)
```

### Componentes

| Componente | Detalhes |
|------------|----------|
| **Frontend** | `index.html` — único arquivo HTML+CSS+JS |
| **Planilha** | Google Sheets "Agenda LPD3" — aba `Página1` (agendamentos) + aba `Closers` (lista de closers) |
| **Apps Script** | Gerencia gravação na planilha (doPost) e lista de closers (doGet) |
| **n8n** | `itd3.app.n8n.cloud` — workflow "Agendamento LPD3 — MVP Guilherme" |
| **Google Calendar** | Agenda compartilhada "Agenda LPD3" — todos os closers têm acesso |
| **Conta do n8n/Google** | `importadorcomjud@gmail.com` — conta de serviço do sistema |

---

## URLs e credenciais importantes

| Item | Valor |
|------|-------|
| **n8n webhook (produção)** | `https://itd3.app.n8n.cloud/webhook/agendamento` |
| **Apps Script URL** | `https://script.google.com/macros/s/AKfycbz3syvz6kquQa_xhPzP2ePJtDnh_fgHoXwBtnWxR4FCp00UMn2Pk4spoYEYX94SFq-u/exec` |
| **Google Calendar ID** | `17b0df867214a18e60b1aee5b4d955f1c5fa1e3e4a96cb4809237b5f0e4f990f@group.calendar.google.com` |
| **Conta Google do sistema** | `importadorcomjud@gmail.com` |

---

## Closers cadastrados

| Nome | E-mail |
|------|--------|
| Guilherme | guilherme@d3prime.com |
| Cinara | cinaracavalheiro@lpdb.com.br |
| Emmanuel | emmanuelferreira@lpdb.com.br |
| Erina | erina@d3prime.com |
| Andreia | andreia@d3prime.com |
| Mariana | marianalini@lpdb.com.br |

> Os e-mails são gerenciados na aba **"Closers"** da planilha — não precisa mexer no código para adicionar ou trocar.

---

## Como fazer alterações no sistema

### Adicionar ou trocar e-mail de um closer
1. Abre a planilha "Agenda LPD3"
2. Aba **Closers**
3. Edita o e-mail na coluna B

### Adicionar um novo closer
1. Aba **Closers** da planilha → adiciona linha com Nome e E-mail
2. O formulário já carrega automaticamente

### Alterar o código do site (index.html)
1. Edita o arquivo `index.html` na pasta local
2. Abre o terminal na pasta e roda:
```
git add index.html
git commit -m "descrição da mudança"
git push origin main
```
3. Aguarda ~2 minutos para o GitHub Pages atualizar

### Alterar o workflow do n8n
1. Acessa `itd3.app.n8n.cloud`
2. Abre o workflow "Agendamento LPD3 — MVP Guilherme"
3. Faz a alteração
4. Clica em **Save** → **Publish**

### Alterar o Apps Script
1. Abre a planilha "Agenda LPD3"
2. Menu **Extensões → Apps Script**
3. Faz a alteração
4. **Deploy → Manage deployments → editar → New version → Deploy**
5. Copia a nova URL e atualiza `APPS_SCRIPT_URL` no `index.html`
6. Faz push para o GitHub

---

## O que está funcionando ✅

- Formulário de agendamento com senha de acesso
- Closers carregando dinamicamente da planilha
- Gravação de agendamentos na planilha (aba Página1)
- Criação de evento no Google Calendar (Agenda LPD3 compartilhada)
- E-mail HTML de confirmação para o lead (com link do Google Meet)
- E-mail CC para o closer

---

## Pendências e próximos passos ⏳

### Pendência imediata
- **Sala de espera no Google Meet:** O evento é criado pela conta `importadorcomjud@gmail.com`, que é o "host" da reunião. Quando o closer tenta entrar, o Google Meet pede aprovação do host (que nunca está online). 

  **Solução planejada:** Conectar a conta Google de cada closer no n8n e usar um Switch node para criar o evento com a credencial do closer responsável. Assim o closer é o host e entra direto.
  
  **Passo a passo para implementar:**
  1. No n8n → Credentials → Add → "Google Calendar OAuth2 API"
  2. Nomear como "Google Calendar - [Nome]" para cada closer
  3. Clicar "Sign in with Google" com a conta de cada closer
  4. Após todas as credenciais criadas, adicionar Switch node no workflow roteando por `$json.body.closer`
  5. Cada ramo usa a credencial correta

  **Observação:** Closers com Microsoft 365/Outlook precisam de tratamento diferente (n8n tem nó Microsoft Outlook). Avaliar quantos são antes de implementar.

### Fase 2 (futura)
- **Lembretes WhatsApp via Z-API:** Enviar mensagem automática 24h, 1h e 15min antes da reunião. A conta Z-API já existe. Implementar no n8n como nós adicionais no workflow.

---

## Estrutura do Apps Script (para referência)

```javascript
function doPost(e) {
  // Recebe { acao: 'inserir', linha: [...] } e grava na aba Página1
}

function doGet(e) {
  // Recebe ?acao=listarClosers e retorna JSON [{nome, email}] da aba Closers
}
```

---

## Estrutura do payload enviado ao n8n

```json
{
  "nome": "Nome do Lead",
  "email": "email@lead.com",
  "telefone": "51999999999",
  "closer": "Guilherme",
  "closerEmail": "guilherme@d3prime.com",
  "sdr": "Nome do SDR",
  "dataHoraInicio": "2026-05-25T14:00:00",
  "dataHoraFim": "2026-05-25T15:00:00",
  "data": "2026-05-25",
  "horario": "14:00",
  "modalidade": "Videochamada",
  "origem": "Linkedin",
  "obs": "observações"
}
```

---

## Contatos

| Papel | Nome | Contato |
|-------|------|---------|
| Criador do index.html | Guilherme Dias | guilherme@d3prime.com |
| Conta Google do sistema | Importador COMJUD | importadorcomjud@gmail.com |
