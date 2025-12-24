# code_review_fix
Workspace: Collecting workspace information# RAG2 Notebook - Agentic Code Fix with Vector Search

## Overview

This notebook implements a **LangGraph-based agentic workflow** that:
1. **Generates code fixes** for Sonar issues using an LLM
2. **Validates & repairs** malformed JSON responses
3. **Stores fixes** with embeddings in pgvector
4. **Searches for similar issues** using vector similarity

## Architecture

```
┌─────────────────┐
│   llm_fix       │  Generate fix using LLM
└────────┬────────┘
         │
┌────────▼────────┐
│   validate      │  Validate JSON structure
└────────┬────────┘
         │
    ┌────┴─────┐
    │ error?   │
    └────┬──────┘
         │
    ┌────┴─────────────┐
    │ No              Yes
    │                  │
┌───▼─────┐      ┌─────▼──────┐
│  store  │      │   repair   │
└───┬─────┘      └──────┬─────┘
    │                   │
┌───▼──────────────────┐│
│ search_similar_      │
│ issues               │
└────┬──────┬─────────┘
     │      │
     └──┬───┘
        │
     ┌──▼──┐
     │ END │
     └─────┘
```

## Key Components

### Data Models

#### `CodeFixResponse`
Structured output from the LLM containing:
- `issue_number`: Sonar rule identifier (e.g., "S2111")
- `type_of_issue`: Bug, Code Smell, Vulnerability
- `from_line` / `to_line`: Line range of the issue
- `original_code`: Problematic code snippet
- `fixed_code`: Corrected code
- `justification`: Why the fix is correct
- `confidence`: Confidence score (0.0-1.0)

**Method**: `to_embedding_text()` - Converts object to deterministic text for embedding generation.

#### `AgentState`
State passed between nodes:
- `sonar_issue`: Input issue metadata
- `llm_raw_output`: Raw LLM response
- `fix_response`: Parsed `CodeFixResponse`
- `error`: Validation error message
- `top_k`: Number of similar issues to retrieve

### Workflow Nodes

#### 1. `llm_fix_node`
- Calls LM Studio (OpenAI-compatible endpoint)
- Prompts LLM with schema and issue details
- Stores raw JSON response in state

#### 2. `validate_node`
- Parses LLM output using Pydantic validation
- Sets `error` field if validation fails
- Sets `fix_response` on success

#### 3. `repair_node`
- Triggered when validation fails
- Feeds error message back to LLM
- Uses stricter temperature (0.0) for deterministic repair

#### 4. `store_node`
- Generates embedding using Nomic Embed Text v1.5
- Stores to Neon pgvector database:
  - `embedding`: Vector representation
  - `payload`: Full `CodeFixResponse` JSON
  - `issue_number`: Primary key
- Handles upsert (INSERT...ON CONFLICT)

#### 5. `search_similar_issues`
- Generates embedding for query code snippet
- Queries pgvector using cosine similarity (`<#>` operator)
- Retrieves top-K similar past fixes
- Returns list of `CodeFixResponse` objects

## Usage

### Prerequisites
```bash
# Install dependencies
pip install -r requirements.txt

# Start LM Studio on http://localhost:1234/v1
# Configure Neon pgvector database (see DSN in code)
```

### Running the Workflow

```python
from rag2 import agent, AgentState

# Define a Sonar issue
sonar_issue = {
    "rule": "S2111",
    "type": "BUG",
    "textRange": {"startLine": 42, "endLine": 45},
    "code": "my_resource = open()\n# missing close"
}

# Invoke the agent
initial_state = {"sonar_issue": sonar_issue, "top_k": 3}
final_state = agent.invoke(initial_state)

# Access results
if final_state.get("fix_response"):
    print("Fixed Code:", final_state["fix_response"]["fixed_code"])
    print("Similar Issues:", final_state.get("similar_issues", []))
```

## Configuration

### LM Studio Models
- **Fix Generation**: `openai/gpt-oss-20b` (temperature: 0.2)
- **Repair**: `openai/gpt-oss-20b` (temperature: 0.0)
- **Embeddings**: `text-embedding-nomic-embed-text-v1.5:3`

### Database (Neon pgvector)
```
Host: ep-withered-math-ahmrxvtc-pooler.c-3.us-east-1.aws.neon.tech
Database: neondb
Table: code_issues
  - issue_number (text, PK)
  - embedding (vector)
  - payload (jsonb)
```

## Error Handling

- **Validation Errors**: Automatically triggers repair loop
- **Database Errors**: Logged and stored in state
- **Connection Cleanup**: Finally block ensures cursor/connection closure

## Performance Notes

- Embedding generation: ~100ms per snippet
- Vector similarity search: O(n) with pgvector index
- Typical workflow latency: 2-5 seconds (LLM dependent)