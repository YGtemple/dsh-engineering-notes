# DSH Engineering Notes

Field notes from working on [DeepSeek Harness](https://github.com/deepseek-ai) plugin development, third-party provider integration, and Windows deployment repair.

Each document records a real investigation: the symptom, the hypotheses that were disproven, the evidence that settled it, and the resulting change.

## Contents

| Document | Subject |
| --- | --- |
| [international-chat-400.md](docs/international-chat-400.md) | Diagnosing an empty-body HTTP 400 on a third-party gateway; differential testing across two request variables; upstream patch + PR |
| [windows-install-repair.md](docs/windows-install-repair.md) | Repairing a portable install that broke its own registry, shortcuts, and autostart entries after a directory rename |
| [deployment-notes.md](docs/deployment-notes.md) | Pinning install paths, suppressing auto-update safely, and why "disable the updater" needs evidence rather than guesswork |
| [trae-api-integration.md](docs/trae-api-integration.md) | Replicating a non-OpenAI gateway: auth scheme, client-identity headers, message body shape, and stream-level errors that return HTTP 200 |

## Principles used throughout

1. **Reproduce before theorising.** Every hypothesis in these notes was tested against the live endpoint or the actual filesystem. Several plausible theories were wrong.
2. **Isolate one variable at a time.** The 400 investigation reduced two dimensions (request body shape, header set) to a 2×2 matrix before drawing any conclusion.
3. **Read the binary, not just the docs.** Compiled plugin output and `app.asar` contain the real control flow; comments and READMEs describe intent, not behaviour.
4. **Prefer reversible changes.** Every repair in these notes has a recorded backup and a documented rollback.
5. **Attribute derived work.** Transport-layer code carried over from MIT-licensed upstreams retains its notices.

## Environment

- Windows 11, DSH Desktop 0.2.0-rc.2
- Node.js v24.14.0, pnpm 12.11.2
- Providers: WorkBuddy (CN), WorkBuddy AI (international), Trae

## License

MIT
