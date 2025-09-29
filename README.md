## Customer Support Automation — n8n Templates

Two ready-to-import n8n workflows for automating customer support on Gmail and powering a Pinecone RAG workflow from your historical email threads.

### What’s included
- **customer-support-gmail-automation-rag-agent.json**: Watches a Gmail inbox for new unread messages, filters for genuine support emails, generates a suggested reply using an agent with Pinecone RAG context, saves a Gmail draft, appends the Q/A to Google Sheets, and labels the thread.
- **gmail-inbox-to-pinecone.json**: One-click/manual workflow to backfill a Pinecone index from sent Gmail threads by extracting clean Question/Answer pairs (customer question → support reply) and embedding them.

### Prerequisites
- n8n instance (self-hosted or cloud). See n8n docs at [n8n.io](https://n8n.io/)
- Gmail account with API access (OAuth2)
- OpenAI API key
- Pinecone account and index created (dimensions should match the embedding model)
- Google Sheets document (for logging generated replies)

### Credentials to set up in n8n
- **Google OAuth2 API**: For Gmail and Google Sheets
- **OpenAI**: For chat and embeddings
- **Pinecone**: For vector store read/write

### How to import a workflow
1. In n8n, go to Workflows → Import from File.
2. Choose either `customer-support-gmail-automation-rag-agent.json` or `gmail-inbox-to-pinecone.json`.
3. After import, open the workflow and attach credentials to all nodes that require them.

---

## Workflow 1: Customer Support Gmail inbox agent with RAG (Pinecone)
File: `customer-support-gmail-automation-rag-agent.json`

### What it does
- **Gmail Trigger** polls unread emails (`q: -from:me is:unread`).
- **Filter Support-Relevant Emails** uses a small LLM to return JSON `{ "needsReply": boolean }`.
- If `needsReply` is true:
  - **Reply Generator agent** calls a Pinecone tool to retrieve similar historical Q/A, then drafts a context-aware reply.
  - **Gmail - Create Draft** creates a reply draft in the original thread and addresses the sender.
  - **Append row in sheet** logs the thread, subject, customer message, and AI-generated reply to Google Sheets.
  - **Add label to thread** applies a Gmail label to mark processed items.

### Nodes to configure
- **Gmail Trigger**: Attach Google OAuth credentials; confirm the search query meets your needs.
- **Filtering step model (gpt-5-mini)**: Attach OpenAI credentials; model name can be changed.
- **Reply Generator agent / Agent chat model**: Attach OpenAI credentials; model name can be changed.
- **Pinecone Vector Store**: Attach Pinecone credentials; set `pineconeIndex` to your index name.
- **Embeddings OpenAI Model**: If used by your Pinecone tool in your setup, attach OpenAI creds and confirm dimensions match the index.
- **Gmail - Create Draft**: Attach Google OAuth credentials.
- **Add label to thread**: Provide a valid Gmail label ID (or replace with your own label).
- **Append row in sheet (Google Sheets)**: Attach Google OAuth credentials and set the target document and sheet.

### Replace placeholders
- In the agent system prompt, replace `[COMPANY_NAME]` with your company name.
- In sticky notes, update `GOOGLE_SHEET_URL` and any internal references.
- The Pinecone index in the template is `[PINECONE-INDEX]` with dimension `1536` (compatible with common OpenAI embedding models). replace `[PINECONE-INDEX]` with your index and ensure your index settings match your chosen embedding model.

### Activating
- Turn the workflow “Active” in n8n after credentials are set. New unread support emails will produce Gmail drafts and log entries.

---

## Workflow 2: Create Vector DB from Gmail inbox (Pinecone)
File: `gmail-inbox-to-pinecone.json`

### What it does
- **Manual Trigger**: Run on demand.
- **Get thread IDs of sent emails**: Fetches Gmail threads labeled `SENT` since 2022-01-01.
- **Get full thread request to Gmail API**: Pulls full thread data via HTTP.
- **Code node**: Decodes MIME parts, extracts plain text, removes quotes/signatures, and builds clean Q/A pairs where the customer’s messages become the Question and your reply becomes the Answer.
- **Default Data Loader → Embeddings → Pinecone Vector Store**: Embeds and inserts the Q/A documents into your Pinecone index.

### Nodes to configure
- **Gmail** and **HTTP Request** nodes: Attach Google OAuth credentials.
- **Embeddings OpenAI**: Attach OpenAI credentials; ensure the embedding model dimension matches your Pinecone index (default 1536).
- **Pinecone Vector Store**: Attach Pinecone credentials; replace `[PINECONE-INDEX]` with the target index name.

### Tuning
- Adjust the Gmail date filter or labels to control which historical threads are indexed.
- `embeddingBatchSize` and text splitter settings can be tuned based on document sizes, we suggest not to chunk the Q/A pairs for better matches.

### Running
- Click “Execute Workflow” to process and upsert Q/A pairs into Pinecone.

---

## Tips and best practices
- **Safety review**: Keep drafts for human-in-the-loop review before sending.
- **Data scope**: Ensure Pinecone is populated with representative support Q/A for good retrieval quality.
- **PII/Compliance**: Verify that storing content in Pinecone complies with your policies.
- **Observability**: Enable execution logging in n8n and keep a Google Sheets log for traceability.

## Troubleshooting
- Authentication errors: Reconnect Google OAuth, OpenAI, and Pinecone credentials in n8n.
- Empty RAG results: Confirm `gmail-inbox-to-pinecone` has populated your index and that the same embedding model/dimensions are used.
- Label errors: Replace the example Gmail label ID with one that exists in your mailbox.
- Rate limits: Add delays or batching if you hit Gmail/OpenAI/Pinecone limits.
