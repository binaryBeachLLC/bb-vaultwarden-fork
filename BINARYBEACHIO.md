# bb-vaultwarden-fork — binarybeachio Vaultwarden fork

This is the binarybeachio fork of [Vaultwarden](https://github.com/dani-garcia/vaultwarden) (an unofficial Bitwarden-compatible server in Rust). The fork exists for two reasons, both implemented entirely in build configuration with zero `.rs` source touched:

1. **Enable the upstream `s3` Cargo feature** so `DATA_FOLDER` can point at Cloudflare R2 instead of a local Docker volume. The feature is already in upstream's source — the official `vaultwarden/server` image just doesn't ship it. We flip one build flag.
2. **Rebrand the SSO login button** "Use single sign-on" → "Sign in with BinaryBeach.io" in the bundled web-vault's English locale, per the binarybeachio cross-fork convention ([`docs/architecture/customizations.md` → "User-visible SSO label"](https://git.binarybeach.io/binarybeach/binarybeachio/src/branch/main/docs/architecture/customizations.md)).

If you're reading this on a future merge from upstream, the [Refresh from upstream](#refresh-from-upstream) section is the playbook.

---

## Upstream

| Field | Value |
|---|---|
| Project | Vaultwarden (Bitwarden-compatible server in Rust, formerly bitwarden_rs) |
| Upstream repo | https://github.com/dani-garcia/vaultwarden |
| Upstream default branch | `main` |
| Currently integrated upstream version | **1.35.8** (released 2026-04-25 by upstream) |
| License | [AGPL-3.0](https://github.com/dani-garcia/vaultwarden/blob/main/LICENSE.txt). The push-mirror to `github.com/binarybeachllc/bb-vaultwarden-fork` (public) plus the public Forgejo repo at `git.binarybeach.io/binarybeach/bb-vaultwarden-fork` together satisfy AGPL §13's source-disclosure obligation for network use. |

`git log main..upstream` = upstream changes I haven't pulled in
`git log upstream..main` = binarybeachio's customizations

## Why we forked

### Patch 1 — `s3` Cargo feature (R2 attachments)

Upstream Vaultwarden's source already includes a complete S3-compatible storage backend via [Apache OpenDAL](https://opendal.apache.org/), gated behind a Cargo feature called `s3` (see `Cargo.toml`):

```toml
s3 = ["opendal/services-s3", "dep:aws-config", "dep:aws-credential-types",
      "dep:aws-smithy-runtime-api", "dep:anyhow", "dep:http", "dep:reqsign"]
```

When the feature is on, `DATA_FOLDER` (and downstream `ATTACHMENTS_FOLDER`, `SENDS_FOLDER`, `ICON_CACHE_FOLDER`, RSA key path) accept `s3://bucket/prefix` URIs. AWS credentials resolve through the standard chain, so Cloudflare R2 works by setting `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` + `AWS_ENDPOINT_URL=https://<account>.r2.cloudflarestorage.com`.

The official `docker.io/vaultwarden/server` image is built with `--features sqlite,mysql,postgresql,enable_mimalloc` — no `s3`. So R2 wiring is a build-flag flip, not a code change. We considered three lighter alternatives:

- **Use the upstream image, mount a Docker volume, restic-backup it to R2.** Works but doubles the storage (volume + R2), and recovery has a fault between volume corruption and the next backup. Not a *bad* posture, but worse than native S3 when native S3 is one ARG away.
- **Pre-compile a custom `vaultwarden` binary outside Docker and inject it into the upstream image.** Same surface as enabling the feature in the upstream Dockerfile; just less reproducible and harder to refresh on upstream version bumps.
- **Wait for upstream to ship `s3` enabled by default.** Upstream's stance is that the image is the conservative default; users who want S3 bring their own build. Reasonable for them, doesn't change our path.

The patch is one line in the build args (per OS), in three files. Total source diff: zero `.rs` files. Cost-per-upstream-merge: near zero (build-arg syntax is stable; new Cargo features upstream might add only require us to keep our existing flag list).

### Patch 2 — SSO button label rebrand

Per the binarybeachio cross-fork convention, every SSO button across our services reads "Sign in with BinaryBeach.io", not the underlying IdP's name. Vaultwarden's web-vault (a Bitwarden-clone JS bundle) ships an `i18n` dictionary at `/web-vault/locales/<lang>/messages.json`. The relevant key is:

```json
"useSingleSignOn": { "message": "Use single sign-on" }
```

We rewrite the *English* `message` value to "Sign in with BinaryBeach.io" with a `sed` step in the Dockerfile's runtime stage, after the web-vault is `COPY`ed in. Other locales are unchanged (their translations are different strings; if a tenant ever needs a non-English rebrand we'll cross that bridge then).

The patch is bracketed by `grep -q` checks both pre- and post-edit, so the build fails fast if upstream restructures the i18n key — surfacing the breakage instead of silently shipping the upstream label.

### Why not contribute upstream?

The `s3` feature flag is already upstream's own design — they intentionally don't enable it in the default image. Contributing "enable s3 by default" would be lobbying for a different design decision, not a feature.

The button label rebrand is binarybeachio-specific branding; not generally useful upstream. Carry locally.

## What's customized

Three files, one .j2 template + the two Dockerfiles it generates. Total: ~30 lines added across the three files, zero `.rs` source touched.

| File | Change | Lines | Conflict risk on upgrade |
|------|--------|-------|--------------------------|
| `docker/Dockerfile.j2` | (a) Append `,s3` to `ARG DB=` for both alpine and debian branches. (b) Insert `RUN grep + sed + grep` rebrand step after `COPY --from=vault`. | +14 | **Low** — both edits sit in lines that rarely change; the rebrand RUN is purely additive. |
| `docker/Dockerfile.alpine` | Same edits as `.j2`, applied to the generated alpine variant. | +14 | **Low** — must stay in sync with `.j2`; if upstream regenerates from the template our edits regenerate too (we patch the template *and* the generated files). |
| `docker/Dockerfile.debian` | Same edits as `.j2`, applied to the generated debian variant. | +14 | **Low** — same as alpine. |

Files **not** changed (deliberately):
- Any `.rs` file in `src/` — zero source code modified. The `s3` feature is purely a Cargo build-flag activation; the SSO rebrand is a runtime-stage `sed` on a built artifact.
- `docker/DockerSettings.yaml` — version pins for the web-vault and Rust toolchain; we follow upstream's choices here. Bump only when refreshing from upstream.
- `Cargo.toml` / `Cargo.lock` — the `s3` feature already exists in upstream's manifest; we just enable it at build time.

**Inert when not used.** The `s3` feature only activates when `DATA_FOLDER` (or per-folder envs) starts with `s3://`. With local paths, the runtime is bit-for-bit identical to upstream's behavior. The SSO rebrand only renders in the English locale; non-English users see upstream's translations.

## Required runtime config

The fork **adds no new env vars**. Wiring R2 uses the *upstream-defined* envs:

| Var | Purpose | Our value |
|-----|---------|-----------|
| `DATA_FOLDER` | Root storage location; accepts `s3://bucket/prefix` when built with `s3` feature | `s3://binarybeach-vault/data` |
| `AWS_ACCESS_KEY_ID` | Standard AWS env (resolved by `aws-config` chain) | from `_shared/.env.r2-vault` (bucket-scoped token) |
| `AWS_SECRET_ACCESS_KEY` | Standard AWS env | from `_shared/.env.r2-vault` |
| `AWS_DEFAULT_REGION` | Required by AWS SDK; R2 ignores the value but the SDK won't proceed without one | `auto` |
| `AWS_ENDPOINT_URL_S3` | Cloudflare R2 endpoint (account-scoped) | `https://<account>.r2.cloudflarestorage.com` |
| `TMP_FOLDER` | **Must stay local** — used for upload buffering | `/tmp/vaultwarden` (default) |
| `TEMPLATES_FOLDER` | **Must stay local** — bundled HTML email templates | upstream default |

OIDC (consumed by upstream's native OIDC support, which landed in v1.35; no fork patch needed):

| Var | Purpose |
|-----|---------|
| `SSO_ENABLED` | Activate OIDC. Off until the Zitadel app is created (see operator runbook in `binarybeachio/docs/architecture/services-bootstrap.md`). |
| `SSO_AUTHORITY` | `https://auth.binarybeach.io` (Zitadel issuer; no `/.well-known/...` suffix). |
| `SSO_CLIENT_ID` / `SSO_CLIENT_SECRET` | From the Zitadel "Vaultwarden" application. |
| `SSO_AUDIENCE_TRUSTED` | Zitadel-specific — Zitadel includes its project id in the `aud` claim, which Vaultwarden's audience check rejects unless explicitly trusted. Set to a regex matching the project id. |
| `SSO_ONLY` | Off during initial deploy / break-glass-user provisioning; flipped on once SSO is verified end-to-end. |

Zitadel side — manual one-time setup (per the binarybeachio "beaten-path config over reconcilers" rule, no automation):

1. Zitadel dashboard → project `binarybeachio platform` → New Application → Web, **CODE flow**, **client_secret_post** auth method.
2. **Redirect URI:** `https://vault.binarybeach.io/identity/connect/oidc-signin`.
3. **Post Logout URI:** `https://vault.binarybeach.io/`.
4. Copy the generated Client ID + Client Secret + Project ID into `infrastructure/vaultwarden/.env` as `VAULTWARDEN_SSO_CLIENT_ID` / `VAULTWARDEN_SSO_CLIENT_SECRET` / `VAULTWARDEN_SSO_AUDIENCE_TRUSTED=^<project-id>$`.
5. Re-run `py infrastructure/_shared/bootstrap.py` to push the envs and trigger redeploy.

## Break-glass — two prongs

Vaultwarden's break-glass surface is structurally different from Plane/Mattermost/Forgejo and worth calling out: the convention's `maxwell.pauly@binarybeach.tech` admin **maps onto two independent recovery paths**, both of which survive an SSO outage:

1. **`/admin` panel (token-gated)** — the system-config surface. Authenticated by an Argon2id-hashed `ADMIN_TOKEN` (derived from `BB_ADMIN_PASSWORD` per `_shared/.env.bb-admin`). Independent of any user account, independent of SSO state. From `/admin` you can disable SSO, manage users, rotate config, etc.
2. **bb-admin user account (with master password)** — provisioned during the SSO_ENABLED-but-not-yet-SSO_ONLY window. The bb-admin signs up via SSO using the convention email, then sets the `BB_ADMIN_PASSWORD` as their Bitwarden master password. After `SSO_ONLY=true` is flipped, the master-password login form is hidden globally — but the bb-admin's master password remains valid; recovery is "use `/admin` token to flip SSO_ONLY off, log in with master password, fix things, flip SSO_ONLY back."

Procedure documented in `binarybeachio/docs/architecture/services-bootstrap.md` under the `## vaultwarden` section.

## Refresh from upstream

```bash
cd C:\Users\maxwe\GitHubRepos\bb-vaultwarden-fork
git fetch upstream
git fetch upstream --tags

# Optional — just look at what's pending:
git log main..upstream/main           # upstream changes since our last integration
git diff main..upstream/main -- docker/   # any docker-stage changes that might collide

# Sync the upstream mirror branch (never modified by us)
git switch upstream
git reset --hard upstream/main
git push origin upstream

# Integration branch
git switch main
git switch -c update/v<X.Y.Z>
git merge v<X.Y.Z>                    # merging the upstream tag, not the moving branch
# Resolve conflicts (likely zero — our patches are in three small spots upstream rarely touches)
# Once happy:
git switch main
git merge --ff-only update/v<X.Y.Z>
git branch -d update/v<X.Y.Z>
git push origin main
git tag v<X.Y.Z>-mine.1               # see Tag scheme below
git push origin v<X.Y.Z>-mine.1

# Then on laptop: rebuild + push image (see Build below)
# Then in binarybeachio repo: bump VAULTWARDEN_IMAGE in infrastructure/vaultwarden/.env
# Then: py infrastructure/_shared/bootstrap.py to trigger Coolify to pull the new image
```

If the SSO rebrand step fails on next refresh (the `grep -q` pre-check), upstream restructured the web-vault's i18n. Inspect `/web-vault/locales/en/messages.json` in the v2026.x web-vault image to find the new key shape, update the `sed` pattern in all three Dockerfiles + the .j2 template, rebuild.

## Build — image registry and tagging

**Tag scheme** per binarybeachio architecture §6 #7: `<upstream-version>-mine.<n>`. So for the v1.35.8 base patched once: `v1.35.8-mine.1`. `mine.<n>` resets to `mine.1` on every upstream version bump; increments per local rebuild within the same upstream version.

**Image registry: our Forgejo registry at `git.binarybeach.io/binarybeach/bb-vaultwarden-fork`** — same as every other Path B fork in binarybeachio. Image stays **private** by default; the source repo is what's public (AGPL §13's source-disclosure target is the source, not the binary).

### Build commands

The official upstream `docker-bake.hcl` builds multi-arch images with full BuildKit caching. For our local build we use the alpine variant first (smaller, faster) and fall back to debian if any AWS Rust crate hits a musl issue.

```bash
# from C:\Users\maxwe\GitHubRepos\bb-vaultwarden-fork (laptop)
# Auth via the same Forgejo PAT used to push source — convention.
set -a; source ../binarybeachio/infrastructure/_shared/.env.forgejo; set +a
echo "$FORGEJO_ADMIN_PAT" | docker login git.binarybeach.io \
    -u "$FORGEJO_ADMIN_USER" --password-stdin

# Single-arch (amd64) build — matches our Hetzner VPS
docker build \
  -f docker/Dockerfile.alpine \
  --build-arg VW_VERSION=v1.35.8-mine.1 \
  -t git.binarybeach.io/binarybeach/bb-vaultwarden-fork:v1.35.8-mine.1 \
  -t git.binarybeach.io/binarybeach/bb-vaultwarden-fork:latest \
  .

docker push git.binarybeach.io/binarybeach/bb-vaultwarden-fork:v1.35.8-mine.1
docker push git.binarybeach.io/binarybeach/bb-vaultwarden-fork:latest
```

Build is heavy: full Rust dependency tree (incl. AWS SDK + opendal) + cross-compile + web-vault asset copy. Expect 15–35 minutes on a laptop with cold caches, 4–10 minutes warm. Disk: ~4 GB intermediate. Bandwidth: ~120 MB push.

If the alpine build fails on the AWS-crate side (musl libc edge cases occasionally surface in `aws-config` / `reqsign` despite both being pure-Rust), retry with `-f docker/Dockerfile.debian` — same args, larger final image (~250 MB vs. ~50 MB), same runtime behavior. Document in this file's "Currently integrated upstream version" row if the debian variant becomes the standing build.

## License compliance

Vaultwarden is AGPL-3.0. AGPL §13 requires that anyone interacting with a modified version over a network has access to the **Corresponding Source**. Source code, not container images. Our image distribution can stay private (it does — the image lives in our private Forgejo registry); only the source needs to be publicly reachable.

Our compliance:

1. **Forgejo source** — `git.binarybeach.io/binarybeach/bb-vaultwarden-fork` is a public-readable repository (Forgejo `DEFAULT_PRIVATE=public`). Anyone hitting `git.binarybeach.io` can browse this fork's source without authentication.
2. **GitHub source mirror** — Forgejo push-mirror to `github.com/binarybeachllc/bb-vaultwarden-fork` provides off-site backup AND a publicly-discoverable source location even if our Forgejo is unreachable. The mirror covers source only; the image is not on GitHub.
3. **In-product source link** — TODO when we surface tenant-facing "About" / footer copy, add a link to https://git.binarybeach.io/binarybeach/bb-vaultwarden-fork. Not a v1 blocker since the source is already publicly available at both URLs above.

## Test plan (manual, until CI exists)

1. **Local image build smoke** — `docker build -f docker/Dockerfile.alpine ...` produces a tagged image. The pre/post `grep` of the SSO label rewrite either passes (build OK) or fails the build with a clear "i18n key not found" line.
2. **`s3` feature smoke** — `docker run --rm git.binarybeach.io/binarybeach/bb-vaultwarden-fork:v1.35.8-mine.1 /vaultwarden --version` should print the version cleanly. Then `docker run --rm <image> sh -c 'env | grep -i s3'` won't show anything (we don't bake creds in), but the binary's `--help` output should show no errors. (Vaultwarden has no built-in "show enabled features" subcommand; the proof is the binary not erroring on `s3://`-URI parsing at runtime.)
3. **Image push** — tags push to `git.binarybeach.io/binarybeach/bb-vaultwarden-fork`.
4. **Coolify deploy** — `infrastructure/vaultwarden/.env` `VAULTWARDEN_IMAGE` set to the fork tag → re-run `bootstrap.py` → Vaultwarden comes up healthy at `https://vault.binarybeach.io/`.
5. **R2 round-trip** — log in as bb-admin, upload an attachment to a test cipher, verify the file lands in `binarybeach-vault` R2 bucket via `wrangler r2 object list binarybeach-vault`.
6. **SSO button rebrand** — visit `https://vault.binarybeach.io/#/login`, scroll to the SSO section, confirm the button reads "Sign in with BinaryBeach.io" (browser locale = English).
7. **Break-glass** — verify both prongs: (a) `https://vault.binarybeach.io/admin` accepts the Argon2 token; (b) bb-admin user can log in with master password (assuming SSO_ONLY=false, or via the recovery sequence if SSO_ONLY=true).
