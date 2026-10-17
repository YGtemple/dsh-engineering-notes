<div align="right">

**[English](README.md)** · [简体中文](README.zh-CN.md)

</div>

# dsh-engineering-notes

Field notes from integrating third-party providers with DeepSeek Harness — transport-level debugging, plugin architecture, and Windows deployment repair.

<sub>Markdown · engineering notes</sub>

---

## What is in here

Notes taken while making external providers work inside DeepSeek Harness. Three areas:

| Area | What it covers |
|---|---|
| Transport debugging | where requests fail between client and provider, and how to tell which layer is at fault |
| Plugin architecture | how the plugin surface is structured, and what a provider integration has to conform to |
| Windows deployment repair | fixing installs that fail on Windows for environment rather than code reasons |

## Why write it down

Integrating a provider looks like configuration work. It is not — most failures are at the transport or environment layer, and the error messages point at the wrong place. Recording what the actual cause was, per symptom, saves the next integration from re-deriving it.

## Format

Each note follows the same shape: symptom, what it looked like, the actual cause, and how to confirm it. That last part matters — a fix you cannot verify is a guess.

## Related

- [agent-debug-notes](https://github.com/YGtemple/agent-debug-notes) — debugging notes from agent sessions
- [dsh-multi-router-intl](https://github.com/YGtemple/dsh-multi-router-intl) — the router this work fed into
- [Profile](https://github.com/YGtemple) — all projects

## License

MIT