---
name: codex-desktop-connect-triage
description: 'A three-layer triage for Codex Desktop (the Codex core inside /Applications/ChatGPT.app) when it will not connect, spins forever, shows Reconnecting, or silently drops messages. Trigger when the user says Codex is unusable, the desktop app keeps reconnecting, they are logged in but cannot send, or they see model does not exist, 404 or 401. The layer order is fixed and must not be skipped: (1) network - does the proxy reach the GUI launchd domain, is that port really listening, is DNS hijacked by Tailscale or poisoned, transient blip or persistent fault; (2) credentials - is auth.json a real ChatGPT login or a placeholder API key, and why a logged-in web shell does not mean the core holds credentials; (3) client and config - model and model_provider in config.toml, how the plan gates which models respond, and switching providers from a GUI with no -p flag. Includes status code criteria, symptom tables, remote control pairing checks and quota window detection.'
license: MIT
compatibility: 'macOS with the ChatGPT desktop app installed (the Codex binary lives under /Applications/ChatGPT.app), plus launchctl, lsof, curl, sqlite3 and python3 on PATH. Runs in Claude Code, Codex, Cursor and any SKILL.md-compatible agent with shell access on the local Mac.'
metadata:
  author: zihao0639-netizen
  version: '1.0.0'
---

# Codex Desktop will not connect — three-layer triage

## 🔴 Core rule: symptoms cannot attribute a cause

**"Cannot connect", "401" and "404" look exactly the same in the UI** — every one of them presents as "spinning / Reconnecting / message sent and nothing happens".

UI symptoms **cannot** distinguish the causes. **You must go into the logs and read the status code.**

```bash
ls -t ~/Library/Logs/com.openai.codex/$(date +%Y/%m/%d)/*t0*.log | head -1
```

**Fixed order, no skipping layers: network → credentials and login state → client and config.**

| What the user says | Possible real causes (at least three each) |
|---|---|
| "It just spins forever" | Proxy port is empty / auth.json holds a fake key / no model permission |
| "It says Reconnecting" | Proxy variables never injected into the GUI domain / credentials expired / backend 5-hour window exhausted |
| "I send a message and nothing happens" | Network dropped silently / thread quota exhausted / wrong `stream` parameter producing a fake 400 |
| "But I am clearly logged in" | Web login ≠ the core having credentials (two completely independent systems) |
| "I can see it in the model list" | The list is the full visible set; actual calls get filtered again by plan |

---

# Layer 1: Network

## 1.1 Did the proxy reach the GUI domain?

```bash
curl -sS -o /dev/null -w "%{http_code}\n" --max-time 6 https://chatgpt.com               # expect 000/timeout (direct connection blocked)
curl -sS -x http://127.0.0.1:<port> -o /dev/null -w "%{http_code}\n" https://chatgpt.com  # expect 403 (proxy works)
launchctl print gui/$(id -u) | grep -E "PROXY"    # ⚠️ must have a value; use print, not getenv (getenv may read the wrong domain)
```

The Codex core (Rust binary at `/Applications/ChatGPT.app/Contents/Resources/codex`)
**only honours the `HTTPS_PROXY` / `HTTP_PROXY` environment variables; it does not read the macOS system proxy**.
Fix it by injecting them into the GUI domain with a LaunchAgent — **the app must be restarted after any change**.

> 🔴 **Field correction (this used to be written backwards)**: the GUI domain **must** contain `HTTP_PROXY` / `HTTPS_PROXY`.
> **But having a value does not mean it is correct**: verify one more step — **something must actually be listening on that port**:
> ```bash
> lsof -nP -iTCP:<port from the variable> -sTCP:LISTEN   # empty = dead port, pin it down immediately
> ```
> **This was the true culprit in one real incident**: the variable pointed at `127.0.0.1:<port>` while the proxy app was not running at all and nothing was bound to that port
> ⇒ every GUI app silently lost its network, the UI merely looked "stuck", and nothing on screen suggested a network problem.
>
> **Next time, doing this one thing alone buys you ten minutes.**

## 1.2 Is DNS taken over by other software (Tailscale and friends)?

**Symptoms**: the app reports `sa_server_request_failed ... net::ERR_CONNECTION_TIMED_OUT`
+ `[WindowContext] Thread catalog sync failed`, yet **the proxy tests all pass** and the Codex core has ESTABLISHED connections
⇒ do not rush into editing config; characterise the problem first.

```bash
tailscale debug prefs 2>/dev/null | grep -E "CorpDNS|ExitNodeID|RouteAll"
scutil --dns 2>/dev/null | grep -E "nameserver\[0\]|if_index" | head -6
```

- `CorpDNS: true` + `nameserver[0]` inside the Tailscale `100.x.y.z` range + `if_index` on a utun interface
  ⇒ **Tailscale MagicDNS is taking over resolution**
- **Turn it off (keep the connection, stop only DNS)**: `tailscale set --accept-dns=false` ← verified exit 0, takes effect immediately;
  system DNS falls back to the router automatically (LAN gateway / link-local address such as `fe80::…%en0`)
- ⚠️ When you shut down, go home, or do not need remote access, it **should** be off; otherwise it forwards all of your resolution

## 1.3 The signature of DNS poisoning (recognisable at a glance)

```bash
for h in chatgpt.com www.google.com www.baidu.com; do
  printf "  %-16s → %s\n" "$h" "$(dscacheutil -q host -a name $h 2>/dev/null | grep ip_address | head -1 | awk '{print $2}')"
done
```

- 🔴 **`google.com` / `chatgpt.com` resolving to `31.13.x.x` / `157.240.x.x` / `69.171.x.x` (Meta ranges)**
  or something like `243.185.x.x` = classic poisoning** (`baidu.com` still resolves fine ⇒ resolution is poisoned, not broken)
- ⚠️ **But poisoning alone does not explain "timeouts even through the proxy"** — **when traffic goes through the proxy the hostname is resolved at the proxy end, not by local DNS**
  ⇒ the real harm of poisoning is to **direct traffic**, and to **PAC scripts that call `dnsResolve` and then get the wrong IP**

## 1.4 Actually run the PAC in node and see whether it routes the target domain through the proxy

More reliable than eyeballing a PAC file (they are routinely hundreds of KB).

```js
// polyfill first: shExpMatch / isPlainHostName / dnsResolve / isInNet / isResolvable / myIpAddress / alert
// implement dnsResolve synchronously with execSync('dscacheutil -q host -a name ' + h)
const FindProxyForURL = new Function(pacSource + '\nreturn FindProxyForURL;')();
console.log(FindProxyForURL('https://chatgpt.com/', 'chatgpt.com'));   // expect PROXY 127.0.0.1:<port>
```

- ⛔ **`${` must never appear in the script** — the shell will eat it and you get `Bad substitution`
  Write the regex character class as `/[.+^$(){}|[\]\\]/g` (keep `$` and `{}` split apart)
- ✅ Write it to a `.js` file with a file-writing tool and run `node` on it; do not use a heredoc

## 1.5 The log filename carries the [process start time] — a substitute for `ps -o lstart` in a sandbox

```
codex-desktop-<uuid>-<PID>-t0-i1-005502-0.log
                                  └─ HHMMSS = when that process started
```

⇒ In a sandbox `ps -p <pid> -o lstart=` often returns empty; use the timestamp in the filename instead to compare "which started first, the app or the proxy".

## 1.6 Decide "persistent failure" vs "brief blip" (this decides whether to touch config at all)

```bash
D=~/Library/Logs/com.openai.codex/$(date +%Y/%m/%d)
grep -h "TIMED_OUT" $D/*.log | grep -oE "^[0-9T:-]+" | sort | sed -n '1p;$p'   # first / last
grep -hc "TIMED_OUT" $(date -v-1d +~/Library/Logs/com.openai.codex/%Y/%m/%d)/*.log  # how many yesterday
```

- **Failure window of only a few minutes, zero timeouts afterwards, 0 yesterday ⇒ transient line drop** (proxy switching nodes / reconnecting), **not a config error**;
  do not go editing `config.toml`
- Criterion: compare "last timeout timestamp" against "current time", then look at the **timeout count in the newest process log** (= 0 means it already recovered)

## 1.7 Aside: logs flooded by a directory that no longer exists — locating and fixing it

`[git-init-watcher] Failed to watch workspace root for git init cwd="<a path that no longer exists>"` (can reach thousands of lines)
= Codex has the **wrong project root path** (for example a stray space typed into the directory name).

**Locate**: recursively search `~/.codex/.codex-global-state.json` for that path string; hits usually come in a cluster:

```
/local-projects/<projectId>/rootPaths                             ← 🎯 the root cause (list of project root paths)
/thread-writable-roots/<threadId>[]                               ← thread inheritance (several places)
/electron-persisted-atom-state/heartbeat-thread-permissions-by-id/<tid>/sandboxPolicy/writableRoots[]  ← several places
```

⇒ **Fixing only the `rootPaths` entry stops the bleeding, but clearing every hit is the clean fix.**

**Fix (the order matters)**:

1. 🔴 **First confirm the app has exited** (`pgrep -f "/Applications/ChatGPT.app"`)
   — otherwise the app overwrites your edit with the **old in-memory state** on quit, and the work is lost
2. Back up → **filter out** that bad path from the three structures above → write it back
3. ⚠️ **Keep it compact JSON with no newlines**: `json.dumps(d, ensure_ascii=False, separators=(',',':'))`
   — this is the native Electron atom state format (the original file measured 0 newlines)
4. Restart the app and verify the path appears **0 times** in the new log
5. **Alternative**: have the user remove the folder from that project's folder list in the app (zero risk, but it does not clear the copies inherited by historical threads)

⚠️ `len(open(P).read())` gives a **character** count, not a byte count; with CJK content the two diverge a lot (a measured gap of nearly 100,000),
so do not conclude from it that "the file got corrupted".

---

# Layer 2: Credentials and login state

```bash
cat ~/.codex/auth.json
```

| auth.json content | Verdict |
|---|---|
| `{"auth_mode":"chatgpt","tokens":{id_token,access_token,refresh_token,account_id}}` | ✅ Normal |
| `{"auth_mode":"apikey","OPENAI_API_KEY":"..."}` with **no tokens** | ❌ Running in API-key mode |
| The key is a 9-digit placeholder such as `123456789` | ❌ Fake, guaranteed 401 |

Verify:

```bash
env -i HOME="$HOME" PATH="/usr/bin:/bin" \
  HTTP_PROXY=http://127.0.0.1:<port> HTTPS_PROXY=http://127.0.0.1:<port> \
  /Applications/ChatGPT.app/Contents/Resources/codex login status
# expected output: Logged in using ChatGPT
```

## 🔴 Key insight: a web login ≠ credentials the Codex core can use

- The Electron shell logs in via Chromium cookies (`~/Library/Application Support/Codex/Default`)
- The Codex core reads `~/.codex/auth.json`
- **Two completely independent systems** ⇒ the UI showing "logged in, avatar and all" does **not** mean the core has credentials
- Log criterion: `desktop_fetch_auth_401 hadToken=false skipRetryReason=no_token_attached`

**Fix**: **rename** the bad auth.json (do not delete it, keep the backup), restart the app —
the log will show `method=account/login/start`, the app jumps to the login screen by itself, and the user signs in once with their ChatGPT account.

---

# Layer 3: Client and config

## 3.1 Model permissions: the plan decides which models "can be called"

**Symptom**: the network works and the account is logged in, but you get

```
404 Not Found: The model `gpt-5.5` does not exist or you do not have access to it
url: https://chatgpt.com/backend-api/codex/responses
```

**"Visible in the list" ≠ "callable"** — the list endpoint returns the full set visible to the client;
at call time the backend filters again by **subscription plan**.

### Query the models available to the account

```bash
cd ~/.codex
TOKEN=$(python3 -c "import json;print(json.load(open('auth.json'))['tokens']['access_token'])")
ACCT=$(python3 -c "import json;print(json.load(open('auth.json'))['tokens']['account_id'])")
curl -sS -x http://127.0.0.1:<port> \
  -H "Authorization: Bearer $TOKEN" -H "chatgpt-account-id: $ACCT" \
  "https://chatgpt.com/backend-api/codex/models?client_version=<version>" \
  | python3 -c "import json,sys;[print(m['slug']) for m in json.load(sys.stdin)['models']]"
```

⚠️ **`client_version` is a required parameter**; omit it and you get 400 `Field required`.
Different version numbers even return different lists ⇒ use the app's own version
(`BROWSER_USE_CODEX_APP_VERSION` in `~/.codex/config.toml`).

### Test each model for real (the only trustworthy confirmation)

```bash
curl -sS -N -x http://127.0.0.1:<port> \
  -H "Authorization: Bearer $TOKEN" -H "chatgpt-account-id: $ACCT" \
  -H "Content-Type: application/json" -H "originator: codex_cli_rs" -H "version: <v>" \
  -d "{\"model\":\"$m\",\"instructions\":\"be brief\",\"input\":[{\"type\":\"message\",\"role\":\"user\",\"content\":[{\"type\":\"input_text\",\"text\":\"say ok\"}]}],\"stream\":true,\"store\":false}" \
  -o /tmp/r.json -w "%{http_code}\n" \
  "https://chatgpt.com/backend-api/codex/responses"
```

⚠️ **`"stream": true` is mandatory**. Passing `false` only earns you
`400 {"detail":"Stream must be set to true"}` — **which does not mean the model is usable**
(format validation runs before model validation). **This step is the most misread one**: people take the 400 as "the model works" or "the model is broken", when in fact only parameter validation was exercised.

| Response | Meaning |
|---|---|
| `200` + SSE `data:` stream | ✅ Usable |
| `404 model_not_found` | ❌ This account has no access to that model |
| `400 not supported when using Codex with a ChatGPT account` | ❌ That model is not open to ChatGPT accounts |

### Fix

Change the top-level `model = "..."` in `~/.codex/config.toml` to the one that returned 200, then **restart the app**.

**Measured record (a ChatGPT account without Plus)**:

| Model | Available |
|---|---|
| `gpt-5.6-luna` / `gpt-reserve` / `gpt-5.6-terra` | ✅ 200 |
| `gpt-5.5` | ❌ 404 (paid-tier model) |
| `gpt-6-luna` | ❌ 400 (not open to ChatGPT accounts) |

## 3.2 Switching to another provider (DeepSeek, for example)

**`model_provider` decides which channel is used; it does not conflict with `auth_mode=chatgpt` — both can coexist and you switch as needed.**

```bash
codex exec --sandbox read-only --skip-git-repo-check \
  -c model_provider=deepseek -c model=deepseek-flash "..."
```

⚠️ `--skip-git-repo-check` is required, otherwise you get `Not inside a trusted directory`.
⚠️ Two non-fatal noise lines appear, **which do not affect the call**:
- `ERROR failed to refresh available models: missing field 'models'`
  — DeepSeek `/models` returns `{object,data}` while Codex expects `{models}`; the formats are incompatible
- `warning Model metadata for <model> not found. Defaulting to fallback metadata`

### 🔴 How to switch channels on the desktop (GUI) — there is no `-p`, **only the top level of config.toml works**

**Conclusion (settled by testing)**: the desktop model dropdown **really does not** contain custom-provider models.
Testing the app-server `model/list` (the endpoint the GUI renders that dropdown from) returns **only 11 GPT models**:
`gpt-6-astra / 5.6-sol / 5.6-terra / 5.6-luna / daybreak-blue / daybreak-red / 5.5 / 5.4 / 5.4-mini / 5.2 / codex-auto-review`.

⇒ To run DeepSeek / Kimi on the desktop, **the only route is the two top-level lines in `config.toml`**:

```toml
model = "deepseek-flash"
model_provider = "deepseek"
```

After the edit you **must restart the ChatGPT desktop app** (config is read once at process start).

#### How to tell at a glance which channel the desktop is on

**Check whether the top level of `config.toml` has a `model_provider`**:
- `model_provider = "deepseek"` present ⇒ desktop messages go to DeepSeek
- Absent (or `= "openai"`) ⇒ GPT, **which needs the proxy**

#### ⚠️ One trap that is very easy to misread

Seeing a batch of threads in the `threads` table with `source='vscode'` + `provider=deepseek`,
**do not** conclude "the desktop dropdown lets me pick DeepSeek". **It is decided by the top-level config.** The evidence chain:

| Observation | Meaning |
|---|---|
| Threads with `source='vscode'` = started from the desktop GUI (CLI shows as `exec`) | This column **tells you who started it** |
| But it does not prove "the GUI can pick a provider" | The top-level config is written into those threads too |
| **Decisive evidence: dig through `config.toml.bak-<YYYYMMDD>` and read the top level of that moment** | Example: one backup had `model = "deepseek-flash"` + `model_provider = "deepseek"` at the top ⇒ at that time the GUI obviously ran everything on DeepSeek |
| Spotting the turning point | After the top level was reverted to `gpt-5.5`, GUI threads immediately all became openai ⇒ the timestamps line up |

**So the first step when triaging this class of problem is `ls -la ~/.codex/config.toml*` to find historical backups and compare the top-level fields.**

#### A ready-made helper (you can build your own)

```bash
~/bin/codex-mode.sh              # show the current channel plus options
~/bin/codex-mode.sh deepseek     # switch to DeepSeek (direct connection from mainland China)
~/bin/codex-mode.sh gpt          # switch back to GPT (needs the proxy)
```

⚠️ The top level is **global**; the desktop app and the CLI default change together. To switch temporarily in the CLI without touching config: `codex -p <name> "..."`.
⚠️ A profile must live in its own file `~/.codex/<name>.config.toml`; **newer Codex builds do not accept inline `[profiles.xxx]` sections**.
⚠️ `model_provider = "kimi"` currently has **no matching `[model_providers.kimi]` definition in config.toml**
⇒ switching to it reports provider not found; add the definition first.
⚠️ Starting app-server bare and asking `model/list` **returns only 11 GPT models**
(gpt-6-astra / 5.6-sol / 5.6-terra / 5.6-luna / daybreak×2 / 5.5 / 5.4 / 5.4-mini / 5.2 / codex-auto-review).
**Do not conclude from that "the desktop cannot select DeepSeek"** — the custom-provider route is not exposed by this endpoint.

**So there are three ways to switch channels on the desktop, in priority order**:

1. Look in the **model dropdown** in the UI (if it was used before ⇒ the entry is there)
2. Add `model = "..."` + `model_provider = "..."` at the **top level** of `config.toml` (global, the CLI follows too), then **restart the desktop app**
3. `codex -p <profile name> "..."` — CLI only, the GUI cannot use it

⚠️ After the desktop switches to DeepSeek, **plugins / MCP degrade**: `browser` / `documents` / `pdf` /
`spreadsheets` / `presentations` all need to reach `chatgpt.com`, so the moment the proxy drops they throw errors (without affecting the main flow).
⚠️ **Current top level of config.toml** (read these two lines first when triaging): which model the top-level `model` points to and whether `model_provider` exists — if it does not, the default is openai.
⚠️ **One `codex exec` costs ~24k tokens** (system prompt + skills dominate) ⇒ do not use it for very short tasks.

---

# Appendix A: Phone remote control — "cannot pair / shows disconnected"

Scenario: Remote in the ChatGPT mobile app wants to control this Mac; the UI is stuck on "Connect this device to this Mac",
or pairing succeeded but the phone shows "disconnected".

## A.1 First check whether the Mac side is paired at all (decides everything in one look)

```python
import sqlite3, os
con = sqlite3.connect('file:%s?mode=ro' % os.path.expanduser('~/.codex/state_5.sqlite'), uri=True)
list(con.execute("SELECT * FROM remote_control_enrollments"))
```

- **0 rows** = never paired successfully ⇒ go to A.2
- **1 row** with `remote_control_enabled = 1` = **the Mac side is already set up** ⇒ the problem is on the phone; stop poking around on the Mac

Key fields: `server_id` / `environment_id` / `websocket_url` (the value is
`wss://chatgpt.com/backend-api/wham/remote/control/server`).

## A.2 Is the channel alive on the Mac side?

```bash
# search the app-server log for remote control
python3 -c "
import sqlite3,os
con=sqlite3.connect('file:%s?mode=ro'%os.path.expanduser('~/.codex/logs_2.sqlite'),uri=True)
for ts,b in con.execute(\"SELECT ts,feedback_log_body FROM logs WHERE feedback_log_body LIKE '%remote control websocket%' ORDER BY id DESC LIMIT 10\"):
    print(ts,b[:180])"
```

Criterion: a line **`status changed previous_status=Connecting next_status=Connected`** = the channel was established.
If no newer `status changed` record follows, it **never dropped**.

## A.3 Did a job from the phone ever actually reach the Mac?

```sql
SELECT updated_at, model_provider, source, tokens_used, title
FROM threads ORDER BY updated_at DESC LIMIT 10
```

`source` is the fingerprint of **who started it**: `vscode` = Codex inside VS Code; `exec` = command line;
`{"subagent":{"thread_spawn":...}}` = a sub-agent. **Not a single phone-originated thread ⇒ the instruction never arrived.**

## 🔴 A.4 The most misread one: chatting on the phone ≠ remote control working

| What you do on the phone | Which channel it uses | Blocked by Codex quota |
|---|---|---|
| Chat conversation | ChatGPT chat quota | ❌ No |
| **Remote-controlling the Mac** | **Codex quota** | ✅ **Yes** |

The user saying "chat works fine on my phone" **does not** imply "remote control should be fine too".
The two channels are completely independent and **must be verified separately**.

## A.5 ⚠️ "Disconnected" very often just means **the connect button was never pressed**

Real case: the Mac side was all green (enrollment record present, websocket Connected, zero drops),
half an hour of triage later the user discovered it themselves — **they had never pressed the "Connect" button in the phone UI**.

⇒ **Before digging deeper, ask the user to tap that connect button on the phone once.** It costs five seconds and can save half an hour.

---

# Appendix B: How to tell "am I out of quota" (and get the reset time)

**The log says it outright; a plain grep finds it**:

```bash
grep -iE "hit your usage limit|try again at" \
  ~/Library/Logs/com.openai.codex/$(date +%Y/%m/%d)/*t0*.log | tail -3
```

It looks like: `"You've hit your usage limit. ... or try again at 11:27 PM."`
⇒ **whatever follows `try again at` is the window reset time** (local time zone).

**Derive the window start**: `reset time − 5h = the start of this 5-hour window`.

**Corroborating signal: after hitting the limit `threads.tokens_used` is recorded as 0** — an action was sent but nothing was consumed, which means quota blocked it.

> ⚠️ **A conclusion must carry a timestamp.** One triage concluded "no limit hit today" (based on rollout records from a certain interval),
> and the limit was hit a few hours later. **Evidence from one time slot cannot cover the whole day.**

---

# Appendix C: What works inside the sandbox (measured)

| Operation | Inside the sandbox |
|---|---|
| `killall -TERM ChatGPT` | ✅ |
| `open -a "ChatGPT"` | ✅ |
| `osascript -e 'quit app "ChatGPT"'` | ❌ permission violation -10004 |
| Editing `~/.codex/*` config | ✅ |
| Reading `~/Library/Logs/...` | ✅ |

---

# 🔴 Which step is most often mistaken for "solved"

| Misread point | False impression | Reality |
|---|---|---|
| `HTTPS_PROXY` has a value in the GUI domain | "The proxy is configured" | **A value ≠ a live port**; you must verify LISTEN with `lsof` |
| `stream:false` returns 400 | "The model is unavailable / available" | Only parameter validation was exercised, **the model was never tested** |
| The model is visible in the dropdown | "It can be called" | The list is the full visible set; the backend filters again by plan |
| The UI shows "logged in, avatar and all" | "Credentials are fine" | Web cookies and `auth.json` are two separate systems |
| Proxy fully green + ESTABLISHED present | "The network is fine" | DNS may be hijacked / the PAC may be judging wrong |
| Timeouts drop to zero after a few minutes | "Config error, go edit `config.toml`" | Most likely just a transient line drop |
| Chat works on the phone | "Remote control must work too" | Two channels, two separate quotas |
| Changes under `~/.codex/*` do not take effect after a restart | "The edit was wrong" | **The app was not exited before editing, so it overwrote the change with in-memory state on quit** |

---

# Closing self-check list

- [ ] `launchctl print gui/$(id -u) | grep PROXY` returns a value
- [ ] `lsof -nP -iTCP:<proxy port> -sTCP:LISTEN` produces output (the port really is listening)
- [ ] `codex login status` = `Logged in using ChatGPT`
- [ ] `auth.json` contains `tokens.access_token`
- [ ] The `model` in `config.toml` is the one that returned 200 in a real test
- [ ] `lsof -nP -iTCP:<proxy port> | grep -c Codex` > 0
- [ ] No `model_not_found` / `Unauthorized` in the logs
- [ ] The conclusion **carries a timestamp** (never let evidence from one time slot cover the whole day)
