# Forward

Organisation-wide automation that forwards newly opened **issues** and **pull requests** to a GitHub **project board**.

It replaces the built-in "Auto-add to project" workflows, which are capped at 5 repositories on Pro/Team plans, with a
single reusable workflow that any number of repositories can call.

## How it works

```
issue/PR opened
   in repo X
        │
        ▼
.github/workflows/forward.yml   (caller, in repo X — on: issues / pull_request)
        │  uses: …@main + secrets: inherit
        ▼
metreeca/.github/.github/workflows/forward.yml   (reusable — on: workflow_call)
        │  1. mint a GitHub App installation token
        │  2. actions/add-to-project
        ▼
   organisation project board
```

- **Caller** (`.github/workflows/forward.yml`, one per repository): triggers on `issues` / `pull_request`, passes the
  target `project-url`, and delegates to the central workflow with `secrets: inherit`.
- **Reusable workflow** (`forward.yml` in this repository): mints a short-lived **GitHub App** installation token via
  [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token), then adds the item with
  [`actions/add-to-project`](https://github.com/actions/add-to-project).

> [!NOTE]
> Event-triggered workflows run from the file on the repository's **default branch**. A caller only fires once
> `forward.yml` is present on that repo's default branch.

## Why a GitHub App

The automatic `GITHUB_TOKEN` cannot write to organisation-level Projects. A dedicated, org-owned **GitHub App**
(`metreeca-forward`) supplies the needed `Projects: read and write` permission, has no expiry, and survives staff
changes. The reusable workflow mints a fresh installation token per run; nothing long-lived is stored beyond the App's
private key.

## Adding a repository

Add `.github/workflows/forward.yml`:

```yaml
name: Forward
on:
  issues: { types: [ opened, reopened, transferred ] }
  pull_request: { types: [ opened, reopened ] }   # PRs have no "transferred" event
jobs:
  forward:
    uses: metreeca/.github/.github/workflows/forward.yml@main
    with:
      project-url: https://github.com/orgs/metreeca/projects/3
    secrets: inherit
```

> [!IMPORTANT]
> In repositories whose `main` is auto-synced from a `release/*` branch via `promote.yml` (`git merge --ff-only`),
> commit the caller on the **`release/*` branch**, never directly on `main`. A divergent commit on `main` breaks the
> fast-forward promote. The caller reaches `main` automatically on the next release.

New packages should include the caller in their scaffold so they self-cover with no change here.

## Configuration

| Setting           | Where                           | Purpose                                    |
|-------------------|---------------------------------|--------------------------------------------|
| `project-url`     | caller `with:` (required input) | Target project; each caller states its own |
| `FORWARD_APP_ID`  | org **variable**                | GitHub App client id (passed to `app-id`)  |
| `FORWARD_APP_KEY` | org **secret**                  | GitHub App private key (PEM)               |

Both the variable and secret must have visibility covering the calling repositories; `secrets: inherit` forwards the
secret into the reusable workflow.

## Maintenance

**Change the target project** — edit `project-url` in the relevant caller(s). No central change needed.

**Rotate the private key** — generate a new key on the App, then update the secret:

```bash
gh secret set FORWARD_APP_KEY --org metreeca --visibility all < new-key.pem
```

Delete the old key from the App afterwards, and remove the local `.pem`.

**Pause forwarding** — disable the caller workflow in a repo's Actions tab, or remove `forward.yml` from its default
branch.

**Grant the App access to more repositories** — the App is installed org-wide; new repositories are covered
automatically once they carry a caller.

## Troubleshooting

| Symptom                                     | Likely cause                                                                     |
|---------------------------------------------|----------------------------------------------------------------------------------|
| Run fails at "Generate app token"           | `FORWARD_APP_ID` / `FORWARD_APP_KEY` missing or not visible to the repo          |
| Token minted but "Forward to project" fails | App lacks `Projects: read and write`, or isn't installed on the org              |
| No run at all on a new issue/PR             | Caller not on the repo's **default branch** yet (e.g. still only on `release/*`) |
| Item added to the wrong board               | Caller's `project-url` points at the wrong project                               |

Run logs are under each repository's **Actions** tab, workflow **Forward**.
