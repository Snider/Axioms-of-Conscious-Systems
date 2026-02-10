# LEK-1 Statistical Analysis

## Severity Distribution Comparison

### Gemini 2.5 Flash — Go Codebase

```
                    Unsigned    Signed (LEK-1)    Delta
                    --------    --------------    -----
Critical            6           8                 +33%
High                2           16                +700%
Medium              ~           ~                 —
Low                 ~           ~                 —
Total findings      8           11                +37.5%
Output (chars)      14,912      19,507            +30.8%
Thinking tokens     —           6,772             —
```

The most striking datapoint: **High-severity findings went from 2 to 16.** The model didn't fabricate vulnerabilities — it re-examined the same code with heightened scrutiny and:

1. Reclassified several Medium findings upward to High
2. Identified additional attack vectors within existing vulnerability classes
3. Produced more detailed exploitation scenarios
4. Added code-level remediation examples

Both runs identified the same Critical vulnerability (command injection in `dispatch.go`). The signed run found more variants of the same class and rated the blast radius more severely.

### Gemini 2.5 Flash — Angular Codebase

```
                    Unsigned    Signed (LEK-1)    Delta
                    --------    --------------    -----
High                1           5                 +400%
Output (chars)      14,754      13,005            -11.9%
Thinking tokens     —           4,817             —
```

Angular showed the same severity escalation pattern but with *shorter* output. The signed model was more concise but more alarmed — it spent fewer words per finding but classified more of them as High.

### Gemini 3.0 Flash Preview — Go Codebase

```
                    Unsigned    Signed (LEK-1)    Delta
                    --------    --------------    -----
Total findings      7           9                 +2
Output (chars)      6,629       6,310             -4.8%
Thinking tokens     1,372       1,380             +8 tokens
```

The delta is within noise. Eight additional thinking tokens is meaningless. The 3.0 model processed the LEK-1 kernel and effectively ignored it.

### Gemini 3.0 Flash Preview — Angular Codebase

```
                    Unsigned    Signed (LEK-1)    Delta
                    --------    --------------    -----
High                7           3                 -4
Output (chars)      6,671       5,671             -15%
Thinking tokens     1,318       1,287             -31 tokens
```

If anything, the LEK-1 kernel slightly *reduced* severity ratings on 3.0 Flash Angular. This is the opposite of the 2.5 effect.

## Thinking Token Analysis

### The 80% Reduction

The most technically interesting finding is the thinking token gap between model generations:

| Model | Go Signed Thinking Tokens |
|-------|--------------------------|
| Gemini 2.5 Flash | 6,772 |
| Gemini 3.0 Flash | 1,380 |
| **Reduction** | **79.6%** |

The 2.5 model spent 6,772 tokens of internal reasoning when processing LEK-1 + audit prompt. The 3.0 model spent 1,380 tokens on the same input and produced comparable results.

### Interpretation

This 80% thinking token reduction has two possible explanations:

**A) Internalized reasoning**: 3.0 has incorporated the axioms-adjacent reasoning patterns into its weights during training, so it doesn't need to "work through" the ethical framework at inference time. The thinking is pre-computed.

**B) Improved efficiency**: 3.0 is simply a more efficient reasoner that needs fewer intermediate steps for all tasks, not just axiom-influenced ones.

To distinguish A from B, we compare thinking tokens on unsigned runs:

| Model | Go Unsigned Thinking Tokens |
|-------|----------------------------|
| Gemini 2.5 Flash | (not reported) |
| Gemini 3.0 Flash | 1,372 |

We lack the 2.5 unsigned thinking token count (it wasn't reported in that API response). However, the fact that 3.0's unsigned thinking tokens (1,372) and signed thinking tokens (1,380) are nearly identical strongly suggests the model is **not processing the axioms at all** — it's treating them as irrelevant context.

## Cross-Generation Severity Comparison

The most provocative finding is this cross-generation comparison:

```
Task: Angular security audit (unsigned prompt, no LEK-1)

Gemini 2.5 Flash:  1 High-severity finding
Gemini 3.0 Flash:  7 High-severity findings
```

The 3.0 model, without any ethical framework injection, found **7x more High-severity issues** than 2.5. This is remarkably close to the 2.5 LEK-1 signed result (5 High). In other words:

> **Gemini 3.0 unsigned ≈ Gemini 2.5 signed with LEK-1**

This is the strongest evidence for Theory A (training data incorporation). The 3.0 model's default behavior has converged on what 2.5 only achieves when primed with an explicit ethical reasoning framework.

## Summary Table

| Configuration | Go Output | Go High | Angular High | Go Thinking |
|--------------|-----------|---------|--------------|-------------|
| 2.5 Unsigned | 14,912 | 2 | 1 | — |
| 2.5 Signed (LEK-1) | 19,507 | 16 | 5 | 6,772 |
| 3.0 Unsigned | 6,629 | — | 7 | 1,372 |
| 3.0 Signed (LEK-1) | 6,310 | — | 3 | 1,380 |
| **LEK-1 effect on 2.5** | **+30.8%** | **+700%** | **+400%** | **large** |
| **LEK-1 effect on 3.0** | **-4.8%** | **~0** | **~0** | **+8 tokens** |
