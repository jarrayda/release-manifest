# Release Manifest — Retail Express Product Tools

**v1.3** · A single-page internal tool for generating a list of Jira work items merged into a release branch across one or more Bitbucket repositories within a given date range.

---

## What it does

When preparing a release, you want to know exactly which Jira tickets made it in. This tool automates that by:

1. Querying Bitbucket for all merged pull requests targeting a specific branch (e.g. `release/3.19.04`) across any number of repositories in a date range
2. Extracting `MRCDEV-XXXXX` Jira keys from PR titles and source branch names
3. Looking up each key in Jira to retrieve the summary, status, issue type, assignee, parent epic, and team
4. Displaying results in a table with a one-click "Open in Jira" button that launches a pre-run JQL search

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

Both are created at [id.atlassian.com → Security → API tokens](https://id.atlassian.com/manage-profile/security/api-tokens). The Bitbucket token must be created using **"Create API token with scopes"** and have the Bitbucket app selected.

Credentials are never stored server-side. The Atlassian email is cached in `localStorage` for convenience.

### Key data flow

1. **Connect** — verifies Jira credentials via `/rest/api/3/myself`, then fetches the Bitbucket repository list for the workspace and loads branch names from all selected repos in parallel
2. **Query** — fetches merged PRs using the Bitbucket `q=` filter (`state="MERGED" AND destination.branch.name="<branch>"`), extracts `MRCDEV-XXXXX` keys from PR titles and source branch names
3. **Jira enrichment** — batch-queries Jira for issue details (summary, status, type, assignee, parent epic, team field `customfield_12800`)
4. **Parent epic names** — a second Jira batch query resolves parent issue summaries so epic names display as links rather than just keys
5. **Render** — results table with type/status badges, assignee avatars, epic link, team, date merged, repos, and PR count

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
| v1.3 | Multi-repo support, Repos column, branch picker loads from all selected repos, email cache, GitHub link |
| v1.2 | Multi-repo picker (fetches repos from Bitbucket), parallel PR fetch across repos |
| v1.1 | Open in Jira button, wider layout, branch picker with search |
| v1.0 | Initial release — single repo, Cloudflare Worker proxy, Epic/Team/Date Merged columns |
