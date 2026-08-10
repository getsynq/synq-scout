# synq-scout

Coalesce Quality Scout — an AI agent for your data estate.

Scout investigates your data: it triages issues, works out what changed, and
suggests tests, using your warehouse and your Coalesce Quality workspace. It runs
entirely on infrastructure you control and only ever connects outward, so it can
sit inside your network with nothing exposed.

This repository is the download channel. The source lives in our monorepo; what
you find here is the release archives, this guide, and an example configuration.

## Install

The archive filename carries the version, so the version has to be resolved
first. GitHub redirects `/releases/latest` to the newest release's tag, which
needs no API token and no login:

```bash
VERSION=$(curl -fsSLI -o /dev/null -w '%{url_effective}' \
  https://github.com/getsynq/synq-scout/releases/latest | sed 's#.*/v##')
OS=$(uname -s | tr '[:upper:]' '[:lower:]')          # darwin or linux
ARCH=$(uname -m | sed 's/x86_64/amd64/; s/aarch64/arm64/')

curl -fL "https://github.com/getsynq/synq-scout/releases/download/v${VERSION}/synq-scout_${VERSION}_${OS}_${ARCH}.tar.gz" \
  | tar -xz
sudo mv synq-scout /usr/local/bin/
```

To pin a version instead, set `VERSION` by hand from the
[Releases](https://github.com/getsynq/synq-scout/releases) page.

Builds are published for macOS and Linux on both amd64 and arm64.

### Upgrading

```bash
synq-scout upgrade --check      # what it would do, without doing it
synq-scout upgrade
```

`upgrade` resolves the latest release, downloads the archive for this platform,
verifies it against the release's `checksums.txt`, and runs the new binary once to
prove it works on this machine before replacing anything. If the binary lives
somewhere you cannot write — `/usr/local/bin` usually is not — it says so and
changes nothing; re-run it with `sudo`. A binary installed by a package manager is
left to that package manager, and a Kubernetes or Docker deployment should upgrade
the image rather than the binary inside it.

`synq-scout` also mentions a newer release on stderr, at most once a day. That
check reads a tag from a public GitHub URL and sends nothing but the tool name and
version — no credentials, no workspace, no identity. It never delays the command it
runs beside and never reports its own failure, so a machine with no route to the
internet behaves exactly like one that is up to date. It is already silent in CI,
when output is not a terminal, and inside a container or a Kubernetes pod. To
switch it off everywhere:

```bash
export QUALITY_NO_UPDATE_CHECK=1     # DO_NOT_TRACK=1 has the same effect
```

On macOS a downloaded binary may be quarantined. If it refuses to start,
`xattr -d com.apple.quarantine /usr/local/bin/synq-scout` clears the flag.

### Docker

```bash
docker pull europe-docker.pkg.dev/synq-cicd-public/synq-public/synq-scout:latest
```

### Kubernetes

Manifests, a Kustomize overlay and a worked example live in
[getsynq/synq-scout-k8s](https://github.com/getsynq/synq-scout-k8s). That is the
recommended shape for a long-running deployment.

### Verify the download

Every release ships a `checksums.txt`:

```bash
sha256sum -c checksums.txt --ignore-missing
```

Confirm which build you have with `synq-scout --version`.

## Sign in

Scout resolves credentials in this order, and takes the first that is available:

1. **Client credentials** — `QUALITY_CLIENT_ID` + `QUALITY_CLIENT_SECRET`, or the
   `synq` block in `agent.yaml`. This is what a server or container should use.
   Create the pair under Settings → API in your workspace.
2. **API token** — `QUALITY_TOKEN`, the `st-…` token from your workspace.
3. **Browser login** — `synq-scout auth login`, which caches a refresh token under
   `~/.synq/oauth/`. The simplest way to try Scout out as yourself.

```bash
synq-scout auth login
synq-scout auth whoami   # identity, workspace and granted scopes
```

The cache is shared with our other CLIs for the same endpoint, so one login covers
all of them.

If your workspace is in North America, add `--region us` (or set
`QUALITY_REGION=us`). The flag works on every command, not just `auth`.

## Configure

`agent.yaml` in the working directory is **optional**. Without it, Scout takes its
credentials and LLM settings from the environment, and everything that needs only
your Quality workspace works. Only the tools that query a warehouse directly —
column profiling, value sampling — need the `connections` section, and they return
an explicit "no direct database connections are available" rather than failing when
it is absent.

Start from [`agent.example.yaml`](agent.example.yaml). The connection blocks can be
generated for you, already carrying the right connection ids, at
<https://app.synq.io/settings/scout>.

Scout needs an OpenAI-compatible endpoint for its model calls. We recommend
[LiteLLM](https://docs.litellm.ai/) as a proxy in front of whichever provider you
have a contract with — Scout never talks to a model provider we chose for you.

Check the whole setup before running anything real:

```bash
synq-scout health
```

## Run

```bash
synq-scout agent                        # long-lived; works tasks as they arrive
synq-scout mcp                          # an MCP server, for Claude Code or any MCP client
synq-scout tools list                   # every tool as a command you can run yourself
synq-scout issue --issue-id <id>        # triage now; repeat the flag for several
```

With no subcommand, Scout runs the agent.

`synq-scout tools` is the quickest way to see what Scout can actually do: `tools
list` enumerates them, `tools describe <tool>` prints the arguments, and
`tools <tool> --<arg> …` runs one directly. Writes default to a dry run there, so
exploring is safe.

## Use it from an MCP client

`synq-scout mcp` serves Scout's tools over MCP, which is how you drive it from
Claude Code or any other MCP client. Point your client at the command:

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

Which tools a client may call is decided by the scopes on its credential, so a
read-only token yields a read-only tool set.

## Documentation

- **`AGENTS.md`, shipped beside the binary, is the operating guide** — which
  surface to pick, the tool loop, the dry-run defaults, output formats, and what each
  call costs. Written for a coding agent driving Scout; also published at
  <https://docs.synq.io/scout/agent-workflow>.
- CLI reference — <https://docs.synq.io/scout/cli>
- MCP tools and permission tiers — <https://docs.synq.io/scout/mcp>
- Kubernetes deployment — <https://github.com/getsynq/synq-scout-k8s>

## Support

Questions and problems: <https://docs.synq.io/support/support>.

## Licence

Apache 2.0 — see [LICENSE](LICENSE).
