# Setup

The repository, workflows and signing key are in place. These are the steps that
need your accounts.

`GPG_PRIVATE_KEY` is already set. Its public half is committed as
`fractal-tray.gpg`, fingerprint `932D7EBA D3B8FF0C 086EDA85 4032A590 2C4428D7`.

## 1. `PR_TOKEN` — so rebase PRs get built

Pull requests opened with the default `GITHUB_TOKEN` **do not trigger
workflows**, so without this the rebase PR would never build.

Create a fine-grained PAT at
<https://github.com/settings/personal-access-tokens/new>:

* Repository access: **only** `ramcq/fractal-tray`
* Permissions: `Contents` → Read and write, `Pull requests` → Read and write
* Expiry: your call, but note the workflow fails loudly when it lapses

```sh
gh secret set PR_TOKEN --repo ramcq/fractal-tray
```

## 2. `ANTHROPIC_API_KEY` — conflict resolution only

A team API key from <https://console.anthropic.com/settings/keys>. Used only when
`tray-icon.patch` stops applying, so a handful of runs a year at most.

```sh
gh secret set ANTHROPIC_API_KEY --repo ramcq/fractal-tray
```

## 3. healthchecks.io — the dead man's switch

Sign up at <https://healthchecks.io/>, create one check:

* Schedule: **Simple**, Period `1 day`, Grace `12 hours` → alerts at 36 h
* Notification: your email, plus a second channel (an ntfy webhook to kappa) —
  their docs recommend two in case one lands in spam

```sh
gh secret set HEALTHCHECKS_URL --repo ramcq/fractal-tray   # https://hc-ping.com/<uuid>
```

Two things worth doing while you are there:

* **Email contact@healthchecks.io and ask for the free non-profit / open-source
  Business plan.** It is an advertised offer, costs nothing, and puts you in a
  support relationship rather than anonymous-free.
* **Set a yearly calendar reminder to log in.** Accounts with no *logins* for over
  a year get a warning email and are deleted 30 days later — pings do not reset
  that clock. The watchdog has its own watchdog.

## 4. cron-job.org — the daily trigger

A second, narrower PAT for this one, because it lives on a free third-party
service. Fine-grained, `ramcq/fractal-tray` only, permission `Actions` → Read and
write, and nothing else. A leak then buys an attacker only the ability to start a
build.

Create a job at <https://console.cron-job.org/>:

* URL: `https://api.github.com/repos/ramcq/fractal-tray/actions/workflows/track-upstream.yml/dispatches`
* Method: `POST`
* Headers:
  * `Accept: application/vnd.github+json`
  * `Authorization: Bearer <the actions-scoped PAT>`
  * `X-GitHub-Api-Version: 2022-11-28`
* Body: `{"ref":"main"}`
* Schedule: once daily, any time
* **Enable the job-failure email.** Jobs auto-disable after 25 consecutive
  failures, which is exactly what a rotated PAT would cause.

Expect HTTP `204 No Content` on success.

Verify by hand first:

```sh
gh workflow run track-upstream.yml --repo ramcq/fractal-tray
gh run list --repo ramcq/fractal-tray --workflow track-upstream.yml --limit 3
```

## 5. Switch your install over

You currently have the bundle installed as branch `master` from
`fractal-origin`, plus a `fractal-tray` remote that a verification step of mine
added. Clear both, then install from the flatpakref:

```sh
flatpak uninstall --user org.gnome.Fractal
flatpak remote-delete --user fractal-tray
flatpak install --user https://ramcq.github.io/fractal-tray/org.gnome.Fractal.flatpakref
```

The install adds the remote itself, as a filtered origin (`no-enumerate`, pinned
to this one ref). Note it will be named `fractal-origin`, which is also the name
the old bundle install used — hence removing that first.

Your session and E2EE device survive: app data lives in
`~/.var/app/org.gnome.Fractal`, keyed on app ID, which has not changed.
