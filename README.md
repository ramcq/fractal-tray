# Fractal, with a tray icon

An unofficial rebuild of the [Fractal](https://gitlab.gnome.org/World/fractal)
Matrix client with a system tray icon, so it can keep syncing and notifying with
its window closed.

> **This is not Fractal.** It is a personal patched rebuild, not affiliated with
> or endorsed by the Fractal project or GNOME. Do not report bugs from this build
> to them. Fractal itself is at
> [gitlab.gnome.org/World/fractal](https://gitlab.gnome.org/World/fractal), and
> the official Flatpak is [on Flathub](https://flathub.org/apps/org.gnome.Fractal).

The repository is the [Flathub manifest for
`org.gnome.Fractal`](https://github.com/flathub/org.gnome.Fractal) plus
`tray-icon.patch`. Upstream is tracked as a git remote and merged in, so each
Fractal release brings Flathub's own integration work — runtime bumps, module
versions — along with it.

## What it adds

* A `StatusNotifierItem` tray icon, via the [`ksni`](https://crates.io/crates/ksni) crate.
* Closing the window hides it instead of quitting. The session keeps syncing, so
  notifications keep arriving.
* Left-click toggles the window; the tray menu has *Show Fractal* and *Quit*.
* The icon takes the `NeedsAttention` state and reports a count in its tooltip
  while any room has pending notifications, summed across all logged-in sessions.
* A *Run in Background* toggle in the primary menu, backed by a new
  `run-in-background` GSettings key (default on). Turning it off removes the tray
  icon and restores the stock close-quits behaviour.
* The visible name is "Fractal (Tray)", so you can tell which build is running.
  The app ID is unchanged.

## Requirements

The tray needs a StatusNotifierItem host on the session bus:

* **KDE Plasma** — works as-is.
* **GNOME Shell** — needs an AppIndicator extension. Many distributions ship one
  but leave it disabled:

  ```sh
  gnome-extensions enable ubuntu-appindicators@ubuntu.com   # or
  gnome-extensions enable appindicatorsupport@rgcjonas.gmail.com
  ```

With no host present the item cannot register. Fractal logs
`Could not register the tray icon: …` and behaves exactly like the unpatched
build — closing the window quits. That is deliberate: the window is never hidden
unless there is a tray icon to bring it back from.

## Install

```sh
flatpak install --user https://ramcq.github.io/fractal-tray/org.gnome.Fractal.flatpakref
```

One command: it adds the remote, pulls the GNOME runtime from Flathub if needed,
and installs the app. Updates then arrive through `flatpak update` as usual.

The remote is added as a **filtered origin** — flatpak marks it
`no-enumerate` with `xa.main-ref` pinned to this one ref and `xa.prio=0`. So it
can only ever offer this build, it is never consulted when resolving some other
app's runtime, and it does not clutter your list of software sources.

This publishes app ID `org.gnome.Fractal` on branch `stable` — the same ref as
Flathub — so it is a drop-in replacement and keeps your existing session and
E2EE device identity (app data lives in `~/.var/app/org.gnome.Fractal`, which is
keyed on app ID). A `--user` install takes precedence over a system one, so
Flathub's copy can stay installed as a fallback.

To go back:

```sh
flatpak uninstall --user org.gnome.Fractal
```

Single builds are also attached to [releases](../../releases) as `.flatpak`
bundles, if you would rather not add a remote or want to pin an older build.

## Building it yourself

```sh
flatpak install --user flathub org.gnome.Sdk//50 \
    org.freedesktop.Sdk.Extension.rust-stable//25.08
flatpak-builder --user --force-clean --install build-dir org.gnome.Fractal.json
```

## Upstreamability

Low. GNOME does not ship tray icons and Fractal has no tray or background mode.
This is a patch to carry, not a proposal.

See [DESIGN.md](DESIGN.md) for how the repository and its automation are put
together.
