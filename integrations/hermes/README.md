# Membase memory for Hermes Agent

A [Hermes Agent](https://github.com/NousResearch/hermes-agent) memory provider backed by
[Membase](https://membase.com), the hosted memory API. With it selected as `memory.provider`,
Hermes:

- **recalls before each turn**: a search of the user's Membase Memories with a short time budget
  (8 s by default). The passages are injected as context, each tagged with the Memory it came
  from (`- [Project decisions] We settled on Postgres for the ledger.`). If the search is slow or
  fails, the turn goes on without memory. When the key reaches a media Memory, the same recall
  searches its photos and videos and hands the agent what each matching one shows or says, as text;
- **gets four tools**: `membase_search`, `membase_search_media`, `membase_save` and `membase_profile`;
- **saves only when asked**: the model calls `membase_save` when the user asks it to remember
  something. Capturing the user's words from every turn is a setting you turn on yourself
  (`capture_turns`, off by default).

Only one external memory provider is active at a time in Hermes. Hermes' own MEMORY.md / USER.md
keep working next to it and stay Hermes's own, unless the Hermes build has `memory.user_store`
and the profile sets it to `provider` (below): then facts about the user go to the Membase
profile instead of USER.md.

## Install

```bash
hermes plugins install membase-ai                            # from the Hermes plugin catalog
# or straight from the repository:
hermes plugins install membaseai/membase/integrations/hermes

hermes memory setup membase-ai                               # paste the key, pick the options below
```

`hermes plugins install` installs the Python dependency (`membase-ai`) into Hermes'
environment and can set `memory.provider: membase-ai` for you. Start a new session
afterwards.
`hermes memory status` shows the settings in use (no network call, never the key).

### Coming from `unibaseio-membase`

Until 0.5.0 this plugin's id was `unibaseio-membase`. Hermes finds a memory provider by the name
of the folder it is installed in (`$HERMES_HOME/plugins/<id>`), and it has no alias for a renamed
plugin, so the new id cannot answer to the old one. An existing install is not touched by the
rename: `memory.provider: unibaseio-membase` keeps working with the version you have, but that
install receives no further updates (`hermes plugins update` reports its catalog entry as
unavailable and keeps the pin). To move to `membase-ai`:

```bash
hermes plugins install membase-ai --enable   # sets memory.provider: membase-ai (replacing unibaseio-membase)
hermes plugins remove unibaseio-membase      # once a new session recalls as before
```

The key stays where it was (`MEMBASE_API_KEY` in the profile's `.env`). Your settings carry over
too: while `membase-ai.json` does not exist, the plugin reads `unibaseio-membase.json`, and the
first `hermes memory setup membase-ai` writes them, with your changes, to `membase-ai.json`. The old
file is never changed or deleted. Removing `unibaseio-membase` *before* switching clears
`memory.provider`, which leaves the profile with no external memory until you select `membase-ai`.

Get a key in the Membase app at **Connect › Developer keys**. Choose **Read & write** if the
agent should be able to save. A **Read** key can search and read the profile but cannot save. Tick
**profile** on the key if you want `membase_profile` to work.

## Configuration

The key lives in the profile's `$HERMES_HOME/.env`. `hermes memory setup` writes it there, and
the plugin reads it through Hermes' per-profile secret scope, never from a file itself.

| Setting | Where | Default | Meaning |
|---|---|---|---|
| `MEMBASE_API_KEY` | `.env` (secret) | required | The developer key: one person's account |
| `base_url` / `MEMBASE_BASE_URL` | `membase-ai.json` / `.env` | `https://api.app.membase.com` | API endpoint |
| `recall_budget_seconds` | `membase-ai.json` | `8` | How long the per-turn recall may take (0.5–8) |
| `capture_turns` | `membase-ai.json` | `false` | Save the user's own words from every completed turn as a note |
| `auto_recall` | `membase-ai.json` | `true` | Search before each turn |
| `recall_limit` | `membase-ai.json` | `5` | Passages injected per turn |
| `recall_max_chars` | `membase-ai.json` | `4000` | Upper bound on the injected block |
| `recall_mode` / `MEMBASE_RECALL_MODE` | `membase-ai.json` / `.env` | `full` | `fast`: one round of retrieval and no model call; at least 10 passages and 8,000 characters handed to the agent, whose own model judges them |
| `tools` / `MEMBASE_TOOLS` | `membase-ai.json` / `.env` | `on` | `off`: no `membase_*` tools; memory is left to recall before each turn (and Hermes's own memory), so no turn spends a model call deciding whether to search or save |

The settings file is `$HERMES_HOME/membase-ai.json` (mode 0600); it never holds the key. Until it
exists, the `unibaseio-membase.json` of releases before 0.5.0 is read in its place.
When a setting appears in both places, the value in the file wins.

No setting picks a Memory. Recall and `membase_search` cover every Memory the key reaches; saves
and captures land in the key's **write target**, the one Memory it reaches. A key that reaches
several Memories has no write target and its saves are refused, so give the key a single Memory
under **Connect › Developer keys** if the agent should save. The `container` setting and
`MEMBASE_CONTAINER` of earlier releases are no longer read; the provider logs a warning when it
still finds them.

## How it behaves

**Recall (prefetch).** Hermes calls the provider before each non-trivial turn. Greetings,
"thanks" and slash commands are skipped. The provider sends the user's message (up to 2,000
characters, after Hermes' secret redaction) to `POST /v1/search` and gets up to `recall_limit`
passages back. The call uses its own client with the recall budget as its timeout and no
retries. Hermes also stops waiting for any external provider after 8 s, which is why a larger
budget has no effect. If a Memory could not answer (listed in the search's `partial`: no working
model, dormant), the block says it was not searched. A missing passage from that Memory is not proof that nothing is remembered.
A `403` (key narrowed or revoked) stops automatic recall for the rest of the session instead of
retrying every turn.

**Photos and videos.** The first recall with a key asks `GET /v1/capabilities` whether the key
reaches a media Memory (and may search it); the answer is kept for that key, so a key that reaches
none never pays for a media search, and a new key is asked again. When it does, each recall also
sends the message to `POST /v1/media/search`, beside the passage search and within the same budget,
and the block lists the matching items: the Memory, `photo` or `video`, the file name and the
item's `med_` id, the moments that matched (time, speaker, words) and the item's one-sentence
summary (`- [Trip] video lisbon.mp4 (id med_…) (0:57-0:59, the old man: "It's not safe here") — an
old man on a tram…`). The id is what `GET /v1/media/{id}` reads, so a host can show the item.
The agent gets descriptions, never the files. This runs with `tools: off` too.

**Tools.** Explicit calls use the SDK's default timeout (90 s), so they can wait out a cold start.

| Tool | Does |
|---|---|
| `membase_search(query, limit?)` | Passages from every Memory in reach, each naming its Memory. Memories that could not answer are listed under `unavailable`. |
| `membase_search_media(query, kind?, limit?)` | Photos and videos in the media Memories in reach, matching what was seen or said, each with its `id` (`med_…`), Memory, file name, summary and matched moments. `kind` is `image` or `video`. |
| `membase_save(content)` | Saves a note to the key's write target. The answer carries the note's `id` (`doc_…`) and its learning `status`: usually `processing` or `pending`, so the Memory learns it on its next run and it is searchable after that. |
| `membase_profile()` | The user's profile facts: standing statements about who they are, newest first. Works only when the key's owner ticked **Your profile**. |

When the key has no write target, `membase_save` says that nothing was saved and how the owner
fixes it. There is no delete or forget tool. Deleting stays with the owner, in the Membase app.

**Capture (opt-in).** With `capture_turns` on, the user's own words from each completed turn are
saved as one note, as they wrote them, to the key's write target. The assistant's reply is never sent
and no role labels are added, so an AI answer is never learned as a fact about the user. Captures never run from `cron`, `subagent` or
`flush` agents, or for turns written by bots. A key without a write target has its captures
refused, and the refusal is logged once.

**The user's profile (`memory.user_store: provider`).** A Hermes build that has the
`memory.user_store` setting can hand facts about the user to the provider instead of writing
USER.md. With it set to `provider` in the profile's `config.yaml`, what Hermes's memory tool
records about the user goes to the Membase profile (`POST /v1/profile-facts/add`, through
`record_user_fact`), and recall before each turn also reads the profile, beside the search, and
hands its standing facts (newest first, up to 3,000 characters) to the agent as data. A profile
that could not be read is said to be unread, never shown as empty. The agent cannot remove a
fact; the user forgets facts in the Membase app. `cron`, `subagent` and `flush` agents record none.

**Errors** come back to the model as one plain sentence:

| Error | Message |
|---|---|
| `403` | "this Membase key was narrowed or revoked, or the action is above its access level; do not retry with it" |
| timeout | "the memory may be waking up, try again shortly" |
| `422 write_target_required` | "this Membase key reaches several Memories and has no write target, so nothing was saved; give the key a single Memory under Connect > Developer keys" |
| other `422` | "no working model or no free turns" |

The key never appears in an error or a log line.

## Limits

- **Cold start.** A search is a turn inside the user's own Membase account. The first one after a
  quiet spell takes longer while that memory wakes. The automatic recall gives up
  after the budget, so the first turn or two of a session can run with no recall. Later turns
  find the memory awake. An explicit `membase_search` waits up to 90 s.
- **One key = one person.** A Membase key belongs to one account, and Hermes keeps one key per
  profile. This suits a **personal agent**: the profile's owner talking to their own Hermes. If
  the same profile answers several people on a messaging gateway (a group chat, a shared bot),
  every one of them reaches the owner's memory, as they do the built-in MEMORY.md. Give each
  person their own profile and key instead. For code that registers the provider itself,
  `MembaseMemoryProvider(client_for=lambda user_id: Membase(api_key=key_of(user_id)))` picks
  each turn's client from the gateway user id. A turn without a user id then gets no memory at
  all, never someone else's.
- **Saves are not instant.** A note is learned on the Memory's next run.
- **Plan usage.** Searches and learning runs execute in the user's Membase account and count
  against its plan.
- **Hosted only.** The catalog build talks to the hosted API. Membase's local mode
  (`Membase(local=...)`) is not offered here: it would run a model with credentials read from
  the process environment, which Hermes' credential rules do not allow for a plugin.

## Network and data disclosure

The only network peer is the Membase API (`https://api.app.membase.com`, or `MEMBASE_BASE_URL`).
The plugin sends:

- each non-trivial user message, as a recall query;
- with a key that reaches a media Memory, the same message as a photo and video query;
- `membase_search`, `membase_search_media`, `membase_save` and `membase_profile` calls;
- the user's own words from each completed turn, only when `capture_turns` is on (never the
  assistant's reply).

The only local state is `$HERMES_HOME/membase-ai.json` (read from the pre-0.5.0
`unibaseio-membase.json` while it does not exist). The plugin starts no background process,
runs no shell command, sends no telemetry and has no self-update code.

## Development

```bash
cd hermes
git clone --depth 1 https://github.com/NousResearch/hermes-agent .cache/hermes-agent
uv venv --python 3.14 .venv
uv pip install --python .venv/bin/python -e .cache/hermes-agent "membase-ai>=0.4.1,<0.5" pytest
.venv/bin/python -m pytest -q                                   # offline
.venv/bin/hermes plugins validate . --install-deps              # the catalog CI gate (run on a clean checkout)
MEMBASE_API_KEY=mbk_... .venv/bin/python examples/live_smoke.py   # real API, read-only (--write saves one note)
```

The tests install this folder as a user plugin in a temporary `HERMES_HOME`. Hermes' own loader
loads it, and the tests drive it through Hermes' `MemoryManager`: prefetch, tool routing,
`sync_all`, the secret scope and plugin-host isolation. Every HTTP request is answered by an
`httpx.MockTransport`, so the real SDK runs without a network.

## Licence

MIT
