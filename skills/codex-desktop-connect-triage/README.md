# codex-desktop-connect-triage

An agent skill that diagnoses why Codex Desktop (the Codex core inside `ChatGPT.app`) will not connect — spins forever, shows "Reconnecting", or swallows sent messages — using a fixed three-layer triage: network, then credentials, then client config.

**Core insight:** "cannot connect", "401" and "404" are indistinguishable in the UI, because all three render as the same spinner. Symptoms cannot attribute a cause, so the status code in `~/Library/Logs/com.openai.codex/` is the only evidence that counts. The layers are ordered for a reason and must not be skipped: proxy variables have to reach the GUI launchd domain *and* something has to be listening on that port; a logged-in web shell does not mean `~/.codex/auth.json` holds credentials; and a model visible in the dropdown is not a model your plan can call.

## Install

```bash
npx skills add zihao0639-netizen/codex-desktop-connect-triage
```

Or copy `skills/codex-desktop-connect-triage/` into your agent's skills directory manually.

Works with Claude Code, Claude.ai, Codex, Cursor and any SKILL.md-compatible agent with shell access.

## What it covers

1. **Symptom table that refuses to guess** — five things users say, at least three real causes behind each, and the one log command that settles it
2. **Layer 1, network** — proxy variables in the GUI domain (`launchctl print gui/$(id -u)`, not `getenv`), the mandatory `lsof` LISTEN check for a dead port, Tailscale MagicDNS takeover, DNS-poisoning fingerprints, running the PAC in `node` instead of eyeballing it, process start time read from log filenames, and persistent fault vs transient blip
3. **Layer 1 aside** — logs flooded by a project root path that no longer exists: locate every hit in `.codex-global-state.json`, quit the app first or your edit gets overwritten from memory, keep the JSON compact
4. **Layer 2, credentials** — `auth.json` verdicts (real ChatGPT tokens / API-key mode / 9-digit placeholder key), `codex login status` under a clean `env -i`, and why the Electron cookie jar and the Rust core are two independent systems
5. **Layer 3, client config** — plan-gated models (`/backend-api/codex/models` with the required `client_version`), the real per-model probe where `"stream": true` is mandatory and a 400 only proves parameter validation ran, and how to read 200 / 404 / 400
6. **Provider switching on a GUI with no `-p`** — top-level `model` + `model_provider` in `config.toml`, restart required, plus the backup-file evidence chain that stops you from misreading historical `source='vscode'` threads
7. **Appendix A, phone remote control** — enrollment row count as the first fork, websocket `status changed` lines, `threads.source` as the origin fingerprint, and the case where the user never pressed Connect
8. **Appendix B, quota** — `try again at` as the reset time, window start derived from it, `tokens_used = 0` as the corroborating signal
9. **Appendix C and closing checklist** — what the sandbox allows (`killall`, `open -a`) and what it does not (`osascript` quit), plus eight checkboxes and a table of the steps most often mistaken for "fixed"

## Requirements

- macOS with the ChatGPT desktop app installed (`/Applications/ChatGPT.app`)
- `launchctl`, `lsof`, `curl`, `sqlite3`, `python3`, `dscacheutil` on PATH
- `tailscale` CLI only for the DNS section
- An agent that can run shell commands on the local Mac

## Security notes

- All commands are read-only diagnostics (`launchctl print`, `lsof`, `scutil`, `sqlite3` opened with `mode=ro`, `codex login status`) plus log greps
- The skill never prints your access token: the model probe reads it into a shell variable and passes it straight to `curl`
- Only two writes are ever proposed, both to paths the user already owns: renaming a broken `auth.json` instead of deleting it, and cleaning a stale project root path out of `.codex-global-state.json` after backing it up and quitting the app first
- No credentials, no third-party endpoints beyond `chatgpt.com`, and no data leaves the machine

## License

MIT — see [LICENSE](LICENSE).
