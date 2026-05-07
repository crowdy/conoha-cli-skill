# Contributing

## Secret-handling rules (must read)

This repository ships Claude Code "skills" (recipes and automation
scripts) that direct an agent to provision real ConoHa infrastructure.
Recipes are copied verbatim by users, so any value baked into a command
example is something the user is likely to run as-is.

To prevent the kind of incident where production-looking data leaks into
samples, contributors **must** follow these rules:

1. **No hardcoded passwords or tokens in command examples.** Replace
   them with `$(openssl rand -base64 24)` (generated at runtime) or
   `<PLACEHOLDER>` (forces the user to substitute). Never write
   `--env ADMIN_PASSWORD=SecurePass123` or similar — users will copy
   that into production.

2. **No real IPs, VM IDs, hostnames, SSH key names, or tenant IDs.**
   Use `<SERVER_IP>`, `<VM_ID>`, `<TENANT_ID>` placeholders. For
   documentation IPs use the RFC 5737 reserved ranges
   (`192.0.2.0/24`, `198.51.100.0/24`, `203.0.113.0/24`).

3. **No insecure SSH options as defaults.** `StrictHostKeyChecking=no`
   creates a MITM vulnerability. If you must show it for a "first run"
   case, mark it clearly as test-only and pair it with a comment
   explaining how to do it securely (`ssh-keyscan` + `known_hosts`).

4. **Don't pass secrets via `--env` (process list / API logs).**
   Cluster keys, SSH private keys, and similar secrets must travel via
   `--upload` (file copy with restricted perms) or a secrets manager,
   not as environment variables that show up in `ps aux` and the
   ConoHa API request log.

5. **Pre-publish drafts and handoff memos belong outside git.**
   `docs/memory/` is gitignored.

## How we enforce this

* **`gitleaks` runs on every PR** via GitHub Actions
  (`.github/workflows/gitleaks.yml`). PRs with detected secrets are
  blocked from merge.
* **`.gitignore`** blocks `.env`, `.env.*`, `*.pem`, `*.key`,
  `id_rsa*`, `*credentials*`, and `docs/memory/`.
* **Reviewers should grep PRs** for:
  * known token prefixes (`sk_`, `pk_`, `whsec_`, `hf_`, `Bearer `)
  * literal IPs (any `\d+\.\d+\.\d+\.\d+`)
  * `password=`, `token=`, `secret=` followed by a non-placeholder
  * `StrictHostKeyChecking=no`

## If you accidentally commit a secret

1. **Rotate it immediately at the source service** — assume the value
   is already public.
2. After rotating, remove the value from `HEAD` in a follow-up commit.
3. To purge it from git history, use `git filter-repo` and force-push,
   then notify other contributors so they re-clone. Do this only after
   rotation.
