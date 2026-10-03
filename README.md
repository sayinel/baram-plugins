# Baram Plugin Registry

Plugin and theme registry and distribution channel for
[Baram](https://github.com/sayinel/baram), served via GitHub Pages.

- `index.json` — the first-party registry index the app's plugin marketplace
  and **Browse Themes** fetch
- `community.json` — the community list the marketplace fetches beside it
- `plugins/*.zip` — plugin and theme packages (SHA-256 verified at install
  time)
- `readme/*.md` — plugin READMEs the marketplace's detail view shows
- `revoked.json` and `revoked.json.sig` — the signed withdrawal list the app
  checks

## Policy

This registry serves two channels, one file each:

- `index.json` — first-party Baram plugins and themes, published by the Baram
  repository's `plugin-release.yml`. A theme entry has `"kind": "theme"`, no
  `trust`, and a `preview` palette for each of its modes.
- `community.json` — community plugins, **sandboxed only**, submitted by pull
  request (below). It does not take themes.

`main` is protected. The Baram repository's two publishing workflows
(`plugin-release.yml` and `revocation-publish.yml`) push to it with a deploy
key; every other change arrives through a pull request that passes the
`validate` check.

## Submitting a community plugin

1. Publish a GitHub Release of your plugin in a public repository you own
   personally, with the ZIP as an asset.
2. Open a pull request that adds exactly one file, `community/<id>.json`:

   ```json
   {
     "id": "hello-counter",
     "publisher": "octocat",
     "repo": "octocat/baram-hello-counter",
     "release": { "tag": "v1.0.0", "asset": "hello-counter-1.0.0.zip", "sha256": "<64 hex>" }
   }
   ```

3. The `validate` check verifies the submission. Until an id's first version
   is published, every submission for it waits for a maintainer's review. A
   later version merges automatically when it asks for no new capability and
   keeps the same name, author, description, icon, homepage, publisher and
   repository.
4. After the merge, `publish-community.yml` copies the ZIP into `plugins/` and
   adds the entry to `community.json`. Baram never downloads from your
   repository.

Full guide: https://baram.ing/en/docs/plugin-dev/community-registry/

## How it is updated

- `index.json` and first-party archives: `plugin-release.yml` in the Baram
  repository, when a `plugin-<dir>-v<version>` or `theme-<dir>-v<version>` tag
  is pushed there. Both jobs run in that repository's `registry-publish`
  environment and wait for a maintainer's approval; a theme release also stops
  unless the ZIP it built matches the SHA-256 recorded in the theme's
  `SHA256SUMS`.
- `community.json`, community archives and READMEs: `publish-community.yml`
  here, after a submission merges, and weekly.
- `revoked.json`: `revocation-publish.yml` in the Baram repository.

To withdraw a plugin or a theme, open an issue. Withdrawal goes through the
signed revocation list; published archives are never deleted.

## Versioning policy

Published versions are **immutable**: once `plugins/<id>-<version>.zip` is
served, its bytes never change. To ship a fix, bump the plugin's or theme's
version and release again — never overwrite an existing ZIP. Re-serving
changed bytes under the same filename would break SHA-256 verification for
clients that fetched the index during the Pages CDN window (~10 min), and
defeats the point of pinned checksums.

## Maintainer runbook

`publish-community.yml` says in its log what it did and why it stopped. These
cases need a person:

- **`✗ aborted:` saying a squash merged but its tree was not checked, or was
  not the validated tree.** main now holds a commit nothing validated, and
  Pages may serve it already: GitHub documents that a push made with
  `GITHUB_TOKEN` triggers no Pages build, but a test repository whose Pages
  site is also built from a branch measured one that did. Usually main moved
  because a submission that `validate.yml`'s merge job auto-merged landed at
  the same moment, and the combined tree is fine. Check it at once:
  `gh workflow run validate.yml --repo sayinel/baram-plugins`. If it passes,
  request a Pages build so the served copy is current:
  `gh api -X POST repos/sayinel/baram-plugins/pages/builds`. If it fails,
  revert that commit by pull request right away, but keep any archive it added
  under `plugins/`: `validate` refuses a pull request that deletes one. The
  next run publishes every `community/<id>.json` that `community.json` is
  behind, so delete that descriptor in the same pull request unless the
  release should come back.
- **A run that was cancelled or timed out.** It prints nothing about where it
  stopped, and it may or may not have merged a squash. Its branches are
  `community-publish/<run id>-<attempt>-<n>`, where the run id is the number
  in the run's URL (`…/actions/runs/<run id>`).
  1. Find its pull requests:
     `gh pr list --repo sayinel/baram-plugins --state all --limit 1000 --search 'head:community-publish/<run id>-' --json number,state,headRefName`.
     The search matches branch names by prefix, so keep the trailing `-`:
     without it, run 123 would also match run 1234.
  2. If one of them is `MERGED`, a squash landed: check it as in the first
     case.
  3. Run `gh workflow run publish-community.yml --repo sayinel/baram-plugins`.
     Its sweep closes those still open and withdraws their `validate` status.
     Until then, an open `community-publish/` pull request can carry a green
     `validate` and look mergeable.
- **Any other `✗ aborted:` line.** Fix the cause, then
  `gh workflow run publish-community.yml --repo sayinel/baram-plugins`. An
  aborted run requests no Pages build, so if a release it delivered is still
  not served after the next run, request the build as above.
- **A line asking to post `state=error context=validate` by hand.** The job's
  `success` status could not be withdrawn, and it satisfies the required check
  for any pull request whose head is that commit. On each commit concerned:
  `gh api -X POST repos/sayinel/baram-plugins/statuses/<sha> -f state=error -f context=validate`.
  When the line names a pull request rather than a commit ("so only its head
  … was withdrawn; post state=error context=validate on its other commits by
  hand"), list its commits with
  `gh api repos/sayinel/baram-plugins/pulls/<n>/commits --paginate --jq '.[].sha'`
  (GitHub lists at most 250 commits for a pull request there).
- **A line asking to close a pull request by hand.**
  `gh pr close <n> --repo sayinel/baram-plugins`, before anyone merges it.
- **Releasing an id that was merged but never published.** Until it is
  published, an id belongs to the author of the pull request that first added
  `community/<id>.json` since its last deletion. Merge a pull request that
  deletes that file: `validate` treats a pull request that only deletes
  descriptors as maintenance, labelled `needs-review` for a person to merge.
- **`community-publish/` branches belong to the publish job.** Every run first
  closes each open pull request from such a branch in this repository as a
  leftover of an earlier run. Do not open one by hand.
