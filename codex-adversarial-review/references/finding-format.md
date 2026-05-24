# Finding Format Reference

## Output Schema

The review produces structured output matching this schema:

```json
{
  "verdict": "approve | needs-attention",
  "summary": "string — terse ship/no-ship assessment",
  "findings": [
    {
      "severity": "critical | high | medium | low",
      "title": "string — short description",
      "body": "string — detailed explanation",
      "file": "string — file path",
      "line_start": 1,
      "line_end": 1,
      "confidence": 0.0,
      "recommendation": "string — concrete fix"
    }
  ],
  "next_steps": ["string — actionable follow-up"]
}
```

## Verdict Rules

- `approve` — cannot defend any substantive adversarial finding from the context
- `needs-attention` — any material risk worth blocking on

## Severity Definitions

- **critical** — exploitable from outside, irreversible data loss, security vulnerability
- **high** — causes functional breakage under realistic production conditions
- **medium** — fails in edge cases with a plausible trigger path
- **low** — theoretical risk without clear exploitation path

## Rendered Format

Findings are sorted by severity (critical first). Each finding rendered as:

```
- [severity] title (file:line_start[-line_end])
  body
  Recommendation: recommendation
```

## Finding Requirements

Every finding must include:
- The affected file path
- `line_start` and `line_end` (1-based)
- A confidence score from 0 to 1 (honest — state if inference-based)
- A concrete, actionable recommendation

## Summary

Write the summary like a terse ship/no-ship assessment, not a neutral recap.

## Calibration

- Prefer one strong finding over several weak ones.
- Do not dilute serious issues with filler.
- If the change looks safe, say so directly and return no findings.
