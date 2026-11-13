# GitOps with Docker: podinfo + Dockhand + Infisical + Tailscale

A fork-and-run template that teaches **GitOps without Kubernetes**. Git is the
source of truth for *what* runs; a pull-based agent makes the host match it.

| Piece | Role |
| --- | --- |
| [podinfo](https://github.com/stefanprodan/podinfo) | The demo app (UI, `/env`, `/configs`, `/version`, `/metrics`, health probes) |
| [Dockhand](https://dockhand.pro) | GitOps engine: polls this repo, redeploys `stacks/podinfo` on change |
| [Infisical](https://infisical.com) Cloud | Secrets — never in git, injected at deploy time |
| [Tailscale](https://tailscale.com) sidecar | Private HTTPS on your tailnet; optional public Funnel |
| [Renovate](https://docs.renovatebot.com) | Opens PRs when podinfo / tailscale / dockhand release |

```
                 ┌────────── GitHub (your fork) ──────────┐
 Renovate PR ──▶ │ stacks/podinfo/compose.yml, config/... │
                 └───────────────▲────────────────────────┘
                                 │ poll (~1 min)
 VPS / homelab VM ───────────────┼────────────────────────────────────
  ┌ dockhand stack (manual) ─────┴──┐   ┌ podinfo stack (GitOps) ──────┐
  │ tailscale ◀─ dockhand :3000 ────┼──▶│ tailscale ◀─ podinfo :9898   │
  └───────▲─────────────────────────┘   └──────▲───────────────────────┘
          │ infisical run (/dockhand)          │ Dockhand Infisical provider (/podinfo)
          └──────────── Infisical Cloud ───────┘
```

## Repository layout

```
bootstrap/dockhand/      # Dockhand + tailscale sidecar — started by hand, once
stacks/podinfo/          # ← Dockhand git stack points here
  compose.yml            # podinfo + tailscale sidecar
  config/config.yaml     # UI message / color / logo
  tailscale/serve.json   # tailnet HTTPS; "AllowFunnel" toggle
.github/workflows/renovate.yml
renovate.json
```

**Why is Dockhand outside GitOps?** Something has to start the thing that does
the deploying. The bootstrap stack is the one manual step; everything under
`stacks/` is driven from git.

---

## 1. Prerequisites

- A Linux VPS or homelab VM with Docker Engine + the compose plugin.
- The [Infisical CLI](https://infisical.com/docs/cli/overview) on that host.
- A Tailscale account with **MagicDNS** and **HTTPS certificates** enabled.
- A free Infisical Cloud account (US `https://app.infisical.com` by default; EU
  users use `https://eu.infisical.com` everywhere a URL appears below).
- A fork of this repo.

## 2. Tailscale: tags, policy and OAuth client

Add to your tailnet policy file (Access controls):

```jsonc
{
  "tagOwners": {
    "tag:podinfo":  ["autogroup:admin"],
    "tag:dockhand": ["autogroup:admin"]
  },
  "nodeAttrs": [
    // Funnel only for podinfo — Dockhand can never be made public by accident.
    { "target": ["tag:podinfo"], "attr": ["funnel"] }
  ]
  // keep your existing "grants"/"acls" so your devices can reach these tags
}
```

Then **Settings → OAuth clients → Generate**: scope **Auth Keys (write)**, tags
`tag:podinfo` and `tag:dockhand`. Copy the client secret and append
`?ephemeral=false&preauthorized=true`:

```
tskey-client-XXXX?ephemeral=false&preauthorized=true
```

That full string is your `TS_AUTHKEY`. Unlike auth keys, it doesn't expire after 90 days.

## 3. Infisical: project, secrets, machine identity

1. Create project **`gitops-demo`**, environment **`prod`**.
2. Create two folders and secrets:

   | Folder | Secret | Value |
   | --- | --- | --- |
   | `/dockhand` | `ENCRYPTION_KEY` | output of `openssl rand -base64 32` |
   | `/dockhand` | `TS_AUTHKEY` | the OAuth string from step 2 |
   | `/podinfo` | `TS_AUTHKEY` | the same OAuth string |
   | `/podinfo` | `DEMO_API_KEY` | any **fake** value, e.g. `demo-v1` |

   > ⚠️ `DEMO_API_KEY` is shown publicly at podinfo's `/env` — that's the point
   > of the demo. Never put a real credential in the `/podinfo` folder.

3. **Organization → Access Control → Identities → Create identity** (`gitops-demo`),
   auth method **Universal Auth**. Add it to the project with the **Viewer** role.
   Save its **Client ID** and **Client Secret**. This one identity is used by
   both the host bootstrap and Dockhand.

## 4. Bootstrap Dockhand (the one manual step)

On the host:

```bash
git clone https://github.com/<you>/<your-fork>.git && cd <your-fork>/bootstrap/dockhand
sudo mkdir -p /opt/dockhand   # Dockhand data dir — same path on host and in container

export INFISICAL_DOMAIN=https://app.infisical.com   # or https://eu.infisical.com
export INFISICAL_PROJECT_ID=<project id from Infisical project settings>
export INFISICAL_TOKEN=$(infisical login --method=universal-auth \
  --client-id=<client id> --client-secret=<client secret> --silent --plain)

infisical secrets-manager run --projectId="$INFISICAL_PROJECT_ID" \
  --env=prod --path=/dockhand -- docker compose up -d
```

(`infisical run …` is the older alias of `infisical secrets-manager run …` and works too.)

The secrets exist only in the `docker compose` process environment and are never
written to disk. Open **https://dockhand.&lt;your-tailnet&gt;.ts.net**, create the admin
user, and enable authentication under **Settings → Authentication**.

> 🔑 Back up `ENCRYPTION_KEY` (it's in Infisical). Losing it makes the
> credentials stored in Dockhand unrecoverable.

## 5. Register the podinfo git stack in Dockhand

1. **Settings → Git → Credentials**: add an HTTPS credential (GitHub username +
   a fine-grained PAT with *Contents: read*) if your fork is private. Public forks need none.
2. **Settings → Git → Repositories**: add `https://github.com/<you>/<your-fork>.git`.
3. **Settings → Secrets**: add an **Infisical** provider:
   API host `https://us.infisical.com` (or EU), Universal Auth with the client
   ID and secret from step 3, Project ID, environment `prod`.
4. **Stacks → New → Git stack**:
   - Repository: your fork, branch `main`
   - Compose file path: `stacks/podinfo/compose.yml`
   - Auto-sync: **on**, schedule every minute (`* * * * *`)
   - Secret provider: the Infisical provider from step 3
   - Environment variables:
     - `DOCKHAND_SECRET_SELECTOR` = `/podinfo` (mark it **secret**). This pulls
       every secret in that folder. Compose picks up `${TS_AUTHKEY}` and `${DEMO_API_KEY}`.
     - *(optional)* `TS_HOSTNAME` = `podinfo`. Change it to rename the tailnet node.
5. **Deploy**. Within a minute, **https://podinfo.&lt;your-tailnet&gt;.ts.net** is live.

Non-secret, fork-specific values (repo URL, project ID, hostname) live in
Dockhand's settings and are not committed to git.

## 6. Renovate

1. Create a fine-grained PAT on your fork with **Contents**, **Pull requests**,
   **Issues** and **Workflows** set to read/write, plus **Metadata: read**.
2. Add it as the repo secret **`RENOVATE_TOKEN`** (Settings → Secrets → Actions).
3. **Actions → renovate → Run workflow**. After that it runs every hour.

Images are pinned by tag; Renovate's first PR also pins digests
(`config:best-practices`). PRs are merged by hand, so you see each step.

---

## Demo exercises

### A. Change the page with a commit
Edit `stacks/podinfo/config/config.yaml` (`ui-message`, `ui-color`, uncomment
`ui-logo`), commit, push. Within about a minute Dockhand notices the commit,
recreates the stack (podinfo reads config only at startup), and the page
changes. You never touched the server. `/configs` shows the mounted file.

### B. Upgrade by merging a PR, then roll back with a revert
Merge a Renovate PR that bumps `ghcr.io/stefanprodan/podinfo`. Watch `/version`
change. Then `git revert` the merge commit and push. That's a rollback, and it
uses the same mechanism.

> PRs labelled `manual-rollout` touch `bootstrap/`. Dockhand doesn't deploy
> itself, so after merging, `git pull` on the host and re-run the step 4 command.

### C. Go public with Funnel (and back)
In `stacks/podinfo/tailscale/serve.json`, set `"AllowFunnel"` to `true`, then commit and push.
podinfo is now reachable from the internet at the same URL. Set it back to `false` to
make it private again. The policy limits Funnel to `tag:podinfo`.

### D. Rotate a secret
Change `DEMO_API_KEY` in Infisical. Note that **nothing happens**: git didn't
change, so the poller has nothing to deploy. Click **Redeploy** in Dockhand (or
push any commit to the stack) and `/env` shows the new value.
**Lesson:** secrets live outside git, so rotating one needs a deploy.

### E. Health and readiness
The compose healthcheck calls `/healthz` via `podcli`. Dockhand shows the
container health. Try `curl -X POST https://podinfo.<tailnet>.ts.net/readyz/disable`
then `GET /readyz` (503), then `/readyz/enable`. `/metrics` exposes Prometheus metrics.

### F. (Optional) Push instead of pull: webhooks
Dockhand can also redeploy on a GitHub webhook
(`https://<dockhand>/api/git/stacks/<id>/webhook`, HMAC secret). That means GitHub
has to reach Dockhand, which conflicts with keeping Dockhand off the internet.
Polling needs no inbound access, which is why this demo uses it. Compare the two models:
**pull** (agent watches git) versus **push** (git calls the agent).

---

## Licensing note

Dockhand is licensed under the **Business Source License 1.1**. It's free for
homelab, education, demos and training. Running it for an organization's
production systems requires a commercial license. podinfo (Apache-2.0),
Tailscale and Renovate have their own terms.
