# jfrog-xray

A [Docker Sandboxes](https://docs.docker.com/ai/sandboxes/) **mixin** that installs the
[JFrog CLI](https://jfrog.com/getcli/) (`jf`) and pre-wires it to your JFrog Platform, so an
agent can run [**JFrog Xray**](https://jfrog.com/xray/) security and license scans without any
in-sandbox configuration.

Xray is a core component of the JFrog Platform, integrated hand-in-hand with Artifactory and
sharing package metadata between the two. That integration is what makes a scan more than a flat
CVE list: Xray reports the **full impact path** of a vulnerability through your dependency graph,
so you can assess real risk and remediate the right thing.

The kit installs `jf` from a pinned, digest-verified JFrog release at sandbox creation time,
exports `JF_URL`/`JF_ACCESS_TOKEN`, and adds a `## JFrog Xray` section to the agent's context
describing the available scan commands. It composes onto any base agent (`claude`, `codex`,
`gemini`, `shell`, …).

## Architecture

The agent and `jf` run inside an isolated microVM. The real access token never enters that
microVM — the sandbox only holds a `proxy-managed` sentinel, and the `sbx` egress proxy (the trust
boundary) swaps in the real token, read from the host secret store, only on allow-listed outbound
requests to your JFrog Platform.

```text
+----------------------------------------------------------------------------+
|  Sandbox microVM   --   isolated, never sees the real token                |
+----------------------------------------------------------------------------+
|                                                                            |
|    Agent   ( claude / codex / gemini ... )                                 |
|       |                                                                    |
|       |  runs                                                              |
|       v                                                                    |
|    jf  CLI                                                                 |
|       JF_URL           =  https://<jfrog_host>                             |
|       JF_ACCESS_TOKEN   =  proxy-managed                                   |
|                           (sentinel placeholder, not the real token)       |
+----------------------------------------------------------------------------+
                                       |
                                       |   HTTPS  |  Authorization: Bearer proxy-managed
                                       v
+----------------------------------------------------------------------------+
|  sbx egress proxy   --   trust boundary                                    |
+----------------------------------------------------------------------------+
|                                                                            |
|    -  allowlist :  releases.jfrog.io ,  <jfrog_host>   ( deny all else )   |
|    -  reads the real token from the host secret store  ( host-side only )  |
|    -  swaps   proxy-managed   ==>   <real token>   on the way out          |
+----------------------------------------------------------------------------+
                                       |
                                       |   HTTPS  |  Authorization: Bearer <real token>
                                       v
+----------------------------------------------------------------------------+
|  JFrog Platform                                                            |
+----------------------------------------------------------------------------+
|                                                                            |
|    Xray          ( vulnerability + license scanning, policy )              |
|       ^                                                                    |
|       |  shared package metadata                                           |
|       v                                                                    |
|    Artifactory   ( binaries + dependency-graph metadata )                  |
+----------------------------------------------------------------------------+
```

## Usage

`jfrog-xray` is a mixin, so it stacks onto a base agent with `--kit`. Your JFrog Platform host is
per-user, so it's supplied as a kit argument (`--kit-arg jfrog_host=<host>`) — hostname only, no
scheme or path (e.g. `mycompany.jfrog.io`).

From this repo over git (the spec lives at the repo root, so no `dir=` is needed; pin to a
commit SHA — remote kit refs must be a full 40-character SHA, not a branch or tag):

```console
sbx run claude \
  --kit "git+https://github.com/ajeetraina/sbx-kits-jfrog-xray.git#ref=<commit-sha>" \
  --kit-arg jfrog_host=mycompany.jfrog.io .
```

From a local clone of this repo (run from the repo root):

```console
sbx run claude --kit . --kit-arg jfrog_host=mycompany.jfrog.io .
```

Then, inside the sandbox:

```console
agent@claude-my-project:~/my-project$ jf audit
agent@claude-my-project:~/my-project$ jf audit --licenses --format=json
agent@claude-my-project:~/my-project$ jf scan ./dist/app.jar
agent@claude-my-project:~/my-project$ jf docker scan myimage:latest
```

### Host-side prerequisite: the access token

The kit declares a `jfrog` credential. On first run, `sbx` prompts you to bind it, or set it up
ahead of time:

```console
sbx secret set jfrog <your-jfrog-access-token>
```

Generate the token in the JFrog Platform UI (**Administration → User Management → Access Tokens**,
or a reference/identity token from your profile). It needs **Xray read + scan** scopes. The token
is stored on the **host**; the sandbox only ever sees the placeholder `proxy-managed`, and the
proxy injects the real value on outbound requests to your JFrog host. See
[credential bindings](https://docs.docker.com/ai/sandboxes/customize/kits/) for env-var/file
sources.

## How auth and egress work

`jf` reads `JF_URL` and `JF_ACCESS_TOKEN` from the environment, so **no `jf config add` step is
needed** — the kit wires both for you:

| Variable | Value inside the sandbox |
| --- | --- |
| `JF_URL` | `https://<jfrog_host>` (from your `--kit-arg`) |
| `JF_ACCESS_TOKEN` | `proxy-managed` — a placeholder, never the real token |

`jf` sends `Authorization: Bearer proxy-managed` to your JFrog host. The sandbox proxy recognizes
the sentinel and swaps in the real token from the host — the same proxy-injection model the
`glab` and `github` kits use. **The real token never enters the sandbox filesystem or process
memory.**

The network allowlist is exactly two hosts:

| Host | Why |
| --- | --- |
| `releases.jfrog.io` | Download the pinned `jf` binary at install time (JFrog's own release host) |
| `<jfrog_host>` | The Xray + Artifactory REST API on your instance, at runtime |

Everything else is denied. If you use image scanning that pulls from a registry, or a self-hosted
platform whose Xray API lives on a separate host, add those with a per-sandbox rule rather than
widening the kit:

```console
sbx policy allow network --sandbox claude-my-project "registry-1.docker.io,auth.docker.io"
```

### Self-hosted platforms

SaaS instances (`mycompany.jfrog.io`) are the common case and work out of the box. A self-hosted
platform with Artifactory and Xray on distinct hostnames, a non-standard port, or a path prefix
may need an explicit `jf config add` inside the sandbox and additional `sbx policy allow` entries.
`JF_URL` covers the single-host unified-URL layout only.

## Version pinning

The install command pins:

- `JF_VERSION=2.121.0`
- SHA256 per-arch, checked with `sha256sum -c` before the binary is placed on `PATH`:
  - `amd64`: `7d9fcfd1d21d779cf18e96a0ae97706c6d15808ff79a8fa1b91f046d0fd419ca`
  - `arm64`: `1858ad5e2acfcaecb5da5b5f623cd667b85cce33b59181d60384dc5f42908351`

No `curl | sh`. To bump: edit `spec.yaml`, update `JF_VERSION` and both checksums. The checksum
of a given arch's binary is the sha256 of the raw file at
`https://releases.jfrog.io/artifactory/jfrog-cli/v2-jf/<version>/jfrog-cli-linux-<arch>/jf`.

## Cleanup

This kit installs a binary and sets env vars inside the sandbox; it writes nothing to the host.
`sbx rm <sandbox>` removes the sandbox and everything the kit installed. The host-side access
token in your `sbx` secret store is untouched — remove it with `sbx secret rm jfrog` if you no
longer need it.
