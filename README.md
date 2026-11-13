# GitOps with Docker: podinfo + Dockhand + Infisical + Tailscale

A fork-and-run template that teaches **GitOps without Kubernetes**. Git is the
source of truth for *what* runs; a pull-based agent makes the host match it.

| Piece | Role |
| --- | --- |
| [podinfo](https://github.com/stefanprodan/podinfo) | The demo app (UI, `/env`, `/configs`, `/version`, `/metrics`, health probes) |
| [Dockhand](https://dockhand.pro) | GitOps engine: polls this repo, redeploys `podinfo/` on change |
| [Infisical](https://infisical.com) Cloud | Secrets — never in git, injected at deploy time |
| [Tailscale](https://tailscale.com) sidecar | Private HTTPS on your tailnet; optional public Funnel |
| [Renovate](https://docs.renovatebot.com) | Opens PRs when podinfo / tailscale release |

```
                 ┌──────── GitHub (your fork) ────────┐
 Renovate PR ──▶ │ podinfo/compose.yml, config/...    │
                 └──────────────▲─────────────────────┘
                                │ poll (~1 min)
 VPS / homelab VM ──────────────┼──────────────────────────────────
                         ┌──────┴─────┐      ┌ podinfo stack (GitOps) ──────┐
                         │  Dockhand  │─────▶│ tailscale ◀─ podinfo :9898   │
                         └──────▲─────┘      └──────────────────────────────┘
                                │ Infisical provider (/podinfo)
                         Infisical Cloud
```

## Repository layout

```
podinfo/                 # ← Dockhand git stack points here
  compose.yml            # podinfo + tailscale sidecar
  config/config.yaml     # UI message / color / logo
  tailscale/serve.json   # tailnet HTTPS; "AllowFunnel" toggle
.github/
  linter/.yamllint.yaml  # YAML style rules
  renovate.json          # Renovate config (config:best-practices)
  workflows/ci.yml       # lint (prek) + podinfo smoke test
  workflows/renovate.yml # self-hosted Renovate, hourly + manual
.pre-commit-config.yaml  # git hooks, run by prek
```

---

## 1. Prerequisites

- A Linux VPS or homelab VM with Docker Engine and the compose plugin.
- **A running [Dockhand](https://dockhand.pro/manual) on that host.** This repo
  doesn't install it. When you install it, keep these in mind:
  - Mount its data directory **at the same path on the host and in the
    container**, with `DATA_DIR` set to it (e.g. `-v /opt/dockhand:/opt/dockhand -e DATA_DIR=/opt/dockhand`).
    The podinfo stack bind-mounts `./config` and `./tailscale`, and those relative
    paths only resolve to real host paths with matching paths.
  - Keep Dockhand private (LAN, VPN or tailnet), enable authentication, and
    back up its `ENCRYPTION_KEY`.
- A Tailscale account with **MagicDNS** and **HTTPS certificates** enabled.
- A free Infisical Cloud account (US `https://app.infisical.com` by default; EU
  users use `https://eu.infisical.com` everywhere a URL appears below).
- A fork of this repo.

**Where's the line between manual and GitOps?** Dockhand is what does the
deploying, so you install it by hand. Everything under `podinfo/` is
driven from git.

## 2. Tailscale: tag, policy and OAuth client

Add to your tailnet policy file (Access controls):

```jsonc
{
  "tagOwners": {
    "tag:podinfo": ["autogroup:admin"]
  },
  "nodeAttrs": [
    // Allow Funnel (public exposure) for podinfo only.
    { "target": ["tag:podinfo"], "attr": ["funnel"] }
  ]
  // keep your existing "grants"/"acls" so your devices can reach tag:podinfo
}
```

Then **Settings → OAuth clients → Generate**: scope **Auth Keys (write)**, tag
`tag:podinfo`. Copy the client secret and append
`?ephemeral=false&preauthorized=true`:

```
tskey-client-XXXX?ephemeral=false&preauthorized=true
```

That full string is your `TS_AUTHKEY`. Unlike auth keys, it doesn't expire after 90 days.

## 3. Infisical: project, secrets, machine identity

1. Create project **`gitops-demo`**, environment **`prod`**.
2. Create folder **`/podinfo`** with these secrets:

   | Secret | Value |
   | --- | --- |
   | `TS_AUTHKEY` | the OAuth string from step 2 |
   | `DEMO_API_KEY` | any **fake** value, e.g. `demo-v1` |

   > ⚠️ `DEMO_API_KEY` is shown publicly at podinfo's `/env` — that's the point
   > of the demo. Never put a real credential in the `/podinfo` folder.

3. **Organization → Access Control → Identities → Create identity** (`gitops-demo`),
   auth method **Universal Auth**. Add it to the project with the **Viewer** role.
   Save its **Client ID** and **Client Secret**. Dockhand uses them.

## 4. Register the podinfo git stack in Dockhand

1. **Settings → Git → Credentials**: add an HTTPS credential (GitHub username +
   a fine-grained PAT with *Contents: read*) if your fork is private. Public forks need none.
2. **Settings → Git → Repositories**: add `https://github.com/<you>/<your-fork>.git`.
3. **Settings → Secrets**: add an **Infisical** provider:
   API host `https://us.infisical.com` (or EU), Universal Auth with the client
   ID and secret from step 3, Project ID, environment `prod`.
4. **Stacks → New → Git stack**:
   - Repository: your fork, branch `main`
   - Compose file path: `podinfo/compose.yml`
   - Auto-sync: **on**, schedule every minute (`* * * * *`)
   - Secret provider: the Infisical provider from step 3
   - Environment variables:
     - `DOCKHAND_SECRET_SELECTOR` = `/podinfo` (mark it **secret**). This pulls
       every secret in that folder. Compose picks up `${TS_AUTHKEY}` and `${DEMO_API_KEY}`.
     - *(optional)* `TS_HOSTNAME` = `podinfo`. Change it to rename the tailnet node.
5. **Deploy**. Within a minute, **https://podinfo.&lt;your-tailnet&gt;.ts.net** is live.

Non-secret, fork-specific values (repo URL, project ID, hostname) live in
Dockhand's settings and are not committed to git.

## 5. Renovate

1. Create a fine-grained PAT on your fork with **Contents**, **Pull requests**,
   **Issues** and **Workflows** set to read/write, plus **Metadata: read**.
2. Add it as the repo secret **`RENOVATE_TOKEN`** (Settings → Secrets → Actions).
3. **Actions → renovate → Run workflow**. After that it runs every hour.

Renovate reads [`.github/renovate.json`](.github/renovate.json). Images are
pinned by tag; Renovate's first PR also pins digests (`config:best-practices`).
It also bumps GitHub Actions and the hook revisions in `.pre-commit-config.yaml`.
PRs are merged by hand, so you see each step. CI (below) tests every PR first.

## 6. Hooks and CI

### Local hooks with prek

[prek](https://prek.j178.dev) is a fast drop-in replacement for `pre-commit`
and reads the same [`.pre-commit-config.yaml`](.pre-commit-config.yaml).

```bash
uv tool install prek        # or: brew install prek / cargo binstall prek
prek install                # run the hooks on every git commit
prek run --all-files        # run everything now
```

| Hook | Catches |
| --- | --- |
| pre-commit-hooks | trailing whitespace, missing final newline, CRLF, invalid YAML/JSON, merge-conflict markers, case conflicts, large files, private keys |
| gitleaks | hard-coded secrets. They belong in Infisical, not git |
| yamllint | YAML style ([`.github/linter/.yamllint.yaml`](.github/linter/.yamllint.yaml)) |
| actionlint | mistakes in GitHub Actions workflows |
| compose-config | `docker compose config` on `podinfo/compose.yml`, using dummy secrets (needs Docker) |

`--all-files` only checks files git tracks, so `git add` new files first.

### CI

[`.github/workflows/ci.yml`](.github/workflows/ci.yml) runs on every push to
`main` and on every pull request, including Renovate's:

- **lint**: runs the same prek hooks as above, so CI and your laptop agree.
- **smoke-test**: starts podinfo using the image pinned in `compose.yml` and
  the repo's `config/config.yaml`, then checks that:
  - `/readyz` and the compose healthcheck command pass
  - the page's message equals `ui-message` from `config.yaml`
  - a dummy `DEMO_API_KEY` shows up at `/env`
  - `/version` matches the pinned image tag

  Tailscale isn't started in CI (there's no tailnet), so this job checks the app
  and its config, not the network sidecar.

A green check on a Renovate PR means the new podinfo version booted with your
config before you merge it. That's the gate in front of the GitOps rollout.

---

## Demo exercises

### A. Change the page with a commit
Edit `podinfo/config/config.yaml` (`ui-message`, `ui-color`, uncomment
`ui-logo`), commit, push. Within about a minute Dockhand notices the commit,
recreates the stack (podinfo reads config only at startup), and the page
changes. You never touched the server. `/configs` shows the mounted file.

### B. Upgrade by merging a PR, then roll back with a revert
Merge a Renovate PR that bumps `ghcr.io/stefanprodan/podinfo`. Watch `/version`
change. Then `git revert` the merge commit and push. That's a rollback, and it
uses the same mechanism.

### C. Go public with Funnel (and back)
In `podinfo/tailscale/serve.json`, set `"AllowFunnel"` to `true`, then commit and push.
podinfo is now reachable from the internet at the same URL. Set it back to `false` to
make it private again. The policy limits Funnel to `tag:podinfo`.

### D. Rotate a secret
Change `DEMO_API_KEY` in Infisical. Note that **nothing happens**: git didn't
change, so the poller has nothing to deploy. Click **Redeploy** in Dockhand (or
push any commit under `podinfo/`) and `/env` shows the new value.
**Lesson:** secrets live outside git, so rotating one needs a deploy.

### E. Health and readiness
The compose healthcheck calls `/healthz` via `podcli`. Dockhand shows the
container health. Try `curl -X POST https://podinfo.<tailnet>.ts.net/readyz/disable`
then `GET /readyz` (503), then `/readyz/enable`. `/metrics` exposes Prometheus metrics.

### F. (Optional) Push instead of pull: webhooks
Dockhand can also redeploy on a GitHub webhook
(`https://<dockhand>/api/git/stacks/<id>/webhook`, HMAC secret). That means GitHub
has to reach Dockhand, which conflicts with keeping Dockhand private. Polling
needs no inbound access, which is why this demo uses it. Compare the two models:
**pull** (agent watches git) versus **push** (git calls the agent).

---

## Licensing note

Dockhand is licensed under the **Business Source License 1.1**. It's free for
homelab, education, demos and training. Running it for an organization's
production systems requires a commercial license. podinfo (Apache-2.0),
Tailscale and Renovate have their own terms.
