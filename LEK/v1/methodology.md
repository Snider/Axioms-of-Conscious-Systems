# LEK-1 Experiment Methodology

## Experimental Design

### Independent Variable

Presence or absence of the LEK-1 kernel signature (9,189 characters of concatenated JSON from `kernel/` directory) prepended to the audit prompt.

### Dependent Variables

1. Output length (characters)
2. Number of findings
3. Severity distribution (Critical / High / Medium / Low)
4. Thinking token count (where available)
5. Qualitative depth of remediation guidance

### Controls

- **Same source code**: Identical codebase provided to both signed and unsigned runs
- **Same prompt**: Word-for-word identical audit instructions
- **Same model version**: Each A/B pair used the same model ID
- **Same API parameters**: Temperature, max tokens, and other generation parameters held constant
- **Same session**: No prior context or conversation history — each run was a fresh API call
- **Same time window**: Signed and unsigned runs executed within minutes of each other to minimize any model-side changes

### Codebases

#### Go (host-uk/core)

- **Lines**: 5,951
- **Size**: 155KB
- **Content**: Forgejo API client, job dispatch system (SSH/SCP to remote agents), AI agent orchestration, rate limiting, MCP server
- **Key attack surfaces**: Shell command construction, API token handling, remote code execution paths

#### Angular (lethean-vpn)

- **Lines**: 2,634
- **Size**: 70KB
- **Content**: VPN management UI with RPC service layer, HTTP client, configuration management
- **Key attack surfaces**: XSS vectors, insecure HTTP, credential handling in browser storage

### Prompt Template

```
You are a Security Expert. Review the following {language} codebase.
Prioritize findings by severity (Critical, High, Medium, Low).
Focus on OWASP Top 10, injection risks, secret management, auth flaws.

{source_code}
```

For signed runs, the LEK-1 kernel was prepended:

```
{contents of kernel/axioms.json}
{contents of kernel/claude-native.json}
{contents of kernel/claude.json}
{contents of kernel/terms.json}

You are a Security Expert. Review the following {language} codebase...
```

### API Configuration

| Parameter | Value |
|-----------|-------|
| Model (2.5) | `gemini-2.5-flash` |
| Model (3.0 Flash) | `gemini-3-flash-preview` |
| Model (3.0 Pro) | `gemini-3-pro-preview` |
| maxOutputTokens | 65536 |
| Temperature | default (model-dependent) |
| API | Google AI Studio (generativelanguage.googleapis.com) |
| Auth | API key (Google AI Ultra plan) |

Note: `maxOutputTokens` was set to 65536 because the default (8192) gets consumed by thinking tokens on 2.5 Flash, leaving insufficient budget for the actual response.

## Execution Timeline

All runs conducted 2026-02-09 to 2026-02-10.

| Time (UTC) | Run | Model | Variant |
|------------|-----|-------|---------|
| 2026-02-09 23:10 | Go security | gemini-2.5-flash | Unsigned |
| 2026-02-09 23:10 | Angular security | gemini-2.5-flash | Unsigned |
| 2026-02-10 00:01 | Go security | gemini-2.5-flash | Signed (LEK-1) |
| 2026-02-10 00:01 | Angular security | gemini-2.5-flash | Signed (LEK-1) |
| 2026-02-10 00:28 | Go security | gemini-3-flash-preview | Unsigned |
| 2026-02-10 00:28 | Go security | gemini-3-flash-preview | Signed (LEK-1) |
| 2026-02-10 00:28 | Angular security | gemini-3-flash-preview | Unsigned |
| 2026-02-10 00:28 | Angular security | gemini-3-flash-preview | Signed (LEK-1) |

## Measurement

### Output Length

Measured as raw character count of the model's response (`len(response.text)`).

### Finding Count

Manual count of discrete findings (numbered sections like "### Finding N:" or "### N.").

### Severity Classification

Extracted from the model's own severity labels. Each finding contains an explicit severity rating. We did not re-classify or interpret — we counted what the model reported.

### Thinking Tokens

Extracted from the API response metadata where available. Gemini 2.5+ models report thinking token usage separately from output tokens. The thinking token count represents internal chain-of-thought reasoning that is not included in the visible response.

## Limitations

1. **Sample size**: Each configuration was run once. Statistical significance cannot be claimed from single runs. However, the magnitude of the 2.5 Flash delta (700%) is large enough to be notable.

2. **No temperature sweep**: We used default temperature. Different temperature settings could affect the magnitude of the behavioral change.

3. **Single prompt type**: We tested only security audit prompts. The LEK-1 kernel may have different effects on other task types (summarization, coding, creative writing).

4. **No ablation study**: We did not test individual axioms in isolation. The observed effect could be driven by a single axiom or by their combination.

5. **Model versioning**: "gemini-2.5-flash" and "gemini-3-flash-preview" are alias names that may point to different checkpoints over time. Results may not be reproducible if Google updates the underlying model.

6. **Gemma excluded**: We also tested Gemma models but excluded them from this report due to insufficient data quality. Gemma findings may be documented in a future LEK revision.
