# nullwatch-py

Python SDK for [nullwatch](https://github.com/nullclaw/nullwatch) — observability and hallucination detection for LLM agents.

`nullwatch-py` instruments Python agent code with spans and evals, detects RAG and tool-call hallucinations, and ships results to the nullwatch service for dashboards, CI checks, and regression suites.

## Install

```bash
pip install nullwatch-py
```

With RAG hallucination detection (requires PyTorch + Transformers):

```bash
pip install "nullwatch-py[rag]"
```

Development tools:

```bash
pip install "nullwatch-py[dev]"
```

Also available on [Test PyPI](https://test.pypi.org/project/nullwatch-py/0.1.0/) as an early preview.

## Quick Start

```python
from nullwatch import NullwatchClient
from nullwatch.scorers import RAGHallucinationScorer, ToolCallScorer, ToolCallGroundingScorer

client = NullwatchClient()  # connects to http://127.0.0.1:7710

# Record a span
with client.span("run-123", "llm.call", model="gpt-4o") as span:
    response = call_llm(prompt)
    span.record_openai_usage(response)

# Check RAG answer for hallucinations
rag_scorer = RAGHallucinationScorer()
client.ingest_eval(rag_scorer.score(
    run_id="run-123",
    contexts=docs,
    question=question,
    answer=response.text,
))

# Validate tool call schema
tool_scorer = ToolCallScorer(tools=MY_TOOLS)
client.ingest_eval(tool_scorer.score("run-123", tool_calls=response.tool_calls))

# Check tool call argument values are grounded in context
grounding_scorer = ToolCallGroundingScorer(context=user_message)
client.ingest_eval(grounding_scorer.score("run-123", tool_call=response.tool_calls[0]))
```

## Spans

Spans represent timed work inside a run. The context manager starts a timer on entry, finishes it on exit, captures errors automatically, and ingests the span.

```python
with client.span("run-123", "llm.call", model="gpt-4o") as span:
    response = call_llm(prompt)
    span.input_tokens = response.usage.prompt_tokens
    span.output_tokens = response.usage.completion_tokens
    span.cost_usd = response.usage.total_cost
```

### Provider helpers

Best-effort adapters that extract token counts from provider response objects without importing provider packages at the module level:

```python
with client.span("run-123", "llm.call", model="gpt-4o") as span:
    response = openai_client.chat.completions.create(...)
    span.record_openai_usage(response)   # extracts prompt_tokens, completion_tokens, cost

with client.span("run-123", "llm.call", model="claude-3-5-sonnet") as span:
    response = anthropic_client.messages.create(...)
    span.record_anthropic_usage(response)  # extracts input_tokens, output_tokens
```

Available helpers: `record_openai_usage(response)`, `record_anthropic_usage(response)`, `record_tokens(input_tokens=..., output_tokens=...)`, `record_cost(cost_usd=...)`.

### Decorators

Wrap functions in spans without a context manager:

```python
@client.trace("retriever.search")
def search_docs(run_id: str, query: str) -> list:
    return retriever.search(query)

@client.atrace("llm.call")
async def call_model(run_id: str, prompt: str) -> str:
    return await model.generate(prompt)
```

The `run_id` is extracted from the function's keyword argument automatically. If none is found, a fresh one is generated.

### Buffered mode

Batch span ingestion through `/v1/spans/bulk` — useful for high-throughput pipelines:

```python
with NullwatchClient(buffered=True, flush_at=100) as client:
    for item in dataset:
        with client.span(item.run_id, "process"):
            process(item)
# flushes automatically on context-manager exit
```

## Evals

Evals attach quality signals to a run:

```python
from nullwatch import Eval

client.ingest_eval(Eval(
    run_id="run-123",
    eval_key="helpfulness",
    scorer="llm-judge",
    score=0.94,
    verdict="pass",
    notes="Response was clear and accurate.",
))
```

## RAG Hallucination Detection

Uses [LettuceDetect](https://github.com/KRLabsOrg/LettuceDetect) (`lettucedect-large-modernbert-en-v1`) — a ModernBERT token classifier that identifies which spans of an answer are not supported by the retrieved context.

Requires `pip install "nullwatch-py[rag]"`.

```python
from nullwatch.scorers import RAGHallucinationScorer

scorer = RAGHallucinationScorer()

eval_ = scorer.score(
    run_id="run-123",
    contexts=["The capital of France is Paris. Population is 68 million."],
    question="What is the capital and population of France?",
    answer="The capital is Paris. The population is 80 million.",
)

client.ingest_eval(eval_)

print(eval_.verdict)  # "fail"
print(eval_.notes)    # 'Hallucinated spans detected: "80 million" (conf=0.97)'
```

By default, any detected hallucinated span that covers more than 30% of the answer triggers a `fail`. Adjust with `fail_threshold`:

```python
# Strict: fail if any hallucinated span appears (> 5% of answer)
scorer = RAGHallucinationScorer(fail_threshold=0.05)

# Lenient: only fail if more than 60% of the answer is hallucinated
scorer = RAGHallucinationScorer(fail_threshold=0.60)
```

## Tool-Call Validity

`ToolCallScorer` validates LLM-generated tool calls against a schema. Catches fabricated tool names, misspelled arguments, missing required fields, malformed JSON, enum violations, and wrong argument types. No ML model needed.

Accepts compact nullwatch schema format or the exact OpenAI-style `tools=[...]` JSON schema you send to the model:

```python
from nullwatch.scorers import ToolCallScorer

scorer = ToolCallScorer(tools=[{
    "name": "search_web",
    "parameters": {
        "query": {"type": "string", "required": True},
        "max_results": {"type": "integer", "required": False},
    }
}])

eval_ = scorer.score(
    run_id="run-123",
    tool_call={"name": "search_web", "arguments": {"querY": "zig lang"}},
)

print(eval_.verdict)  # "fail"
print(eval_.notes)    # Unknown argument 'querY' (did you mean: ['query'])?
```

Supports OpenAI and Anthropic tool call formats out of the box:

```python
# OpenAI format (arguments as JSON string)
scorer.score(run_id="run-123", tool_call={
    "type": "function",
    "function": {"name": "search_web", "arguments": '{"query": "zig lang"}'},
})

# Anthropic format (type=tool_use, input instead of arguments)
scorer.score(run_id="run-123", tool_call={
    "type": "tool_use",
    "name": "search_web",
    "input": {"query": "zig lang"},
})

# Batch validation
scorer.score(run_id="run-123", tool_calls=[call1, call2, call3])
```

## Tool-Call Grounding

`ToolCallGroundingScorer` checks whether tool call **argument values** are grounded in the provided context — the semantic complement to `ToolCallScorer`.

`ToolCallScorer` answers: *"Is this call structurally valid?"*
`ToolCallGroundingScorer` answers: *"Did the model invent these argument values?"*

Two backends available:

- **`keyword`** (default) — zero-dependency heuristic, checks argument values against context words and numbers
- **`llm`** — uses an OpenAI-compatible LLM judge (e.g. local Ollama) for deeper semantic reasoning

```python
from nullwatch.scorers import ToolCallGroundingScorer

# Keyword backend (no extra dependencies)
scorer = ToolCallGroundingScorer(
    context="The user wants to search for Python documentation in the nullwatch-py project.",
)

eval_ = scorer.score(
    run_id="run-123",
    tool_call={"name": "search_docs", "arguments": {"query": "Kubernetes cluster deployment"}},
)

print(eval_.verdict)  # "fail"
print(eval_.notes)    # Argument 'query' value 'Kubernetes...' may be hallucinated

# LLM backend (using local Ollama)
scorer = ToolCallGroundingScorer(
    context=user_message,
    backend="llm",
    llm_url="http://localhost:11434/v1",
    llm_model="qwen3:0.6b",
)
```

Operational arguments (paths, URLs, shell commands, `max_results`, `timeout`, etc.) are automatically excluded from grounding checks — the scorer focuses only on content arguments that should be derived from the user's request.

Use both scorers together for full coverage:

```python
schema_eval = ToolCallScorer(tools=tools).score(run_id, tool_call=call)
grounding_eval = ToolCallGroundingScorer(context=context).score(run_id, tool_call=call)

client.ingest_eval(schema_eval)
client.ingest_eval(grounding_eval)
```

## Querying Runs

```python
summary = client.get_run("run-123")
print(summary.span_count, summary.eval_count, summary.verdict)
print(summary.total_cost_usd, summary.total_duration_ms)

# Filter evals and spans
client.list_evals(verdict="fail", eval_key="rag_hallucination")
client.list_evals(run_id="run-123", scorer="schema-validator")
client.list_spans(status="error")
client.list_spans(run_id="run-123", model="gpt-4o")
client.list_runs(verdict="fail", limit=20)
```

## Client API

```python
client = NullwatchClient(
    base_url="http://127.0.0.1:7710",  # or NULLWATCH_URL env var
    api_key=None,                       # or NULLWATCH_API_KEY env var
    timeout=10,
    raise_on_error=True,
    default_source="my-app",
    buffered=False,
    flush_at=100,
    redact=None,  # optional callable (payload: dict) -> dict
)

client.health()
client.capabilities()
client.is_alive()

client.ingest_span(span)
client.ingest_spans([span_a, span_b])
client.ingest_eval(eval_)

client.get_run("run-123")
client.list_runs(verdict="fail", limit=20)
client.list_spans(run_id="run-123", status="error")
client.list_evals(verdict="fail", eval_key="rag_hallucination")

client.flush()   # flush buffered spans
client.close()   # flush + release resources
```

### Redaction

Remove secrets and sensitive fields before ingest:

```python
client = NullwatchClient(
    redact=lambda payload: {**payload, "model": "[REDACTED]"} if "model" in payload else payload
)
```

## Testing Utilities

Test helpers let you assert telemetry behaviour without running a real nullwatch server:

```python
from nullwatch import NullwatchClient, MemoryTransport
from nullwatch.testing import AssertionError as NWAssertionError

transport = MemoryTransport()
client = NullwatchClient(transport=transport)

with client.span("run-123", "tool.execute", tool_name="search"):
    pass

# Assert a span was recorded
transport.assert_span_recorded(operation="tool.execute", tool_name="search")

# Assert no evals failed
transport.assert_no_failed_evals()

# Assert a specific eval was recorded
transport.assert_eval_recorded(eval_key="rag_hallucination", verdict="pass")

# Access raw recorded data
print(transport.spans)
print(transport.evals)
```

## CLI

```bash
# Check service connectivity
nullwatch-py ping

# Ingest a span from a JSON file
nullwatch-py ingest-span span.json

# Ingest an eval from a JSON file
nullwatch-py ingest-eval eval.json

# Print a run summary
nullwatch-py run run-123
```

Environment variables `NULLWATCH_URL` and `NULLWATCH_API_KEY` are respected by all commands.

## Architecture

```
nullwatch/
  __init__.py          Public exports
  client.py            NullwatchClient — spans, evals, queries, decorators
  models.py            Span, Eval, RunSummary, HallucinationResult
  testing.py           MemoryTransport and assertion helpers
  cli.py               Debugging CLI
  scorers/
    base.py            BaseScorer protocol
    rag_hallucination.py   RAGHallucinationScorer (LettuceDetect)
    tool_call.py           ToolCallScorer (schema validation)
    tool_call_grounding.py ToolCallGroundingScorer (semantic grounding)
```

The core client has zero required dependencies (stdlib only). Scorer dependencies are optional and isolated behind extras. The transport layer is replaceable for testing.

## API Compatibility

Maps directly to the nullwatch service contract:

```
POST /v1/spans        ingest one span
POST /v1/spans/bulk   ingest many spans (buffered mode)
POST /v1/evals        ingest one eval
GET  /v1/runs         list runs
GET  /v1/runs/{id}    get run summary
GET  /v1/spans        list spans with filters
GET  /v1/evals        list evals with filters
GET  /v1/capabilities server capabilities
GET  /health          health check
```

## Development

```bash
pip install -e ".[dev]"
make lint    # ruff check
make test    # pytest (161 tests, no external services required)
```

## License

MIT
