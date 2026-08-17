# go-macos.github.io

The landing page for the [go-macos](https://github.com/go-macos) organisation —
pure-Go (`CGO_ENABLED=0`) access to macOS system APIs through
[ebitengine/purego](https://github.com/ebitengine/purego).

Live at <https://go-macos.github.io/>.

## What is here

A single-page Hugo site, deliberately small:

| Path | Purpose |
| --- | --- |
| `hugo.toml` | site config **and** the repository cards (`params.repos`) |
| `layouts/index.html` | the whole page — markup, CSS and the theme toggle |
| `content/_index.md` | front matter only; the prose lives in the layout |
| `static/img/logo.svg` | the 88px organisation mark |
| `static/favicon.svg` | the same mark as the tab icon |

To add or change a repository card, edit the `[[params.repos]]` block in
`hugo.toml`; nothing in the layout needs touching. Each card states the mechanism
the package uses to reach the OS (`reach`) and how many repositories in the fleet
require it (`used_by`) — that count is measured from `go.mod` files, so update it
when it changes rather than rounding it up.

## Theme

Light, dark and system, in that cycle, with **system as the default** — the
saved choice is applied before first paint so there is no flash.

## Building it

```sh
hugo server -D      # local preview on :1313
hugo --minify       # what CI builds
```

CI (`.github/workflows/deploy.yml`) builds with Hugo extended and publishes
through the GitHub Pages artifact flow on every push to `main`.

BSD-3-Clause.
