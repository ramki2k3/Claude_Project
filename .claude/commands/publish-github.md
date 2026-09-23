---
description: Security-scan, then push this project to a GitHub repo, update README + repo About section, and deploy GitHub Pages via Actions
argument-hint: <github repo URL or owner/repo> [extra notes]
---

# Publish this project to GitHub

Target repo and notes from the user: **$ARGUMENTS**

Work through the steps below in order. **Step 3 (security scan) is a hard gate:** nothing leaves this machine until it passes. Keep the user posted with one short line per step, and finish with the report in step 8.

General rules for this command:
- Push over **SSH** (`git@github.com:<owner>/<repo>.git`). On this machine the saved HTTPS credentials belong to a different GitHub account, so HTTPS pushes fail with 403.
- Never force-push, rewrite history or delete branches without asking the user first.
- End commit messages with the attribution line from the current session's instructions.
- GitHub's anonymous REST API allows only 60 requests an hour. Call it sparingly (a few calls per run), and **never** poll it in a loop. To wait for a deploy, poll the Pages site URL instead.
- `gh` may not be installed. Check with `command -v gh` before relying on it.

## 1. Work out the target repo

1. Parse `$ARGUMENTS` into `OWNER/REPO`. Accept `https://github.com/OWNER/REPO(.git)`, `git@github.com:OWNER/REPO.git` or `OWNER/REPO`.
2. If `$ARGUMENTS` is empty, use the existing `origin` remote. If there is none, **ask the user** for the repo and stop until they answer.
3. Look the repo up once: `curl -s https://api.github.com/repos/OWNER/REPO`. Note `visibility`, `default_branch`, `has_pages`, `description`, `homepage` and `topics`.
   - **404:** the repo doesn't exist, or it's private and not visible anonymously. Ask the user to create it on github.com (empty, with no README), then continue.
   - **Private repo:** warn that GitHub Pages on a free plan needs a public repo.
4. Check SSH access: `ssh -T git@github.com`. It should print `Hi <user>!`. If that user isn't `OWNER`, and isn't a collaborator on the repo, the push will fail. Tell the user now.

## 2. Prepare the local repo

1. If this folder isn't a git repo, run `git init -b main`.
2. Set `origin` to the SSH URL: use `git remote add` if it's missing, or `git remote set-url` if it points somewhere else. Confirm first before replacing a remote that points at a *different* repo.
3. If the remote already has commits (`git ls-remote origin`), fetch and check whether local and remote histories are related. If they've diverged or are unrelated, **stop and ask** the user how to proceed. Don't merge or overwrite on your own.
4. Make sure `.gitignore` covers at least: `.env*`, `*.pem`, `*.key`, `*.p12`, `id_rsa*`, `.DS_Store`, `node_modules/`, and `.claude/settings.local.json`.

## 3. Security scan (gate: must pass before any push)

Scan **everything that will become public**:
- every file that will be committed (`git ls-files` plus untracked, non-ignored files)
- every local commit not yet on the remote (`git log -p origin/<branch>..HEAD`, or all history if the remote is empty), because once pushed, history is public too

Run these checks:

1. **Secrets.** If `gitleaks` is installed, run `gitleaks detect --source . --redact --no-banner`, which covers git history. Also always run `grep -rnE` over the files and unpushed diffs for:
   - private key blocks: `-----BEGIN [A-Z ]*PRIVATE KEY-----`
   - cloud and API keys: `AKIA[0-9A-Z]{16}`, `AIza[0-9A-Za-z_-]{35}`, `sk-[A-Za-z0-9_-]{20,}`, `sk-ant-[A-Za-z0-9_-]+`
   - GitHub and Slack tokens: `gh[pousr]_[A-Za-z0-9]{36}`, `github_pat_[A-Za-z0-9_]+`, `xox[abprs]-[A-Za-z0-9-]+`
   - generic assignments: `(api[_-]?key|secret|passwd|password|token|bearer)["'\s]*[:=]\s*["'][^"'\s]{8,}`
2. **Files that must not be published:** `.env*`, `*.pem`, `*.key`, `*.p12`, `id_rsa*`, credential or service-account JSON, database dumps, `.claude/settings.local.json`.
3. **Personal data:** real email addresses (anything except `example.com`, `noreply`, and the attribution line), phone numbers, internal hostnames or IPs.
   - In this project, the email inside `FORMSUBMIT_ENDPOINT` becomes public on the live site by design. If it's a real address, **tell the user explicitly** and get a yes before publishing.
4. **Web-page risks** (for HTML/JS that will go live on Pages):
   - `innerHTML`, `outerHTML`, `insertAdjacentHTML` or `document.write` fed with user input that isn't passed through `escapeHtml()`
   - `eval` or `new Function`
   - `<script src>` from third parties without `integrity`
   - `http://` resources (mixed content)
   - `target="_blank"` without `rel="noopener"`
   - `fetch` calls to any host other than the intended ones (here, only FormSubmit)
5. **Project rules** from `CLAUDE.md`. For this repo, confirm that this finds nothing unexpected:
   ```
   grep -nE "localStorage|sessionStorage|indexedDB|document\.cookie|!important|alert\(|confirm\(" index.html
   ```
6. **Workflow hygiene** in `.github/workflows/*`: `permissions:` is least-privilege, there are no secrets echoed to logs, and there are no `pull_request_target` triggers that run untrusted code.

Result:
- **Any secret or must-not-publish file:** **STOP.** Push nothing. Show each finding with file and line, redacting the secret value itself. Suggest the fix: remove the file, move the value to an env var or secret, and rotate the key. If the secret is already in a local commit, tell the user it has to be removed from history, and ask before doing that.
- **Personal data or web-risk warnings:** list them and ask the user whether to fix them or publish anyway.
- **Clean:** say "Security scan passed" with a one-line summary of what was checked, then continue.

## 4. README

- **Existing `README.md`:** read it and update only what's stale or missing. Keep the user's own wording and structure.
- **No README:** create one for people visiting the repo. Include:
  - title and one-line summary
  - screenshot, if the app can be rendered (see the headless Chrome command in `CLAUDE.md`, and save it under `docs/`)
  - live-demo link: `https://OWNER.github.io/REPO/` (for a repo named `OWNER.github.io`, it's `https://OWNER.github.io/`)
  - quick start, features, and configuration (e.g. `FORMSUBMIT_ENDPOINT` plus the FormSubmit activation step)
  - limitations
- Base every claim on the actual code. Don't invent features, badges or licences.
- If the README changes, re-run the step 3 checks on it.

## 5. Commit and push

1. Stage only the intended files. Review `git status` and never `git add` anything that step 3 flagged.
2. Commit with a descriptive message plus the attribution line.
3. `git push -u origin <branch>`.
4. **If the push is rejected:**
   - **403 / permission denied:** the wrong account. Show which account SSH authenticates as and what access it needs.
   - **Non-fast-forward:** stop and ask. Don't force-push.

## 6. GitHub Pages via GitHub Actions

1. Create or update `.github/workflows/pages.yml`:
   - triggers: `push` to the default branch, plus `workflow_dispatch`
   - `permissions: { contents: read, pages: write, id-token: write }`
   - `concurrency: { group: pages, cancel-in-progress: false }`
   - one `deploy` job with `environment: github-pages`
   - steps: `actions/checkout` → `actions/configure-pages` → assemble `_site/` → `actions/upload-pages-artifact` (path `_site`) → `actions/deploy-pages`
   - **Publish only the site files** into `_site/` (for this repo, just `index.html` plus a `.nojekyll`), not README, CLAUDE.md, `.claude/` or `docs/`
   - Use each action's **current major version**. Look them up once with `curl -s https://api.github.com/repos/actions/<name>/releases/latest`; older versions trigger Node deprecation warnings.
2. If the workflow changed, commit and push it (step 5).
3. **Check Pages is enabled** (`has_pages` from step 1, re-checked once if needed). The workflow can't switch Pages on by itself. If it's off, give the user these exact steps and wait for them to say "done":
   1. Sign in on github.com as **OWNER**. Click the avatar in the top-right to check. If the repo has no ⚙️ **Settings** tab, the browser is signed in as the wrong account.
   2. Open `https://github.com/OWNER/REPO/settings/pages`.
   3. Under **Build and deployment → Source**, choose **GitHub Actions**. It saves automatically.
4. Once Pages is on, trigger a deploy. Without `gh`, push an empty commit (`git commit --allow-empty -m "Trigger GitHub Pages deploy"`). With `gh`, run `gh workflow run pages.yml`.
5. **Verify the site:** poll the Pages URL (not the API) with a bounded loop, for example up to 3 minutes: `curl -s -o /dev/null -w '%{http_code}'` until it returns 200. Then confirm the served page matches the local file, e.g. `curl -s <url> | diff -q - index.html`.
   - **If it fails:** fetch the latest run's failed step once via `/actions/runs?per_page=1` and its jobs, and explain the error in plain words. "Get Pages site failed / Not Found" means Pages isn't enabled with Source set to GitHub Actions.

## 7. Repo "About" section

Aim for these values:
- **Description:** one sentence from the README summary, 350 characters maximum.
- **Website:** the Pages URL from step 6.
- **Topics:** 3–8 lowercase, hyphenated tags that describe the project, e.g. `kanban`, `vanilla-js`, `github-pages`, `project-management`.

Apply them with the first option that works:
1. **`gh` installed and authenticated as someone with admin rights** (`gh auth status`):
   - `gh repo edit OWNER/REPO --description "…" --homepage "…" --add-topic a --add-topic b`
2. **`GITHUB_TOKEN` (or `GH_TOKEN`) set in the environment:**
   - `curl -X PATCH -H "Authorization: Bearer $GITHUB_TOKEN" https://api.github.com/repos/OWNER/REPO -d '{"description":"…","homepage":"…"}'`
   - then `PUT /repos/OWNER/REPO/topics` with `{"names":[…]}`
   - Never print the token or write it to any file.
3. **Neither is available** (SSH can't edit repo settings). Give the user the exact text to paste:
   1. Open `https://github.com/OWNER/REPO`.
   2. On the right side of the Code tab, click the ⚙️ gear next to **About**.
   3. Paste the description, tick **Use your GitHub Pages website** (or paste the URL into Website), and add the topics.
   4. Click **Save changes**.

Afterwards, verify with one `curl -s https://api.github.com/repos/OWNER/REPO` call and check `description`, `homepage` and `topics`.

## 8. Final report

Reply with:
- **Security scan:** passed, or what was found and what was done about it.
- **Commits pushed:** short hashes and one-line summaries.
- **README:** created or updated, and what changed.
- **About section:** applied automatically, or the values the user still needs to paste.
- **GitHub Pages:** the live URL and whether it was verified, or the exact remaining step for the user.
- **Anything waiting on the user:** each item as a numbered step.
