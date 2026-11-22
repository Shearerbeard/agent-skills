# Publishing backends

One backend per artifact, recorded in its config.md under "Backend settings".
Both backends give stable URLs: a redeploy overwrites the same address, which
is what the refresh workflow depends on. Before the first publish of any
slug, confirm the target and visibility with the user - publishing pushes
content to a remote host, and a public URL is the default on both backends.

## github-pages (default)

A single dedicated GitHub repo serves every artifact, one directory per
artifact, Pages enabled on the default branch. URLs look like:

    https://<owner>.github.io/<repo>/<slug>/

**First-time setup (once per repo, verified per machine):**

1. Create the repo if it does not exist: gh repo create <owner>/<repo>
   --public (use --private only if the user's GitHub plan allows Pages on
   private repos; check before promising it).
2. Clone it to a stable local path (record that path in every config that
   uses it, e.g. ~/dev/artifacts), then commit and push an initial root
   index.html so the default branch exists - the Pages API rejects a source
   branch that has no commits.
3. Enable Pages only if it is not already enabled - check first:
   gh api repos/<owner>/<repo>/pages
   On a 404, enable it (substitute the real default branch; do not assume
   main):
   gh api repos/<owner>/<repo>/pages -X POST -F "source[branch]=<default-branch>" -F "source[path]=/"
   A 409 on the POST means it was already enabled - treat that as done, not
   an error.
4. The root index.html lists the artifact slugs, so the bare URL answers
   "what is published here". Rebuild that index on every publish and
   removal.

**Publish / redeploy an artifact:**

1. Write the render to <clone>/<slug>/index.html (copy from the config's
   html path).
2. Regenerate the root index.html listing.
3. Commit and push with a message like "publish <slug>: <title>" (or
   "refresh <slug>"). Pages rebuilds in well under a minute.

**Failure handling:** if the push is rejected, pull --rebase and retry; if
the Pages API 404s, the repo lost its Pages setting - re-run setup step 3.
If gh is unauthenticated, stop and tell the user to run gh auth login; never
fall back to inventing a URL.

**Config block to record:**

    backend: github-pages
    repo: <owner>/<repo>
    clone: <absolute local clone path>
    slug: <slug>

## netlify

One Netlify site serves every artifact as a directory-based site, one
subdirectory per artifact. Requires the netlify CLI (npm install -g
netlify-cli, or the platform's package manager - e.g. brew install
netlify-cli on macOS) and a logged-in account (netlify login). If either is
missing, say so and stop - do not attempt token plumbing unless the user has
already set NETLIFY_AUTH_TOKEN + NETLIFY_SITE_ID, in which case the CLI
works without login.

**First-time setup (once):** from the intended site directory (a stable
local path, e.g. ~/dev/artifacts-netlify), run netlify sites:create and
record the site id it returns. Do NOT use netlify init - that configures
continuous deployment from a git repo, which is a different publishing
model. URLs look like https://<site>.netlify.app/<slug>/.

**Publish / redeploy an artifact:**

1. Write the render to <sitedir>/<slug>/index.html and regenerate the root
   index.html listing (same convention as github-pages).
2. netlify deploy --dir=<sitedir> --prod --site <site-id>
   The --prod flag is what makes the URL stable; a draft deploy is a
   throwaway URL and never counts as publishing.

**Failure handling:** a non-zero exit leaves the live site untouched - read
the error, fix (auth, site id, dir), retry. Never publish a half-written
sitedir: write files first, deploy once.

**Config block to record:**

    backend: netlify
    site: <site-name-or-id>
    dir: <absolute local site directory>
    slug: <slug>

## Choosing between them

- github-pages: default. gh is already part of the toolchain, the repo
  doubles as versioned history of every render, and teammates can open PRs
  against an artifact.
- netlify: pick when the user already runs Netlify, wants deploy previews or
  password protection (paid), or does not want the renders in git history.

Keep the root index.html honest for both backends (it is the landing
page). Record the chosen backend in the artifact config as soon as you
select it.
