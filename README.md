# Baram Plugin Registry

Plugin registry and distribution channel for [Baram](https://github.com/sayinel/baram),
served via GitHub Pages.

- `index.json` — the registry index the app's marketplace fetches
- `plugins/*.zip` — plugin packages (SHA-256 verified at install time)

## Policy

This registry currently hosts **first-party Baram plugins only**. Community
submissions are not accepted yet. Do not open PRs adding plugin entries;
they will be closed.

## How it is updated

Content is pushed exclusively by the `plugin-release.yml` workflow in the
main Baram repo: pushing a tag `plugin-<dir>-v<version>` there builds the
plugin, packages the ZIP, computes its SHA-256, and commits the ZIP plus an
updated `index.json` here. Manual commits are reserved for maintenance.
