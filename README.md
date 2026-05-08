# Azure AI Search — SharePoint Index with ACL-Based Permission Filtering

This folder contains everything needed to build an Azure AI Search indexing pipeline for **SharePoint Online**, with **ACL-based permission filtering** (user and group OIDs captured at index time and enforced at query time). It also includes a notebook for building an agentic retrieval pipeline on top of the index.

## Folder Structure

| File | Description |
|------|-------------|
| `01_ks1-sharepoint-datasource.rest` | Creates a SharePoint Online data source with ACL ingestion enabled |
| `02_ks1-sharepoint-index.rest` | Defines the search index schema with permission filter fields, vector search, and semantic configuration |
| `03_ks1-sharepoint-skillset.rest` | Configures the skillset (ContentUnderstandingSkill + EmbeddingSkill) |
| `04_ks1-sharepoint-indexer.rest` | Creates the indexer and maps ACL metadata fields |
| `99_sharepoint-query.rest` | Sample queries to verify ACL ingestion and test permission-filtered search |
| `101_knowledge_base.ipynb` | Agentic retrieval pipeline using Azure AI Search Knowledge Bases and Foundry Agent Service |
| `102_create_search_pipeline.ipynb` | Python SDK equivalent of the REST pipeline (Steps 1–4 above) |
| `sample.env` | Template for environment variables — copy to `.env` and fill in your values |
| `requirements.txt` | Python dependencies for the notebooks |

## How It Works

### Part 1 — Index Pipeline (REST files 01–04)

Execute the REST files in order to build the indexing pipeline:

1. **Datasource** — Connects to SharePoint Online. Setting `indexerPermissionOptions: ["userIds", "groupIds"]` instructs the indexer to capture Entra ID user OIDs and group OIDs from SharePoint ACLs.
2. **Index** — Defines the schema. `UserIds` and `GroupIds` are collection fields with `permissionFilter` set; `permissionFilterOption: "enabled"` activates query-time ACL enforcement.
3. **Skillset** — Runs ContentUnderstandingSkill (text chunking) and AzureOpenAIEmbeddingSkill (vector generation). See [Skillset notes](#skillset-notes) below.
4. **Indexer** — Processes documents and maps `metadata_user_ids` / `metadata_group_ids` from the datasource into the index fields.

### Part 2 — Python SDK Pipeline (102_create_search_pipeline.ipynb)

Implements the same Steps 1–4 using the `azure-search-documents` Python SDK. Use this notebook to create or re-create the pipeline programmatically without the REST files.

| Step | Cell | Description |
|------|------|-------------|
| 0 | Install | Install `azure-search-documents --pre` and dependencies |
| 1 | Config | Load `.env`; derive all resource names from `NAME_PREFIX` |
| 2 | Data Source | Create SharePoint data source with `indexer_permission_options=["userIds", "groupIds"]` |
| 3 | Index | Create index with `permission_filter_option="enabled"`; `UserIds` / `GroupIds` fields carry `permission_filter`; `snippet` field uses `ja.microsoft` analyzer |
| 4 | Skillset | Create skillset (ContentUnderstandingSkill + AzureOpenAIEmbeddingSkill); index projections map ACL fields |
| 5 | Indexer | Create and run the indexer; check execution status |
| 6 | Search Test | Hybrid vector + keyword search with permission-filtered results using the caller's Entra token |

### Part 3 — Agentic Retrieval (101_knowledge_base.ipynb)

Builds a knowledge source, knowledge base, and Foundry Agent Service agent on top of the index. Queries are permission-filtered — results are scoped to documents the signed-in user can access in SharePoint.

| Step | Description |
|------|-------------|
| 1 | Load `.env`; set up Azure AI Search and Foundry credentials |
| 2 | Create a **Knowledge Source** pointing to the SharePoint index |
| 3 | Create a **Knowledge Base** that wraps the knowledge source with an LLM for query planning |
| 4 | Test the knowledge base directly via `KnowledgeBaseRetrievalClient` (with permission filtering) |
| 5 | Create an MCP tool connection from the Foundry project to the AI Search MCP endpoint |
| 6 | Create a **Foundry Agent** with the knowledge base MCP tool attached |
| 7 | Start a conversation — the agent calls the knowledge base when the query requires document retrieval |
| 8 | (Optional) Cleanup — delete agent, knowledge base, knowledge source, and index |

## Prerequisites

### For REST Files (Parts 1)

- Azure AI Search service — API version `2025-11-01-Preview`
- SharePoint Online site with an app registration (Application ID + Secret)
- Azure OpenAI service with a `text-embedding-3-large` deployment
- Azure AI Services resource (for ContentUnderstandingSkill)

### For Notebooks (Parts 2 & 3)

- Everything above, plus:
- A [Microsoft Foundry project](https://learn.microsoft.com/azure/ai-foundry/how-to/create-projects) in a [region that supports agentic retrieval](https://learn.microsoft.com/azure/search/search-region-support)
- A [supported LLM](https://learn.microsoft.com/azure/search/search-agentic-retrieval-how-to-create#supported-models) deployed to your project (e.g., `gpt-5-mini`)
- Python 3.13+ with packages listed in `requirements.txt`

## Setup

### 1. Environment Variables

Copy `sample.env` to `.env` and fill in your values:

| Variable | Description |
|----------|-------------|
| `AZURE_SEARCH_ENDPOINT` | Azure AI Search endpoint (e.g., `https://<service>.search.windows.net`) |
| `AZURE_SEARCH_API_KEY` | Azure AI Search admin API key |
| `NAME_PREFIX` | Prefix for all resource names (e.g., `ks-sharepoint-demo`). All derived names — datasource, index, skillset, indexer — are built as `{NAME_PREFIX}-<type>` |
| `AZURE_OPENAI_ENDPOINT` | Azure OpenAI endpoint |
| `AZURE_OPENAI_API_KEY` | Azure OpenAI API key |
| `AI_SERVICES_KEY` | Azure AI Services API key (for ContentUnderstandingSkill) |
| `AI_SERVICES_SUBDOMAIN_URL` | Azure AI Services subdomain URL |
| `SHAREPOINT_CONNECTION_STRING` | SharePoint connection string: `SharePointOnlineEndpoint=https://...;ApplicationId=...;ApplicationSecret=...;TenantId=...` |
| `AZURE_TENANT_ID` | Entra ID tenant ID (used by the notebook for user token acquisition) |
| `PROJECT_ENDPOINT` | Microsoft Foundry project endpoint (notebooks only) |
| `PROJECT_RESOURCE_ID` | Full Azure resource ID of the Foundry project (notebooks only) |
| `AZURE_OPENAI_GPT_DEPLOYMENT` | GPT model deployment name (notebooks only, default: `gpt-5-mini`) |

### 2. REST Files

All REST files read configuration from `.env` via the VS Code REST Client `{{$dotenv VARIABLE_NAME}}` syntax. No manual editing of variable values inside the files is required.

1. Install the [REST Client](https://marketplace.visualstudio.com/items?itemName=humao.rest-client) extension for VS Code.
2. Ensure your `.env` file is populated.
3. Execute `01` through `04` in order by clicking **Send Request**.
4. Use `99_sharepoint-query.rest` to verify ACL ingestion and test permission-filtered search.

For `99_sharepoint-query.rest`, you need an Entra ID access token scoped to `https://search.azure.com`:

```bash
az account get-access-token --resource https://search.azure.com --query accessToken -o tsv
```

Paste the token into the `@access-token` variable at the top of the file.

### 3. Notebooks

```bash
pip install -r requirements.txt
```

Run `102_create_search_pipeline.ipynb` to create the pipeline via Python SDK, then `101_knowledge_base.ipynb` for agentic retrieval.

## Features

- **SharePoint Online data source** — Connects via an Entra app registration; no on-premises gateway required
- **ACL permission filtering** — Entra user OIDs and group OIDs are captured during indexing and used to trim search results at query time
- **Content Understanding Skill** — Extracts text chunks from documents (2000-character chunks, 200-character overlap)
- **Japanese Analyzer** — The `snippet` field uses `ja.microsoft` for morphological analysis of Japanese text
- **Vector Search** — 3072-dimensional HNSW embeddings with scalar quantization (int8) and oversampling for rescoring
- **Semantic Search** — BM25 + semantic reranking
- **Agentic Retrieval** — Knowledge base–driven retrieval with MCP tool integration for Foundry Agent Service

## Notes

### Japanese Analyzer (`ja.microsoft`)

The `snippet` field is configured with `analyzer_name="ja.microsoft"` (REST: `"analyzer": "ja.microsoft"`). This analyzer provides morphological analysis optimized for Japanese text.

> **Important:** This analyzer setting must be specified via the REST API or the Python SDK (as shown in `02_ks1-sharepoint-index.rest` and `102_create_search_pipeline.ipynb`). It **cannot** be set through the Azure AI Search Portal, the Azure AI Foundry Portal, or via the automatic index creation path of a Knowledge Source definition — those interfaces do not expose custom analyzer configuration.

See [Language analyzers in Azure AI Search](https://learn.microsoft.com/azure/search/index-add-language-analyzers) for the full list of supported analyzers.

### Skillset Notes

The skillset in this folder uses **ContentUnderstandingSkill** and **AzureOpenAIEmbeddingSkill** only. `ChatCompletionSkill` is intentionally omitted.

> **Limitation (as of this publication):** `ChatCompletionSkill` cannot coexist with ACL permission filtering in the same skillset. Combining the two results in an indexer error. For this reason, image verbalization via `ChatCompletionSkill` is not available in this ACL-enabled pipeline.

For reference, `ChatCompletionSkill` is used in [`SampleContentUnderstandingSkill/03_ks1-themepark-guide-skillset.rest`](https://github.com/notanaha/SampleContentUnderstandingSkill/blob/main/03_ks1-themepark-guide-skillset.rest?plain=1) — it takes the images extracted by `ContentUnderstandingSkill` as input and generates text descriptions of those images. That pattern is applicable in pipelines that do not require ACL filtering.

## API Version

All REST calls use Azure AI Search API version `2025-11-01-Preview`.

## License

MIT
