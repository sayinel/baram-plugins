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

## Versioning policy

Published versions are **immutable**: once `plugins/<id>-<version>.zip` is
served, its bytes never change. To ship a fix, bump the plugin version and
release again — never overwrite an existing ZIP. Re-serving changed bytes
under the same filename would break SHA-256 verification for clients that
fetched the index during the Pages CDN window (~10 min), and defeats the
point of pinned checksums.
