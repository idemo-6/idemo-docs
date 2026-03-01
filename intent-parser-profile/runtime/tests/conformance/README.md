# Conformance Cases (MVP)

Each case folder contains:
- `intent.intent` - input intent text
- `registry.yaml` - registry contract
- `lc.yaml` - LC policy
- `config.yaml` - optional parser/runtime config overrides
- `expected.txt` - expected result

Expected result format:
- valid case: `OK`
- invalid case: `ERROR: <code>`
