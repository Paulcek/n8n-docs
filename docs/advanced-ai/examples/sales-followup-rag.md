---
contentType: howto
title: Build a sales follow-up workflow with CRM and RAG
description: Use n8n to generate personalized sales follow-up messages using CRM data and RAG-powered product knowledge.
---

# Build a sales follow-up workflow with CRM and RAG

Use n8n to automate personalized sales follow-ups by combining CRM data with internal product knowledge. This workflow triggers when a deal stage changes in your CRM, retrieves customer history and relevant product documents using [RAG](/advanced-ai/rag-in-n8n.md), generates tailored messages for email, WhatsApp, and SMS channels, then records the activity back in your CRM.

[[ workflowDemo("file:///advanced-ai/examples/sales_followup_rag.json") ]]

## Key features

This workflow uses:

* A [webhook trigger](/integrations/builtin/core-nodes/n8n-nodes-base.webhook.md) to receive CRM deal stage change events (for example, "Contacted" or "Proposal Sent").
* An [HTTP Request node](/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/index.md) to fetch the customer's CRM history, including previous interactions, products of interest, and sales notes.
* A [vector store](/advanced-ai/rag-in-n8n.md) query to retrieve relevant internal documents (product sheets, offers, price lists) using RAG.
* An [AI Agent](/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/index.md) that generates personalized follow-up messages for each communication channel based on the CRM context and retrieved documents.
* An [IF node](/integrations/builtin/core-nodes/n8n-nodes-base.if.md) to route the generated messages to the appropriate channel: email, WhatsApp Business API, or SMS via Twilio.
* A final HTTP Request node to record the follow-up activity back in the CRM.

## Workflow structure

The workflow follows six stages:

### 1. CRM trigger: DealStageUpdated

A [Webhook node](/integrations/builtin/core-nodes/n8n-nodes-base.webhook.md) listens for deal stage change events from your CRM. The incoming payload should include:

| Field | Description |
| --- | --- |
| `leadId` | The unique identifier for the lead or deal |
| `stage` | The new deal stage (for example, `Contacted`, `Proposal Sent`) |
| `preferredChannel` | The follow-up channel: `email`, `whatsapp`, or `sms` |

### 2. Fetch customer history: GetCustomerHistory

An [HTTP Request node](/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/index.md) calls your CRM API to retrieve:

* Previous interactions and communication history
* Products the customer has shown interest in
* Internal sales notes and deal context

### 3. Vector store query: ProductDocsRAG

A [Vector Store node](/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.vectorstoreinmemory.md) queries your document knowledge base to retrieve relevant product sheets, current offers, and price lists. This uses [RAG](/advanced-ai/rag-in-n8n.md) to ground the AI's responses in your actual product information.

/// note | Set up your vector store first
Before using this workflow, populate your vector store with product documentation, offers, and price lists. Refer to [RAG in n8n](/advanced-ai/rag-in-n8n.md) for instructions on inserting documents into a vector store.
///

### 4. AI agent: FollowUpWriterAI

An [AI Agent](/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent/index.md) receives the CRM history and retrieved documents, then generates personalized follow-up messages. The agent outputs a structured JSON response:

```json
{
  "email_followup": "...",
  "whatsapp_followup": "...",
  "sms_followup": "...",
  "reasoning_summary": "..."
}
```

The system prompt instructs the agent to:

* Reference specific products and past interactions from the CRM data
* Use retrieved product documents for accurate, up-to-date information
* Adapt tone and length for each channel (formal for email, concise for SMS, conversational for WhatsApp)

### 5. Channel routing: MessageChannel

An [IF node](/integrations/builtin/core-nodes/n8n-nodes-base.if.md) routes the message based on `preferredChannel`:

* **Email**: sends via an email API (for example, [SendGrid](/integrations/builtin/app-nodes/n8n-nodes-base.sendgrid.md), [Gmail](/integrations/builtin/app-nodes/n8n-nodes-base.gmail.md))
* **WhatsApp**: sends via the WhatsApp Business API using an [HTTP Request node](/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/index.md)
* **SMS**: sends via [Twilio](/integrations/builtin/app-nodes/n8n-nodes-base.twilio.md)

### 6. CRM update: RecordFollowUpActivity

A final [HTTP Request node](/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/index.md) posts the follow-up details back to your CRM, creating an activity record with the message sent, channel used, and timestamp.

## Prerequisites

Before using this workflow, you need:

* **CRM API access**: API credentials for your CRM (for example, [Salesforce](/integrations/builtin/credentials/salesforce.md), [HubSpot](/integrations/builtin/credentials/hubspot.md), or any CRM with a REST API).
* **Vector store with product data**: A populated vector store containing your product sheets, offers, and price lists. Refer to [RAG in n8n](/advanced-ai/rag-in-n8n.md).
* **LLM credentials**: API key for your language model (for example, [OpenAI](/integrations/builtin/credentials/openai.md)).
* **Channel credentials** (at least one):
    * Email service API key (SendGrid, Gmail, etc.)
    * WhatsApp Business API token
    * Twilio account SID and auth token

## Template variables

Configure these credentials and variables in the workflow:

| Variable | Description |
| --- | --- |
| `CRM_API_BASE_URL` | Base URL for your CRM API |
| `CRM_API_KEY` | API key or token for CRM authentication |
| `WHATSAPP_API_TOKEN` | WhatsApp Business API bearer token |
| `TWILIO_ACCOUNT_SID` | Twilio account SID for SMS |
| `TWILIO_AUTH_TOKEN` | Twilio auth token for SMS |

## Using the example

--8<-- "_snippets/examples-color-key.md"
