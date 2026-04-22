# Release Manifest — Retail Express Product Tools

**v1.4** · A single-page internal tool for generating a list of Jira work items merged into a release branch across one or more Bitbucket repositories within a given date range.

---

## What it does

When preparing a release, you want to know exactly which Jira tickets made it in. This tool automates that by:

1. Querying Bitbucket for all merged pull requests targeting a specific branch (e.g. `release/3.19.04`) across any number of selected repositories in a date range
2. Extracting `MRCDEV-XXXXX` Jira keys from PR titles and source branch names
3. Looking up each key in Jira to retrieve the summary, status, issue type, assignee, parent epic, and team
4. Displaying results in a table with a one-click **Open in Jira** button that launches a pre-run JQL search

---

## How it works

The tool is a single static HTML file (`release-manifest.html`) hosted on GitHub Pages. There is no backend — all API calls are proxied through a Cloudflare Worker to avoid browser CORS restrictions.

### Architecture

```
Browser (GitHub Pages)
    │
    ├── Bitbucket Cloud API  ──►  Cloudflare Worker  ──►  api.bitbucket.org
    │                                (CORS proxy)
    └── Jira Cloud API       ──►  Cloudflare Worker  ──►  maropost.atlassian.net
```

### Cloudflare Worker (CORS Proxy)

Located at `https://releasemanifest.jarrayda.workers.dev`

The worker accepts a `POST` request with a JSON body:

```json
{ "url": "https://api.bitbucket.org/...", "headers": { "Authorization": "Basic ..." } }
```

It fetches the target URL server-side with the supplied headers and returns the response body with `Access-Control-Allow-Origin: *`. This sidesteps the CORS restrictions that both Bitbucket and Jira impose on browser-originated requests.

Worker source:

```js
export default {
  async fetch(request) {
    if (request.method === 'OPTIONS') {
      return new Response(null, {
        headers: {
          'Access-Control-Allow-Origin': '*',
          'Access-Control-Allow-Methods': 'GET, POST, OPTIONS',
          'Access-Control-Allow-Headers': '*',
        }
      });
    }
    const { url, headers } = await request.json();
    const resp = await fetch(url, {
      headers: { ...headers, 'Accept': 'application/json' }
    });
    return new Response(await resp.text(), {
      status: resp.status,
      headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' }
    });
  }
};
```

### Authentication

Two separate Atlassian API tokens are required:

| Token | Purpose | Required scopes |
|-------|---------|-----------------|
| **Jira API token** | Authenticates Jira REST API calls (`/rest/api/3/...`) | Any scope (standard Atlassian API token) |
| **Bitbucket API token** | Authenticates Bitbucket REST API calls | `Repositories: Read`, `Pull requests: Read` |

Both are created at [id.atlassian.com → Security → API tokens](https://id.atlassian.com/manage-profile/security/api-tokens). The Bitbucket token must be created using **"Create API token with scopes"** with the Bitbucket app selected.

The Atlassian email is saved to `localStorage` after a successful connect for convenience. It can be cleared via the **✕ clear** link next to the email field. Note: clearing browser cache does not clear `localStorage` — use "Clear site data" or the in-app clear button.

### Key data flow

1. **Connect** — verifies Jira credentials via `/rest/api/3/myself`, then fetches the full Bitbucket repository list for the workspace. On success the connect card auto-collapses to a compact "Connected as…" bar.

2. **Repo selection** — a multi-select picker lists all repos in the workspace (all selected by default). Changing selection triggers a branch reload.

3. **Branch type filter** — filters branches server-side via Bitbucket's `q=` parameter before fetching. Defaults to Release, Feature, and Main. Selecting "Other" fetches all and filters client-side. Changing types re-fetches branches.

4. **Branch list** — fetched from all selected repos in parallel (2025 cutoff). Release branches are sorted by version number descending (highest first, e.g. `3.19.04` before `3.19.03.3`). Recency of PR activity determines ordering for non-release branches. Includes a searchable dropdown with highlight matching.

5. **Query** — fetches merged PRs using the Bitbucket `q=` filter (`state="MERGED" AND destination.branch.name="<branch>"`), run in parallel across all selected repos. Extracts `MRCDEV-XXXXX` keys from PR titles and source branch names.

6. **Jira enrichment** — batch-queries Jira for issue details (summary, status, type, assignee, parent epic, team field `customfield_12800`). A second batch resolves parent issue summaries so epic names display as links.

7. **Render** — results table with type/status badges, assignee avatars, epic link, team, date merged, repos (pill badges per repo), and PR count. Includes Export CSV and Open in Jira buttons.

---

## Configuration

Hardcoded constants in the HTML (update if workspace/URLs change):

```js
const JIRA_URL     = 'https://maropost.atlassian.net';
const BB_WORKSPACE = 'maropost-retail-cloud';
const WORKER       = 'https://releasemanifest.jarrayda.workers.dev';
```

---

## Deployment

The tool is a single static file — no build step required.

1. Edit `release-manifest.html` directly
2. Commit and push to the `main` branch
3. GitHub Pages serves it automatically at `https://jarrayda.github.io/release-manifest/release-manifest.html`

The previous version is archived as `release-manifest.html.old` before each update.

---

## Version history

| Version | Changes |
|---------|---------|
| v1.4 | Collapsible connect card (auto-collapses on connect), version-sorted release branches, branch type filter re-fetches from API, local date fix for timezone offset, cached email clear button, expand/disconnect flow |
| v1.3 | Branch type multi-select (Release/Feature/Main/Other), fetches branches from all selected repos combined, `✕ clear` button for cached email, View Source link in topbar, wider layout (1820px) |
| v1.2 | Multi-repo support — repo picker fetches real repo list from Bitbucket, all repos selected by default, PRs fetched in parallel across repos, Repos column in results grid |
| v1.1 | Open in Jira button, wider layout, searchable branch picker, PR count fix (off-by-one), branch sort by recent PR activity |
| v1.0 | Initial release — single repo, Cloudflare Worker CORS proxy, Epic/Team/Date Merged columns, Jira search endpoint updated to `/rest/api/3/search/jql` |
