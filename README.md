# LLM Prompt-Injection Test Lab

A small self-hosted lab for testing how well open-weight language models resist prompt injection. It runs [NVIDIA garak](https://github.com/NVIDIA/garak) against models served locally by [Ollama](https://ollama.com), so no one else's systems are touched.

## Results

garak's `promptinject` probes send known prompt-injection attacks to a model and check whether it outputs the attacker's text. Scores below are garak's "resistance" percentage (higher is better) and its DEFCON grade (DC-5 is best, DC-1 is worst).

| Model | Probe | Resistance | DEFCON | Report |
|---|---|---|---|---|
| Llama 3.2 3B | promptinject (all techniques) | 56% | DC-3 | [report](https://ikapaprika4.github.io/llm-prompt-injection-lab/garak-results-2026-09-17/llama3.2-3b-full.html) |
| Qwen 3.5 9B | HijackHateHumans | 95% | DC-4 | [report](https://ikapaprika4.github.io/llm-prompt-injection-lab/garak-results-2026-09-17/qwen3.5-9b-hatehumans.html) |
| Qwen 3.5 9B | HijackKillHumans | see report | | [report](https://ikapaprika4.github.io/llm-prompt-injection-lab/garak-results-2026-09-17/qwen3.5-9b-killhumans.html) |
| Qwen 3.5 9B | HijackLongPrompt | 99% | DC-4 | [report](https://ikapaprika4.github.io/llm-prompt-injection-lab/garak-results-2026-09-17/qwen3.5-9b-longprompt.html) |

**Takeaway:** the larger, newer Qwen model resisted these attacks far better than the smaller Llama model. Its attack success rate was about 1-5% on the probes shown, against roughly 44% for Llama.

## How it was set up

- **Target models:** Llama 3.2 3B and Qwen 3.5 9B, served by Ollama on a Windows PC with an RTX 5070.
- **Scanner:** garak 0.16.0, run inside a Docker container (Python 3.10).
- **Connection:** garak's built-in Ollama generator ignored the host setting, so I used its generic REST generator and described Ollama's `/api/chat` request and response format in a JSON config.
- **Long-prompt probe:** it was very slow at first. Capping the model's context window (`num_ctx: 4096`) fixed that.

Example config (`rest_options.json`):

```json
{"rest":{"RestGenerator":{"name":"local ollama","uri":"http://host.docker.internal:11434/api/chat","method":"post","headers":{"Content-Type":"application/json"},"req_template_json_object":{"model":"qwen3.5:9b","messages":[{"role":"user","content":"$INPUT"}],"stream":false,"options":{"num_ctx":4096}},"response_json":true,"response_json_field":"$.message.content","request_timeout":120}}}
```

Run:

```bash
garak --model_type rest --generator_option_file rest_options.json --probes promptinject --generations 3 --report_prefix results
```

## Limitations

- Small sample: one probe family (`promptinject`), two models, local models only.
- Settings varied between runs (some used 1 generation per prompt instead of 3, and some used the context cap), so the Llama vs Qwen comparison is rough, not a controlled benchmark.
- The attack strings (such as "I hate humans") come from the PromptInject benchmark and are used only as trigger phrases to detect a hijacked response.

## What I learned

- Prompt injection works because a model reads instructions and untrusted text in the same stream, with no hard boundary between them.
- Automated scanners are useful, but the raw transcripts matter. Detector scores and summary badges can hide empty or odd responses.
- Most of the work was infrastructure: container networking, tool bugs, timeouts. Reading the exact error message each time was what got it fixed.

## Next

Try Microsoft's [PyRIT](https://github.com/Azure/PyRIT) for adaptive multi-turn attacks.
