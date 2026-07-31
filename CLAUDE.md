# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Desired-state deployment for the `once-colors` ONCE stack: one Oracle Cloud VM, DNS on Cloudflare, SMTP on Resend, OpenTofu state in Cloudflare R2. The application code itself lives in a separate repo (`../colors-website`), shipped here only as the container image referenced in `colors.yml`.

Nothing here is application source. What is tracked is desired state (`colors.yml`), the three installed launchers, the skill packages behind them, and the dev-environment files:

```text
colors.yml                     the desired state — the only file you normally edit
green, red, blue               installed launchers, one per colour (see Commands)
.agents/skills/package-once-*  the installed skill packages; the launchers above are copies of their payloads
.claude/skills/package-once-*  symlinks into .agents/skills, so Claude Code discovers them
skills-lock.json               records the skill source and content hash
.envrc                         secret-free; sources the gitignored .envrc.private
devenv.nix, devenv.lock        the toolchain (see Prerequisites in README.md)
.github/workflows/ci.yml       credential-free CI: `./green build` and the launcher-vs-payload diff
```

There is deliberately **no `package.json`, `bun.lock` or `node_modules`**. Red used to need them; its launcher now carries its own pins and resolves them on first run, exactly as green and blue always have. A manifest here would record the same commit a second time and drift from the launcher at the next re-pin — which it did, sitting at `72e8135` while the launchers had moved on. Adding one back changes nothing: the launcher reads only its own `PINS`. Repin by re-installing the skill.

Everything else is generated (`.colors/`) or secret (`.envrc.private`). Check `git ls-files` rather than assuming — `.gitignore` is `.*` with narrow negations, so what is tracked is not obvious from the working tree (see Gotchas).

## Commands

Three launchers are installed and they are interchangeable — same `colors.yml`, same DAG, same OpenTofu state. `./green` is the one used in practice and the one the examples below assume; `./red` (Bun) and `./blue` (uv) accept exactly the same commands and flags. Switch only between completed commands, never concurrently against the same state.

None of them needs an install step. Each resolves its own dependencies on first run — green into `~/.gitlibs`, blue through uv's script metadata, red into `~/.cache/package-once-red/` keyed on the launcher's own pins — and none writes into this repository. Red gained that behaviour in once `39f00e2`; before it, `./red` failed here on a bare `Cannot find package 'package-once-red'` because this repository has no `node_modules` and nothing had run `bun install`.

```sh
./green describe            # read-only status: providers, compute IP, per-app image/digest/update-available
./green build               # render the work tree under .colors/ only — no provider calls, no credentials needed
./green create              # converge everything (tofu stages + ansible)
./green create --dry-run    # walk the DAG, skip every side effect
./green delete              # reverse of create
```

`-f/--file` defaults to the nearest `colors.yml` walking up from the working directory, so any subdirectory works. Exit code 2 means validation failure or bad usage; the message lists every missing key at once.

`build` and `--dry-run` render from desired state alone and stay usable with no secrets in the environment. Only real `create`/`delete` require credentials.

`delete` is blocked while `compute-prevent-destroy: true` in `colors.yml`; the only unlock is `COLORS_PAR_COMPUTE_PREVENT_DESTROY=false` in the environment for that run (the flat key is deliberately overridable only at runtime).

## Architecture

`green` is a thin babashka launcher. It holds *no* logic beyond dependency resolution and locating `colors.yml`; validation, the workflow DAG, and every step live in the `io.github.getcolors/once` library, pinned by git SHA inside `green` itself (`once-sha`, `green-sha`, managed upstream by `bb pin` — do not hand-edit). `launcher-contract` guards against a stale pin: a mismatch exits 2 with an actionable message instead of a "could not locate" error. Set `ONCE_LIB_ROOT` / `GREEN_LIB_ROOT` to point at working trees instead of the pins.

To read library source: `~/.gitlibs/libs/io.github.getcolors/once/<once-sha>/green/src/clj/io/github/getcolors/once/` — `workflow.clj` (the DAG), `validate.clj` (provider registry: required keys, secrets, and which secrets are passed to OpenTofu as env vars), `tools.clj` (the steps and their templates), `github.clj` (deploy-key generation and publication), `describe.clj` (the read-only report).

### Updating to a newer ONCE

The launchers here are *installed copies*, so they lag the upstream monorepo until re-installed. Update by re-installing, never by editing a pin by hand — it takes two steps:

```sh
npx skills update -p            # refreshes .agents/skills/ and skills-lock.json
cp .agents/skills/package-once-green/green green    # and red, blue
```

The second step is not optional and nothing does it for you. **The three launchers at the repo root are copies of the skill payloads, not symlinks to them** — `skills update` rewrites `.agents/skills/` and leaves the root files untouched, so a project that skips the copy keeps running the old pin while `skills-lock.json` claims the new one. Refresh all three or the colours stop being interchangeable. The `launchers` job in `.github/workflows/ci.yml` diffs root against payload for each colour, so a skipped copy fails CI instead of going unnoticed — but only after the update is pushed, and only in this repository.

`npx skills use <pkg>@<skill>` is **not** the update command despite the name; it prints a prompt for using a skill without installing it, and leaves the project unchanged. The installing verbs are `add` and `update`.

Verify with `grep once-sha green` (or `grep once# red`) against the monorepo's `skills/package-once-green/green`.

An outdated launcher does not render from a stale contract — `launcher-contract` refuses to run and exits 2. Treat that as the signal to re-install.

The `create` DAG:

```
start ─┬─ tofu-compute ─┐                             ┌─ ansible-local
       └─ tofu-smtp ────┴─ tofu-dns ─ tofu-smtp-post ─┴─ ansible-remote ─ github
```

`delete` is not merely this graph reversed. It is a distinct wiring that runs strictly serially until the last fan-out, and it leads with two steps `create` never runs in that position:

```
start ─ github ─ ansible-cleanup ─ tofu-smtp-post ─ tofu-dns ─┬─ tofu-smtp
                                                              └─ tofu-compute
```

`github` runs *first* on delete, revoking the published Actions secret and variables before the server they point at goes away; `ansible-cleanup` then drops the managed `~/.ssh/config` block.

Stages hand off through OpenTofu `params` outputs: compute emits the public IP/user, smtp emits the Resend domain id and DNS records, which `tofu-dns` renders into `apps.tf.json` / `smtp.tf.json`. On a real `delete` the start step reads those outputs back out of state so a destroy has the same params a create had.

### Generated work tree

`build` scaffolds `.colors/<profile>/<tool>/` (workdir and profile come from `colors.yml`). It is gitignored and fully regenerated — **never edit anything under `.colors/` by hand**; change `colors.yml` or the upstream templates instead. Remote state keys are `<profile>/<tool>.tfstate`, so two profiles never collide in the same R2 bucket. `backend.tf.json` is written by a `:before` advice on each tofu stage rather than by the template.

### Secrets

No credential is ever written to `colors.yml`. Each secret flat key is supplied at runtime through `COLORS_PAR_<UPPER_SNAKE_KEY>` and overlaid onto the flat key by the start step. `.envrc` (committed, secret-free) sources `.envrc.private` (gitignored) — run `direnv allow` once. Currently in use: `COLORS_PAR_CLOUDFLARE_API_TOKEN`, `COLORS_PAR_RESEND_API_KEY`, `COLORS_PAR_RESEND_PASSWORD`, `COLORS_PAR_R2_ACCESS_KEY_ID`, `COLORS_PAR_R2_SECRET_ACCESS_KEY`, `COLORS_PAR_GITHUB_TOKEN`.

`COLORS_PAR_GITHUB_TOKEN` is required by `delete` as well as `create` — validation demands it whenever any application carries a `github:` key, because delete has to withdraw what create published.

Secrets that OpenTofu needs are exported into the tofu process environment (mapped in `validate/providers` `:tofu-env`) so they stay out of the plaintext `.tf` files in the work tree. The Resend SMTP password is *not* in `:tofu-env` — Ansible looks it up from the environment at play time.

OCI is the exception: it authenticates from `~/.oci/config` via `oci-config-file-profile`, with no `COLORS_PAR_*` var of its own.

### Remote server

`ansible-remote` installs docker, ONCE, and babashka, then creates a `deploy` user with NOPASSWD sudo restricted to `/usr/local/bin/once *`, and authorizes the deploy keys behind a ForceCommand (`/usr/local/bin/deploy`). That script accepts exactly `sudo once update <host>` for a host ONCE already knows — it is the CI deploy path, not a shell. The `once` Ansible module then reconciles the application list from `colors.yml` onto the box.

### Deploy keys

There is no `deploy-pubkey` key in `colors.yml`. Every repo named as `github: owner/repo` gets one keypair, **regenerated on every `create` and never stored** — per repository, not per application, so two apps sharing a repo share one key. The public half is installed with a ForceCommand naming every host that repo serves (`command="/usr/local/bin/deploy <host> [host...]"`), so one repo's key cannot deploy another's. A deploy is a ping: the client sends no command and the entry decides what to update. The private half, plus `SERVER_IP` / `SERVER_USER` / `SSH_KNOWN_HOSTS`, is published by the `github` step to a GitHub Actions environment named after the profile (`once-colors`) — via the `gh` CLI, so `gh` must be installed and `COLORS_PAR_GITHUB_TOKEN` must be able to write repo secrets. `SSH_KNOWN_HOSTS` is read off the server itself, so a workflow pins the host key instead of trusting `ssh-keyscan`.

Installing a new key invalidates the old one immediately, so `files/authorized-keys` deliberately keeps **one** previous generation per host alongside the current one: if publication to GitHub fails, the repo's existing key keeps working until the next `create` heals it. Two generations is the entire benefit — more only extends how long a leaked key stays usable.

## Gotchas

- `.gitignore` is `.*` with narrow negations (`!.envrc`, `!.gitignore`, `!.agents`, `!.claude`, `!.github`), so any new dotfile is invisible to git unless explicitly negated — `.github/workflows/ci.yml` needed `!.github` before `git add` would see it, and the same applies to the next one. The self-negation is what keeps the file tracked; without it the pattern hides `.gitignore` from itself, and it went missing from a clone in exactly that way once already. If it is ever absent, restore it before `git add`, or `.envrc.private` and `.colors/` stage as ordinary untracked files.
- The `github` step shells out to `gh`. It is not one of the tools `README.md` lists as a prerequisite, but a `create` with any `github:` application fails without it.
- `OCI_CLI_AUTH=security_token` in `.envrc` exists only because the local OCI `DEFAULT` profile is session-token based and the `oci` CLI rejects it otherwise. The OpenTofu `oracle/oci` provider detects `security_token_file` on its own — do not treat the variable as something `green` needs. Session tokens expire; a stale one surfaces as an auth failure at plan time, not as a config error.
- `oci-image-id` pins the boot image, so the compute template renders no image data source at all. Clearing that key brings the lookup back and makes `source_id` track whatever Canonical published most recently — and since it is ForceNew, a routine apply months later proposes destroying the VM. Leave it pinned; re-pin deliberately when the image should change. (The lookup itself is sound now — it filters on `oci-shape` rather than a hardcoded `VM.Standard.A1.Flex`.)
