---
contentType: howto
title: Creare un workflow di follow-up commerciale con CRM e RAG
description: Usa n8n per generare messaggi di follow-up commerciale personalizzati combinando dati CRM e documenti interni tramite RAG.
---

# Creare un workflow di follow-up commerciale con CRM e RAG

Usa n8n per automatizzare i follow-up commerciali personalizzati combinando i dati del CRM con la documentazione interna sui prodotti. Questo workflow si attiva quando cambia lo stage di una trattativa nel CRM, recupera lo storico del cliente e i documenti di prodotto pertinenti tramite [RAG](/advanced-ai/rag-in-n8n.md), genera messaggi personalizzati per email, WhatsApp e SMS, e infine registra l'attivita nel CRM.

[[ workflowDemo("file:///advanced-ai/examples/sales_followup_rag.json") ]]

## Funzionalita principali

Questo workflow utilizza:

* Un [webhook trigger](/integrations/builtin/core-nodes/n8n-nodes-base.webhook.md) per ricevere gli eventi di cambio stage dal CRM (ad esempio "Contacted" o "Proposal Sent").
* Un nodo [HTTP Request](/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/index.md) per recuperare lo storico del cliente dal CRM, incluse interazioni precedenti, prodotti di interesse e note commerciali.
* Una query al [vector store](/advanced-ai/rag-in-n8n.md) per recuperare documenti interni pertinenti (schede prodotto, offerte, listini) tramite RAG.
* Un [AI Agent](/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/index.md) che genera messaggi di follow-up personalizzati per ogni canale di comunicazione, basandosi sul contesto CRM e sui documenti recuperati.
* Un [nodo IF](/integrations/builtin/core-nodes/n8n-nodes-base.if.md) per instradare i messaggi verso il canale appropriato: email, WhatsApp Business API o SMS tramite Twilio.
* Un nodo HTTP Request finale per registrare l'attivita di follow-up nel CRM.

## Struttura del workflow

Il workflow si articola in sei fasi:

### 1. Trigger CRM: DealStageUpdated

Un [nodo Webhook](/integrations/builtin/core-nodes/n8n-nodes-base.webhook.md) ascolta gli eventi di cambio stage provenienti dal CRM. Il payload in ingresso deve includere:

| Campo | Descrizione |
| --- | --- |
| `leadId` | L'identificativo univoco del lead o della trattativa |
| `stage` | Il nuovo stage della trattativa (ad esempio `Contacted`, `Proposal Sent`) |
| `preferredChannel` | Il canale di follow-up: `email`, `whatsapp` o `sms` |

### 2. Recupero storico cliente: GetCustomerHistory

Un nodo [HTTP Request](/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/index.md) chiama l'API del CRM per recuperare:

* Le interazioni precedenti e lo storico delle comunicazioni
* I prodotti per cui il cliente ha mostrato interesse
* Le note commerciali interne e il contesto della trattativa

### 3. Query al vector store: ProductDocsRAG

Un nodo [Vector Store](/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreinmemory.md) interroga la knowledge base documentale per recuperare schede prodotto, offerte in corso e listini pertinenti. Questo sfrutta il [RAG](/advanced-ai/rag-in-n8n.md) per ancorare le risposte dell'AI alle informazioni reali sui prodotti.

/// note | Configura prima il vector store
Prima di utilizzare questo workflow, popola il vector store con la documentazione dei prodotti, le offerte e i listini. Consulta [RAG in n8n](/advanced-ai/rag-in-n8n.md) per le istruzioni su come inserire documenti in un vector store.
///

### 4. Agente AI: FollowUpWriterAI

Un [AI Agent](/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/index.md) riceve lo storico CRM e i documenti recuperati, quindi genera messaggi di follow-up personalizzati. L'agente produce una risposta JSON strutturata:

```json
{
  "email_followup": "...",
  "whatsapp_followup": "...",
  "sms_followup": "...",
  "reasoning_summary": "..."
}
```

Il system prompt istruisce l'agente a:

* Fare riferimento a prodotti specifici e interazioni passate dai dati CRM
* Utilizzare i documenti di prodotto recuperati per informazioni accurate e aggiornate
* Adattare tono e lunghezza per ogni canale (formale per email, conciso per SMS, conversazionale per WhatsApp)

### 5. Instradamento canale: MessageChannel

Un [nodo IF](/integrations/builtin/core-nodes/n8n-nodes-base.if.md) instrada il messaggio in base al campo `preferredChannel`:

* **Email**: invia tramite un'API email (ad esempio [SendGrid](/integrations/builtin/app-nodes/n8n-nodes-base.sendgrid.md), [Gmail](/integrations/builtin/app-nodes/n8n-nodes-base.gmail.md))
* **WhatsApp**: invia tramite WhatsApp Business API usando un nodo [HTTP Request](/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/index.md)
* **SMS**: invia tramite [Twilio](/integrations/builtin/app-nodes/n8n-nodes-base.twilio.md)

### 6. Aggiornamento CRM: RecordFollowUpActivity

Un nodo [HTTP Request](/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/index.md) finale registra i dettagli del follow-up nel CRM, creando un record di attivita con il messaggio inviato, il canale utilizzato e il timestamp.

## Prerequisiti

Prima di utilizzare questo workflow, sono necessari:

* **Accesso API al CRM**: credenziali API per il tuo CRM (ad esempio [Salesforce](/integrations/builtin/credentials/salesforce.md), [HubSpot](/integrations/builtin/credentials/hubspot.md) o qualsiasi CRM con API REST).
* **Vector store con dati prodotto**: un vector store popolato con schede prodotto, offerte e listini. Consulta [RAG in n8n](/advanced-ai/rag-in-n8n.md).
* **Credenziali LLM**: chiave API per il modello linguistico (ad esempio [OpenAI](/integrations/builtin/credentials/openai.md)).
* **Credenziali canale** (almeno uno):
    * Chiave API del servizio email (SendGrid, Gmail, ecc.)
    * Token WhatsApp Business API
    * Account SID e auth token di Twilio

## Variabili template

Configura queste credenziali e variabili nel workflow:

| Variabile | Descrizione |
| --- | --- |
| `CRM_API_BASE_URL` | URL base dell'API del CRM |
| `CRM_API_KEY` | Chiave API o token per l'autenticazione al CRM |
| `WHATSAPP_API_TOKEN` | Token bearer per WhatsApp Business API |
| `TWILIO_ACCOUNT_SID` | Account SID di Twilio per gli SMS |
| `TWILIO_AUTH_TOKEN` | Auth token di Twilio per gli SMS |

## Utilizzo dell'esempio

--8<-- "_snippets/examples-color-key.md"
