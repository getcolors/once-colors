# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

Desired-state deployment for the `colors-website` ONCE stack: one Oracle Cloud VM, DNS on Cloudflare, SMTP on Resend, OpenTofu state in Cloudflare R2. Only five files are tracked (`colors.yml`, `green`, `.envrc`, `README.md`, this file) — everything else is generated or secret. The application code itself lives in a separate repo (`../colors-website`), shipped here only as the container image referenced in `colors.yml`.

## Commands

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

`green` is a thin babashka launcher. It holds *no* logic beyond dependency resolution and locating `colors.yml`; validation, the workflow DAG, and every step live in the `io.github.bigconfig-ai/once` library, pinned by git SHA inside `green` itself (`once-sha`, `green-sha`, managed upstream by `bb pin` — do not hand-edit). `launcher-contract` guards against a stale pin: a mismatch exits 2 with an actionable message instead of a "could not locate" error. Set `ONCE_LIB_ROOT` / `GREEN_LIB_ROOT` to point at working trees instead of the pins.

To read library source: `~/.gitlibs/libs/io.github.bigconfig-ai/once/<once-sha>/green/src/clj/io/github/bigconfig_ai/once/` — `workflow.clj` (the DAG), `validate.clj` (provider registry: required keys, secrets, and which secrets are passed to OpenTofu as env vars), `tools.clj` (the steps and their templates).

The DAG (`create`; `delete` runs it in reverse, preceded by an ansible-cleanup step that drops the managed `~/.ssh/config` block):

```
start ─┬─ tofu-compute ─┐                        ┌─ ansible-local
       └─ tofu-smtp ────┴─ tofu-dns ─ smtp-post ─┴─ ansible-remote
```

Stages hand off through OpenTofu `params` outputs: compute emits the public IP/user, smtp emits the Resend domain id and DNS records, which `tofu-dns` renders into `apps.tf.json` / `smtp.tf.json`. On a real `delete` the start step reads those outputs back out of state so a destroy has the same params a create had.

### Generated work tree

`build` scaffolds `.colors/<profile>/<tool>/` (workdir and profile come from `colors.yml`). It is gitignored and fully regenerated — **never edit anything under `.colors/` by hand**; change `colors.yml` or the upstream templates instead. Remote state keys are `<profile>/<tool>.tfstate`, so two profiles never collide in the same R2 bucket. `backend.tf.json` is written by a `:before` advice on each tofu stage rather than by the template.

### Secrets

No credential is ever written to `colors.yml`. Each secret flat key is supplied at runtime through `COLORS_PAR_<UPPER_SNAKE_KEY>` and overlaid onto the flat key by the start step. `.envrc` (committed, secret-free) sources `.envrc.private` (gitignored) — run `direnv allow` once. Currently in use: `COLORS_PAR_CLOUDFLARE_API_TOKEN`, `COLORS_PAR_RESEND_API_KEY`, `COLORS_PAR_RESEND_PASSWORD`, `COLORS_PAR_R2_ACCESS_KEY_ID`, `COLORS_PAR_R2_SECRET_ACCESS_KEY`.

Secrets that OpenTofu needs are exported into the tofu process environment (mapped in `validate/providers` `:tofu-env`) so they stay out of the plaintext `.tf` files in the work tree. The Resend SMTP password is *not* in `:tofu-env` — Ansible looks it up from the environment at play time.

OCI is the exception: it authenticates from `~/.oci/config` via `oci-config-file-profile`, with no `COLORS_PAR_*` var of its own.

### Remote server

`ansible-remote` installs docker, ONCE, and babashka, then creates a `deploy` user with NOPASSWD sudo restricted to `/usr/local/bin/once *`, and authorizes `deploy-pubkey` behind a ForceCommand (`/usr/local/bin/deploy`). That script accepts exactly `sudo once update <host>` for a host ONCE already knows — it is the CI deploy path, not a shell. The `once` Ansible module then reconciles the application list from `colors.yml` onto the box.

## Gotchas

- `.gitignore` is `.*` with `!.envrc`, so any new dotfile is invisible to git unless explicitly negated.
- `OCI_CLI_AUTH=security_token` in `.envrc` exists only because the local OCI `DEFAULT` profile is session-token based and the `oci` CLI rejects it otherwise. The OpenTofu `oracle/oci` provider detects `security_token_file` on its own — do not treat the variable as something `green` needs. Session tokens expire; a stale one surfaces as an auth failure at plan time, not as a config error.
- The compute template's image data source hardcodes `shape = "VM.Standard.A1.Flex"` and ignores `oci-shape`. Harmless while the shape stays Ampere/ARM (A1, A2), but it would select the wrong image for an x86 shape.
