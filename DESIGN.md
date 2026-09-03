# Design

## Repository shape

A standalone repository, **not** a GitHub fork. `upstream` is
`flathub/org.gnome.Fractal` as a plain git remote, with its push URL
deliberately broken. Full Flathub history is preserved, so `git merge
upstream/master` has a real merge base.

Not a fork because in a fork `gh pr create` and the web UI default the PR base to
the *upstream* repository. Automation must never be one flag away from opening a
pull request against Flathub.

```
main ──┬── Flathub history (merged from upstream/master)
       └── tray-icon.patch, workflows, README, DESIGN, manifest delta
```

## Manifest delta

* `--talk-name=org.kde.StatusNotifierWatcher` in `finish-args`. The `talk` policy
  is bidirectional, so it also covers the host reading our item properties and
  DBusMenu.
* **No `--own-name`.** A sandboxed app cannot own the
  `org.kde.StatusNotifierItem-$PID-$ID` well-known name the specification asks
  for, so the patch calls `ksni`'s
  `disable_dbus_name(ashpd::is_sandboxed())` and registers under the connection's
  unique bus name. Same approach as Chromium and the Nextcloud client Flatpak.
* `tray-icon.patch` as a `patch` source, placed **last** in the `sources` array —
  away from the `tag`/`commit` lines upstream rewrites every release, to minimise
  merge conflicts.
* `fractal-cargo-sources.json` regenerated, because the build vendors crates
  offline. `ksni` is the only added crate; it reuses Fractal's existing `zbus 5`,
  `tokio`, `serde` and `futures-util`.

## Patch

New code is `src/tray/{mod.rs,linux.rs}`, following the platform-split idiom of
`secret` and `location`: a `TrayIconExt` trait, a Linux backend, and a no-op stub
elsewhere. `TrayIcon` is a GObject owning the `ksni` service handle and
aggregating notification counts across sessions via `ExpressionListModel`.

Two details that are load-bearing:

* `Application::present_main_window()` finds the window through `windows()`, not
  `active_window()`. `active_window()` is unreliable for a hidden window, and
  returning `None` there would silently construct a *second* main window.
* `quit_application()` sets `is_quitting` before closing the window, so the close
  handler saves state and then genuinely quits rather than hiding.

The window is only ever hidden when a tray icon actually registered
(`should_hide_main_window()`), so a missing SNI host degrades to stock behaviour
rather than an unreachable process.

Ref identity: app ID `org.gnome.Fractal`, branch `stable` — the same ref as
Flathub, making this a drop-in origin. Only the `.desktop` `Name` is suffixed;
`metainfo.xml` is untouched because it gains release notes every version and
would raise conflict odds for no benefit. Non-English locales still show
"Fractal", since `Name` is translated from `po/`.

## Build and publish

`build.yml`, in `ghcr.io/flathub-infra/flatpak-github-actions:gnome-50` with
`--privileged`.

The image, rather than plain `ubuntu-latest`, is load-bearing. Ubuntu 24.04 ships
**flatpak-builder 1.4.2**, under which meson's default `libdir` becomes `lib64`,
so libshumate installs `shumate-1.0.pc` to `/app/lib64/pkgconfig` — off
pkg-config's path — and the `fractal` module then fails with `Dependency
"shumate-1.0" not found`. The image carries **1.4.9**, which yields `/app/lib`.
The manifest is left byte-identical to Flathub's rather than papered over with a
`--libdir` config-opt: this build should be what Flathub builds, plus the patch.

The image carries the base SDK only, so the workflow installs
`org.freedesktop.Sdk.Extension.rust-stable//25.08` explicitly — flatpak-builder
does not auto-install `sdk-extensions`. The branch is **25.08, not 50**:
`org.gnome.Sdk//50` declares `version = 25.08` on its
`org.freedesktop.Sdk.Extension` point, and no `rust-stable//50` exists.

Runner disk is not a constraint despite GitHub documenting "14 GB SSD" — current
`ubuntu-latest` runners report 145 GB total with ~86 GB free, against a 5.8 GB
image and a ~5 GB build.

* `pull_request`: build only.
* `push` to `main`: build, GPG-sign, publish.

x86_64 only. No build cache: GitHub evicts caches after 7 days unused while
releases are months apart, so it would be cold every time — and the dominant cost
is `cargo build --release` of one opaque module, which caching cannot touch
without `sccache` and a manifest that diverges from the one Flathub can build.
The 10 GB cache-service quota is a separate limit from runner disk, so ample disk
does not change this.

Measured, not estimated: a full run is **~30 minutes** — 79 s to pull the 5.8 GB
image, 19 s for the SDK extension, 27 m 18 s for all four modules including the
cargo release build, ~45 s to bundle, sign and publish. Free on a public
repository. At that duration a cold restore of ~5 GB of cache could plausibly
cost more than it saves.

Publish is inlined shell rather than a third-party action: `build-export` →
`build-update-repo --prune --prune-depth=0` → `index.flatpakrepo` →
`upload-pages-artifact` / `deploy-pages`. No `--generate-static-deltas`: deltas
exist to speed transfers *between* commits, and only one commit is ever kept, so
they would inflate the repo for nothing. Latest commit
only; rollback is served by `.flatpak` bundles attached to releases, which is
both simpler and more useful than ostree history a reader would have to pin by
commit.

## Tracking upstream

`poll.yml`, triggered by **`workflow_dispatch` only**. GitHub's 60-day
inactivity auto-disable applies solely to workflows with a `schedule` trigger, so
having no `schedule` removes that failure mode rather than working around it. It
also avoids the documented `schedule` behaviour where runs "can be delayed during
periods of high loads" and "some queued jobs may be dropped".

cron-job.org fires the dispatch daily. The poll compares `upstream/master`
against `HEAD` — no state store, because the merge base *is* the state.

On new upstream commits:

1. `git merge upstream/master`. This is what carries runtime and module bumps.
   A scripted bump of only the pinned `fractal` tag would have built Fractal 14
   against the GNOME 49 runtime and libshumate 1.5.
2. Full-clone Fractal at the new tag (not `--depth 1`, so `git apply -3` has the
   preimage blobs), apply `tray-icon.patch`.
3. Regenerate `Cargo.lock` with the **SDK's** cargo. It is applied with
   `--exclude=Cargo.lock` and regenerated rather than merged, because the
   lockfile changes every release and its diff would conflict every time.
   Regeneration must use the SDK's own cargo: Fractal is edition 2024 →
   resolver 3 → the running toolchain's version affects dependency resolution,
   so a distro cargo would produce a different lockfile.
4. Regenerate `fractal-cargo-sources.json`.
5. Open a PR with a PAT, **not** `GITHUB_TOKEN` — PRs opened with the default
   token do not trigger workflows, so the build would never run.

If `git apply` conflicts, `claude-code-action@v1` attempts the resolution and
opens a **draft** PR stating what it could not reconcile. It is instructed not to
invent a resolution it is unsure of: a green Flatpak build is a weak signal about
behaviour, and this is a client whose blast radius includes E2EE device state.

A human always merges. Merging publishes.

## Monitoring

The pipeline's dominant failure mode is silence, and GitHub has no "workflow has
not run in N days" signal. So `poll.yml` pings a healthchecks.io check on every
completion, with a `failure()` job hitting `/fail`. Period 24 h, grace 12 h.
Because the dispatch is unconditional and daily, that ping is a daily end-to-end
proof that cron → GitHub → workflow all still work.

Note healthchecks.io deletes accounts after a year with no *logins* — pings do
not reset that clock.

## Secrets

| Name | Purpose |
| --- | --- |
| `GPG_PRIVATE_KEY` | Single-use RSA-4096 signing key, generated for this repo alone, no expiry |
| `PR_TOKEN` | Fine-grained PAT, `contents`+`pull-requests` write, this repo — so PRs trigger builds |
| `ANTHROPIC_API_KEY` | Conflict-resolution agent |
| `HEALTHCHECKS_URL` | Dead man's switch ping URL |

A separate fine-grained PAT scoped to `actions: write` on this repo alone lives at
cron-job.org, so a leak from a free third party buys only the ability to trigger a
build.

The signing key has **no passphrase**, and its fingerprint is derived in CI from
the imported key rather than stored. A passphrase held in the same secret store as
the key it protects buys nothing; the public half is committed as
`fractal-tray.gpg` so it is auditable, and CI base64-encodes it into
`index.flatpakrepo`. The key does not expire, because expiry would break client
verification at an arbitrary future date for no security gain on a
single-purpose key.

## Inherited from Flathub, deliberately changed

`.github/workflows/update_sources.yaml` is **deleted**. It triggers on pull
requests touching the manifest and regenerates `fractal-cargo-sources.json` from
a pristine clone of the pinned tag — i.e. from the *unpatched* `Cargo.lock` —
which would drop `ksni` from the vendored crates and break the offline build. It
also pushes to the PR head.

Its trigger is scoped to the `master` and `beta` branches, so on our `main`
default it would not fire as things stand. It is removed as a latent hazard, not
an active one: anything that renamed the default branch or added a `master` would
arm it. A later upstream edit to the file surfaces as a delete/modify conflict;
keep it deleted.

`update-cargo-sources.sh` is kept as upstream's reference tool but must not be run
here for the same reason: our vendored list must come from the *patched*
lockfile. `track-upstream.yml` does that correctly.

## Ruled out

* **OCI / GHCR remote.** Flatpak requires a non-standard `/index/static` endpoint
  from `flatpak-oci-specs`; ghcr.io does not serve it. `flatpak install
  docker://…` works on flatpak ≥ 1.17 but creates an origin with no URL — bundle
  equivalent, no `flatpak update`.
* **A GitHub fork.** PR base default, above.
* **`schedule` triggers.** 60-day auto-disable; upstream went 224 days between
  Fractal 13 and 14.
* **release-monitoring.org / newreleases.io.** Both watch releases or tags;
  `flathub/org.gnome.Fractal` has zero of each. Anitya has no outbound webhooks
  and tracks upstream GNOME GitLab tags, not Flathub's integration.
* **Claude routines / Cowork scheduling as the trigger.** Research preview,
  1-hour floor, undisclosed daily run cap, no SLA, and a green run does not imply
  the task succeeded.
* **Self-hosted runner or cache server.** Self-hosted runners on public repos let
  fork PRs execute code on the host; the cache would only accelerate the modules
  that are not the bottleneck.
* **`andyholmes/flatter`.** node20, one release, no functional commits in six
  months, and its incremental-repo feature relies on a cache that is cold at this
  cadence. Read as a reference for `index.flatpakrepo`; not depended on.
