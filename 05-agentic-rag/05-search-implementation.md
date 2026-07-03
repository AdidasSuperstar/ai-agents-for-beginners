# Azure AI Search Implementation Notes

This document summarizes the changes made to `code_samples/05-python-agent-framework.ipynb` to replace the in-memory demo knowledge base with a real Azure AI Search index.

## Goal

The notebook now demonstrates Agentic RAG using Azure AI Search as the retrieval source. The agent receives a search tool, decides when to call it, and grounds answers in documents returned from the configured Azure AI Search index.

## Required Configuration

The notebook expects these environment variables:

```env
AZURE_AI_PROJECT_ENDPOINT=your-foundry-project-endpoint
AZURE_AI_MODEL_DEPLOYMENT_NAME=your-model-deployment-name
AZURE_SEARCH_SERVICE_ENDPOINT=https://your-search-service.search.windows.net
AZURE_SEARCH_API_KEY=your-search-query-or-admin-key
AZURE_SEARCH_INDEX_NAME=demo-datasource-ks-index
```

`AZURE_SEARCH_INDEX_NAME` is optional in the notebook because it defaults to `demo-datasource-ks-index`, matching the index shown in the Azure portal during the demo.

## Package Installation

The setup cell was updated to include the Azure AI Search SDK:

```python
%pip install agent-framework azure-ai-projects azure-identity azure-search-documents python-dotenv -q
```

The important added package is:

```text
azure-search-documents
```

## Imports Added

The setup/import cell now imports the Azure Search credential and client classes:

```python
from azure.core.credentials import AzureKeyCredential
from azure.search.documents import SearchClient
```

The notebook still uses `DefaultAzureCredential` for the Azure AI Foundry client:

```python
from azure.identity import DefaultAzureCredential
```

## Environment Variables Loaded

The setup cell now loads both the Foundry configuration and the Azure Search configuration:

```python
endpoint = os.getenv("AZURE_AI_PROJECT_ENDPOINT")
deployment_name = os.getenv("AZURE_AI_MODEL_DEPLOYMENT_NAME")
search_endpoint = os.getenv("AZURE_SEARCH_SERVICE_ENDPOINT")
search_index_name = os.getenv("AZURE_SEARCH_INDEX_NAME", "demo-datasource-ks-index")
search_api_key = os.getenv("AZURE_SEARCH_API_KEY")
```

The required-variable check was expanded to validate the Search endpoint and key:

```python
missing = [k for k, v in {
    "AZURE_AI_PROJECT_ENDPOINT": endpoint,
    "AZURE_AI_MODEL_DEPLOYMENT_NAME": deployment_name,
    "AZURE_SEARCH_SERVICE_ENDPOINT": search_endpoint,
    "AZURE_SEARCH_API_KEY": search_api_key,
}.items() if not v]
```

## Azure AI Search Client Initialization

A new notebook cell initializes `SearchClient`:

```python
search_client = SearchClient(
    endpoint=search_endpoint,
    index_name=search_index_name,
    credential=AzureKeyCredential(search_api_key),
)

print(f"Azure AI Search index configured: {search_index_name}")
```

This creates the client used by the retrieval tool.

## In-Memory Knowledge Base Replaced

The old notebook used a hard-coded dictionary named `TRAVEL_KNOWLEDGE_BASE`. That was replaced with a tool that queries Azure AI Search directly.

A helper formats search results while skipping internal Search metadata and vector fields:

```python
def format_search_result(result: dict) -> str:
    visible_fields = []

    for field_name, field_value in result.items():
        if field_name.startswith("@search."):
            continue
        if field_value is None:
            continue
        if field_name.lower().endswith("vector"):
            continue

        field_text = " ".join(str(field_value).split())
        if len(field_text) > 500:
            field_text = f"{field_text[:500]}..."
        visible_fields.append(f"{field_name}: {field_text}")

    return "\n".join(visible_fields)
```

The new tool is named `search_knowledge_base`:

```python
@tool(approval_mode="never_require")
def search_knowledge_base(
    query: Annotated[str, "The search query for the Azure AI Search index"]
) -> str:
    """Search the Azure AI Search index for relevant information."""
    results = search_client.search(search_text=query, top=3)
    matches = []

    for result in results:
        formatted_result = format_search_result(dict(result))
        if formatted_result:
            matches.append(formatted_result)

    return (
        "\n\n---\n\n".join(matches)
        if matches
        else "No matching documents found in the Azure AI Search index."
    )
```

## Smoke Test Cell Added

A small test cell was added after the tool definition to verify that the index can be queried before the agent uses it:

```python
sample_results = search_client.search(search_text="architecture", top=1)

for result in sample_results:
    print(format_search_result(dict(result)))
```

This confirmed that the configured index returns documents successfully.

## Agent Updated to Use Azure Search

The main RAG agent now receives `search_knowledge_base` as its tool:

```python
agent = client.as_agent(
    tools=[search_knowledge_base],
    name="AzureSearchRAGAgent",
    instructions="""You are a knowledgeable assistant. Before answering questions:
1. ALWAYS search the Azure AI Search index first
2. Base your answers on retrieved information
3. If information is not in the index, say so clearly
4. Cite concrete details from the retrieved snippets.""",
)
```

The demo prompt was changed to match the actual indexed content:

```python
response = await agent.run(
    "What does the knowledge base say about Barbie and Gloria?",
)
print(response)
```

## Checker Agent Updated

The iterative retrieval example also now uses `search_knowledge_base`:

```python
checker_agent = client.as_agent(
    tools=[search_knowledge_base],
    name="AzureSearchRAGCheckerAgent",
    instructions="""You are a meticulous screenplay knowledge-base assistant who verifies answers against retrieved document snippets.
When answering questions about characters, scenes, dialogue, or events:
1. Search the Azure AI Search index for the main names, titles, or phrases in the question
2. Search again using specific character names, scene details, or quoted phrases found in the first results
3. Compare the retrieved snippets for consistent evidence about actions, relationships, and context
4. Ground the final answer in the retrieved snippets and mention the source document or blob URL when available
5. If the index does not contain enough evidence, clearly say what is missing instead of guessing.""",
)
```

The checker prompt was updated as well:

```python
response = await checker_agent.run(
    "Compare what the knowledge base says about Barbie and Ken.",
)
print(response)
```

## Markdown Updates

The notebook text was updated so it no longer describes only the old travel dictionary. It now explains that:

- Azure AI Search is the external retrieval source.
- The search index is exposed as an agent tool.
- The agent grounds answers in retrieved index snippets.
- The maker-checker pattern can perform multiple retrieval rounds against the same index.

## Validation Performed

The following notebook cells were run successfully after the changes:

1. Package installation cell.
2. Import and environment setup cell.
3. Azure AI Foundry client initialization cell.
4. Azure AI Search client initialization cell.
5. Azure AI Search-backed tool definition cell.
6. Direct Azure AI Search smoke test cell.
7. Main RAG agent cell.

The smoke test returned a document from the configured index, and the main RAG agent produced an answer grounded in content from the indexed `BARBIE FINAL 2023 GG SCREENPLAY.pdf` document.

## Teaching Summary

The key implementation pattern is:

1. Load Search endpoint, index name, and API key from environment variables.
2. Instantiate `SearchClient`.
3. Wrap `search_client.search(...)` in an Agent Framework `@tool`.
4. Pass that tool to `client.as_agent(...)`.
5. Instruct the agent to search before answering.

This turns Azure AI Search into an agent-controlled retrieval tool for Agentic RAG.
