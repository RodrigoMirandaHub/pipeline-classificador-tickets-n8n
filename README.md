# 🎫 Pipeline Classificador de Tickets — Fintech

> Automação inteligente de suporte com N8N + Groq (LLaMA 3.3) + Google Sheets + Gmail

![N8N](https://img.shields.io/badge/N8N-workflow-orange) ![Groq](https://img.shields.io/badge/Groq-LLaMA_3.3-blue) ![Google Sheets](https://img.shields.io/badge/Google-Sheets-green) ![Status](https://img.shields.io/badge/status-funcionando-brightgreen)

---
<img width="1661" height="558" alt="Screenshot_79" src="https://github.com/user-attachments/assets/a66de861-4368-46db-a5d4-f9244af22cb9" />

## 📋 Sobre o Projeto

Pipeline de IA que classifica automaticamente tickets de suporte de uma fintech brasileira. Ao receber uma mensagem via webhook, o sistema:

1. Normaliza o payload recebido
2. Envia para o Groq (LLaMA 3.3) classificar com IA
3. Detecta se é um caso **crítico** (fraude, clonagem, transação não autorizada)
4. Dispara **alerta por e-mail** para casos críticos
5. Grava **todos os tickets** no Google Sheets para análise

---

## 🏗️ Arquitetura

```
POST /webhook/classificar-ticket
         │
         ▼
┌─────────────────────┐
│  Webhook Receiver   │  Recebe o ticket
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Normalizar Payload │  Garante campos obrigatórios
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   Groq API          │  LLaMA 3.3 classifica o ticket
│   (LLaMA 3.3-70b)  │
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│  Parse Classificação│  Extrai JSON da resposta da IA
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   É Crítico?        │  Verifica severidade
└──────┬──────┬───────┘
       │      │
    true    false
       │      │
       ▼      ▼
   Gmail   Google Sheets
  (alerta)  (registro)
       │      │
       └──────┘
          │
          ▼
   Responder Webhook
   (retorna JSON)
```

---

## 🤖 Classificação com IA

O modelo LLaMA 3.3-70b classifica cada ticket em:

### Categorias
| Categoria | Exemplos |
|---|---|
| `fraude` | Cartão clonado, transação não reconhecida |
| `pagamento` | Pix não chegou, boleto com problema |
| `onboarding` | Cadastro, abertura de conta, verificação |
| `suporte_geral` | Dúvidas gerais, informações |

### Severidades
| Severidade | Ação |
|---|---|
| `critico` | **Alerta imediato por e-mail** |
| `alto` | Registrado no Sheets com prioridade |
| `medio` | Registrado no Sheets |
| `baixo` | Registrado no Sheets |

---

## 📥 Payload de Entrada

```json
{
  "ticket_id": "TK-001",
  "cliente": "João Silva",
  "mensagem": "Meu cartão foi clonado e há uma compra de R$ 3.400 que não reconheço.",
  "canal": "app"
}
```

## 📤 Resposta

```json
{
  "sucesso": true,
  "ticket_id": "TK-001",
  "classificacao": {
    "categoria": "fraude",
    "severidade": "critico",
    "resumo": "Cartao clonado",
    "acao_sugerida": "Contatar cliente e bloquear cartao"
  },
  "processado_em": "2026-06-03T18:02:25.947-04:00"
}
```

---

## 🧪 Casos de Teste

### Caso Crítico — Fraude
```bash
curl -X POST http://localhost:5678/webhook/classificar-ticket \
  -H "Content-Type: application/json" \
  -d '{
    "ticket_id": "TK-001",
    "cliente": "João Silva",
    "mensagem": "cartao clonado transacao nao autorizada de R$ 5000",
    "canal": "app"
  }'
```
**Resultado esperado:** E-mail de alerta disparado, severidade `critico`

---

### Caso Baixo — Dúvida Geral
```bash
curl -X POST http://localhost:5678/webhook/classificar-ticket \
  -H "Content-Type: application/json" \
  -d '{
    "ticket_id": "TK-002",
    "cliente": "Maria Costa",
    "mensagem": "qual o horario de atendimento",
    "canal": "chat"
  }'
```
**Resultado esperado:** Gravado no Sheets, severidade `baixo`

---

## ⚙️ Como Configurar

### Pré-requisitos
- N8N instalado (`npm install -g n8n`)
- Conta no [Groq](https://console.groq.com) (gratuito)
- Conta Google (Gmail + Sheets)

### Passo a Passo

**1. Importar o workflow**
- Baixa o arquivo `pipeline-classificador-tickets-groq.json`
- No N8N: `...` → Import from File

**2. Configurar Groq API**
- Cria uma API Key em [console.groq.com](https://console.groq.com)
- No nó `Groq API — Classificar` → Header `Authorization: Bearer SUA_KEY`

**3. Configurar Google Sheets**
- Cria uma planilha com aba `tickets` e os cabeçalhos:
```
ticket_id | cliente | canal | mensagem | categoria | severidade | resumo | acao_sugerida | timestamp_recebido
```
- Cola o ID da planilha no nó `Google Sheets — Gravar Ticket`

**4. Configurar Gmail**
- Conecta via OAuth2 no nó `Gmail — Alerta Crítico`
- Substitui o e-mail de destino no campo `To`

**5. Publicar e testar**
- Toggle `Published` no topo
- Testa com os curls acima

---

## 📊 Google Sheets — Estrutura

| Campo | Descrição |
|---|---|
| ticket_id | ID único do ticket |
| cliente | Nome do cliente |
| canal | app / chat / email / telefone |
| mensagem | Mensagem original |
| categoria | fraude / pagamento / onboarding / suporte_geral |
| severidade | critico / alto / medio / baixo |
| resumo | Resumo gerado pela IA |
| acao_sugerida | Ação recomendada pela IA |
| timestamp_recebido | Data/hora de recebimento |

---

## 🛠️ Stack

- **N8N** — Orquestração do workflow
- **Groq API** — LLM (LLaMA 3.3-70b-versatile)
- **Google Sheets** — Armazenamento e análise
- **Gmail** — Alertas críticos
- **Webhook** — Trigger via HTTP POST
<img width="1661" height="558" alt="Screenshot_79" src="https://github.com/user-attachments/assets/88c02b5b-2f89-4b6e-9586-b1cd7f636c30" />
<img width="1661" height="558" alt="Screenshot_79" src="https://github.com/user-attachments/assets/aaf7e41d-fd6f-48be-b39b-c63a4e53d48e" />

---

## 👨‍💻 Autor

**Rodrigo Miranda**
- GitHub: [@RodrigoMirandaHub](https://github.com/RodrigoMirandaHub)
- LinkedIn: [linkedin.com/in/rodrigo-miranda](https://linkedin.com/in/rodrigo-h/miranda)

---

*Projeto desenvolvido como parte do portfólio de automação com IA — Plano de 90 dias para vaga de Analista de Software II*
