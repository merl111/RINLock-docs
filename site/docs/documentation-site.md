# Maintaining the documentation site

The public site is [merl111.github.io/RINLock-docs](https://merl111.github.io/RINLock-docs/).
It is built from the existing repository Markdown with MkDocs and Material.
The generated site lives in the public `merl111/RINLock-docs` repository because
the account's current plan does not support Pages from the private source repository.
The source repository remains private. Source and release links still require
repository access. There is no runtime dashboard or telemetry endpoint on Pages;
RINLock's dashboard remains a terminal application.

## Local build and preview

From a source checkout, install Python 3.12+ and create an isolated environment:

```sh
python3 -m venv /tmp/rinlock-docs-venv
/tmp/rinlock-docs-venv/bin/pip install -r requirements-docs.txt
make docs PYTHON=/tmp/rinlock-docs-venv/bin/python
```

The build stages selected content under `.docs-build/content`, then writes the
complete static site to `.docs-build/site`. Build warnings, missing internal
links and broken anchors fail the build. The artifact checker also verifies
HTML resources, Markdown discovery and manifest digests. The output directory
is ignored by Git.

Preview at the same project path as Pages:

```sh
mkdir -p /tmp/rinlock-docs-preview
ln -sfn "$PWD/.docs-build/site" /tmp/rinlock-docs-preview/RINLock-docs
python3 -m http.server 8765 --bind 127.0.0.1 --directory /tmp/rinlock-docs-preview
```

Open `http://127.0.0.1:8765/RINLock-docs/`. Rebuild after edits. Search, page navigation,
Markdown downloads and theme controls work without a backend. Canonical and
HTML discovery URLs use the public address; visible Markdown links are relative
and also work in the local preview.

Mermaid code fences render as diagrams through Material's native integration.
The theme loads the Mermaid runtime from its CDN when a diagram is present;
JavaScript and access to that CDN are required. Raw Markdown and the agent bundle
retain the original diagram source. The artifact checker verifies that Mermaid
fences are marked for rendering, and diagram changes should also be checked in
the browser in both color themes.

## Publishing boundary

`docs-site/pages.json` is the explicit publication list. It contains Markdown
pages and specific configuration examples, validation records and screenshots.
The builder also includes the reviewed theme override and stylesheet, generated
indexes and the theme's own static assets. It does not copy the repository tree,
source code, Git history, credentials, databases, local agent state or telemetry.
Links to source files outside the list point to the private repository at the
documented commit.

To add a guide, update the manifest and `mkdocs.yml` navigation, then build and
check it. Review every newly listed file for public suitability. Screenshots and
validation examples must contain synthetic or deliberately disposable data.
Removing something from the site does not erase copies already downloaded.

## Agent formats

The build produces these from the same staged Markdown used for the website:

- `llms.txt`: a short index linking directly to Markdown and configuration files.
- `llms-full.txt`: current guides in one Markdown document; historical stage 1/2
  notes are excluded from this bundle and remain individually accessible.
- A `.md` sibling for each HTML page, advertised by `rel="alternate"` and a
  visible **Read as Markdown** link. Relative links in these copies lead to
  other Markdown files or downloadable assets.
- `agent-manifest.json`: schema version 1, project version, Git revision,
  `source_dirty`, site URL and resource entries with URL, title, type, byte length
  and SHA-256 digest. It includes both indexes but not its own digest.

The full bundle rewrites local links to absolute Markdown URLs, so links retain
their meaning when copied into an agent's context. Manifest digests identify
bytes, not a trusted signature. Use the HTTPS origin and source revision to
establish provenance. The manifest is generated, not maintained separately.

## GitHub Actions and Pages

The private repository's `.github/workflows/docs.yml` builds and checks docs on
pull requests and pushes to `main`, and supports manual dispatch. Successful
main-branch builds retain a `docs-site` artifact for review. No cross-repository
token is stored in CI, so a private-source push alone does not publish the site.

After committing and pushing reviewed source changes, publish a snapshot from
the clean checkout with an authenticated `gh` CLI and Git push access:

```sh
make publish-docs PYTHON=/tmp/rinlock-docs-venv/bin/python
```

This rebuilds and checks the site, clones `merl111/RINLock-docs` into a temporary
directory, replaces its generated `site/`, installs the Pages workflow, and
commits and pushes the snapshot. It refuses a dirty source tree and verifies the
destination repository name and visibility. The commit records the private
source revision. No private Git history or application code is copied. Publication
commits use the authenticated GitHub account's no-reply email address.

The public repository's `.github/workflows/pages.yml` uploads the static site and
deploys it with `pages: write` and `id-token: write` in the `github-pages`
environment. Its template is `docs-site/pages-workflow.yml` in the source repo.
Actions are pinned to immutable commits and Python build dependencies are pinned
in `requirements-docs.txt`. The public repository's **Settings → Pages → Build
and deployment** source must be **GitHub Actions**. Keep the application repository
private. See [GitHub's Pages documentation](https://docs.github.com/en/pages/getting-started-with-github-pages/creating-a-github-pages-site).

The site documents the most recently published development snapshot, not the
latest release tag or necessarily the newest private-source commit. Version and
commit appear on every page and in the agent manifest. To undo a bad publication,
revert the documentation change in the source repository, commit, and run the
publish command again. If the owner configures an environment approval gate,
that gate must be satisfied before deployment.

When changing site location, update `site_url` in `mkdocs.yml` and the public
links in the README and guides. The builder derives generated resource URLs
from `site_url`; the artifact checker tests the configured project-path prefix.
