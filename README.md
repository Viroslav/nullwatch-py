# nullwatch-py

Python SDK for [nullwatch](https://github.com/nullclaw/nullwatch).

The core client has no required dependencies — it uses stdlib only. Hallucination detection requires `lettucedetect`.

## Install

```bash
pip install nullwatch-py
pip install "nullwatch-py[rag]"  # with hallucination detection
```

## Usage

```python
from nullwatch import NullwatchClient
from nullwatch.scorers import RAGHallucinationScorer, ToolCallScorer

client = NullwatchClient()  # connects to http://127.0.0.1:7710
```

### Spans

```python
# context manager — auto-finishes and ingests on exit
with client.span("run-123", "llm.call", model="gpt-4o") as s:
    response = call_llm(prompt)
    s.input_tokens = response.usage.prompt_tokens
    s.output_tokens = response.usage.completion_tokens
    s.cost_usd = response.usage.total_cost
```

### Evals

```python
from nullwatch import Eval

client.ingest_eval(Eval(
    run_id="run-123",
    eval_key="helpfulness",
    scorer="llm-judge",
    score=0.94,
    verdict="pass",
))
```

### RAG hallucination detection

Uses [LettuceDetect](https://github.com/KRLabsOrg/LettuceDetect) (`lettucedect-large-modernbert-en-v1`) — a ModernBERT token classifier that identifies which spans of an answer are not supported by the retrieved context.

```python
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

### Tool-call validity

Validates LLM-generated tool calls against a schema — catches fabricated tool names, misspelled arguments, and wrong types. No ML model needed.

You can pass either:

- the compact `nullwatch-py` schema format shown below, or
- the same OpenAI-style `tools=[...]` JSON schema you send to the model

```python
scorer = ToolCallScorer(tools=[{
    "name": "search_web",
    "parameters": {
        "query": {"type": "string", "required": True},
    }
}])

eval_ = scorer.score(
    run_id="run-123",
    tool_call={"name": "search_web", "arguments": {"querY": "zig lang"}},
)

print(eval_.verdict)  # "fail"
print(eval_.notes)    # Unknown argument 'querY' (did you mean: ['query'])?
```

### Querying runs

```python
summary = client.get_run("run-123")
print(summary.span_count, summary.eval_count, summary.verdict)

client.list_evals(verdict="fail", eval_key="rag_hallucination")
client.list_spans(status="error")
```

## License

MIT
