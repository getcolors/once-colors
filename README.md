# once-colors

Infrastructure for the **www.getcolors.ai** website: one Oracle Cloud VM running
[ONCE](https://once.com), with DNS on Cloudflare, transactional mail on Resend,
and OpenTofu state in Cloudflare R2.

Everything is driven from a single desired-state file, `colors.yml`, applied by
the `./green` launcher. The website's source lives in a separate repository and
arrives here only as a container image reference.

## Prerequisites

| Tool | Used for |
| --- | --- |
| [babashka](https://babashka.org) (`bb`) | runs `./green` |
| [OpenTofu](https://opentofu.org) (`tofu`) | compute, DNS, and SMTP stages |
| `ansible-playbook` | server and local SSH-config stages |
| [direnv](https://direnv.net) | loads credentials from `.envrc.private` |
| `oci` CLI + `~/.oci/config` | Oracle Cloud authentication |

You also need the SSH private key matching `oci-ssh-authorized-keys` loaded in
`ssh-agent` — the compute stage and Ansible both connect to the server with it.

## Setup

```sh
direnv allow          # activates .envrc
./green describe      # read-only: should print the current stack status
```

`direnv allow` sources `.envrc`, which in turn sources `.envrc.private` if it
exists. That private file is **not** in the repository; create it with the
credentials below (get them from the respective dashboards or a teammate):

```sh
export COLORS_PAR_CLOUDFLARE_API_TOKEN=...
export COLORS_PAR_RESEND_API_KEY=...
export COLORS_PAR_RESEND_PASSWORD=...
export COLORS_PAR_R2_ACCESS_KEY_ID=...
export COLORS_PAR_R2_SECRET_ACCESS_KEY=...
```

Oracle Cloud is the exception: it authenticates through the profile named by
`oci-config-file-profile` in `colors.yml`, so it needs no variable here. That
profile is session-token based, so `oci session authenticate` (or
`oci session refresh`) must have been run recently — an expired token shows up
as an authentication failure when a compute stage runs.

## Usage

```sh
./green describe          # status: providers, server IP, per-app image + whether an update is available
./green build             # render the working tree under .colors/ — no cloud calls, no credentials needed
./green create --dry-run  # walk every stage, skip all side effects
./green create            # converge the real thing
./green delete            # tear it down (see below)
```

`build` and `--dry-run` work on a fresh checkout with no secrets at all, which
makes them the safe way to check a `colors.yml` edit. Validation reports every
problem at once and exits with status 2 before touching a provider.

`./green` can be run from any subdirectory: it walks up to find `colors.yml`.

### Changing the deployment

Edit `colors.yml`, then `./green create`. That is the whole loop — the working
tree under `.colors/` is generated output and is never edited by hand. Common
edits:

- **Ship a new image** — change `once.applications[].image`.
- **Resize the VM** — `oci-ocpus`, `oci-memory-in-gbs`, `oci-shape`.
- **Add an application** — append to `once.applications` with `host` and
  `image`; the DNS record and the ONCE app are both derived from it.

### Deleting

`./green delete` refuses while `compute-prevent-destroy: true` is set in
`colors.yml`. To actually destroy the server, override it for that one run:

```sh
COLORS_PAR_COMPUTE_PREVENT_DESTROY=false ./green delete
```

The flag is intentionally not disable-able by editing the file alone.

## How a deploy reaches the server

CI does not have shell access to the box. The Ansible stage creates a `deploy`
user whose authorized key (`deploy-pubkey` in `colors.yml`) is pinned to a
ForceCommand script accepting exactly one command:

```sh
ssh deploy@<server> sudo once update <host>
```

The host must already be a known ONCE application; anything else is rejected.
The matching private key never lives in this repository.

## Layout

```
colors.yml     desired state — the only file you normally edit
green          launcher (babashka); delegates to the pinned `once` library
.envrc         direnv config, secret-free, committed
.envrc.private credentials, gitignored, create locally
.colors/       generated OpenTofu + Ansible trees, gitignored — do not edit
```

Note that `.gitignore` ignores all dotfiles (`.*`) except `.envrc`, so a new
dotfile needs an explicit negation to be tracked.

See [CLAUDE.md](CLAUDE.md) for the workflow DAG, how stages pass values to each
other, and where the `once` library source lives.
