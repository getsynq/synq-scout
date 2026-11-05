# Driving `synq-scout`

This guide is for a coding agent that uses `synq-scout`. It says what Scout can do, how to reach it, which command to use, and which mistakes to avoid. Read it from the top.

Scout is an agent for your data estate. It answers questions about your warehouse and your Coalesce Quality workspace: what an asset is, what changed, what broke and why, what to test. It can also act: set the status of an issue, add a comment, open an incident, create monitors and tests. Each capability is a tool, and all ways to run Scout use the same tool set.

Scout only makes outbound connections, so it can run inside your network with nothing exposed.

---

## 1. Pick the right surface

There are three ways to use Scout. The first one is usually correct.

**1. The hosted MCP server, with no install.** Scout's tools are served over MCP at a URL. An MCP client (Claude Code, Cursor, Claude Desktop) can use them with nothing running on your side. If you only want your agent to query the estate, use this and do not install the binary.

| Region | URL |
|---|---|
| EU | `https://mcp.synq.io/mcp` |
| US | `https://mcp.us.synq.io/mcp` |
| AU | `https://mcp.au.synq.io/mcp` |

**2. `synq-scout mcp`: the same tools, served locally.** Use it when the data must stay in your network, or when you need the warehouse tools, which query your databases with credentials that you hold. It speaks MCP over stdio, so give your client the command, not a URL:

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

**3. `synq-scout tools`: one tool, one command, no client.** This is the fastest way to answer one question, and the only surface that shows the exact schema of a tool. Start here to learn what Scout can do.

`synq-scout agent` is a fourth mode. It runs for a long time and takes triage work from your workspace without help. It is a deployment, not a command you drive. See the installation notes next to this guide.

---

## 2. Authenticate, and confirm what you are pointed at

Scout uses the first credential that it finds:

1. `QUALITY_CLIENT_ID` + `QUALITY_CLIENT_SECRET`, for servers and containers.
2. `QUALITY_TOKEN`, an `st-…` API token.
3. A browser login, cached under `~/.synq/oauth/`.

```bash
synq-scout auth login          # browser flow
synq-scout auth whoami         # who you are, which workspace, which permissions
```

Run `auth whoami` before anything that writes. It is the only command that shows the workspace where a change will land.

- `auth status` shows every stored credential in all regions.
- `auth token` prints a fresh access token for scripts.
- `auth logout` removes a credential.
- By default a login goes to the shared slot, so one login serves every Coalesce Quality CLI on the machine. `auth login --isolated` gives Scout its own credential.
- The login asks only for the permissions that Scout's tools need. If a permission is not granted, its tool is not in the MCP tool list.

`synq-scout auth login --coalesce` signs you in with your Coalesce account. Add `--organization <org-id>` to skip the organization choice. This credential carries your Coalesce Quality roles in the organization's workspace, not consented permissions. Other Coalesce Quality CLIs share it like any login, and it refreshes automatically. `auth status` shows its organization and roles.

If your workspace is not in the EU, pass `--region us` or `--region au`, or set `QUALITY_REGION`. The flag works on every command, not only `auth`. Scout remembers the region of a successful login, so later commands need no flag.

Then check the whole setup:

```bash
synq-scout health
```

`health` checks the API, the model endpoint and every configured warehouse connection. Run it before you decide that a tool is broken.

---

## 3. The loop

1. Find the tool. Do not guess a tool name: the list comes from the build you have.

   ```bash
   synq-scout tools list
   ```

2. Read its schema before you call it.

   ```bash
   synq-scout tools describe get_entity_details
   ```

   `describe` prints the arguments, which of them are required, and the enum values of each field. It costs less than a failed call, and much less than a wrong one.

3. Call it.

   ```bash
   synq-scout tools search_entities --query "orders"
   synq-scout tools get_entity_details --entity-id "<id from the search above>"
   ```

   The flags come from the schema: `issueId` becomes `--issue-id`, a string array becomes a repeatable flag, and a nested field takes JSON. For a deeply nested argument, send the whole payload:

   ```bash
   synq-scout tools save_domain --json '{"name":"Finance","definition":{...}}'
   ```

4. Read the response, then follow where it points. When there is an obvious next step, a response has a `tool_usage_guidance` field. It names the next tool and the argument to carry over. Follow it. Do not guess, and do not run the same list again with changed filters.

---

## 4. Never do these

- Do not call a tool before you `describe` it. Argument names come from a schema, not from a convention, and a plausible guess usually fails.
- Do not confirm a gated write yourself. A tool that would reset a check's baseline, overwrite something the UI owns or delete a resource returns `requires_user_confirmation` with a list of ids, and applies nothing. These ids are a human authorization. Show the impact, get agreement from a person, then send the call again with the confirmation. If you send the ids back only because you saw them in the response, the protection does nothing.
- Do not read an empty result as "no data". Scout returns an explicit "no results for &lt;filter&gt;" instead of an empty list. If you get that message, the filter is the problem, not the estate.
- Do not take an id from a handle. Some responses put a repeated sub-record into a table with short handles. A handle is valid only in that one response. Never send one back as an argument.
- Do not assume that a write happened under `--dry-run`. See §5.

---

## 5. Dry run: the default differs per surface

`synq-scout tools` defaults to `--dry-run=true`. Read calls run as normal. A write call returns its expected response, but nothing reaches the backend. So exploring is safe, and "I ran the tool and nothing changed" is the expected result, not a bug.

```bash
synq-scout tools set_issue_status --issue-ids <id> --status fixed                 # dropped
synq-scout tools set_issue_status --issue-ids <id> --status fixed --dry-run=false # applied
```

The response says when a write was dropped. `mcp` and `agent` default to `--dry-run=false`, because a client that drives them expects its writes to land.

---

## 6. Reading the output

`-o` selects the format: `json` (default), `yaml`, or `toon`. `toon` is a compact table format that uses far fewer tokens on repeated records.

```bash
synq-scout tools list_open_issues -o toon
```

Logs never mix with the payload. Under `tools`, `auth` and shell completion, logs go to stderr and stdout has only the result, so `synq-scout tools … -o json | jq` is safe. If a call fails with no visible reason, set `LOG_LEVEL=DEBUG`. `LOG_FORMAT=text` makes the logs readable.

The responses are made for an agent, not for a UI:

- List rows carry a few main fields. Use the `include_*` arguments to get more.
- Counts and breakdowns are already computed. Do not make a second call to add them up.
- Long text is truncated, with its original size and the way to get the rest. Nothing is cut without a note.

### Ids: send back exactly what you got

A response gives one spelling of each reference, and every other tool takes that spelling. Do not remove a prefix, change the case, or build an id from parts you recognise. An id that looks like a table name plus a suffix is still opaque. A rebuilt id usually points at nothing and looks correct. Most assets are named by `entity_id`, a monitor by `monitor_entity_id`, and a segment of a monitor by `segment_entity_id`.

If a tool accepts more than one spelling, `describe` says so, and the response gives the canonical id it resolved. Use that one from then on. If a reference does not resolve, you get an error that says what was wrong. An empty result means that the reference resolved and nothing was there.

---

## 7. Warehouse tools need connections; most tools do not

Everything that your Coalesce Quality workspace can answer works with no configuration. Tools that query your warehouses directly, such as column profiling and value sampling, need the connections in `agent.yaml`. If the connections are missing, these tools say "no direct database connections are available". That message means configuration, not a fault.

- Start from the example configuration next to this guide.
- Your workspace can generate the connection blocks, with the correct connection ids, at [Settings → Scout](https://app.synq.io/settings/scout).
- The field reference is [the published schema](https://schemas.synq.io/synq-scout/v1/config.html). It is the authority on field names, so read it instead of guessing. The example has a `yaml-language-server` line that points at the schema, so your editor validates the file.
- `agent.yaml` is optional. A missing file is not an error.

---

## 8. What each thing costs

- `tools list` and `describe` are free. They need no credentials and make no backend call, because the tool metadata is built locally.
- A read tool makes one API call. It is cheap, but a list with `include_*` options can return a lot of data. Use the default shape first, and widen it only when you need to.
- A warehouse tool (`profile_columns`, `sample_column_values`) runs a real query on your warehouse, and your warehouse bills it. Limit its scope before you run it widely.
- In `mcp` and `agent`, the model calls are the cost, with the provider that your endpoint uses. Scout never selects a provider for you.
- `upgrade` replaces this binary with the latest release, verified against the release's `checksums.txt`. `upgrade --check` reports what it would do and changes nothing. Neither one touches the workspace or a warehouse.
- About once a day, Scout prints a note about a newer release on stderr. It stays silent when an agent drives it, when the output is not a terminal, in CI and in a container, so the note is never in output you parse. `QUALITY_NO_UPDATE_CHECK=1` turns it off.

---

## 9. When something fails

| Symptom | Cause |
|---|---|
| `no direct database connections are available` | Configuration, not a fault. See §7 |
| A write returned success but nothing changed | `--dry-run` is on by default under `tools`. See §5 |
| `requires_user_confirmation` | A gated write. Get agreement from a person, do not confirm it yourself. See §4 |
| A tool is missing from `tools list` | Your credential's permissions do not include it, or your build is older than the tool |
| `PermissionDenied` | The credential does not have the scope. `auth whoami` lists what it has |
| `InvalidArgument` | Read `tools describe` again. Usually a field name or an enum value is wrong |
| Wrong workspace | Run `auth whoami`, and check `--region` |

`synq-scout health` tells "Scout is broken" apart from "Scout cannot reach something". The second one is more common.
