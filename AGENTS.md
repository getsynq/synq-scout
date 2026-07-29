# Driving `synq-scout`

This is the operating guide for a coding agent using `synq-scout`: what it can do,
how to reach it, which command to pick, and the mistakes that waste a session. It
is written to be acted on top-down.

**Scout is an agent over your data estate.** It answers questions about your
warehouse and your Coalesce Quality workspace — what an asset is, what changed,
what broke and why, what to test — and it can act: set an issue's status, comment,
open an incident, author monitors and tests. Every capability is a **tool**, and
the same tool set backs all three ways of running it.

Scout only ever connects outward, so it can run inside your network with nothing
exposed.

---

## 1. Pick the right surface

Three ways in. The first is almost always the right answer.

**1. The hosted MCP server — no install.** Scout's tools are served over MCP at a
URL, so an MCP client (Claude Code, Cursor, Claude Desktop) can use them with
nothing running on your side. If your goal is "let my agent query the estate",
stop here — you do not need this binary at all.

| Region | URL |
|---|---|
| EU | `https://mcp.synq.io/mcp` |
| US | `https://mcp.us.synq.io/mcp` |
| AU | `https://mcp.au.synq.io/mcp` |

**2. `synq-scout mcp` — the same tools, served locally.** Use this when the data
must not leave your network, or when you need warehouse tools that query your
databases directly with credentials you hold. It speaks MCP over stdio, so point
your client at the command rather than a URL:

```json
{
  "mcpServers": {
    "coalesce-quality": {
      "command": "synq-scout",
      "args": ["mcp"]
    }
  }
}
```

**3. `synq-scout tools` — one tool, one command, no client.** The fastest way to
answer a single question, and the only surface where you can see a tool's exact
schema. Start here when exploring what Scout can do.

`synq-scout agent` is the fourth mode: long-lived, picking up triage work from your
workspace on its own. It is a deployment, not something you drive — see the
installation notes that ship beside this guide.

---

## 2. Authenticate, and confirm what you are pointed at

Scout takes the first credential it finds:

1. `QUALITY_CLIENT_ID` + `QUALITY_CLIENT_SECRET` — for servers and containers.
2. `QUALITY_TOKEN` — an `st-…` API token.
3. A browser login, cached under `~/.synq/oauth/`.

```bash
synq-scout auth login          # browser flow
synq-scout auth whoami         # who you are, which workspace, which permissions
```

**Read `auth whoami` before anything that writes.** It is the only thing that tells
you which workspace a mutation will land in. `auth status` shows every stored
credential across regions, `auth token` prints a fresh access token for scripting,
`auth logout` removes one.

If your workspace is not in the EU, pass `--region us` (or `au`), or set
`QUALITY_REGION`. It works on every command, not just `auth`, and a successful
login is remembered — so later commands need no flag.

Then confirm the whole setup at once:

```bash
synq-scout health
```

This checks the API, the model endpoint and every configured warehouse
connection. Run it before blaming a tool for failing.

---

## 3. The loop

**1. Find the tool.** Do not guess a tool name; the list is derived from the build
you have.

```bash
synq-scout tools list
```

**2. Read its schema before calling it.**

```bash
synq-scout tools describe get_entity_details
```

`describe` prints the arguments, which are required, and the enum values a field
accepts. This is cheaper than a failed call and far cheaper than a wrong one.

**3. Call it.**

```bash
synq-scout tools search_entities --query "orders"
synq-scout tools get_entity_details --entity-id "<id from the search above>"
```

Flags are derived from the schema: `issueId` becomes `--issue-id`, a string array
becomes a repeatable flag, and anything nested takes JSON. For a deeply nested
argument, pass the whole payload instead:

```bash
synq-scout tools save_domain --json '{"name":"Finance","definition":{...}}'
```

**4. Read the response, then follow where it points.** Tool responses carry a
`tool_usage_guidance` field when there is an obvious next step — it names the
specific follow-up tool and the argument to carry over. Follow it rather than
guessing, and rather than re-running the same list with tweaked filters.

---

## 4. Never do these

- **Do not call a tool without `describe`-ing it first.** Argument names are
  derived from a schema, not from convention, and a plausible guess usually fails.
- **Do not self-confirm a gated write.** Tools that would reset a check's baseline,
  overwrite something the UI owns, or delete a resource return
  `requires_user_confirmation` with a list of ids, and apply nothing. Those ids are
  a *human* authorization. Surface the impact, get a person to agree, then re-send
  with the confirmation. Passing the ids straight back because you saw them in the
  response defeats the entire mechanism.
- **Do not treat an empty result as "no data".** Scout returns an explicit "no
  results for &lt;filter&gt;" rather than an empty list. If you got that message, the
  filter is the problem, not the estate.
- **Do not parse an id out of a handle.** Some responses intern a repeated
  sub-record into a table with short handles. Handles are local to that one
  response — never pass one back as an argument, and never assume it means anything
  next time.
- **Do not assume a write happened under `--dry-run`.** See below.

---

## 5. Dry run: the default differs per surface

`synq-scout tools` defaults to **`--dry-run=true`**. Read calls pass through
normally; a write call returns its optimistic response but **nothing reaches the
backend**. That makes exploring safe, and it makes "I ran the tool and nothing
changed" the expected outcome rather than a bug.

```bash
synq-scout tools set_issue_status --issue-id <id> --status FIXED                 # dropped
synq-scout tools set_issue_status --issue-id <id> --status FIXED --dry-run=false # applied
```

The response says so explicitly when a mutation was dropped. `mcp` and `agent`
default to `--dry-run=false`, because a client driving them expects writes to land.

---

## 6. Reading the output

`-o` selects the format: `json` (default), `yaml`, or `toon` — a compact tabular
encoding that costs far fewer tokens on repeated records.

```bash
synq-scout tools list_open_issues -o toon
```

**Logs never contaminate the payload.** Under `tools`, `auth` and shell
completion, logs go to stderr and stdout carries only the result, so
`synq-scout tools … -o json | jq` is safe. Set `LOG_LEVEL=DEBUG` when a call fails
for no visible reason; `LOG_FORMAT=text` makes that readable.

Responses are shaped for an agent, not a UI: list rows carry a few load-bearing
fields, with `include_*` arguments to ask for more. Counts and breakdowns are
pre-computed, so do not make a second call to total something up. Long text is
truncated with its original size and how to fetch the rest — never silently cut.

---

## 7. Warehouse tools need connections; most tools do not

Anything answerable from your Coalesce Quality workspace works with no
configuration at all. Tools that query your warehouses directly — column
profiling, value sampling — need those connections declared in `agent.yaml`, and
say so plainly ("no direct database connections are available") when they are
missing. That message means configuration, not breakage.

The example configuration that ships beside this guide is the starting point, and
your workspace can generate the connection blocks for you, already carrying the
right connection ids, from
[Settings → Scout](https://app.synq.io/settings/scout).

The field-by-field reference is
[the published schema](https://schemas.synq.io/synq-scout/v1/config.html), and it is
authoritative in a way prose is not — read it rather than inferring field names. The
example carries a `yaml-language-server` line pointing at the same schema, so an
editor validates the file as you write it.

`agent.yaml` is optional. A missing file is not an error.

---

## 8. What each thing costs

- `tools list` / `describe` — free. No credentials needed, no backend call; tool
  metadata is built locally.
- A read tool — one API call. Cheap, but a list with `include_*` toggles on can
  return a lot; prefer the default shape and widen deliberately.
- A warehouse tool (`profile_columns`, `sample_column_values`) — **a real query
  against your warehouse**, billed by your warehouse. Scope it before running it
  broadly.
- `mcp` / `agent` — the model calls are the cost, against whichever provider your
  endpoint fronts. Scout never picks a provider for you.

---

## 9. When something fails

| Symptom | Cause |
|---|---|
| `no direct database connections are available` | Configuration, not a fault — see §7 |
| A write returned success but nothing changed | `--dry-run` is on by default under `tools` — §5 |
| `requires_user_confirmation` | A gated write. Get human agreement; do not self-confirm — §4 |
| A tool is missing from `tools list` | Your credential's permissions do not include it, or this build predates it |
| `PermissionDenied` | The credential lacks the scope; `auth whoami` lists what it has |
| `InvalidArgument` | Re-read `tools describe` — usually a wrong field name or enum value |
| Wrong workspace | `auth whoami`, and check `--region` |

`synq-scout health` distinguishes "Scout is broken" from "Scout cannot reach
something", which is the more common case.
